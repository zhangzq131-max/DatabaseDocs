# MySQL MDL (Metadata Lock) 锁实现分析

## 1. 概述

MDL (Metadata Lock) 是 MySQL 8.0 中用于保护数据库对象元数据的锁系统。它确保在执行 DDL 操作时，其他并发操作不会访问正在被修改的对象，从而避免元数据不一致的问题。

MDL 锁系统的主要职责：
- 保护表结构、视图、存储过程、触发器等对象的元数据一致性
- 在 DDL 执行期间阻止并发 DML 操作
- 实现全局读锁、备份锁等功能
- 与存储引擎锁协同工作，避免死锁

## 2. 整体架构

MDL 锁系统采用分层架构设计，主要组件关系如下：

```
┌─────────────────────────────────────────────────────────────────┐
│                        SQL Layer                                 │
│   (THD -> MDL_context -> MDL_request)                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      MDL Subsystem                               │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐   │
│  │  MDL_map      │───▶│   MDL_lock    │───▶│  MDL_ticket   │   │
│  │ (Global Hash) │    │ (Lock Object) │    │ (Lock Grant)  │   │
│  └───────────────┘    └───────────────┘    └───────────────┘   │
│                              │                                   │
│                              ▼                                   │
│                     ┌───────────────┐                           │
│                     │    MDL_key    │                           │
│                     │ (Object ID)   │                           │
│                     └───────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1 核心类关系

| 类名 | 职责 | 源码位置 |
|------|------|----------|
| `MDL_key` | 标识被锁对象的唯一键 | `sql/mdl.h:350-772` |
| `MDL_request` | 待处理的锁请求 | `sql/mdl.h:784-885` |
| `MDL_ticket` | 已授予的锁票据 | `sql/mdl.h:965-1089` |
| `MDL_lock` | 锁对象，管理等待/授予队列 | `sql/mdl.cc:426-1042` |
| `MDL_context` | 锁持有者上下文（每个连接一个） | `sql/mdl.h:1392-1691` |
| `MDL_map` | 全局锁哈希容器 | `sql/mdl.cc:157-271` |

### 2.2 锁策略 (Lock Strategy)

MDL 锁系统使用策略模式处理不同命名空间的锁：

```cpp
// sql/mdl.cc:810-824
inline static const MDL_lock_strategy *get_strategy(const MDL_key &key) {
  switch (key.mdl_namespace()) {
    case MDL_key::GLOBAL:
    case MDL_key::TABLESPACE:
    case MDL_key::SCHEMA:
    case MDL_key::COMMIT:
    case MDL_key::BACKUP_LOCK:
    case MDL_key::RESOURCE_GROUPS:
    case MDL_key::FOREIGN_KEY:
    case MDL_key::CHECK_CONSTRAINT:
      return &m_scoped_lock_strategy;  // 范围锁策略
    default:
      return &m_object_lock_strategy;  // 对象锁策略
  }
}
```

## 3. 关键数据结构

### 3.1 MDL_key - 锁对象标识

`MDL_key` 是元数据锁的唯一标识符，由命名空间、数据库名和对象名组成。

```cpp
// sql/mdl.h:350-406
struct MDL_key {
  enum enum_mdl_namespace {
    GLOBAL = 0,          // 全局读锁
    BACKUP_LOCK,         // 备份锁
    TABLESPACE,          // 表空间
    SCHEMA,              // 数据库/模式
    TABLE,               // 表和视图
    FUNCTION,            // 存储函数
    PROCEDURE,           // 存储过程
    TRIGGER,             // 触发器
    EVENT,               // 事件调度器事件
    COMMIT,              // 提交锁
    USER_LEVEL_LOCK,     // 用户级锁
    LOCKING_SERVICE,     // 锁定服务
    SRID,                // 空间参考系统
    ACL_CACHE,           // ACL缓存
    COLUMN_STATISTICS,   // 列统计信息
    RESOURCE_GROUPS,     // 资源组
    FOREIGN_KEY,         // 外键
    CHECK_CONSTRAINT,    // 检查约束
    NAMESPACE_END
  };
  // 键格式: <namespace_id>\0<db_name>\0<object_name>\0
  char m_ptr[MAX_MDLKEY_LENGTH];  // 最大长度 387 字节
};
```

### 3.2 MDL_request - 锁请求

表示一个待处理的锁请求，包含锁类型、持续时间和目标对象。

```cpp
// sql/mdl.h:784-885
class MDL_request {
 public:
  enum_mdl_type type;              // 锁类型
  enum_mdl_duration duration;      // 锁持续时间
  MDL_key key;                     // 目标对象键
  MDL_ticket *ticket;              // 满足后指向票据
  MDL_request *next_in_list;       // 链表指针
  MDL_request **prev_in_list;
};
```

### 3.3 MDL_ticket - 锁票据

表示已授予的锁，是 `MDL_context` 和 `MDL_lock` 之间的连接纽带。

```cpp
// sql/mdl.h:965-1089
class MDL_ticket : public MDL_wait_for_subgraph {
 public:
  MDL_ticket *next_in_context;     // context 链表
  MDL_ticket **prev_in_context;
  MDL_ticket *next_in_lock;        // lock 链表
  MDL_ticket **prev_in_lock;

