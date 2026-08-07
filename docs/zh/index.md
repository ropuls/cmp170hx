# CMP 170HX 维基

**本页涵盖：** NVIDIA CMP 170HX 是什么、社区从它身上夺回了什么、今天确切能做什么，以及根据你的目的该去哪里继续。

CMP 170HX 是一款加密货币挖矿加速卡，基于 **GA100** —— 与 NVIDIA A100 相同的 826 mm² 7 nm 芯片。它有两个型号：PCI ID `10de:20c2` 报告 8192 MiB，`10de:2082` 报告 10240 MiB，两者都暴露 70 个流式多处理器、4480 个 CUDA 核心和 280 个第三代张量核心，计算能力 8.0。NVIDIA 故意用四种独立方式削弱它：SM 发射速率被熔丝降到约 1/32（FP32 FMA 和所有张量核心路径），HBM 容量被限制到堆栈实际容量的零头，PCIe 链路被限制在 Gen1 速率且只能协商其 16 条布线通道中的 x4，NVLink 和 ECC 被熔丝关闭。2023 年一次仔细的拆解得出结论：这种组合"保证了 GPU 的无用"，并因固件签名而判定这些限制无法破解。

在 2023 年至 2026 年 7 月之间，一个分布式社区用纯软件方式夺回了大部分能力——没有刷写 VBIOS，也没有伪造签名。已发布的解锁器是针对 NVIDIA 开源内核模块的六补丁集合。它用精心构造的签名载荷重新触发 SEC2 Booter Load，以打开四个特权级别掩码，然后在 GSP 启动窗口内执行四次寄存器写入：`0x0082381c` = `0x88888888` 和 `0x00823820` = `0x00000008` 恢复完整 SM 发射速率，FBPA CFG1 `0x009a0204` 加上 MMU LMR `0x00100ce0` 恢复真正的 A100 内存几何结构。实测结果：FP32 非张量从约 0.39 提升到约 12.6 TFLOPS（26 到 32 倍增益），BF16 张量从 6.4 提升到 171–193 TFLOPS，FP64 在 torch GEMM 上达到 11.6 TFLOPS（张量路径：一次 clpeak 转储在同一运行中打印 6.31 TFLOPS 标量对 11.96 的 `wmma_fp64`），8 GB 卡报告并使用 **65536 MiB**。整个过程大约消耗一秒驱动加载时间，每次模块加载都会重跑，且不写入任何 flash。一次会话中基准测试的八张租用卡差异低于 2.5%，并通过了 8/8 全显存字节对比完整性测试。

> [!WARNING]
> **阅读其他内容之前，有两点必须先搞清楚**
>
> **容量按型号区分，不可互换。** 8 GB 卡解锁为 **64 GB**。10 GB 卡解锁为 **40 GB**，40 GB 是受支持的配置。针对 10 GB 卡的 80 GB 驱动分支已构建、测试并放弃：它报告 81920 MiB，但实际使用超过约 40 GB 就会失败。一套独立的、脚本驱动的相干寄存器组确实能到达 40 GiB 以上的真实内存，但它未发布、处于实验阶段，且大约每次触发只能得到一个 CUDA 上下文。
>
> **PCIe 速率和 PCIe 宽度是两个不同的问题，有两种不同的修复。** Gen1 到 Gen2 是*软件*解锁，只存在于未发布的分支上。超过 x4 *宽度*需要手工焊接 24 个交流耦合电容到板卡上。两者互不相干。

## 当前状态

| 能力 | 状态 | 详情 |
|---|---|---|
| 移除 SM 节流（计算） | **已发布，稳定** | 两次寄存器写入；FLR 后仍然有效。[工作原理](unlock/compute-throttle.md) |
| 8 GB 卡到 64 GB | **已发布，稳定，生产可用** | CFG1 `0x02779000`，LMR `0x0000020B`。[内存几何](unlock/memory-geometry.md) |
| 10 GB 卡到 40 GB | **已发布，稳定** | CFG1 `0x02669000`，LMR `0x0000028A` |
| 重启后保持 | **自动** | 每次 GSP 启动时重新应用；不刷写任何内容。[安装](procedures/install.md) |
| 多卡机组 | **可用** | 已实测 8 卡机组，见[多 GPU](procedures/multi-gpu.md) |
| GPC 时钟偏移 / 降压 | **通过 NVML 可用** | 8 GB 型号上 `[-1000..+1000]` MHz。[调校](operations/tuning.md) |
| 功耗限制（`nvidia-smi -pl`） | **可用** | 出厂 100–250 W，OC VBIOS 上 300 W。[功耗](operations/power-and-psu.md) |
| PCIe Gen2（链路**速率**） | **可用，仅未发布分支** | 不在已发布的 `master` 中；现场表现不确定。[Gen2](unlock/pcie-gen2.md) |
| PCIe x16（链路**宽度**） | **仅硬件改造** | 24 × 0402 220 nF X7R 电容。[物理改造](operations/physical-mods.md) |
| Gen2 与 x16 同时 | **仅观察到一次** | 6.63–6.67 GB/s，一台机组，2026-07-26，中等置信度 |
| 10 GB 卡到 80 GB | **分支被拒；相干寄存器组实验性** | `80` 分支报告 81920 MiB，超过约 40 GB 即失败。一次无驱动相干触发通过了 77.5 GiB 无折叠测试，但每次触发只能得到一个 CUDA 上下文。[80 GB](frontier/80gb.md) |
| PCIe Gen3 / Gen4 | **未解决** | 两个代际熔丝都读为 `0x00000001`。[Gen3/Gen4](frontier/pcie-gen3-gen4.md) |
| 超过 70 个 SM | **未解决** | 所有通往 GPC 禁用熔丝的写入路径都被锁存 |
| ECC | **未找到杠杆** | 熔丝关闭，无遥测，`MASTER_EN` 只读。[ECC](frontier/ecc.md) |
| NVLink | **此板卡上不可能** | 已熔丝关闭。板卡侧接口 IC 是否焊装仍在讨论中，倾向于未焊装。[NVLink](frontier/nvlink.md) |
| 对等直连（P2P） | **缺失** | [P2P](frontier/p2p.md) |
| MIG | **单一报告，未发布** | `0x820840` 第 0 位；等待第二张卡和 pull request |
| 待机功耗降低 | **无杠杆** | 只存在性能状态 P0；`nvidia-pstated` 返回 `NVAPI_ERROR` |
| 显存时钟控制 | **NVML 拒绝**，但补丁模块可达 | NVML MEM VF 偏移范围是 `[0..0]`，`-lmc` 不受支持，但补丁模块加重启已在实践中降低 HBM 时钟：1728 MHz → 212.2 TF / 181.2 W；1620 MHz（NDIV 60）→ 211.6 TF / 172.9 W；1404 MHz（NDIV 52）→ 210.5 TF / 169.3 W |
| VBIOS 修改 | **对解锁杠杆关闭** | 容量捆扎和 PCIe 代际捆扎位于 Davies-Meyer MAC 范围内。无符号的 FwSec 尾部在其之外，确实包含可编辑字段，包括 `0x45E45` 处的板卡功耗上限和 `freqDelta`，但写入它们需要 CH341A 夹子。[VBIOS](hardware/vbios.md) |

