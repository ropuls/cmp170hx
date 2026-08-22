# PCIe Gen3 与 Gen4：为什么它们仍然被锁

**本页涵盖：** 关于 CMP 170HX 上 5 GT/s 以上的 PCIe 代际墙的所有已知信息：两层相互独立的锁定（一对 OTP 熔丝，加上已签名 DevInit 镜像中的五字节修改）、为什么 2026-07-24 的 Gen2 突破没有把 Gen3 一起带下来、可用的 Gen2 补丁所读取但从不写入的确切寄存器、完整的被证伪方法目录，以及按成本排序的剩余途径。

**头条结论：CMP 170HX 上从未有 Gen3 或 Gen4 链路训练成功过。** 截至 2026-07-28，语料中没有任何来源报告这张卡出现 `LnkSta: Speed 8GT/s` 或 `16GT/s`。Gen3 的*通告*（advertisement）在 2026-07-24 被做成功了（`LnkCap: Port #1, Speed 8GT/s, Width x4` 和 `LnkCtl2: Target Link Speed: 8GT/s`），而 `LnkSta` 仍钉在 `Speed 2.5GT/s, Width x4`。记录中的收尾立场由维护者于 2026-07-27 陈述：Gen3 需要 GSP-RM 固件补丁——"Gen 3 完全不行，它需要一个 GSP 补丁"，以及"我没见过任何人做出过可用的 GSP 补丁"。

> [!NOTE]
> **开放问题**
>
> 本页描述的是未解决的工作。这里没有任何内容随正式版发布。Gen2（5 GT/s）*确实*已在软件中解决，并于 2026-07-29 随 `master` 发布：参见 [PCIe Gen2](../unlock/pcie-gen2.md)。

> [!WARNING]
> **速率不是宽度**
>
> PCIe 链路**速率**（Gen1 到 Gen2）和 PCIe 链路**宽度**（x4 到 x16）在这张卡上是两个完全不同的问题，有两条完全不同的修复途径。速率是固件和熔丝层面的，即本页主题。宽度是 PCB 元件拆空，只能靠手工焊接 24 颗交流耦合电容修复，详见[物理改造](../operations/physical-mods.md)。在原生 x4 宽度上做 Gen3 软件解锁正是社区声明的下一个目标，恰恰因为它不需要焊接。

---

## 状态一览

| 代际 | 速率 | 170HX 上的状态 | 机制 |
|---|---|---|---|
| Gen1 | 2.5 GT/s | 出厂状态，冷启动总是能训练成功 | 已签名 DevInit 编程 CMP PCIe 表 |
| Gen2 | 5.0 GT/s | **软件已解决**，自 2026-07-29 起随 `master` 发布 | 通过 SEC2 Booter 的组合寄存器序列，加上一次根端口重训练 |
| Gen3 | 8.0 GT/s | **未解决。** 能力可以被通告，链路从不训练 | 维护者于 2026-07-27 称需要 GSP-RM 补丁，且至今没有可用的 GSP 补丁产出 |
| Gen4 | 16.0 GT/s | **未解决且无法测试。** 没有任何贡献者拥有 Gen4 主机 | 熔丝位 25 `GEN4_SPEED_DISABLED` 加上被抑制的 DevInit 块 |

链路状态指纹供参考：

| 状态 | LnkCap | LnkCap2 | LnkCtl2 | LnkSta |
|---|---|---|---|---|
| 出厂（锁定） | `0x00456101` | `0x00000002` | `0x0000` | `0x1041` |
| 已安装解锁器并训练成功 | `0x00456102` | `0x00000006` | `0x0002` | `0x1042` |
| 第三轮向量伪造 | 写入 `0x00457104`（宽度 x16） | 写入 `0x0180001E`；被截回 `0x00456102` / `0x00000006` | 未记录 | 未记录 |
| Gen3 通告，2026-07-24 | `Port #1, Speed 8GT/s, Width x4` | 未记录 | 目标 8GT/s 被接受 | 保持 2.5GT/s x4 |

最后两行是**两个不同的实验**，从未同时观察到，且它们的 `LnkCap` 值编码了不同的宽度（x16 与 x4）。不要把两者读成同一次运行。

---

## 为什么 Gen2 掉了而 Gen3 没有

在 2026-07-24 之前，主流模型把支持速率向量视为按规范连续的（Gen4 要求 Gen2 和 Gen3），因此 Gen2、Gen3、Gen4 被当作同一个问题："要么向量打开，要么什么都不打开"。然后向量打开了，而且只打开到 Gen2。

Gen2 成果的工作方式是：通过 SEC2 Booter 载荷写入原语打开一组特权级掩码，清除 `CYA_0` 中的 `DIS_G2` 鸡位（chicken bit），把 `LINK_CONFIG_0` 的 MAX_RATE 强制为 2，驱动 XP3G 覆盖槽位和 `PRIV_MISC_1`，然后让**上游根端口**重训练链路。端点的 `LnkCap`/`LnkCap2` 是 PHY 的反射：一旦 Gen2 门打开，它们会重新生成 `0x00456102` / `0x00000006`，无需任何东西直接写入。它们重新生成为 Gen2 **且不再进一步**，即使伪造写入显式地写入 Gen1-4 的值：

```text
round-3 spoof: 0x88084 <- 0x00457104   (A100 Max Link Speed = 4)
               0x880A4 <- 0x0180001E   (A100 supported vector, Gen1-4)
observed post: CAP=0x00456102 CAP2=0x00000006
```

硬件把写入截断到了 Gen2。一位两天后验证出 Gen2 那一半的研究者总结道："Lnkctrl2 被一个直接硬件掩码封顶在 gen2（这就是为什么 gen 3/4 这么痛苦）。但用正确的寄存器写入你可以达到 gen2。"

有两种可能性仍然存在，语料中没有任何东西能把它们分开：

1. 关于这颗硅片的连续性论证从一开始就是错的，或者
2. Gen3 熔丝是独立强制的，位于支持速率向量之后的下游。

