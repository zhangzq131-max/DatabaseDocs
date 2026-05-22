# 半连接的语义定义、SQL 标准与 IN/EXISTS 的关系

## 一、SQL 标准中是否有半连接的定义？

**SQL 标准（ISO/IEC 9075）中没有定义 `SEMI JOIN` 这个语法关键字。**

SQL 标准定义的连接类型包括：

- `INNER JOIN`、`LEFT OUTER JOIN`、`RIGHT OUTER JOIN`、`FULL OUTER JOIN`
- `CROSS JOIN`、`NATURAL JOIN`

但 `SEMI JOIN` 和 `ANTI JOIN` 均不在标准语法中。它们是**关系代数**中的操作符，而非 SQL 语言的直接组成部分。

在关系代数中，半连接有严格的数学定义：

> **r₁ ⋉ r₂ = π_{attrs(r₁)}(r₁ ⋈ r₂)**

即：对 r₁ 和 r₂ 做自然连接后，只投影回 r₁ 的属性集合。这意味着：

1. 结果只包含 r₁ 的列（不包含 r₂ 的列）
2. r₁ 中每行最多出现一次（即使 r₂ 中有多行匹配）
3. 只有在 r₂ 中存在匹配行的 r₁ 行才会出现在结果中

虽然 SQL 标准没有 `SEMI JOIN` 语法，但 **半连接的语义是通过 `IN` 和 `EXISTS` 子查询间接表达的**。这也是为什么 MySQL 源码中将这两种子查询的优化称为"半连接转换"——因为它们在语义层面就是半连接。

---

## 二、IN 和 EXISTS 与半连接的关系

**`IN` 和 `EXISTS` 是 SQL 语言中表达半连接语义的两种方式。** 它们在逻辑上是等价的，但语法形式不同。

### 2.1 IN 子查询

```sql
SELECT * FROM orders
WHERE customer_id IN (SELECT id FROM customers WHERE country = 'CN')
```

`IN` 的语义是：对于外层每行，检查 `customer_id` 是否存在于子查询结果集合中。这就是半连接语义——"外层值是否在内层集合中有匹配"。

MySQL 源码中，`IN` 子查询转换为半连接时，会提取等值条件：

```
1. IN/=ANY predicates on the form:

  SELECT ...
  FROM ot1 ... otN
  WHERE (oe1, ... oeM) IN (SELECT ie1, ..., ieM
                           FROM it1 ... itK
                          [WHERE inner-cond])
   [AND outer-cond]

  are transformed into:

  SELECT ...
  FROM (ot1 ... otN) SJ (it1 ... itK)
                     ON (oe1, ... oeM) = (ie1, ..., ieM)
                        [AND inner-cond]
  [WHERE outer-cond]
```

关键点：`IN` 子查询天然提供了等值条件 `(oe1,...,oeM) = (ie1,...,ieM)`，这些条件被提取到 `sj_outer_exprs` 和 `sj_inner_exprs` 中，供 `build_sj_cond` 构建半连接 ON 条件。

### 2.2 EXISTS 子查询

```sql
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM customers c WHERE c.id = o.customer_id AND c.country = 'CN')
```

`EXISTS` 的语义是：对于外层每行，检查子查询是否返回至少一行结果。这也是半连接语义——"内层是否存在匹配行"。

MySQL 源码中，`EXISTS` 子查询转换为半连接时，**没有天然等值条件**：

```
2. EXISTS predicates on the form:

  SELECT ...
  FROM ot1 ... otN
  WHERE EXISTS (SELECT expressions
                FROM it1 ... itK
                [WHERE inner-cond])
   [AND outer-cond]

  are transformed into:

  SELECT ...
  FROM (ot1 ... otN) SJ (it1 ... itK)
                     [ON inner-cond]
  [WHERE outer-cond]
```

