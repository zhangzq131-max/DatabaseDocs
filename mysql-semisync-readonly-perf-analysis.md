# MySQL 8.0 半同步复制插件导致只读事务性能瓶颈深度分析

> 源码版本：MySQL 8.0.41  
> 分析依据：perf 火焰图 + 源码追踪  
> 报告日期：2026-03-31

---

## 一、问题现象

在安装了 `semisync_master.so` / `semisync_slave.so`（即 `rpl_semi_sync_source` / `rpl_semi_sync_replica`）插件的实例上，纯只读 SELECT 事务的吞吐量严重下降。perf 抓取的热点调用栈如下：

```
worker_main
  └─ do_command
       └─ dispatch_command
            └─ launch_hook_trans_begin          ← 占 38% CPU
                 └─ Trans_delegate::trans_begin
                      └─ plugin_lock            ← 占 23% CPU
                           └─ futex_wait        ← 占 17%
                                └─ native_queued_spin_lock_slowpath  ← 占 30%
```

关键数字：**每一条打到用户表的 SELECT，都因等待全局互斥锁 `LOCK_plugin` 而阻塞，且争用随并发线程数线性恶化。**

---

## 二、完整调用链源码追踪

### 2.1 入口：`dispatch_command` → `launch_hook_trans_begin`

每条 SQL 在完成解析后，`mysql_execute_command`（`sql/sql_parse.cc:3182`）会无条件调用：

```3182:3193:sql/sql_parse.cc
    int ret = launch_hook_trans_begin(thd, all_tables);
    if (ret) {
      my_error(ret, MYF(0));
      return -1;
    }

  } else {
    int ret = launch_hook_trans_begin(thd, all_tables);
    if (ret) {
      my_error(ret, MYF(0));
      return -1;
    }
```

**该函数对 slave 线程和普通连接线程均生效，没有针对只读路径的快速跳出。**

### 2.2 `launch_hook_trans_begin` 的过滤逻辑

```1385:1488:sql/rpl_handler.cc
int launch_hook_trans_begin(THD *thd, Table_ref *all_tables) {
  // ...
  // 已在同一事务内调用过则直接返回
  if (thd->get_transaction()->was_trans_begin_hook_invoked()) {
    return 0;
  }

  // 跳过纯管理命令：SHOW / SET / EMPTY / USE / KILL / SHUTDOWN ...
  if ((is_set || is_show || is_empty || is_use || is_stop_gr || is_shutdown ||
       is_reset_persist || is_kill) &&
      !lex->uses_stored_routines()) {
    return 0;
  }

  // 对于 SELECT / DO，只有无表或仅查 perf_schema/info_schema/sys 才可跳过
  if (is_query_block) {
    // ...
    if (!is_udf && all_tables == nullptr) {
      hold_command = false;   // 无表 SELECT 可跳过
    }
    if (is_process_list || is_perf_schema_table || is_sys_db) {
      hold_command = false;   // 系统库查询可跳过
    }
  }

  // 凡是 hold_command == true，就触发 hook
  if (hold_command) {
    RUN_HOOK(transaction, trans_begin, (thd, ret));
    // ...
  }
  return ret;
}
```

**结论：任何访问用户业务表的 `SELECT`（包括只读事务）都不会被提前跳过，`hold_command` 保持 `true`，`RUN_HOOK` 必然执行。**

### 2.3 `RUN_HOOK` 展开

```458:462:sql/rpl_handler.h
#define RUN_HOOK(group, hook, args) \
  (group##_delegate->is_empty() ? 0 : group##_delegate->hook args)
```

只要 `transaction_delegate` 的观察者列表非空（即 semisync 插件已注册），就会调用：

```854:876:sql/rpl_handler.cc
int Trans_delegate::trans_begin(THD *thd, int &out) {
  // ...
  Trans_param param;
  TRANS_PARAM_ZERO(param);
  param.server_uuid = server_uuid;
  param.thread_id   = thd->thread_id();
  // ...

  int ret = 0;
  FOREACH_OBSERVER_ERROR_OUT(ret, begin, &param, out);
  return ret;
}
```

