# MySQL Classic 协议：困惑速查与基本知识（精简版）

本文从 [mysql-protocol-design.md](./mysql-protocol-design.md) 提炼，用**问答**对齐常见困惑，并保留理解 Classic 所需的最小公共知识。细节、源码路径与 MTR 细则仍以原文为准。

---

## 一、5 分钟基本知识（Classic）

1. **Classic 是什么**  
   传统 **Client/Server Protocol**（默认 3306、TCP/socket），与 **X Protocol**（Protobuf、常见 33060）**不是同一套**。JDBC / libmysqlclient / `mysql` 默认连传统端口即 Classic。

2. **报文外壳**  
   每个逻辑包：**3 字节小端长度 + 1 字节序号**，再跟负载。大包会拆片，序号递增。

3. **连接阶段**  
   握手、能力位 `CLIENT_*`、认证插件；然后进入命令阶段。

4. **命令阶段**  
   负载**第 1 字节**是 **`COM_*`**（`enum_server_command`），余下是参数。服务端 **`get_command` → `dispatch_command`** 按本条命令处理。

5. **两条主路径（同一连接可交替）**  
   - **`COM_QUERY`**：整段 SQL 文本在包里 → 文本结果集为主。  
   - **`COM_STMT_PREPARE` / `COM_STMT_EXECUTE` / …**：预处理子协议 → 二进制参数与二进制结果行为主。  

   **同一条 Classic 连接可以这一包用 QUERY、下一包用 STMT，无需二选一。**（X Protocol 不能与 Classic 混在同一条链路上。）

6. **常见响应**  
   OK / ERR；结果集结束受 `CLIENT_DEPRECATE_EOF` 等影响（EOF 或 OK 形态）。

7. **「链路上 / wire」**  
   指**线上实际传输的协议字节**，不特指某一种驱动，也不等于「生产环境」。

---

## 二、你的困惑 → 直接回答

### Q1：文本协议和二进制协议到底是什么关系？

**同一套 Classic 里的两种编码/用法**，共用包头、握手、`COM_*` 模型。  
- **文本**：典型入口 **`COM_QUERY`**，结果行多为长度编码字符串等。  
- **二进制（PS 子协议）**：入口 **`COM_STMT_*`**，参数与结果行用 NULL 位图、`MYSQL_TYPE_*` 紧凑格式等。  

不是两套独立「传输层协议」。

### Q2：二进制协议能跟「预处理语句」画等号吗？

**不能。**  
- **预处理**：语义是「先准备、再执行、可重复绑参」；Classic 里主要靠 **`COM_STMT_*`**，也可通过 **`COM_QUERY` 里写 SQL 的 `PREPARE`/`EXECUTE`** 达到类似效果（入口不同）。  
- **二进制**：主要指 **`COM_STMT_*` 路径上的参数/行怎么打包**。  

口语里「走二进制」多等于「走 `COM_STMT_*`」，和「预处理」重叠大，但**概念维度不同**（编码 vs 生命周期）。

### Q3：服务端「prepare/execute」是不是既服务二进制也服务文本？

**对。** 内部可共用 **`Prepared_statement`** 等逻辑：  
- **二进制入口**：`COM_STMT_PREPARE` / `COM_STMT_EXECUTE`。  
- **文本入口**：客户端发 **`COM_QUERY`**，内容为 **`PREPARE … FROM …` / `EXECUTE …`**。  

### Q4：口语里的二进制协议，服务端到底认哪些 `COM_xxx`？

与 **客户端→服务端** 的 PS 子协议直接相关的是：  
**`COM_STMT_PREPARE`、`COM_STMT_EXECUTE`、`COM_STMT_SEND_LONG_DATA`、`COM_STMT_FETCH`、`COM_STMT_RESET`、`COM_STMT_CLOSE`**（定义见 `my_command.h`）。  

没有单独的「二进制解析器」名字，仍是 **按 `COM_*` 分支 + 解包**。  
**`COM_QUERY` 不属于这一组**（整句 SQL 文本走 QUERY）。

### Q5：一条普通文本 SQL，若走「二进制 PS」，对应什么命令？

