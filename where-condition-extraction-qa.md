## MySQL WHERE 条件处理问答汇总

### 问题 1：tree_or 的行为

简单说：**`tree_or()` 就是把两个范围树做“OR 合并”，用来表示 `cond1 OR cond2` 的范围条件**，但它对索引的保留非常苛刻。

#### 1. 输入 / 输出是什么？

- 输入：`SEL_TREE *tree1`, `SEL_TREE *tree2`，分别代表 `cond1`、`cond2` 的范围树。
- 输出：一个新的 `SEL_TREE *res`，语义是“在每个索引上，表示 `cond1 OR cond2` 能用的范围”。

源码里有一句注释：`// tree1 and tree2 are not usable after tree_or()`，说明它会**就地修改并“吃掉”原来的树**。

#### 2. 对 `SEL_TREE.type` 的处理

大致规则：

- 任一为 `IMPOSSIBLE` → 返回另一个（因为 `FALSE OR X = X`）。
- 若都不是 `KEY`（比如 `ALWAYS` / 无法生成范围）→ 结果通常也是 `ALWAYS` 或退化。
- 只有当至少一边是 `KEY` 时，才可能产生可用于范围扫描的 `keys[]`。

#### 3. 对每个索引 `keys[i]` 怎么处理？

核心行为：

- **只有“两个子树在同一个索引上都有范围条件”时，才会保留这个索引**：
  - 如果 `tree1->keys[i]` 和 `tree2->keys[i]` 都非空：  
    - 调用 `key_or(tree1->keys[i], tree2->keys[i])`，在该索引上做 **区间并集**，得到 `res->keys[i]`。
  - 只要有一边是 `nullptr`：  
    - **这个索引在 OR 结果里被丢弃**（因为没法在这个索引上同时表达两个分支的 OR）。
- 同时更新 `keys_map`：只给那些“在两边都有 SEL_ROOT 且成功并集”的索引置位。

这就是你在文档里看到的那句总结：

> OR 只能保留「两边都有范围」的索引，否则该索引会从 `keys_map` 中被清除。

#### 4. `key_or()` 里发生了什么？

在单个索引的 `SEL_ROOT` 层面，`key_or()` 做的是**同一索引上的区间并集**：

- 同一 keypart（同一个 `part`）上的多个区间通过 `SEL_ARG::next/prev` 串起来，OR 合并时会把两边的区间做 union；
- 有时并集会变得很“宽”（甚至接近全范围），并且可能导致 `SEL_TREE::inexact` 变为 `true`，表示“范围树是过宽近似，执行时还要做二次过滤”。

#### 5. 和 Index Merge 的关系

当遇到这种情况：

- `cond1` 只在某些索引上有范围条件；
- `cond2` 在另一批索引上有范围条件；
- 两边**没有一个索引是“双方都有范围”的**，

那么：

- `tree_or()` 在 `keys[]` 上几乎什么都保不住（大部分 `keys[i]` 变成 `nullptr`）；
- 但它会把两个子树打包成 `SEL_IMERGE` 放进 `tree->merges` 里，作为 **Index Merge Union** 的候选方案；
- 后面 `get_key_scans_params()` 会基于 `merges` 评估 index_merge 的代价。

这就是文档里那种场景：

```sql
WHERE city = 'Beijing' OR amount > 5000
```

- 左边只对 `idx_city` 有范围；
- 右边只对 `idx_amount` 有范围；
- `tree_or()` 结果：`keys[]` 全空，但 `merges` 里有一个 union 方案（`idx_city` ∪ `idx_amount`）。

#### 6. 总结一句话

- **`tree_and()`：尽量保留并收紧所有索引上的范围（交集）。**
- **`tree_or()`：只在“两个分支都能用同一个索引”时，才在这个索引上做区间并集；否则该索引直接被丢弃，OR 逻辑交给 Index Merge 或全表扫描来兜底。**

---

### 问题 2：丢弃索引后的 filter 行为

是的，会继续当成普通的 `WHERE` 条件来做 filter，只是**不再通过这个索引的 range 来实现**。

稍微展开一下层次：

- **`tree_or()` / `SEL_TREE` 只负责“能不能用索引做 range / index_merge”**  
  - 当某个索引在 OR 合并时被丢掉（对应 `keys[i]` 变成 `nullptr`，或者只进了 `merges`），只是说明：  
    > “在这个 OR 条件下，没法在索引 `i` 上构造一个既正确又高效的范围扫描方案”  
  - 并不改变原始 `WHERE` 条件本身。

- **真正决定“行要不要留下”的，始终是完整的条件表达式**  
  - 即使走了 range / index_merge，执行层在读到行之后，**仍然会用原始的 `WHERE`（以及 `JOIN` 条件）再算一遍真假**；  
  - 对于 `SEL_TREE::inexact == true` 的情况，官方注释也明确要求“range 之后必须二次过滤”。