### 2.4 `FOREACH_OBSERVER_ERROR_OUT` 宏 — 锁的核心所在

```498:546:sql/rpl_handler.cc
#define FOREACH_OBSERVER_ERROR_OUT(r, f, args, out)                    \
  r = 0;                                                               \
  Prealloced_array<plugin_ref, 8> plugins(PSI_NOT_INSTRUMENTED);       \
  read_lock();          /* ① 获取 Delegate 的 OS rwlock 读锁 */        \
  Observer_info_iterator iter = observer_info_iter();                  \
  Observer_info *info = iter++;                                        \
  bool replication_optimize_for_static_plugin_config =                 \
      this->use_spin_lock_type();                                      \
  for (; info; info = iter++) {                                        \
    plugin_ref plugin = (replication_optimize_for_static_plugin_config \
                             ? info->plugin                            \
                             : my_plugin_lock(0, &info->plugin));      \
    /* ② 默认路径：调用 plugin_lock，内部加全局互斥锁 LOCK_plugin */   \
    // ...                                                             \
    hook_error = ((Observer *)info->observer)->f(args, error_out);     \
    /* ③ 调用 semisync 的 begin hook（见 2.5 节，是空函数！）*/         \
  }                                                                    \
  unlock();                                                            \
  if (!plugins.empty()) plugin_unlock_list(...);  /* ④ 再次加 LOCK_plugin */
```

### 2.5 semisync source 的 `begin` hook 是一个空函数

```124:124:plugin/semisync/semisync_source_plugin.cc
static int repl_semi_report_begin(Trans_param *, int &) { return 0; }
```

`repl_semi_report_begin` **什么业务逻辑都没有**，直接返回 0。  
但框架层在调用它之前，**必须先完成 `plugin_lock` 的引用计数增减操作**。

### 2.6 `plugin_lock` — 全局 `LOCK_plugin` 互斥锁

```975:983:sql/sql_plugin.cc
plugin_ref plugin_lock(THD *thd, plugin_ref *ptr) {
  LEX *lex = thd ? thd->lex : nullptr;
  plugin_ref rc;
  DBUG_TRACE;
  mysql_mutex_lock(&LOCK_plugin);   // ← 全局唯一互斥锁！
  rc = my_intern_plugin_lock_ci(lex, *ptr);
  mysql_mutex_unlock(&LOCK_plugin);
  return rc;
}
```

`LOCK_plugin` 是 MySQL 中**最重量级的全局互斥锁之一**，保护整个插件数组的引用计数和状态变更。它在以下场景都会被竞争：
- 任何调用 `plugin_lock` / `plugin_unlock` 的线程
- `INSTALL PLUGIN` / `UNINSTALL PLUGIN`
- `SHOW PLUGINS`
- 以及每一条触发 observer hook 的 SQL

---

## 三、锁竞争完整因果链

```
每条 SELECT（含只读事务）
  │
  ▼
launch_hook_trans_begin()              [sql/sql_parse.cc:3182]
  │  hold_command = true（访问用户表）
  ▼
RUN_HOOK(transaction, trans_begin)     [sql/rpl_handler.h:458]
  │  transaction_delegate 非空（semisync 已注册观察者）
  ▼
Trans_delegate::trans_begin()          [sql/rpl_handler.cc:854]
  │
  ▼
FOREACH_OBSERVER_ERROR_OUT 宏展开
  │
  ├─ read_lock()                        ← Delegate OS rwlock 读锁
  │     默认模式(opt_replication_optimize_for_static_plugin_config=false)
  │     → mysql_rwlock_rdlock(&lock)
  │     → 读写锁本身存在内核态竞争
  │
  └─ my_plugin_lock(0, &info->plugin)  ← 对每个 Observer 各调用一次
       │
       ▼
       plugin_lock()                   [sql/sql_plugin.cc:975]
         mysql_mutex_lock(&LOCK_plugin)  ← 全局唯一互斥锁
           │  高并发下大量线程堆积
           ▼
           futex_wait
             └─ native_queued_spin_lock_slowpath   ← perf 观测到的热点
                                 （CPU 在此空转/休眠等待锁）

调用 repl_semi_report_begin()          ← 空函数，什么都不做，return 0
  ↑
  └── 付出了上述全部锁开销，却得到零收益
```

