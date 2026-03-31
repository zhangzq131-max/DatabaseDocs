# MySQL 8.0.41 Global_read_lock 源码深度分析

> 源码路径：`sql/sql_class.h`、`sql/lock.cc`、`plugin/innodb_memcached/`

---

## 一、Global_read_lock 类的定位与职责

`Global_read_lock` 是每个连接线程（`THD`）中的一个成员对象（`thd->global_read_lock`），**负责在连接级别持有全局读锁（GRL，Global Read Lock）**。它是 `FLUSH TABLES WITH READ LOCK` SQL 语句的底层实现载体。

```cpp
// sql/sql_class.h:2078
Global_read_lock global_read_lock;
```

---

## 二、状态机（三态）

```cpp
// sql/sql_class.h:830
enum enum_grl_state {
    GRL_NONE,                      // 当前连接未持有全局读锁
    GRL_ACQUIRED,                  // 已持有全局读锁，阻止 DDL 和写操作，但 COMMIT 还允许
    GRL_ACQUIRED_AND_BLOCKS_COMMIT // 已持有全局读锁，并额外阻止 COMMIT
};
```

| 状态 | 含义 |
|------|------|
| `GRL_NONE` | 当前连接未持有全局读锁 |
| `GRL_ACQUIRED` | 已持有全局读锁，**阻止 DDL 和写操作**，但 COMMIT 还允许 |
| `GRL_ACQUIRED_AND_BLOCKS_COMMIT` | 已持有全局读锁，并额外**阻止 COMMIT** |

---

## 三、底层实现机制：两把 MDL 锁

该类并不维护任何自定义锁结构，完全依赖 MDL（元数据锁）子系统：

```cpp
// sql/sql_class.h:868
/**
  In order to acquire the global read lock, the connection must
  acquire shared metadata lock in GLOBAL namespace, to prohibit
  all DDL.
*/
MDL_ticket *m_mdl_global_shared_lock;

/**
  Also in order to acquire the global read lock, the connection
  must acquire a shared metadata lock in COMMIT namespace, to
  prohibit commits.
*/
MDL_ticket *m_mdl_blocks_commits_lock;
```

- **`m_mdl_global_shared_lock`**：GLOBAL 命名空间的 `MDL_SHARED` 锁 → 任何写操作（DDL/DML）在打开表时都需要 `MDL_INTENTION_EXCLUSIVE`（IX），该 S 锁与 IX 不相容，从而**全局阻塞所有写操作**
- **`m_mdl_blocks_commits_lock`**：COMMIT 命名空间的 `MDL_SHARED` 锁 → COMMIT 时需获取该命名空间的 IX 锁，从而**阻塞所有事务提交**

---

## 四、关键方法实现

### 4.1 `lock_global_read_lock()` — 第一步加锁

```cpp
// sql/lock.cc:1047
bool Global_read_lock::lock_global_read_lock(THD *thd) {
  if (!m_state) {
    MDL_request mdl_request;
    MDL_REQUEST_INIT(&mdl_request, MDL_key::GLOBAL, "", "", MDL_SHARED,
                     MDL_EXPLICIT);

    /* Increment static variable first to signal innodb memcached server
       to release mdl locks held by it */
    Global_read_lock::m_atomic_active_requests++;
    if (thd->mdl_context.acquire_lock(&mdl_request,
                                      thd->variables.lock_wait_timeout)) {
      Global_read_lock::m_atomic_active_requests--;
      return true;
    }

    m_mdl_global_shared_lock = mdl_request.ticket;
    m_state = GRL_ACQUIRED;
  }
  return false;
}
```

获取 GLOBAL 命名空间的共享锁，状态升至 `GRL_ACQUIRED`，同时递增全局计数器通知 InnoDB Memcached（详见第六节）。

### 4.2 `make_global_read_lock_block_commit()` — 第二步加锁