**至少两步，没有单字节合一：**  
1. **`COM_STMT_PREPARE`**（负载里带**完整 SQL 字符串**）  
2. **`COM_STMT_EXECUTE`**（**stmt_id** 等；无 `?` 时常无参块）  

**不是**用一条 `COM_QUERY` 当「二进制执行」。

### Q6：普通 SQL 只执行一次，二进制 PS 是不是更划算？

**多数更亏**：多至少一轮往返 + 服务端 prepare 成本，又**摊不到多次 execute** 上。  
**重复执行同一句、主要换参数**时，`COM_STMT_*` 更有意义。

### Q7：一条连接能文本和二进制混着用吗？

**能。** 每条请求各自一个 `COM_*`，服务端按**本条**处理；**stmt_id 仅本连接有效**。  
（仅指 Classic 内部；不要和 X Protocol 混在同连接。）

### Q8：`mysqltest --ps-protocol` 和 SQL 里的 `PREPARE`/`EXECUTE` 是一回事吗？

**不是。**  
- **`--ps-protocol`**：对**匹配规则下的整段 SQL**，客户端调 **`mysql_stmt_prepare` + `mysql_stmt_execute`** → **`COM_STMT_*`**。  
- **SQL 里的 `PREPARE`/`EXECUTE`**：通常整段仍作为 **`COM_QUERY`** 发出。  

脚本里写 `PREPARE`/`EXECUTE` 往往还**不在** `mysqltest` 的 `ps_re` 白名单，更容易走 **`run_query_normal`**。

### Q9：MTR 和 `mysqltest` 什么关系？

**MTR（`mysql-test-run.pl`）** 起服务器、调度用例；**`mysqltest`** 执行单个 `.test`、比对 `.result`。  
**`--ps-protocol`** 是传给 **`mysqltest`** 的选项。

### Q10：JDBC、PyMySQL 怎么「走二进制 PS」？有没有叫「binary protocol」的开关？

驱动里**很少**出现字面量「binary protocol」，常见的是 **「服务端预处理」**：  
- **Connector/J**：连接串 **`useServerPrepStmts=true`**（常配合 **`cachePrepStmts`**）→ **`COM_STMT_*`**。  
- **PyMySQL**：常见实现**不**走 `COM_STMT_*`，参数化多仍是转义后 **`COM_QUERY`**。需要服务端 PS 可看 **MySQL Connector/Python** 的 **`MySQLCursorPrepared`** 等。  

**不是「所有驱动都没有显式方式」**，而是名字多为 **server-side prepared statement**，语义上对应 **`COM_STMT_*`**。

### Q11：为什么 `COM_QUERY` 不能也按「二进制协议」那样打包？

**不是技术上绝对做不到，而是协议这样定义 + 历史与兼容的取舍。**

1. **命令语义不同**  
   **`COM_QUERY`** 的含义一直是：「**本条包里是一整段 SQL 文本**，服务端**当场解析并执行**」。包体格式约定就是 **命令字节 + SQL 字符串**（可加 **`CLIENT_QUERY_ATTRIBUTES`** 等扩展），**不**带 stmt id、也不走「先 PREPARE 再 EXECUTE」那条状态机。  
   **二进制参数、二进制结果集、stmt 生命周期** 被放在 **`COM_STMT_*`** 里，形成**另一条、边界清晰**的路径。

2. **兼容与生态**  
   几十年来客户端都假定：**发 `COM_QUERY` → 拿文本结果集**（列定义 + 文本化行）。若把 **`COM_QUERY` 的响应**改成与 **`COM_STMT_EXECUTE` 完全相同的二进制行格式**，**所有旧驱动、代理、抓包工具**都会失效，除非再发明能力位与双格式分支，复杂度会堆在**最常用**的那条命令上。

3. **已有专用通道**  
   需要「二进制打包」的执行路径时，协议层给出的答案就是：**走 `COM_STMT_PREPARE` + `COM_STMT_EXECUTE`**，而不是把 `COM_QUERY` 做成二合一。

