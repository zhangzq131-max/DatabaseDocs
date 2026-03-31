# MySQL 8.0.41 慢查询日志实现深度分析

> 基于 MySQL 8.0.41 源码（`sql/log.cc`、`sql/sql_parse.cc`、`sql/sql_class.cc`、`sql/lock.cc` 等）

---

## 一、整体架构与调用链路

慢日志从查询开始到落盘，经历以下核心阶段：

```
do_command()               [sql/sql_parse.cc]
  │
  ├─ 网络读包: get_command()          ← 等待客户端报文（此时尚未计时）
  │
  └─ dispatch_command()
       │
       ├─ thd->set_time()             ← ★ 计时正式开始（start_utime 在此设置）
       │
       ├─ 解析 → 优化 → 执行
       │    └─ mysql_lock_tables()    ← ★ TABLE 锁等待时间累加
       │    └─ InnoDB 行锁等待        ← ★ 行级锁时间通过回调累加
       │
       ├─ thd->update_slow_query_status()  ← 判断是否超过 long_query_time
       │
       └─ log_slow_statement()
            ├─ log_slow_applicable()  ← 最终写入决策
            └─ log_slow_do()          ← 写文件 / 写 mysql.slow_log 表
```

---

## 二、计时起始点：`start_utime` 的设置

### 2.1 源码定位

`**sql/sql_parse.cc:1731**`（`dispatch_command()` 函数开头）：

```cpp
// sql/sql_parse.cc:1731
thd->set_time();
```

`**sql/sql_class.cc:3200**`（`THD::set_time()` 实现）：

```cpp
void THD::set_time() {
  start_utime = my_micro_time();   // 记录微秒级当前时间
  m_lock_usec = 0;                 // 同时将锁时间计数器清零
  if (user_time.tv_sec || user_time.tv_usec)
    start_time = user_time;
  else
    my_micro_time_to_timeval(start_utime, &start_time);
  // ...
}
```

### 2.2 关键结论：计时不包含网络读取时间

在 `do_command()` 中，调用顺序是：

```
rc = thd->get_protocol()->get_command(...)  // 第一步：阻塞读取网络报文
↓
dispatch_command(...)                        // 第二步：进入命令分发
    └─ thd->set_time()                       // 第三步：此时才开始计时
```

**因此 `Query_time` 不包含以下时间**：

- 客户端"思考时间"（上一个响应发出到下一条 SQL 发来的间隔）
- 网络传输延迟（客户端发送报文的网络 RTT）
- 服务端等待接收报文的阻塞时间

> **运维意义**：若 DBA 对比客户端侧与慢日志中的耗时差异，二者差值通常就是网络往返延迟 + 客户端处理时间。`Query_time` 只衡量 MySQL 服务器内部执行时间。

### 2.3 多语句场景下的重置

当客户端在一个报文中发送多条 SQL（以分号分隔），每条语句执行完毕后会调用：

```cpp
// sql/sql_parse.cc:2137
thd->set_time(); /* Reset the query start time. */
```

保证多语句包中每条 SQL 都有独立的计时基准。

---

## 三、锁时间统计（`Lock_time`）

### 3.1 存储结构

锁时间存储在 `THD` 类的私有成员中（`sql/sql_class.h:1574`）：

```cpp
private:
  ulonglong m_lock_usec;  // 单位：微秒，累加锁相关时间（非纯等待，详见第十一节）

public:
  ulonglong get_lock_usec() { return m_lock_usec; }
  void inc_lock_usec(ulonglong usec);    // 累加
  void push_lock_usec(ulonglong &top);   // 压栈（存储过程用）
  void pop_lock_usec(ulonglong top);     // 弹栈（存储过程用）
```

`m_lock_usec` 在每次 `set_time()` 时被清零，保证每条语句的锁时间独立统计。

### 3.2 TABLE 级锁时间统计（MyISAM / 表锁）

位于 `sql/lock.cc:334-375`，`mysql_lock_tables()` 函数：

```cpp
ulonglong lock_start_usec = my_micro_time();   // 计时起点：在 lock_external() 之前

// lock_external() 逐表调用 ha_external_lock()
// 对 InnoDB：不阻塞，是事务注册和初始化
// 对 MyISAM：可能阻塞（fcntl 文件锁）

// thr_multi_lock() 是 MySQL 层表锁，有竞争时才阻塞等待

ulonglong lock_end_usec = my_micro_time();
thd->inc_lock_usec(lock_end_usec - lock_start_usec);  // 整个函数体时长都计入
```

此段时间涵盖（注意：不区分阻塞与非阻塞）：

- `lock_external()`：存储引擎外部锁（对 InnoDB 是无阻塞的初始化，对 MyISAM 可能阻塞）
- `thr_multi_lock()`：MySQL 层表级读写锁等待（有竞争时阻塞）

> 注意：计时范围不等于"等待时间"，详见第十一节的深度辨析。

### 3.3 InnoDB 行级锁时间统计