```cpp
// sql/lock.cc:1121
bool Global_read_lock::make_global_read_lock_block_commit(THD *thd) {
  if (m_state != GRL_ACQUIRED) return false;
  MDL_REQUEST_INIT(&mdl_request, MDL_key::COMMIT, "", "", MDL_SHARED,
                   MDL_EXPLICIT);

  if (thd->mdl_context.acquire_lock(&mdl_request,
                                    thd->variables.lock_wait_timeout))
    return true;

  m_mdl_blocks_commits_lock = mdl_request.ticket;
  m_state = GRL_ACQUIRED_AND_BLOCKS_COMMIT;
  return false;
}
```

额外获取 COMMIT 命名空间的共享锁，禁止任何 COMMIT，状态升至 `GRL_ACQUIRED_AND_BLOCKS_COMMIT`。

### 4.3 `unlock_global_read_lock()` — 释放锁

```cpp
// sql/lock.cc:1091
void Global_read_lock::unlock_global_read_lock(THD *thd) {
  if (m_mdl_blocks_commits_lock) {
    thd->mdl_context.release_lock(m_mdl_blocks_commits_lock);
    m_mdl_blocks_commits_lock = nullptr;
  }
  thd->mdl_context.release_lock(m_mdl_global_shared_lock);
  Global_read_lock::m_atomic_active_requests--;
  m_mdl_global_shared_lock = nullptr;
  m_state = GRL_NONE;
}
```

### 4.4 `can_acquire_protection()` — 防冲突检测

```cpp
// sql/sql_class.h:854
bool can_acquire_protection() const {
  if (m_state) {
    my_error(ER_CANT_UPDATE_WITH_READLOCK, MYF(0));
    return true;
  }
  return false;
}
```

**本连接自检**：若本连接已持有 GRL，就不能再做写操作。在 `sql_base.cc`、`lock.cc`、`sql_trigger.cc` 等所有写操作入口前广泛调用。

---

## 五、两步设计的原因（防死锁）

`lock.cc` 中的注释详细解释了为何 FTWRL 必须分两步执行：

```
FTWRL 执行顺序：
1) lock_global_read_lock()       → 阻止新的写锁，即阻塞所有新的 UPDATE/DDL
2) close_cached_tables()         → 等待当前正在更新的表关闭（即 FLUSH TABLES）
3) make_global_read_lock_block_commit() → 阻止 COMMIT
```

若将步骤 1 和步骤 3 合并为一步（同时阻止写操作和 COMMIT），会引发以下三线程死锁：

```
thd1: SELECT * FROM t FOR UPDATE;        -- 持有行锁
thd2: UPDATE t SET a=1;                  -- 被 thd1 的行锁阻塞
thd3: FLUSH TABLES WITH READ LOCK;       -- 在 close_cached_tables() 里被 thd2 的表实例阻塞
thd1: COMMIT;                            -- 如果步骤3已执行，被 thd3 阻塞
```

`thd1 → 阻塞 thd2 → 阻塞 thd3 → 阻塞 thd1`，形成死锁。

分两步的关键在于：**在步骤 2 完成之前（所有表已关闭），COMMIT 仍被允许**，thd1 可以正常提交，从而解除死锁链。

---

## 六、GRL 持有期间的 DDL 权限分析

### 6.1 MDL 兼容矩阵

GLOBAL 命名空间使用 Scoped Lock 策略，其兼容矩阵如下（来自 `sql/mdl.cc:2084`）：

```
         | Type of active   |
 Request |   scoped lock    |
  type   | IS(*)  IX   S  X |
---------+------------------+
IS       |  +      +   +  + |
IX       |  +      +   -  - |  ← IX 被 S 阻塞
S        |  +      -   +  - |
X        |  +      -   -  - |

"+" 表示可以授予，"-" 表示阻塞等待
(*) IS 与所有锁兼容，不做计数
```

