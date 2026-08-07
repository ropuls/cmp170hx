# 术语表

**本页涵盖：** 本 wiki 任何地方用到的每个缩写、寄存器昵称、工具名和行话，附准确全称和简短解释。它还标记了项目自己的文档分支编造、是错的、且仍被抄进第三方指南的缩写全称。

通篇适用两个约定。凡 NVIDIA 从未为某个内部块名发布过全称，本页就直说，而不是猜测。凡同一寄存器存在两个名字，就列在一条里，而不是两条。

---

## 更正过的全称：不要再重复这些 {#corrected-expansions-do-not-repeat-these}

`cmpunlocker` 的 `docs` 分支（`docs/docs/ARCHITECTURE.md`）包含五个缩写全称，它们不出现在发布版源码、任何分支快照或任何 NVIDIA 发布头文件中。它们是编造的。它们已经传播进下游指南。

> [!WARNING]
> **流传的错误全称**
>
> | 术语 | 错误全称（及出处） | 正确 |
> |---|---|---|
> | PLM | "Program Logic Modules"（`ARCHITECTURE.md` 第 38 行） | **Privilege Level Mask（特权级掩码）**，一个按寄存器划分的访问控制掩码 |
> | PMA | "Power Management Array"（第 30 行） | **Physical Memory Allocator（物理内存分配器）**，RM 内存管理对象 |
> | SS0 / SS1 | "Suspension State" 寄存器（第 29 行） | `FEATURE_OVERRIDE_SM_SPEED_SELECT`（`0x0082381c`）和 `..._SM_SPEED_SELECT_1`（`0x00823820`） |
> | LMR | "LM Request" / "LM (Local Memory) Request register"（第 28 行） | `NV_PFB_PRI_MMU_LOCAL_MEMORY_RANGE`，本地内存**范围（Range）** |
> | PMM | "the PMM (Permute Mask Model)"（第 41 行） | 代码中不存在这样的块。该术语是伪造的。 |
>
> 两个相关的事实错误随之传播：同一文件称 SS0 和 SS1 都被写入 `0xffffffff`（发布版补丁写的是 `0x88888888` 和 `0x00000008`），并称解锁靠"注入自定义 PLM 序列"实现（实际是用超大的签名缓冲区重跑 Booter Load 来打开四个具名的特权级掩码寄存器）。见[计算限速](../unlock/compute-throttle.md)和[特权级掩码](../unlock/privilege-level-masks.md)。

与该分支无关的两个进一步术语陷阱：

- **ROP。** 在普通 GPU 词汇中，ROP 指 *Raster Operations Pipeline*（光栅化操作管线）。在本 wiki 中，它处处指 **Return-Oriented Programming（返回导向编程）**，即用来驱动 SEC2 Booter 的利用技术。见 [ROP 链](../unlock/rop-chain.md)。
- **XR7。** 若干改装文章（以及本项目自己简报的早期草稿）说 PCIe 耦合电容是 "XR7"。介电代码是 **X7R**。见[物理改装](../operations/physical-mods.md)。

---

## A

**A100D**
:   NVIDIA DRIVE A100（`10DE:20BB`，板代码 PG199）的非正式名称，一块 32 GB GA100 部件，在本语料中只作为对比设备出现。在它上面观察到 Booter 状态 `0x54`，从未被解码。名为 `PG199` 的 `cmpunlocker` 分支不含任何 A100D 支持。

**ACR**
:   访问控制区域（Access Control Region）。NVIDIA 针对 GPU 微控制器的签名安全引导框架，负责划定 framebuffer 中的写保护区域，以及串行化安全引擎访问的互斥锁。持有的 ACR 互斥锁是解释 SEC2 邮箱卡住的反复出现的理由之一。

**AER**
:   高级错误报告（Advanced Error Reporting），标准 PCIe 错误日志能力。在健康的 170HX 上，`lspci -vvv` 在能力偏移 `[420]` 处显示 AER，且所有 UESta/CESta 位清零。判断 Gen2 或电容改装链路是否真的干净，AER 计数器是正确工具。