InnoDB 行锁等待通过插件接口上报，入口为 `sql/sql_thd_api.cc:371`：

```cpp
void thd_storage_lock_wait(MYSQL_THD thd, long long value) {
  thd->inc_lock_usec(value);   // value 由 InnoDB 计算并传入
}
```

`inc_lock_usec()` 的实现（`sql/sql_class.cc:3213`）带有详细注释：

```cpp
void THD::inc_lock_usec(ulonglong lock_usec) {
  /*
    If mysql_lock_tables() is called multiple times,
    we sum all the lock times here.
    This is the desired behavior, to know how much
    time was spent waiting on SQL tables.
    When Innodb reports additional lock time for DATA locks,
    it is counted as well.
  */
  m_lock_usec += lock_usec;
  MYSQL_SET_STATEMENT_LOCK_TIME(m_statement_psi, m_lock_usec);
}
```

注意：InnoDB 通过 `lock0wait.cc` 仅在 `QUE_THR_LOCK_ROW`（行锁）情形下调用此接口，InnoDB 表级锁等待不会触发该路径，也不计入 `Lock_time`。

> **运维意义**：慢日志中的 `Lock_time` 包含 `mysql_lock_tables()` 整体耗时与 InnoDB 行锁等待时间，但**不包含 MDL 等待时间**。如果 `Lock_time` 显著，说明存在行锁或表锁竞争；但 `Lock_time ≈ 0` 并不代表没有锁等待——MDL 阻塞时 `Lock_time` 仍近似为零。

### 3.4 存储过程中的锁时间隔离

调用存储过程时（`sql/sp_head.cc:2725-2928`），为防止过程内部语句的锁时间污染外层调用者的统计，使用栈机制隔离：

```cpp
// 进入存储过程前，保存并清零调用者的锁计数
ulonglong lock_usec_before_sp_exec;
thd->push_lock_usec(lock_usec_before_sp_exec);

// ... 执行存储过程内部所有语句 ...

// 退出存储过程后，恢复调用者的锁计数
thd->pop_lock_usec(lock_usec_before_sp_exec);
```

`push_lock_usec()` 的实现：

```cpp
void push_lock_usec(ulonglong &top) {
  top = m_lock_usec;   // 保存当前值
  m_lock_usec = 0;     // 清零，供 SP 内部语句独立统计
}
```

> **运维意义**：CALL 语句本身记录在慢日志中的 `Lock_time` 并不包含存储过程内部各语句的锁时间（各内部语句若超时会独立记录）。

---

## 四、`Query_time` 的计算

慢日志写入时（`sql/log.cc:1317-1323`）：

```cpp
if (thd->start_utime) {
  query_utime = (current_utime - thd->start_utime);  // 从 set_time() 到写日志的总耗时
  lock_utime  = thd->get_lock_usec();                // 累积的锁等待时间
} else {
  query_utime = 0;
  lock_utime  = 0;
}
```

`current_utime` 是调用 `slow_log_write()` 时的实时时钟，即：

```
Query_time = 记录日志时刻 - set_time() 时刻
           = 解析时间 + 优化时间 + 执行时间 + 锁等待时间 + 结果发送时间
```

**重要关系**：`Lock_time` 是 `Query_time` 的子集，`Lock_time ≤ Query_time`。

---

## 五、是否写入慢日志的决策逻辑

### 5.1 慢查询标记：`update_slow_query_status()`

执行完毕后（`sql/sql_class.cc:3228`）：

```cpp
void THD::update_slow_query_status() {
  if (my_micro_time() > start_utime + variables.long_query_time)
    server_status |= SERVER_QUERY_WAS_SLOW;
}
```

`long_query_time` 在内部以微秒存储（`long_query_time_double * 1e6`），精度为微秒级。

### 5.2 写入条件判断：`log_slow_applicable()`

`sql/log.cc:1592`，完整判断链如下：


| 条件                          | 不写入的情形                                        |
| --------------------------- | --------------------------------------------- |
| `in_sub_stmt`               | 触发器 / 存储函数内部语句不单独记录                           |
| `killed == KILL_CONNECTION` | 连接被强制中断时不记录                                   |
| `ER_PARSE_ERROR`            | SQL 语法解析错误不记录                                 |
| `SERVER_QUERY_WAS_SLOW`     | 超过 `long_query_time` 才记录（或无索引且开启相关选项）         |
| `min_examined_row_limit`    | 扫描行数不足阈值时不记录                                  |
| `opt_slow_log`              | 全局慢日志开关                                       |
| `enable_slow_log`           | 语句级开关（管理命令可被关闭）                               |
| `slave_thread`              | 副本线程默认不记录，除非 `log_slow_replica_statements=ON` |


触发写入的条件（满足其一即可）：

- `SERVER_QUERY_WAS_SLOW`（执行时间超过 `long_query_time`）
- `warn_no_index`（未使用索引 AND `log_queries_not_using_indexes=ON` AND 非 status 类命令）

