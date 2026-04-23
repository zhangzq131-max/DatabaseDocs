# MySQL 5.7 Query Cache 实现分析、使用方式与移除原因

本文整理自上面的完整讨论，包含三部分：

1. MySQL 5.7 `query cache` 的实现方案分析  
2. `query cache` 的实际使用方式  
3. 为什么 5.7 的 `query cache` 存在严重问题并在后续版本移除

---

## 一、MySQL 5.7 Query Cache 实现方案分析

`MySQL 5.7` 的 Query Cache 是一个 **“SQL 文本结果缓存 + 表级反向索引失效”** 方案，核心实现集中在 `sql/sql_cache.cc` 和 `sql/sql_cache.h`。

### 核心结论

- **读路径前置到 parse 之前**：`mysql_parse()` 一开始就尝试命中缓存，命中就直接回包，跳过解析/优化/执行。
- **写路径是“注册 + 流式写包 + 完成提交”**：先 `store_query()` 注册 writer，执行期通过 `query_cache_insert()` 持续写结果包，最后 `end_of_result()` 把结果标记为完整可读。
- **索引结构是双哈希**：  
  - `queries`：`cache_key -> query block`  
  - `tables`：`table_key -> table block`  
  同时用链表维护 `table -> dependent queries`，便于按表失效。
- **失效以“表”为粒度**：DML/DDL/事务提交会触发 `invalidate`，命中表后批量删掉相关 query blocks。
- **并发模型是“全局锁 + 查询块读写锁”**：全局 `structure_guard_mutex` 序列化结构操作，query block 上有 `rwlock` 保护结果读写。
- **这是 5.7 的老方案且已废弃**：`query_cache_type` 在 5.7 已标注 deprecated。

### 1) 入口时机：先查缓存再解析

`mysql_parse()` 里先执行 `query_cache.send_result_to_client()`，未命中才继续 parse/execute。  
而“写入缓存”的注册发生在 `SELECT` 执行路径里，且在锁表之后通过 `query_cache.store_query()` 完成。

### 2) Key 设计：不仅是 SQL 文本

`send_result_to_client()` / `store_query()` 会构建包含上下文的 key：SQL + DB + flags。  
`flags` 包含协议能力、字符集、`sql_mode`、`limit`、时区、事务状态等，保证不同会话语义不会误复用缓存。

### 3) 写路径：注册 writer -> 流式 append -> 完成

- `store_query()`：分配 query block，注册依赖表，绑定 writer（`thd->query_cache_tls.first_query_block`）。
- `query_cache_insert()`：执行期间每次发包时 append 到 result block 链。
- `end_of_result()`：结果成功结束后把 block 类型置为 `RESULT`，写入 `found_rows`，并释放 writer。
- `abort()`：执行异常时清理未完成项，避免脏数据进入缓存。

### 4) 命中读取：完整性 + 权限 + 引擎校验

`send_result_to_client()` 命中后还会继续做检查：

- 必须是可缓存 `SELECT`（无 `SQL_NO_CACHE`）
- 结果块必须是完整态（`RESULT`）
- 临时表遮蔽检查
- 权限检查（`check_table_access`）
- 引擎 callback 二次确认（必要时触发失效）

全部通过后才把缓存结果写回客户端。

### 5) 失效模型：按表反向索引批量清理

失效按“表 key”查 `tables` 哈希，找到 table block 后顺着依赖链删除 query blocks。  
事务场景下会先把变更表记入 `changed_tables`，在提交阶段统一失效。

### 6) 并发与内存管理

- 并发：`try_lock()/lock()/lock_and_suspend()` 管理全局访问状态，叠加 query block 的读写锁。
- 内存：自管 block + bins，内存不足时通过 `free_old_query()` 淘汰旧查询并累加 `lowmem_prunes`。
- 另外有 `pack`/`flush` 机制处理整理和清理，但会放大全局锁竞争。

### 7) `query_cache_type` 与 SQL 语法配合

5.7 支持 `OFF / ON / DEMAND`：

- `ON`：默认缓存可缓存 `SELECT`，`SQL_NO_CACHE` 可排除
- `DEMAND`：仅缓存 `SELECT SQL_CACHE`

该能力在 5.7 已经标记为 deprecated。

---

## 二、Query Cache 的使用方式（5.7）