已发布 `master` 上受支持的驱动恰好是 **`610.43.03`**（默认）和 **`610.43.02`**；其他任何版本构建都会硬失败。595 / 590 / 580 的移植存在于一个分支上，已通过源码验证，但从未被报告成功启动。

## 从这里开始

**"我刚买了一张，想让它工作。"**
先读[这是什么卡](start/what-is-this-card.md)了解背景，然后[识别你的卡](start/identify-your-card.md)确定你持有哪个型号（这决定后续一切），再读[风险](start/risks.md)，然后[快速入门](start/quick-start.md)和[安装](procedures/install.md)。在通电之前，务必阅读[散热](operations/cooling.md)和[功耗与电源](operations/power-and-psu.md)：这张卡是被动散热、没有风扇，它的单个 8-pin 插座是 **EPS** 插座，不是 PCIe 插座。把 PCIe 线缆强行插进去会损坏板卡。安装完成后用[验证](procedures/verify.md)确认，而不是靠读 dmesg。

**"我想解锁它，而且想知道我到底在敲什么。"**
先看[解锁总览](unlock/overview.md)，然后[工作原理](unlock/how-it-works.md)。机制分为[Falcon 与 Booter](unlock/falcon-and-booter.md)、[特权级别掩码](unlock/privilege-level-masks.md)、[计算节流](unlock/compute-throttle.md)和[内存几何](unlock/memory-geometry.md)。如果出了问题，[故障排查](procedures/troubleshooting.md)按你看到的确切字符串组织。[驱动版本](procedures/driver-versions.md)解释为什么版本锁定不可协商。

**"我想在寄存器层面理解它是怎么工作的。"**
先看[硬件总览](hardware/overview.md)获取完整规格，然后是[GA100 芯片](hardware/ga100-silicon.md)、[熔丝与 OTP](hardware/fuses-and-otp.md)、[内存子系统](hardware/memory-subsystem.md)和[PCIe 子系统](hardware/pcie-subsystem.md)。项目中每个地址、数值和回读都收集在[寄存器参考](unlock/register-reference.md)中，并在[寄存器索引](appendix/register-index.md)中编目。[VBIOS 页面](hardware/vbios.md)解释为什么固件攻击路线是关闭的。

**"我想帮忙解决剩下的问题。"**
从[状态板](frontier/status-board.md)和[开放问题](frontier/open-questions.md)开始，它们按可解性排序，每个都附带具体的下一步实验。有几个很便宜：一次三方启动对比就能解决 `RMPcieLinkSpeed` 的 `0x1` 对 `0x2` 之争，一次头文件查找就能确定 LMR 量级字段宽度，一次常量修改加一次重启就能测试*相干* 80 GB 三重触发是否与 `80` 分支实际发布的非相干版本表现不同。先读[死路](history/dead-ends.md)：大量精力已经花在现已关闭的路径上，该页面精确记录了每条路径关闭的原因。

## 本维基如何标注置信度

读者必须一眼就能区分已定论的事实与活跃的猜测。

- **普通散文是已确认的。** 它不需要标记，因为不需要。当某个数值来自代码时，会指出源文件；当它来自实测时，会给出条件。
- `> [!WARNING]` **实验性** 告警标记未发布分支的材料和任何仅基于单一报告的内容。关于 PCIe Gen2 的一切都属于这一类，因为 Gen2 补丁从未合并到 `master`。
- `> [!CAUTION]` 告警标记任何可能损坏硬件或静默破坏数据的内容。本维基中最重要的实例并不显眼：在 1400 MHz 上限时，+325 MHz 时钟偏移**会在不崩溃的情况下破坏内存**，因此一个能跑完的运行并不能证明设置是安全的。
- `> [!NOTE]` **开放问题** 告警标记尚未有人解决的问题，并且总是说明尝试过什么、下一步会是什么。

仅基于单一观察的声明会在句子本身说明："一名测试者报告"、"某一天的一台机组"。出现在多个地方的数值已经对照单一规范值核对过，当两个来源确实不一致且没有证据能解决时，本维基会说明该值未知，而不是默默选择其一。完整的约定见[如何阅读本维基](start/how-to-read-this-wiki.md)，底层声明如何裁决见[方法论](appendix/methodology.md)。