两种情形下都会累加 `long_query_count` 状态变量（**即使慢日志总开关关闭，计数仍会增加**）。

---

## 六、日志记录内容详解

### 6.1 标准模式（默认）

```
# Time: 2025-03-28T10:00:01.123456Z
# User@Host: root[root] @ localhost [127.0.0.1]  Id: 42
# Query_time: 3.141592  Lock_time: 0.001234  Rows_sent: 100  Rows_examined: 500000
SET timestamp=1743156001;
SELECT * FROM orders WHERE ...;
```

**字段含义**：


| 字段               | 含义                               | 来源代码                            |
| ---------------- | -------------------------------- | ------------------------------- |
| `# Time:`        | 日志**写入**时间（查询结束时刻），非查询开始时间       | `current_utime`                 |
| `SET timestamp=` | 查询**开始**时间（Unix 秒），用于 binlog 一致性 | `query_start_utime / 1000000`   |
| `Query_time`     | 总执行耗时（秒，微秒精度）                    | `current_utime - start_utime`   |
| `Lock_time`      | 等锁总时间（TABLE 锁 + InnoDB 行锁）       | `thd->get_lock_usec()`          |
| `Rows_sent`      | 实际返回给客户端的行数                      | `thd->get_sent_row_count()`     |
| `Rows_examined`  | 存储引擎扫描的行数                        | `thd->get_examined_row_count()` |


### 6.2 扩展模式（`log_slow_extra=ON`）

开启后额外输出（`sql/log.cc:746-795`）：

```
# Query_time: ...  Lock_time: ...  Rows_sent: ...  Rows_examined: ...
# Thread_id: 42  Errno: 0  Killed: 0
# Bytes_received: 128  Bytes_sent: 8192
# Read_first: 1  Read_last: 0  Read_key: 500000  Read_next: 499999  Read_prev: 0
# Read_rnd: 0  Read_rnd_next: 500001
# Sort_merge_passes: 0  Sort_range_count: 0  Sort_rows: 100  Sort_scan_count: 1
# Created_tmp_disk_tables: 1  Created_tmp_tables: 2
# Start: 2025-03-28T10:00:01.000000Z  End: 2025-03-28T10:00:04.141592Z
```

扩展字段的值是通过对比语句执行前后的 `System_status_var` 快照差值得到（`sql/sql_parse.cc:1239-1241`）：

```cpp
if (opt_log_slow_extra) {
  thd->copy_status_var(&query_start_status);  // 语句开始时快照
}
// ... 执行 ...
// 写日志时用 thd->status_var - query_start_status 得到增量
```

---

## 七、`# Time:` 与 `SET timestamp=` 的区别

这是一个容易混淆的重要细节：

```
# Time: 2025-03-28T10:00:04.141592Z     ← 查询结束写日志的时刻
SET timestamp=1743156001;               ← 查询开始时刻（秒精度）
SELECT ...;
```

- `**# Time:**`：`current_utime`，即调用 `slow_log_write()` 的时刻，约等于查询结束时间
- `**SET timestamp=**`：`query_start_utime / 1000000`，即 `set_time()` 设置的 `start_utime` 转换为秒

两者之差就是 `Query_time`。`SET timestamp=` 的存在是为了在重放慢日志时，`NOW()`、`CURRENT_TIMESTAMP` 等函数能返回查询原本的开始时间，保证 binlog 重放的一致性。

---

## 八、无索引查询节流（`log_throttle_queries_not_using_indexes`）

对于仅因"未使用索引"触发的慢日志记录，MySQL 实现了时间窗口节流机制（`sql/log.cc:1836`）：

```cpp
Slow_log_throttle log_throttle_qni(
    &opt_log_throttle_queries_not_using_indexes,
    &LOCK_log_throttle_qni,
    Log_throttle::LOG_THROTTLE_WINDOW_SIZE,   // 60 秒窗口
    slow_log_write,
    "throttle: %10lu 'index not used' warning(s) suppressed.");
```

在 60 秒窗口内，超过 `log_throttle_queries_not_using_indexes` 条数的同类告警会被压制，并在下一条未被压制的日志中输出汇总（"本窗口共抑制了 N 条"）。

> **运维意义**：如果看到含 `throttle: xxx 'index not used' warning(s) suppressed` 的慢日志行，实际发生的未走索引查询数量远超日志中可见的数量。

---

## 九、输出目标：文件 vs 表

通过 `log_output` 系统变量控制，在 `sql/log.cc:1446-1462` 初始化处理器链：


| `log_output` | 行为                                |
| ------------ | --------------------------------- |
| `FILE`（默认）   | 写入 `slow_query_log_file` 指定的文件    |
| `TABLE`      | 写入 `mysql.slow_log` 表（CSV 引擎）     |
| `FILE,TABLE` | 同时写入两处                            |
| `NONE`       | 不写入任何地方（但 `long_query_count` 仍计数） |


`mysql.slow_log` 表的 `start_time` 字段存储的是 `current_utime`（查询**结束**时间），与文件格式的 `# Time:` 语义一致。