 private:
  enum_mdl_type m_type;            // 锁类型
  MDL_context *m_ctx;              // 拥有者
  MDL_lock *m_lock;                // 锁对象
  bool m_is_fast_path;             // 是否使用快速路径获取
  PSI_metadata_lock *m_psi;        // PFS instrumentation
};
```

### 3.4 MDL_lock - 锁对象

代表一个具体的锁对象，管理授予和等待队列。

```cpp
// sql/mdl.cc:426-1042
class MDL_lock {
 public:
  MDL_key key;                     // 对象键
  mysql_prlock_t m_rwlock;         // 保护内部状态的读写锁
  Ticket_list m_granted;           // 已授予的票据列表
  Ticket_list m_waiting;           // 等待中的票据列表
  
  // 快速路径状态（原子计数器）
  std::atomic<fast_path_state_t> m_fast_path_state;
  
  // 死锁权重计数器（防止低优先级锁饥饿）
  ulong m_hog_lock_count;          // SNW/SNRW/X 计数
  ulong m_piglet_lock_count;       // SW 计数
  
  const MDL_lock_strategy *m_strategy;  // 锁策略
};
```

### 3.5 MDL_context - 锁持有者上下文

每个连接（THD）拥有一个 `MDL_context`，管理该连接持有的所有锁。

```cpp
// sql/mdl.h:1392-1691
class MDL_context {
 public:
  MDL_wait m_wait;                 // 等待状态管理
  MDL_ticket_store m_ticket_store; // 持有的票据（按持续时间分组）
  MDL_context_owner *m_owner;      // 关联的 THD
  
