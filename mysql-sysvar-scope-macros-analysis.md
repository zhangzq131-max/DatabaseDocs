# MySQL 8.0.41 系统变量作用域宏分析

## 概述

在 MySQL 8.0.41 源码中，每一个系统变量（System Variable）在注册时都必须声明其**存储位置**和**可访问范围**。源码通过三个宏 `GLOBAL_VAR`、`SESSION_VAR`、`SESSION_ONLY` 来完成这一声明，它们定义在 `sql/sys_vars.h` 中。

```cpp
// sql/sys_vars.h, line 118-126
#define GLOBAL_VAR(X)                                                         \
  sys_var::GLOBAL, (((const char *)&(X)) - (char *)&global_system_variables), \
      sizeof(X)

#define SESSION_VAR(X)                             \
  sys_var::SESSION, offsetof(System_variables, X), \
      sizeof(((System_variables *)0)->X)

#define SESSION_ONLY(X)                                 \
  sys_var::ONLY_SESSION, offsetof(System_variables, X), \
      sizeof(((System_variables *)0)->X)
```

这三个宏展开后传递给 `sys_var` 基类构造函数的前三个参数为：

| 参数位置 | 含义 |
|---------|------|
| 第一个参数 | 作用域标志（`flag_enum` 枚举值） |
| 第二个参数 | 变量值在内存中的偏移量（`ptrdiff_t offset`） |
| 第三个参数 | 变量值的字节大小（`size_t size`） |

---

## 作用域标志定义

`sys_var` 类中定义了以下作用域相关的枚举标志（`sql/set_var.h`）：

```cpp
// sql/set_var.h, line 126-130
enum flag_enum {
  GLOBAL       = 0x0001,
  SESSION      = 0x0002,
  ONLY_SESSION = 0x0004,
  SCOPE_MASK   = 0x03FF,
  // ...
};
```

`check_scope()` 方法决定了一个变量在不同操作类型下是否可访问：

```cpp
// sql/set_var.h, line 311-323
bool check_scope(enum_var_type query_type) {
  switch (query_type) {
    case OPT_PERSIST:
    case OPT_PERSIST_ONLY:
    case OPT_GLOBAL:
      return scope() & (GLOBAL | SESSION);       // 0x0003
    case OPT_SESSION:
      return scope() & (SESSION | ONLY_SESSION); // 0x0006
    case OPT_DEFAULT:
      return scope() & (SESSION | ONLY_SESSION); // 0x0006
  }
  return false;
}
```

---

## 三个宏的详细对比

### 1. `GLOBAL_VAR(X)` — 纯全局变量

#### 标志值
`sys_var::GLOBAL`（`0x0001`）

#### 存储位置
变量 `X` 是一个**独立的全局 C++ 变量**（通常是 `extern` 声明的全局变量），其内存偏移量相对于 `global_system_variables` 结构体基地址计算：

```cpp
(((const char *)&(X)) - (char *)&global_system_variables)
```

> `global_system_variables` 是 `System_variables` 类型的全局实例，声明于 `sql/mysqld.h`：
> ```cpp
> extern MYSQL_PLUGIN_IMPORT struct System_variables global_system_variables;
> ```

#### 作用域检查

| 操作 | `scope() & mask` | 结果 |
|------|-----------------|------|
| `SET GLOBAL` / `PERSIST` | `0x0001 & 0x0003 = 0x0001` | ✅ 允许 |
| `SET SESSION` / `SET DEFAULT` | `0x0001 & 0x0006 = 0x0000` | ❌ 禁止 |

**结论**：只能通过 `SET GLOBAL` 设置，不支持 `SET SESSION`。

#### 典型用途
适用于**服务器级别的全局配置**，该配置对所有连接生效，没有每会话独立设置的需求。

#### 代码示例（`sql/sys_vars.cc`）

```cpp
// automatic_sp_privileges — 全局布尔配置
static Sys_var_bool Sys_automatic_sp_privileges(
    "automatic_sp_privileges",
    "Creating and dropping stored procedures alters ACLs",
    GLOBAL_VAR(sp_automatic_privileges), CMD_LINE(OPT_ARG), DEFAULT(true));

// back_log — 只读全局配置（启动时设置，运行时不可更改）
static Sys_var_ulong Sys_back_log(
    "back_log",
    "The number of outstanding connection requests MySQL can have.",
    READ_ONLY GLOBAL_VAR(back_log), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(0, 65535), DEFAULT(0), BLOCK_SIZE(1));

// replica_transaction_retries — 可动态修改的全局变量
static Sys_var_ulong Sys_replica_transaction_retries(
    "replica_transaction_retries",
    "Number of times the replication applier will retry a transaction ...",
    GLOBAL_VAR(slave_trans_retries), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(0, ULONG_MAX), DEFAULT(10), BLOCK_SIZE(1));
```