---

## 十、运维关键要点总结

### 10.1 时间语义精要


| 慢日志字段            | 代表时刻                            | 是否包含网络 I/O           |
| ---------------- | ------------------------------- | -------------------- |
| `# Time:`        | 查询**结束**（写日志时）                  | 不含                   |
| `SET timestamp=` | 查询**开始**（`dispatch_command` 入口） | 不含（网络读包在此之前）         |
| `Query_time`     | 服务器内部总耗时                        | 不含网络读，**含**结果集发送     |
| `Lock_time`      | 等待获取锁的总时间                       | 不含（是 Query_time 的子集） |


### 10.2 Lock_time 的真实含义

`Lock_time` = **TABLE 级锁等待** + **InnoDB 行级锁等待**，是两者的累加值。

- 若 `Lock_time / Query_time > 50%`：查询主要瓶颈是**锁竞争**，排查方向为事务大小、并发更新、死锁
- 若 `Lock_time` 接近 0 但 `Query_time` 很高：瓶颈在**扫描/排序/临时表**，排查方向为索引和执行计划

### 10.3 Rows_examined 与实际 I/O

`Rows_examined` 是存储引擎**调用层**的计数，而非物理 I/O 次数：

- 对于 InnoDB，命中 Buffer Pool 的行也会被计入
- 覆盖索引扫描和回表都会计入
- 是判断全表扫描/低效索引最直接的指标

### 10.4 副本延迟诊断

默认情况下副本线程的执行不会记入慢日志（`thd->slave_thread` 检查）。若需要诊断副本上 SQL 线程的执行性能，需开启：

```sql
SET GLOBAL log_slow_replica_statements = ON;
```

### 10.5 `long_query_count` 与慢日志开关的关系

```cpp
// sql/log.cc:1618
if (log_this_query) thd->status_var.long_query_count++;
```

`long_query_count`（即 `SHOW GLOBAL STATUS LIKE 'Slow_queries'` 的值）**在判断满足超时条件时就计数**，与 `slow_query_log` 开关无关。因此即使关闭了慢日志文件写入，通过监控 `Slow_queries` 增量仍可感知慢查询频率。

### 10.6 `log_slow_extra` 扩展字段的诊断价值

开启 `log_slow_extra=ON` 后新增字段的运维价值：


| 字段                        | 诊断价值                                        |
| ------------------------- | ------------------------------------------- |
| `Read_key`                | 索引查找次数，与 `Rows_examined` 组合判断索引选择性          |
| `Read_next` / `Read_prev` | 范围扫描行数，高值配合 `Rows_sent` 低值说明过滤率差            |
| `Sort_merge_passes`       | > 0 表示 filesort 溢出到磁盘，`sort_buffer_size` 不足 |
| `Created_tmp_disk_tables` | > 0 表示 tmp table 溢出，`tmp_table_size` 不足     |
| `Bytes_sent`              | 结果集大小，排查大结果集传输                              |
| `Start` / `End`           | 精确的查询起止时间戳（微秒级）                             |


---

## 十一、`Lock_time` 的精确含义深度辨析

### 11.1 结论先行

`Lock_time` **既不是纯粹的"等待锁的时间"，也不是完整的"所有锁等待时间"**。它存在两个方向的误差：一是包含了非等待的引擎初始化开销（多算），二是遗漏了 MDL 等待和 InnoDB 表级锁等待（少算）。

---

### 11.2 `mysql_lock_tables()` 计时：包含了非等待开销

`sql/lock.cc` 中，计时起点在调用 `lock_external()` **之前**：

```cpp
// sql/lock.cc:334-375
ulonglong lock_start_usec = my_micro_time();   // ← 计时已开始，尚未进入任何等待

if (sql_lock->table_count &&
    lock_external(thd, sql_lock->table, sql_lock->table_count)) {
  // lock_external() 逐表调用 ha_external_lock()
  // 对 InnoDB：这是事务注册和状态初始化，不是等锁
  ...
}

// thr_multi_lock() 是 MySQL 层表锁，有竞争时才阻塞
rc = thr_lock_errno_to_mysql[(int)thr_multi_lock(...)];

ulonglong lock_end_usec = my_micro_time();
thd->inc_lock_usec(lock_end_usec - lock_start_usec);  // 整段函数都计入
```

InnoDB 的 `ha_innobase::external_lock()`（`storage/innobase/handler/ha_innodb.cc:18624`）并不等待任何锁，其核心工作是：注册事务到 handlerton、设置 `select_lock_type`、调用 `TrxInInnoDB::begin_stmt(trx)`。即使完全没有锁竞争，这些初始化耗时也会被计入 `Lock_time`。

**实际效果**：对于 InnoDB 上的普通查询，即便无任何并发冲突，`Lock_time` 也会有几微秒到几十微秒的固定开销，反映的是引擎层正常的语句初始化时间，而非等待。

---