GRL 持有者在 GLOBAL 命名空间持有 **S 锁**。任何 DDL/DML 写操作都需要 GLOBAL **IX 锁**。对照矩阵：`IX 请求` vs `已持有 S 锁` = **`-`（阻塞）**。

### 6.2 CREATE DATABASE / CREATE TABLE 的加锁路径

**CREATE DATABASE 调用链：**

```
mysql_create_db()                          [sql/sql_db.cc]
  └─ lock_schema_name()                    [sql/lock.cc:739]
        ├─ can_acquire_protection()        ← 本连接自检
        └─ acquire_locks({
               GLOBAL IX,                  ← 被 GRL 的 S 锁阻塞（其他连接）
               SCHEMA X,
               BACKUP_LOCK IX
           })
```

**CREATE TABLE 调用链：**

```
open_tables()                              [sql/sql_base.cc]
  ├─ can_acquire_protection()              ← 本连接自检
  └─ acquire_lock(GLOBAL IX)              ← 被 GRL 的 S 锁阻塞（其他连接）
```

### 6.3 结论对比

| 场景 | CREATE DATABASE | CREATE TABLE | 行为原因 |
|------|:--------------:|:------------:|--------|
| **GRL 持有连接自己** | ❌ 立即报错 | ❌ 立即报错 | `can_acquire_protection()` 检测到 `m_state != GRL_NONE`，直接上报 `ER_CANT_UPDATE_WITH_READLOCK`（错误码 1223），不尝试获取 MDL |
| **其他连接** | ❌ 阻塞等待 | ❌ 阻塞等待 | GLOBAL IX 锁请求在 MDL 层被 GRL 的 GLOBAL S 锁阻塞，等待 `lock_wait_timeout` 后报 `ER_LOCK_WAIT_TIMEOUT`（错误码 1205） |

> **本质区别**：持锁连接是 SQL 层预检查直接拒绝（立即报错），其他连接是 MDL 层阻塞排队等待（超时后报错）。

---

## 七、与 InnoDB Memcached 插件的关联

### 7.1 问题背景

MySQL 内置了 `daemon_memcached` 插件，允许应用通过 Memcached 协议**绕过 SQL 层直接读写 InnoDB 表**。插件内部维护一批后台连接（`innodb_conn_data_t`），这些连接通过 `handler_open_table()` **持有 InnoDB 表的 MDL 锁**（写模式下为 `MDL_SHARED_WRITE` 等），用于持续处理 key-value 操作。

**冲突点**：`FLUSH TABLES WITH READ LOCK` 的第二步 `close_cached_tables()` 必须等待所有正在打开该表的连接**释放 MDL 表锁**才能继续。但 Memcached 插件的后台连接不是普通 SQL 连接，不受 `can_acquire_protection()` 检查约束，其 MDL 表锁不会主动释放，导致 FTWRL 卡死。

### 7.2 解决方案：原子计数器作为跨层信号量

关键在于 `lock_global_read_lock()` 的操作顺序：

```cpp
// sql/lock.cc:1058
/* Increment static variable first to signal innodb memcached server
   to release mdl locks held by it */
Global_read_lock::m_atomic_active_requests++;       // ← 先递增计数器（发出信号）
if (thd->mdl_context.acquire_lock(&mdl_request,    // ← 再申请 MDL S 锁
                                  thd->variables.lock_wait_timeout)) {
  Global_read_lock::m_atomic_active_requests--;
  return true;
}
```

**先递增计数器，再去申请 MDL S 锁**——给 Memcached 后台线程一个窗口期，让它在 S 锁被真正持有前提前释放表锁。

### 7.3 Memcached 后台线程的响应逻辑

Memcached 插件的后台线程（`innodb_bk_thread`）每隔 `bk_commit_interval` 秒轮询计数器：

```cpp
// plugin/innodb_memcached/innodb_memcache/src/innodb_engine.cc:334
while (!memcached_shutdown) {
    release_mdl_lock = handler_check_global_read_lock_active();
    // ... 等待 bk_commit_interval 秒 ...
}
```