典型的 `GLOBAL_VAR` 变量举例：

| 变量名 | 说明 |
|--------|------|
| `max_connections` | 最大连接数 |
| `back_log` | 连接队列长度 |
| `basedir` | 安装目录路径 |
| `automatic_sp_privileges` | 存储过程 ACL 自动管理 |
| `replica_transaction_retries` | 复制事务重试次数 |
| PFS 相关变量 | Performance Schema 参数 |

---

### 2. `SESSION_VAR(X)` — 全局+会话双作用域变量

#### 标志值
`sys_var::SESSION`（`0x0002`）

#### 存储位置
变量 `X` 是 `System_variables` 结构体（`sql/system_variables.h`）中的**成员字段**，使用 `offsetof` 计算偏移：

```cpp
offsetof(System_variables, X)
```

`System_variables` 结构体在两个地方实例化：

1. **全局实例**：`global_system_variables`（服务器全局默认值）
2. **会话实例**：`thd->variables`（每个 THD 连接对象独有）

#### 作用域检查

| 操作 | `scope() & mask` | 结果 |
|------|-----------------|------|
| `SET GLOBAL` / `PERSIST` | `0x0002 & 0x0003 = 0x0002` | ✅ 允许 |
| `SET SESSION` / `SET DEFAULT` | `0x0002 & 0x0006 = 0x0002` | ✅ 允许 |

**结论**：同时支持 `SET GLOBAL`（修改全局默认值）和 `SET SESSION`（修改当前会话值）。

#### 读写逻辑

```cpp
// sql/set_var.cc, line 397-403
uchar *sys_var::session_var_ptr(THD *thd) {
  return ((uchar *)&(thd->variables)) + offset;  // 读取/写入会话变量
}

uchar *sys_var::global_var_ptr() {
  return ((uchar *)&global_system_variables) + offset;  // 读取/写入全局变量
}
```

- `SET GLOBAL var = X`：写入 `global_system_variables` 中的字段，影响后续**新建会话**的默认值。
- `SET SESSION var = X`（或 `SET var = X`）：写入 `thd->variables` 中的字段，仅影响**当前会话**。
- 新会话创建时会从 `global_system_variables` 拷贝初始值。

#### 典型用途
适用于**有全局默认值，但允许每个会话独立覆盖**的变量。这是 MySQL 中最常见的变量类型。

#### 代码示例（`sql/sys_vars.cc`）

```cpp
// auto_increment_increment — 会话级可覆盖，同时写入 binlog
static Sys_var_ulong Sys_auto_increment_increment(
    "auto_increment_increment",
    "Auto-increment columns are incremented by this",
    HINT_UPDATEABLE SESSION_VAR(auto_increment_increment), CMD_LINE(OPT_ARG),
    VALID_RANGE(1, 65535), DEFAULT(1), BLOCK_SIZE(1), NO_MUTEX_GUARD,
    IN_BINLOG);

// max_heap_table_size — MEMORY 表大小限制
static Sys_var_ulonglong Sys_max_heap_table_size(
    "max_heap_table_size",
    "...",
    HINT_UPDATEABLE SESSION_VAR(max_heap_table_size), CMD_LINE(REQUIRED_ARG),
    VALID_RANGE(16384, (ulonglong) ~(intptr)0),
    DEFAULT(16 * 1024 * 1024), BLOCK_SIZE(1024));

// sql_warnings — 控制警告信息显示
static Sys_var_bit Sys_sql_warnings(
    "sql_warnings", "sql_warnings",
    SESSION_VAR(option_bits), NO_CMD_LINE,
    OPTION_WARNINGS, DEFAULT(false));
```

典型的 `SESSION_VAR` 变量举例：

| 变量名 | 说明 |
|--------|------|
| `auto_increment_increment` | 自增步长 |
| `auto_increment_offset` | 自增偏移 |
| `max_heap_table_size` | MEMORY 表最大大小 |
| `sort_buffer_size` | 排序缓冲区大小 |
| `sql_mode` | SQL 模式 |
| `transaction_isolation` | 事务隔离级别 |
| `binlog_format` | 二进制日志格式 |
| `character_set_client` | 客户端字符集 |
| `time_zone` | 时区 |
| `windowing_use_high_precision` | 窗口函数精度 |

---

### 3. `SESSION_ONLY(X)` — 纯会话变量

#### 标志值
`sys_var::ONLY_SESSION`（`0x0004`）

#### 存储位置
与 `SESSION_VAR` 相同，使用 `offsetof(System_variables, X)` 计算偏移量，存储在 `thd->variables` 中。但**不存在有意义的全局对应值**。

#### 作用域检查