### 11.3 InnoDB 行锁等待：仅统计 row lock，table lock 被排除

InnoDB 向 MySQL 层上报锁等待时间的入口在 `storage/innobase/lock/lock0wait.cc`，但有严格的条件守卫：

```cpp
// lock0wait.cc:243-248
if (thr->lock_state == QUE_THR_LOCK_ROW) {   // 仅行锁才打开计时
  srv_stats.n_lock_wait_count.inc();
  start_time = std::chrono::steady_clock::now();
}

// ... os_event_wait(slot->event) 阻塞等待 ...

// lock0wait.cc:322-335
if (thr->lock_state == QUE_THR_LOCK_ROW) {   // 仅行锁才上报
  const auto diff_time = std::chrono::steady_clock::now() - start_time;
  // ...
  thd_set_lock_wait_time(trx->mysql_thd, diff_time);  // 最终写入 Lock_time
}
```

`QUE_THR_LOCK_TABLE`（InnoDB 表级锁等待，如 `LOCK TABLE ... IN EXCLUSIVE MODE` 发生冲突时）不满足这个条件，其等待时间**不会上报到 `Lock_time`**。

---

### 11.4 最重要的盲区：MDL 等待完全不在 `Lock_time` 中

这是对运维影响最大的问题。调用链顺序如下：

```
open_and_lock_tables()          [sql/sql_base.cc:6596-6605]
  ├─ open_tables()              ← MDL 在此获取（可能等待数秒乃至数分钟）
  └─ lock_tables()
       └─ mysql_lock_tables()   ← Lock_time 计时从这里才开始
```

`sql/mdl.cc` 中全文不含任何 `inc_lock_usec` 或 `thd_storage_lock_wait` 调用，MDL 等待与 `m_lock_usec` 完全无关。

**直接后果**：当 `ALTER TABLE`、`FLUSH TABLES WITH READ LOCK`、或长事务持有 MDL，导致后续 DML 在 MDL 队列中等待时：

- `Lock_time` ≈ 几微秒（因为等待期间根本没进入 `mysql_lock_tables()`）
- `Query_time` = 等待时间 + 实际执行时间（可能高达数十秒）
- 在慢日志中，这类查询看起来像"执行本身很慢，没有锁冲突"

这是仅靠慢日志无法诊断 MDL 阻塞的根本原因。

---

### 11.5 Lock_time 的完整构成对照表


| 来源                            | 是否计入 `Lock_time` | 说明                                                  |
| ----------------------------- | ---------------- | --------------------------------------------------- |
| `mysql_lock_tables()` 整体耗时    | **计入**           | 含无竞争时的 InnoDB 事务初始化开销，非纯等待                          |
| InnoDB 行级锁等待                  | **计入**           | `lock0wait.cc` 精确测量，仅限 `QUE_THR_LOCK_ROW`           |
| InnoDB 表级锁等待                  | **不计入**          | `QUE_THR_LOCK_TABLE` 路径不调用 `thd_set_lock_wait_time` |
| **MDL（元数据锁）等待**               | **不计入**          | `open_tables()` 阶段，`mdl.cc` 无任何相关调用                 |
| MySQL 层 `thr_multi_lock` 表锁等待 | **计入**           | 在 `mysql_lock_tables()` 内部，属于计时范围                   |
| MyISAM `fcntl` 文件锁等待          | **计入**           | `lock_external()` 内调用 `ha_external_lock()`，在计时范围内   |


---

### 11.6 运维诊断指南（修正版）

**场景一：`Lock_time` 显著，`Query_time ≈ Lock_time`**

InnoDB 行锁竞争为主因。排查方向：事务持锁时间过长、高并发 UPDATE/DELETE 同一行、死锁频发。

**场景二：`Lock_time ≈ 0`，但 `Query_time` 异常高**

不能得出"没有锁等待"的结论。需检查：

- `SHOW PROCESSLIST` 中是否有 `Waiting for table metadata lock` 状态
- `performance_schema.events_waits_history` 中的 MDL 等待事件
- 是否有 DDL（`ALTER TABLE`、`TRUNCATE`）或 `FLUSH TABLES` 正在执行

**场景三：`Lock_time` 极小（< 1ms），`Query_time` 也较正常**

`Lock_time` 中包含的是 InnoDB 正常的语句初始化开销，属于正常现象，无需关注。

**正确理解 `Lock_time` 的上界和下界：**

```
实际等锁时间下界 ≈  Lock_time - (mysql_lock_tables 非阻塞开销)  ≈  InnoDB 行锁等待
实际等锁时间上界 ≥  Lock_time + MDL等待时间 + InnoDB表级锁等待时间
```

慢日志的 `Lock_time` 字段仅是锁相关时间的一个局部视图，完整的锁等待分析必须结合 `performance_schema` 的 `data_locks`、`metadata_locks` 等表。

---

## 十二、历史演进：`Lock_time` 是否计入慢查询阈值的变迁

### 12.1 旧设计：`utime_after_lock`（MySQL ≤ 8.0.26）