注意：`EXISTS` 转换后没有 `ON (oe = ie)` 的等值部分——因为 `EXISTS` 本身不表达"哪个外层列等于哪个内层列"。等值关系隐含在子查询的 WHERE 条件中（如 `c.id = o.customer_id`），需要通过后续的**去相关（decorrelation）**步骤来提取。

在源码调用上下文中可以看到（`sql/sql_resolver.cc` 3159-3171行）：

```cpp
if (subq_pred->substype() == Item_subselect::IN_SUBS)
  build_sj_exprs(thd, &nested_join->sj_outer_exprs,
                 &nested_join->sj_inner_exprs, subq_pred, subq_select);
else {  // this is EXISTS
  // Expressions from the SELECT list will not be used; unlike in the case of
  // IN, they are not part of sj_inner_exprs.
```

### 2.3 IN 与 EXISTS 的核心差异

| 维度 | IN 子查询 | EXISTS 子查询 |
|------|----------|-------------|
| **语义** | 外层值是否属于内层返回的集合 | 内层是否返回非空结果集 |
| **等值条件** | 天然包含 `outer_expr = inner_expr` | 等值条件隐含在子查询 WHERE 中 |
| **NULL 处理** | `IN` 对 NULL 有特殊规则（三值逻辑） | `EXISTS` 只关心"有行/无行"，不受 NULL 影响 |
| **转换难度** | 等值条件直接提取到 `sj_outer_exprs/sj_inner_exprs` | 需要去相关步骤才能提取等值条件 |

---

## 三、IN 和 EXISTS 的等价性分析

### 3.1 非否定形式：IN 和 EXISTS 始终等价

对于 `WHERE x IN (SELECT ...)` 和 `WHERE EXISTS (SELECT ... WHERE inner = x)`，**在任何 NULL 场景下都给出相同结果**。逐一验证：

**场景 1：外层值在子查询集合中有匹配**

```
t1.a = 5, 子查询结果 = {5, NULL}
```

- `5 IN (5, NULL)`：5=5 为 TRUE → IN 为 **TRUE**
- `EXISTS (SELECT * WHERE t2.b = 5)`：t2.b=5 的行存在 → EXISTS 为 **TRUE**
- 结果一致 ✓

**场景 2：外层值在子查询集合中无匹配，子查询含 NULL**

```
t1.a = 3, 子查询结果 = {5, NULL}
```

- `3 IN (5, NULL)`：3=5 FALSE，3=NULL UNKNOWN → 没有 TRUE，至少一个 UNKNOWN → IN 为 **UNKNOWN** → WHERE 过滤掉
- `EXISTS (SELECT * WHERE t2.b = 3)`：5=3 FALSE，NULL=3 UNKNOWN → WHERE 过滤掉所有行 → 子查询返回空集 → EXISTS 为 **FALSE** → WHERE 过滤掉
- 结果一致 ✓（虽然 IN 给 UNKNOWN、EXISTS 给 FALSE，但 WHERE 都过滤掉，最终结果相同）

**场景 3：外层值为 NULL**

```
t1.a = NULL, 子查询结果 = {5, NULL}
```

- `NULL IN (5, NULL)`：NULL=5 UNKNOWN，NULL=NULL UNKNOWN → IN 为 **UNKNOWN** → WHERE 过滤掉
- `EXISTS (SELECT * WHERE t2.b = NULL)`：所有比较都为 UNKNOWN → WHERE 不返回行 → EXISTS 为 **FALSE** → WHERE 过滤掉
- 结果一致 ✓

**结论：`IN` 和 `EXISTS` 在非否定形式下始终等价，无论是否存在 NULL。**

### 3.2 否定形式：NOT IN 和 NOT EXISTS 在子查询含 NULL 时不等价

这才是真正存在差异的地方：

**场景：子查询结果含 NULL**

```
t1.a = 5, 子查询结果 = {3, NULL}
```