 private:
  mysql_prlock_t m_LOCK_waiting_for;  // 保护 m_waiting_for
  MDL_wait_for_subgraph *m_waiting_for;  // 当前等待的对象（死锁检测）
  LF_PINS *m_pins;                 // 无锁哈希的 hazard pointers
};
```

### 3.6 MDL_ticket_store - 票据存储

使用侵入式链表 + 哈希索引管理票据，优化查找性能。

```cpp
// sql/mdl.h:1096-1288
class MDL_ticket_store {
 private:
  Duration m_durations[MDL_DURATION_END];  // 按持续时间分组的链表
  std::unique_ptr<Ticket_map> m_map;       // 哈希索引（超过阈值时启用）
  const size_t THRESHOLD = 256;            // 启用哈希的阈值
};
```

## 4. 锁类型详解

### 4.1 enum_mdl_type - 锁类型枚举

```cpp
// sql/mdl.h:181-314
enum enum_mdl_type {
  MDL_INTENTION_EXCLUSIVE = 0,  // IX - 意向排他锁（仅用于范围锁）
  MDL_SHARED,                   // S  - 共享锁（仅访问元数据）
  MDL_SHARED_HIGH_PRIO,         // SH - 高优先级共享锁（IS表填充）
  MDL_SHARED_READ,              // SR - 共享读锁（SELECT使用）
  MDL_SHARED_WRITE,             // SW - 共享写锁（INSERT/UPDATE/DELETE）
  MDL_SHARED_WRITE_LOW_PRIO,    // SWLP - 低优先级写锁（LOW_PRIORITY DML）
  MDL_SHARED_UPGRADABLE,        // SU - 可升级共享锁（ALTER第一阶段）
  MDL_SHARED_READ_ONLY,         // SRO - 只读共享锁（LOCK TABLE READ）
  MDL_SHARED_NO_WRITE,          // SNW - 无写共享锁（ALTER复制阶段）
  MDL_SHARED_NO_READ_WRITE,     // SNRW - 无读写锁（LOCK TABLE WRITE）
  MDL_EXCLUSIVE,                // X  - 排他锁（CREATE/DROP/RENAME）
  MDL_TYPE_END
};
```

### 4.2 锁类型详细说明

| 锁类型 | 缩写 | 典型使用场景 | 特点 |
|--------|------|-------------|------|
| `MDL_INTENTION_EXCLUSIVE` | IX | 范围锁，DML 语句 | 意向锁，兼容其他 IX |
| `MDL_SHARED` | S | 存储过程访问、PREPARE | 仅访问元数据 |
| `MDL_SHARED_HIGH_PRIO` | SH | INFORMATION_SCHEMA | 高优先级，忽略等待的 X 锁 |
| `MDL_SHARED_READ` | SR | SELECT、子查询 | 可读取数据 |
| `MDL_SHARED_WRITE` | SW | INSERT/UPDATE/DELETE | 可修改数据 |
| `MDL_SHARED_WRITE_LOW_PRIO` | SWLP | LOW_PRIORITY DML | 低于 SRO 的优先级 |
| `MDL_SHARED_UPGRADABLE` | SU | ALTER TABLE 第一阶段 | 可升级到 X |
| `MDL_SHARED_READ_ONLY` | SRO | LOCK TABLES READ | 阻止所有写操作 |
| `MDL_SHARED_NO_WRITE` | SNW | ALTER 复制数据阶段 | 允许读，阻止写 |
| `MDL_SHARED_NO_READ_WRITE` | SNRW | LOCK TABLES WRITE | 阻止读写，允许 SHOW |
| `MDL_EXCLUSIVE` | X | CREATE/DROP/RENAME | 独占访问 |

### 4.3 enum_mdl_duration - 锁持续时间

```cpp
// sql/mdl.h:316-336
enum enum_mdl_duration {
  MDL_STATEMENT = 0,    // 语句结束时自动释放
  MDL_TRANSACTION,      // 事务结束时自动释放
  MDL_EXPLICIT,         // 需显式释放（HANDLER、LOCK TABLES）
  MDL_DURATION_END
};
```

## 5. 兼容性矩阵

### 5.1 范围锁 (Scoped Lock) 兼容性

范围锁用于 GLOBAL、COMMIT、TABLESPACE、SCHEMA 等命名空间。

```
                | Type of active   |
        Request |   scoped lock    |
         type   | IS(*)  IX   S  X |
       ---------+------------------+
       IS       |  +      +   +  + |
       IX       |  +      +   -  - |
       S        |  +      -   +  - |
       X        |  +      -   -  - |

注: IS 没有实际使用，"+" 表示兼容，"-" 表示不兼容
```

**源码定义** (`sql/mdl.cc:2066-2169`):
```cpp
const MDL_lock::MDL_lock_strategy MDL_lock::m_scoped_lock_strategy = {
  // granted_incompatible 数组
  {MDL_BIT(MDL_EXCLUSIVE) | MDL_BIT(MDL_SHARED),
   MDL_BIT(MDL_EXCLUSIVE) | MDL_BIT(MDL_INTENTION_EXCLUSIVE),
   0, 0, 0, 0, 0, 0, 0, 0,
   MDL_BIT(MDL_EXCLUSIVE) | MDL_BIT(MDL_SHARED) | MDL_BIT(MDL_INTENTION_EXCLUSIVE)},
  // ...
};
```

### 5.2 对象锁 (Object Lock) 兼容性

对象锁用于 TABLE、FUNCTION、PROCEDURE 等命名空间。

```
    Request  |  Granted requests for lock            |
     type    | S  SH  SR  SW  SWLP  SU  SRO  SNW  SNRW  X  |
   ----------+---------------------------------------------+
   S         | +   +   +   +    +    +   +    +    +    -  |
   SH        | +   +   +   +    +    +   +    +    +    -  |
   SR        | +   +   +   +    +    +   +    +    -    -  |
   SW        | +   +   +   +    +    +   -    -    -    -  |
   SWLP      | +   +   +   +    +    +   -    -    -    -  |
   SU        | +   +   +   +    +    -   +    -    -    -  |
   SRO       | +   +   +   -    -    +   +    +    -    -  |
   SNW       | +   +   +   -    -    -   +    -    -    -  |
   SNRW      | +   +   -   -    -    -   -    -    -    -  |
   X         | -   -   -   -    -    -   -    -    -    -  |