在 MySQL 8.0.26 及以前的所有版本（5.1 / 5.5 / 5.6 / 5.7 / 8.0 早期），`THD` 类中存在一个名为 `utime_after_lock` 的时间戳变量，其设计图如开发者在 Bug#53191（2010年）注释中所画：

```
   ----- start_utime          ← dispatch_command() 入口，set_time() 设置
 l    .
 o    .  mysql_lock_tables() 等待期间
 c    .
 k    .
   ----- utime_after_lock     ← mysql_lock_tables() 结束时设置
 q    .
 u    .  真正的 SQL 执行期间
 e    .
 r    .
    ------ current_time

Lock_time  = utime_after_lock - start_utime   ← 含 table 锁 + InnoDB 行锁
Query_time = current_time - utime_after_lock   ← 纯执行时间，不含锁等待
```

旧版 `update_slow_query_status()` 使用的判断为：

```cpp
// 旧代码（8.0.26 及以前）
if (my_micro_time() > utime_after_lock + variables.long_query_time)
  server_status |= SERVER_QUERY_WAS_SLOW;
```

**关键效果**：`long_query_time` 阈值与 `utime_after_lock` 比较，意味着**锁等待时间被完全排除在慢查询判定之外**。一条因等锁而耗时 10 秒但 SQL 本身执行仅 1 毫秒的查询，在 `long_query_time=1` 时**不会**进入慢日志。

这一设计有明确的官方文档依据，MySQL 5.7 手册原文：

> "The time to acquire the initial locks is not counted as execution time."

InnoDB 行锁等待的处理也配合此机制，`thd_storage_lock_wait()` 通过 `utime_after_lock += value` 将时间戳向后推移，使阈值比较始终以"最后一次锁结束"为起点，确保纯执行时间始终被排除在外。

---

### 12.2 历史 Bug：锁等待导致慢查询但不记录

此设计导致一个长期被用户投诉的行为——**等锁的慢查询无法被慢日志捕捉**。

**Bug #65139（2012年4月，MySQL 5.5.21）**

```
S1: LOCK TABLES t1 WRITE;
S2: SELECT * FROM t1;       ← 等待 10 秒后获得锁，执行 <1ms
S1: UNLOCK TABLES;
结果：S2 的查询在 long_query_time=1 时 【不出现】在慢日志中
```

MySQL 开发者的回复：*"current behavior is intended and explained in the manual"*

**Bug #101142（2020年，MySQL 5.7.31 / 8.0.20）**  
同样场景，用户再次报告。MySQL 验证团队回复：

> "The purpose of the slow query log is to catch the queries that are not optimised. Hence, it took us a lot of time to implement the feature, that was demanded by so many users, which is **not to count lock time as a part of the query execution time**."

此时（2020年，8.0.20）该行为仍被官方视为"设计如此"。

---

### 12.3 转折点：BUG#33236909，MySQL 8.0.28（2022年1月）

GitHub 提交 `3e7f875a`（2021-09-13）彻底重构了这一机制，正式发布于 **MySQL 8.0.28**（2022年1月18日发布，官方 Release Notes 确认）。

**问题背景**：Performance Schema 的 `LOCK_TIME` 与慢日志的 `Lock_time` 数值不一致，且 `utime_after_lock` 多次被修改导致代码难以维护。

**核心变更**（`sql/sql_class.cc` diff）：

```diff
 // set_time() 中：
-  start_utime = utime_after_lock = my_micro_time();
+  start_utime = my_micro_time();
+  m_lock_usec = 0;

 // update_slow_query_status() 中：
-  if (my_micro_time() > utime_after_lock + variables.long_query_time)
+  if (my_micro_time() > start_utime + variables.long_query_time)
     server_status |= SERVER_QUERY_WAS_SLOW;

 // set_time_after_lock() 被删除，替换为：
+  void inc_lock_usec(ulonglong lock_usec) { m_lock_usec += lock_usec; ... }
```

`**sql/sql_class.h` diff**：

```diff
-  ulonglong start_utime, utime_after_lock;
+  ulonglong start_utime;
+  ulonglong m_lock_usec;   ← 累计锁等待时间（微秒）
```

`**sql/sql_thd_api.cc` diff（InnoDB 行锁上报接口）**：

```diff
 void thd_storage_lock_wait(MYSQL_THD thd, long long value) {
-  thd->utime_after_lock += value;
+  thd->inc_lock_usec(value);
 }
```

**行为变化**：阈值判断从 `current_time - utime_after_lock` 改为 `current_time - start_utime`，意味着：

- **8.0.27 及以前**：锁等待时间不计入 `long_query_time` 阈值，等锁慢查询不记录
- **8.0.28 及以后**：总耗时（含锁等待）计入阈值，等锁慢查询**会被正确记录**

---

### 12.4 遗留问题：Lock_time 字段仍不反映 MDL 等待

8.0.27 修复了"等锁慢查询不记录"的问题，但引入了新的混淆：`Lock_time` 字段的值与直觉不符。