- `5 NOT IN (3, NULL)`：5=3 FALSE，5=NULL UNKNOWN → NOT (UNKNOWN) = **UNKNOWN** → WHERE 过滤掉（5 本应被保留，却被过滤了！）
- `NOT EXISTS (SELECT * WHERE t2.b = 5)`：3=5 FALSE，NULL=5 UNKNOWN → 子查询返回空集 → NOT EXISTS = **TRUE** → 5 被保留
- **结果不一致！** NOT IN 丢失了本该保留的行。

**原因分析**：`NOT IN` 的求值逻辑是 **逐元素比较再取反**。只要子查询集合中有任何一个 NULL 导致比较结果为 UNKNOWN，整个 `NOT IN` 就可能变成 UNKNOWN（而非 TRUE），从而在 WHERE 中被过滤掉。而 `NOT EXISTS` 只关心"子查询是否返回行"这个整体判断，NULL 对它没有影响。

**极端场景**：

```
t1.a = 5, 子查询结果 = {NULL}（仅含 NULL）
```

- `5 NOT IN (NULL)` → NOT(UNKNOWN) = UNKNOWN → 过滤掉
- `NOT EXISTS (SELECT * WHERE t2.b = 5)` → 子查询空集 → TRUE → 保留
- **NOT IN 返回空集，NOT EXISTS 返回全部外层行——完全不同！**

### 3.3 等价性总结表格

| 形式 | 子查询无 NULL | 子查询含 NULL |
|------|:------------:|:------------:|
| `IN` vs `EXISTS` | 等价 ✓ | 等价 ✓ |
| `NOT IN` vs `NOT EXISTS` | 等价 ✓ | **不等价 ✗** |

核心规律：

- **`IN` 和 `EXISTS` 恒等价**——三值逻辑的差异在 WHERE 过滤层面被消除
- **`NOT IN` 和 `NOT EXISTS` 仅当子查询结果不含 NULL 时等价**——子查询含 NULL 时，`NOT IN` 会错误地多过滤行
- 差异的根源在于：`NOT IN` 的求值是**逐元素**的（受单个 NULL 影响），而 `NOT EXISTS` 的求值是**整体**的（只看子查询是否返回行）

---

## 四、NOT IN / NOT EXISTS → 反连接（Anti-Join）

否定形式对应**反连接**（anti-join）语义：

| SQL 形式 | 关系代数 | 含义 |
|----------|---------|------|
| `NOT EXISTS (...)` | r₁ ▷ r₂ | 外层行在内层**不存在**匹配 |
| `NOT IN (...)` | r₁ ▷ r₂ | 外层值**不在**内层集合中 |

MySQL 源码中用 `can_do_aj` 标志区分（`sql/sql_resolver.cc` 2901行）：

```cpp
const bool do_aj = subq_pred->can_do_aj;
```

当 `do_aj = true` 时，转换结果使用 `AJ`（Anti-Join）而非 `SJ`（Semi-Join），执行时表现为 `LEFT JOIN` + `WHERE inner IS NULL` 的模式。

MySQL 的处理方式是：在半连接转换之前，先对 `IN` 子查询做 `IS TRUE` / `IS NOT FALSE` 的装饰（decoration），确保只有明确的 TRUE 结果才进入半连接。这在源码注释中有明确说明（`sql/sql_resolver.cc` 2878行）：

```
5. The cases 1/2 (respectively 3/4) above also apply when the predicate is
decorated with IS TRUE or IS NOT FALSE (respectively IS NOT TRUE or IS
FALSE).
```

---

## 五、半连接是否一定要消除重复行？

### 从语义定义看

半连接的数学定义 `r₁ ⋉ r₂ = π_{attrs(r₁)}(r₁ ⋈ r₂)` 中，投影操作 π 天然消除了重复——因为投影是基于集合语义的。所以**半连接的语义本身就要求消除重复行**。

这里的"重复"指的是：因内层多行匹配而导致的外层行重复产出。以 `IN` 子查询为例：