硅片上最强的证据支持（2）。在 XP3G 特权级掩码被打开（`PLM[4] XP3G_PLM(0x8e1b0) reg=0xffffffff`）后，PHY 速率寄存器被强制写入一个 Gen3 能力范围内的值并正确读回，而链路仍然训练在 Gen1：

```text
XP3G rate=0x00340036 ovr0=0x4
lnksta=0x10410040 speed=1        # lspci: Speed 2.5GT/s
```

第二条较弱的证据线：在 Gen2 训练成功的卡上，`LnkSta2` 报告 `EqualizationComplete-` 和 `EqualizationPhase1-`，也就是说，Gen3 强制的 PHY 均衡过程从未运行过。概括社区观点的表述是*"Gen2 是软件锁定，Gen3 是硬件熔丝"*（置信度：中；从未有人报告过带实测 PHY 行为的 Gen3 强制尝试）。

---

## 锁定层 1：OTP 熔丝

三个熔丝选项寄存器构成了代际锁的指纹。全部在两个物理 170HX SKU 上读取，并与一个 15 卡 Ampere 对照组比较。

| 寄存器 | 地址 | 170HX | 对照组 | 备注 |
|---|---|---|---|---|
| `FUSE_PCIE_GEN23_DIS`（`OPT_PCIE_BOOT_GEN23_DISABLE`） | `0x0082057c` | `0x00000001` | 所有三个 A100 SKU、A10、A5000、A6000、RTX 3080/3080 Ti/3090/3090 Ti、一块 ES 件、一块 Drive A100 和一块 GA10x 对照卡上均为 `0x00000000` | Gen2 补丁唯一尝试写入的一个 |
| `FUSE_PCIE_GEN3_DIS`（`OPT_PCIE_BOOT_GEN3_DISABLE`） | `0x00820580` | `0x00000001` | 其他所有地方均为 `0x00000000` | **从未被任何人写入** |
| `FUSE_PCIE_MAGIC_D` | `0x00820520` | `0x16680000`（位 25 置位） | A100 SXM4 40G / PCIe 40G / PCIe 80G / Drive A100 上为 `0x00200000`；A10/A5000/A6000 上为 `0x01a00000`；RTX 30 系上为 `0x10a80000` | 位 25 被文档记载为 `GEN4_SPEED_DISABLED`，引用 NVIDIA bug 2220334 |

因此该锁的三寄存器指纹是 **`1` / `1` / `0x16680000`**。

还有两个熔丝事实对任何规划攻击的人都很重要：

- **lane 是干净的。** `OPT_PCIE_LANE_DISABLE` `0x00820394`、`CTRL_OPT_PCIE_LANE` `0x0082082C` 和 `STATUS_OPT_PCIE_LANE` `0x00820C2C` 全部读出 `0x00000000`。只有速率被熔断。这独立证实 x4 宽度是板卡问题。
- **软件覆盖路径被熔断关闭。** `FUSE_EN_SW_OVERRIDE` `0x00820040` = `0`，`FUSE_DIS_SW_OVR` `0x00820084` = `1`。`0x00820148` 是会让 DevInit 写入 A100 `MAGIC_D` 值的 DevInit 门控位，它是一个 OTP 备用位，读出来是 `0`，永远无法从软件设置。这就是为什么整个项目中最干净的 A/B 实验（2026-07-22）看到 XVE 目标在同一启动中落地并保持，而 `0x820520` 保持 `0x16680000`、`0x820148` 保持 `0`。

### `OPT_GEN3` 与 `OPT_MAGIC`：只读、只记录、从不写入

这是本页在代码层面最重要的事实。可用的 Gen2 补丁 `0007-pcie-gen2.patch` `#define` 了全部三个熔丝选项寄存器并打印全部三个，但其 23 项经 Booter 路由的写入表中只包含其中一个：

```c
/* from 0007-pcie-gen2.patch */
#define PCIE_GEN2_OPT_GEN23_ADDR   0x0082057cU   /* write attempted -> fails on silicon */
#define PCIE_GEN2_OPT_GEN3_ADDR    0x00820580U   /* read only */
#define PCIE_GEN2_OPT_MAGIC_ADDR   0x00820520U   /* read only */
```

这三个值一起出现在 `NV_PRINTF` 参数列表中，格式为 `OPT=%08x/%08x/%08x`（GEN23 / GEN3 / MAGIC）。在 Gen2 分支的启动中，该打印读取：

```text
OPT=00000001/00000001/16680000
```

这一行本身就是一个有用的 dmesg 指纹：Gen1 构建发出 34 行 `SEC2_DEBUG`，Gen2 构建发出 80 行。

> [!NOTE]
> **行数不是可靠的跨构建指纹**
>
> 34（Gen1 构建）/ 80（Gen2 构建）是高置信度记录，而另一次 Gen2 分支 610.43.03 启动计数为 152，置信度中。不要把不一致读成安装失败。

任何分支中都没有代码路径——独立的净室工具集中也没有——请求过高于 2 的目标链路速率：Gen2 代码中的 `constants.yaml` 把 `target_gen: 2` 固定住，`TARGET_LINK_SPEED` 被写成 `2`，LTSSM 速率字段被设为 `2`，成功测试是 `LnkCap2 & 0x4`。

### `OPT_GEN23` 写入失败，而 Gen2 仍然工作

唯一*被尝试*的熔丝写入并没有落地。来自一次插桩构建的逐字记录，同一启动中的两张 GPU 都是如此：

```text
NVRM: GPU0 _kgspBootGspRm: SEC2_DEBUG: PCIe xp3g booter OPT_GEN23(0x82057c)=0x00000000 \
  attempt=1 status=0xffff rd=0x00000001 OVR0=0x00000000 VAL0=0x00000000 \
  OVR3=0x00000000 VAL3=0x00000000
NVRM: GPU0 _kgspBootGspRm: SEC2_DEBUG: PCIe xp3g booter FAILED to set OPT_GEN23
```