**Bug #115841（2024年8月，MySQL 8.0.39 / 8.4.2）** 复现了如下场景：

```sql
-- Session 1:
LOCK TABLE t WRITE;      -- 持有 MDL_SHARED_NO_READ_WRITE

-- Session 2 (long_query_time=0):
SELECT * FROM t;         -- 在 open_tables() 阶段等待 MDL，等待 5 秒后获取
```

慢日志结果（来自 Bug 报告原文）：

```
# Query_time: 5.001421  Lock_time: 0.000019  Rows_sent: 0  Rows_examined: 0
SELECT * FROM t;
```

**分析**：

- `Query_time = 5.001421s`：正确，因为 `start_utime` 在 `open_tables()` 之前设置，总耗时包含 MDL 等待
- `Lock_time = 0.000019s`：极小，MDL 等待发生在 `open_tables()` 中，而 `m_lock_usec` 仅由 `mysql_lock_tables()` 内的 `inc_lock_usec()` 和 InnoDB 行锁的 `thd_storage_lock_wait()` 更新，两者都不覆盖 MDL

这个 Bug 在 MySQL 8.0.39 和 8.4.2 均已被验证为真实问题，截至 8.0.41 尚未修复。

---

### 12.5 三阶段完整演进总结


| 阶段       | 版本范围      | 机制                     | 阈值判断                                           | 等锁查询是否记录          | Lock_time 含义                                               |
| -------- | --------- | ---------------------- | ---------------------------------------------- | ----------------- | ---------------------------------------------------------- |
| **旧设计**  | ≤ 8.0.27  | `utime_after_lock` 时间戳 | `current - utime_after_lock > long_query_time` | **不记录**           | `utime_after_lock - start_utime`（含 table 锁+行锁，排除 MDL）      |
| **新设计**  | ≥ 8.0.28  | `m_lock_usec` 累加器      | `current - start_utime > long_query_time`      | **记录**            | `m_lock_usec`（仅 mysql_lock_tables 开销+InnoDB 行锁）            |
| **遗留缺陷** | 8.0.28–现在 | —                      | —                                              | 记录，但 Lock_time 误导 | MDL 等待**不计入** Lock_time，造成 Query_time 高而 Lock_time ≈ 0 的假象 |


**对运维的关键启示**：

- **MySQL 8.0.27 及以前**：慢日志可能遗漏大量因锁竞争导致的慢查询，不能用"没有慢日志"来说明"没有锁问题"
- **MySQL 8.0.28 及以后**：慢日志能捕捉锁等待慢查询，但 `Lock_time ≈ 0` 并非意味着没有锁等待——MDL 等待被纳入了 `Query_time` 但未计入 `Lock_time`

---

## 十三、`utime_after_lock` 原始设计动机：为何锁等待曾被排除在外

### 13.1 核心哲学：慢日志是"查询优化工具"，不是"运维监控工具"

慢日志在 MySQL 早期（3.x/4.x 时代）被设计出来时，有一个非常明确的单一目标：

> **找出执行效率差的 SQL，引导 DBA 去优化查询本身。**

在这个定义下，`utime_after_lock` 的设计逻辑是完全自洽的。查询变慢的根本原因只有两类，而设计者认为慢日志只应关注第一类：

```
一条查询"慢"的原因分两类：

1. 查询本身效率差  ← 慢日志要抓的
   - 全表扫描、没走索引、排序、临时表...
   - 换个时间单独跑也一样慢
   - 解决方案：加索引、改写 SQL

2. 外部并发造成等待  ← 慢日志认为"不归我管"
   - 等别的 session 释放锁
   - 并发低的时段跑就很快
   - 解决方案：架构调整、事务拆分、减少热点竞争
```

---

### 13.2 MyISAM 表锁时代的现实压力

`utime_after_lock` 诞生于 MyISAM 统治的年代，而 MyISAM 是**表级锁**。这给设计者带来了一个非常现实的困境——**同一条 SQL 是否"慢"，完全取决于外部并发，与 SQL 质量无关**：

```
SELECT * FROM orders WHERE id = 1;   -- 走主键，本质执行 0.1ms

情形 A：系统空闲，无并发写入
  Lock_time = 0s,   Query_time = 0.0001s  → 不算慢 ✓

情形 B：另一 session 正在大批量 INSERT，持有表写锁
  等锁 8s，实际执行 0.1ms

  旧设计（排除锁时间）：Query_time = 0.0001s → 不入慢日志 ✓
                        这条 SQL 没有问题，不该被标记
  若纳入锁时间：        Query_time = 8.0001s → 入慢日志 ✗
                        产生噪音，DBA 对此无能为力
```

如果把锁等待算进去，同一条已经完美优化的 SQL 在高并发时会**持续淹没慢日志**。DBA 对这些记录什么也做不了——SQL 已经最优，锁竞争是架构问题，不是 SQL 问题。这会让慢日志**丧失实用价值**。

