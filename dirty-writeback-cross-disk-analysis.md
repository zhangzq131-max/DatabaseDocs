# Linux 3.10 全局脏页回写与跨盘带宽现象分析

本文档整理自对 **Linux 3.10**（RHEL/CentOS 7 系内核 `linux-3.10.0-862.14.4.el7`）源码的讨论，并补充不同内核世代对该类问题的处理演进。场景描述（mysqld 在数据盘、系统盘另有大量写入）仅为说明用途，实际现象需结合 blktrace、`/proc/vmstat`、`/sys/class/bdi/` 等数据验证。

---

## 1. 现象简述（可能与实际存在偏差）

- **进程 A（mysqld）**：频繁创建、写入、删除临时文件；单独运行时数据盘上“刷脏”不剧烈，磁盘带宽通常不会持续打满。
- **进程 B**：在**系统盘**上大量写入页缓存脏数据。
- **观察**：当 B 活跃时，**数据盘**回写与带宽占用明显升高，甚至打满。

**注意**：若数据盘与系统盘共用同一队列/RAID/控制器，或存在绑定 IO 调度与硬件瓶颈，也会产生“看似跨盘连带”的效果，需与下文内核机制区分。

---

## 2. 问题在 3.10 代码中的根源（全局阈值 + 遍历所有 BDI）

### 2.1 全局脏页阈值与 BDI 份额

- `vm.dirty_background_ratio` / `vm.dirty_ratio`（或对应的 `*_bytes`）基于**可脏内存**计算**全局**背景与上限阈值（`global_dirty_limits()`，见 `mm/page-writeback.c`）。
- 每个块设备对应一个 **BDI**：`backing_dev_info`；脏页计数分为全局（如 `NR_FILE_DIRTY`）与 per-BDI（如 `BDI_RECLAIMABLE`）统计。

### 2.2 `over_bground_thresh()`：先看全局，再看该 BDI

在 `fs/fs-writeback.c` 中，是否“仍高于后台阈值”的判断包含两层：

1. **全局**可回收脏页是否超过全局 `background_thresh`。
2. **该 BDI** 的可回收脏页是否超过 `bdi_dirty_limit(bdi, background_thresh)`（基于 `writeout_completions` 等动态分配各设备脏页“份额”）。

因此：**只要全局脏页总量超过背景线**，后台写回逻辑会认为仍处于“需要后台刷写”的状态，与各盘上**谁**在写脏无直接一一对应关系。

### 2.3 `wakeup_flusher_threads()`：有脏 IO 的 BDI 都可能被拉活

当需要整体推进写回时（例如回收路径、`sync`、显式唤醒等），`wakeup_flusher_threads()` 会遍历 `bdi_list`，对**存在脏 IO**的 BDI 投递写回工作。这与“只有系统盘在写”的直觉不一致：**全局压力升高时，其它盘上已有脏页的 BDI 也会被一并推进写回**。

### 2.4 `balance_dirty_pages()`：写脏任务的限速也受全局与该 BDI 共同约束

用户态/内核路径在弄脏页后会通过 `balance_dirty_pages_ratelimited()` 进入 `balance_dirty_pages()`。其中使用：

- 全局 `nr_dirty` 与全局 `dirty_thresh`；
- 本 BDI 的 `bdi_dirty`、`bdi_thresh`（来自 `bdi_dirty_limit()`）；
- `bdi_position_ratio()` 等控制“在全局与 BDI 两层上的位置误差”。

因此另一磁盘上的大量写脏会抬高全局脏页水位，从而**间接**影响 mysqld 所在 BDI 上进程的脏页限速与睡眠节奏（不一定直接等于“只刷系统盘”，但会改变全局与 BDI 的平衡状态）。

### 2.5 mysqld 仅自己运行时常“看起来刷脏少”的原因（便于对照）

临时文件若寿命短，脏页可能在未达到全局高压前就被截断/删除；全局 `NR_FILE_DIRTY` 可能长期处于背景阈值以下，从而较少触发“全系统级”的写回压力与其它 BDI 的并发刷写。该推论需用实际计数器验证。

---

## 3. 小结：3.10 上的机制性解释

| 机制 | 作用 |
|------|------|
| 全局脏页阈值 | 用**整台机器**的脏页量约束内存与回写，不区分业务与磁盘。 |
| per-BDI `bdi_dirty_limit` | 在多个设备之间分配“脏页预算”，并参与限速；**不能完全隔离**“系统盘写入”与“数据盘回写”在全局水位上的联动。 |
| `wakeup_flusher_threads` | 按 BDI 逐个唤醒；**任一 BDI 有脏 IO**时都可能参与写回。 |
| 调度/块层 | CFQ 等同代 I/O 调度与队列深度会进一步影响观测到的带宽与延迟。 |

**一句话**：在 3.10 上，**全局脏页控制是主轴**；系统盘把全局水位抬高后，数据盘上**本就存在的脏页**会更积极地被写回线程处理，表现为数据盘带宽上升。