| 操作 | `scope() & mask` | 结果 |
|------|-----------------|------|
| `SET GLOBAL` / `PERSIST` | `0x0004 & 0x0003 = 0x0000` | ❌ 禁止 |
| `SET SESSION` / `SET DEFAULT` | `0x0004 & 0x0006 = 0x0004` | ✅ 允许 |

**结论**：只能通过 `SET SESSION`（或不带作用域前缀）设置，完全**禁止 `SET GLOBAL`**。

#### 错误处理

当尝试对 `SESSION_ONLY` 变量执行全局操作时，会触发错误：

```cpp
// sql/set_var.cc, line 1596-1599
if (!var->check_scope(type)) {
  int err = (is_global_persist()) ? ER_LOCAL_VARIABLE : ER_GLOBAL_VARIABLE;
  my_error(err, MYF(0), var->name.str);
  return -1;
}
```

错误示例：
```sql
-- 尝试全局设置 SESSION_ONLY 变量
SET GLOBAL sql_log_bin = 0;
-- ERROR 1228 (HY000): Variable 'sql_log_bin' is a SESSION variable
--                     and can't be used with SET GLOBAL
```

#### 命令行参数
`SESSION_ONLY` 变量通常与 `NO_CMD_LINE` 搭配使用，因为启动参数只影响全局默认值，而该类型变量不存在全局默认值语义：

```cpp
SESSION_ONLY(sql_log_bin), NO_CMD_LINE, DEFAULT(true), ...
```

#### 典型用途
适用于以下场景的变量：
- **复制/GTID 内部状态控制**：如 `gtid_next`、`pseudo_thread_id`，用于复制应用线程在会话内传递上下文
- **会话级安全标记**：如 `sql_log_bin`（控制当前会话是否记录 binlog）
- **客户端协议协商**：如 `resultset_metadata`（控制结果集元数据发送行为）
- **会话级限制**：如 `require_row_format`（限制当前会话只允许行格式事件）

#### 代码示例（`sql/sys_vars.cc`）

```cpp
// pseudo_thread_id — 复制时模拟线程 ID（写入 binlog）
static Sys_var_uint Sys_pseudo_thread_id(
    "pseudo_thread_id", "This variable is for internal server use",
    SESSION_ONLY(pseudo_thread_id), NO_CMD_LINE,
    VALID_RANGE(0, UINT_MAX32), DEFAULT(0), BLOCK_SIZE(1),
    NO_MUTEX_GUARD, IN_BINLOG, ON_CHECK(check_session_admin));

// sql_log_bin — 控制当前会话是否写入 binlog
static Sys_var_bool Sys_log_binlog(
    "sql_log_bin", "Controls whether logging to the binary log is done",
    SESSION_ONLY(sql_log_bin), NO_CMD_LINE, DEFAULT(true),
    NO_MUTEX_GUARD, NOT_IN_BINLOG, ON_CHECK(check_sql_log_bin),
    ON_UPDATE(fix_sql_log_bin_after_update));

// gtid_next — 为下一个事务指定 GTID
static Sys_var_gtid_next Sys_gtid_next(
    "gtid_next",
    "Specifies the Global Transaction Identifier for the following transaction.",
    SESSION_ONLY(gtid_next), NO_CMD_LINE, DEFAULT("AUTOMATIC"),
    NO_MUTEX_GUARD, NOT_IN_BINLOG, ON_CHECK(check_gtid_next));

// original_commit_timestamp — 复制时记录原始提交时间
static Sys_var_uint Sys_original_commit_timestamp(
    "original_commit_timestamp",
    "The time when the current transaction was committed on the originating source ...",
    SESSION_ONLY(original_commit_timestamp), NO_CMD_LINE,
    VALID_RANGE(0, MAX_COMMIT_TIMESTAMP_VALUE),
    DEFAULT(MAX_COMMIT_TIMESTAMP_VALUE), BLOCK_SIZE(1),
    NO_MUTEX_GUARD, IN_BINLOG,
    ON_CHECK(check_session_admin_or_replication_applier));
```

典型的 `SESSION_ONLY` 变量举例：

| 变量名 | 说明 |
|--------|------|
| `sql_log_bin` | 控制当前会话是否记录 binlog |
| `pseudo_thread_id` | 复制内部使用的线程 ID |
| `gtid_next` | 为下一个事务指定 GTID |
| `gtid_next_list` | 为下一批事务指定 GTID 集合 |
| `pseudo_replica_mode` | 开启复制伪装模式 |
| `original_commit_timestamp` | 记录原始提交时间戳 |
| `original_server_version` | 记录原始服务器版本 |
| `immediate_server_version` | 记录直接来源服务器版本 |
| `resultset_metadata` | 控制结果集元数据发送 |
| `require_row_format` | 限制会话只允许行格式 |
| `use_secondary_engine` | 控制是否使用辅助存储引擎 |
| `show_create_table_skip_secondary_engine` | 跳过 SHOW CREATE TABLE 辅助引擎信息 |