4. **请求侧的小扩展**  
   较新的 **`CLIENT_QUERY_ATTRIBUTES`** 等可以在 **`COM_QUERY`** 包内附带**类二进制**的命名参数区，但**SQL 主体通常仍是文本**，且**响应**仍按 **`COM_QUERY` 的文本结果集** 规范走——这属于在**不推翻 QUERY 语义**的前提下做增量，而不是把 `COM_QUERY` 整个改成 STMT 那种二进制对话。

**一句话**：**`COM_QUERY` 被定义为「即席文本 SQL + 文本结果集」的简单路径；二进制 PS 是 `COM_STMT_*` 的职责。拆开是为了语义清楚、兼容稳定；要二进制行就用 STMT 流程。**

### Q12：二进制协议是不是更优？

**没有统一答案，要看场景。**

| 维度 | 文本 `COM_QUERY` 往往更合适 | 二进制 `COM_STMT_*` 往往更合适 |
|------|------------------------------|--------------------------------|
| **即席 SQL、只跑一次** | 一次往返、无 PREPARE 开销 | 至少 PREPARE + EXECUTE，通常**更慢、更费**（见 Q6） |
| **同一句 SQL 执行很多次、主要换参数** | 每次可能重复传长 SQL、重复解析 | 解析摊在 PREPARE，EXECUTE 只带 stmt_id 与二进制参数，**往返与 CPU 常更省** |
| **大行集/多列** | 文本编码可能更长 | 紧凑类型编码有时**省带宽**（需实测） |
| **调试与抓包** | 人类可读 | 可读性差 |
| **实现复杂度** | 驱动/服务端路径简单 | 需管理 stmt 生命周期、stmt_id、`max_prepared_stmt_count` 等 |

**结论**：**二进制 PS 不是「升级版协议」**；它是为 **重复执行、参数化** 优化的**另一条路径**。即席查询用 **`COM_QUERY`** 通常更简单、单次更划算；**别为用二进制而用二进制**。

### Q13：若是全新设计的数据库，还有必要保留「文本 + 二进制」两套吗？

**不必须。** MySQL Classic 的双路径是 **历史演进 + 海量兼容** 的结果；**从零设计**可以更激进地统一。

常见取舍：

| 思路 | 说明 |
|------|------|
| **统一成一种线格式** | 例如**一律**用带类型标签的帧（Protobuf、MessagePack、自描述二进制），**即席查询**也发「操作码 + SQL 字符串或 AST 句柄」；**重复执行**发「语句句柄 + 参数块」。对外仍是一种协议，内部再分子消息类型，而不是两套「哲学」并立。 |
| **只保留 RPC + 结构化请求** | 不设「整段 SQL 文本」的独立路径：客户端只发 **parse / execute / close** 等 RPC，由服务端缓存执行计划；交互全是结构化字段，**天然偏二进制**，文本 SQL 仅出现在「parse 请求的一个字符串字段」里。 |
| **应用层与传输层分离** | 对外 **HTTP/JSON**（人类友好），对内或高性能客户端走 **同一语义的二进制编码**；不是两条互不相关的协议，而是**同一 API 的两种序列化**。 |
| **刻意保留「双轨」** | 若仍要兼容「telnet 里手打 SQL」「万能抓包可读」，可保留 **明文文本会话** 与 **二进制会话** 两种模式，但现代产品里往往用 **TLS + 专用客户端** 弱化这条需求。 |

**实践建议**：新系统更常见的是 **一种线协议 + 多种消息类型**（或 gRPC 式 unary/stream），用 **版本号/能力协商** 扩展，而不是复制 MySQL「`COM_QUERY` vs `COM_STMT_*`」这种 **两条完整故事线**。是否保留「看起来像两套协议」，取决于你是否还要服务 **无结构客户端、运维随手连、生态里海量假定文本结果** 的场景。

### Q14：MySQL 为什么不沿用 `COM_QUERY` 做 PS，而要单独搞一套二进制子协议？（网上能查到的说法）

官方与资料里的要点可以归纳为 **「能力需求不同 + 演进方式」**，而不是「`COM_QUERY` 技术上绝对做不到」。