```

**源码定义** (`sql/mdl.cc:2176-2332`):
```cpp
const MDL_lock::MDL_lock_strategy MDL_lock::m_object_lock_strategy = {
  // granted_incompatible 数组 - 指定与已授予锁的冲突
  {0,                              // IX 不用于对象锁
   MDL_BIT(MDL_EXCLUSIVE),         // S 与 X 不兼容
   MDL_BIT(MDL_EXCLUSIVE),         // SH 与 X 不兼容
   MDL_BIT(MDL_EXCLUSIVE) | MDL_BIT(MDL_SHARED_NO_READ_WRITE),
   // ... 更多类型定义
  },
  // ...
};
```

### 5.3 等待优先级矩阵

为防止低优先级锁请求饥饿，系统使用多套优先级矩阵：

```cpp
// sql/mdl.cc:2269-2332
// 当 hog/piglet 锁连续授予超过 max_write_lock_count 时切换矩阵
// hog 类型: SNW, SNRW, X
// piglet 类型: SW

{{...}, // 默认矩阵
 {...}, // piglet 锁超阈值
 {...}, // hog 锁超阈值  
 {...}  // 两者都超阈值
}
```

## 6. 快速路径优化 (Fast Path)

### 6.1 设计思想

对于高频、低冲突的"非突兀"(unobtrusive) 锁类型，使用原子计数器替代复杂的链表操作：

```cpp
// sql/mdl.cc:680-720
// 非突兀锁类型定义：
// - 范围锁: IX
// - 对象锁: S, SH, SR, SW, SWLP
//
// 特点：
// 1. 同类锁互相兼容
// 2. DML 操作常用
// 3. 可通过原子增量/减量获取/释放
```

### 6.2 Fast Path 状态编码

```cpp
// sql/mdl.cc:826-908
std::atomic<fast_path_state_t> m_fast_path_state;

// 对象锁编码：
// bits 0-19:   S/SH 锁计数 (20位)
// bits 20-39:  SR 锁计数  (20位)
// bits 40-59:  SW/SWLP 锁计数 (20位)
// bit 60:      HAS_SLOW_PATH 标志
// bit 61:      HAS_OBTRUSIVE 标志
// bit 62:      IS_DESTROYED 标志

// 范围锁编码：
// bits 0-59:   IX 锁计数
// bit 60-62:   标志位
```

### 6.3 Fast Path 获取流程

```cpp
// sql/mdl.cc:2920-3030
bool MDL_context::try_acquire_lock_impl(MDL_request *mdl_request, ...) {
  // 1. 检查是否已有更强锁
  if (find_ticket(mdl_request, ...)) return true;
  
  // 2. 获取快速路径增量
  fast_path_state_t unobtrusive_lock_increment = 
      lock->get_unobtrusive_lock_increment(mdl_request->type);
  
  // 3. 如果是"突兀"锁或需要特殊处理，走慢速路径
  if (!unobtrusive_lock_increment || m_needs_thr_lock_abort) {
    // 慢速路径...
  }
  
  // 4. 尝试原子增量获取
  do {
    old_state = lock->m_fast_path_state.load();
    new_state = old_state + unobtrusive_lock_increment;
    // 检查是否有阻塞的突兀锁...
  } while (!lock->fast_path_state_cas(&old_state, new_state));
  
  // 5. 创建 fast path 票据
  ticket->m_is_fast_path = true;
}
```

## 7. 死锁检测机制

### 7.1 Wait-For Graph 遍历

MDL 使用图遍历算法检测死锁：

```cpp
// sql/mdl.cc:290-342
class Deadlock_detection_visitor : public MDL_wait_for_graph_visitor {
  MDL_context *m_start_node;      // 搜索起点
  MDL_context *m_victim;          // 选中受害者
  uint m_current_search_depth;    // 当前深度
  bool m_found_deadlock;          // 发现死锁
  