---

## 4. 不同 Linux 内核版本对该类问题的处理方式（演进概览）

以下按“是否更容易从内核侧隔离/限流”为主线，版本为上游主线大致世代；发行版会 backport 部分补丁，以具体 `uname -r` 与发行说明为准。

### 4.1 Linux 3.10 及同代（如 RHEL7 基线）

- **全局 `dirty_*` 阈值** + **per-BDI 统计与 `bdi_dirty_limit`**，无主线的 **per-cgroup 写回域**。
- 块 cgroup（blkcg）在 3.10 已存在**限速思路**，但与页缓存写回的精细归属、独立 wb 队列相比，后续版本集成度更高。
- **缓解手段**多为调 `vm.dirty_*`、业务分层盘、避免与其它写盘任务共享瓶颈、升级内核或换用支持更细粒度策略的环境。

### 4.2 Linux 4.2：cgroup 写回（重要分水岭）

- 引入 **cgroup writeback**：对 **memory cgroup（memcg）** 与 **blkcg** 的联合场景，使写回路径能把 IO **归因到产生脏页的 cgroup**，并为非 root cgroup 建立独立的 **`bdi_writeback`（wb）执行域**（每个 memcg–blkcg–bdi 组合可对应独立 wb）。
- **意义**：可在统一 hierarchy 下做“**谁弄脏谁回写**”的记账与调度，减轻“全局脏页一高、所有有脏 BDI 都被拖着跑”的粗放感（实际效果依赖 workload 与是否启用 cgroup v1/v2 及控制器挂载方式）。

参考：上游合入主题为 4.2 的 cgroup writeback 相关工作（LKML “Cgroup writeback support for 4.2”）。

### 4.3 Linux 4.5+（约）：cgroup v2 块控制器雏形与后续打磨

- **cgroup v2** 逐步成为推荐统一层次；**io**（块）控制器在后续小版本持续修正（不同子版本功能集不同）。
- 使用 **cgroup v2 + io** 控制器可对**业务 cgroup** 做 **权重/上限** 等限制，与 memcg 叠加时，才更易把“系统盘大批写”与“数据库盘”在**资源语义**上分开（仍取决于进程是否划入不同 cgroup）。

### 4.4 Linux 5.0+：blk-iocontrol 与 io.cost 等（版本以具体补丁为准）

- 出现如 **blk-iocost**（5.x 主线中逐步可用，具体从属版本查发行版）等机制，用 **延迟/成本** 模型做更平滑的 cgroup IO 控制，减轻仅靠队列深度与限速的抖动。
- **意义**：在多租户或混合负载下，比单纯依赖全局 `vm.dirty_*` 更易做**公平性与上限**控制。

### 4.5 Linux 5.10+ / 5.15+ LTS：维护性与控制面成熟

- cgroup v2、io 控制器、`memory.high`/`memory.max` 等与脏页、回收、写回的交互在主线与 LTS 上持续修补。
- 长期演进还包括 **writeback 中心结构**（multiqueue、per-node wq 等）的改进，改善伸缩性与延迟；具体行为随版本差异较大。

### 4.6 汇总对比表

| 内核世代 | 对“全局脏页抬高 → 多盘同时刷”的缓解方向 |
|----------|----------------------------------------|
| **3.10** | 主要靠全局 `vm.dirty_*`、磁盘与业务隔离；无主线 cgroup 写回域 |
| **4.2+** | cgroup 写回：脏页归属与 wb 域细化，便于按 cgroup 治理 |
| **4.5+ / v2** | cgroup v2 + io：更易对进程组做 IO 上限与权重 |
| **5.x** | io.cost 等、blk 层控制更细；需查具体 LTS/发行版是否启用 |

---

## 5. 实践建议（与版本无关的排查与缓解）

1. **验证**：`grep nr_dirty /proc/vmstat`；各 BDI `grep . /sys/class/bdi/*/read_ahead_kb` 与同路径下统计；观察 `flush-*` 内核线程与 `iostat` 是否数据盘写放大。
2. **调参**：在可接受范围内调整 `dirty_background_ratio` / `dirty_background_bytes`、`dirty_ratio` / `dirty_bytes`，观察全局水位与业务延迟的权衡。
3. **分层**：系统日志与批量写与数据库数据盘分离；避免与 mysqld 共用同一慢速通道。
4. **升级路径**：若需按服务隔离回写与 IO，评估迁移到 **较新内核 + cgroup v2（memory + io）** 与发行版文档中的启用步骤。

---

## 6. 参考源码位置（本仓库 3.10 树）

- `mm/page-writeback.c`：`global_dirty_limits`、`balance_dirty_pages`、`bdi_dirty_limit`、`bdi_position_ratio` 等。
- `fs/fs-writeback.c`：`over_bground_thresh`、`wakeup_flusher_threads`、`wb_check_background_flush`、`bdi_writeback_workfn` 等。
- `mm/backing-dev.c`：`bdi_wq`、BDI 注册与写回 work 初始化。