**AON 岛**（也称 **常开岛（always-on island）**、**GC6 岛**、**PGC6**）
:   GPU 中跨引擎复位保持存活的常供电域。其中的寄存器能扛过 [FLR](#f)；之外的不能。这个不对称是有关解锁最重要的单一结构事实：`FEAT_OVR_PLM`（`0x00823804`）、SS0 和 SS1 是 AON 的，能扛过 FLR，而 CFG1、每 FBPA CFG1、CSTATUS、LMR、FB 几何 PLM 和 AON LMR 影子 `0x001180f0` 不能。这就是为什么计算解锁先于显存解锁交付。机制描述本身（`SECURE_SCRATCH_14` 位于标记为 RW-4R 的 PGC6 域中）为中等置信度。

---

## B

**BAR0**
:   基地址寄存器 0（Base Address Register 0）。16 MB 内存映射寄存器窗口，本 wiki 中几乎每个寄存器都经它读写。工具通过 mmap `/sys/bus/pci/devices/<BDF>/resource0` 访问它。BAR0 全部读 `0xffffffff` 意味着卡已经从总线上掉下来。

**BAR1 / 可调整大小 BAR**
:   BAR1 是暴露给主机的 framebuffer 窗口。170HX 在 `[bb0]` 处通告一个物理可调整大小 BAR（Resizable BAR）能力，但窗口被限制为 64 MiB，所以大 BAR 技巧不可用。

**BAR2**
:   驱动自己的 `kbusVerifyBar2` 自检使用的 MMU 翻译窗口。解码为 `NV_ERR_MEMORY_ERROR`（`0x72`）并带日志字符串 `"BAR 0/BAR 2 failed."` 的失败，来自该测试命中 Booter 划出的 WPR2 区域，而不是损坏的显存。

**BDF**
:   总线:设备.功能（Bus:Device.Function），卡的 PCI 地址，例如 `0000:0a:00.0`。用户态辅助工具 `tools/retrain.sh` 中硬编码的 `0a:00.0` 是机器相关的 PCIe Gen2 失败的根因。

**Booter / Booter Load**
:   驱动在 [SEC2](#s) Falcon 上运行以认证并启动 [GSP-RM](#g) 的 NVIDIA 签名 ACR 引导加载程序微码。解锁的原理是给 Booter Load 一个故意超大的签名缓冲区，让一次受控溢出在 Booter 自己的特权上下文内执行 [ROP 链](../unlock/rop-chain.md)。Booter 在发布流程的每次运行中都报告 `0xffff`，无论成败，所以寄存器回读是唯一真正的判决。

**BSI scratch**
:   `0x001180xx` 安全暂存块（例如 `0x001180f8` 处的 `SECURE_SCRATCH_14`，以及 `0x001180f0` 处的 AON LMR 影子）。从 PL0 读它们返回 `0xbadf5108`。"BSI" 的全称在本语料中未被确立。

---

## C

**Canary**
:   栈金丝雀：Booter 在其保存的返回地址下方存储、返回前重新检查的随机逐启动值，以便检测天真的缓冲区溢出。170HX Booter 从 DMEM `0x6340` 加载其金丝雀。不匹配会以 SEC2 邮箱 `0x47` 恐慌。发布版载荷在构造的签名缓冲区的多个偏移处写入 **假金丝雀** 值 `0xc0deca7e`。

**CE**
:   拷贝引擎（Copy Engine）。GPU 的 DMA 引擎。两处相关：发布版补丁 0005 在这些卡上禁用基于 VAS 的 CE 清理路径；一份 Xid 31 抓取在 64 GB 窗口顶部把 `ENGINE CE2 HUBCLIENT_HSCE2` 命名为故障客户端。

**CFG0 / CFG1**
:   `NV_PFB_FBPA_CFG0` 和 `NV_PFB_FBPA_CFG1`，内存控制器配置寄存器。CFG1 是定义每分区寻址深度、也是主要显存解锁目标的寄存器。广播 CFG1 是 `0x009a0204`；按 FBPA 的单播副本是 `0x00900204 + n*0x4000`，n = 0..23。原厂 CFG1 在**两个** SKU 上都是 `0x02449000`；解锁后是 `0x02779000`（8 GB 卡）或 `0x02669000`（10 GB 卡）。字节 [23:16] 是层级：原厂 `0x44`、`0x66` = 每 FBPA 2048 MiB、`0x77` = 每 FBPA 4096 MiB。两张卡的每个活动分区上，实时每 FBPA CFG0 都读 `0x07981800`。

**CMP**
:   Cryptocurrency Mining Processor（加密货币矿卡处理器），NVIDIA 面向计算受限挖矿部件的产品线。CMP 170HX 是该线中基于 GA100 的成员，2021 年 9 月 1 日发布。

**CSTATUS_RAMAMOUNT**
:   每分区容量回读寄存器，位于 `0x0090020C + n*0x4000`。原厂时在两个 SKU 上都读 `0x200`（每 FBPA 512 MiB）。这里的 `0xbadf20NN` 值意味着该分区被熔断筛选，低字节编码实例号。

**CPU-RM**
:   单片驱动模式，资源管理器运行在主机 CPU 而非 GSP 上，用 `NVreg_EnableGpuFirmware=0` 选择。它把 SM 锁定在 1140 MHz 基频，而 GSP-RM 锁定在 1410 MHz。

**CYA_0**
:   BAR0 `0x0008c2c0`。bit 2 是 `DIS_G2`，即 Gen2 禁用。Gen2 分支清除它。

---

## D

**DEVINIT**
:   嵌入 VBIOS、在任何固件运行前执行的设备初始化脚本。若干未解决的限制（ECC、NVLink、可能还有 PCIe Gen3）被认为在 DEVINIT 时就已确立，这就是为什么启动后的寄存器级覆盖够不到它们。

**DKMS / srcversion**
:   DKMS 为每个内核重建树外内核模块。`srcversion` 是模块的源码哈希；比较 `/sys/module/nvidia/srcversion` 与 `/lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko` 的 srcversion，是判断实际运行的是打补丁模块还是原厂模块的定论性测试。

**DIO**
:   Falcon 的次级数据 I/O 边带接口，Booter 经它访问常开暂存寄存器。这三个字母在任何公开 NVIDIA 文档中都没有展开。对 `0x1180f8` 的一次被投毒 DIO 读取返回 `0xdead5ec1`。

**DLLLA**
:   数据链路层链路活动（Data Link Layer Link Active），PCIe 链路状态寄存器的 bit 13（`0x2000`）。GPU 总是报告 `LnkSta 0x1042` 且 DLLLA 清零，即使在已训练的 Gen2 x4 链路上也如此，所以补丁 `0008` 的成功谓词永不触发，"retrain completed without Gen2 link" 这一行在**每台主机上都是假阴性**。`0x7042` 是同一份抓取中*上游根端口*的 LnkSta，不是另一类主机；`0008` 读的是 GPU 的。

**DMEM / IMEM**
:   Falcon 数据内存和指令内存。独立工具在 DMEM `0x0900` 加载了 63,232 字节、在 IMEM `0x0000` 加载了 45,824 字节。IMEM 按 256 字节块对齐。一旦 Falcon 进入 [HS 模式](#h)，DMEM 既不能读也不能写：写入被静默丢弃，`DMEM_PRIV_LEVEL_MASK`（`0x00840284`）显示 `wr_prot == 0`。

**`dmem.bin`**
:   `/lib/firmware/nvidia/ga100/gsp/dmem.bin` 处可选的外部载荷覆盖。它是一个开发钩子。缺失时报告状态 `0x59`，这是正常、健康的路径。

---

## E

**ECC**
:   纠错码（Error-Correcting Code）内存。170HX 上熔断关闭（`FUSE_ECC_EN = 0x0`），无已知杠杆、无遥测：`nvidia-smi -q` 把每个 ECC 字段都报告为 `N/A`。名为 `ecc` 的分支完全没有 ECC 代码。见 [ECC](../frontier/ecc.md)。

**EPS 8-pin**
:   卡实际使用的 CPU 风格 8-pin 电源接口，额定 300 W，内部带两条独立的 12 V 轨。它**不是** PCIe 8-pin（额定 150 W），两者的 12 V 和地引脚分配也不同。见[风险](risks.md)。

---

## F

**Falcon**
:   NVIDIA 的小型嵌入式微控制器家族，常被展开为 *FAst Logic CONtroller*，以 SEC2、GSP 的引导核心、FECS 等形式存在。Falcon 有自己的 IMEM/DMEM、一个密码协处理器和硬件强制安全模式。

**FBHUB**
:   framebuffer 集线器，引擎客户端与 framebuffer 分区之间的交叉开关。`FBHUB_NUM_ACTIVE_LTCS`（`0x00100800`）在 8 GB 卡上读 `0x10`（16），在 10 GB 卡上读 `0x14`（20）。

**FBP / FBPA**
:   FBP 是 framebuffer 分区（framebuffer partition），包含 L2 切片和两个 FBPA 的显存子系统切片。FBPA（常展开为 *frame buffer partition adapter*，帧缓冲分区适配器）是 DRAM 控制器本身。8 GB 卡在 8 个 FBP 上有 16 个活动 FBPA，4096 位总线；10 GB 卡在 10 个 FBP 上有 20 个活动 FBPA，5120 位总线。探测工具走 24 个 FBPA 槽，因为完整 GA100 有 24 个。

**FECS**
:   图形管线中的前端上下文切换微控制器（FrontEnd Context Switch）。`FECS_FEAT_OVERRIDE`（`0x00409664`）和 `FECS_FEAT_READOUT_1`（`0x00409668`）镜像 PRI 功能覆盖状态，在非特权上下文读 `0xbadf5040`。

**熔断筛选（Floorsweeping）**
:   制造时通过熔丝永久禁用缺陷或过剩单元（GPC、TPC、FBPA、NVLink），以挽救部分缺陷芯片。筛选掩码是**逐芯片**的，不是逐 SKU 的：四张 170HX 卡读到的 `OPT_GPC_DISABLE` 值分别是 `0x85`、`0x45`、`0x13` 和 `0xa8`，而四张都仍枚举 70 个 SM。绝不要硬编码筛选值。

**FLR**
:   功能级复位（Function Level Reset），由 `echo 1 > /sys/bus/pci/devices/<BDF>/reset` 触发的 PCIe 按功能复位。170HX 在 DevCap 中通告 `FLReset+`，这正是解锁 harness 之所以可能。一次成功的 FLR **确实**清除 WPR2，也确实清除 SEC2 复位 PLM 污染（`0x8f` 回到 `0xff`），但不会复位 [AON 岛](#a)。

**FRTS**
:   FWSEC 在 GSP 引导前在 framebuffer 中建立固件驻留区域的命令，从 `kgspPrepareForBootstrap` 调用。该缩写在本语料中任何地方都没有被确立全称。

**FWSEC / FWSECLIC**
:   引导早期运行在一个 Falcon 上的 VBIOS 驻留固件安全微码，执行包括 FRTS 划分在内的任务。发布版补丁 0002 很大程度上是为了让 FWSEC 失败可诊断，把致命断言转成 `SEC2_DEBUG: FWSEC status=0x%x` 风格的日志行。FWSECLIC 是其许可证检查同伴。

---

## G

**GA100**
:   A100 和 CMP 170HX 共用的 Ampere 数据中心芯片：TSMC 7 nm N7、542 亿晶体管、826 mm²、BGA-2743 封装、CUDA 计算能力 8.0。`PMC_BOOT_0` 在每颗探测过的 GA100 上都读 `0x170000a1`。

**GPC / TPC / SM**
:   图形处理集群（Graphics Processing Cluster）、纹理处理集群（Texture Processing Cluster）、流式多处理器（Streaming Multiprocessor）。170HX 在两个 SKU 上都枚举 5 个活动 GPC、35 个活动 TPC 和 **70 个 SM**（4480 个 CUDA 核心），已经在其熔丝下限。完整 GA100 会是 8 个 GPC 和 64 个 TPC。

**GSP**
:   GPU 系统处理器（GPU System Processor），Ampere 及之后的 RISC-V 微控制器，在芯片上运行大部分资源管理器。

**GSP-RM**
:   在 GSP 上运行的资源管理器固件镜像。它在主机上的对应物是 Kernel-RM / CPU-RM。本 wiki 中的引导失败几乎总是 GSP-RM 引导失败。

---

## H

**HBM2 / HBM2e**
:   高带宽内存（High Bandwidth Memory），GA100 使用的堆叠 DRAM。170HX 上的理论峰值是 1555.2 GB/s（1215 MHz DDR、5120 位）。实测数值依工具和访问模式分布在 1305.86 至 1600 GB/s；不存在单一规范数值。

**HS 模式**（Heavy Secure，重安全）
:   Falcon 的最高特权模式。代码只在签名验证后进入 HS；一旦进入 HS，IMEM `0x00` 的低安全引导程序被擦除，DMEM 对主机不可访问，Falcon 能写其他情况下 PL0 阻挡的寄存器。整个显存解锁之所以存在，正是因为一组特定的寄存器写入只有从 HS 可达。

**HULK**
:   NVIDIA 用于启用调试和厂商功能的内部许可证/证书机制。170HX 在其 `0xFE000`-`0xFEFFF` 的许可证区域携带一个预构建但为空的 HULK 目录。已被调查并作为路线关闭。

---

## I

**InfoROM**
:   VBIOS 镜像中按板卡划分的持久数据区域，存序列号和校准数据。在 DRIVE A100 上，它占了带同一固件的两块物理不同 GPU 之间字节差异的 99.5 %。

**IOMMU**
:   主机输入/输出内存管理单元。安装解锁器后 PCIe 仍停在第 1 代时，首先要检查的就是直通模式（`iommu=pt`）；安装器会自动设置 `intel_iommu=on iommu=pt` 或 AMD 对应项。

---

## L

**LMR**
:   `0x00100ce0` 处的 `NV_PFB_PRI_MMU_LOCAL_MEMORY_RANGE`：MMU 对本地内存有多少的视图。编码为 `size_MiB = MAG[9:4] << SCALE[3:0]`。原厂值是 `0x00000208`（8 GB 卡）和 `0x00000288`（10 GB 卡）；解锁值是 `0x0000020B`（64 GB）和 `0x0000028A`（40 GB）。它的 PLM 是 `0x001fa7c4`（`..._LOCAL_MEMORY_RANGE__PRIV_LEVEL_MASK`），并在 `0x001180f0` 有一个 AON 影子。**不要**把 LMR 展开为 "LM Request"。

**LnkCap / LnkCap2 / LnkCtl2 / LnkSta**
:   链路能力、支持速率、目标速率和训练状态的 PCIe Express 能力寄存器。原厂 170HX：`LnkCap 0x00456101`、`LnkCap2 0x00000002`、`LnkSta 0x1041`。带解锁器：`LnkCap 0x00456102`、`LnkCap2 0x00000006`、`LnkCtl2 0x0002`、`LnkSta 0x1042`。通告的能力不是已训练的链路。

**LTC**
:   二级缓存切片。170HX 有 32 MB L2，对比 A100 的 40 MB。

**LTSSM**
:   链路训练与状态状态机（Link Training and Status State Machine），协商速率和宽度的 PCIe 状态机。在这张卡上，Gen2 补丁中昵称为 LTSSM 的寄存器是 BAR0 `0x0008872c`，写入 `0x00000006`。该块其他地方涉及的字段包括 LTSSM_DIRECTIVE（0 = NORMAL，1 = CHANGE_SPEED）和 [19:18] 处的 SPEED 字段。

---

## M

**MIG**
:   多实例 GPU（Multi-Instance GPU），Ampere 的硬件分区功能。可以通过设置 `0x820840` 的 bit 0 在已解锁的 170HX 上启用，之后 `nvidia-smi` 报告 `MIG M. Enabled`，可见 65536 MiB。

    > [!WARNING]
    > **实验性**
        MIG 启用是社区写入，**不在**发布版解锁器中。

**MOK / Secure Boot**
:   机器所有者密钥（Machine Owner Key）注册，让已签名树外模块在 UEFI Secure Boot 下加载的机制。打补丁的模块未签名，所以 `mokutil --sb-state` 报告 `SecureBoot enabled` 时 `install.sh` 硬失败。

---

## N

**NVGI / PciAt / FwSec body**
:   GA100 VBIOS 镜像的三个主要区域。NVGI 最早，在任何固件运行前由 PBUS/XVE 的从 ROM 初始化序列执行；PciAt 持有 PCI 可见的身份；FwSec body 持有签名的固件。8 GB 与 10 GB VBIOS 镜像之间的全部功能差异归结为 NVGI 引导程序中的 2 个字节。

**NVLink**
:   170HX 上熔断关闭（`FUSE_NVLINK_DIS`）。任何固件或驱动改动都无法恢复。板侧 NVLink 接口 IC 是否焊接未解决。见 [NVLink](../frontier/nvlink.md)。

**nvidia-open**
:   NVIDIA 的开源 GPU 内核模块。`cmpunlocker` 打补丁的是这个代码树，不是专有的，且恰好接受版本 `610.43.03`（默认）和 `610.43.02`。

---

## O

**OTP**
:   一次性可编程（One-Time Programmable）。承载计算限速（`OPT_SM_SPEED_SELECT`，九个独立熔丝）、设备 ID、PCIe 代际禁用和筛选掩码的熔丝。暴露它们的寄存器是只读熔丝影子。主 kill 熔丝 `0x008203f0` 读 `0x00000000`（未熔断），这就是这一切之所以可能。

---

## P

**P2P**
:   对等直连（peer-to-peer），GPU 到 GPU 传输。这张卡上没有。

**PLM**（Privilege Level Mask，特权级掩码）
:   一个按寄存器划分的访问控制掩码，决定哪些特权级（PL0 主机，到 PL3 重安全）可以读写它所守护的寄存器。打开 PLM 就是整个游戏：发布版驱动内路径按顺序恰好打开四个，每个最多尝试两次：

    | 索引 | 名称 | 地址 | 目标值 |
    |---|---|---|---|
    | 0 | `WPR_CFG` | `0x001fa7cc` | `0xfffff0ff` |
    | 1 | `FBPA` | `0x009a0148` | `0xffffffff` |
    | 2 | `WPR` | `0x001fa7c4` | `0xffffffff` |
    | 3 | `FEAT` | `0x00823804` | `0xffffffff` |

    `WPR_CFG` 回读 `0xfffff0ff` 是**正确的**，不是失败。说"所有 PLM 必须显示 `0xffffffff`"的指南过度严格。

**PMA**
:   物理内存分配器（Physical Memory Allocator），拥有 framebuffer 页面的 RM 对象（`pmaRegisterRegion`、`pmaGetFreeMemory`、`PMA_REGION_DESCRIPTOR`）。发布版补丁 0003 执行一次"迟 PMA 扩展"，把高 PMA 区域扩展到覆盖新暴露的 framebuffer，记录 `SEC2_DEBUG: late PMA extension status=0x%x`。它与电源管理毫无关系。

**PMC_BOOT_0**
:   BAR0 `0x00000000`，芯片身份寄存器。在每颗 GA100 上读 `0x170000a1`。GA10x 控制部件读 `0xb74000a1`。

**PRAMIN**
:   特权 BAR0 窗口，让 CPU 直接访问显存中一个可移动区域。发布版补丁 0004 在 `fbAddrSpaceSizeMb > 0x2000` 时把 PRAMIN 基址夹回一个原厂 8 GB 推导的偏移（`(0x2000ULL << 20) - DRF_SIZE(NV_PRAMIN)`），否则窗口会从 65536 MB 计算、落在不可达的 BAR0 空间之外。PRAMIN 也是证明 10 GB 卡上存在 80 个不同的 GiB 物理 DRAM 的工具。

**PRI**
:   GPU 的内部特权寄存器总线。被阻挡或不对应任何目标的读取返回 `0xbadfXXXX` 毒值而非数据：`0xbadf5040` = 被特权级掩码阻挡，`0xbadf1100` = 目标不存在，`0xbadf20NN` = 目标存在但该 FBPA 被熔断筛选，`0xbadf5108` = 从 PL0 读 AON 安全暂存。

**`probe.sh`**
:   只读表征工具（`tools/mmio-probe`）。它只读 mmap `resource0`，转储约 120 至 130 个具名寄存器加 24 次每 FBPA 读取，**从不写 BAR0**。常量：`FBPA_BASE = 0x900000`、`FBPA_STRIDE = 0x4000`、`CSTATUS_RAM = 0x20C`。它输出 `registers.json`、`lspci.txt`、`nvidia-smi.txt`、`gpu-summary.csv` 和 `probe.log`。它是标准验证工具：任何写入后，用 `probe.sh` 回读寄存器，而不是相信工具声称的成功。

**PTE kind**
:   GPU 页表项的 "kind" 字段，描述压缩和铺排格式。发布版补丁 0005 针对这些设备 ID 强制 `*pteKind = NV_MMU_PTE_KIND_GENERIC_MEMORY`，取代 `..._COMPRESSIBLE_DISABLE_PLC`。

---

## R

**RFRD**
:   VBIOS SPI ROM 中的镜像布局描述符记录，位于绝对 `0x2000`。一个社区解析器把它误标为"功耗表"，其实不是。

**ROP 链**
:   返回导向编程链（Return-Oriented Programming chain）。完全由已签名代码中现成的短指令序列（"gadget"）通过覆写返回地址串成的载荷，因此无需签名任何新代码。在这张卡上，链放在超大的 GSP 签名缓冲区中，由 SEC2 Booter 执行。Booter 从 DMEM `0xFF5C` 取被劫持的返回地址；`0xFF48` 是 `0x4d4` 弹出块中的已存 r3 槽，也是 `0x18` 字节帧网格的基址，所以在已被取代的独立链中，N 次写尾部从 `0xFF48 + N*0x18` 开始。见 [ROP 链](../unlock/rop-chain.md)。

---

## S

**SBR**
:   次级总线复位（Secondary Bus Reset），比 FLR 更强的复位，由上游桥发出。SBR 断开并重新初始化常开电源域，因此它能清除植根于扛过 FLR 的 AON 暂存中的卡死。

**SCP**
:   Falcon 的安全协处理器（Secure Co-Processor），用于签名验证和密钥处理的密码块。Falcon 引擎复位恢复期间轮询 `SCP_CTL_P2PRX` 的 bit 3（SFK_LOADED）。

**SEC2**
:   GPU 的安全引擎 Falcon，位于 BAR0 基址 `0x00840000`（邮箱 0 在 `0x00840040`）。它运行 Booter 微码，也是解锁所利用的引擎。它的复位 PLM 可观察量（地址报告为 `0x008403C4`，身份有争议）干净时读 `0xff`，`secure_teardown` 运行后读 `0x8f`，驱动仍在加载的部分点火状态读 `0x00cf`。

**签名缓冲区**
:   持有 GSP 固件签名的内存描述符。原厂大小为 4096 字节；发布版补丁把它扩大到 `0x0000f800`（63,488 字节）并用载荷填充，dword `0x000004a7`。更早被放弃的方法在磁盘上补丁 `gsp_tu10x.bin`，被 `fwsignature_ga100` 段只有 `0x1000` 字节挡住。

**SS0 / SS1**
:   `FEATURE_OVERRIDE_SM_SPEED_SELECT`（`0x0082381c`）和 `..._SM_SPEED_SELECT_1`（`0x00823820`）。它们控制**每条指令单元的发射速率**，不是哪些 SM 活动。解锁写入 `0x88888888` 和 `0x00000008`。被锁的卡例如在 SS0 读 `0x53540175`。它们是 AON 的，能扛过 FLR。它们不是 "Suspension State" 寄存器。

**跳线 / 跳线电阻（Strap / strap resistor）**
:   一颗 0402 电阻加一个空的相邻焊盘，把元件在两位置间移动会翻转一个硬件采样配置位。170HX 带五对跳线（十个焊盘，位号 R986 至 R1005），另有一对 DEVID_SEL 在别处。主 PCIe 设备 ID 熔进芯片，**不可**由跳线设置（`FUSE_DEVID_SW_OVR_DIS 0x00820584` 在每张探测过的卡上都为 1）。

---

## V

**VBIOS**
:   卡的固件 ROM。公开存在四份 170HX 镜像；TechPowerUp 对其中两份的 "16 GB" 和 "0 GB" 尺寸标签是错的，两者都不会解锁显存。VBIOS 版本对解锁是否生效没有影响。见 [VBIOS](../hardware/vbios.md)。

**VSEC**
:   厂商特定扩展能力（Vendor-Specific Extended Capability），PCIe 配置空间扩展能力块。对 Gen2 有两个寄存器重要：`VSEC_DEVICE`（`0x0008860c`，bit 0 经 Booter 载荷置位）和 `VSEC_HIERARCHY`（`0x00088610`，Booter 阶段之后是普通主机 BAR0 写入）。

---

## W

**WPR / WPR1 / WPR2**
:   写保护区域（Write Protected Region）。MMU 拒绝让非特权代理写入、用于持有 ACR 和 GSP 固件状态的 framebuffer 范围。WPR2 lo/hi 在 `0x001fa824` 和 `0x001fa828`。禁用时读 `0x1FFFFE00 / 0x00000000`；Booter 运行后读 `0x01F77000 / 0x01FFEE00`。发布版补丁保存两者一次，并在**每次** Booter Load 尝试前重写它们，而不是清除它们。"WPR2 already up" 是早期主导失败，现已降级为一条继续运行的警告。

**WprMeta**
:   描述 WPR 布局的元数据结构，包括 `fbSize` 和 `sizeOfSignature`，由驱动填充、Booter 验证。

---

## X

**Xid**
:   NVIDIA 驱动发出的错误标识符。这里要紧的：

    | Xid | 在本语料中的含义 |
    |---|---|
    | 31 | MMU 故障，`FAULT_INFO_TYPE_REGION_VIOLATION`。分配越过解锁窗口的可用顶部。卡在重启前 CUDA 中不可用。在 80 GB 时，触碰约 40 GB 以上的内核导致与功耗上限无关的致命 GPU 丢失；报告的 Xid 码包括 Xid 31（被描述为无害）和 CUDA 内存测试后的 Xid 154，主导报告症状是卡死。Xid 31 是旁观者建议的，并未被故障卡的操作者证实为*那个*特征。 |
    | 45 | 由 SIGKILL 运行中的 CUDA 验证内核诱发；强制一次复位循环。 |
    | 119 | GSP RPC 超时。两个不同变体：等函数 4097 `GSP_INIT_DONE` 60 秒（引导从未完成），以及函数 103 `GSP_RM_ALLOC` 6 秒（启动后卡死，每次 `nvidia-smi` 重复）。 |
    | 154 | 超配 80 GB 配置下 CUDA 内存测试后的主导失败；把卡限制为每次点火一个 CUDA 上下文。 |

**XP3G**
:   `0x0008e1xx` 处的 PCIe 链路层覆盖块，包括 `XP3G_OVR0` `0x0008e110`、`XP3G_VAL0` `0x0008e120`、`XP3G_OVR3` `0x0008e11c`、`XP3G_VAL3` `0x0008e12c`，以及 PLM 四件套 `0x0008e1b0` / `0x0008e1b4` / `0x0008e1b8` / `0x0008e1bc`。Gen2 补丁经 Booter 载荷原语推送一个 23 项 `xp3gTable`（18 次 PLM 打开加 5 次值写入）。NVIDIA 未发布该名称的全称。

**XVE**
:   NVIDIA 对 PCI Express 端点和配置空间块的内部名称，基址 `0x00088xxx`。Gen2 家族分支向表里加了三个 XVE 能力 PLM：`0x00088ff4`（XVE）、`0x00088ab4`（XVE_B）、`0x00088ff8`（XVE_C）。这些字母在任何公开 NVIDIA 文档中都没有展开。

---

## 数字、代码与文件路径

**`0x008200FC`**
:   一个寄存器，两个名字。分支源码写 `{0x008200fcU, 0xffffffffU, "OPT_PLM"}`，所以 `OPT_PLM` 是代码名；`FUSE_SS_PLM` 是净室工具对同一寄存器的名字。发布版 master **不**写它。它是否可写、冷卡上读什么，是开放的。

**`0xbadfXXXX`**
:   见 [PRI](#p)。这些绝不是存储的数据。

**`0xc0deca7e`**
:   放在构造的签名缓冲区中的假金丝雀哨兵。

**分支名**
:   有 **12** 个未发布分支快照（`80`、`Gen2`、`PG199`、`clanker_driver-port`、`debug-gen2`、`deced`、`docs`、`ecc`、`far`、`housekeeping`、`memory`、`multiple-cards`），加发布版 `master` 共 13 棵代码树。说"十三个未发布分支"的文档差一个。

**`/lib/modules/$(uname -r)/updates/cmpunlocker/`**
:   安装写入打补丁模块加三个标记文件的地方：`driver_version`、`card_profile`（`8gb` 或 `10gb`）和 `unlock_geometry`。多卡分支加 `gpu_inventory`。

**`SEC2_DEBUG`**
:   解锁路径的日志标签。`sudo dmesg | grep SEC2_DEBUG` 是唯一的主要诊断。存在两个兄弟标签：`SEC2_DEBUG_HEAP` 和 `SEC2_DEBUG_LATE_PMA`。全部在 `LEVEL_ERROR` 下发出，所以不需要额外调试标志。完全没有 SEC2_DEBUG 行意味着打补丁的模块从未运行。

---

## 本 wiki 引用的工具

| 工具 | 在这里的用途 |
|---|---|
| `clpeak` | OpenCL 带宽和计算微基准；Gen1 x4 约 0.85 GB/s 数值的来源 |
| `cuda_memtest` | GPU 显存验证；80 GB 配置重启后通过一次然后失败 |
| `gpu-burn` | 带错误计数器的持续计算压力；稳定的 40 GB 卡干净通过 5 分钟 |
| `mixbench` | 混合精度吞吐；其 `1769.47 GB/sec` 数值是理论的，不是实测 |
| `nvtop` | 实时按 GPU 遥测，包括 PCIe 代际和宽度 |
| `ocl_pcie_bw` | OpenCL 主机到设备带宽；6.63 至 6.67 GB/s 的 Gen2 x16 数值来源 |
| `pcielink.sh` | 社区链路训练报告数据收集脚本；打印 GPU 和桥的身份加完整 LnkCap/LnkSta/AER 集 |
| `probe.sh` | 只读寄存器普查；见上方 [PRI](#p) |
| `verify.sh` | 多卡分支上的按 BDF 解锁验证 |
| `CH341A` | SPI 闪存编程器。GPU EEPROM 是 1.8 V，所以需要 1.8 V 适配器 |

---

## 另见

- [如何阅读本 wiki](how-to-read-this-wiki.md)：置信度约定。
- [寄存器参考](../unlock/register-reference.md)：所有地址一张表。
- [识别你的卡](identify-your-card.md)：搞清楚你拿的是哪个 SKU。