`handler_check_global_read_lock_active()` 直接调用：

```cpp
// plugin/innodb_memcached/innodb_memcache/src/handler_api.cc:366
bool handler_check_global_read_lock_active() {
  return Global_read_lock::global_read_lock_active();
}
```

一旦 `release_mdl_lock = true`，在下次处理连接时触发 `innodb_open_mysql_table()`：

```cpp
// innodb_engine.cc:933
/* This special case added to facilitate unlocking
   of MDL lock during FLUSH TABLE WITH READ LOCK */
if (engine && release_mdl_lock &&
    (engine->enable_binlog || engine->enable_mdl)) {
    innodb_open_mysql_table(conn_data, conn_option, engine);
}
```

`innodb_open_mysql_table()` 第一步就主动关闭表、**释放 MDL 表锁**，然后尝试重新打开（此时会被 GRL 的 GLOBAL S 锁阻塞排队等待）：

```cpp
// innodb_engine.cc:761
/* Close the table before opening it again */
innodb_close_mysql_table(conn_data);         // ← 释放 MDL 表锁
...
conn_data->mysql_tbl = handler_open_table(   // ← 重新打开，此时被 GRL 阻塞
    conn_data->thd, ..., HDL_WRITE);
```

### 7.4 完整协调时序

```
FTWRL 连接                                  Memcached 后台线程
──────────────────────────────              ──────────────────────────────
m_atomic_active_requests++                  轮询: release_mdl_lock = true
         ↓                                              ↓
尝试获取 GLOBAL S 锁                         innodb_close_mysql_table()
（等待 Memcached 释放 MDL 表锁）               → 主动释放 MDL 表锁 ✓
         ↓                                              ↓
GLOBAL S 锁获取成功                           尝试 handler_open_table()
         ↓                                    → 被 GLOBAL S 锁阻塞，进入等待
close_cached_tables() 成功
         ↓
make_global_read_lock_block_commit()
（GLOBAL + COMMIT 双 S 锁持有，
  服务器进入全局静止点）
```

---

## 八、GRL 的调用场景汇总

| 调用位置 | 触发 SQL | 作用 |
|---|---|---|
| `sql/sql_reload.cc` | `FLUSH TABLES WITH READ LOCK` | 主要使用场景：热备份一致性快照 |
| `sql/sys_vars.cc` | `SET GLOBAL read_only = 1` / `super_read_only = 1` | 切换只读模式时需要排他地修改该变量 |
| `sql/rpl_source.cc` | `RESET MASTER` | 重置 binlog 时持锁保证一致性 |
| `sql/sql_parse.cc` | `UNLOCK TABLES` | 释放 GRL |
| `sql/sql_class.cc` | THD 析构 | 连接断开时若持有 GRL 则自动释放 |

---

## 九、总结

`Global_read_lock` 本质上是**每个连接私有的 GRL 状态机**，通过两个 MDL ticket（GLOBAL S 锁 + COMMIT S 锁）实现服务器级的全局静止点（quiesced state）。

| 组件 | 作用 |
|------|------|
| `m_mdl_global_shared_lock` | 阻塞所有 DDL 和 DML 写操作（与其他连接的 GLOBAL IX 锁不相容） |
| `m_mdl_blocks_commits_lock` | 阻塞所有事务 COMMIT（与 COMMIT 命名空间的 IX 锁不相容） |
| `m_atomic_active_requests` | 跨层信号量，通知 InnoDB Memcached 插件主动释放 MDL 表锁 |
| `can_acquire_protection()` | SQL 层预检查，防止持锁连接自身发起写操作（立即报错而非阻塞） |

它是 MySQL 热备份工具（`mysqldump --single-transaction`、XtraBackup）和主从同步位点一致性保证的核心基础设施。
