# MySQL 8.0.41 TempTable 临时表磁盘存储机制分析

## 概述

TempTable 是 MySQL 8.0 引入的新临时表存储引擎，用于替代旧版的 HEAP/MEMORY 引擎处理内部临时表。相比旧引擎，TempTable 支持将数据溢出到磁盘，避免内存不足导致的查询失败。

本文基于 MySQL 8.0.41 源码，分析 TempTable 何时会使用磁盘存储数据。

---

## 1. 核心系统变量

TempTable 有两个关键的配置参数，定义在 `sql/sys_vars.cc`：

```cpp
static Sys_var_ulonglong Sys_temptable_max_ram(
    "temptable_max_ram",
    "Maximum amount of memory (in bytes) the TempTable storage engine is "
    "allowed to allocate from the main memory (RAM) before starting to "
    "store data on disk.",
    GLOBAL_VAR(temptable_max_ram), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(2 << 20 /* 2 MiB */, ULLONG_MAX), DEFAULT(1 << 30 /* 1 GiB */),
    BLOCK_SIZE(1));

static Sys_var_bool Sys_temptable_use_mmap("temptable_use_mmap",
                                           "Use mmap files for temptables",
                                           GLOBAL_VAR(temptable_use_mmap),
                                           CMD_LINE(OPT_ARG), DEFAULT(true));
```

### 参数说明

| 参数 | 默认值 | 最小值 | 作用 |
|------|--------|--------|------|
| `temptable_max_ram` | 1 GB | 2 MB | TempTable 允许从 RAM 分配的最大内存阈值 |
| `temptable_use_mmap` | true | - | 是否允许使用 mmap 文件存储溢出数据 |

**源码位置：**
- 变量声明：`sql/mysqld.cc:1100-1101`
- 系统变量定义：`sql/sys_vars.cc:4917-4929`

---

## 2. 内存监控机制

TempTable 通过 `MemoryMonitor` 结构追踪全局 RAM 使用情况，定义在 `storage/temptable/include/temptable/allocator.h`：

```cpp
struct MemoryMonitor {
 protected:
  /** Log increments of heap-memory consumption. */
  static size_t ram_increase(size_t bytes) {
    return ram.fetch_add(bytes) + bytes;
  }
  
  /** Log decrements of heap-memory consumption. */
  static size_t ram_decrease(size_t bytes) {
    return ram.fetch_sub(bytes) - bytes;
  }
  
  /** Get heap-memory threshold level. */
  static size_t ram_threshold() { return temptable_max_ram; }
  
  /** Get current level of heap-memory consumption. */
  static size_t ram_consumption() { return ram; }

 private:
  /** Total bytes allocated so far by all threads in RAM. */
  static std::atomic<size_t> ram;
};
```

**关键点：**
- `ram` 是一个全局原子变量，追踪所有线程的 TempTable 内存总消耗
- `ram_threshold()` 返回 `temptable_max_ram` 配置值
- 内存统计是全局性的，跨所有连接线程

---

## 3. 内存到磁盘的切换逻辑

核心判断逻辑位于 `Allocator::allocate()` 方法中（`storage/temptable/include/temptable/allocator.h:359-379`）：

```cpp
const Source block_source = [block_size]() {
  // Decide whether to switch between RAM and MMAP-backed allocations.
  if (MemoryMonitor::ram_consumption() >= MemoryMonitor::ram_threshold()) {
    return Source::MMAP_FILE;
  } else {
    if (MemoryMonitor::ram_increase(block_size) <=
        MemoryMonitor::ram_threshold()) {
      return Source::RAM;
    } else {
      MemoryMonitor::ram_decrease(block_size);
      return Source::MMAP_FILE;
    }
  }
}();
```

### 判断流程

```
┌─────────────────────────────────────────────────────────────┐
│                    分配新 Block                              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 当前 RAM 消耗 >= temptable_max_ram ?                        │
└─────────────────────────────────────────────────────────────┘
                    │                   │
                   Yes                 No
                    │                   │
                    ▼                   ▼
            ┌──────────────┐   ┌──────────────────────────────┐
            │ MMAP_FILE    │   │ 尝试增加 ram_counter          │
            │ (磁盘 mmap)  │   │ ram_increase(block_size)      │
            └──────────────┘   └──────────────────────────────┘
                                        │
                                        ▼
                            ┌──────────────────────────────┐
                            │ 增加后的值 <= threshold ?    │
                            └──────────────────────────────┘
                                    │           │
                                   Yes         No
                                    │           │
                                    ▼           ▼
                            ┌──────────────┐ ┌──────────────────────┐
                            │ RAM          │ │ 回退增加              │
                            │ (内存分配)   │ │ ram_decrease         │
                            └──────────────┘ │ 返回 MMAP_FILE       │
                                             └──────────────────────┘
```

