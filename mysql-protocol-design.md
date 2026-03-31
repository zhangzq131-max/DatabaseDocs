# MySQL 8.0 Classic 应用层协议实现说明

本文基于 MySQL Server 8.0.41 源码，只描述 **MySQL 自行定义的 Classic 协议内容**（报文结构、能力协商、命令与响应格式、预处理二进制子协议等），不涉及底层传输（TCP/SSL/套接字 API 等）。

> **范围**：Classic 协议在服务端的主要编码/解码与分发逻辑。MySQL 8 的 **X Protocol**（Protobuf）不在本文展开。

### 关键结论：同一条连接可混用「文本」与「二进制 PS」

在 **同一条 Classic 连接**（同一 TCP/socket 会话、握手完成后）上，**允许交替使用**不同命令，无需整连接只选一种模式：

- 某次请求发 **`COM_QUERY`**（整段 SQL 文本、文本结果集为主），下一次可发 **`COM_STMT_PREPARE` / `COM_STMT_EXECUTE`**（预处理、二进制参数/结果），再下一次又可发 **`COM_QUERY`** ……
- 每条命令的负载首字节为对应的 **`COM_*`**，服务端在 **`do_command` → `dispatch_command`** 中**按本条命令**处理。
- **预处理语句 id（stmt_id）** 由服务端在本连接上分配，**仅对本连接有效**。

**注意**：此处混用仅指 **Classic 内部的 `COM_QUERY` 与 `COM_STMT_*`**。**X Protocol** 与 Classic **不是同一条链路**，不能在同一连接上混用。

### 术语：文中「Classic」指什么？