一次直接的高安全写入记录了 `PLM[4] OPT_GEN23(0x82057c) status=0xffff reg=0x1 (write FAILED)`。该寄存器是一个纯 OTP 熔丝感应反射，在任何特权级都没有写端口。**因此 Gen2 是在 `OPT_GEN23` 从未被清除的情况下工作的。** 熔丝影子不是杠杆；`CYA_0`、`LINK_CONFIG_0`、XP3G 覆盖和 `PRIV_MISC_1` 才是。

这对规划很重要，因为 Gen3 最常被引用的"廉价下一步"是通过同一张"在 `0x0082057c` 上已经成功"的表写入 `0x00820580 = 0`。这个前提是错的：该表对 `0x0082057c` 的写入被观察到在硅片上失败。这个实验仍然值得跑（只花一次启动），但其先验概率应该很低。

---

## 锁定层 2：DevInit 配置表

PCIe 速率限制还存在于 SPI 闪存中**未加密** DevInit falcon 镜像内的一个 PCIe 配置表中，而不在传统的 x86 VBIOS 部分。

| 项目 | CMP 170HX | A100 |
|---|---|---|
| 表位置，闪存 | `0x420ED`（镜像 `0xA20ED`） | `0x408A0`（镜像 `0xA08A0`） |
| 运行时 DMEM 基址 | `0xF1D` | `0xE50` |
| 表 `+0xC7..+0xCB` 处的五个字节 | 闪存 `0x421B4-0x421B8` 处为 `00 00 08 00 06` | 闪存 `0x40967-0x4096B` 处为 `00 00 14 00 06` |
| 表 `+0x0F` 处的抑制标志 | `0x01` | `0x00` |
| DevInit 镜像 | 闪存 `0xDE00`（反汇编基址 `0x8000`），在 bank 2 的 `+0x60000` 处重复 | 布局相同 |
| BIT 表（I、i、C、D、x、p、u、B、M） | 与 A100 逐字节相同 | 与 CMP 逐字节相同 |

抑制标志是关键字节。在 `0x31B3F-0x31B92` 的反汇编中，代码读取它并跳离整个 Gen4 编程块：

```text
ld b8 r9, D[tab+0x0F]
bra ne -> skip whole block
```

当该块运行时，它计算两个只写寄存器：

```text
0x88CE4 = (old & ((b1<<8) | b0)) | ((b3<<8) | b2)   ; 因为 b0=b1=b3=0 而化简为字节 [+0xC9]
0x88CE0 = (old & ~0x3F) | (b4 & 0x3F)               ; 两部件上 b4 = 0x06
```

`0x88CE0` 和 `0x88CE4` 在整个 DevInit 反汇编中都是只写的（一次性 Gen4 初始化配置），并且位于物理层 16.0 GT/s 扩展能力（影子 `0x88C1C` 处的 PCIe 能力 ID `0x0026`）的 XVE 影子内。周围的 Gen4 序列还写入 LTSSM 超时 `0x8D1A0 = 0x1B1F2327` 和 `0x8D1A4 = 0x0B0F1317`（与 A100 活跃值相同）以及 `0x88610 = 0x1001`。

一个在 CMP DevInit 反汇编上运行的符号化迷你解释器后来确定，**总共**有十三个 DevInit 字节不同，而不是五个；其中十一个差异字节被归因于非 PCIe 的 SKU 特性（HBM、NVLink、ECC）。PCIe 相关的消费方是 `[0xC9]` 到 `0x88CE4` 和 `0x132B70`，`[0x1C-0x1F]` 到 `0x8C2C0` 加上 `0x918050`/`0x91C050`/`0x920050` 系列，以及 `[0x3F]`（A100 上为 `0x00`，CMP 上为 `0x0C`）到 `0x8C040`。

> [!CAUTION]
> **重新刷写不是一条路**
>
> 全部五个 PCIe 字节 **100 %** 落在 Davies-Meyer `csecret(2)` MAC 范围 `0x2200-0x43C00` 内。无密钥伪造是一个 2^128 的第二原像问题。Ampere RSA 签名校验会拒绝被编辑的镜像，卡将无法启动。`0x40B4B`、`0x40F05-3D` 和 `0x40FC5-CB` 处的 Gen 能力字节也在 MAC 范围内。不要尝试刷写被修改的 DevInit：参见 [VBIOS](../hardware/vbios.md) 和[恢复](../procedures/recovery.md)。

### 被证伪的直觉：strap 字段是单调限制性的

语料中最有价值的修正之一。社区 `pcie_set_speed` 补丁的直觉方向正好是反的。已签名 FWSEC 中的 devinit 读-改-写是 `mov r9 0x14118f78; ld; and 0x3ff / or 0x400; st`，位于 VBIOS 偏移 `0xE88C`，每个 ROM 中都有 26 处引用；170HX 与 A100 的差异是**被写入的值**，而不是代码。strap 字段是限制性的：`0` = 所有代际启用，`3` = 170HX 的设置（清除 Gen2/3/4），`0xF` = 越界 / 全部禁用。提高上限需要**更低**的 strap 值，而且不存在写端口。

一个相关的地址空间说法于 2026-07-27 被撤回：FWSEC falcon 代码中每个 `0x14xx....` 常量都是与 aperture 基址 `0x14000000` 相或的 BAR0 偏移，因此 `0x14118F78` 是 BAR0 偏移 `0x118F78`，位于普通的 16 MB 窗口内，而不是在单独的">16 MB Falcon PRIV 总线"上。**在驱动加载的情况下**，主机读取 `0x118F78` 返回 `0xbadf1100`，即 NVIDIA 的特权毒值（priv-poison）模式，因此 FWSEC 上下文之外的主机可达性仍未得到证明。

---

## 熔丝实际在哪里被消费

DevInit 根本不读取这两颗 Gen 熔丝。CMP DevInit 反汇编中 `0x82xxxx` 访问的完整列表是 `0x820C14`/`0x820D38`（FBIO/FBP floorsweep）、`0x820684`（`FUSE_NVLINK_DIS`）、`0x82380C`/`0x823814`、`0x820520`、`0x820148`、`0x8243xx`、`0x8202xx`、`0x8201xx`、`0x82033C`/`0x82030C`。`0x82057C` 和 `0x820580` 都没有出现。

