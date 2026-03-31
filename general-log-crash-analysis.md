# MySQL 8.0.41：语句执行时 Crash 与 General Log 记录分析

## 结论摘要

| 场景 | general_log_raw=ON | general_log_raw=OFF |
|------|--------------------|---------------------|
| **执行阶段 crash** | ✅ 会记录 | ✅ 会记录 |
| **解析阶段 crash** | ✅ 会记录 | ❌ 不会记录 |
| **alloc_query 失败** | ❌ 不会记录 | ❌ 不会记录 |

**要点**：General log 的写入都在**实际执行**（`mysql_execute_command`）**之前**完成。因此只要进程在“执行阶段”崩溃，当前这条语句**一定会**已经写进 general log；只有在“解析阶段”就崩溃且未开 raw 时，才可能没有记录。

---

## 1. General Log 写入时机（COM_QUERY）

源码路径：`sql/sql_parse.cc`、`sql/log.cc`。

### 1.1 开启 `log_raw`（opt_general_log_raw = ON）

写入发生在**分配 query 之后、解析与执行之前**：

```2011:2025:D:\codebase\mysql-server\sql\sql_parse.cc
    case COM_QUERY: {
      ...
      if (alloc_query(thd, com_data->com_query.query,
                      com_data->com_query.length))
        break;  // fatal error is set

      const char *packet_end = thd->query().str + thd->query().length;

      if (opt_general_log_raw)
        query_logger.general_log_write(thd, command, thd->query().str,
                                       thd->query().length);
      ...
      dispatch_sql_command(thd, &parser_state);
```

- 顺序：`alloc_query` → **general_log_write** → `parser_state.init` → `dispatch_sql_command` → 解析 → 执行。
- 因此：只要没有在 `alloc_query` 里就失败退出，**之后**任意阶段（解析、执行）crash，这条语句**都会**已经写入 general log。

### 1.2 未开启 log_raw（默认）

写入发生在 **dispatch_sql_command 内部、解析与 rewrite 之后、执行之前**：

```5286:5371:D:\codebase\mysql-server\sql\sql_parse.cc
  if (!err) {
    ...
    if (thd->rewritten_query().length() == 0) mysql_rewrite_query(thd);
    ...
    if (!(opt_general_log_raw || thd->slave_thread)) {
      if (thd->rewritten_query().length())
        query_logger.general_log_write(thd, COM_QUERY,
                                       thd->rewritten_query().ptr(),
                                       thd->rewritten_query().length());
      else {
        query_logger.general_log_write(thd, COM_QUERY, thd->query().str, qlen);
      }
    }
  }
  ...
  if (!err) {
    ...
    error = mysql_execute_command(thd, true);
```

- 顺序：`parse_sql` → `mysql_rewrite_query` → **general_log_write** → `mysql_execute_command`。
- 因此：只有 `!err`（解析成功）时才会执行到 `general_log_write`；**执行阶段**若发生 crash，此时早已写过 general log，**会记录**；若在 **解析阶段**就 crash（如 parser 段错误），不会执行到 `general_log_write`，**不会记录**。

---

## 2. 各类“失败/崩溃”场景

### 2.1 alloc_query 失败（2016–2017 行）

- 直接 `break`，不会执行后面的 `general_log_write`（raw 的也不会），也不会进入 `dispatch_sql_command`。
- **结论**：无论是否 raw，**都不会**记入 general log。

### 2.2 parser_state.init 失败（2034 行）

- 在 raw 的 `general_log_write` **之后**才调用 `parser_state.init`，若这里失败会 `break`。
- **结论**：**raw=ON** 时已写入，**会记录**；**raw=OFF** 时未进 dispatch，**不会记录**。

### 2.3 解析阶段 crash（parse_sql 或插件中段错误等）

- **raw=ON**：在进入 `dispatch_sql_command` 前就已写入，**会记录**。
- **raw=OFF**：`general_log_write` 在解析成功后的 `if (!err)` 里，解析阶段 crash 不会执行到此处，**不会记录**。

### 2.4 执行阶段 crash（mysql_execute_command 或存储引擎等）

- 两种模式下，`general_log_write` 都在 `mysql_execute_command` **之前**完成。
- **结论**：无论 raw 开关，**都会记录**。

---

## 3. 实现细节补充

### 3.1 General log 写入入口（log.cc）

- 实际写文件/表由 `Query_logger::general_log_write()` 完成（约 1373–1407 行）。
- 会先做 `log_command(thd, command)`、`opt_general_log` 等判断，再写 FILE/TABLE。

### 3.2 多语句与循环

- 多语句时，第一条在 `dispatch_sql_command` 内写入并执行；后续语句在循环中再次调用 `dispatch_sql_command`（约 2055、2142 行），每条语句在各自“解析成功且未 raw”或“raw 且已 alloc_query”的路径上写一次。
- 若某条语句在执行时 crash，该条语句在本条执行前已经写过 general log，因此**会**出现在 general log 中。

---

## 4. 建议

- 若希望**尽可能**在 crash（含解析阶段）时也留下语句记录，可开启 **general_log_raw=ON**（对应 `--log-raw`），这样在解析前就写入。
- 若只关心“执行阶段”崩溃：当前实现下，无论是否 raw，执行阶段 crash 的语句**都会**被记入 general log。