**Classic 协议**（官方文档常称 **Client/Server Protocol**、[MySQL Internals](https://dev.mysql.com/doc/internals/en/client-server-protocol.html) 中的传统客户端–服务器协议）指：

- 从早期 MySQL 沿用至今的 **默认 SQL 会话协议**：通常经 **TCP 或本地 socket** 连接 **`mysqld`**（默认端口 3306 一类）。
- 报文为 **3+1 字节头 + 负载**，命令阶段以 **`COM_*`（如 `COM_QUERY`、`COM_STMT_PREPARE`）** 区分请求类型；其上有 **文本结果集** 与 **`COM_STMT_*` 预处理子协议（二进制参数/行）** 等。
- 与 **MySQL X Protocol**（多基于 **Protobuf**、常见独立端口如 33060、栈与 API 完全不同）**不是同一套协议**。

源码里 **`Protocol_classic`**、`mysql_com.h` 中的 **`CLIENT_*` / `COM_*`**、本文讨论的 **`dispatch_command`** 等，都是 **Classic** 这一条线。日常使用的 **JDBC / libmysqlclient / `mysql` 客户端** 等默认连传统端口时，走的也是 **Classic**（除非显式使用 X DevAPI 等走 X Protocol）。

文中此前使用的 **「线上」** 指 **链路上传输的协议字节（wire）**，**不特指 JDBC**，也不等于「生产环境」。

---

## 1. 协议在源码中的位置

MySQL 把「线上字节 ↔ 语义」集中在以下几处：

| 内容 | 源码位置 |
|------|----------|
| 协议抽象（读命令、写 OK/ERR/结果集等） | `sql/protocol.h`、`sql/protocol_classic.{h,cc}` |
| 命令分发（`COM_*` → 具体处理） | `sql/sql_parse.cc`：`do_command`、`dispatch_command` |
| 连接握手与认证报文、插件交互 | `sql/auth/sql_authentication.cc`：`acl_authenticate`、`MPVIO_EXT` 等 |
| 能力位、命令枚举、部分报文常量 | `include/mysql_com.h` |

`sql/protocol_classic.cc` 内含大量 Doxygen `@page`（如 `page_protocol_basics`、`page_protocol_command_phase`），与 [MySQL Internals / Client/Server Protocol](https://dev.mysql.com/doc/internals/en/client-server-protocol.html) 一致，是**字节级协议说明**的首选源码对照。

---

## 2. 文本协议与二进制协议的关系

二者**不是两套独立传输协议**，而是 **同一条 Classic 连接上的两种序列化方式**，共用：外层报文封装（长度+序号）、握手与认证、`COM_*` 命令字节、OK/ERR 等通用响应。

| 维度 | 文本侧（常称 Text Protocol） | 二进制侧（Prepared Statement 子协议） |
|------|------------------------------|----------------------------------------|
| **典型入口命令** | `COM_QUERY`：SQL 以**字符串**放在包负载里 | `COM_STMT_PREPARE` / `COM_STMT_EXECUTE` / `FETCH` / `CLOSE` 等 |
| **请求形态** | 人类可读的 SQL 文本（可加 `CLIENT_QUERY_ATTRIBUTES` 等扩展） | 语句 id、游标标志、**按类型的二进制参数**、NULL 位图等 |
| **结果集行格式** | 列值多为 **Length-Encoded String** 等文本化编码 | **NULL 位图 + 各 `MYSQL_TYPE_*` 的紧凑二进制布局** |
| **元数据** | 列定义包结构双方一致（仍属 Classic） | PREPARE 后额外返回参数/列元数据；执行结果与文本结果集“列定义类似、行不同” |

**在服务端源码中的对应关系**（`sql/protocol_classic.h`）：`Protocol_classic` 之上 **`Protocol_text`** 负责按**文本**方式 `store_*` 写出列值；**`Protocol_binary`** 继承 `Protocol_text`，在执行预处理语句等路径上改用**二进制**行编码。因此可以理解为：**同一套 `Protocol` 接口，在“普通查询”与“PS 执行”两种场景下换用不同的值编码策略**。

**使用关系**：客户端仍可同时使用 `COM_QUERY` 与 `COM_STMT_*`；同一连接内交替发送即可。二进制子协议解决的是 **重复执行同一语句时的参数与结果带宽/解析成本**，并不替代 `COM_QUERY` 的即席 SQL 能力。

### 2.1 二进制协议能否与「预处理语句」画等号？

**不能简单画等号。**

| 概念 | 含义 |
|------|------|
| **预处理语句（Prepared Statement）** | 一种**能力/用法**：SQL 先 **PREPARE**（解析并保留语句与元数据），再多次 **EXECUTE**（只传参数或语句 id），强调**解析复用**与**参数化**。在 Classic 里主要由 **`COM_STMT_*` 命令族**承载；SQL 层的 **`PREPARE`/`EXECUTE` 语句**则是同一思想下的另一条入口（多为 `COM_QUERY`）。 |
| **二进制协议（此处特指 Classic 里的二进制编码）** | 一种**线上数据形态**：参数与结果行用 **NULL 位图、`MYSQL_TYPE_*` 紧凑格式** 等编码，与文本结果集里的 Length-Encoded String 相对。它附着在 **`COM_STMT_EXECUTE` 的响应**、**`COM_STMT_PREPARE` 相关报文**等路径上，是 **PS 子协议里典型的值编码方式**。 |

**关系**：日常说「走二进制协议」多指 **用 `COM_STMT_*` 这条 PS 子协议在说话**；此时与「用 C API / 协议做预处理」**高度重叠**，但**概念上**仍是 **编码格式（怎么打包字节）** 与 **会话语义（prepare/execute 生命周期）** 两个维度。**预处理语句**还可以指 **SQL 文本里的 `PREPARE`/`EXECUTE`**，那条路往往仍是 **`COM_QUERY`**，并不等同于「整连接只谈二进制包形态」。此外，**`COM_QUERY` 在具备 `CLIENT_QUERY_ATTRIBUTES` 等能力时，请求侧也可携带类二进制的参数附件**，与「只有 PS 才叫二进制」的口语说法也不完全重合。

**一句话**：**二进制编码是「怎么表示」；预处理语句是「先准备再执行」的一类用法，Classic 里主要由 `COM_STMT_*` 体现并常用二进制编码，但二者不是同一个定义，不能无条件等同。**

### 2.2 常见理解核对（二进制编码 vs. 服务端 prepare/execute）

下面是一种常见归纳，**大体正确**，仅需收紧两处表述：

| 说法 | 是否成立 | 说明 |
|------|----------|------|
| 「二进制协议是一种**编码方式**，主要配合 **prepare/execute** 使用」 | **基本对** | Classic 里「二进制」多指 **`COM_STMT_*` 上的参数与结果行编码**（相对 `COM_QUERY` 的文本结果集）。与 **PS 子协议**在实践上**强绑定**，不宜理解成与 prepare/execute 完全无关的另一套平行体系。 |
| 「prepare/execute 只是服务端的一种执行方式，**既能服务二进制协议，也能服务普通文本协议**」 | **对** | 服务端共用 **`Prepared_statement`** 等逻辑：**二进制路径**为 **`COM_STMT_PREPARE` / `COM_STMT_EXECUTE`**；**文本路径**为客户端发 **`COM_QUERY`**，内容为 SQL **`PREPARE … FROM …` / `EXECUTE …`**（见 §12.4）。同一思想，两种入口。 |

**收紧后的表述**：**线上「二进制」主要是 `COM_STMT_*` 包形态下的编码；服务端「先准备再执行」既可通过 `COM_STMT_*` 触发，也可通过 `COM_QUERY` 里的 SQL 级 PREPARE/EXECUTE 触发。**

### 2.3 单次执行的普通 SQL：二进制 PS 是否更省？

**多数情况下不会更省，反而更贵。** 对**同一条文本 SQL 只执行一次**时，PS 路径至少需要 **`COM_STMT_PREPARE` + `COM_STMT_EXECUTE`**（见 §5.4），比 **`COM_QUERY` 一次发完** 多 **至少一轮往返**，且服务端多一次 **prepare**（解析、元数据、语句对象等）成本，**无法通过多次 execute 摊薄**。

因此：**即席、单次 SQL 用文本 `COM_QUERY` 通常更合适；`COM_STMT_*` 更适合「同一语句反复执行、主要换参数」的场景。**（带宽上二进制结果集可能对大行集略有优势，但单次场景往往被多出来的 prepare 往返与 CPU 抵消。）

---

## 3. 报文封装（MySQL 自定义，非 TCP）

Classic 在字节流上的**最外层**格式由 MySQL 规定（实现上由 `NET` 与 `my_net_*` 完成读写，这里只记协议语义）：

- 每个**逻辑包**前：**3 字节小端负载长度 + 1 字节序号**（sequence id）。
- 负载超过单包上限时，按固定块大小**拆成多包**，每包自带上述包头，序号递增；最后一块可小于上限（协议允许末尾出现长度为 0 的包片，语义见 Internals）。

命令阶段中，客户端发往服务器的每个命令通常以 **序号从 0 开始** 的一包（或多包）表示；负载首字节为命令类型（见下节）。

---

## 4. 连接阶段协议（握手与认证）

连接建立后，在进入「命令阶段」之前，服务端与客户端交换的是 **握手与认证** 子协议（具体字段与插件相关报文在 Internals 的 *Connection Phase* 中描述；源码在 `sql_authentication.cc` 等）：

- 服务端发送 **Initial Handshake**，携带协议版本、服务器能力、字符集、随机挑战（scramble）等。
- 客户端回 **Handshake Response**（能力位、用户名、认证数据、可选库名等）；若协商使用认证插件，后续可能有多轮 **Auth Switch / 插件数据** 报文，由 `MPVIO_EXT` 状态机驱动（含 `RESTART` 切换插件再试）。
- **`COM_CHANGE_USER`**（命令阶段）在负载上携带新用户/库/认证信息，服务端走与连接类似的认证解析路径（`acl_authenticate(thd, COM_CHANGE_USER)`）。

能力位 **`CLIENT_*`** 决定握手与后续报文能否使用某特性（如 `CLIENT_PLUGIN_AUTH`、`CLIENT_SSL` 等）；完整定义见 `mysql_com.h`。

---

## 5. 命令阶段（Command Phase）

### 5.1 包语义

每个命令包的**负载**结构为：

1. **1 字节**：`enum_server_command`（如 `COM_QUERY`、`COM_INIT_DB`、`COM_STMT_PREPARE` 等），定义见 `mysql_com.h`。
2. **余下字节**：该命令的参数区，格式随命令不同。

服务端 **`Protocol_classic::get_command()`**（`protocol_classic.cc`）在读完一整包逻辑负载后：

- 取首字节为命令码（非法则视为 `COM_END`）；
- 将剩余数据交给 **`parse_packet()`**，填入 **`COM_DATA`** 联合体（如 `com_init_db`、`com_stmt_execute` 等）。

解析失败则 `ER_MALFORMED_PACKET`。

### 5.2 命令到执行的映射

**`dispatch_command()`**（`sql_parse.cc`）按命令码分支，例如：

- **文本 SQL**：`COM_QUERY` → 词法/语法解析与执行。
- **切换库**：`COM_INIT_DB` → 负载为库名（以连接字符集解释）。
- **工具类**：`COM_PING`、`COM_QUIT`、`COM_SET_OPTION`、`COM_RESET_CONNECTION` 等。
- **预处理语句子协议**：`COM_STMT_PREPARE` / `EXECUTE` / `FETCH` / `CLOSE` / `RESET` / `SEND_LONG_DATA` 等；执行路径使用 **二进制结果集** 格式（`Protocol_binary`）。
- **复制、克隆等**：如 `COM_REGISTER_SLAVE`、`COM_CLONE` 等另有专门分支。

各命令的请求/响应包布局在 `protocol_classic.cc` 的对应 `@page` 中有表格说明（如 `page_protocol_com_query`）。

### 5.3 口语里的「二进制协议」在服务端对应哪些 `COM_xxx`？

Classic 里常说的 **「二进制协议 / PS 子协议」** 指的是：**命令包首字节为下面这一组 `enum_server_command` 的请求**（定义见 **`include/my_command.h`**）。服务端 **`Protocol_classic::get_command()`** 读出首字节后，由 **`dispatch_command()`**（`sql/sql_parse.cc`）按命令码 `switch` 分发；**没有**单独叫「二进制解析器」的模块，仍是 **按 `COM_*` 分支 + `parse_packet()` 解负载**。

**客户端 → 服务端、与预处理语句（二进制编码）直接相关的命令字：**

| `COM_*` | 作用（简述） |
|---------|----------------|
| **`COM_STMT_PREPARE`** | 提交 SQL 文本，服务端做 prepare，返回 stmt id 与参数/列元数据。 |
| **`COM_STMT_EXECUTE`** | 按 stmt id 执行；负载里含参数（二进制编码、NULL 位图等）。 |
| **`COM_STMT_SEND_LONG_DATA`** | 分片发送大对象参数数据。 |
| **`COM_STMT_FETCH`** | 服务端游标场景下继续取行。 |
| **`COM_STMT_RESET`** | 重置语句状态（含 long data、游标等）。 |
| **`COM_STMT_CLOSE`** | 关闭语句；**无响应包**。 |

执行 **`COM_STMT_EXECUTE` / `COM_STMT_FETCH`** 时，服务端通过 **`Protocol_binary`** 等路径向客户端发 **二进制结果集行**（与 `COM_QUERY` 的文本行格式相对）。这是**响应侧编码**，不是另一个客户端发来的 `COM_` 命令。

**刻意对比**：**`COM_QUERY`** 表示「一整段 SQL 文本在包体里」，走文本协议结果集；**不属于**上面的 PS 命令族。日常说「走二进制协议」时，**不要**把 **`COM_QUERY`** 算进去（除非讨论 `CLIENT_QUERY_ATTRIBUTES` 等附件格式，那是 `COM_QUERY` 包内的扩展，语义上仍是 **QUERY 命令**）。

**一句话**：**服务端认的是包第一个字节上的 `COM_STMT_PREPARE` / `COM_STMT_EXECUTE` / `COM_STMT_FETCH` / `COM_STMT_SEND_LONG_DATA` / `COM_STMT_RESET` / `COM_STMT_CLOSE`；这些才是 Classic 里与「二进制 PS」一一对应的命令码。**

### 5.4 一条普通「文本 SQL」若走二进制 PS，对应哪些 `COM_`？

**不是**用 **`COM_QUERY`** 把整句 SQL 发过去（那是文本协议路径）。

客户端（如 **`mysql_stmt_prepare` + `mysql_stmt_execute`**）对**同一条**普通文本 SQL 的典型顺序是：

1. **`COM_STMT_PREPARE`**：负载里除命令字节外，携带 **完整 SQL 字符串**；服务端完成 prepare，返回 **statement id** 与（如有）参数/结果列元数据。  
2. **`COM_STMT_EXECUTE`**：负载里是 **stmt_id**、执行标志等；**无 `?` 占位符**时一般不传参；服务端执行该 stmt，结果用 **二进制结果集** 编码返回。

因此：**文本 SQL 作为字节串出现在 `COM_STMT_PREPARE` 里；「执行这条 SQL」在协议上对应 `COM_STMT_EXECUTE`，而不是 `COM_QUERY`。** 没有占位符也仍是 **至少两条命令**（先 PREPARE 再 EXECUTE），不存在「一条 `COM_` 同时表示整条文本 SQL 的二进制执行」的单一命令码。

### 5.5 为何不把 `COM_QUERY` 也做成「全程二进制」打包？

**协议层面的取舍，而非「二进制不可能用在 QUERY 上」的绝对限制。**

- **`COM_QUERY` 的约定**自早期版本即：**负载里主要是 SQL 文本**，服务端**本条命令内完成解析与执行**，响应为**文本结果集**（列定义 + 文本化行）。这是**即席查询**的最简路径。
- **二进制参数、二进制结果行、stmt id 与多轮 EXECUTE** 被归入 **`COM_STMT_*`**，与 QUERY **分工**：要二进制 PS 就 **PREPARE + EXECUTE**，而不是把 QUERY 的响应格式改成与 STMT 相同（否则破坏**海量存量客户端**对 QUERY 的假定）。
- **`CLIENT_QUERY_ATTRIBUTES`** 等可在 **QUERY 包**上增加附件，但**不改变**「QUERY = 一条文本 SQL 为主 + 文本结果集」的总体模型；更彻底的二进制对话仍走 **STMT**。

详见精简问答：[mysql-classic-protocol-qa.md](mysql-classic-protocol-qa.md) **Q11**。

---

## 6. 协议基础类型（编码规则）

服务端组包/解包大量复用同一套规则（文档见 `protocol_classic.cc` 开头 **Protocol Basics**）：

- **定长整数**：小端 `int<1>` … `int<8>`（如 `int4store` / `uint4korr` 等）。
- **长度编码整数**（Length-Encoded Integer）：1 / 3 / 4 / 9 字节几种形式；首字节为 `0xFE` 时需结合**包长度**判断是 8 字节整数还是 **EOF 包**（历史兼容）。
- **字符串形式**：定长、以 `0x00` 结尾、长度前缀（Length-Encoded String）、或“直至包尾”（Rest of Packet String）。

这些类型出现在握手、字段元数据、行数据、错误信息等处。

---

## 7. 通用响应包

对多数命令，服务器返回下列之一（布局见 Basics 文档与 `mysql_com.h` 约定）：

- **OK Packet**：成功、受影响行数、`last_insert_id`、状态标志、可选信息/会话跟踪等（具体字段随能力位扩展）。
- **ERR Packet**：错误码、SQLSTATE（5 字节）、消息等。
- **EOF Packet**：传统上用于结果集结束；若客户端声明 **`CLIENT_DEPRECATE_EOF`**，则结果集末尾改用 **OK** 形式表示，EOF 仅用于旧语义场景。

---

## 8. 结果集（文本与二进制）

### 8.1 文本协议（`COM_QUERY` 等）

典型顺序（概念上）：

1. **列数**（如 Length-Encoded Integer）。
2. **列定义**（多个 *Column Definition*，每列含 catalog/schema/table/名称、字符集、类型、标志等）。
3. **行数据**（每行一包或连续：各列按类型以文本或长度编码形式发送）。
4. 结束：**EOF/OK**（取决于 `CLIENT_DEPRECATE_EOF`）。

服务端通过 **`Protocol`** 的 `start_result_metadata`、`send_field_metadata`、`start_row`、`store_*`、`end_row`、`send_eof`/`send_ok` 等路径编码上述内容（`protocol.h` 注释中描述了推荐调用顺序）。

### 8.2 预处理语句（二进制协议）

- **PREPARE** 响应：语句 id、列数、参数个数、参数/列元数据等（含 `store_ps_status` 等辅助组包）。
- **EXECUTE** 响应：二进制行格式，含 **NULL 位图**、各参数类型的紧凑编码（`protocol_classic.cc` 中 *Binary Protocol* 章节有按 `MYSQL_TYPE_*` 的字段布局与示例十六进制）。

**`CLIENT_QUERY_ATTRIBUTES`** 等能力会影响 `COM_QUERY` / `COM_STMT_EXECUTE` 负载中是否携带命名参数、参数集个数等（同文件 `COM_QUERY` 文档）。

---

## 9. 能力位与行为差异（协议语义）

除握手与插件外，常见由 **`CLIENT_*`** 改变的协议行为包括（详见 `mysql_com.h` 注释）：

- **`CLIENT_PROTOCOL_41`**：4.1 以后报文格式（与现代客户端一致）。
- **`CLIENT_DEPRECATE_EOF`**：结果集结束包形态。
- **`CLIENT_SESSION_TRACK`**：OK 包中可携带会话状态变更信息。
- **`CLIENT_MULTI_RESULTS` / `CLIENT_PS_MULTI_RESULTS`**：多结果集、存储过程与 PS 的组合行为。
- **`CLIENT_COMPRESS` / `CLIENT_ZSTD_COMPRESSION_ALGORITHM`**：报文级压缩协商（仍属 MySQL 报文层约定，而非 TCP 定义）。

---

## 10. 源码阅读路径（协议视角）

1. **`include/mysql_com.h`**：`enum_server_command`、`CLIENT_*`、与报文相关的常量。
2. **`sql/protocol_classic.cc`**：从文件开头 **@page page_protocol_basics** 起，按需跳到各 `COM_*` 页面；**`parse_packet()`** / **`get_command()`** 看服务端如何解包。
3. **`sql/sql_parse.cc`**：**`dispatch_command`** 的 `switch (command)` 看每个命令在服务器端的语义落点。
4. **`sql/auth/sql_authentication.cc`**：**`acl_authenticate`** 看握手/认证与插件如何消费、产生协议报文。

按 **`get_command` → `parse_packet` → `dispatch_command`** 追一条命令，即可把「线上负载」与「服务器行为」对齐；细粒度字节布局以 **`protocol_classic.cc` 内 Doxygen** 为准。

---

## 11. 预处理语句（Prepared Statement）服务端实现纲要

### 11.1 入口：`dispatch_command` 与 `COM_STMT_*`

`sql/sql_parse.cc` 中按命令分派：

| 命令 | 处理函数（均在 `sql/sql_prepare.cc` 除非另注） |
|------|-----------------------------------------------|
| `COM_STMT_PREPARE` | `mysql_stmt_precheck` → `mysqld_stmt_prepare` |
| `COM_STMT_EXECUTE` | `copy_bind_parameter_values` → `mysqld_stmt_execute` |
| `COM_STMT_FETCH` | `mysqld_stmt_fetch`（服务端游标） |
| `COM_STMT_SEND_LONG_DATA` | `mysql_stmt_get_longdata` |
| `COM_STMT_RESET` | `mysqld_stmt_reset` |
| `COM_STMT_CLOSE` | `mysqld_stmt_close`（**无响应包**） |

`mysql_stmt_precheck()`：按 `stmt_id` 在 **`thd->stmt_map`** 中查找已有语句；**PREPARE** 时 **`new Prepared_statement` 并 `insert`**，分配唯一 **`m_id`**（客户端看到的 statement id）。

### 11.2 PREPARE：`mysqld_stmt_prepare` → `Prepared_statement::prepare`

1. **二进制协议**：Classic 连接上会 **`thd->push_protocol(thd->protocol_binary)`**，使返回给客户端的元数据/结果集走 **二进制编码**（执行完再 `pop_protocol`）。
2. **`stmt->prepare(thd, query, length, nullptr)`** 核心步骤包括：
   - 将 SQL 装入 `THD` 查询串，切换 **`stmt_arena`** 到语句私有 **`m_arena`**；
   - **`parser_state.m_lip.stmt_prepare_mode = true`**，**`parse_sql()`** 解析占位符 `?`，得到 **`LEX`**（`m_lex`）与 **`Item_param` 数组**（`init_param_array`）；
   - **`m_lex->context_analysis_only |= CONTEXT_ANALYSIS_ONLY_PREPARE`**：只做分析，不执行写盘等副作用；
   - **`prepare_query(thd)`**：按 `sql_command` 走不同分支（`open_tables`、权限、`fix_fields`、半连接转换等），与“普通查询”共用大量优化前逻辑，但停留在 **prepare/validate** 语义；
   - 成功则通过 **`prepare_query` 末尾路径** 向客户端发送 **PREPARE 应答**（语句 id、参数个数、列/参数元数据等；注释写明由 `prepare_query` 侧发元数据包）。
3. 失败则从 **`stmt_map` 中 erase** 并销毁 PSI 记录。

**要点**：PREPARE 阶段完成 **词法/语法分析 + 语义校验 + 打开表/解析列**，把 **`LEX` 树和参数元数据** 固化在 **`Prepared_statement::m_arena`** 中，**不执行**语句本体（无结果集数据行）。

### 11.3 EXECUTE：`mysqld_stmt_execute` → `set_parameters` → `execute_loop` → `mysql_execute_command`

1. 同样 **`push_protocol(protocol_binary)`**，结果集二进制发送。
2. **`stmt->set_parameters(thd, &expanded_query, has_new_types, parameters)`**：把 **`COM_STMT_EXECUTE` 包里的二进制参数**（及可选新类型描述）写入各 **`Item_param`**，并生成 **`expanded_query`**（带字面量的展开 SQL，用于 general log / slow log 等，非执行必需路径的唯一形式）。
3. **`execute_loop`**：处理 **long data 错误态**、密码过期、**`reprepare`**（元数据变更、`ER_NEED_REPREPARE`、secondary engine 等）后调用 **`execute()`**。
4. **`Prepared_statement::execute()`**：
   - 把 **`thd->lex`** 切到语句的 **`m_lex`**，**`thd->stmt_arena = &m_arena`**，必要时 **`alloc_query` 使用展开 SQL**；
   - 若带 **cursor**，为 **`Sql_cmd_dml`** 挂上 **`Query_fetch_protocol_binary`** 等结果投递对象；
   - 最终调用 **`mysql_execute_command(thd, true)`** —— 与 **普通 SQL 执行主路径相同**，只是 **不再重新 parse**，直接使用已准备好的 **`LEX`** 与已绑定的 **参数值**。

**要点**：执行阶段 = **绑定参数 + 复用已解析的 `LEX` + 走统一执行引擎**；协议上的“二进制”主要体现在 **参数输入与结果集输出** 的编码，**优化器/执行器** 与 `COM_QUERY` 同源。

### 11.4 与 SQL 层 `PREPARE stmt` / `EXECUTE stmt` 的关系

同一套 **`Prepared_statement`** 类也服务于 **SQL 语法** 的预处理（`SQLCOM_PREPARE` / `SQLCOM_EXECUTE`）。**`mysql_sql_stmt_execute()`** 用 **`find_by_name`**、**`set_parameters`**（来自 `LEX` 的 `USING` 变量）并 **`execute_loop`**，注释说明此时结果用 **文本协议** 发送，与 **`COM_STMT_EXECUTE`** 的 **二进制协议** 形成对照。

### 11.5 核心数据结构（`sql/sql_prepare.h`）

- **`Prepared_statement`**：`m_id`、`m_lex`、`m_param_array` / **`m_param_count`**、`m_query_string`、`m_arena`（语句内存池）、游标相关 **`m_cursor`** 等。
- **`thd->stmt_map`**：连接级 **`stmt_id → Prepared_statement*`** 映射。

阅读顺序建议：**`dispatch_command`（`COM_STMT_*`）→ `mysqld_stmt_prepare` / `mysqld_stmt_execute` → `Prepared_statement::prepare` / `prepare_query` / `execute` / `execute_loop`**。

---

## 12. MTR 与 `mysqltest` 的 `--ps-protocol`

### 12.1 选项名称说明

上游 **MySQL 8.0** 源码中，MTR / `mysqltest` 使用的选项是 **`--ps-protocol`**，**没有**名为 `--ps-mode` 的开关。若在其他文档或定制分支中看到不同叫法，需以对应脚本为准；与「用预处理语句协议跑测试」对应的标准选项即 **`--ps-protocol`**。

### 12.2 `--ps-protocol` 的作用

1. **传递关系**：执行 **`mysql-test-run.pl --ps-protocol`** 时，框架会把该选项传给 **`mysqltest`**（与直接对 `mysqltest` 使用 **`--ps-protocol`** 等价）。
2. **语义**（官方 [mysqltest 文档](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQLTEST.html)）：**Use the prepared-statement protocol for communication** —— 在满足条件时，用 **C API 预处理接口**（`mysql_stmt_init`、`mysql_stmt_prepare`、`mysql_stmt_execute` 等）发 SQL，对应线上的 **`COM_STMT_*`** 子协议，而不是仅走 `mysql_real_query` 类普通查询路径。
3. **并非每条语句都走 PS**：判定见下文 **§12.3**。正则 **`ps_re`** 定义在 **`client/mysqltest/regular_expressions.cc`**（*Filter for queries that can be run using the MySQL Prepared Statements C API*）。另：**`run_query_stmt`** 在 **`mysql_stmt_execute` 前不调用 `mysql_stmt_bind_param`**（注释假定**无 `?` 占位符**）；分支条件**并未**因含 `?` 而改走普通协议，含占位符且仍匹配 `ps_re` 的语句会进入 PS 路径，但往往在 execute 阶段报错，测试上应避免依赖这种组合。
4. **`--cursor-protocol`**：文档说明其 **implies `--ps-protocol`**；实现里会对 statement 设置游标相关属性（如 `STMT_ATTR_CURSOR_TYPE`），走服务端游标路径。
5. **用例内开关**：`.test` 中可使用 **`enable_ps_protocol` / `disable_ps_protocol`**（及 **`$ENABLE_PS_PROTOCOL`** 等变量）在单用例粒度控制，而不完全依赖命令行。

**一句话**：**`--ps-protocol` 让 MTR 驱动的 `mysqltest` 在匹配规则下用客户端预处理语句 API 执行 SQL，从而覆盖服务端 `COM_STMT_PREPARE` / `COM_STMT_EXECUTE` 等路径。**

### 12.3 开启 `--ps-protocol` 后，哪些语句仍走「普通协议」（`run_query_normal`）

`mysqltest.cc` 中 **`run_query()`** 仅在同时满足下列三项时调用 **`run_query_stmt`**（二进制 / PS C API），否则一律 **`run_query_normal`**（`mysql_real_query` 等普通路径）：

1. **`ps_protocol_enabled` 为真**  
   若用例里写了 **`disable_ps_protocol`**，会把全局的 **`ps_protocol_enabled`** 置为 false（见 **`prop_list`** 与 **`Q_DISABLE_PS_PROTOCOL`**），后续语句不再走 PS，直至 **`enable_ps_protocol`** 恢复。

2. **`complete_query` 为真**  
   即本次调用 **`flags` 同时带 `QUERY_SEND_FLAG` 与 `QUERY_REAP_FLAG`**（一次完成「发送 + 收结果」）。**仅 `send` 不 `reap`、或仅 `reap` 上一轮 `send`** 的分段执行不会走 **`run_query_stmt`**。

3. **`search_protocol_re(&ps_re, query)` 返回 1**  
   在去掉**前导空白**和**可选的块注释 `/* ... */`** 后，从剩余串开头做 **`ps_re`** 的 **`match_continuous`** 匹配；**额外规则**：对 **`ps_re`**（及相同的 **`sp_re`**），若整段查询字符串里**任意位置出现分号 `;`**，函数直接返回 **0**（不走 PS）。因此像 **`SELECT 1;`** 这类带分号的语句会退回普通协议，即使前缀匹配 `SELECT`。

**未列入 `ps_re` 前缀集合的语句**（在「无分号、complete_query、PS 已启用」前提下）也不会走二进制路径。源码中的白名单主要包括（节选，以 **`regular_expressions.cc`** 为准）：

- `REPLACE` / `INSERT` / `UPDATE` / `DELETE` / `SELECT` / `INSERT … SELECT` 等 DML；
- 部分 `CREATE` / `DROP` / `ALTER USER` / `RENAME` / `TRUNCATE` / `ANALYZE` / `CHECKSUM` / `OPTIMIZE` / `REPAIR`；
- `GRANT`、`KILL`、`REVOKE ALL PRIVILEGES`、`DO`、`CALL`、`COMMIT`、**`SET OPTION`**（注意：普通 **`SET @x=…` / `SET NAMES`** 等**不在**列表中）；
- 若干 **`SHOW CREATE …`**、复制相关 **`SHOW`/`SLAVE`/`RESET`**、`INSTALL/UNINSTALL PLUGIN` 等。

**典型不会匹配 `ps_re`、因而走普通协议的语句类型**（示例类举，非穷举）：

- **`EXPLAIN`**、**`USE db`**、**`DESCRIBE`/`DESC`**、**`PREPARE`/`EXECUTE`/`DEALLOCATE`**（SQL 层 PS）、**`BEGIN`/`START TRANSACTION`/`ROLLBACK`/`SAVEPOINT`**（列表中有 **`COMMIT`** 但无 **`ROLLBACK`**）；
- 以 **`WITH`** 开头的 CTE 查询（正则要求以 **`SELECT`** 等开头，**`WITH … SELECT`** 不匹配）；
- 一般 **`SHOW …`**（除白名单列出的几种 **`SHOW CREATE`**、**`SHOW BINLOG`** 等之外）；
- **`CREATE PROCEDURE`/`FUNCTION`/`TRIGGER`/`EVENT`**、**`DROP PROCEDURE`** 等（白名单只有 **`CREATE TABLE`/`DATABASE`/…** 等固定组合）；
- **`LOAD DATA`**、**`LOCK TABLES`/`UNLOCK TABLES`**、**`FLUSH`**（非 **`RESET …`** 那几种）、**`XA`**、管理类语句等凡前缀未出现在 **`ps_re_str`**  alternation 中的。

**与 `--view-protocol` / `--sp-protocol` 的关系**：若启用这两种模式，原 SQL 可能被改写成 **`SELECT * FROM mysqltest_tmp_v`** 或 **`CALL mysqltest_tmp_sp()`**，最终以**改写后的字符串**参与 **`ps_re`** 判断；`CALL`、`SELECT` 均在白名单内，故**可能**仍走 PS（若同时满足无 `;`、`complete_query` 等）。

### 12.4 `--ps-protocol` 与 SQL 语句 `PREPARE` / `EXECUTE` 的关系

**「无关」主要指：谁在发、走哪条协议。**

- **`--ps-protocol` 下 `mysqltest` 的行为**：对**整条**文本 SQL（例如一整句 `SELECT …`），在客户端依次调用 **`mysql_stmt_prepare`**、**`mysql_stmt_execute`**，对应线上 **`COM_STMT_PREPARE`** / **`COM_STMT_EXECUTE`**。实现**不会**解析该字符串是否为 SQL 里的 `PREPARE`/`EXECUTE` 命令，也不是「识别到 SQL 中的 `EXECUTE` 再特殊处理」。
- **SQL 里的 `PREPARE … FROM …` / `EXECUTE …`**：属于**普通 SQL**，通常作为一条字符串经 **`COM_QUERY`**（文本协议）发给服务器；服务器在 **`dispatch_command` → `COM_QUERY`** 中解析，走 **`SQLCOM_PREPARE`** / **`SQLCOM_EXECUTE`** 等。

因此：**`mysqltest` 用 C API 对整句 SQL 做二进制 PS**，与**你在脚本里写的 SQL 级 `PREPARE`/`EXECUTE`**，**不是同一条机制**。

**在服务器内部**：二者都可能落到 **`Prepared_statement`** 等实现，但**入口不同**：

- **SQL `PREPARE`/`EXECUTE`**：先 **`COM_QUERY`**，再进入 SQL 解析及 **`mysql_sql_stmt_execute`** 等路径。
- **二进制 PS**：先 **`COM_STMT_PREPARE` / `COM_STMT_EXECUTE`**，再 **`mysqld_stmt_prepare` / `mysqld_stmt_execute`**。

**脚本里若直接写 `PREPARE`/`EXECUTE`**：这类前缀**一般不在** **`ps_re`** 白名单中（见 §12.3），多数仍走 **`run_query_normal`**，即 **`COM_QUERY`** 发出，**不会**再被 `mysqltest` 套一层 `mysql_stmt_prepare`。

**一句话**：**`--ps-protocol` 是客户端用二进制 PS API 执行「匹配到的那类 SQL 文本」；SQL 语言里的 `PREPARE`/`EXECUTE` 通常是另一条入口（`COM_QUERY`）。二者不是同一开关的同一件事，仅服务器内部都可能用到预处理语句相关实现。**

---

## 13. `mysqltest` 与 MTR（`mysql-test-run.pl`）的关系

### 13.1 分工

| | **MTR（`mysql-test-run.pl`）** | **`mysqltest`** |
|---|-------------------------------|-----------------|
| **角色** | 测试框架 / 调度器：套件与用例选择、并行、端口、启停 **`mysqld`**、环境变量、combinations、汇总报告 | 单用例执行器：解释 **`.test`** 迷你语言 |
| **输入** | 命令行、用例列表、套件名等 | 某个 **`.test`** 文件（或 stdin） |
| **输出** | 整次运行的 pass/fail、日志目录、reject 文件组织等 | 将本用例实际输出与 **`.result`** 比对，不一致则失败并产生 diff 等 |
| **与服务器** | 负责启动实例并传入 **`--mysqld`** 等 | 按用例里的 **`connect`** 连到已由 MTR 拉起的实例 |

官方说明：**`mysql-test-run.pl` invokes `mysqltest` to run individual test cases**；**`mysqltest`** 通常由 MTR 调用，也可单独运行便于调试。

### 13.2 调用链（概念）

```text
mysql-test-run.pl
  → 准备目录、启动 mysqld、设置 MYSQL_TEST / 端口等
  → 对每个用例启动：mysqltest [选项] --test-file=xxx.test --result-file=xxx.result ...
  → mysqltest 解析 .test（query、connect、if、source …）
  → 通过 libmysqlclient 连接服务器执行 SQL（可选 --ps-protocol 等）
  → 与 .result 比对，退出码供 MTR 统计
```

### 13.3 小结

- **MTR**：管「测哪些、环境怎么起、跑多少轮、结果怎么汇总」。
- **`mysqltest`**：管「这一个 `.test` 里如何连接、执行、格式化输出、是否与期望一致」。

二者是 **调度框架 + 用例执行引擎** 的关系；**`--ps-protocol`** 属于 **`mysqltest` 的选项**，经 MTR 透传后作用于每个用例的执行路径。