GSP-RM 确实读取它们，而且这些读取位置已被定位：

| 固件 | 地址 | 指令证据 |
|---|---|---|
| `470.42.01 gsp.bin` | `0x5D55834` 处的熔丝读取跳转表 | `li a2, 0x580` 和 `li a2, 0x57c` |
| `580.105.08 gsp_tu10x.bin` | `0x4DD9B00`（`jalr fuse_read`） | `li a2, 0x57c` |

正是这对位置构成了当前需求陈述"它需要 GSP 补丁"的原因。对 `gsp_tu10x.bin` 的完整反汇编扫描还确定了 GSP **不**做什么：在 `0x88CE4`、`0x88CE0`、`0x88084`、`0x880A4`、`0x880A8`、`0x820520` 或 `0x82057C` 上没有任何写入被发现。GSP 只为了链路管理而触碰 PCIe（`0x88088` 对位 0-1 的读-改-写、带 Gen1/2/3 分支的速率读取，其中 Gen4 落入默认路径、`0x8A088`、内部读取 `0x88A48`/`0x88A4C`/`0x88A64`，以及作为 `0x82000 | offset` 的动态熔丝块访问）。

> [!NOTE]
> **值得保留的方法说明**
>
> 早期一次朴素的 4 字节常量搜索错误地报告 GSP 镜像中"没有 XVE 引用"，因为 RISC-V 通过 `lui`/`addi` 动态构建这些地址。需要完整的模式扫描。加密的 GSP 区域仍然不可读，因此即使修正后的扫描也不是穷尽的。

---

## 速率能力的寄存器级地图

来自 RM 反汇编，附带说明：可用的 Gen2 结果证明实际图景比这个地图暗示的更宽松（置信度：中）。

| 寄存器 | 作用 | 访问 | 170HX 上的观测 |
|---|---|---|---|
| `0x85080` | 支持速率来源 [23:20]，跳转表索引 | RO，RM 反汇编的 4.1M 行中零写入者 | 从注入点读出 `0xBADF1100`（毒值） |
| `0x85084` | 允许代际掩码 [3:0]，GSP-RM 每次重训练重新推导 | 从可达上下文为 RO | `0xBADF1100` |
| `0x88084` | `MAX_LINK_SPEED` [3:0] | PHY 反射，标记为 R-XVF | 出厂 `0x00456101` |
| `0x8808C` | `SUPPORTED_LINK_SPEED` [7:1] | PHY 反射，标记为 R-EVF（无写端口） | 对主机有 PROT 墙 |
| `0x880A8` | `TARGET_LINK_SPEED` | RW 但被 SUPPORTED 封顶 | 出厂 `0x00000001` |
| `0x8841C` | `PRIV_MISC_1` CYA Gen2/3 覆盖位 11-16、30、31 | PLM 下 RW | `0x20340500` 到 `0x20342d00` |
| `0x88610` | `VSEC_HIERARCHY`，位 12 门控 PRIV_MISC_1 重新编程 | PLM 下 RW | 活跃 `0x00001001` |
| `0x8872C` | LTSSM 触发（写入 `6`） | PLM 下 RW | 不是真正的重训练 |
| `0x8C1C0` | `PL_LINK_RATE`，代际字段 [19:16] | PLM 下 RW | 被 0007 写入 `0x00240036` |
| `0x881C0` | `PPCI.UNK1C0`，[17:16] `LNK_CAP_SPEED`、[21:20] `SYSTEM_MAX_SPEED` | 主机读取被阻止 | `0xbadf5040`；A100 对应件 `0x8C1C0` 读出 `0x00040036` |

速率向量编码：Gen1 = `0x1`，Gen1_2 = `0x3`，Gen1_2_3 = `0x7`，Gen1_2_3_4 = `0xF`。

在一张参考 A100 80GB 上进行的强制代际扫描确定了链路速率实际存在于哪里，也是可用的最干净的对照测量：

| 强制代际 | `0x88088`（[19:16] 处速率） | `0x880a8`（[3:0] 处目标） | `0x88084` |
|---|---|---|---|
| Gen1 | `0x11010140` | `0x001e0001` | `0x00456104` 或 `0x00457104` |
| Gen2 | `0x11020140` | `0x001f0002` | 不变，nibble 恒为 4 |
| Gen3 | `0x11030140` | `0x001f0003` | 不变 |
| 原生 | `0x11040140` | `0x001f0004` | 不变 |

---

## 已尝试并失败的方法

### 寄存器与配置空间攻击