### 1) 先确认实例支持

```sql
SHOW VARIABLES LIKE 'have_query_cache';
```

- `YES`：编译支持 Query Cache
- `NO`：实例不可用

### 2) 基础开启参数

关键参数：

- `query_cache_size`：缓存总内存（`0` 即关闭）
- `query_cache_type`：`OFF | ON | DEMAND`
- `query_cache_limit`：单条结果可缓存上限

动态设置示例：

```sql
SET GLOBAL query_cache_size = 64*1024*1024;
SET GLOBAL query_cache_type = ON;
SET GLOBAL query_cache_limit = 1*1024*1024;
```

`my.cnf` 示例：

```ini
[mysqld]
query_cache_size=64M
query_cache_type=ON
query_cache_limit=1M
query_cache_min_res_unit=4K
```

### 3) 三种模式如何用

- `query_cache_type=OFF`：不读、不写缓存
- `query_cache_type=ON`：默认缓存可缓存 `SELECT`，可用 `SQL_NO_CACHE` 排除
- `query_cache_type=DEMAND`：默认不缓存，只有 `SQL_CACHE` 才缓存

示例：

```sql
SELECT SQL_NO_CACHE * FROM t WHERE id=1;
SELECT SQL_CACHE * FROM t WHERE id=1;
```

### 4) 会话级控制

```sql
SET SESSION query_cache_type = OFF;
SET SESSION query_cache_type = ON;
SET SESSION query_cache_type = DEMAND;
```

适合把报表连接和 OLTP 连接分开策略。

### 5) 观测指标

```sql
SHOW STATUS LIKE 'Qcache%';
```

重点看：

- `Qcache_hits`
- `Qcache_inserts`
- `Qcache_not_cached`
- `Qcache_lowmem_prunes`
- `Qcache_free_memory`

经验判断：

- `hits` 稳定增长：有收益
- `lowmem_prunes` 高：缓存小或碎片重
- `not_cached` 高：大量语句先天不可缓存

### 6) 运维命令

```sql
RESET QUERY CACHE;
FLUSH QUERY CACHE;
```

- `RESET QUERY CACHE`：清空缓存项
- `FLUSH QUERY CACHE`：整理碎片

### 7) 失效规则（必须理解）

- 表有写入（`INSERT/UPDATE/DELETE/DDL`）会失效依赖该表的缓存
- 事务表常在提交后统一失效
- 临时表、权限/会话语义变化等会导致无法复用
- `SQL_NO_CACHE` 明确禁用缓存

### 8) 适用与不适用场景

适用：

- 读多写少
- 重复完全相同 SQL
- 结果集不大、变更不频繁

不适用：

- 高频写入
- 高并发 OLTP

### 9) 现实建议

5.7 可用但已废弃；8.0 已移除。  
长期建议优先考虑应用层或代理层缓存（如 Redis / Proxy），并结合索引与 SQL 优化。

---

## 三、为什么 5.7 的 Query Cache 问题严重并被移除

核心原因：**收益不稳定，但成本在高并发下是全局性的**。随着核数和写并发上升，负收益越来越常见。

### 严重问题（按影响排序）

1. **全局锁争用严重**  
   命中、写入、失效、flush 都会触碰全局结构锁，成为高并发热点。

2. **失效粒度粗（表级）**  
   某表一写，依赖该表的缓存项可能大量失效，读写混合场景下反复重建，命中率不稳定。

3. **内存碎片和维护开销高**  
   自管内存块会碎片化，`pack/flush` 与淘汰逻辑进一步增加同步成本。

4. **命中条件苛刻，复用率受限**  
   不仅要 SQL 相同，还要协议、字符集、时区、`sql_mode`、事务状态等一致。

5. **事务一致性逻辑复杂**  
   提交时机、引擎回调和失效协同复杂，正确性成本高。

6. **与现代优化方向冲突**  
   Query Cache 的全局共享架构不适合横向扩展和高并发，后续生态更倾向引擎优化与外部缓存。

### 为什么是“移除”而不是“重写”

要修复根问题，几乎需要重做成另一套缓存体系（细粒度依赖追踪、低锁结构、语义隔离等），复杂度与风险很高；而应用层/中间层缓存 ROI 更高、可控性更好。  
因此路线变为：**5.7 废弃，8.0 移除**。