---

## 综合对比表

| 特性 | `GLOBAL_VAR` | `SESSION_VAR` | `SESSION_ONLY` |
|------|:---:|:---:|:---:|
| **作用域标志** | `GLOBAL`（0x0001） | `SESSION`（0x0002） | `ONLY_SESSION`（0x0004） |
| **支持 `SET GLOBAL`** | ✅ | ✅（修改全局默认值） | ❌ |
| **支持 `SET SESSION`** | ❌ | ✅（修改当前会话值） | ✅ |
| **支持 `SET PERSIST`** | ✅ | ✅ | ❌ |
| **存储结构** | 独立全局变量 | `System_variables` 成员 | `System_variables` 成员 |
| **全局实例** | `global_system_variables` | `global_system_variables` | 无意义 |
| **会话实例** | 无 | `thd->variables` | `thd->variables` |
| **新会话继承全局值** | 无法继承（无会话值） | ✅ 新会话从全局默认值拷贝 | ✅ 字段存在但通常由会话逻辑设置 |
| **命令行参数** | 通常支持 `CMD_LINE(...)` | 通常支持 `CMD_LINE(...)` | 通常 `NO_CMD_LINE` |
| **典型权限要求** | `SUPER` / `SYSTEM_VARIABLES_ADMIN` | 全局修改需要权限，会话修改通常无需特殊权限 | `SET SESSION` 修改，部分需 `SUPER` |
| **典型场景** | 服务器全局配置 | 有全局默认且允许会话覆盖的配置 | 复制内部状态、会话级控制标记 |

---

## 内存模型图示

```
                         ┌──────────────────────────────────────┐
                         │      global_system_variables         │
                         │  (struct System_variables 全局实例)  │
                         │                                      │
  GLOBAL_VAR(X)  ───────►│  (实际是独立全局变量，偏移相对于此) │
  SESSION_VAR(X) ───────►│  auto_increment_increment            │
                         │  sql_mode                            │
                         │  transaction_isolation               │
                         │  sql_log_bin     ← SESSION_ONLY 也   │
                         │  pseudo_thread_id  存在此处但通常    │
                         │  gtid_next         无意义的全局值    │
                         └──────────────────────────────────────┘
                                        │
                              新会话创建时拷贝（SESSION_VAR 字段）
                                        │
                   ┌────────────────────┴────────────────────┐
                   ▼                                         ▼
         ┌──────────────────┐                     ┌──────────────────┐
         │  thd1->variables │                     │  thd2->variables │
         │  (Session 1)     │                     │  (Session 2)     │
         │                  │                     │                  │
         │  auto_incr = 2   │                     │  auto_incr = 1   │
         │  sql_mode = X    │                     │  sql_mode = Y    │
         │  sql_log_bin=OFF │                     │  sql_log_bin=ON  │
         │  gtid_next = Z   │                     │  gtid_next = AUTO│
         └──────────────────┘                     └──────────────────┘
              ▲                                         ▲
         SESSION_VAR 和 SESSION_ONLY 的会话值都存在这里
```

---

## 选用原则

在 MySQL 源码开发中，为一个新的系统变量选择宏时，可按以下规则判断：

1. **使用 `GLOBAL_VAR`**：该变量描述服务器的全局属性（如内存分配上限、线程池大小、文件路径），各会话不需要独立值，且通常可以在启动参数中配置。

2. **使用 `SESSION_VAR`**：该变量有合理的全局默认值，但不同客户端连接可能需要不同的设置（如字符集、时区、事务隔离级别、查询限制参数）。这是 **最常见**的选择。

3. **使用 `SESSION_ONLY`**：该变量仅在会话上下文中有意义，设置全局值没有语义（如复制线程内部传递的元数据、GTID 控制变量、`sql_log_bin` 等会话级安全开关）。通常与 `NO_CMD_LINE` 配合使用。

---

## 参考源码位置

| 文件 | 内容 |
|------|------|
| `sql/sys_vars.h` | 三个宏的定义（第 118–126 行） |
| `sql/set_var.h` | `sys_var` 类定义、`flag_enum` 枚举、`check_scope()` 方法 |
| `sql/set_var.cc` | `session_var_ptr()`、`global_var_ptr()`、`value_ptr()` 实现 |
| `sql/sys_vars.cc` | 所有系统变量的具体声明 |
| `sql/system_variables.h` | `System_variables` 结构体定义 |
| `sql/mysqld.h` | `global_system_variables` 外部声明 |
| `sql/sql_class.h` | `THD::variables` 成员声明 |