---

## 7. 异步刷脏与平时刷脏：频率、行为差异及对数据盘的影响

从「谁在刷、多久刷一次、每次刷多大、刷谁」几个方面看差异，用于解释：**mysqld 平时磁盘写入很小，但别的进程在系统盘大量写入后，数据盘写入会变得很大**。

### 7.1 频率：平时 vs 整机脏页压住背景线之后

**平时（整机脏页压力较低）**

- 主要靠 **周期性回写**：由 `dirty_writeback_interval`（默认约 5 秒量级）驱动的 **kupdate 风格**遍历，优先处理在时间上「足够老」的脏 inode。
- **`over_bground_thresh()` 常为假**：全局可回收脏页未达到 `dirty_background_*`，`wb_check_background_flush` 里 **`for_background`** 路径往往 **很快退出**——不满足「仍高于后台阈值」时就不会长跑式刷。
- **mysqld 临时文件**：若脏页生命周期只有秒级，未必被周期扫到即已 **unlink/关闭** 并走页缓存回收；脏页有机会 **丢弃而不落盘**——对应观测上「平时数据盘写入很小」。

**系统盘上其它进程大量写页缓存之后（整机脏页持续压住背景阈值）**

- **全局脏页长期高于 `dirty_background_thresh`**：`over_bground_thresh()` 对相关 BDI 更容易 **长时间为真**。
- **数据盘 BDI**：`wb_check_background_flush` 中会构造 **`nr_pages = LONG_MAX`**、**`for_background = 1`** 的作业，即在仍超阈值期间 **持续推进** inode 链表上的刷写。
- **`balance_dirty_pages()`** 也会在跨线时 **`bdi_start_background_writeback`**，delayed work **插队更频繁**。
- 结果：**后台刷写轮转更勤、单次逻辑推进上限更大**，mysqld 临时文件那一段 **短暂的脏窗口**里，inode 被选中去 `writepages` 的概率明显上升；即便文件最后被删除，**删之前可能已经提交整块设备写入**——数据盘 **`wkB/s` 会陡增**。

**一句话**：平时偏「低频扫旧脏 + 阈值以下即止」；加压后是「过线则 `LONG_MAX` 级持续推进 + wakeup 更勤」，二者频率与持续时间差一个量级并不少见。

### 7.2 行为：仍为异步，但干劲不同

| 维度 | 平时偏多 | 全局脏页抬升之后 |
|------|----------|-------------------|
| 写回模式 | 仍以 `WB_SYNC_NONE` 机会主义为主 | 同样是异步刷脏；差异在 **触发条件与推进强度** |
| `nr_pages` | 周期性任务中与全局估算等相关 | **`for_background` 时使用 `LONG_MAX`**，逻辑上在条件满足时可长时间刷 |
| 是否挑磁盘 | 各 BDI **自己的 inode 链表** | 仍为 per-BDI，但全局过线会使 **数据盘 flusher 「长跑」** |

**说明**：自始至终都不是 `fsync` 那种 **`WB_SYNC_ALL`** 语义；差别在于 **阈值是否踩住、`for_background` 是否持续生效**，以及单次/累计刷写量。

### 7.3 为何「写系统盘 → 数据盘写放大」可与源码对上号

1. **全局阈值一套**：系统盘推高 `NR_FILE_DIRTY`，**全局背景水位被踩住**，数据盘对应 BDI 同样落入「要重视后台刷」的状态（见 §2 `over_bground_thresh`）。
2. **异步刷赃更狠、窗口更长**：临时文件脏页在仍存在 mapping 的时间内更易被 **`queue_io` / `writeback_sb_inodes`** 推到块层；延迟分配等在真正提交 BIO 时可表现为 **大块顺序写**。
3. **`bdi_dirty_limit` / `writeback_chunk_size` 等为第二层节拍**：仍可调节节奏，但在 **整机长期过线 + 多台子交替脏**时，观感常是 **数据盘长时间很忙**，而非「内核只刷系统盘」。

### 7.4 建议用的观测（不涉及改内核）

配合验证：

- `/proc/vmstat`：`nr_dirty`、`nr_writeback` 等是否在 **先于或同步于**数据盘写放大；
- `/proc/meminfo` 中 **Dirty** 与 **`vm.dirty_background_*`** 的对比；
- `iostat -x`：数据盘 `wkB/s` 与 **`flush-<device>`** 活跃时段是否与系统盘大批量写重叠；
- **调 `dirty_background_bytes`** 做实参对比试验，往往能复现「系统盘一开闸 → 数据盘跟涨」幅度的变化。

---

## 8. 免责声明

内核行为受 **硬件、文件系统、InnoDB 刷盘策略、RAID 写策略、是否与 tmpfs/同一控制器的块设备** 等多因素影响。本文仅基于公开主线演进与 3.10 源码结构的机制说明，**不构成对生产环境的直接调参处方**。