| # | 方法 | 为什么看似可行 | 怎么死的 | 日期 |
|---|---|---|---|---|
| 1 | `setpci` 写入 LnkCap2（配置 `0x2C`）并设置所有速率 | 它就是列出支持速率的寄存器 | 被静默丢弃。硬件只读，在 NVIDIA 的 `dev_nv_xve3g_fn0` 头中标为 `R-EVF`：任何特权级都没有写端口，所以打开 PLM 也无济于事 | 2026-07-24 |
| 2 | 单独调高 `TARGET_LINK_SPEED`（`0x880A8`）并重训练 | TARGET 确实可写 | 链路重新训练在 Gen1；端点在 TS1/TS2 有序集中重新通告 Gen1，被只读的 SUPPORTED 字段限制 | 2026-07-24 |
| 3 | 主机 BAR0 写入 `0x88070` / `0x8808C` / `0x88090` | 紧邻能力块 | 对主机有 PROT 墙：读取返回 0，写入被忽略 | 2026-07-24 |
| 4 | 单独做高安全 XP3G PHY 速率覆盖 | PLM 已打开，覆盖寄存器可写，速率读回 Gen3 能力值 `0x00340036` | 链路保持 Gen1。它确实证明了一个正面事实：`0x10B9` SEC2 CSB 邮箱 gadget 能到达 XP3G/PCIe 特权块。后来成为可用 Gen2 组合的*一个组成部分* | 2026-07-24 |
| 5 | 高安全 `FEAT_OVR` 写入加重训练 | 计算解锁正是通过这条路径工作的 | `0x823800` 读回 `0xfffffe8e`（写入生效了），`OPT_GEN23` 保持 `0x1`，链路保持 Gen1，AER = 0。当时的结论：PCIe 覆盖使能在 FEAT_OVR 中被熔断**关闭**，不像 `SM_SPD` 那样熔断**打开**。注意 [FEAT_OVR 清单](nvlink.md#route-b-a-feat_ovr-style-attack) 在该块中未列出任何 PCIe 寄存器，所以把它当作探测结果而不是定位到的寄存器 | 2026-07-24 |
| 6 | 直接写入 `OPT_GEN23`（`0x82057C` <- 0） | 明显的杠杆 | 从主机、从 HS-ROP、通过 Booter 载荷都失败。已发布的 Gen2 补丁仍尝试它，仍然失败，Gen2 照常工作 | 2026-07-23 |
| 7 | 通过 Booter 设置 `VSEC_DEVICE` 位 0 | 已发布序列的一部分 | `pre=0x00000800 want=0x00000801`，两次失败，`rd=0x00000800`。这对"瞬态窗口"模型很尴尬，该模型把窗口关闭归咎于 RM 清除了补丁从未设置的位 | 2026-07-23 |
| 8 | 在 postbl 阶段写入推导出的允许代际掩码 `0x85084` | "GSP 写入 `0x85084`"是真的 | 从注入点读 `0x85080` 和 `0x85084` 都得到 `0xBADF1100`，写入被丢弃。GSP 在注入点永远达不到的特权级写入它，而且反正每次重训练都会重新推导 | 2026-07-24 |
| 9 | VFIO/QEMU 下 BAR0 `0x8872c` 值扫描 | 紧邻 LTSSM | `0x6` 稳定并让 LTSSM 保持在 Gen1 x4；`0x2` 和 `0xA` 暴露额外的 Gen2 行为但最终卡死 VFIO/QEMU 功能。已发布的 0007 恰好写入 `0x6`，其自己的日志说"skip mid-boot retrain" | 2026-07-12 |
| 10 | 把 `0x88084` `MAX_LINK_SPEED` 当作可写上限 | 一项分析认为不存在主机可写的后备寄存器 | 对暂存寄存器的高安全写入成功，而对整个 XP-PL `LINK_CONFIG` 簇（`0x8C044` / `0x8C048` / `0x8C04C`）的同样写入被拒绝。转述者认为该分析可能错了，但被核查的部分站得住：该簇确实不同于可用的补丁所用的 `0x8C040`/`0x8C2C0`/`0x8C1C0` | 2026-07-12 |
| 11 | 把 `0x8c044`（XP_PL）当作链路速率寄存器 | 命名候选 `0x8c044/0x2` | 读出 `0xbadf5040`，即特权掩码哨兵；探测写入测试跳过它。值得注意的是，在参考 A100 上，同样的三个寄存器在*每一*代际都读出 `0xbadf5040` | 2026-07-20 |

### 固件与签名攻击

| # | 方法 | 怎么死的 |
|---|---|---|
| 12 | 编辑 VBIOS devinit Gen-strap 字节 | 跨 3 个 devinit 位置的 5 个字节（通过搜索对 Falcon 寄存器 `0x14118F78`、字节模式 `78 8f 11 14` 的引用找到）。与 A100 SXM4 的差异字节：命中 #8 `0xBB` 到 `0xE2`，命中 #10 `72 DE` 到 `52 DD`，命中 #11 `97/59` 到 `95/39`。全部五个都在 `csecret(2)` MAC 范围内。**已关闭** |
| 13 | 重新刷写被编辑的 VBIOS（`nvflash` / CH341A） | Ampere RSA 签名校验拒绝它；卡将无法启动 |
| 14 | RAM 补丁 TOCTOU（在加载与校验之间补丁已签名固件） | 在 Ampere 上已关闭：签名验证发生在 DMA 进入 IMEM **期间**，所以不存在加载与验证之间的窗口。这推广到针对该部件的任何固件级攻击 |
| 15 | `csigenc` ACL-`0x13` 溢出（在 1 位启动预言机之外泄漏高安全秘密） | 离线死路。`envydis` 显示 SEC2 booter 安全主体在 `csecret(6)` AES 下从 `0x101` 到 `0x86FB` 都是密文，明文桩中零 SCP/加密操作码。没有可固定的 ROP 地址 |
| 16 | 主密钥签名绕过 / 任意高安全 Falcon 代码 | 不存在漏洞。已知的时序漏洞只产生**仅数据的寄存器戳**，而不是任意 Falcon 代码，因为主体是 AES 加密且不可签名的。明文在 `0x101` 结束。不存在高安全可达的 Ampere CVE |
| 17 | 泄漏的生产 HULK 证书 | 位于 ROM 中 `0xFE504`，`csecret(40)`，`STRICT_ID_MATCH=NO`。被 `RmActivateHulk` fmodel 标志门控，在生产硅片上为 false；需要证书文件；而且卡上 FEAT_OVR 写入反正不会移动 `OPT_GEN23`（见 #5）。基本无意义 |
| 18 | `csecret(6)`/`csecret(2)` 故障注入（EM 或电压毛刺） | 约 400-2000 美元的设备、数周工作、无保证，而且之后该部件对 PCIe 而言**仍然**受熔丝限制。工具离线验证过，设备从未购置。曾提出 ChipSHOUTER CW520，从未尝试 |

### 硬件与平台攻击

| # | 方法 | 怎么死的 |
|---|---|---|
| 19 | 把 A100 的 strap 配置复制到 170HX 上 | 一位已经让 Gen2 x16 工作起来的测试者试过：**启动时卡未被检测到**。后续回答很直接："the straps don't do anything"（strap 什么也不做）、"falcon is driving the rewrites"（是 falcon 在驱动重写）、"there's no gen3 override register"（没有 gen3 覆盖寄存器）。Strap4（`R999`/`R1000`，靠近 `U808`）被映射为 `PCIE_CFG`。第二位研究者独立发现，把 strap 配置文件与实时 A100 转储对比，两天后也是一条死路 |
| 20 | 普通 PCIe redriver | redriver 只重新放大；端点仍然以自己熔丝封顶的 TX 速率发送。只有**重定时器（retimer）**——它终结链路并能向两侧通告不同速率——才能伪造 TS1/TS2 Rate-ID。已命名候选：Astera Aries、TI DS160PR810 类。从未尝试 |
| 21 | 从驱动内部完整移除并重新扫描（"方案 A"） | 三个注意事项：GSP 启动钩子运行在 `probe()` 内部，所以在那里调用 `pci_stop_and_remove_bus_device()` 是对自身上下文的一次 use-after-free；重新扫描后驱动重新探测、GSP 启动、写入运行、然后再次扫描（需要一个模块级 once 标志）；活跃的 CUDA 客户端会被丢弃。最终发布的是方案 B（上游桥重训练） |
| 22 | 设备 ID 伪造伪装成 A100 | 探测过的每个 Ampere 部件上 `FUSE_DEVID_SW_OVR_DIS` `0x00820584` = `0x00000001`。写入 XVE 配置影子 dword0 `0x88000 = 0x208210de` 只改变主机可见 ID，而 `MAGIC_D` 位 25、PPCI_2 SPEED 和被抑制的 `0x88CE4` 都保持不变 |
| 23 | 刷一张真品 A100 80GB VBIOS 以恢复 PCIe 4.0 | 测试过且失败，2026-07-19 报告："Theyve tested that and it doesnt work. the pcie 4.0 bit at least." |
| 24 | VBIOS `CTRL_OPT` / HULK 选项区作为 PCIe 杠杆 | 结构上不可能："CTRL_OPT is remove only, not add"（CTRL_OPT 只能移除，不能添加） |

### 值得记录的虚假声明

- 一个号称达到 **PCIe Gen 4** 的 fork 在 2026-07-19 一小时内被拆穿（"This is BS, didn't work for me at all"）。两位测试者的主机反正都只限于 Gen3，所以 Gen4 结果根本无法被观察到。2026-07-21 撤回。
- **"PCIe Gen 3 实际上正在工作"——通过 AI 驱动的实验**（2026-07-24）。从未发布任何测量、任何寄存器写入或任何链路状态输出。该说法以玩笑口吻出现，紧接着的讨论仍然把 Gen3 和 Gen4 当作未解决。
- 一条宣传 170HX 为"PCIe 3.0"的公开租赁列表被平台判定为错误报告；同一天记录到 `OPT_GEN23` 写入失败。
- **"Gen 3.0 和 4.0 因为 die 中的熔丝阻塞器而是死路"** 被频道内反驳："the fuses are signals used by the firmware to control function"（熔丝是固件用来控制功能的信号）、"they're not hard efuses that actually destroy functionality"（它们不是真正摧毁功能的硬 efuse）。反驳更有依据，因为 Gen2 解锁证明至少有一个熔断的代际限制是固件介导且可击败的。悬而未决，倾向反驳。

---

## Gen4 影子实验及其启动循环

一个独立的净室补丁 `0007-pcie-gen4-shadow.patch`（不要与 cmpunlocker 的 `0007-pcie-gen2.patch` 混淆，那是另一个同编号但不同的补丁）以启动循环收场，仍是 Gen4 最有趣的未完成产物。

> [!CAUTION]
> **这个实验会让启动循环一直持续到模块被移除**
>
> 上游补丁 `0001`-`0006` 每次启动使用 4 次 Booter 载荷运行，启动正常。Gen4 影子补丁把它提高到 7-11 次，包括熔丝和重训练尝试。**真正的** BooterLoad 随后以 `mailbox0 != 0`（状态 `0xffff`）失败，之后 RM 无限重试 `_kgspBootGspRm`，`wprStart` 每次重试都沿着帧缓冲下滑（每次重试的 WPR 分配），最终回绕。

一个原因被排除：设置 `CMP_PCIE_RETRAIN=0` 时循环仍然存在，排除了驱动内重训练。两个假设存活下来且从未定案：

- **H-COUNT。** 在真正启动前紧接着执行太多 Booter / 特权序列器会耗尽序列器状态。注意 `kgspExecuteBooterLoad_TU102` 在每次运行前做 `kflcnReset(SEC2)`，因此 SEC2 不累积状态，但特权序列器是独立硬件，不会被重置，WPR2/PLM 寄存器和 XVE 写入也会存活。
- **H-WRITE。** 某次特定写入恰好破坏了 Booter 用来从 sysmem DMA 其签名的链路上的 PCIe 块。首要嫌疑：`0x8C2C0`（LTSSM 配置），然后是 `0x8C040`（SPEED）。

二分测试脚手架已经以编译期开关的形式存在：`CMP_PCIE_ONCE=1`（每个模块生命周期应用一次，因为写入是持久的，所以失败的第一次循环之后是值已应用的干净第二次循环）、`CMP_PCIE_ATTEMPTS=1`，以及分组 `CMP_PCIE_XVE_LTSSM_WRITES`、`CMP_PCIE_VECTOR_SPOOF`、`CMP_PCIE_UNK1C0_WRITE`、`CMP_PCIE_XVE_PHY_WRITES`。规定的二分顺序是先 LTSSM，再向量伪造，再 UNK1C0，最后 PHY。结果从未被记录。

---

## 最有希望的剩余途径

按成本排序，最便宜的在前。以下均未做过。

### 1. 通过 xp3g 表写入 `0x00820580 = 0`，然后请求 TLS = 3

成本：一次启动。`FUSE_PCIE_GEN3_DIS` 从未被任何人写入。表机制已存在于 `0007-pcie-gen2.patch` 中；增加一个条目并调高 `target_gen` 是几行改动。根据上面的 #6，预期结果是 `booter FAILED to set` 一行和 `rd=0x00000001`，但这个负面结果值得记录在案。决定性观察是 `LnkCap2` 是否曾达到 `0x0000000E`。

### 2. `FUSE_PCIE_MAGIC_D` 可写吗？读、写 `0x00200000`、再读回

成本：五分钟，从未发布过。证据真的相互矛盾。一项分析注释位 25 `GEN4_SPEED_DISABLED` 并明确把寄存器标为 **"(writable)"（可写）**，与"无需写入"的 `GEN23_DIS` 形成对比。一个独立的净室链脚本记录了把 `0x00820520 = 0x00200000`（A100 / Drive 参考值）写入作为一条*可用* Gen2 链的一部分。但 PCIe 现场手册把 `0x820580` / `0x820520` 列为只读熔丝选项影子，且 `0007` 只读 `0x00820520`。由于 Gen4 无法测试，这从未被实际验证。

### 3. 从 SEC2 高安全上下文内部读取 `0x85080` / `0x85084` / `0x881C0`

成本：一次插桩构建。三者从主机和从注入点都读出毒值。`0x8e1b0` 和 `0x823800` 已被证明可从 HS 到达，因此读取可行。这是定位实际提供支持速率向量的 strap 层的唯一途径。

### 4. 测试 `0x823830`-`0x82383C` 处的第二个功能覆盖组

成本：一次 HS 写入后读回。从 PL0 读取返回 `0xbadf5040`；HS 读取返回真实值。没有手动 PLM 覆盖该组，也从未执行过 HS 写后读回。被明确列入"可写性仍未知 / 值得测试"。

### 5. 在强制 Gen3 尝试期间转储 `LnkSta2` 均衡字段

成本：带插桩的一次启动。反假设是 `GEN3_DIS` 可能在启动时被锁存到一个可重写的 PHY/strap 配置寄存器，而不是直接被模拟 PHY 消费，如果那样，就会存在一个启动后寄存器可以覆盖。提出者自己押注自己的想法是错的。能定案的测量是均衡 Phase 1 是否曾被进入。

### 6. GSP-RM 补丁

截至 2026-07-27 的陈述需求，也是没人交付的原因："我没见过任何人做出过可用的 GSP 补丁。" 具体起点是上面那对熔丝读取位置（`470.42.01 gsp.bin` 中的 `0x5D55834`、`580.105.08 gsp_tu10x.bin` 中的 `0x4DD9B00`）。问题是能否像 Gen2 覆盖转移 Gen2 路径那样转移这种消费。加密的 GSP 区域仍然不可读，这是长期存在的障碍。

### 7. 刷入带 `[+0x0F] = 0x00` 和 `[+0xC9] = 0x14` 的已签名或以其他方式被接受的闪存

这是唯一能定案 DevInit 五字节编辑**单独**能否恢复 Gen4 的实验。熔丝参考 gist 断言"PCIe 双重锁定：`FUSE_PCIE_GEN23_DIS` = `0x1`（熔丝）+ devinit（5 字节）。仅固件补丁不够"，但该结论是对熔丝值的推断，而不是尝试固件补丁的结果。从来没有人刷写过被修改的 DevInit 表，没有签名密钥谁也做不到。

### 8. 线级 retimer

受设备和板卡工作限制。拥有元件和板卡制造能力的人需要构建一个伪造 TS1/TS2 Rate-ID 的中介层。已命名候选：Astera Aries、TI DS160PR810 类。没有尝试过任何东西。

### 9. 找到一台 Gen4 主机

在技术被阻塞之前先被硬件阻塞。研究 Gen4 的人直说："I can't do PCIe Gen 4 because I don't have a computer that supports it"（我做不了 PCIe Gen 4，因为没有支持它的电脑），另外还说"devinit routes are genuinely horrible to try to work on"（devinit 路线真的很难搞）。

### 低优先级线索

一位研究者把 `Mellanox-ConnectX-5-PCIe-Gen-4-Enablement` 标记为类似的"发布时被降级部件"案例，明确说"not expecting much"（不抱太大期望）。没有尝试任何东西。

---

## 记录中的移动靶问题

Gen3 路线在大约 40 小时内有四次方向转变，在把任何单条引语当作项目立场之前值得了解：

| 时间戳 | 立场 |
|---|---|
| 2026-07-26 06:38 | "a devinit route might be the only way"（devinit 路线可能是唯一的路） |
| 2026-07-26 14:25 | "My current fix doesn't use devinit, and it's a dead end"（我当前的修复不用 devinit，而且它是死路） |
| 2026-07-26 14:42 | "We need to use devinit"（我们需要用 devinit） |
| 2026-07-27 22:57 | "Gen 3 doesn't work whatsoever, it's going to require a GSP patch"（Gen 3 完全不行，它需要 GSP 补丁）/ "I haven't seen anybody at all get a working GSP patch"（我没见过任何人做出可用的 GSP 补丁） |

最后一条就是截至 2026-07-28 的状态。

更早的"四层墙"现场手册（日期 2026-07-24）得出结论，四层（运行时寄存器写入、寄存器语义、持久固件、硅片熔丝）在经验上全部关闭，并在两个表面上验证：一次 4032 次运行的离线固件模糊扫描（从 66 个函数中抽取 126 个函数-寄存器对，每个对 32 个单比特值扫描）和硅片上直接写入探测。它自己的第 6 节包含它在 Gen2 上出错的原因：*"完整的社区 Gen2 序列……从未作为单个组合写入运行：每个组成部分都单独被证明是惰性的，所以它是一个低概率组合。"* 这个低概率组合成功了。**该结论的 Gen3 那一半仍然成立。**

---

## 如果你在测试一个 Gen3 声明

> [!WARNING]
> **用 LnkSta 验证，绝不用 LnkCap**
>
> `LnkCap` 是通告的能力，当链路仍在 Gen1 训练时它可以读出更高的代际。这个陷阱被指为大多数站不住脚的"它工作了"声明的来源。2026-07-24 的 Gen3 通告结果正是这种情况。

```bash
# 三个诚实的字段
sudo lspci -vvs <bdf> | grep -E 'LnkCap:|LnkCap2:|LnkSta:'
cat /sys/bus/pci/devices/<bdf>/current_link_speed
nvidia-smi --query-gpu=pcie.link.gen.gpucurrent --format=csv
```

这张卡上有两个已知的假信号：

- 在 Gen2 训练成功的 170HX 上，`/sys/.../max_link_speed` 仍然读出 `2.5 GT/s`，而 `current_link_speed` 读出 `5.0 GT/s`。从配置空间诊断，不要从 sysfs 属性诊断。
- 自 2023 年以来，`nvidia-smi` 在出厂卡上报告 `PCIe Generation Max : 2`，而 `Device Current` 和 `Device Max` 都读出 `1`，`LnkCap2` 只列出 2.5 GT/s。只作为指纹有用。

ASPM 在其他平台上是一个真正的假阴性陷阱（许多平台把链路空闲降到 Gen1），但 170HX 本身在其 `LnkCap` 中通告 `ASPM not supported`，所以它在这里是第一个诊断线索而不是可能的原因。

---

## 实测值

| 数值 | 值 | 条件 | 置信度 |
|---|---|---|---|
| `FUSE_PCIE_GEN23_DIS` `0x0082057c` | `0x00000001` | 两个 170HX SKU、两块物理单元，各读两次；13 块对比件上为 `0x00000000` | 高 |
| `FUSE_PCIE_GEN3_DIS` `0x00820580` | `0x00000001` | 同上 | 高 |
| `FUSE_PCIE_MAGIC_D` `0x00820520` | `0x16680000`（位 25 置位） | 170HX；A100 家族 `0x00200000` | 高 |
| `OPT_PCIE_LANE_DISABLE` `0x00820394` | `0x00000000` | 170HX | 高 |
| `CTRL_OPT_PCIE_LANE` `0x0082082c` | `0x00000000` | 170HX | 高 |
| `STATUS_OPT_PCIE_LANE` `0x00820c2c` | `0x00000000` | 170HX | 高 |
| `FUSE_EN_SW_OVERRIDE` `0x00820040` | `0x00000000` | 170HX 和所有数据中心 GA100；消费级部件上为 `0x00000001` | 高 |
| `FUSE_DIS_SW_OVR` `0x00820084` | `0x00000001` | 所有卡 | 高 |
| `0x00820148`（DevInit MAGIC_D 门控） | `0x00000000` | OTP 备用位，永远无法从软件设置 | 高 |
| Gen2 dmesg 中的 `OPT=` 三元组 | `00000001/00000001/16680000` | 一次完整 Gen2 运行后的 GEN23 / GEN3 / MAGIC | 高 |
| PLM 打开后的 XP3G 速率 | `0x00340036`，`ovr0 = 0x4` | 写入生效，链路保持 Gen1（`lnksta=0x10410040`，速率 1） | 高 |
| `0x85080` / `0x85084` | `0xBADF1100`（毒值） | 从注入点读取 | 高 |
| `0x881C0` 主机读取 | `0xbadf5040` | 特权屏蔽模式 | 高 |
| A100 上的 `0x8C1C0` | `0x00040036` | PPCI_2 UNK1C0 参考 | 高 |
| A100 `0x8C044` / `0x8C048` / `0x8C04C` | 每一代际都是 `0xbadf5040` | 即使在参考卡上也被屏蔽 | 高 |
| CMP `0x88CE4` | `0x0000003F` | 对比 A100 `0x00000014` | 高 |
| CMP `0x88CE0` 低 6 位 | `0x02` | 对比 A100 `0x06` | 高 |
| CMP `0x8C040` `PPCI_2.CONFIG_LINK` | `0x800C4C00`（SPEED = 3） | BAR0 mmap，无驱动；A100 `0x80004C00`（SPEED = 0） | 高 |
| CMP `0x8C2C0` | `0x068731B7` | 对比 A100 `0x060711B2` | 高 |
| CMP `0x880A8` | `0x00000001` | 对比 A100 `0x001F0004` | 高 |
| CMP `0x88084` / `0x880A4` | `0x00456101` / `0x00000002` | 对比 A100 `0x00457104` / `0x0180001E` | 高 |
| `0x118F78` / `0x132B70` | CMP 和 A100 上都是 `0` / `0` | **BAR0 mmap，未加载驱动**（同样的地址在驱动运行的主机读取中返回 `0xbadf1100`）；相同的值不可能编码 SKU 限制 | 高 |
| `0x132B30` / `0x132B6C` / `0x132B50` | 两者上都是 `0x00000400` / `0x08000020` / `0x03780000` | 空闲，无驱动 | 高 |
| LTSSM 超时 `0x8D1A0` / `0x8D1A4` | `0x1B1F2327` / `0x0B0F1317` | CMP 与 A100 相同 | 高 |
| 每次启动的 Booter 载荷运行次数 | 4（补丁 0001-0006，启动正常）对比 7-11（Gen4 实验，启动循环） | GSP 启动 | 高 |
| Booter 载荷运行状态 | 每次运行都是 `0xffff`，即使写入落地了 | 寄存器回读是唯一有效判据 | 高 |
| SEC2_DEBUG dmesg 行数 | 29（存档单卡抓取）、34（Gen1 构建）、80（Gen2 构建）、134（存档双卡 Gen2 分支 610.43.03 日志）、152（两套双卡 Gen2 机器上的 `pcielink.sh`） | **不是可靠的跨构建指纹**；不要把不一致读成安装失败 | 高 |
| Gen3 通告结果 | `LnkCap Speed 8GT/s`、`LnkCtl2 Target 8GT/s`、`LnkSta Speed 2.5GT/s` | 2026-07-24 | 高 |

---

## 参见

- [PCIe Gen2](../unlock/pcie-gen2.md)，了解确实有效的机制
- [PCIe 子系统](../hardware/pcie-subsystem.md)，了解寄存器块地图
- [熔丝与 OTP](../hardware/fuses-and-otp.md)，了解完整熔丝群
- [VBIOS](../hardware/vbios.md)，了解 DevInit 镜像布局与签名
- [物理改造](../operations/physical-mods.md)，了解只改变宽度的电容改造
- [死路](../history/dead-ends.md) 与[开放问题](open-questions.md)
- [状态板](status-board.md)
