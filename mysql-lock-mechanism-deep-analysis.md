# MySQL 8.0.41 锁机制深度分析：MDL 元数据锁 & THR_LOCK 表级锁

> 源码路径：`D:\codebase\mysql-server`  
> 文档版本：基于 MySQL 8.0.41  
> 适用场景：开发 MDL / THR_LOCK 相关新特性的开发者参考

---

## 目录

1. [锁体系总览](#1-锁体系总览)
2. [MDL 元数据锁（Metadata Locking）](#2-mdl-元数据锁)
   - [2.1 核心文件](#21-核心文件)
   - [2.2 MDL 命名空间（enum_mdl_namespace）](#22-mdl-命名空间)
   - [2.3 MDL 锁类型（enum_mdl_type）](#23-mdl-锁类型)
   - [2.4 MDL 锁持续时间（enum_mdl_duration）](#24-mdl-锁持续时间)
   - [2.5 核心数据结构](#25-核心数据结构)
   - [2.6 MDL_lock 内部结构](#26-mdl_lock-内部结构)
   - [2.7 兼容性矩阵（Compatibility Matrix）](#27-兼容性矩阵)
   - [2.8 快速路径与慢速路径（Fast/Slow Path）](#28-快速路径与慢速路径)
   - [2.9 锁获取流程](#29-锁获取流程)
   - [2.10 锁释放流程](#210-锁释放流程)
   - [2.11 锁升级与降级](#211-锁升级与降级)
   - [2.12 MDL 等待机制（MDL_wait）](#212-mdl-等待机制)
   - [2.13 死锁检测](#213-死锁检测)
   - [2.14 防饥饿机制（Piglet & Hog）](#214-防饥饿机制)
   - [2.15 MDL 与事务集成](#215-mdl-与事务集成)
   - [2.16 MDL 全局哈希表（MDL_map）](#216-mdl-全局哈希表)
3. [THR_LOCK 表级锁](#3-thr_lock-表级锁)
   - [3.1 核心文件](#31-核心文件)
   - [3.2 锁类型枚举（thr_lock_type）](#32-锁类型枚举)
   - [3.3 核心数据结构](#33-核心数据结构)
   - [3.4 锁初始化](#34-锁初始化)
   - [3.5 thr_lock() 加锁逻辑](#35-thr_lock-加锁逻辑)
   - [3.6 wait_for_lock() 等待机制](#36-wait_for_lock-等待机制)
   - [3.7 thr_unlock() 解锁流程](#37-thr_unlock-解锁流程)
   - [3.8 wake_up_waiters() 唤醒机制](#38-wake_up_waiters-唤醒机制)
   - [3.9 多表锁序（thr_multi_lock）](#39-多表锁序-thr_multi_lock)
   - [3.10 优先级控制与防饥饿](#310-优先级控制与防饥饿)
4. [MDL 与 THR_LOCK 的协作关系](#4-mdl-与-thr_lock-的协作关系)
5. [开发新特性要点](#5-开发新特性要点)
6. [关键代码行号索引](#6-关键代码行号索引)

---

## 1. 锁体系总览

MySQL 8.0 有多个层次的锁，最重要的两层是：

```
┌─────────────────────────────────────────────────────────────┐
│  SQL 层                                                       │
│  ┌─────────────────┐    ┌──────────────────────────────────┐ │
│  │   MDL 元数据锁   │    │  THR_LOCK 表级锁（MyISAM等引擎） │ │
│  │  sql/mdl.h/.cc   │    │  include/thr_lock.h              │ │
│  │                  │    │  mysys/thr_lock.cc               │ │
│  └─────────────────┘    └──────────────────────────────────┘ │
│           ↓                            ↓                      │
│  保护表的元数据（DDL/DML并发）     保护表的数据（引擎级）      │
├─────────────────────────────────────────────────────────────┤
│  存储引擎层                                                   │
│  InnoDB 行锁（独立体系）、MyISAM 文件锁等                      │
└─────────────────────────────────────────────────────────────┘
```

**关键关系**：
- 对非临时表，必须先通过 MDL 才能获取 THR_LOCK（见 `sql/lock.cc:199-203`）
- MDL 在 `MDL_context`（每连接一个）中管理；THR_LOCK 在 `THR_LOCK`（每表一个）中管理
- InnoDB 不使用 THR_LOCK，完全依赖 MDL + 自身行锁

---

## 2. MDL 元数据锁（Metadata Locking）

### 2.1 核心文件

| 文件 | 路径 | 说明 |
|------|------|------|
| 头文件 | `sql/mdl.h` | 公开 API、数据结构定义 |
| 实现文件 | `sql/mdl.cc` | 完整实现，含 MDL_lock 类定义 |
| SQL 层调用 | `sql/sql_base.cc` | `open_table()`、`lock_tables()` 中调用 MDL |
| 事务层调用 | `sql/sql_class.h` | `THD::mdl_context` 成员 |

### 2.2 MDL 命名空间

定义在 `sql/mdl.h:400-421`，存储于 `MDL_key` 的第一个字节：

```cpp
// sql/mdl.h:400
enum enum_mdl_namespace {
  GLOBAL = 0,         // 全局读锁（FLUSH TABLES WITH READ LOCK）
  BACKUP_LOCK,        // 阻止导致备份不一致的操作（如 DDL）
  TABLESPACE,         // 表空间级别
  SCHEMA,             // Schema（数据库）级别
  TABLE,              // 表和视图（最常用）
  FUNCTION,           // 存储函数
  PROCEDURE,          // 存储过程
  TRIGGER,            // 触发器
  EVENT,              // 事件调度器事件
  COMMIT,             // 全局读锁期间阻塞 COMMIT
  USER_LEVEL_LOCK,    // GET_LOCK() 用户自定义锁（连接断开时中止等待）
  LOCKING_SERVICE,    // 名称插件 RW 锁服务
  SRID,               // 空间参考系统
  ACL_CACHE,          // ACL 缓存（不受 max_write_lock_count 控制）
  COLUMN_STATISTICS,  // 列统计信息（直方图）
  RESOURCE_GROUPS,    // 资源组
  FOREIGN_KEY,        // 外键名称
  CHECK_CONSTRAINT,   // 检查约束名称
  NAMESPACE_END
};
```

**策略分类**（`sql/mdl.cc:819-833`）：

| 策略 | 命名空间 | 说明 |
|------|---------|------|
| `m_scoped_lock_strategy` | GLOBAL, TABLESPACE, SCHEMA, COMMIT, BACKUP_LOCK, RESOURCE_GROUPS, FOREIGN_KEY, CHECK_CONSTRAINT | 仅支持 IX / S / X 三种类型 |
| `m_object_lock_strategy` | TABLE, FUNCTION, PROCEDURE, TRIGGER, EVENT, USER_LEVEL_LOCK 等 | 支持全部 10 种锁类型 |

### 2.3 MDL 锁类型

定义在 `sql/mdl.h:196-329`：

| 枚举值 | 简写 | 值 | 使用场景 |
|--------|------|-----|---------|
| `MDL_INTENTION_EXCLUSIVE` | IX | 0 | 仅用于 Scoped 锁；schema/global 下保护，查询 DD cache 时也使用 |
| `MDL_SHARED` | S | 1 | 仅读元数据（存储过程、PREPARE 语句、HANDLER OPEN） |
| `MDL_SHARED_HIGH_PRIO` | SH | 2 | 高优先级共享锁；INFORMATION_SCHEMA 查询；可忽略 pending X 锁 |
| `MDL_SHARED_READ` | SR | 3 | SELECT / LOCK TABLE READ；有意图读数据 |
| `MDL_SHARED_WRITE` | SW | 4 | INSERT / UPDATE / DELETE / SELECT FOR UPDATE |
| `MDL_SHARED_WRITE_LOW_PRIO` | SWLP | 5 | LOW_PRIORITY DML |
| `MDL_SHARED_UPGRADABLE` | SU | 6 | ALTER TABLE 第一阶段（可升级到 SNW/SNRW/X） |
| `MDL_SHARED_READ_ONLY` | SRO | 7 | LOCK TABLE READ（阻塞所有修改） |
| `MDL_SHARED_NO_WRITE` | SNW | 8 | ALTER TABLE 复制数据阶段（允许并发读） |
| `MDL_SHARED_NO_READ_WRITE` | SNRW | 9 | LOCK TABLE WRITE（只允许 S 和 SH） |
| `MDL_EXCLUSIVE` | X | 10 | CREATE/DROP/RENAME TABLE、DDL 执行阶段 |
| `MDL_TYPE_END` | — | 11 | 哨兵 |

**锁强度排序**（强度递增）：IX < S ≈ SH < SR < SW ≈ SWLP < SU ≈ SRO < SNW < SNRW < X

### 2.4 MDL 锁持续时间

定义在 `sql/mdl.h:333-351`：

| 枚举值 | 说明 | 释放时机 |
|--------|------|---------|
| `MDL_STATEMENT` | 语句级别 | 语句结束或事务结束时自动释放 |
| `MDL_TRANSACTION` | 事务级别 | 事务结束时自动释放（Commit/Rollback） |
| `MDL_EXPLICIT` | 显式级别 | 必须调用 `MDL_context::release_lock()` 手动释放 |

`MDL_ticket_store` 按三种 duration 分别维护链表（见 `sql/mdl.h:1115-1307`）。

### 2.5 核心数据结构

#### MDL_key（`sql/mdl.h:365-788`）

锁的唯一标识键，最大长度 `MAX_MDLKEY_LENGTH = 1 + NAME_LEN + 1 + NAME_LEN + 1`。

```
内存布局：
┌──────────┬──────────────┬──┬─────────────┬──┐
│ 1字节    │ db_name      │\0│ object_name │\0│
│namespace │ (≤NAME_LEN)  │  │ (≤NAME_LEN) │  │
└──────────┴──────────────┴──┴─────────────┴──┘
```

私有成员（`sql/mdl.h:783-788`）：
```cpp
uint16 m_length{0};              // 总字节数
uint16 m_db_name_length{0};      // db_name 字节数
uint16 m_object_name_length{0};  // object_name 字节数
char   m_ptr[MAX_MDLKEY_LENGTH]; // 存储区
static PSI_stage_info m_namespace_to_wait_state_name[NAMESPACE_END];
```

#### MDL_request（`sql/mdl.h:801-820`）

调用者构造的锁请求，传给 `MDL_context::acquire_lock()`：

```cpp
// sql/mdl.h:801
class MDL_request {
 public:
  enum_mdl_type     type{MDL_INTENTION_EXCLUSIVE};  // 请求锁类型
  enum_mdl_duration duration{MDL_STATEMENT};         // 持续时间
  MDL_request      *next_in_list{nullptr};           // 请求链表（批量操作用）
  MDL_request     **prev_in_list{nullptr};
  MDL_ticket       *ticket{nullptr};                 // 授予后非空，指向 ticket
  MDL_key           key;                             // 锁标识键
};
```

#### MDL_ticket（`sql/mdl.h:~990-1103`）

表示"已授予"或"等待中"的锁实例（一个 ticket 对应一次锁获取）：

```cpp
// 主要私有字段（sql/mdl.h 约 1068-1103）
MDL_ticket *next_in_context;    // 在 MDL_context 的 ticket 链表中
MDL_ticket **prev_in_context;
MDL_ticket *next_in_lock;       // 在 MDL_lock 的 granted/waiting 链表中
MDL_ticket **prev_in_lock;
enum_mdl_type   m_type;         // 锁类型
MDL_context    *m_ctx;          // 所属 context（哪个连接）
MDL_lock       *m_lock;         // 所属 MDL_lock（哪个对象）
bool            m_is_fast_path; // 是否通过 fast path 获取
bool            m_hton_notified;// 是否已通知存储引擎
PSI_metadata_lock *m_psi;       // 性能监控接口
```

关键方法：
- `is_upgradable_or_exclusive()` — 判断是否可升级/独占锁
- `downgrade_lock(enum_mdl_type)` — 降级（如 X→SNW）
- `get_deadlock_weight()` — 返回死锁检测权重（DML=25, DDL=100）

#### MDL_context（`sql/mdl.h:1411-1705`）

每个连接 (`THD`) 一个实例，是 MDL 操作的入口：

```cpp
// sql/mdl.h:1411（精简）
class MDL_context {
 public:
  // --- 核心 API ---
  bool try_acquire_lock(MDL_request *mdl_request);
  bool acquire_lock(MDL_request *mdl_request, Timeout_type lock_wait_timeout);
  bool acquire_locks(MDL_request_list *requests, Timeout_type lock_wait_timeout);
  bool upgrade_shared_lock(MDL_ticket *mdl_ticket, enum_mdl_type new_type,
                           Timeout_type lock_wait_timeout);
  void release_lock(MDL_ticket *ticket);
  void release_statement_locks();
  void release_transactional_locks();
  void rollback_to_savepoint(const MDL_savepoint &mdl_savepoint);

  // --- 核心字段 ---
  MDL_wait          m_wait;             // 当前等待槽（条件变量包装）
 private:
  MDL_ticket_store  m_ticket_store;     // 按 duration 分类的 ticket 存储
  MDL_context_owner *m_owner;           // 指向 THD
  bool              m_needs_thr_lock_abort; // 是否需要中止 THR_LOCK 等待
  bool              m_force_dml_deadlock_weight;
  mysql_prlock_t    m_LOCK_waiting_for; // 保护 m_waiting_for
  MDL_wait_for_subgraph *m_waiting_for; // 当前正在等待的 ticket（死锁检测用）
  LF_PINS          *m_pins;            // LF_HASH 的 hazard pointer
  uint              m_rand_state;       // 死锁检测随机状态
};
```

### 2.6 MDL_lock 内部结构

`MDL_lock` 完整定义在 `sql/mdl.cc:427-1051`（非头文件），每个被锁对象对应一个实例：

```cpp
// sql/mdl.cc:427（精简）
class MDL_lock {
 public:
  typedef unsigned short bitmap_t;       // 锁类型位图类型
  typedef longlong fast_path_state_t;    // 快速路径状态类型

  // --- 策略结构体（控制兼容性矩阵和优化路径）---
  struct MDL_lock_strategy {
    bitmap_t m_granted_incompatible[MDL_TYPE_END];      // 已授予不兼容位图
    bitmap_t m_waiting_incompatible[4][MDL_TYPE_END];   // 等待不兼容位图（4套优先级）
    fast_path_state_t m_unobtrusive_lock_increment[MDL_TYPE_END]; // 快速路径增量
    bool m_is_affected_by_max_write_lock_count;         // 是否受最大写锁计数影响
    bool (*m_needs_notification)(const MDL_ticket *);
    void (*m_notify_conflicting_locks)(MDL_context *, MDL_lock *);
    bitmap_t (*m_fast_path_granted_bitmap)(const MDL_lock &);
    bool (*m_needs_connection_check)(const MDL_lock *);
  };

 public:
  MDL_key     key;           // 所保护对象的键
  mysql_prlock_t m_rwlock;   // 读写锁（保护本 MDL_lock 下所有队列）

  // --- 锁队列 ---
  Ticket_list m_granted;     // 已授予的 ticket 链表（含类型位图）
  Ticket_list m_waiting;     // 等待中的 ticket 链表（含类型位图）

  // --- 防饥饿计数 ---
  ulong m_hog_lock_count;    // 连续授予 hog 锁（X/SNRW/SNW）次数
  ulong m_piglet_lock_count; // 连续授予 piglet 锁（SW）次数
  uint  m_current_waiting_incompatible_idx; // 当前使用哪套优先级矩阵（0-3）

  // --- 快速路径状态（原子变量）---
  uint m_obtrusive_locks_granted_waiting_count; // obtrusive 锁计数
  static const fast_path_state_t IS_DESTROYED = 1ULL << 62; // 对象已销毁标志
  static const fast_path_state_t HAS_OBTRUSIVE = 1ULL << 61; // 有 obtrusive 锁
  static const fast_path_state_t HAS_SLOW_PATH = 1ULL << 60; // 有 slow path 锁
  std::atomic<fast_path_state_t> m_fast_path_state; // 打包的快速路径状态

 private:
  const MDL_lock_strategy *m_strategy; // 指向 scoped 或 object 策略实例
};
```

**静态策略实例**（`sql/mdl.cc`）：
- `MDL_lock::m_scoped_lock_strategy`（2075 行）— 用于 GLOBAL/SCHEMA 等
- `MDL_lock::m_object_lock_strategy`（2185 行）— 用于 TABLE 等

### 2.7 兼容性矩阵

#### Object 锁兼容性矩阵（已授予，`sql/mdl.cc:2194-2239`）

矩阵含义：行=请求类型，列=已授予类型，`+`=可兼容，`-`=不兼容需等待

```
         | 已授予 →  S   SH   SR   SW  SWLP  SU  SRO  SNW  SNRW   X
  请求 ↓ |
  S      |           +    +    +    +    +    +    +    +    +     -
  SH     |           +    +    +    +    +    +    +    +    +     -
  SR     |           +    +    +    +    +    +    +    +    -     -
  SW     |           +    +    +    +    +    +    -    -    -     -
  SWLP   |           +    +    +    +    +    +    -    -    -     -
  SU     |           +    +    +    +    +    -    +    -    -     -
  SRO    |           +    +    +    -    -    +    +    +    -     -
  SNW    |           +    +    +    -    -    -    +    -    -     -
  SNRW   |           +    +    -    -    -    -    -    -    -     -
  X      |           -    -    -    -    -    -    -    -    -     -
```

实现方式（`sql/mdl.cc:2218-2239`）：每行存储为位图，`MDL_BIT(type)` 表示该位被置 1 即不兼容：

```cpp
// MDL_BIT 宏定义（sql/mdl.cc:414）
#define MDL_BIT(A) static_cast<MDL_lock::bitmap_t>(1U << A)

// 例：SW (index 4) 的 m_granted_incompatible：
// 不兼容：SRO(7), SNW(8), SNRW(9), X(10)
MDL_BIT(MDL_EXCLUSIVE) | MDL_BIT(MDL_SHARED_NO_READ_WRITE) |
    MDL_BIT(MDL_SHARED_NO_WRITE) | MDL_BIT(MDL_SHARED_READ_ONLY)
```

#### Scoped 锁兼容性矩阵（`sql/mdl.cc:2086-2115`）

仅涉及 IX、S、X 三种类型（IS 不计入）：

```
         | 已授予 →  IS(*)  IX    S    X
  请求 ↓ |
  IS     |           +      +    +    +
  IX     |           +      +    -    -
  S      |           +      -    +    -
  X      |           +      -    -    -
```

#### 兼容性检查核心逻辑（`sql/mdl.cc:2396-2411`）

```cpp
bool MDL_lock::can_grant_lock(enum_mdl_type type_arg,
                              const MDL_context *requestor_ctx) const {
  bitmap_t waiting_incompat = incompatible_waiting_types_bitmap()[type_arg];
  bitmap_t granted_incompat  = incompatible_granted_types_bitmap()[type_arg];

  // 三重检查：等待队列无冲突 AND fast path 授予无冲突 AND slow path 授予无冲突
  if (!(m_waiting.bitmap() & waiting_incompat)) {
    if (!(fast_path_granted_bitmap() & granted_incompat)) {
      if (!(m_granted.bitmap() & granted_incompat))
        return true;  // 可以授予
    }
  }
  // ... 同 context 已有更强锁的特殊处理
}
```

### 2.8 快速路径与慢速路径

**设计目标**：减少公共锁临界区，提升高并发 DML 的可扩展性。

#### 分类（`sql/mdl.cc:2343-2364`）

**对象锁（Object Namespace）**：

| 类型 | 路径 | fast_path_state 编码 |
|------|------|---------------------|
| S, SH | fast path | bits 0–19（共用计数） |
| SR | fast path | bits 20–39 |
| SW, SWLP | fast path | bits 40–59（共用计数） |
| SU, SRO, SNW, SNRW, X | slow path | — |

**Scoped 锁**：
| 类型 | 路径 |
|------|------|
| IX | fast path（bits 0–59） |
| S, X | slow path |

#### Fast Path 加锁流程

1. 计算 increment = `m_strategy->m_unobtrusive_lock_increment[type]`（非零表示 unobtrusive）
2. 原子读 `m_fast_path_state`
3. 若 `HAS_OBTRUSIVE` 未设置，CAS 增加计数 → **成功，无需获取 m_rwlock**
4. 若 CAS 失败或有 obtrusive 锁，回退到 slow path

#### Slow Path 加锁流程

1. `mysql_prlock_wrlock(&m_rwlock)`（写锁保护）
2. 调用 `can_grant_lock()`
3. 若可授予：插入 `m_granted`，更新 `m_fast_path_state` 中的标志位
4. 若不可授予：插入 `m_waiting`，释放 m_rwlock，进入 `MDL_wait::timed_wait()`

**不变量 [INV1]**：当 `HAS_OBTRUSIVE` 标志已置位，所有对 `m_fast_path_state` 的修改必须在 `m_rwlock` 保护下进行。

### 2.9 锁获取流程

#### try_acquire_lock（无等待，`sql/mdl.cc:2685-2808`）

```
MDL_context::try_acquire_lock(request)
  └─ try_acquire_lock_impl()
       ├─ find_ticket()   如果已有更强锁，复用 ticket
       ├─ mdl_locks.find_or_insert(key)  获取/创建 MDL_lock 对象
       ├─ fast path CAS  原子增计数（无锁）
       └─ slow path      wrlock + can_grant_lock + 插入 m_granted
```

#### acquire_lock（可等待，`sql/mdl.cc:3359-3591`）

```
MDL_context::acquire_lock(request, timeout)
  ├─ try_acquire_lock_impl()  尝试立即获取
  │    └─ 成功 → 返回
  ├─ 创建 ticket 并加入 m_waiting 队列
  ├─ will_wait_for(ticket)   设置 m_waiting_for（死锁检测用）
  ├─ find_deadlock()         死锁检测（DFS）
  │    └─ 若发现死锁 → victim->m_wait.set_status(VICTIM)
  ├─ m_wait.timed_wait()     条件变量等待
  │    └─ 被唤醒原因：GRANTED / VICTIM / TIMEOUT / KILLED
  └─ 处理唤醒结果
```

#### acquire_locks（批量，`sql/mdl.cc:3638-3689`）

对请求列表排序后，逐个调用 `acquire_lock()`。

### 2.10 锁释放流程

**单个释放**（`sql/mdl.cc:4095-4204`）：

```
MDL_context::release_lock(ticket)
  ├─ fast path: 原子减 m_fast_path_state
  └─ slow path: wrlock + m_granted.remove_ticket()
                └─ MDL_lock::reschedule_waiters()  唤醒等待者
```

**批量释放**：
- `release_statement_locks()`  — 释放 `MDL_STATEMENT` duration 的锁
- `release_transactional_locks()` — 释放 `MDL_TRANSACTION` duration 的锁
- `rollback_to_savepoint()` — 回滚到 savepoint，释放该 savepoint 之后的锁

**reschedule_waiters**（`sql/mdl.cc:2556`）：

遍历 `m_waiting` 队列，对每个 waiting ticket 重新检查 `can_grant_lock()`，可授予则移到 `m_granted` 并 `m_wait.set_status(GRANTED)` + signal。

### 2.11 锁升级与降级

#### 升级（`sql/mdl.cc:3745-3849`）

```cpp
bool MDL_context::upgrade_shared_lock(MDL_ticket *ticket,
                                      enum_mdl_type new_type,
                                      Timeout_type timeout) {
  // 如果已有目标类型或更强类型，直接返回
  if (ticket->has_stronger_or_equal_type(new_type)) return false;

  // 创建新的 MDL_request，以 new_type 请求同一 key
  MDL_request mdl_new_lock_request;
  MDL_REQUEST_INIT_BY_KEY(&mdl_new_lock_request, &ticket->m_lock->key,
                          new_type, MDL_TRANSACTION);
  // 通过正常路径获取（可等待）
  if (acquire_lock(&mdl_new_lock_request, timeout)) return true;

  // 合并：更新原 ticket 类型为 new_type，释放新 ticket
  // （在 m_rwlock 保护下）
}
```

典型升级路径：`SU → SNW → X`（ALTER TABLE 三阶段）

#### 降级（`sql/mdl.cc:~4296`）

```cpp
void MDL_ticket::downgrade_lock(enum_mdl_type new_type) {
  // 在 m_rwlock 写锁下：
  // 1. 若原来是 obtrusive 类型，检查是否降为 fast path 类型
  // 2. 更新 m_type
  // 3. 调整 m_fast_path_state 标志
  // 4. reschedule_waiters()  允许之前被阻塞的请求前进
}
```

### 2.12 MDL 等待机制

`MDL_wait` 是对条件变量的包装（`sql/mdl.h:1343-1370`）：

```cpp
class MDL_wait {
 public:
  enum enum_wait_status {
    WS_EMPTY = 0,  // 初始状态
    GRANTED,       // 锁已授予
    VICTIM,        // 被死锁检测选为 victim
    TIMEOUT,       // 等待超时
    KILLED         // 连接被杀死
  };

  bool set_status(enum_wait_status result_arg);
  enum_wait_status get_status();
  void reset_status();
  enum_wait_status timed_wait(MDL_context_owner *owner,
                              struct timespec *abs_timeout,
                              bool signal_timeout,
                              const PSI_stage_info *wait_state_name);
 private:
  mysql_mutex_t    m_LOCK_wait_status;
  mysql_cond_t     m_COND_wait_status;
  enum_wait_status m_wait_status;
};
```

`timed_wait` 核心实现（`sql/mdl.cc:1807-1848`）：

```cpp
// 条件变量循环：退出条件为 status 非空 OR 连接被杀 OR 超时
while (!m_wait_status && !owner->is_killed() && !is_timeout(wait_result)) {
    wait_result = mysql_cond_timedwait(&m_COND_wait_status,
                                       &m_LOCK_wait_status, abs_timeout);
}
// 根据退出原因设置最终状态
if (m_wait_status == WS_EMPTY) {
    if (owner->is_killed())        m_wait_status = KILLED;
    else if (set_status_on_timeout) m_wait_status = TIMEOUT;
}
```

### 2.13 死锁检测

#### 检测入口（`sql/mdl.cc:4044-4084`）

```cpp
void MDL_context::find_deadlock() {
  while (true) {
    Deadlock_detection_visitor dvisitor(this);
    if (!visit_subgraph(&dvisitor)) break;  // 无死锁

    MDL_context *victim = dvisitor.get_victim();
    victim->m_wait.set_status(MDL_wait::VICTIM);  // 唤醒 victim
    victim->unlock_deadlock_victim();
    if (victim == this) break;
    // 否则重新扫描（防止检测遗漏）
  }
}
```

#### 访问者类（`sql/mdl.cc:291-406`）

```cpp
class Deadlock_detection_visitor : public MDL_wait_for_graph_visitor {
  MDL_context *m_start_node;  // 检测起点（自身）
  MDL_context *m_victim;      // 当前选出的 victim
  uint m_current_search_depth;
  bool m_found_deadlock;
  static const uint MAX_SEARCH_DEPTH = 32;  // 最大递归深度
};
```

关键回调：
- `enter_node(ctx)`: 若深度 ≥ 32，强制报告死锁（保守策略）
- `inspect_edge(ctx)`: 若回到起点 `m_start_node`，发现环
- `opt_change_victim_to(ctx)`: 按 `get_deadlock_weight()` 选择权重较小的 victim

#### 等待图遍历（`sql/mdl.cc:3861-3991`）

`MDL_lock::visit_subgraph(waiting_ticket, gvisitor)`：
1. 读锁 `m_rwlock`
2. 遍历 `m_granted` 中与 `waiting_ticket` 类型不兼容的 ticket
3. 对每个这样的 ticket，调用其所属 `MDL_context` 的 `visit_subgraph()`
4. `MDL_context::visit_subgraph()` 沿 `m_waiting_for` 指针展开（`sql/mdl.cc:4024-4033`）

#### 死锁权重（`sql/mdl.cc:1698-1741`）

```cpp
uint MDL_ticket::get_deadlock_weight() const {
  // 持有 GLOBAL 锁或升级锁（≥ MDL_SHARED_UPGRADABLE）= DDL 权重
  if (m_lock->key.mdl_namespace() == MDL_key::GLOBAL ||
      m_type >= MDL_SHARED_UPGRADABLE)
    return DEADLOCK_WEIGHT_DDL;  // 100
  return DEADLOCK_WEIGHT_DML;    // 25
}
```

**常量**（`sql/mdl.h:955-957`）：

| 常量 | 值 | 说明 |
|------|-----|------|
| `DEADLOCK_WEIGHT_DML` | 25 | DML 事务权重（优先选为 victim） |
| `DEADLOCK_WEIGHT_DDL` | 100 | DDL 语句权重（尽量不选为 victim） |
| `DEADLOCK_WEIGHT_ULL` | 50 | 用户级锁权重 |

**m_rwlock 读优先的必要性**（`sql/mdl.cc:536-567`）：

死锁检测过程中同时持有多个 `MDL_lock::m_rwlock` 读锁，若互斥锁偏向写等待（fair/write-prefer），则两个并发死锁检测线程加上两个请求写锁的线程可能互相阻塞，导致检测器本身死锁。读优先确保检测器不被阻塞。

### 2.14 防饥饿机制

`Piglet`（SW = 写入并发低优先级）和 `Hog`（SNW/SNRW/X = 高优先级写）连续授予次数超过 `max_write_lock_count` 时，切换到更公平的等待优先级矩阵（`sql/mdl.cc:2240-2341`）：

**四套优先级矩阵**（`MDL_lock_strategy::m_waiting_incompatible[0..3]`）：

| 索引 | 触发条件 | 说明 |
|------|---------|------|
| 0 | 默认 | 标准优先级：Hog/Piglet 优先 |
| 1 | `m_piglet_lock_count >= max_write_lock_count` | SW 优先级降低（SRO 优先于 SW） |
| 2 | `m_hog_lock_count >= max_write_lock_count` | Hog 优先级降低（非 Hog 类型优先） |
| 3 | 两者均超限 | SRO 优先于 SW/SWLP，非 Hog 优先于 Hog |

计数和切换逻辑（`sql/mdl.cc:672-687`）：

```cpp
bool MDL_lock::count_piglets_and_hogs(enum_mdl_type type) {
  if (MDL_BIT(type) & MDL_OBJECT_HOG_LOCK_TYPES) {  // X, SNRW, SNW
    if (m_waiting.bitmap() & ~MDL_OBJECT_HOG_LOCK_TYPES) {
      m_hog_lock_count++;
      if (switch_incompatible_waiting_types_bitmap_if_needed()) return true;
    }
  } else if (type == MDL_SHARED_WRITE) {
    if (m_waiting.bitmap() & MDL_BIT(MDL_SHARED_READ_ONLY)) {
      m_piglet_lock_count++;
      if (switch_incompatible_waiting_types_bitmap_if_needed()) return true;
    }
  }
  return false;
}
```

### 2.15 MDL 与事务集成

`THD` 继承 `MDL_context_owner` 并持有 `MDL_context` 成员（`sql/sql_class.h:928-946`）：

```cpp
class THD : public MDL_context_owner,
            public Query_arena,
            public Open_tables_state {
  // ...
  MDL_context mdl_context;  // 每个连接独立的 MDL 上下文
};
```

**典型 DML 调用链**：

```
mysql_execute_command()
  └─ open_tables()                          [sql/sql_base.cc]
       └─ open_table()
            └─ thd->mdl_context.try_acquire_lock(&mdl_request)
                                  [MDL_SHARED_READ 或 MDL_SHARED_WRITE]
  └─ mysql_lock_tables()                    [sql/lock.cc]
       └─ thr_multi_lock()                  [mysys/thr_lock.cc]
  └─ ha_*() 存储引擎操作
  └─ trans_commit_stmt()
       └─ thd->mdl_context.release_statement_locks()
```

**典型 DDL 调用链**（ALTER TABLE）：

```
mysql_alter_table()
  ├─ MDL_SHARED_UPGRADABLE (SU)      第一阶段开始
  ├─ upgrade_shared_lock(SNW)        准备提交
  ├─ upgrade_shared_lock(X)          执行元数据修改
  ├─ downgrade_lock(SNW)             复制数据阶段降级（允许读）
  └─ release_lock()                  完成
```

### 2.16 MDL 全局哈希表

`MDL_map`（单例 `mdl_locks`，`sql/mdl.cc:1053`）用 Lock-Free 哈希表存储所有活跃的 `MDL_lock` 对象：

```cpp
// sql/mdl.cc:158-272（精简）
class MDL_map {
  LF_HASH m_locks;                   // Lock-Free 哈希表（主存储）
  MDL_lock *m_global_lock;           // GLOBAL 预分配实例
  MDL_lock *m_commit_lock;           // COMMIT 预分配实例
  MDL_lock *m_acl_cache_lock;        // ACL_CACHE 预分配实例
  MDL_lock *m_backup_lock;           // BACKUP_LOCK 预分配实例
};
```

关键方法：
- `find(pins, key, pinned)` — 单例 namespace 直接返回预分配指针；否则 `lf_hash_search()`
- `find_or_insert(pins, key, pinned)` — 查找或插入（`sql/mdl.cc:1250`）
- `remove(pins, lock)` — 引用计数为零时删除

初始化（`sql/mdl.cc:1134-1149`）：

```cpp
void MDL_map::init() {
  lf_hash_init2(&m_locks, sizeof(MDL_lock), LF_HASH_UNIQUE, ...);
  // 预分配常用 singleton 锁对象
  m_global_lock   = MDL_lock::create(&key_GLOBAL);
  m_commit_lock   = MDL_lock::create(&key_COMMIT);
  m_acl_cache_lock= MDL_lock::create(&key_ACL_CACHE);
  m_backup_lock   = MDL_lock::create(&key_BACKUP_LOCK);
}
```

---

## 3. THR_LOCK 表级锁

### 3.1 核心文件

| 文件 | 路径 | 说明 |
|------|------|------|
| 头文件 | `include/thr_lock.h` | 数据结构和 API 声明 |
| 实现 | `mysys/thr_lock.cc` | 完整实现 |
| SQL 层调用 | `sql/lock.cc` | `mysql_lock_tables()` → `thr_multi_lock()` |

**注意**：THR_LOCK 主要用于 MyISAM、MEMORY 等不支持行锁的存储引擎。InnoDB 不使用 THR_LOCK（但仍需先获取 MDL）。

### 3.2 锁类型枚举

定义在 `include/thr_lock.h:51-95`，**枚举值顺序直接影响兼容性判断逻辑**：

```cpp
enum thr_lock_type {
  TL_IGNORE = -1,          // 忽略（内部使用）
  TL_UNLOCK,               // 已解锁/释放状态
  TL_READ_DEFAULT,         // 解析器占位，open_tables 时转换为实际类型
  TL_READ,                 // 普通读锁（允许并发插入）
  TL_READ_WITH_SHARED_LOCKS,   // 带共享锁的读（SELECT ... LOCK IN SHARE MODE）
  TL_READ_HIGH_PRIORITY,   // 高优先级读（可忽略等待中的写锁）
  TL_READ_NO_INSERT,       // 不允许并发插入的读（LOCK TABLE READ）
  TL_WRITE_ALLOW_WRITE,    // 允许其他写入的写锁（BDB 遗留）
  TL_WRITE_CONCURRENT_DEFAULT, // 解析器占位
  TL_WRITE_CONCURRENT_INSERT,  // 并发插入写锁（MyISAM 追加插入）
  TL_WRITE_DEFAULT,        // 解析器占位
  TL_WRITE_LOW_PRIORITY,   // 低优先级写（优先级低于读）
  TL_WRITE,                // 普通写锁（高优先级）
  TL_WRITE_ONLY            // 最高优先级写；拒绝所有新锁请求（DDL使用）
};
```

**读/写分界**：`(int)lock_type <= (int)TL_READ_NO_INSERT` 为读锁请求。

**兼容性大致规则**（枚举序数越小越弱）：
- `TL_WRITE_ALLOW_WRITE` 允许多个并发写
- `TL_WRITE_CONCURRENT_INSERT` 允许并发读（`TL_READ` 和 `TL_READ_HIGH_PRIORITY`）
- `TL_WRITE` / `TL_WRITE_LOW_PRIORITY` 排他（阻塞所有读写）
- `TL_WRITE_ONLY` 拒绝所有新请求（DDL 执行时使用）

### 3.3 核心数据结构

#### THR_LOCK_INFO（`include/thr_lock.h:119-122`）

标识持锁/等待的线程：

```cpp
struct THR_LOCK_INFO {
  my_thread_id thread_id;   // 线程 ID（用于同线程锁兼容检测）
  mysql_cond_t *suspend;    // 条件变量指针（等待时的唤醒手柄）
};
```

初始化（`mysys/thr_lock.cc:330-334`）：

```cpp
void thr_lock_info_init(THR_LOCK_INFO *info, my_thread_id thread_id,
                        mysql_cond_t *suspend) {
  info->thread_id = thread_id;
  info->suspend = suspend;
}
```

#### THR_LOCK_DATA（`include/thr_lock.h:124-133`）

单个锁请求/持锁实例，是队列中的结点：

```cpp
struct THR_LOCK_DATA {
  THR_LOCK_INFO *owner{nullptr};         // 持有者线程信息
  THR_LOCK_DATA *next{nullptr};          // 链表下一结点
  THR_LOCK_DATA **prev{nullptr};         // 链表前驱指针的地址（O(1) 删除）
  THR_LOCK      *lock{nullptr};          // 所属 THR_LOCK（哪张表）
  mysql_cond_t  *cond{nullptr};          // 等待时非空（= owner->suspend）
  thr_lock_type  type{TL_IGNORE};        // 锁类型（TL_UNLOCK 表示未持锁）
  void          *status_param{nullptr};  // 引擎状态回调参数（MyISAM 使用）
  void          *debug_print_param{nullptr};
  struct PSI_table *m_psi{nullptr};      // PSI 监控句柄
};
```

**`cond` 的生命周期**：
- 进入等待：`data->cond = owner->suspend`（`wait_for_lock()` 第 397 行）
- 获得锁：`data->cond = nullptr`（`free_all_read_locks()` 第 723 行，或 `wake_up_waiters()` 中）
- 超时/中止：也置 `nullptr`，加 `type = TL_UNLOCK`

#### st_lock_list（`include/thr_lock.h:135-137`）

队列头结构：

```cpp
struct st_lock_list {
  THR_LOCK_DATA *data{nullptr};   // 队首
  THR_LOCK_DATA **last{nullptr};  // 指向"最后一个结点的 next 字段地址"，用于 O(1) 尾部追加
};
```

尾部插入惯用法（`mysys/thr_lock.cc:538-540`）：

```cpp
(*lock->read.last) = data;   // 将新结点接到链尾
data->prev = lock->read.last;
lock->read.last = &data->next;  // 更新 last 指针
```

#### THR_LOCK（`include/thr_lock.h:139-154`）

表级锁中心结构（每张表一个实例）：

```cpp
struct THR_LOCK {
  LIST          list;              // 全局锁链表结点（thr_lock_thread_list 用）
  mysql_mutex_t mutex;             // 保护本表所有队列的互斥量
  st_lock_list  read_wait;         // 等待中的读请求队列
  st_lock_list  read;              // 已授予的读锁队列
  st_lock_list  write_wait;        // 等待中的写请求队列
  st_lock_list  write;             // 已授予的写锁队列
  ulong  write_lock_count{0};      // 连续写锁计数（防止读饥饿）
  uint   read_no_write_count{0};   // TL_READ_NO_INSERT 锁数量
  // 引擎状态回调（MyISAM 并发插入同步用）：
  void (*get_status)(void *, int){nullptr};   // 获取锁时快照表状态
  void (*copy_status)(void *, void *){nullptr};
  void (*update_status)(void *){nullptr};     // 写锁释放前更新全局状态
  void (*restore_status)(void *){nullptr};    // 读锁释放前恢复状态
  bool (*check_status)(void *){nullptr};      // 检查并发插入是否可行
};
```

### 3.4 锁初始化

```cpp
// mysys/thr_lock.cc:306-320
void thr_lock_init(THR_LOCK *lock) {
  new (lock) THR_LOCK();
  mysql_mutex_init(key_THR_LOCK_mutex, &lock->mutex, MY_MUTEX_INIT_FAST);
  // 初始化四个队列（last 指向自身 data 字段地址 = 空队列）
  lock->read.last       = &lock->read.data;
  lock->read_wait.last  = &lock->read_wait.data;
  lock->write_wait.last = &lock->write_wait.data;
  lock->write.last      = &lock->write.data;
  // 加入全局链表
  mysql_mutex_lock(&THR_LOCK_lock);
  thr_lock_thread_list = list_add(thr_lock_thread_list, &lock->list);
  mysql_mutex_unlock(&THR_LOCK_lock);
}
```

### 3.5 thr_lock() 加锁逻辑

完整实现在 `mysys/thr_lock.cc:476-684`，关键决策流程：

```
thr_lock(data, owner, lock_type, timeout)
  mysql_mutex_lock(&lock->mutex)
  │
  ├─ [读锁路径] if lock_type <= TL_READ_NO_INSERT
  │    ├─ 若有活跃写锁 (lock->write.data != NULL)：
  │    │    ├─ 同线程 OR 写锁类型 < TL_WRITE_CONCURRENT_INSERT
  │    │    │    OR (写=CONCURRENT_INSERT AND 读<=READ_HIGH_PRIORITY)
  │    │    │    → 直接加入 read 队列（允许同线程读写共存）
  │    │    ├─ 写锁=TL_WRITE_ONLY AND 非同线程 → ABORTED
  │    │    └─ 其他情况 → 进入 read_wait 等待
  │    │
  │    └─ 若无活跃写锁：
  │         ├─ write_wait 为空 OR 队首<=LOW_PRIORITY OR 自己=HIGH_PRIO
  │         │    OR 本线程已有读锁 → 直接加入 read 队列
  │         └─ 存在高优先级等待写 → 进入 read_wait 等待（让路）
  │
  └─ [写锁路径] else
       ├─ 若 TL_WRITE_CONCURRENT_INSERT 且引擎不支持 → 升级为 TL_WRITE
       │
       ├─ 若有活跃写锁：
       │    ├─ 写锁=TL_WRITE_ONLY AND 非同线程 → ABORTED
       │    ├─ 同线程 OR (TL_WRITE_ALLOW_WRITE 且无等待写且已有=TL_WRITE_ALLOW_WRITE)
       │    │    → 直接加入 write 队列
       │    └─ 其他 → 进入 write_wait 等待
       │
       └─ 若无活跃写锁：
            ├─ write_wait 为空：
            │    ├─ 无活跃读 OR (类型<=CONCURRENT_INSERT 且无 read_no_write_count)
            │    │    → 直接加入 write 队列
            │    └─ 有活跃读 → 进入 write_wait 等待
            └─ 有等待写 → 进入 write_wait 等待（公平排队）
  │
  └─ 若需等待 → wait_for_lock(wait_queue, ...)
  │
  end: mysql_mutex_unlock(&lock->mutex)
```

**读写兼容性矩阵**（`mysys/thr_lock.cc:508-527`注释中的矩阵）：

```
已持写锁类型 →  WRITE_ALLOW_WRITE  CONCURRENT_INSERT
  读请求 ↓
  READ              +                  +
  READ_WITH_SL      +                  +
  READ_HIGH_PRIORITY+                  +
  READ_NO_INSERT    +                  -

  + = 读可与此写共存（不入等待队列）
  - = 读需等待
```

### 3.6 wait_for_lock() 等待机制

实现在 `mysys/thr_lock.cc:356-474`：

```
wait_for_lock(wait_queue, data, owner, in_wait_list, timeout)
  │
  ├─ 若未入队：将 data 追加到 wait_queue 尾部
  ├─ locks_waited++
  ├─ data->cond = owner->suspend   // 设置等待手柄
  ├─ enter_cond_hook(...)          // PSI 等待状态记录
  ├─ before_lock_wait()            // 可选钩子
  │
  ├─ 循环：mysql_cond_timedwait(data->cond, &lock->mutex, &timeout)
  │    退出条件：
  │    ├─ data->cond == nullptr     → 锁已授予
  │    ├─ is_killed_hook()          → 连接被杀
  │    └─ ETIMEDOUT                 → 超时
  │
  ├─ after_lock_wait()
  │
  ├─ if (data->cond != NULL || data->type == TL_UNLOCK)
  │    → 锁未授予（超时/中止）：从等待队列摘除，wake_up_waiters()
  └─ else
       → 锁已授予：调用 get_status()，返回 THR_LOCK_SUCCESS
  │
  └─ mysql_mutex_unlock(&data->lock->mutex)
```

**重要**：`wait_for_lock()` 在返回前会释放 `lock->mutex`，与 `thr_lock()` 结尾的 `goto end: unlock` 逻辑不同。

### 3.7 thr_unlock() 解锁流程

实现在 `mysys/thr_lock.cc:733-759`：

```cpp
void thr_unlock(THR_LOCK_DATA *data) {
  mysql_mutex_lock(&lock->mutex);

  // 1. 从活跃链表（read 或 write）摘除
  if ((*data->prev) = data->next)
    data->next->prev = data->prev;
  else if (lock_type <= TL_READ_NO_INSERT)
    lock->read.last = data->prev;
  else
    lock->write.last = data->prev;

  // 2. 调用引擎状态回调
  if (lock_type >= TL_WRITE_CONCURRENT_INSERT)
    if (lock->update_status) lock->update_status(data->status_param);
  else
    if (lock->restore_status) lock->restore_status(data->status_param);

  // 3. 维护计数
  if (lock_type == TL_READ_NO_INSERT) lock->read_no_write_count--;

  // 4. 标记为未持锁
  data->type = TL_UNLOCK;

  // 5. 唤醒等待者
  wake_up_waiters(lock);
  mysql_mutex_unlock(&lock->mutex);
}
```

### 3.8 wake_up_waiters() 唤醒机制

实现在 `mysys/thr_lock.cc:769-868`，是调度逻辑的核心：

```
wake_up_waiters(lock)
  │
  ├─ 若无活跃写锁 (lock->write.data == NULL)
  │    ├─ 若无活跃读锁 (lock->read.data == NULL)
  │    │    └─ 优先唤醒写等待：
  │    │         ├─ 若 write_wait 队首类型=TL_WRITE_LOW_PRIORITY
  │    │         │    且 read_wait 有高优先级读 → 先唤醒读
  │    │         ├─ 否则检查 write_lock_count：
  │    │         │    若 > max_write_lock_count → 重置计数，强制释放全部读等待
  │    │         └─ 逐个唤醒 write_wait 队头（signal cond，cond = null）
  │    │              若类型=TL_WRITE_ALLOW_WRITE 则批量唤醒同类型
  │    │              若类型 >= TL_WRITE_LOW_PRIORITY → 同时释放读等待
  │    │
  │    └─ 若有活跃读锁：
  │         若 write_wait 队首类型 <= TL_WRITE_CONCURRENT_INSERT
  │           且无 read_no_write_count 阻塞 → 唤醒写等待（ALLOW_WRITE/CONCURRENT_INSERT）
  │           + 同时释放读等待
  │         若 write_wait 为空 → free_all_read_locks()
  │
  └─ 若有活跃写锁 → 什么都不做（还有锁持有中）
```

#### free_all_read_locks()（`mysys/thr_lock.cc:686-729`）

将 `read_wait` 队列整体并入 `read` 队列，逐个 signal：

- 对 `TL_READ_NO_INSERT` 且当前有并发插入写锁 → 不能一起释放，保留在 `read_wait`
- 其他读锁 → `data->cond = nullptr; mysql_cond_signal(cond)`

### 3.9 多表锁序（thr_multi_lock）

实现在 `mysys/thr_lock.cc:897-921`：

```cpp
enum enum_thr_lock_result thr_multi_lock(THR_LOCK_DATA **data, uint count,
                                         THR_LOCK_INFO *owner,
                                         ulong lock_wait_timeout) {
  if (count > 1) sort_locks(data, count);  // 排序，防死锁

  for (pos = data; pos < data + count; pos++) {
    result = thr_lock(*pos, owner, (*pos)->type, lock_wait_timeout);
    if (result != THR_LOCK_SUCCESS) {
      thr_multi_unlock(data, pos - data);  // 回滚已获取的锁
      return result;
    }
  }
  thr_lock_merge_status(data, count);  // 合并 MyISAM 状态
  return THR_LOCK_SUCCESS;
}
```

**排序规则**（`mysys/thr_lock.cc:876-878`）：

```cpp
#define LOCK_CMP(A, B) \
  ((uchar*)(A->lock) - (uint)((A)->type) < (uchar*)(B->lock) - (uint)((B)->type))
```

含义：先按表地址排序（`lock` 指针值），相同表则写锁（type 值大）排在前面。这确保了**全局一致的加锁顺序**，避免多表死锁。

### 3.10 优先级控制与防饥饿

**优先级顺序**（高到低，`mysys/thr_lock.cc:54-59`）：

```
TL_WRITE_ONLY > TL_WRITE > TL_READ_HIGH_PRIORITY > TL_READ >
TL_WRITE_LOW_PRIORITY > TL_WRITE_CONCURRENT_INSERT > TL_WRITE_ALLOW_WRITE
```

**防止写饥饿读**（`mysys/thr_lock.cc:784-793`）：

```cpp
if (lock->write_lock_count++ > max_write_lock_count) {
  lock->write_lock_count = 0;
  if (lock->read_wait.data) {
    free_all_read_locks(lock, false);  // 强制释放所有等待读
    goto end;
  }
}
```

- `max_write_lock_count`（默认 `~(ulong)0`，即几乎无限）控制连续写锁最大次数
- 超限后重置计数并优先释放所有等待读请求

**`TL_READ_HIGH_PRIORITY` 特殊处理**（`mysys/thr_lock.cc:553-557`）：

```cpp
} else if (!lock->write_wait.data ||
           lock->write_wait.data->type <= TL_WRITE_LOW_PRIORITY ||
           lock_type == TL_READ_HIGH_PRIORITY ||          // ← 高优先读可跳过等待写
           has_old_lock(lock->read.data, data->owner)) {  // ← 同线程已有读锁
  // 直接入 read 活跃队列
```

---

## 4. MDL 与 THR_LOCK 的协作关系

```
应用层 SQL
    │
    ▼
lock_tables_check() → 断言：非临时表必须已有 MDL
    │
    ▼
mysql_lock_tables()                    [sql/lock.cc]
    ├─ external_lock()                  通知引擎（BEGIN）
    └─ thr_multi_lock(
           sql_lock->locks,             THR_LOCK_DATA 数组
           sql_lock->lock_count,
           &thd->lock_info,             THR_LOCK_INFO（线程信息）
           timeout
       )
           │
           └─ sort_locks() + thr_lock() × N
```

**关键映射**（`sql/lock.cc:678-687`）：

`get_lock_data()` 调用各表的 `handler::store_lock()`，将 MDL 类型映射为 THR_LOCK 类型：

| MDL 类型 | 对应 THR_LOCK 类型 |
|---------|-------------------|
| SR（SELECT） | TL_READ |
| SW（INSERT/UPDATE/DELETE） | TL_WRITE_CONCURRENT_DEFAULT → TL_WRITE_CONCURRENT_INSERT 或 TL_WRITE |
| SRO（LOCK TABLE READ） | TL_READ_NO_INSERT |
| SNRW（LOCK TABLE WRITE） | TL_WRITE |
| X（DDL） | TL_WRITE_ONLY（部分情况） |

---

## 5. 开发新特性要点

### 添加新的 MDL 命名空间

1. 在 `sql/mdl.h:MDL_key::enum_mdl_namespace` 末尾（`NAMESPACE_END` 之前）添加枚举值
2. 在 `sql/mdl.cc` 的 `m_namespace_to_wait_state_name[]` 数组中添加对应 `PSI_stage_info`
3. 在 `MDL_lock::get_strategy()` 决策块（819行）中指定使用 scoped 还是 object 策略
4. 若需要特殊通知逻辑，实现 `m_needs_notification`/`m_notify_conflicting_locks` 回调

### 添加新的 MDL 锁类型

1. 在 `sql/mdl.h:enum_mdl_type` 末尾（`MDL_TYPE_END` 之前）添加枚举值
2. 更新 `sql/mdl.cc` 中 `m_object_lock_strategy`（或 `m_scoped_lock_strategy`）的：
   - `m_granted_incompatible[]`：新类型与已有类型的兼容性
   - `m_waiting_incompatible[4][]`：四套优先级矩阵
   - `m_unobtrusive_lock_increment[]`：若为 unobtrusive 类型则设置 fast path 编码
3. 若为 unobtrusive 类型，需确保 fast path 状态编码无重叠（bits 0-59 当前分配）
4. 更新 `MDL_ticket::get_deadlock_weight()` 中的权重判断
5. 若为 hog 类型（high priority exclusive），更新 `MDL_OBJECT_HOG_LOCK_TYPES` 宏

### 修改 THR_LOCK 优先级策略

1. `wait_for_lock()` 已设计为通用；优先级决策完全在 `thr_lock()` 的条件判断中
2. 修改 `wake_up_waiters()` 中的调度逻辑以改变唤醒策略
3. 调整 `max_write_lock_count` 对应的防饥饿逻辑（784-793行）

### 调试技巧

- MDL 调试：设置 `DBUG_TRACE` 和 MDL P_S 表（`performance_schema.metadata_locks`）
- THR_LOCK 调试：启用 `EXTRA_DEBUG` 宏使能 `check_locks()` 一致性检查（`mysys/thr_lock.cc:256-302`）
- 死锁检测：`DEADLOCK_WEIGHT_*` 常量和 `MAX_SEARCH_DEPTH` 可调整敏感度

### 线程安全注意事项

| 操作 | 需要的锁 |
|------|---------|
| MDL fast path 获取/释放 | 无锁（原子 CAS） |
| MDL slow path 获取/释放 | `MDL_lock::m_rwlock`（写锁） |
| 死锁检测遍历 | `MDL_lock::m_rwlock`（读锁，优先于写等待） |
| MDL_wait::set_status | `MDL_wait::m_LOCK_wait_status` |
| THR_LOCK 所有操作 | `THR_LOCK::mutex` |
| 全局锁列表 | `THR_LOCK_lock`（全局 mutex） |

---

## 6. 关键代码行号索引

### MDL（`sql/mdl.h` 和 `sql/mdl.cc`）

| 符号 | 文件 | 行号 |
|------|------|------|
| `enum_mdl_type` | mdl.h | 196–329 |
| `enum_mdl_duration` | mdl.h | 333–351 |
| `MDL_key::enum_mdl_namespace` | mdl.h | 400–421 |
| `MDL_key`（私有成员） | mdl.h | 783–788 |
| `MDL_request`（类定义） | mdl.h | 801–820 |
| `MDL_wait`（类定义） | mdl.h | 1343–1370 |
| `MDL_wait::timed_wait` | mdl.cc | 1807–1848 |
| `MDL_context`（类定义） | mdl.h | 1411–1705 |
| `MDL_lock`（类定义） | mdl.cc | 427–1051 |
| `MDL_lock::MDL_lock_strategy`（结构体） | mdl.cc | 466–531 |
| `MDL_lock::m_scoped_lock_strategy` | mdl.cc | 2075–2178 |
| `MDL_lock::m_object_lock_strategy` | mdl.cc | 2185–2378 |
| `MDL_lock::get_strategy()` | mdl.cc | 819–833 |
| `MDL_lock::can_grant_lock()` | mdl.cc | 2396–2411 |
| `MDL_lock::count_piglets_and_hogs()` | mdl.cc | 672–687 |
| `MDL_lock::visit_subgraph()` | mdl.cc | 3861–3991 |
| `MDL_context::try_acquire_lock` | mdl.cc | 2685–2808 |
| `MDL_context::acquire_lock` | mdl.cc | 3359–3591 |
| `MDL_context::acquire_locks` | mdl.cc | 3638–3689 |
| `MDL_context::upgrade_shared_lock` | mdl.cc | 3745–3849 |
| `MDL_ticket::downgrade_lock` | mdl.cc | ~4296 |
| `MDL_context::release_lock` | mdl.cc | 4095–4204 |
| `MDL_context::find_deadlock` | mdl.cc | 4044–4084 |
| `MDL_ticket::get_deadlock_weight` | mdl.cc | 1698–1741 |
| `Deadlock_detection_visitor` | mdl.cc | 291–406 |
| `MDL_map`（类定义） | mdl.cc | 158–272 |
| `MDL_map::init` | mdl.cc | 1134–1149 |
| `MDL_map::find_or_insert` | mdl.cc | ~1250 |
| `MDL_BIT` 宏 | mdl.cc | 414 |

### THR_LOCK（`include/thr_lock.h` 和 `mysys/thr_lock.cc`）

| 符号 | 文件 | 行号 |
|------|------|------|
| `thr_lock_type`（枚举） | thr_lock.h | 51–95 |
| `enum_thr_lock_result` | thr_lock.h | 104–109 |
| `THR_LOCK_INFO`（结构体） | thr_lock.h | 119–122 |
| `THR_LOCK_DATA`（结构体） | thr_lock.h | 124–133 |
| `st_lock_list`（结构体） | thr_lock.h | 135–137 |
| `THR_LOCK`（结构体） | thr_lock.h | 139–154 |
| `max_write_lock_count` | thr_lock.cc | 123 |
| `thr_lock_init` | thr_lock.cc | 306–320 |
| `thr_lock_data_init` | thr_lock.cc | 338–344 |
| `thr_lock_info_init` | thr_lock.cc | 330–334 |
| `wait_for_lock` | thr_lock.cc | 356–474 |
| `thr_lock`（主函数） | thr_lock.cc | 476–684 |
| `free_all_read_locks` | thr_lock.cc | 686–729 |
| `thr_unlock` | thr_lock.cc | 733–759 |
| `wake_up_waiters` | thr_lock.cc | 769–868 |
| `LOCK_CMP` 宏 | thr_lock.cc | 876–878 |
| `sort_locks` | thr_lock.cc | 880–895 |
| `thr_multi_lock` | thr_lock.cc | 897–921 |
| `thr_multi_unlock` | thr_lock.cc | 923（起始） |
| `thr_lock_merge_status` | thr_lock.cc | 949（起始） |
| `thr_abort_locks_for_thread` | thr_lock.cc | 1016–1051 |

---

*文档生成时间：2026-04-01*  
*源码版本：MySQL 8.0.41*