Bug #17193（2006 年，最早的相关 Feature Request）中，开发者的回复措辞也印证了这一思路：

> _"developers are already working on some way to give you **information about** lock waits for slow queries"_

注意用词——他们考虑的是"提供锁等待信息"（即记录 `Lock_time` 字段），而不是"把锁等待纳入阈值判断"。**记录信息** 和 **触发阈值** 被刻意分开处理。

---

### 13.3 `Lock_time` 字段存在的意义：解释注脚，而非触发器

这解释了一个表面上的矛盾：锁时间不触发阈值，却仍然被记录在日志行里。这是刻意设计的**补充诊断信息**：

```
情形 1：
# Query_time: 5.2s   Lock_time: 0.001s   Rows_examined: 500000
→ 该查询因本身执行慢而进入慢日志
  Lock_time ≈ 0 告诉你："排除锁的干扰，这条 SQL 确实很慢，去看执行计划"

情形 2：
# Query_time: 5.2s   Lock_time: 4.9s    Rows_examined: 10
→ 该查询因本身执行慢（> long_query_time）而进入慢日志
  Lock_time 高告诉你："实际执行只用了 0.3s，大部分时间是等锁，
                       Query_time 要打折扣，优化方向是锁竞争而非 SQL"
```

`Lock_time` 是对 `Query_time` 的解释注脚——帮助 DBA 判断慢的根源，而非独立的触发条件。

---

### 13.4 为什么 InnoDB 时代这个设计开始动摇

到了 InnoDB 成为主流引擎之后，"锁等待"的语义发生了根本变化，原有设计的合理性逐渐受到质疑：

| 维度 | MyISAM 表锁时代 | InnoDB 行锁时代 |
|------|--------------|--------------|
| 锁等待触发条件 | 任意并发写操作都会阻塞全表 | 精确到行，仅争同一行才阻塞 |
| 能否通过 SQL 优化缓解 | 几乎不能，是并发架构问题 | 有时可以（缩小事务、避免热点行） |
| 锁等待与查询质量的关系 | 完全无关 | 存在一定相关性 |
| 对客户端的体感影响 | 等锁期间客户端一样超时 | 等锁期间客户端一样超时 |

最后一条是 8.0.28 改变设计的根本驱动力：**不管什么原因，客户端等待了 10 秒就是等待了 10 秒**。从业务可用性视角看，"慢查询"的定义应该是"让用户感受到慢的查询"，而不只是"SQL 本身执行效率低的查询"。

此外，随着 Performance Schema 日益完善，锁等待分析已经有了更专业的工具（`data_locks`、`metadata_locks`、`events_waits_history` 等），慢日志也不必再严守"只做优化工具"的单一定位，可以同时服务于运维监控。

---

### 13.5 设计演进的本质

`utime_after_lock` 的兴衰，本质上是 **MySQL 对"慢查询"定义的演进**：

```
旧定义（MyISAM 时代）：
  慢查询 = "SQL 本身执行慢"
  → 锁等待是外部因素，排除在外
  → 慢日志 = 优化工具

新定义（InnoDB + 云原生时代）：
  慢查询 = "让客户端等待超过阈值的任何查询"
  → 锁等待也是客户端等待的一部分
  → 慢日志 = 优化工具 + 运维监控工具
```

这一转变并非颠覆，而是扩展。`Lock_time` 字段从"解释注脚"升格为"也参与触发"，慢日志的能力边界得到了扩充，但原有的 SQL 优化诊断功能并未削弱。

---

## 十四、核心源码文件索引


| 功能                      | 文件                                      | 关键行              |
| ----------------------- | --------------------------------------- | ---------------- |
| 计时开始 `set_time()`       | `sql/sql_class.cc`                      | 3200             |
| TABLE 锁时间统计             | `sql/lock.cc`                           | 334, 374-375     |
| InnoDB 行锁时间上报接口         | `sql/sql_thd_api.cc`                    | 371-373          |
| 锁时间累加实现                 | `sql/sql_class.cc`                      | 3213-3225        |
| 存储过程锁时间隔离               | `sql/sp_head.cc`                        | 2725-2726, 2928  |
| 慢查询标记                   | `sql/sql_class.cc`                      | 3228-3231        |
| 写入条件判断                  | `sql/log.cc`                            | 1592-1630        |
| Query_time 计算           | `sql/log.cc`                            | 1317-1323        |
| 文件格式写入                  | `sql/log.cc`                            | 682-855          |
| 表格式写入                   | `sql/log.cc`                            | 983-1152         |
| 无索引节流                   | `sql/log.cc`                            | 1836-1841        |
| InnoDB 行锁等待计时           | `storage/innobase/lock/lock0wait.cc`    | 243-248, 322-335 |
| InnoDB 锁等待时间上报          | `storage/innobase/handler/ha_innodb.cc` | 1988-1994        |
| InnoDB external_lock 实现 | `storage/innobase/handler/ha_innodb.cc` | 18624            |


