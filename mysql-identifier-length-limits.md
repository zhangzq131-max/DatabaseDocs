# MySQL 标识符名称长度限制深度分析

> 源码路径：`D:\codebase\mysql-server`  
> 文档版本：基于 MySQL 8.0.41  
> 分析日期：2026-04-25

---

## 目录

1. [核心源码定义](#1-核心源码定义)
2. [各类型名称限制汇总](#2-各类型名称限制汇总)
3. [关键检查函数分析](#3-关键检查函数分析)
4. [UTF8MB4 编码的特殊情况](#4-utf8mb4-编码的特殊情况)
5. [历史设计原因](#5-历史设计原因)
6. [实际影响与建议](#6-实际影响与建议)

---

## 1. 核心源码定义

### 1.1 主定义位置

核心常量定义在 `include/mysql_com.h:57-68`：

```cpp
#define SYSTEM_CHARSET_MBMAXLEN 3          // 系统字符集最大字节/字符（基于 UTF8MB3）
#define FILENAME_CHARSET_MBMAXLEN 5        // 文件名字符集最大字节/字符
#define NAME_CHAR_LEN 64                   // 标识符字符数上限
#define USERNAME_CHAR_LENGTH 32            // 用户名字符数上限

#ifndef NAME_LEN
#define NAME_LEN (NAME_CHAR_LEN * SYSTEM_CHARSET_MBMAXLEN)  // = 192 字节
#endif

#define USERNAME_LENGTH (USERNAME_CHAR_LENGTH * SYSTEM_CHARSET_MBMAXLEN)  // = 96 字节
```

### 1.2 字符集实际 mbmaxlen

UTF8MB4 字符集的实际定义在 `strings/ctype-utf8.cc:7796-7830`：

```cpp
CHARSET_INFO my_charset_utf8mb4_general_ci = {
    ...
    1,                            /* mbminlen     */  // 最小 1 字节
    4,                            /* mbmaxlen     */  // 最大 4 字节！
    ...
};
```

**关键发现**：UTF8MB4 的 `mbmaxlen = 4`，但 `SYSTEM_CHARSET_MBMAXLEN` 定义为 `3`。

### 1.3 相关常量

```cpp
// include/my_hostname.h:42
static constexpr int HOSTNAME_LENGTH = 255;  // 主机名字节上限

// sql/sql_const.h:47
#define MAX_DBKEY_LENGTH (NAME_LEN * 2 + 1 + 1 + 4 + 4)  // TDC key 最大长度

// sql/mdl.h:339
#define MAX_MDLKEY_LENGTH (1 + NAME_LEN + 1 + NAME_LEN + 1)  // = 387 字节
```

---

## 2. 各类型名称限制汇总

### 2.1 标识符限制表


| 名称类型              | 字符数上限 | 字节数上限 | 源码定义位置                 | 检查函数                                      |
| ----------------- | ----- | ----- | ---------------------- | ----------------------------------------- |
| **库名 (Database)** | 64    | 192   | `NAME_CHAR_LEN`        | `sql/table.cc:3585` `check_db_name()`     |
| **表名 (Table)**    | 64    | 192   | `NAME_CHAR_LEN`        | `sql/table.cc:3660` `check_table_name()`  |
| **字段名 (Column)**  | 64    | 192   | `NAME_CHAR_LEN`        | `sql/table.cc:3687` `check_column_name()` |
| **索引名 (Index)**   | 64    | 192   | `NAME_CHAR_LEN`        | `sql/sql_table.cc:7019`                   |
| **外键名 (FK)**      | 64    | 192   | `NAME_CHAR_LEN`        | `sql/sql_table.cc:6449`                   |
| **视图名 (View)**    | 64    | 192   | `NAME_CHAR_LEN`        | 同表名检查                                     |
| **存储过程名**         | 64    | 192   | `NAME_CHAR_LEN`        | `sql/sp.cc:2383`                          |
| **存储函数名**         | 64    | 192   | `NAME_CHAR_LEN`        | `sql/sp.cc:2383`                          |
| **触发器名**          | 64    | 192   | `NAME_CHAR_LEN`        | `sql/parse_tree_partitions.cc:75`         |
| **事件名**           | 64    | 192   | `NAME_CHAR_LEN`        | `sql/sp.cc:2383`                          |
| **分区名**           | 64    | 192   | `NAME_CHAR_LEN`        | `sql/parse_tree_partitions.cc:75`         |
| **检查约束名**         | 64    | 192   | `NAME_CHAR_LEN`        | `sql/sql_check_constraint.cc:49`          |
| **用户名**           | 32    | 96    | `USERNAME_CHAR_LENGTH` | `sql/sql_udf.cc:619`                      |
| **主机名**           | 255   | 765*  | `HOSTNAME_LENGTH`      | —                                         |


*注：主机名字节上限取决于编码，UTF8MB4 下为 255 × 4 = 1020 字节，但实际使用 `HOSTNAME_LENGTH = 255` 作为字节上限。

### 2.2 注释/说明长度限制

```cpp
// include/mysql_com.h:81-86
#define TABLE_COMMENT_INLINE_MAXLEN 180      // 表内联注释
#define TABLE_COMMENT_MAXLEN 2048            // 表注释
#define COLUMN_COMMENT_MAXLEN 1024           // 字段注释
#define INDEX_COMMENT_MAXLEN 1024            // 索引注释
#define TABLE_PARTITION_COMMENT_MAXLEN 1024  // 分区注释
#define TABLESPACE_COMMENT_MAXLEN 2048       // 表空间注释
```

### 2.3 索引相关限制

```cpp
// sql/sql_const.h:53
#define MAX_KEY_LENGTH 3072U      // 索引所有列的总字节上限
#define MAX_KEY MAX_INDEXES       // 每表最大索引数（默认 64，最大 255）
#define MAX_REF_PARTS 16U         // 最大引用列数
```

---

## 3. 关键检查函数分析

### 3.1 check_table_name() — 表名/库名检查

定义在 `sql/table.cc:3660-3685`：

```cpp
Ident_name_check check_table_name(const char *name, size_t length) {
  // --- 第一层：字节长度检查 ---
  if (!length || length > NAME_LEN)  // NAME_LEN = 192
    return Ident_name_check::WRONG;
  
  // --- 第二层：字符数检查 ---
  size_t name_length = 0;  // 字符计数
  const char *end = name + length;
  
  while (name != end) {
    if (use_mb(system_charset_info)) {
      int len = my_ismbchar(system_charset_info, name, end);
      if (len) {
        name += len;        // 跳过多字节字符的全部字节
        name_length++;      // 只计为 1 个字符
        continue;
      }
    }
    name++;
    name_length++;
  }
  
  // 尾部空格或超过 64 字符则拒绝
  if (last_char_is_space)
    return Ident_name_check::WRONG;
  else if (name_length > NAME_CHAR_LEN)  // NAME_CHAR_LEN = 64
    return Ident_name_check::TOO_LONG;
  
  return Ident_name_check::OK;
}
```

### 3.2 check_column_name() — 字段名检查

定义在 `sql/table.cc:3687-3709`：

```cpp
bool check_column_name(const char *name) {
  size_t name_length = 0;
  bool last_char_is_space = true;
  
  while (*name) {
    last_char_is_space = my_isspace(system_charset_info, *name);
    if (use_mb(system_charset_info)) {
      int len = my_ismbchar(system_charset_info, name,
                            name + system_charset_info->mbmaxlen);
      if (len) {
        name += len;
        name_length++;
        continue;
      }
    }
    if (*name == NAMES_SEP_CHAR) return true;  // 禁止分隔符
    name++;
    name_length++;
  }
  
  // 错误条件：空名、尾部空格、或超过 64 字符
  return last_char_is_space || (name_length > NAME_CHAR_LEN);
}
```

### 3.3 check_string_char_length() — 索引名/外键名检查

用于索引名、外键名、分区名等检查，定义在多处调用：

```cpp
// sql/sql_table.cc:7019
if (check_string_char_length(key->name, "", NAME_CHAR_LEN,
                             system_charset_info, true)) {
    my_error(ER_TOO_LONG_IDENT, MYF(0), key->name.str);
    return true;
}
```

---

## 4. UTF8MB4 编码的特殊情况

### 4.1 双重限制机制

MySQL 对标识符施加**双重限制**：


| 检查层      | 条件                             | 常量                   |
| -------- | ------------------------------ | -------------------- |
| **字节长度** | `length <= NAME_LEN`           | `NAME_LEN = 192`     |
| **字符数**  | `name_length <= NAME_CHAR_LEN` | `NAME_CHAR_LEN = 64` |


### 4.2 UTF8MB4 编码下的实际限制

UTF8MB4 编码中，不同字符占用不同字节数：


| 字符类型         | 字节/字符 | 最大字符数（字节限制下） | 字节数上限（64字符） |
| ------------ | ----- | ------------ | ----------- |
| ASCII/latin1 | 1     | 64           | 64          |
| 中文字符（BMP内）   | 3     | 64           | 192         |
| Emoji/补充字符   | 4     | **48**       | 256         |


**关键结论**：在 UTF8MB4 编码下，如果标识符包含大量 emoji 或补充字符（4字节/字符），则：

- 字节限制（192）会先触发
- 实际最大字符数降至 **48 个**

### 4.3 源码验证

```cpp
// sql/table.cc:3664 第一层检查
if (!length || length > NAME_LEN)  // 字节 <= 192
    return Ident_name_check::WRONG;

// sql/table.cc:3682 第二层检查  
else if (name_length > NAME_CHAR_LEN)  // 字符 <= 64
    return Ident_name_check::TOO_LONG;
```

---

## 5. 历史设计原因

### 5.1 为什么 SYSTEM_CHARSET_MBMAXLEN = 3？

源码中明确记载了历史原因：

```cpp
// sql/mdl.h:575
static_assert(MAX_MDLKEY_LENGTH == 387, "UTF8MB3");

// sql/mdl.h:603-606 注释
/*
  With the UTF8MB3 charset space reserved for the db name/object name is
  64 * 3 bytes. utf8_general_ci collation is used for the Routine, Event and
  Resource group names. With this collation, the normalized object name uses
  just 2 bytes for each character (max length = 64 * 2 bytes). MDL_key has
  still some space to store the object names.
*/
```

### 5.2 时间线


| 时间             | 事件                                                  |
| -------------- | --------------------------------------------------- |
| MySQL 5.5.3 之前 | `utf8` 字符集仅支持 BMP 字符（最多 3 字节），即现在的 `utf8mb3`        |
| MySQL 5.5.3    | 引入 `utf8mb4`，支持完整 Unicode（最多 4 字节）                  |
| MySQL 8.0      | `utf8mb4` 成为默认字符集，但 `SYSTEM_CHARSET_MBMAXLEN` 保持为 3 |


### 5.3 设计决策分析

1. **向后兼容**：保持标识符长度限制不变，避免破坏现有系统
2. **内存效率**：MDL key 等内部结构基于固定大小设计，改用 4 会增加内存占用
3. **历史包袱**：InnoDB 早期设计注释明确提到：

```cpp
// storage/innobase/include/rem0types.h:55-62
/*
  REC_ANTELOPE_MAX_INDEX_COL_LEN is measured in bytes and is the maximum
  indexed field length (or indexed prefix length) for indexes on tables of
  ROW_FORMAT=REDUNDANT and ROW_FORMAT=COMPACT format.
  Before we support UTF-8 encodings with mbmaxlen = 4, a UTF-8 character
  may take at most 3 bytes. So the limit was set to 3*256...
  This constant MUST NOT BE CHANGED, or the compatibility of InnoDB data
  files would be at risk!
*/
```

---

## 6. 实际影响与建议

### 6.1 实际测试示例

```sql
-- 纯 ASCII（64字符）- 成功
CREATE DATABASE `1234567890123456789012345678901234567890123456789012345678901234`;

-- 超过 64 字符 - 失败
CREATE DATABASE `12345678901234567890123456789012345678901234567890123456789012345`;
-- ERROR 1470 (42000): Identifier name '...' is too long

-- 中文（64字符，192字节）- 成功（UTF8MB3编码下）
CREATE TABLE `中中中...（64个中字）` (id INT);

-- Emoji（假设全部4字节）- 最多48个
-- 48个emoji = 192字节，可成功
-- 64个emoji = 256字节，超过192字节限制，失败
```

### 6.2 最佳实践建议


| 场景            | 建议                  |
| ------------- | ------------------- |
| 纯 ASCII 环境    | 可使用完整 64 字符         |
| 国际化应用（中文等）    | 建议控制在 60 字符以内，留安全余量 |
| 包含 Emoji 的标识符 | 严格控制在 **48 字符以内**   |
| 索引名           | 同上限制，注意与列名区分        |


### 6.3 错误代码


| 错误码                           | 错误名                         | 含义               |
| ----------------------------- | --------------------------- | ---------------- |
| `ER_TOO_LONG_IDENT` (1470)    | Identifier name is too long | 超过 64 字符或 192 字节 |
| `ER_WRONG_COLUMN_NAME` (1166) | Wrong column name           | 字段名格式错误（含非法字符等）  |
| `ER_WRONG_TABLE_NAME` (1103)  | Wrong table name            | 表名格式错误           |
| `ER_WRONG_DB_NAME` (1102)     | Wrong database name         | 库名格式错误           |


### 6.4 源码修改指南

若需支持更长的标识符（不推荐，会破坏兼容性）：

1. 修改 `include/mysql_com.h` 中的 `NAME_CHAR_LEN`
2. 同步修改 `SYSTEM_CHARSET_MBMAXLEN`（考虑 UTF8MB4 应为 4）
3. 更新 `MAX_MDLKEY_LENGTH` 相关计算
4. **注意**：这会改变内部数据结构大小，影响存储格式兼容性

---

## 附录：关键代码行号索引


| 符号                        | 文件            | 行号        |
| ------------------------- | ------------- | --------- |
| `SYSTEM_CHARSET_MBMAXLEN` | mysql_com.h   | 57        |
| `NAME_CHAR_LEN`           | mysql_com.h   | 59        |
| `NAME_LEN`                | mysql_com.h   | 66        |
| `USERNAME_CHAR_LENGTH`    | mysql_com.h   | 63        |
| `HOSTNAME_LENGTH`         | my_hostname.h | 42        |
| `MAX_MDLKEY_LENGTH`       | mdl.h         | 339       |
| `MAX_KEY_LENGTH`          | sql_const.h   | 53        |
| `check_table_name()`      | table.cc      | 3660-3685 |
| `check_column_name()`     | table.cc      | 3687-3709 |
| `check_db_name()`         | table.cc      | 3585-3599 |
| `utf8mb4 mbmaxlen=4`      | ctype-utf8.cc | 7821      |
| `static_assert UTF8MB3`   | mdl.h         | 575       |


---

*文档生成时间：2026-04-25*  
*源码版本：MySQL 8.0.41*