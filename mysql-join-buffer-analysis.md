# MySQL 8.0 Join Buffer 源码深度分析

> 基于 MySQL 8.0.41 源码，分析 join buffer 的内存管理机制与多表连接场景下的实现细节。

---

## 目录

1. [概述与演进](#1-概述与演进)
2. [整体架构](#2-整体架构)
3. [优化阶段：join buffer 决策](#3-优化阶段join-buffer-决策)
4. [Hash Join：主力 join buffer 实现](#4-hash-join主力-join-buffer-实现)
   - 4.1 [行数据的序列化：pack_rows](#41-行数据的序列化pack_rows)
   - 4.2 [HashJoinRowBuffer：内存哈希表](#42-hashjoinrowbuffer内存哈希表)
   - 4.3 [HashJoinIterator：执行控制](#43-hashjoiniterator执行控制)
   - 4.4 [溢出磁盘：HashJoinChunk](#44-溢出磁盘hashjoинchunk)
5. [BKA：索引访问场景的 join buffer](#5-bka索引访问场景的-join-buffer)
6. [多表连接场景下的 join buffer](#6-多表连接场景下的-join-buffer)
7. [关键参数与调优](#7-关键参数与调优)
8. [总结](#8-总结)

---

## 1. 概述与演进

MySQL 的 join buffer 机制历经多次演进：

| 版本 | 策略 | 说明 |
|------|------|------|
| 早期 | BNL (Block Nested Loop) | 分块嵌套循环，线性扫描匹配 |
| 5.6+ | BKA (Batched Key Access) | 批量索引访问，结合 MRR |
| **8.0.18+** | **Hash Join** | 彻底取代 BNL，使用哈希表提升等值连接性能 |

**核心变化**：在 MySQL 8.0 中，原来的 `BNL` 已经被 Hash Join 所替代。源码中 `QEP_TAB::OT_BNL` 在执行器中会被映射为 `HashJoinIterator`（见 `sql_executor.cc`）：

```cpp
// sql/sql_executor.cc:2239
static bool UseHashJoin(QEP_TAB *qep_tab) {
  return qep_tab->op_type == QEP_TAB::OT_BNL;  // BNL → Hash Join
}
```

---

## 2. 整体架构

join buffer 相关的核心文件结构如下：

```
sql/
├── sql_join_buffer.h          # JOIN_CACHE 枚举定义 (ALG_BNL / ALG_BKA)
├── sql_optimizer.cc           # setup_join_buffering()：优化阶段决策
├── sql_executor.cc            # ConnectJoins()：构建执行器树
├── pack_rows.h / .cc          # 行数据序列化/反序列化工具
└── iterators/
    ├── hash_join_buffer.h/.cc   # HashJoinRowBuffer：内存哈希表
    ├── hash_join_iterator.h/.cc # HashJoinIterator：Hash Join 执行器
    ├── hash_join_chunk.h/.cc    # HashJoinChunk：磁盘溢出文件
    └── bka_iterator.h/.cc       # BKAIterator + MultiRangeRowIterator
```

**调用链全景**：

```
优化阶段
  └─ setup_join_buffering()          [sql_optimizer.cc]
       └─ tab->set_use_join_cache(ALG_BNL / ALG_BKA)

执行计划构建阶段
  └─ ConnectJoins()                  [sql_executor.cc]
       ├─ UseHashJoin() → CreateHashJoinAccessPath()
       └─ UseBKA()     → CreateBKAAccessPath()

执行阶段（Iterator 模型）
  ├─ HashJoinIterator::Init() / Read()
  │    └─ HashJoinRowBuffer::StoreRow()
  │         └─ MEM_ROOT alloc + ankerl::unordered_dense::segmented_map
  └─ BKAIterator::Init() / Read()
       └─ MultiRangeRowIterator::Init() → handler::multi_range_read_init()
```

---

## 3. 优化阶段：join buffer 决策

优化器在 `make_join_readinfo()` → `setup_join_buffering()` 中决定每张表是否使用 join buffer，以及使用何种算法。

### 3.1 决策函数

```cpp
// sql/sql_optimizer.cc:3452
static bool setup_join_buffering(JOIN_TAB *tab, JOIN *join, uint no_jbuf_after)
```

决策的核心流程如下：

```
检查 bnl_on / bka_on 开关
    ↓
检查各种禁止条件（LATERAL 依赖、外连接嵌套层级、半连接策略等）
    ↓
根据 tab->type() 分支：
  JT_ALL / JT_INDEX_SCAN / JT_RANGE / JT_INDEX_MERGE
      → set_use_join_cache(ALG_BNL)    // 全表/范围扫描 → Hash Join
  JT_REF / JT_EQ_REF
      → 查询 MRR 支持情况
      → set_use_join_cache(ALG_BKA)    // 索引查找 → BKA
```

### 3.2 禁止 join buffer 的条件

以下情形不允许使用 join buffer（直接 `goto no_join_cache`）：

| 条件 | 原因 |
|------|------|
| `tableno == join->const_tables` | 第一个非常量表不能缓存 |
| `tab->use_quick == QS_DYNAMIC_RANGE` | 动态范围访问不兼容 |
| `tableno > no_jbuf_after` | 超出允许的缓冲表编号上限 |
| 外连接嵌套层级不一致 | 内层表的第一张表必须先决定是否缓存 |
| LATERAL 依赖 | 缓冲多行后再物化 lateral 派生表效率极差 |
| SJ_OPT_LOOSE_SCAN 半连接 | LooseScan 策略不兼容 |
| BKA 下有 NULL 守卫条件 | 子查询中 IN 的特殊情形 |

### 3.3 BKA 的额外检查

```cpp
// sql/sql_optimizer.cc:3666
rows = tab->table()->file->multi_range_read_info(
    tab->ref().key, 10, 20, &bufsz, &join_cache_flags, &cost);

// 禁用 BKA 的条件：
// 1. MRR 不可用
// 2. handler 使用默认 MRR 实现（性能差）
// 3. HA_MRR_NO_ASSOCIATION 标志（无法建立范围到行的对应关系）
if ((rows == HA_POS_ERROR) ||
    (join_cache_flags & HA_MRR_USE_DEFAULT_IMPL) ||
    (join_cache_flags & HA_MRR_NO_ASSOCIATION))
  goto no_join_cache;
```

---

## 4. Hash Join：主力 join buffer 实现

### 4.1 行数据的序列化：pack_rows

在存入 join buffer 之前，行数据需要从各张表的 `record[0]` 缓冲区序列化为连续字节串。

**核心数据结构**：

```cpp
// sql/pack_rows.h
namespace pack_rows {

struct Column {
  Field *const field;
  const enum_field_types field_type;  // 缓存字段类型，提升 30% 性能
};

struct Table {
  TABLE *table;
  Prealloced_array<Column, 8> columns;  // 仅包含 read_set 中需要的列
  bool copy_null_flags;                  // 是否需要 NULL 标志位
};

class TableCollection {
  // 对多张表的封装，用于 hash join / BKA / 聚合
  Prealloced_array<Table, 4> m_tables;
  table_map m_tables_bitmap;
  bool m_has_blob_column;   // 有 BLOB 时无法预计算行大小上界
};

}  // namespace pack_rows
```

序列化过程（`StoreFromTableBuffers`）：将每张表所有需要的列按顺序打包为连续内存，NULL 标志位和行 ID（rowid）可选地一并写入。反序列化通过 `LoadIntoTableBuffers` 将其还原到 `record[0]`。

---

### 4.2 HashJoinRowBuffer：内存哈希表

这是 join buffer 的核心数据结构，定义在 `sql/iterators/hash_join_buffer.h`。

**内存组织全景**：

```
┌─────────────────────────────────────────────────────────┐
│               HashJoinRowBuffer                          │
│                                                         │
│  ┌──────────────────────────────┐                       │
│  │   HashMap (heap内存)          │                       │
│  │  ankerl::unordered_dense      │                       │
│  │  ::segmented_map              │                       │
│  │                               │                       │
│  │  Key: ImmutableStringWithLen  │ ←── join条件列的值    │
│  │  Val: LinkedImmutableString   │ ←── 指向行数据的链表  │
│  └──────────────────────────────┘                       │
│                                                         │
│  ┌──────────────────────────────┐                       │
│  │   m_mem_root (MEM_ROOT)       │ ←── key + value存储   │
│  │   初始块: 16 KB               │     使用 join_buffer_ │
│  │   容量上限: join_buffer_size   │     size 控制         │
│  └──────────────────────────────┘                       │
│                                                         │
│  ┌──────────────────────────────┐                       │
│  │   m_overflow_mem_root         │ ←── 最后一行溢出分配  │
│  │   初始块: 256 字节            │                       │
│  └──────────────────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

**LinkedImmutableString**：相同 key 的多行数据通过链表组织——每个 value 头部编码一个指向前一个 value 的指针（`next_ptr`），从而在哈希表单个 bucket 中存储多行：

```
HashMap bucket:
  key → LinkedImmutableString(row3) → row3_data
                    ↓ next_ptr
               LinkedImmutableString(row2) → row2_data
                    ↓ next_ptr
               LinkedImmutableString(row1) → row1_data
                    ↓ next_ptr
               nullptr
```

**StoreRow() 内存控制核心逻辑**：

```cpp
// sql/iterators/hash_join_buffer.cc:216
StoreRowResult HashJoinRowBuffer::StoreRow(THD *thd, bool reject_duplicate_keys) {
  // 1. 从 join 条件计算哈希键，追加到 m_buffer
  for (const HashJoinCondition &cond : m_join_conditions) {
    cond.join_condition()->append_join_key_for_hash_join(...);
  }

  // 2. 将 key 序列化到 m_mem_root（延迟 commit，若已存在则不 commit）
  key = ImmutableStringWithLength::Encode(m_buffer.ptr(), m_buffer.length(), &ptr);
  m_hash_map->emplace(key, LinkedImmutableString{nullptr});

  // 3. 插入新 key 后，根据哈希表实际占用调整 m_mem_root 容量上限
  const size_t bytes_used =
      m_hash_map->bucket_count() * sizeof(bucket_type) +
      m_hash_map->values().capacity() * sizeof(value_type);
  if (bytes_used >= m_max_mem_available) {
    m_mem_root.set_max_capacity(1);  // 强制下次分配失败 → BUFFER_FULL
    full = true;
  } else {
    m_mem_root.set_max_capacity(m_max_mem_available - bytes_used);
  }

  // 4. 序列化行数据到 m_mem_root（或 m_overflow_mem_root）
  m_last_row_stored = StoreLinkedImmutableStringFromTableBuffers(next_ptr, &full);

  return full ? BUFFER_FULL : ROW_STORED;
}
```

**关键设计**：内存检查是在插入之后执行的，因此 buffer 实际用量可能略超 `join_buffer_size`，但保证至少能插入一行。

---

### 4.3 HashJoinIterator：执行控制

`HashJoinIterator` 是一个有限状态机，通过状态变量 `m_state` 控制执行流程。

**三种 Hash Join 模式**：

```cpp
// sql/iterators/hash_join_iterator.h:639
enum class HashJoinType {
  IN_MEMORY,                       // 内存够，全程不落盘
  SPILL_TO_DISK,                   // 内存不足，分区落盘
  IN_MEMORY_WITH_HASH_TABLE_REFILL // 禁止落盘但内存不足，多次 refill
};
```

**状态机转换图**：

```
                    ┌─────────────────────────────┐
                    │         Init()               │
                    │   BuildHashTable()           │
                    └───────────┬─────────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            │                   │                   │
      内存充足             内存不足               内存不足
   (IN_MEMORY)          (SPILL_TO_DISK)    (NO_SPILL + REFILL)
            │                   │                   │
            ▼                   ▼                   ▼
  READING_ROW_FROM         开始分区            READING_ROW_FROM
  _PROBE_ITERATOR         落盘处理            _PROBE_ITERATOR
            │                   │                   │
            ▼                   ▼                   ▼
  READING_FIRST_ROW    LOADING_NEXT_CHUNK   哈希表满时重读
  _FROM_HASH_TABLE     _PAIR → 重建哈希表    probe input
            │
            ▼
  READING_FROM_HASH_TABLE（遍历同 key 的多行）
            │
            ▼
         END_OF_ROWS
```

**BuildHashTable() 关键路径**：

```cpp
// sql/iterators/hash_join_iterator.cc:410
bool HashJoinIterator::BuildHashTable() {
  // 初始化并清空 row buffer
  if (InitRowBuffer()) return true;

  for (;;) {
    int res = m_build_input->Read();  // 读 build 端（内表）一行
    if (res == -1) {                  // EOF：全部放入内存，纯内存 Hash Join
      m_build_iterator_has_more_rows = false;
      SetReadingProbeRowState();
      return false;
    }

    const StoreRowResult result = m_row_buffer.StoreRow(thd(), reject_dup_keys);

    if (result == BUFFER_FULL) {
      if (!m_allow_spill_to_disk) {
        // 禁止落盘：切换到 REFILL 模式，开启 probe row saving
        if (m_join_type != JoinType::INNER)
          InitWritingToProbeRowSavingFile();
        m_hash_join_type = HashJoinType::IN_MEMORY_WITH_HASH_TABLE_REFILL;
        SetReadingProbeRowState();
        return false;
      }
      // 允许落盘：初始化 chunk 文件，将剩余行写出
      InitializeChunkFiles(...);
      WriteRowsToChunks(thd(), m_build_input.get(), ...);  // build 剩余行
      m_hash_join_type = HashJoinType::SPILL_TO_DISK;
      SetReadingProbeRowState();
      return false;
    }
  }
}
```

---

### 4.4 溢出磁盘：HashJoinChunk

当内存不足时，数据被分区写入磁盘临时文件（`IO_CACHE`）。

**分区策略**：

```cpp
// sql/iterators/hash_join_iterator.cc:306
// 使用与哈希表不同的种子（kChunkPartitioningHashSeed = 899339）
// 避免分区后的 chunk 内部也出现严重哈希冲突
const uint64_t join_key_hash = MY_XXH64(..., xxhash_seed);
const size_t chunk_index = join_key_hash & (chunks->size() - 1); // 2 的幂次
```

**Chunk 数量计算**：

```cpp
// sql/iterators/hash_join_iterator.cc:370
static bool InitializeChunkFiles(size_t estimated_rows_produced_by_join,
                                 size_t rows_in_hash_table, ...) {
  constexpr double kReductionFactor = 0.9;  // 保守估计，宁多勿少
  const size_t chunks_needed =
      std::max(1, ceil(remaining_rows / (rows_in_hash_table * 0.9)));
  const size_t num_chunks = std::min(kMaxChunks, chunks_needed);
  // 对齐到 2 的幂次（kMaxChunks = 128）
  const size_t num_chunks_pow_2 = my_round_up_to_next_power(num_chunks);
  ...
}
```

**落盘模式下的完整执行流程**：

```
Phase 1：Build
  读 build input → 装入内存哈希表
  内存满 → 将剩余 build 行按哈希分区写入 chunk_pair[i].build_chunk

Phase 2：Probe（同时写磁盘）
  读 probe input → 查内存哈希表，输出匹配行
  同时将 probe 行按同一分区写入 chunk_pair[i].probe_chunk

Phase 3：处理磁盘 chunks
  for each (build_chunk[i], probe_chunk[i]):
    将 build_chunk[i] 读入内存哈希表
    扫描 probe_chunk[i]，逐行查哈希表输出结果
    （若 build_chunk 仍超内存，则再次分块处理）
```

**ChunkPair 的对称性**：build 和 probe 使用完全相同的分区哈希函数，确保能在同一个 chunk pair 中找到所有匹配行。

---

## 5. BKA：索引访问场景的 join buffer

BKA 适用于内表有可用索引（`JT_REF`/`JT_EQ_REF`）的情形，通过批量收集外表行的 join key，一次性调用 MRR（Multi-Range Read）接口读取内表，改善随机 I/O。

### 5.1 BKAIterator

```cpp
// sql/iterators/bka_iterator.h:82
class BKAIterator final : public RowIterator {
  MEM_ROOT m_mem_root;                              // 缓冲外表行 + MRR buffer
  Mem_root_array<hash_join_buffer::BufferRow> m_rows; // 缓冲的外表行数组
  pack_rows::TableCollection m_outer_input_tables;    // 外表列集合
  size_t m_mrr_bytes_needed_per_row;                  // MRR buffer 每行预留
  size_t m_bytes_used;                                // 已使用的内存
  const size_t m_max_memory_available;                // = join_buffer_size
  MultiRangeRowIterator *m_mrr_iterator;              // 内表 MRR 执行器
};
```

**内存分配策略**：BKA 的 `m_mem_root` 同时服务于两种用途：
1. 存储外表行数据（序列化字节串）
2. 为 MRR 的内表行缓冲区预留空间

缓冲外表行时，如果当前已用内存 + MRR 预留量超过 `join_buffer_size`，就停止读取外表并开始 MRR 查询：

```cpp
// ReadOuterRows() 停止条件（bka_iterator.cc）
m_bytes_used += row_size + m_mrr_bytes_needed_per_row;
if (m_bytes_used >= m_max_memory_available) {
  // 停止读外表，开始 MRR
  break;
}
```

### 5.2 MultiRangeRowIterator

```cpp
class MultiRangeRowIterator final : public TableRowIterator {
  HANDLER_BUFFER m_mrr_buffer;                // MRR 内部行缓冲
  uchar *m_match_flag_buffer;                 // 外连接/半连接的匹配标志位
  const hash_join_buffer::BufferRow *m_begin; // 外表行数组起点
  const hash_join_buffer::BufferRow *m_end;   // 外表行数组终点
};
```

MRR 初始化时，将外表行中的 join key 提取为一批 `KEY_MULTI_RANGE`，交给存储引擎（如 InnoDB）按任意顺序（通常是主键顺序）批量读取，避免随机 I/O。

---

## 6. 多表连接场景下的 join buffer

### 6.1 左深树结构

MySQL 的多表连接计划是一棵**左深树**（left-deep tree）。对于 `t1 JOIN t2 JOIN t3 JOIN t4`，执行器树形如：

```
              HashJoin(t3)
             /            \
        HashJoin(t2)       t3 scan
       /            \
  HashJoin(t1)       t2 scan
 /            \
t1 scan        ???（不可能，t1 是最左表）
```

实际结构（t1 最外层驱动，逐步向内 hash join）：

```
HashJoin(probe=t1∧t2∧t3, build=t4)
    └── probe: HashJoin(probe=t1∧t2, build=t3)
                   └── probe: HashJoin(probe=t1, build=t2)
                                   └── probe: TableScan(t1)
                                   └── build: TableScan(t2)
                   └── build: TableScan(t3)
    └── build: TableScan(t4)
```

### 6.2 ConnectJoins() 递归构建

```cpp
// sql/sql_executor.cc
// ConnectJoins() 递归地为每张表判断应该用什么 join 策略
for (uint i = first_idx; i < last_idx; ) {
  QEP_TAB *qep_tab = &qep_tabs[i];

  // 每张表独立判断：hash join 还是 BKA 还是 nested loop
  if (UseHashJoin(qep_tab) && !QueryMixesOuterBKAAndBNL(...)) {
    path = CreateHashJoinAccessPath(thd, qep_tab,
                                    build=table_path, probe=path, ...);
  } else if (UseBKA(qep_tab) && !QueryMixesOuterBKAAndBNL(...)) {
    path = CreateBKAAccessPath(thd, join, outer_path=path,
                               inner_path=table_path, ...);
  } else {
    path = CreateNestedLoopAccessPath(thd, path, table_path, ...);
  }
}
```

### 6.3 每层 join buffer 独立管理

**每个 `HashJoinIterator` 都有独立的 `HashJoinRowBuffer`，各自独占一份 `join_buffer_size` 内存**。

```
4 表连接的内存使用：

HashJoinIterator(t4)
  └── m_row_buffer（最多 join_buffer_size 字节存 t4 的行）

HashJoinIterator(t3)
  └── m_row_buffer（最多 join_buffer_size 字节存 t3 的行）

HashJoinIterator(t2)
  └── m_row_buffer（最多 join_buffer_size 字节存 t2 的行）

总内存 ≈ (N-1) × join_buffer_size
```

**注意**：各层 hash table 并非同时在内存中。由于迭代器采用拉模型（pull model），外层 HashJoinIterator 在调用 `m_probe_input->Read()` 时，内层 HashJoinIterator 才被驱动，且其 hash table 已经建好。因此，最深层（第一个被 init 的）hash table 先建，被 probe 完后（理论上可以被回收，但目前实现中在 `Init()` 重新调用时才会被清空重建）。

### 6.4 外连接与反连接的特殊处理

当存在外连接时，`QueryMixesOuterBKAAndBNL()` 检查是否存在"外层 BKA + 内层 Hash Join"的组合：

```cpp
// sql/sql_executor.cc:2272
static bool QueryMixesOuterBKAAndBNL(JOIN *join) {
  bool has_outer_bka = false;
  bool has_bnl = false;
  for (uint i = join->const_tables; i < join->primary_tables; ++i) {
    QEP_TAB *qep_tab = &join->qep_tab[i];
    if (UseHashJoin(qep_tab))                               has_bnl = true;
    else if (qep_tab->op_type == OT_BKA &&
             qep_tab->last_inner() != NO_PLAN_IDX)          has_outer_bka = true;
  }
  return has_bnl && has_outer_bka;  // 两者共存时，均退化为 nested loop
}
```

**原因**：`MultiRangeRowIterator` 无法作为 Hash Join 的内表（因为 MRR 需要与 BKA 的 `match_flag_buffer` 配合工作），因此禁止这种组合。

### 6.5 Probe Row Saving 机制

在以下两种情况下，probe 端的行需要被保存到临时文件，以防止多次读取导致外连接/反连接输出重复行：

- **SPILL_TO_DISK 且单个 chunk 过大**：probe chunk 需要被多次扫描
- **IN_MEMORY_WITH_HASH_TABLE_REFILL**：禁止落盘但 build 端过大，需要多次 refill

```
probe row saving 文件 (write)
  ↓ 所有未匹配的 probe 行写入
swap()
probe row saving 文件 (read)
  ↓ 下一次扫描时读取（跳过已匹配行）
```

---

## 7. 关键参数与调优

### 7.1 join_buffer_size

控制每个 `HashJoinRowBuffer` / `BKAIterator` 的内存上限：

```sql
SET join_buffer_size = 256 * 1024;  -- 默认 256 KB，范围 128B ~ 4GB（32位）
```

- **内存检查时机**：插入一行**之后**检查，可能轻微超出
- **最小值保障**：即使设置极小值，代码也会保证至少 16 KB（`max(max_mem_available, 16384)`）
- **多表乘效应**：N 表连接同时运行时，实际峰值内存约为 `(N-1) × join_buffer_size`

### 7.2 optimizer_switch 控制开关

```sql
SET optimizer_switch = 'hash_join=on';  -- 控制 Hash Join（默认 on）
SET optimizer_switch = 'bnl=on';        -- 历史变量，实际控制 Hash Join
SET optimizer_switch = 'bka=on';        -- 控制 BKA（默认 on）
SET optimizer_switch = 'mrr=on';        -- BKA 依赖 MRR
SET optimizer_switch = 'mrr_cost_based=on'; -- MRR 是否做代价评估
```

### 7.3 落盘阈值

不存在直接控制 Hash Join 落盘的参数，触发落盘的条件是 `HashJoinRowBuffer` 满（即 build 端数据量超过 `join_buffer_size`）。可通过以下方式规避落盘：

1. 增大 `join_buffer_size`
2. 确保 build 端（通常是较小的表）作为内表（优化器通常会自动选择）
3. 对于禁止落盘的场景（有 `LIMIT` 无排序/聚合），MySQL 自动启用 `IN_MEMORY_WITH_HASH_TABLE_REFILL` 模式

### 7.4 EXPLAIN ANALYZE 观察

```sql
EXPLAIN ANALYZE SELECT ...;
-- 输出示例：
-- -> Hash join  (cost=... rows=...)
--     -> Table scan on t2  (actual time=... rows=...)
--     -> Hash
--         -> Table scan on t1  (actual time=... rows=...)
```

- 出现 `Hash` 节点表示该表被用作 build 端（装入内存哈希表）
- 出现 `Batched key access` 表示使用 BKA

---

## 8. 总结

### 设计要点

| 特性 | Hash Join (BNL替代) | BKA |
|------|--------------------|----|
| **适用场景** | 全表扫描/范围扫描的等值连接 | 内表有可用索引的等值连接 |
| **buffer 作用** | 存放 build 端行，构建哈希表 | 缓冲外表行的 join key，批量 MRR |
| **内存结构** | `MEM_ROOT` + `ankerl::segmented_map` | `MEM_ROOT`（行数据 + MRR buffer）|
| **内存控制** | `join_buffer_size` 限制 MEM_ROOT 容量 | `join_buffer_size` 限制总分配量 |
| **溢出处理** | 分区落盘（最多 128 个 chunk pair）| 批处理（batch 满即发起 MRR）|
| **多 join 内存** | 每层独立，总量 ≈ (N-1) × buffer_size | 同左 |

### 核心代码路径速查

| 功能 | 文件 | 函数/类 |
|------|------|---------|
| 优化阶段决策 | `sql/sql_optimizer.cc` | `setup_join_buffering()` |
| 执行树构建 | `sql/sql_executor.cc` | `ConnectJoins()` |
| Hash Join 内存哈希表 | `sql/iterators/hash_join_buffer.cc` | `HashJoinRowBuffer::StoreRow()` |
| Hash Join 执行控制 | `sql/iterators/hash_join_iterator.cc` | `HashJoinIterator::BuildHashTable()` |
| 磁盘落盘 chunk | `sql/iterators/hash_join_chunk.cc` | `HashJoinChunk::WriteRowToChunk()` |
| BKA 外表缓冲 | `sql/iterators/bka_iterator.cc` | `BKAIterator::ReadOuterRows()` |
| 行序列化/反序列化 | `sql/pack_rows.cc` | `StoreFromTableBuffers()` / `LoadIntoTableBuffers()` |

---

*文档生成时间：2026-03-26*
*参考源码版本：MySQL 8.0.41*