**perf 中占比 38% 的 `launch_hook_trans_begin` 开销几乎全部来自锁竞争，而不是 semisync 的业务逻辑。**

---

## 四、为什么只读事务受影响最严重

| 维度 | 写事务 | 只读事务 |
|------|--------|---------|
| 真正的 semisync 等待点 | `before_commit` → 等待 ACK，耗时 ms 级 | **无**（begin hook 是空函数） |
| `LOCK_plugin` 持有时间 | 极短（`plugin_lock` 本身很快） | 同上 |
| 锁竞争占总耗时比例 | 被 ACK 等待稀释，占比低 | **全部开销都是锁竞争，占比极高** |
| 事务本身执行时间 | 通常更长（有写入、redo log 等） | 通常极短（μs 级点查） |

只读事务越短，`LOCK_plugin` 的获取时间占该事务总耗时的比例就越高。在高并发场景下，大量短事务同时争抢同一把全局锁，形成**漏斗效应**，QPS 上不去。

---

## 五、两个相关优化参数的工作原理

MySQL 8.0 提供了两个系统变量来缓解此问题：

### 5.1 `replication_optimize_for_static_plugin_config`

```78:78:sql/rpl_handler.cc
bool opt_replication_optimize_for_static_plugin_config{false};
```

**默认值：`OFF`（即当前触发瓶颈的模式）**

| 状态 | 锁类型 | plugin_lock 调用 |
|------|--------|-----------------|
| OFF（默认） | OS `mysql_rwlock_t` | 每次 hook **都调用** `plugin_lock` → 争抢 `LOCK_plugin` |
| ON | 无锁化 `Shared_spin_lock`（原子操作） | **不调用** `plugin_lock`，提前持久化引用计数 |

开启后，`FOREACH_OBSERVER` 宏中：

```465:468:sql/rpl_handler.cc
  for (; info; info = iter++) {
    plugin_ref plugin = (replication_optimize_for_static_plugin_config
                             ? info->plugin                  // ← 直接用缓存引用，无锁
                             : my_plugin_lock(0, &info->plugin));  // ← 默认路径，加锁
```

**代价：** 开启后 `UNINSTALL PLUGIN` 会被推迟到变量重置为 OFF 之后才能执行，适合插件配置稳定的场景。

### 5.2 `replication_sender_observe_commit_only`

```79:79:sql/rpl_handler.cc
std::atomic<bool> opt_replication_sender_observe_commit_only{false};
```

**默认值：`OFF`**

开启后，复制 observer 的所有 hook 调用（除 `after_commit`）都被跳过。`trans_begin` hook 也在跳过范围内，可以从根源绕过此路径。

---

## 六、根本原因总结

| # | 根因 | 所在代码 |
|---|------|---------|
| 1 | `launch_hook_trans_begin` 对所有访问用户表的 SELECT 无差别触发，包括只读事务 | `sql/sql_parse.cc:3182` |
| 2 | `semisync_source` 注册了 `Trans_observer`，导致 `transaction_delegate` 非空，`RUN_HOOK` 必须执行 | `plugin/semisync/semisync_source_plugin.cc:403` |
| 3 | `repl_semi_report_begin` 是完全空的函数（`return 0`），毫无业务价值，却强迫框架走完整 hook 路径 | `plugin/semisync/semisync_source_plugin.cc:124` |
| 4 | 默认配置下 `FOREACH_OBSERVER` 为每个 observer 调用 `plugin_lock`，内部加全局互斥锁 `LOCK_plugin` | `sql/sql_plugin.cc:979` |
| 5 | `LOCK_plugin` 是全局单点锁，N 个并发只读线程对其产生 N 倍争用，退化为串行 | `sql/sql_plugin.cc:979` |