  static const uint MAX_SEARCH_DEPTH = 32;  // 最大搜索深度
};
```

### 7.2 检测流程

```cpp
// sql/mdl.cc:3982-4023
void MDL_context::find_deadlock() {
  while (true) {
    Deadlock_detection_visitor dvisitor(this);
    
    if (!visit_subgraph(&dvisitor)) {
      // 未发现死锁，退出
      break;
    }
    
    // 发现死锁，选择受害者
    victim = dvisitor.get_victim();
    victim->m_wait.set_status(MDL_wait::VICTIM);
    victim->unlock_deadlock_victim();
    
    if (victim == this) break;  // 自己是受害者
    
    // 受害者不是自己，继续检测
  }
}
```

### 7.3 死锁权重选择

```cpp
// sql/mdl.h:937-942
class MDL_wait_for_subgraph {
  static const uint DEADLOCK_WEIGHT_DML = 0;    // DML 最低权重
  static const uint DEADLOCK_WEIGHT_ULL = 50;   // 用户锁
  static const uint DEADLOCK_WEIGHT_DDL = 100;  // DDL 最高权重
  
  virtual uint get_deadlock_weight() const = 0;
};
```

选择策略：
- 权重越高越不容易被选为受害者
- DDL 操作优先级高于 DML
- 尽量不牺牲已投入较多资源的操作

## 8. 使用场景分析

### 8.1 SELECT 语句

```sql
SELECT * FROM t1 WHERE id = 1;
```

**MDL 锁序列**：
1. 获取 SCHEMA 的 IX 锁（范围锁）
2. 获取表 t1 的 SR 锁（对象锁）
3. 锁持续时间为 MDL_TRANSACTION

### 8.2 INSERT/UPDATE/DELETE 语句

```sql
INSERT INTO t1 VALUES (1, 'test');
```

**MDL 锁序列**：
1. 获取 GLOBAL 的 IX 锁（范围锁）
2. 获取 SCHEMA 的 IX 锁（范围锁）
3. 获取表 t1 的 SW 锁（对象锁）

### 8.3 ALTER TABLE 语句

```sql
ALTER TABLE t1 ADD COLUMN c2 INT;
```

**MDL 锁序列**（多阶段）：
1. **第一阶段**：获取 SU 锁（可升级）
2. **复制数据阶段**：升级到 SNW 锁（允许并发读）
3. **最后阶段**：升级到 X 锁（短暂独占）

### 8.4 DROP TABLE 语句

```sql
DROP TABLE t1;
```

**MDL 锁序列**：
1. 获取 SCHEMA 的 IX 锁
2. 获取表 t1 的 X 锁
3. 阻止所有并发访问

### 8.5 LOCK TABLES 语句

```sql
LOCK TABLES t1 READ;
```

**MDL 锁序列**：
- 获取表 t1 的 SRO 锁
- 阻止所有写操作

```sql
LOCK TABLES t1 WRITE;
```

**MDL 锁序列**：
- 获取表 t1 的 SNRW 锁
- 阻止所有读写操作

### 8.6 全局读锁 (FLUSH TABLES WITH READ LOCK)

```sql
FLUSH TABLES WITH READ LOCK;
```

**MDL 锁序列**：
1. 获取 GLOBAL 的 S 锁
2. 获取 COMMIT 的 S 锁
3. 阻止所有数据修改和提交

## 9. 关键函数实现

### 9.1 can_grant_lock

判断锁请求是否可以满足：

```cpp
// sql/mdl.cc:2387-2449
bool MDL_lock::can_grant_lock(enum_mdl_type type_arg,
                              const MDL_context *requestor_ctx) const {
  bitmap_t waiting_incompat_map = incompatible_waiting_types_bitmap()[type_arg];
  bitmap_t granted_incompat_map = incompatible_granted_types_bitmap()[type_arg];
  
  // 三个条件：
  // 1. 无更高优先级的等待请求
  // 2. 无冲突的 fast path 锁
  // 3. 无冲突的 granted 锁（排除自己的）
  
  if (!(m_waiting.bitmap() & waiting_incompat_map)) {
    if (!(fast_path_granted_bitmap() & granted_incompat_map)) {
      if (!(m_granted.bitmap() & granted_incompat_map))
        return true;
      // 检查冲突锁是否属于自己...
    }
  }
  return false;
}
```

### 9.2 acquire_lock

获取锁（可能等待）：

```cpp
// sql/mdl.cc:3360-3490
bool MDL_context::acquire_lock(MDL_request *mdl_request, Timeout_type timeout) {
  // 1. 尝试无等待获取
  if (try_acquire_lock_impl(mdl_request, &ticket)) return true;
  
  // 2. 需要等待，加入等待队列
  will_wait_for(ticket);
  
  // 3. 死锁检测
  find_deadlock();
  
  // 4. 等待锁授予或超时
  wait_status = m_wait.timed_wait(m_owner, &abs_timeout, ...);
  
  // 5. 处理等待结果
  switch (wait_status) {
    case GRANTED:   // 成功获取
    case VICTIM:    // 死锁受害者
    case TIMEOUT:   // 超时
    case KILLED:    // 连接被杀
  }
}
```

### 9.3 release_lock

释放锁：

```cpp
// sql/mdl.cc:4033-4140
void MDL_context::release_lock(enum_mdl_duration duration, MDL_ticket *ticket) {
  MDL_lock *lock = ticket->m_lock;
  
  // Fast path 释放（原子减量）
  if (ticket->m_is_fast_path) {
    lock->fast_path_state_add(-lock->get_unobtrusive_lock_increment(ticket->get_type()));
    MDL_ticket::destroy(ticket);
    return;
  }
  
  // Slow path 释放
  lock->remove_ticket(this, m_pins, &MDL_lock::m_granted, ticket);
  
  // 尝试唤醒等待者
  lock->reschedule_waiters();
}
```

## 10. 性能优化要点

### 10.1 锁对象缓存

```cpp
// sql/mdl.cc:157-271
class MDL_map {
  LF_HASH m_locks;               // 无锁哈希
  MDL_lock *m_global_lock;       // 预分配全局锁
  MDL_lock *m_commit_lock;       // 预分配提交锁
  std::atomic<int32> m_unused_lock_objects;  // 未使用对象计数
  