```sql
SELECT * FROM orders WHERE customer_id IN (SELECT id FROM customers WHERE country = 'CN')
```

即使 `customers` 表中有 100 行 `country = 'CN'` 的记录，`orders` 中某个 `customer_id` 匹配的那一行也只会出现在结果中 1 次，而非 100 次。

### MySQL 五种半连接策略的去重机制

MySQL 定义了五种半连接策略常量（`sql/sql_select.h`）：

```cpp
#define SJ_OPT_NONE 0
#define SJ_OPT_DUPS_WEEDOUT 1
#define SJ_OPT_LOOSE_SCAN 2
#define SJ_OPT_FIRST_MATCH 3
#define SJ_OPT_MATERIALIZE_LOOKUP 4
#define SJ_OPT_MATERIALIZE_SCAN 5
```

每种策略的核心区别就在于 **去重时机和方式** 不同：

| 策略 | 去重时机 | 去重方式 | 去重对象 | 需要临时表 |
|------|---------|---------|---------|-----------|
| DuplicateWeedout | 连接之后 | 临时表记录 rowid，重复则丢弃 | 外层行组合 | 是 |
| FirstMatch | 连接过程中 | 找到首匹配即跳回，不产生重复 | 不产生重复 | 否 |
| LooseScan | 连接之前 | 索引分组取首行 | 内层重复值 | 否（需要索引） |
| MaterializeLookup | 连接之前 | 物化时去重 | 内层重复值 | 是（物化表） |
| MaterializeScan | 连接之前 | 物化时去重 | 内层重复值 | 是（物化表） |

- **DuplicateWeedout**：将半连接当作普通 inner join 执行，然后在外层表的行 ID 上做去重。用临时表记录已见过的外层行 ID 组合，重复行直接丢弃。
- **FirstMatch**：找到内层表的第一行匹配后立即"短路返回"，不再扫描内层的后续行。这是最早终止的去重方式——不产生重复行，也就无需后续消除。
- **LooseScan**：利用索引的前缀键将相同的内层值分组在一起，每组只取第一行。不需要临时表，成本最低，但要求内层表有合适的索引。
- **MaterializeLookup / MaterializeScan**：先物化内层表到临时表中，物化过程本身带有去重（deduplication），确保临时表中每个键值只有一行，然后再与外层表做 lookup 或 scan。

---

## 六、整体关系图

```
关系代数层        SQL 语法层           MySQL 优化层
─────────        ─────────           ──────────

r₁ ⋉ r₂    ←→   WHERE x IN (SELECT ...)    →  SJ nest + ON (oe=ie) + inner-cond
(半连接)    ←→   WHERE EXISTS (SELECT ...)   →  SJ nest + ON inner-cond (需去相关)

r₁ ▷ r₂    ←→   WHERE x NOT IN (SELECT ...) →  AJ nest (LEFT JOIN + IS NULL)
(反连接)    ←→   WHERE NOT EXISTS (SELECT ..) →  AJ nest (LEFT JOIN + IS NULL)
```

核心要点：

1. **SQL 标准没有 `SEMI JOIN` 关键字**，半连接语义通过 `IN`/`EXISTS` 子查询表达
2. **`IN` 和 `EXISTS` 在任何 NULL 场景下恒等价**——三值逻辑的差异在 WHERE 过滤层面被消除
3. **`NOT IN` 和 `NOT EXISTS` 仅当子查询不含 NULL 时等价**——子查询含 NULL 时 `NOT IN` 会错误地多过滤行
4. **MySQL 将 IN/EXISTS 统一转换为半连接嵌套**（SJ nest），`IN` 天然携带等值条件，`EXISTS` 需额外去相关
5. **否定形式 (`NOT IN`/`NOT EXISTS`) 对应反连接**，实现方式为 LEFT JOIN + WHERE 内层 IS NULL
6. **半连接必须消除重复行**，这是其语义定义的一部分，五种策略的区别在于去重时机和方式