### 触发磁盘存储的条件

| 条件 | 说明 |
|------|------|
| `ram_consumption() >= ram_threshold()` | 当前所有 TempTable 的 RAM 总消耗已达到阈值 |
| `ram_increase(block_size) > ram_threshold()` | 新 Block 分配后会导致总内存超过阈值 |

---

## 4. 磁盘存储实现方式

当需要使用磁盘时，TempTable 采用 **MMAP 文件**方式。

### 源码位置

`storage/temptable/include/temptable/memutils.h:191-243`：

```cpp
inline void *Memory<Source::MMAP_FILE>::fetch(size_t bytes) {
#ifdef _WIN32
  const int mode = _O_RDWR;
#else
  const int mode = O_RDWR;
#endif

  char file_path[FN_REFLEN];
  File f = create_temp_file(file_path, mysql_tmpdir, "mysql_temptable.", mode,
                            UNLINK_FILE, MYF(MY_WME));
  if (f < 0) {
    return nullptr;
  }

  /* 预分配文件空间，写入 0xa 字节 */
  if (my_fallocator(f, bytes, 0xa, MYF(MY_WME)) != 0 ||
      my_seek(f, 0, MY_SEEK_SET, MYF(MY_WME)) == MY_FILEPOS_ERROR) {
    my_close(f, MYF(MY_WME));
    return nullptr;
  }

  void *ptr = my_mmap(nullptr, bytes, PROT_READ | PROT_WRITE, MAP_SHARED, f, 0);

  // 文件描述符可立即关闭，mmap 仍有效
  my_close(f, MYF(MY_WME));

  return (ptr == MAP_FAILED) ? nullptr : ptr;
}
```

### 特点

| 特性 | 说明 |
|------|------|
| **存储位置** | `mysql_tmpdir` 目录（由 `tmpdir` 系统变量指定） |
| **文件名格式** | `mysql_temptable.*` |
| **实现方式** | `mmap()` 将文件映射到进程内存地址空间 |
| **文件生命周期** | 创建后立即 unlink，文件描述符在 mmap 后关闭 |
| **访问方式** | 通过内存地址访问，操作系统负责页面调度 |

### mmap 禁用时的处理

```cpp
static void *allocate(size_t bytes) {
  if (!temptable_use_mmap) {
    throw Result::RECORD_FILE_FULL;
  }
  void *memory = fetch(bytes);
  if (memory == nullptr) {
    throw Result::RECORD_FILE_FULL;
  }
  return memory;
}
```

如果 `temptable_use_mmap=false` 或 mmap 创建失败，会抛出 `RECORD_FILE_FULL` 异常。

---

## 5. 降级到 InnoDB 机制

如果 TempTable 无法存储数据（mmap 失败或禁用），MySQL 会自动降级到 InnoDB 存储引擎。

### 源码位置

`sql/sql_tmp_table.cc:2252-2260`：

```cpp
int error =
    table->file->create(share->table_name.str, table, &create_info, nullptr);
if (error == HA_ERR_RECORD_FILE_FULL &&
    table->s->db_type() == temptable_hton) {
  // TempTable 无法创建，切换到 InnoDB
  table->file =
      get_new_handler(table->s, false, table->in_use->mem_root, innodb_hton);
  error = table->file->create(share->table_name.str, table, &create_info,
                              nullptr);
}
```

### 统计计数

```cpp
if (table->s->db_type() != temptable_hton) {
  table->in_use->inc_status_created_tmp_disk_tables();
}
```

**注意：** TempTable 的 mmap 文件不算入 `Created_tmp_disk_tables` 状态计数，只有降级到 InnoDB 的临时表才算。

---

## 6. Block 分配策略

TempTable 的内存分配采用 Block（大块）+ Chunk（小块）两级结构。

### Block 大小限制

`storage/temptable/include/temptable/constants.h:59-68`：

```cpp
/** log2(allocator max block size in MiB). */
constexpr size_t ALLOCATOR_MAX_BLOCK_MB_EXP = 9;

/** Limit on the size of a block created by `Allocator`. */
constexpr size_t ALLOCATOR_MAX_BLOCK_BYTES = 1_MiB << ALLOCATOR_MAX_BLOCK_MB_EXP;
// = 512 MiB

/** `Storage` page size. */
constexpr size_t STORAGE_PAGE_SIZE = 64_KiB;
```