  // 超过阈值时自动清理未使用锁对象
  void lock_object_unused(MDL_context *ctx, LF_PINS *pins);
};
```

### 10.2 单例锁优化

```cpp
// sql/mdl.cc:1173-1208
// GLOBAL、COMMIT、ACL_CACHE、BACKUP_LOCK 使用预分配对象
// 避免哈希查找开销

MDL_lock *MDL_map::find(LF_PINS *pins, const MDL_key *mdl_key, bool *pinned) {
  if (is_lock_object_singleton(mdl_key)) {
    // 直接返回预分配对象，无需哈希查找
    switch (mdl_key->mdl_namespace()) {
      case MDL_key::GLOBAL:  return m_global_lock;
      case MDL_key::COMMIT:  return m_commit_lock;
      // ...
    }
    *pinned = false;  // 无需 pin
  }
}
```

### 10.3 读偏好读写锁

```cpp
// sql/mdl.cc:532-567
// m_rwlock 使用读偏好模式 (PSI_FLAG_RWLOCK_PR)
// 保证死锁检测器不会自身死锁
// 
// 例如两个 context 同时检测死锁：
// ctxA read-locks obj1 -> ctxA goes deeper
// ctxB read-locks obj2 -> ctxB goes deeper
// 
// 如果写偏好，ctxC 写锁 obj1 会阻塞 ctxA 的后续读
// 读偏好确保 ctxA/ctxB 能继续检测
```

## 11. 总结

MySQL MDL 锁系统是一个精心设计的元数据保护机制：

1. **分层架构**：`MDL_key` → `MDL_lock` → `MDL_ticket` → `MDL_context` 清晰分离职责
2. **策略模式**：范围锁和对象锁使用不同兼容性规则
3. **快速路径**：高频锁类型使用原子操作，显著提升性能
4. **死锁检测**：图遍历算法 + 权重选择，合理处理死锁
5. **饥饿预防**：动态切换优先级矩阵，防止低优先级锁饥饿
6. **内存管理**：自动清理未使用锁对象，控制内存占用

MDL 锁与存储引擎锁（如 InnoDB 行锁）协同工作，共同保证数据库操作的并发安全性。