---

## 七、优化建议

### 方案 A：开启静态插件配置优化（推荐，立竿见影）

```sql
SET GLOBAL replication_optimize_for_static_plugin_config = ON;
```

- **效果：** 彻底消除 `plugin_lock` 对 `LOCK_plugin` 的竞争，改用无锁的原子共享自旋锁
- **适用场景：** semisync 插件长期稳定加载，不动态卸载的生产环境
- **副作用：** `UNINSTALL PLUGIN` 被 defer，需先将此变量设回 OFF

### 方案 B：跳过非 commit hook（适合主库）

```sql
SET GLOBAL replication_sender_observe_commit_only = ON;
```

- **效果：** `trans_begin` hook 完全不触发，`launch_hook_trans_begin` 路径提前返回
- **适用场景：** 主库场景，只关心 commit 时的半同步等待

### 方案 C：两者同时开启（最优）

```sql
SET GLOBAL replication_optimize_for_static_plugin_config = ON;
SET GLOBAL replication_sender_observe_commit_only = ON;
```

对只读事务来说，`replication_sender_observe_commit_only` 可以让 `launch_hook_trans_begin` 提前返回（跳过整个 observer 遍历），`replication_optimize_for_static_plugin_config` 保障写路径也不受 `LOCK_plugin` 拖累。

### 方案 D：业务层——区分读写实例

- 只读实例（Replica）不需要加载 `semisync_source`（master 插件），应只加载 `semisync_replica`
- `semisync_replica` 不注册 `Trans_observer`，`transaction_delegate->is_empty()` 为 true，`RUN_HOOK` 直接返回 0，整条 hook 路径开销接近零

### 方案 E（长期）：内核侧优化

在 `launch_hook_trans_begin` 的 `is_query_block` 分支中增加只读事务快速路径：
- 若事务为只读（`thd->tx_read_only` 或 `autocommit=1`）且 semisync 仅关注写事务，可在此处提前 `return 0`
- 或者在 `FOREACH_OBSERVER` 宏中，先通过无锁方式检查 observer 的 `begin` 函数指针是否为默认空实现再决定是否调用 `plugin_lock`

---

## 八、各方案效果对比

| 方案 | 消除 `LOCK_plugin` 争用 | 消除 `Delegate rwlock` 争用 | 需要重启 | 适用范围 |
|------|------------------------|---------------------------|---------|---------|
| A（静态配置优化） | ✅ 完全消除 | ✅ 改为无锁自旋 | 否 | 主/备均可 |
| B（只观测 commit） | ✅ hook 不调用 | ✅ hook 不调用 | 否 | 主库 |
| C（A+B） | ✅ | ✅ | 否 | 主库最优 |
| D（备库不装 source 插件） | ✅ 天然避免 | ✅ 天然避免 | 是（卸载插件） | 只读备库 |
| 不装 semisync 插件 | ✅ | ✅ | 是 | 无半同步需求时 |

---

## 九、附：关键源码文件索引

| 文件 | 关键内容 |
|------|---------|
| `sql/sql_parse.cc:3182` | `launch_hook_trans_begin` 调用点 |
| `sql/rpl_handler.cc:1385` | `launch_hook_trans_begin` 完整实现，含 SELECT 过滤逻辑 |
| `sql/rpl_handler.cc:854` | `Trans_delegate::trans_begin` 实现 |
| `sql/rpl_handler.cc:498` | `FOREACH_OBSERVER_ERROR_OUT` 宏，`my_plugin_lock` 调用点 |
| `sql/rpl_handler.h:56-268` | `Delegate` 基类，两种锁类型的切换逻辑 |
| `sql/sql_plugin.cc:975` | `plugin_lock` → `LOCK_plugin` 互斥锁 |
| `plugin/semisync/semisync_source_plugin.cc:124` | `repl_semi_report_begin` 空函数 |
| `plugin/semisync/semisync_source_plugin.cc:403` | `Trans_observer` 注册表 |