**1. 官方对「子协议」的定位（MySQL 4.1 起）**  
源码文档 [Prepared Statements（Command Phase）](https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_command_phase_ps.html) 写明：预处理子协议 **新增** `COM_STMT_*` 命令，并定义 **比 Text Resultset 更紧凑的结果集格式**，用于 **替代** `COM_QUERY` 响应里的文本结果集（在同一文档体系中对照 [Text Resultset](https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_com_query_response_text_resultset.html)）。也就是说，设计上是 **专门一条命令族 + 专门一种结果编码**，与「整包 SQL 字符串」的 QUERY 模型并列。

**2. 常见文档里写的「为什么要 PS / 二进制」**（性能与安全，多来自手册与 4.1 时代指南类资料）  
- **解析只做一次**，重复执行主要传参数，减少 **重复解析与长 SQL 反复占带宽**（如镜像的 [Using Server-Side Prepared Statements](https://litux.nl/mirror/mysqlguide4.1-5.0/0672326736/ch06lev1sec9.html) 等）。  
- **参数与（在 STMT 路径上的）结果**用 **原生二进制形态**，减轻字符串往返转换。  
- **参数化**有利于 **防 SQL 注入**（见 [MySQL 手册 Prepared Statements](https://dev.mysql.com/doc/refman/en/sql-prepared-statements.html) 等）。  
这些目标用 **「先 PREPARE、再 EXECUTE、带 stmt_id」** 的状态机表达最直接；若全部塞进 **`COM_QUERY` 一种包型**，要在 **兼容旧客户端「只认纯文本 SQL + 文本结果」** 的前提下做能力协商与多形态负载，**复杂度和风险**会集中在最热路径上。

**3. Oracle 工作日志：承认「二进制结果」曾与 PS 绑定**  
[WL#3359: Decouple binary protocol from prepared statements](https://dev.mysql.com/worklog/task/?id=3359) 标题即 **「把 binary protocol 与 prepared statements 解耦」**——说明在提出该任务时的 **MySQL 实现/产品形态里，二者是绑在一起的**：要用 **二进制方式传结果集**，在实践上往往就得走 **prepared statement** 那条路（因而也要承受 PS 的折衷）。WL 正文写得很直白：用 **prepared statements** 有 **额外往返**、**query cache** 等代价；而 **二进制结果集对性能几乎总是好的**，所以希望 **即使不用 PS 也能用二进制结果**（例如用 `COM_SET_OPTION` 等开关切换结果格式——任务里讨论的方案）。  

**你的理解成立**：**WL#3359 表明（至少在当时语境下）二进制协议/二进制结果集与 prepare statement 路径是耦合的**；该 WL 的方向正是把这种耦合拆开。这也与 4.1 起把 **紧凑结果集格式** 放在 **`COM_STMT_*` 文档树**里、与 **`COM_QUERY` 文本结果集**并列的演进一致。

**4. 与本文 Q11 一致的设计取舍**  
**向后兼容**、**命令语义单一**（QUERY = 即席文本 + 文本结果）、**新能力用新命令扩展**，与网上/手册强调的 **性能与参数化** 合在一起，就是「为什么线上看到的是 **`COM_STMT_*` 子协议**」的常见答案。

### Q15：MySQL X Protocol 是什么？为什么要新搞一套？

**X Protocol** 是与 **Classic（`COM_*`）并列的另一条客户端–服务器协议：消息多用 **Protocol Buffers（Protobuf）** 编码，经 **X Plugin** 提供服务（常见监听 **33060**，与 Classic 的 **3306** 分离）。**不能**与 Classic 混在同一条连接里。

**为什么提出（官方叙事要点）**  
随 **MySQL 5.7.12** 推出 **Document Store / X DevAPI** 这一整条能力：在**同一数据库内核**上同时服务 **关系型 SQL** 与 **文档/JSON 集合、CRUD 式 API**，减少「SQL vs NoSQL、多套技术栈」的割裂。官方博客说明需要 **「支持这些扩展能力的新协议」**，并配套 **MySQL Shell、Connector/Node.js 等** 与 **X DevAPI**（方法链、管道化等）。参见 [MySQL 5.7.12 Part 6: Document Store](https://dev.mysql.com/blog-archive/mysql-5-7-12-part-6-mysql-document-store-a-new-chapter-in-the-mysql-story/) 与手册 [Document Store](https://dev.mysql.com/doc/refman/8.0/en/document-store.html)（版本可选 5.7/8.0）。

**与 Classic 的分工（理解用）**  
- **Classic**：海量存量工具与驱动、**SQL 会话、`COM_QUERY`/`COM_STMT_*`**，长期稳定兼容。  
- **X Protocol**：**DevAPI、文档集合、新式 CRUD**，用 **Protobuf 消息** 扩展，**不必**把文档/CRUD 硬塞进 30 年前的 `COM_*` 帧格式。

技术细节与消息定义见 [X Protocol / Messages（dev 文档）](https://dev.mysql.com/doc/dev/mysql-server/latest/mysqlx_protocol_messages.html) 等。

### Q16：哪些「新能力」**必须**走 X Protocol？（或强依赖 X Plugin）

依据手册与常见用法，可这样区分（**不等于**「这些功能在库里不存在」，而是 **访问路径** 要求 X）：

| 类别 | 是否依赖 X Protocol | 说明 |
|------|---------------------|------|
| **把 MySQL 当 Document Store 用**（集合、文档 CRUD、以 X DevAPI 为主访问模型） | **是** | 手册写明：使用 Document Store 需要 **X Plugin**，使服务器能用 **X Protocol** 与客户端通信，这是 **prerequisite**。见 [Using MySQL as a Document Store](https://dev.mysql.com/doc/refman/8.0/en/document-store.html)（8.0；其他版本同章）。 |
| **X DevAPI**（各语言 Connector / Shell 里 `mysqlx`、集合/表 CRUD、管道化等） | **是** | 文档称兼容 X Protocol 的客户端包括 **MySQL Shell**、**带 X DevAPI 的 Connectors**；开发时用 **X DevAPI** 即按 **X Protocol** 连（通常 **33060**）。 |
| **在 X 会话里执行 SQL** | **走 X，但不是「SQL 本身只有 X」** | 手册写 **X Protocol supports both CRUD and SQL operations**——SQL 也可经 X 发，但**传统 SQL 应用**仍普遍用 **Classic（3306）**，**不强制** X。 |
| **普通关系型 SQL、JDBC 经典 URL、`mysql` 客户端** | **否** | 默认 **Classic**。 |
| **InnoDB Cluster / AdminAPI（`dba.*` 管理集群）** | **否（限制用 Classic）** | Shell 文档写明：AdminAPI 相关操作用 **Classic**、**TCP/IP**；**不支持**对 AdminAPI 使用 X Protocol（实例间连接等也有类似限制）。见 [InnoDB Cluster Limitations](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-innodb-cluster-limitations.html) 等。 |

**一句话**：**「必须 X」的主要是 Document Store + X DevAPI 这条产品线的客户端访问方式**；**内核里的 JSON、SQL、复制等能力大多仍可通过 Classic 使用**，AdminAPI 反而要求 **不要用 X**。

---

## 三、和长篇设计文档的关系

| 需求 | 建议阅读 |
|------|----------|
| 源码文件、`parse_packet`、`dispatch_command`、PS 实现纲要 | [mysql-protocol-design.md](./mysql-protocol-design.md) §1、§5、§11 |
| `mysqltest` `ps_re`、哪些语句不走 PS、MTR 分工 | 原文 §12、§13 |
| 字节级字段、各 `COM_*` 包布局 | `protocol_classic.cc` 内 Doxygen + [MySQL Internals](https://dev.mysql.com/doc/internals/en/client-server-protocol.html) |
| X Protocol / Document Store / 为何与 Classic 分立 | 本文 **Q15**；**哪些能力必须 X** 见 **Q16** |

---

*本文 **Classic 细节** 与 `mysql-protocol-design.md` 对齐（MySQL 8.0.41）；**X Protocol** 仅 **Q15** 作入门，不深究消息级规范。*