- **当某索引在 `tree_or()` 中被丢弃时，可能发生的是：**
  - 优化器改用：
    - 其他索引（比如另一条 range 或 index_merge），再加上剩余谓词做 filter，或者  
    - 直接全表扫描（`type=ALL`），然后**所有 OR 条件都在行级用 CPU 计算过滤**。
  - 语义上 `WHERE` 不变，只是**那部分条件不再通过该索引的范围来“提前过滤”，而是退化为“扫描后 filter”**。

所以可以记一句话：  

> `tree_or()` 丢掉的是“这个索引用来做 range 访问的资格”，**不是丢掉条件本身**；条件依然会在执行阶段参与过滤，只是可能以“行后过滤”的方式生效。

---

### 问题 3：复杂 WHERE 的按表抽取与 range 条件剥离

### 简单结论先说清楚

- **“从复杂 WHERE 里抽某一张表的条件”**：主要由 `make_cond_for_table()`（再加上 `reduce_cond_for_table()` 等）完成，它递归遍历 `Item` 条件树，按 `used_tables()` 位图、AND/OR 规则，把**在当前 join 前缀中“可用”且涉及目标表的谓词**抽出来，挂到这张表的 `tab->condition()` 上。
- **“同时又把 range scan 已经用掉的条件抽掉”**：在经典 join 优化器里，**大多数情况下并不会真的从 `WHERE` 树删掉这些谓词**，而是：
  - 在 `test_quick_select()` / Range Optimizer 里，把能转成 range 的谓词标记为“由访问方法覆盖”；
  - 在代价估计和过滤率计算（`calculate_condition_filter()`）时，不再把这些谓词算进“剩余过滤条件”；
  - 执行阶段很多时候仍然会再算一遍（为了安全和 inexact 情况），只是不会再影响访问方式选择。

下面分两步讲清楚这两个问题。

---

## 一、怎么从复杂 WHERE 中抽取“属于某表”的过滤条件？

### 1. 整体入口：`make_join_query_block()` 调 `make_cond_for_table()`

在你已经写过的流程里，第二阶段 Range 分析前有一步：

- `JOIN::make_join_query_block()`（`sql/sql_optimizer.cc`）  
  会遍历 join 顺序里的每个 `JOIN_TAB`，对全局 `cond` 调用：

```cpp
tmp = make_cond_for_table(
  thd,
  cond,                // 整棵 WHERE / JOIN 条件树
  used_tables,         // 当前前缀中“已可用”的表集合
  current_map,         // 当前这张表的 bit（以及某些外连接需要的额外 bit）
  false                // 是否过滤昂贵表达式
);
tab->set_condition(tmp);
```

得到的 `tmp` 就是**这张表在当前 join 前缀下可以用来 filter 的条件**。

### 2. 关键函数：`make_cond_for_table()`

定义在 `sql/sql_optimizer.cc`，声明在 `sql/sql_optimizer.h`：

```cpp
Item *make_cond_for_table(THD *thd, Item *cond,
                          table_map tables,    // 前缀可用表
                          table_map used_table,// 当前表（或 0 表示所有表）
                          bool exclude_expensive_cond);
```

核心逻辑是递归遍历 `Item` 树，结合 `used_tables()` 位图来“筛条件”：

#### （1）先做“与当前表无关”的快速剪枝

```cpp
if (used_table &&                           // 正在为某张具体表抽条件
    !(cond->used_tables() & used_table) &&  // 该谓词根本不引用这张表
    !(cond->is_expensive() && used_table == tables))
  return nullptr;
```

- 对“为某张表抽条件”（`used_table != 0`）的场景：
  - 如果谓词完全不引用这张表 → 直接返回 `nullptr`，表示**这条谓词不属于这张表的 filter**；
  - 只有一个例外：首表 + 昂贵谓词的特殊情况，这里略过细节。

#### （2）递归处理 AND / OR

- **AND 节点：**  
  - 为每个子节点递归调用 `make_cond_for_table()`；
  - 把所有非空子条件重新拼成一个新的 `Item_cond_and`：
    - 0 个 → 返回 `nullptr`（等价 TRUE）；
    - 1 个 → 直接返回那一个；
    - ≥2 个 → 用 `fix_fields()` 固化一棵新的 AND。
  - 含义：**AND 里只要有一部分子条件对当前表可用，就可以“部分下推”**。

- **OR 节点：**  
  - 每个分支仍然递归 `make_cond_for_table()`，但 `used_table` 传 0，只根据 `tables`（前缀可用表集合）判断；
  - 只要有一个分支返回 `nullptr`，整个 OR 就返回 `nullptr` —— 表示**这个 OR 整体不能安全下推**（否则会改变逻辑语义）；  
  - 只有当所有分支都可抽取时，才重建一个新的 OR。
  - 这就是 “OR 只能整体下推，不能只推其中一部分”的语义来源。

#### （3）叶子谓词的前缀可用性检查

到了非 AND/OR 的叶子谓词：