### Block 大小增长策略

采用指数增长策略，定义在 `storage/temptable/include/temptable/allocator.h:105-122`：

```cpp
struct Exponential {
  static size_t block_size(size_t number_of_blocks, size_t n_bytes_requested) {
    size_t block_size_hint;
    if (number_of_blocks < ALLOCATOR_MAX_BLOCK_MB_EXP) {
      block_size_hint = (1ULL << number_of_blocks) * 1_MiB;
    } else {
      block_size_hint = ALLOCATOR_MAX_BLOCK_BYTES;
    }
    return std::max(block_size_hint, Block::size_hint(n_bytes_requested));
  }
};
```

**Block 大小增长序列：**
- 1 MB → 2 MB → 4 MB → 8 MB → 16 MB → 32 MB → ... → 512 MB（最大）

---

## 7. 内存来源枚举

`storage/temptable/include/temptable/memutils.h:67-72`：

```cpp
enum class Source {
  /** Memory is allocated on disk, using mmap()'ed file. */
  MMAP_FILE,
  /** Memory is allocated from RAM, using malloc() for example. */
  RAM,
};
```

---

## 8. 总结：TempTable 使用磁盘的场景

| 场景 | 触发条件 | 存储方式 | 统计影响 |
|------|---------|---------|---------|
| **RAM 达到阈值** | 全局 TempTable RAM 总消耗 ≥ `temptable_max_ram` | mmap 临时文件 | 不计入 `Created_tmp_disk_tables` |
| **单次分配超限** | 新 Block 分配后总内存 > `temptable_max_ram` | mmap 临时文件 | 不计入 `Created_tmp_disk_tables` |
| **mmap 禁用** | `temptable_use_mmap=false` | 降级到 InnoDB 磁盘表 | 计入 `Created_tmp_disk_tables` |
| **mmap 创建失败** | `tmpdir` 空间不足或 mmap 调用失败 | 降级到 InnoDB 磁盘表 | 计入 `Created_tmp_disk_tables` |
| **磁盘空间不足** | InnoDB 临时表空间不足 | 查询失败（ER_RECORD_FILE_FULL） | - |

---

## 9. 监控与调优建议

### 监控指标

```sql
-- 查看临时表创建统计
SHOW STATUS LIKE 'Created_tmp%';

-- 查看当前配置
SHOW VARIABLES LIKE 'temptable%';
SHOW VARIABLES LIKE 'tmpdir';
SHOW VARIABLES LIKE 'internal_tmp_mem_storage_engine';
```

### 相关状态变量

| 变量 | 说明 |
|------|------|
| `Created_tmp_tables` | 创建的内部临时表总数 |
| `Created_tmp_disk_tables` | 创建的磁盘临时表数（仅统计 InnoDB 降级的情况） |

### Performance Schema 监控

可通过 Performance Schema 内存表监控 TempTable 内存使用：

```sql
SELECT * FROM performance_schema.memory_summary_global_by_event_name 
WHERE EVENT_NAME LIKE 'memory/temptable%';
```

### 调优建议

1. **增大 `temptable_max_ram`**：如果频繁降级到磁盘，可增大此值（默认 1GB）
2. **确保 `tmpdir` 空间充足**：mmap 文件需要磁盘空间支持
3. **保持 `temptable_use_mmap=true`**：允许 mmap 比直接降级到 InnoDB 性能更好
4. **监控 `Created_tmp_disk_tables`**：如果此值增长快，说明 TempTable 经常降级

---

## 10. 相关源码文件索引

| 文件路径 | 主要功能 |
|----------|---------|
| `storage/temptable/include/temptable/allocator.h` | 内存分配器，RAM/mmap 切换逻辑 |
| `storage/temptable/include/temptable/memutils.h` | 内存来源枚举，RAM 和 mmap 实现 |
| `storage/temptable/include/temptable/block.h` | Block 抽象，内存块管理 |
| `storage/temptable/include/temptable/constants.h` | 常量定义（最大 Block 大小等） |
| `storage/temptable/src/handler.cc` | TempTable handler 接口实现 |
| `sql/sql_tmp_table.cc` | 临时表创建逻辑，TempTable 到 InnoDB 降级 |
| `sql/sys_vars.cc` | 系统变量定义（temptable_max_ram 等） |
| `sql/mysqld.cc` | 全局变量声明 |

---

## 参考文献

- MySQL 8.0 Reference Manual: [TempTable Storage Engine](https://dev.mysql.com/doc/refman/8.0/en/temptable-storage-engine.html)
- MySQL Source Code: `storage/temptable/` directory