```cpp
if ((cond->used_tables() & ~tables) ||                      // 引用了前缀之外的表
    (!used_table && exclude_expensive_cond && cond->is_expensive())) // 昂贵谓词延后
  return nullptr;
```

- 如果谓词引用了**尚未出现在当前前缀的表** → 无法在当前阶段评估，只能等后面的表加入 → 不抽取到这张表上；  
- 在“抽取 const 条件 / 所有表条件”的情况下（`used_table == 0`），还可以通过 `exclude_expensive_cond` 把昂贵谓词留到执行阶段。

> 结果：  
> 对于一个再复杂的嵌套 WHERE / ON 树，`make_cond_for_table()` 都是按 **“AND 允许部分抽取，OR 只允许整体抽取” + `used_tables()` 子集关系** 来选择出“**在当前 join 前缀下、可以安全挂在这张表上的谓词**”，组成 `tab->condition()`。

---

## 二、range 已用条件与 filter 条件怎么区分、不会“重复计入”？

你问的第二半句是：**既要把“用于 range scan 的条件”从 filter 里剥离掉，又要在 filter 里保留剩余谓词，尤其是复杂嵌套的场景。**

这里分两层：

1. **代价/过滤率计算层：不把“已用于访问方法的谓词”再算进 filter。**
2. **执行层：哪些谓词还会在行级真正再算一遍？**

### 1. 代价与过滤率：`calculate_condition_filter()`

在 `best_access_path()` 里，选完某个访问方式（ref/range/index/ALL）后，会调用：

```cpp
filter_effect = calculate_condition_filter(tab, best_ref, ...);
```

里面有非常关键的三步（你文档第 5.6 节已经写过）：

```text
filter = 1.0

阶段1：排除已用列
  - ref/range 访问已覆盖的列不重复计算
  - 如果 cond_set ⊆ 已用列，直接返回 1.0

阶段2：利用 range 优化器对其他索引的估算（精度较高）

阶段3：递归计算 WHERE 条件树（get_filtering_effect）
```

这里的“**已用列**”，就是指：

- `ref` 访问所用到的 `Key_use` 中对应的列；
- `range` 访问在 `SEL_TREE` / `SEL_ARG` 中已经变成范围边界的列。

算法大致做的是：

- 建一个“访问方法已用列”的位图；
- 在计算 filter 时：
  - 只要一个谓词用到的列集合完全被“已用列”覆盖，就认为这个谓词**属于 access condition**，不再贡献额外过滤率；
  - 只有涉及**访问方法没用到的列**的谓词，才算作真正的“剩余过滤条件”。

> 也就是说，**在代价估计层面，“range scan 已经用掉的条件”不会再被算进 filter 里**，避免“双重统计”。

### 2. 执行层：谓词是不是还会“再算一遍”？

经典 join 优化器比较保守：

- 即使某些谓词已经被识别为 range/ref access condition，执行阶段通常**还是会在 `tab->condition()` 中重新评估它们**，以保障：
  - 复杂表达式、类型转换、函数副作用等“被 range 近似掉”的情况不会漏掉；
  - `SEL_TREE::inexact == true` 的场景肯定要二次过滤。

- 真正做到“不再在 filter 中显式出现”的，主要是几类安全场景：
  - **const 表**、某些简单 `ref` / `const` 访问；
  - 联合 **Index Condition Pushdown（ICP）** 等机制时，部分类似谓词被挪到存储引擎内部执行。
  
从“技术月报”的视角，你可以这样总结：

- **逻辑上**：  
  - Range Optimizer 和 ref 优化，会把一部分 WHERE 谓词标为“由访问方法覆盖”；  
  - `calculate_condition_filter()` 在估算过滤率时，会**排除这些谓词**，把它们视为已经用于缩小访问行数的条件。

- **实现上**：  
  - 条件提取走 `make_cond_for_table()` / `reduce_cond_for_table()`；  
  - 访问条件 vs 过滤条件的列集划分，通过 `Key_use`、`SEL_TREE` / `SEL_ARG` 以及 `calculate_condition_filter()` 内部的列位图来完成；  
  - 执行层为了安全，常常还是会对 `tab->condition()` 进行完整求值，只是在行数估计和选择访问路径时已经把“访问方法已用谓词”从 filter 里逻辑剥离。

---

### 三、可以放进文档的一句话总结

> **对任意复杂的 WHERE 条件树**，MySQL 先用 `make_cond_for_table()` 按 `used_tables()` 和 AND/OR 规则，把“在当前 join 前缀可用、且涉及某表”的谓词抽出来挂到这张表上；  
> **在选择 ref/range 访问方式后**，通过 `calculate_condition_filter()` 的“排除已用列”机制，把已经被访问方法消化掉的谓词从过滤率计算中剥离，只对真正剩余的条件估算 filter effect——执行阶段通常仍会对整棵条件树再求值一次，以兜住 Range Optimizer 的近似与 inexact 情况。

