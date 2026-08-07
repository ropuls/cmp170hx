# PCIe 子系统

**本页涵盖：** CMP 170HX PCIe 接口出厂时的物理与固件状态。卡出厂时带的两项限制、速率上限背后的熔丝与 DevInit 证据、宽度上限背后的删减交流耦合电容、原厂卡的精确寄存器与 `lspci` 状态，以及会被误认为这两者的平台级混杂因素。速率上限的软件破解见[Gen2 解锁](../unlock/pcie-gen2.md)；宽度上限的焊接工作见[物理改造](../operations/physical-mods.md)。

## 一句话总结：两个上限、两套机制、两种修复

原厂 CMP 170HX 以 **PCIe 第 1 代（2.5 GT/s）× x4** 训练。这是两个完全独立的限制，恰好共存于同一块板上，破解其中任何一个对另一个毫无作用。

| | 速率上限 | 宽度上限 |
|---|---|---|
| 观察状态 | 2.5 GT/s（第 1 代） | 训练 x4，通告 x16 |
| 机制 | OTP 熔丝加签名 DevInit 表，固件强制执行 | 16 条通道中的 12 条出厂时省略交流耦合电容 |
| 所在位置 | 硅片与 SPI 闪存 | PCB |
| 破解方式 | 未发布分支上的驱动补丁（仅 Gen2） | 手工焊接 24 × 0402 电容 |
| 状态 | Gen2 于 2026-07-24 软件达成，**未发布**；Gen3 和 Gen4 未达成 | 2026 年 4 月以来多位独立改装者复现 |
| 会改变另一个吗？ | 不会。Gen2 补丁未改造卡是 Gen2 x4。 | 不会。完全改造未打补丁卡是 Gen1 x16。 |

两者独立的最清晰单一证明：一张原厂、从未焊接的 8 GB 卡运行 Gen2 代码时报告 `LnkCap: Port #0, Speed 5GT/s, Width x16`，而 `LnkSta` 读 `Speed 5GT/s, Width x4 (downgraded)`。能力寄存器说 x16；训练出的链路说 x4。软件无法弥合这个差距，因为 12 条通道上没有电气路径。

> [!WARNING]
> **实验性质**
>
> 本页关于 Gen2 的一切都描述未发布分支代码。正式发布的 `master` 不含任何 PCIe 补丁。见[Gen2 解锁](../unlock/pcie-gen2.md)。

## 原厂链路状态

### `lspci` 打印什么

```console
$ sudo lspci -s 0a:00.0 -vvv | grep -E 'LnkCap|LnkSta|LnkCtl'
LnkCap: Port #0, Speed 2.5GT/s, Width x16, ASPM not supported
        ClockPM+ Surprise- LLActRep- BwNot- ASPMOptComp+
LnkCtl: ASPM Disabled; RCB 64 bytes, Disabled- CommClk+
LnkSta: Speed 2.5GT/s, Width x4 (downgraded)
LnkCap2: Supported Link Speeds: 2.5GT/s, Crosslink- Retimer- 2Retimers- DRS-
LnkCtl2: Target Link Speed: 2.5GT/s, EnterCompliance- SpeedDis-
LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete-, EqualizationPhase1-
```

两个细节值得记牢。`LnkCap` 通告 **Width x16**，所以卡知道自己有十六条通道；`LnkSta` 上的 `(downgraded)` 标记意味着链路*协商*降级了——当接收端在通道 4 到 15 上看不到信号时就会这样。而 `LnkCap2` 只列出 2.5 GT/s 为支持速率，按 PCIe 规范，这会钳制你写入 `LnkCtl2` 的任何目标链路速率：在原厂卡上写 `0x2` 读回 `0x1`。

内核每次启动都用自己的话表达同一件事：

```text
pci 0000:0a:00.0: 8.000 Gb/s available PCIe bandwidth, limited by 2.5 GT/s PCIe x4 link at
  0000:09:01.0 (capable of 32.000 Gb/s with 2.5 GT/s PCIe x16 link)
```

注意内核在拿什么比较：**2.5 GT/s x16 下的 32 Gb/s**。它抱怨的是宽度，不是速率。

### 原始寄存器值

下面所有地址都是 XVE 配置空间的 BAR0 镜像。PCIe Express 能力位于配置偏移 `0x78`（不是 `0x60`），XVE 影子基址是 `0x88000`，所以配置 `cap+0x0C` 映射到 BAR0 `0x00088084`，以此类推。

| 字段 | 配置偏移 | BAR0 镜像 | 原厂，未解锁 | 使用解锁器 |
|---|---|---|---|---|
| LnkCap | `CAP_EXP+0x0C` | `0x00088084` | `0x00456101` | `0x00456102` |
| LnkCtl / LnkSta | `CAP_EXP+0x10` | `0x00088088` | LnkCtl `0x0140`，LnkSta `0x1041` | LnkCtl `0x0140`，LnkSta `0x1042` |
| LnkCap2 | `CAP_EXP+0x2C` | `0x000880a4` | `0x00000002` | `0x00000006` |
| LnkCtl2 / LnkSta2 | `CAP_EXP+0x30` | `0x000880a8` | `0x0000` / `0x0000` | `0x0002` / `0x0001` 或 `0x0000` |
| DevCap2 | | | `0x00070803` | `0x00070813` |
| DevCtl2 | | | `0x1400` | `0x0400`（一台机器 `0x7410`） |

`nvidia-smi` 在没有装解锁器的卡上把 `pcie.link.gen.current, pcie.link.gen.max, pcie.link.width.current` 报告为 **1, 1, 4**，装了解锁器后为 **2, 2, 4**。宽度在两种情况下都不动。

> [!NOTE]
> **不要重复一个流传的地址**
>
> 一份广为流传的现场手册把 LnkCap2 的 BAR0 镜像列为 `0x8808C`。那在内部是不一致的：XVE 镜像基址 `0x88000` 时，配置 `0xA4` 映射到 `0x880A4`，而 `0x8808C` 映射到配置 `0x8C`。分支补丁（`#define PCIE_GEN2_LINK_CAP2_ADDR 0x000880a4U`）和社区 `pcielink.sh` 诊断工具都用 `0x880A4`。请用 `0x880A4`。

### 其他原厂配置空间事实

| 项 | 值 |
|---|---|
| 插槽供电上限（DevCap） | 75 W |
| MaxPayload / MaxReadReq | 256 字节 / 512 字节 |
| ASPM | 不支持（LnkCap 中通告） |
| FLReset | 支持 |
| BAR0 | 32 位不可预取区域中的 16 MB，窗口 `0x1000000` |
| BAR1 | 64 MB，64 位可预取 |
| BAR3 | 32 MB，64 位可预取 |
| 可调整大小 BAR 能力 | 存在于 `[bb0 v1]`，但每个 BAR 只通告一个支持的大小 |

BAR1 无论卡报告多大的帧缓冲都停在 64 MiB，所以即使报告 81920 MiB 的卡也无法做全显存主机映射。因此可调整大小 BAR 是"通告了但功能上惰性"。旧的"ReBAR 需要 PCIe 3.0"反对意见是错的（ReBAR 是配置空间能力，2007 年起就在规范里，与链路代数无关），但也没人在 Gen2 训练成功的 170HX 上演示过可用的 ReBAR。

## 速率上限

### 熔丝证据

三个 OTP 熔丝影子构成锁的指纹。170HX 读 **1 / 1 / `0x16680000`**，而每个对比 Ampere 部件都读 **0 / 0 / 第 25 位未置位的某值**。

| 寄存器 | 地址 | 170HX（两个 SKU） | A100（全部三个 SKU）与 Drive A100 | 备注 |
|---|---|---|---|---|
| `FUSE_PCIE_GEN23_DIS`（`OPT_PCIE_BOOT_GEN23_DISABLE`） | `0x0082057c` | `0x00000001` | `0x00000000` | A10、A5000、A6000、RTX 3080 / 3080 Ti / 3090 / 3090 Ti 和一张 GA10x 对照卡上也是 0 |
| `FUSE_PCIE_GEN3_DIS`（`OPT_PCIE_BOOT_GEN3_DISABLE`） | `0x00820580` | `0x00000001` | `0x00000000` | 同一队列 |
| `FUSE_PCIE_MAGIC_D` | `0x00820520` | `0x16680000`（第 25 位置位） | `0x00200000` | 第 25 位文档化名为 `GEN4_SPEED_DISABLED`，引用 NVIDIA bug 2220334。A10/A5000/A6000 读 `0x01a00000`；RTX 30 系列读 `0x10a80000` |

两个 170HX 值都在两块物理单元上测量，每块部件读两次，横跨 15 卡对比队列。`0x20c2` 读数来自驱动侧转储打印的 `OPT=00000001/00000001/16680000`；`0x2082` 读数来自独立探测的 `registers.json`。

两个相关熔丝关死了显而易见的变通方案。`0x820040` 的 `FUSE_EN_SW_OVERRIDE` 读 0、`0x820084` 的 `FUSE_DIS_SW_OVR` 读 1，所以软件覆盖路径在硅片层被禁用。`0x820148` 是一个读 0、且软件永远无法置位的 OTP 备用位；DevInit 只在 `0x820148 & 1` 时把 A100 值 `0x00200000` 写入 `MAGIC_D`——这正是 DevInit 在 CMP 上从不写它的原因。

`0x0082057c` 的 `OPT_GEN23` 已从每个可用特权被攻击过：普通主机写入、HS 特权驱动写入、以及 SEC2 Booter 载荷。每次尝试都以回读仍为 `0x00000001` 失败。它是纯熔丝感测反射，没有写端口。更多见[熔丝与 OTP](fuses-and-otp.md)和[死路](../history/dead-ends.md)。

> [!NOTE]
> **熔丝不是杠杆**
>
> Gen2 解锁在 **`OPT_GEN23` 仍读 `0x00000001` 的情况下工作**。发布分支补丁仍然尝试写入、仍然失败，Gen2 仍然训练成功。真正起作用的杠杆是 CYA_0、LINK_CONFIG_0、XP3G 和 PRIV_MISC_1 覆盖，而不是熔丝影子。

### DevInit 层

熔丝只是三层中的一层。第二层是 SPI 闪存中**未加密** DevInit Falcon 镜像里的 PCIe 配置表，它独立于传统 x86 VBIOS。

| 项 | CMP 170HX | A100 |
|---|---|---|
| PCIe 配置表，闪存偏移 | `0x420ED`（镜像 `0xA20ED`） | `0x408A0`（镜像 `0xA08A0`） |
| 运行时 DMEM 基址 | `0xF1D` | `0xE50` |
| 五个字节，表偏移 `+0xC7` 到 `+0xCB` | 闪存 `0x421B4`–`0x421B8` 处 `00 00 08 00 06` | 闪存 `0x40967`–`0x4096B` 处 `00 00 14 00 06` |
| 表偏移 `+0x0F` 的抑制标志 | `0x01` | `0x00` |
| DevInit 镜像位置 | 闪存 `0xDE00`（反汇编基址 `0x8000`），bank 2 中 `+0x60000` 处有副本 | |

抑制标志是关键：在 CMP 上，`ld b8 r9, D[tab+0x0F]; bra ne` 会跳过整个 Gen4 编程块（反汇编 `0x31B3F`–`0x31B92`）。该块运行时计算 `0x88CE4 = (old & ((b1<<8)|b0)) | ((b3<<8)|b2)`，因 `b0 = b1 = b3 = 0` 简化为字节 `[+0xC9]`；`0x88CE0 = (old & ~0x3F) | (b4 & 0x3F)`，两块部件上 `b4 = 0x06`。更宽的符号分析发现总共有**十三个** DevInit 字节不同，其中十一个可归因于非 PCIe 的 SKU 功能（HBM、NVLink、ECC）；与 PCIe 相关的有 `[0xC9]`（送入 `0x88CE4`）、`[0x1C-0x1F]`（送入 `0x8C2C0`）和 `[0x3F]`（送入 `0x8C040`）。CMP 与 A100 的 BIT 表逐字节相同。

编辑这些字节是关闭的路线。全部五个都**100% 位于** Davies-Meyer `csecret(2)` MAC 范围 `0x2200`–`0x43C00` 内，所以无密钥伪造是 2^128 次第二原像，而重刷编辑过的镜像会直接通过 Ampere RSA 签名检查失败。见[VBIOS](vbios.md)。

> [!CAUTION]
> **不要尝试修改后重刷**
>
> 编辑过的 DevInit 或 VBIOS 镜像会被签名检查拒绝，卡将无法启动。恢复需要外部编程器。动闪存之前先读[恢复](../procedures/recovery.md)。

第三层是运行时：DevInit 本身从不读 `0x82057C` 或 `0x820580`。对 CMP DevInit 反汇编的穷举搜索只找到 `0x820C14`/`0x820D38`（FBIO/FBP 筛选）、`0x820684`（`FUSE_NVLINK_DIS`）、`0x82380C`/`0x823814`、`0x820520`、`0x820148`、`0x8243xx`、`0x8202xx`、`0x8201xx`、`0x82033C`/`0x82030C`。GSP-RM 才是消费者：`470.42.01 gsp.bin` 中 `0x5D55834` 的熔丝读取跳转表使用 `li a2, 0x580` 和 `li a2, 0x57c`，而 `580.105.08 gsp_tu10x.bin` 在 `0x4DD9B00` 用 `li a2, 0x57c` 做 `jalr fuse_read`。这就是 Gen3 路线目前被描述为需要 GSP 补丁的原因。

### 启动顺序

FWSEC-DevInit 在 SEC2 Booter 运行之前编程并**闩锁** `SUPPORTED_LINK_SPEED`，而 SEC2 Booter 正是解锁时序洞小工具所在。因此闩锁的能力在任意利用窗口打开时已经固定。内存和计算解锁能落位，是因为 `FEAT_OVR`（`0x82381C` / `0x823804`）和 FBPA（`0x9A0204`）是 16 MB BAR0 内部的普通寄存器，PLM 一打开就可写。闩锁的 PHY 能力则不是。Gen2 结果实际做的是让 PHY 反射重新生成到 Gen2，然后在任何东西重新钳制之前训练链路。

### 速率能力逐寄存器在哪

| 寄存器 | 地址 | 访问 | 备注 |
|---|---|---|---|
| 支持速率来源 | `0x00085080` | 只读，`[23:20]` | 从主机读 `0xBADF1100`（毒值）；在 410 万行 RM 反汇编中找不到写入者 |
| 允许的 Gen 掩码 | `0x00085084` | GSP-RM 在每次重新训练时重新推导 | 也读毒值 |
| `MAX_LINK_SPEED` | `0x00088084` `[3:0]` | PHY 反射，标记 `R-XVF` | 无写端口 |
| `SUPPORTED_LINK_SPEED` | `0x0008808C` `[7:1]` | PHY 反射，标记 `R-EVF` | 任何特权都无写端口 |
| `TARGET_LINK_SPEED` | `0x000880A8` `[3:0]` | RW，但被 SUPPORTED 钳制 | |
| `LINK_CONTROL_STATUS` | `0x00088088` | `[19:16]` 为现行协商速率 | |
| `PRIV_MISC_1` | `0x0008841C` | PLM 下 RW | CYA Gen2/3 覆盖位 11–16、30、31 |
| `VSEC_HIERARCHY` | `0x00088610` | PLM 下 RW | 第 12 位门控 PRIV_MISC_1 重编程；现行值 `0x00001001` |
| LTSSM 重新训练触发 | `0x0008872C` | PLM 下 RW | 写 `6` |
| `PPCI_2.CONFIG_LINK`（`LINK_CONFIG_0`） | `0x0008C040` | PLM 下 RW | `[3:0]` LTSSM_DIRECTIVE、`[4]` LTSSM_STATUS、`[19:18]` SPEED（0 = 最大，2 = 5.0 GT/s，3 = 2.5 GT/s）。CMP 读 `0x800C4C00`（SPEED = 3）；A100 读 `0x80004C00`（SPEED = 0） |
| `CYA_0` | `0x0008C2C0` | PLM 下 RW | 第 2 位是 `DIS_G2` 鸡位。CMP `0x068731B7` 对 A100 `0x060711B2` |
| `PL_LINK_RATE` | `0x0008C1C0` | | A100 读 `0x00040036` |
| `PPCI.UNK1C0` | `0x000881C0` | 主机读取返回 `0xbadf5040` | rnndb：`[17:16]` LNK_CAP_SPEED、`[21:20]` SYSTEM_MAX_SPEED |

通篇使用的速率向量编码：Gen1 = `0x1`、Gen1_2 = `0x3`、Gen1_2_3 = `0x7`、Gen1_2_3_4 = `0xF`。

块布局沿用 envytools rnndb 对 GK104 及以后部件的命名：**PPCI** 在 `0x88000`（配置影子加 priv）、**PPCI_HDA** 在 `0x8A000`、**PPCI_2** 在 `0x8C000`（LTSSM 与速率块，含 `0x8C040` 的 `CONFIG_LINK` 和 `0x8C080` 的 `WIDTH`，后者在 A100 上读 `0x00001010`）。完整清单在[寄存器索引](../appendix/register-index.md)。

## 宽度上限

### 是缺件，不是熔丝也不是固件

170HX 十六条 PCIe 数据通道中的十二条出厂时物理省略了交流耦合电容。每个差分对两颗，所以十二条通道意味着**缺 24 颗器件**。NVIDIA 只焊了它打算让卡用的四条通道。通道 0 到 3 焊装；通道 4 到 15 未焊装。

三条独立证据排除了所有软件解释：

1. **没有置位的通道熔丝。** `0x00820394` 的 `OPT_PCIE_LANE_DISABLE`、`0x0082082C` 的 `CTRL_OPT_PCIE_LANE` 和 `0x00820C2C` 的 `STATUS_OPT_PCIE_LANE` 在队列中每张卡（包括两块 170HX）上都读 `0x00000000`。x16 电气宽度在硅片层完好。
2. **没有代码碰宽度。** 对 Gen2 代码中每个与 PCIe 相关的写入做穷举审计，发现只写 `LINK_CTRL_2 [3:0]`、`LINK_CONFIG_0 [19:18]`、`CYA_0` 第 2 位、`PRIV_MISC_1` 位 11–14、`PL_LINK_RATE`、`OPT_GEN23`、XP3G 槽 0 和 3、VSEC 设备与层级位、以及配置空间的 `LNKCTL2` TLS。`LINK_CAP` 被读，但只测它的低速半字节；`LINK_CAP[9:4]` 的 Max Link Width 字段从不被读或写；`LNKSTA` 用 `PCI_EXP_LNKSTA_CLS` 和 `PCI_EXP_LNKSTA_DLLLA` 掩码，但绝不用 `PCI_EXP_LNKSTA_NLW`。对正式发布的 master 和全部十二个未发布分支 grep "capacitor"、"AC coupling"、"solder" 或任何通道宽度寄存器，一无所获。
3. **已知完好的 x16 主机端口仍然训练 x4。** 2026-07-26 在一台主机的两张卡上测量：sysfs 报告 GPU 宽度为 `cur 4 / max 16`（两张都是），第二张 GPU 的上游端口本身是 x16 能力的（`cur 4 / max 16`），而链路仍然训练 x4。转接卡和插槽分叉假说由 PCB 分析回答，而不是软件里的任何东西。

### 器件

| 属性 | 规范值 |
|---|---|
| 数量 | 24（每差分对 2 颗 × 12 条缺件通道） |
| 封装 | 0402 |
| 容值 | 220 nF（0.22 µF） |
| 介质 | **X7R**（经常被误写为 "XR7"） |
| 耐压 | 6.3 V 或更高。已知成功的 x16 改造用 6.3 V 器件；PCIe 把发送端直流共模限制在 3.6 V，所以 6.3 V 有充足余量 |
| 参考位号 | C1100 到 C1350 范围，例如每对 C1120 / C1125 / C1130 / C1135 |
| 确认的厂商部件 | Taiyo Yuden `MAASJ105SB7224KFCA01`（220 nF、6.3 V、X7R、0402）。Samsung `CL05B224KO5NNNC`（16 V）是报告可用的替代 |
| 见过的分销商编号 | DigiKey `1276-1176-1-ND` 和 Digi-Key `3886834`。两者很可能只是同一厂商部件的不同包装；视为未验证别名，按厂商部件购买 |

这个值不是猜的：它直接读自 NVIDIA A100 GA100-883 参考原理图 **P1001-B02 第 3 页 "IO: PCIe CONNECTOR"**，170HX 板卡严格跟随该原理图。一位测试者报告 100 nF 替代也能工作。

### 实测结果

```text
before:  LnkSta: Speed 2.5GT/s, Width x4 (downgraded)
after:   LnkSta: Speed 2.5GT/s, Width x16
```

用 `sudo lspci -s <bdf> -vvv | grep LnkSta` 验证。速率字段不动，这正是预期结果。

部分焊接会降级协商而不是失败。PCIe 宽度协商按合法宽度 16、8、4、1 回退，所以 24 颗电容中正确焊了 12 到 23 颗的卡训练为 **x8**。一位改装者跨三张卡的进步是 x4、然后 x8、然后 x16；另一张卡"经过小调整"后走 x4、x8、x16。改造后得到 x8 意味着焊接不完整或桥接，而不是独立的硬件上限。回流并检查全部 24 个焊点。

> [!CAUTION]
> **这是在一块你无法替换的卡上做细间距返工**
>
> 0402 器件位于密集的高速差分区域。桥接的一对不仅无法加宽链路，还可能破坏一条原本工作通道上的信号。含铅焊锡被报告能让这项工作"极其轻松"；针头涂锡膏加热风枪可以让器件自对准。完整步骤与照片见[物理改造](../operations/physical-mods.md)。

## 按配置的带宽

| 配置 | 实测 | 方法与条件 | 置信度 |
|---|---|---|---|
| Gen1 x4 | 写 0.85 GB/s，读 0.84 GB/s | clpeak `enqueueWriteBuffer` / `enqueueReadBuffer`，2023 年公布的表格 | 高 |
| Gen1 x4 | 发送 0.80 GB/s，接收 0.84 GB/s，双向 0.81 | 一张由外部硬件小组转述的 OpenCL-Benchmark 截图，10 GB 到 40 GB 卡；工具把链路标为 "Gen1 x16" | 中 |
| Gen1 x16（电容改造，无 Gen2） | 2.88 GB/s 平稳、零错误 | 改造卡；标称约 4 GB/s，缺口归因于 PCIe 1.1 信令开销 | 中 |
| Gen2 x4 | 发送 1.68 GB/s，接收 1.71 GB/s | OpenCL-Benchmark，一张存档截图，未改造卡；设置脚本独立预测 "~0.85 to ~1.7 GB/s, exactly 2x" | 中 |
| Gen1 x8 → Gen2 x8（一张卡上的 A/B） | 1.67 GB/s 到 3.24 GB/s | OpenCL，在一张协商到 x8 的电容改造卡上。这既是**宽度**结果也是速率结果；不要把它引用为 Gen2 x4 数字 | 中 |
| Gen2 x16 | 6.63 到 6.67 GB/s（`ocl_pcie_bw`）；同一次运行的 nvtop 截图显示 `PCIe GEN 2@16x`、TX 7.061 GiB/s。另一台机器报告四张卡各 5.97 GB/s | 装了解锁器的电容改造卡 | 中 |

> [!WARNING]
> **Gen2 x16 建立在一个单一观察上**
>
> Gen2 x16 只被观察到**一次**：2026-07-26，一台机器，一张截图，一张 24 颗电容改造完整的卡。没有 `lspci` 抓取把它连接到更早那次全部 Gen2 结果都是 x4 的调查，没有烤机，没有随时间的 AER 计数器，没有第二台机器。请把 6.63–6.67 GB/s 视为中置信度，把 Gen2 x16 的**稳定性**视为未确立。

一个被描述为 Gen1 x16 的卡流传着 `0.71 GB/s` 的双向数字。对那个配置来说太低了（标称约 4 GB/s），而且该卡真实的通道状态从未确立。不要把它引用为 Gen1 x16 测量。

这些数字在实践中意味着什么，见[性能](../operations/performance.md)和[大模型推理](../operations/llm-inference.md)。简短版：Gen1 x4 下链路是图形的约束（Unigine Superposition 锁在 5 fps，1080p 游戏 15–20 fps，单个 1080p60 远程游戏流就饱和链路），而对流水线并行的大模型解码，链路几乎无关紧要（5120 隐藏维度模型每 token 每跳移动 10,240 字节，所以饱和单条 PCIe 1.0 通道大约需要每秒 25,000 token）。张量与专家并行即使在 Gen2 x16 下也被判定不可行。

## 看起来像原因但并不是的东西

| 嫌疑 | 为什么看着合理 | 为什么不是原因 |
|---|---|---|
| `NV_PTOP_FS4` `0x0002241c` | 文档化的位名字面就是 `GEN2_PCIE`（第 0 位）和 `GEN2_PCIE_SPEED`（第 7 位） | 8 GB（`0x20c2`）卡读 `0x00000000`，10 GB（`0x2082`）卡读 `0x00000081`。一张训练 Gen4 的 GA10x 对照卡读同样的 `0x00000081`，而 10 GB 170HX 读 `0x00000081` 却仍被锁在 Gen1。如果这些位门控速率，这两个观察不可能同时成立。`0x00022470` 的 `PTOP_FS_STATUS` 读 `0x0000003f` |
| 板载跳线 | U808 附近有可见的跳线电阻焊盘；Strap4（R999/R1000）映射为 `PCIE_CFG` | 把 A100 跳线配置复制到 170HX 上导致**启动时检测不到卡**。尝试者的结论："跳线什么都不做"、"是 falcon 在驱动重写" |
| 设备 ID 伪装 | 把卡伪装成 A100 继承其设置 | `0x00820584` 的 `FUSE_DEVID_SW_OVR_DIS` 在每块探测过的 Ampere 部件上都读 `0x00000001`；ID 来自只读熔丝 `0x008204D8` 和 `0x0082056C`。写入 XVE 配置影子 dword0 只改变主机可见的 ID，所有锁原样不动 |
| 刷真 A100 80GB VBIOS | BIT 表逐字节相同，PCB 近乎相同 | 测试过并失败；Gen4 位至少不会带过去 |
| PCIe redriver | 便宜且易得 | redriver 只重新放大，所以端点仍然用自己的熔丝封顶的 TX 速率做源。只有 **retimer**（终结链路并能向两侧通告不同速率）才能伪造 TS1/TS2 Rate-ID。从未尝试 |
| ASPM | 很多平台把空闲链路降到 Gen1 | 测试时是真实的假阴性陷阱，所以要在负载下测试；但 170HX 在自己的 `LnkCap` 里通告 `ASPM not supported`，所以在提出它的那个案例里它不是原因 |

一件值得记录的奇事：在**两张**物理 10 GB 卡上，`0x008204D8` 的 `FUSE_PCIE_DEVIDA` 读 `0x00002082`，而 `0x0082056C` 的 `FUSE_PCIE_DEVIDB` 读 `0x000020c2`。一张 10 GB 卡把 8 GB 变体的设备 ID 当作自己的次级熔丝。横跨 13 张对比卡，熔丝 B 等于熔丝 A 的第 6 位置位（`+0x40`），例如 A100 PCIe 80G `0x20b5`/`0x20f5`。另测得：这些 10 GB 单元上 **`OPT_SKU_ID` 在 `0x00821060` = `0x00000068`**（`0x00000080` 是 8 GB / `0x20C2` 的值），`OPT_INTERNAL_SKU` 在 `0x008203f4` = 0。

## 平台与互连

| 拓扑 | 结论 |
|---|---|
| 裸机 PCIe 插槽 | 支持，且是参考配置 |
| Oculink | 可用。本质是直连 PCIe 转接，有时带 redriver 处理时序 |
| Thunderbolt 3 eGPU 扩展坞 | **完全破坏解锁**，不只是 PCIe。`nvidia-smi` 返回 "No devices were found"，dmesg 显示完整的 GSP 引导失败链（`Booter failed with non-zero error code: 0x15`、`failed to execute Booter Load: 0xffff`、`Max GSP-RM boot attempts exceeded: 4/4`、`RmInitAdapter failed! (0x62:0xffff:2119)`） |
| 虚拟机 GPU 直通 | Gen2 能力被通告，但训练不发生。维护者 2026-07-24 承认，未修复 |
| 被动 SlimSAS / MCIO，70 cm | Gen4 x8 下不可靠（大量错误），Gen3 x8 下稳定。线缆标记 `HNW-SS-8654-AA75`。大多数转接板带 `ICS 9ZXL1950DKIL`——那是**时钟缓冲器，不是 redriver**；`NFHK N-W54B-P` 变体被识别为带真 redriver |
| PCIe 交换机扇出（例如 PEX88096） | 交换机不创造带宽，而且因为 170HX **没有 P2P**，交换机后面的卡无法绕过上行。观察窗口内没人把 170HX 部署在交换机后面 |
| 集群中的 InfiniBand 或高速织网 | Gen1 或 Gen2 下零收益。一位多节点操作者连 10 GbE 都跑不满 |

此卡没有 P2P，任何分支都不含 P2P 启用代码。见[P2P](../frontier/p2p.md)。

## 诊断规则

1. **读 `LnkSta`，绝不读 `LnkCap`。** `LnkCap` 是通告能力，链路还在 Gen1 训练时它就能读成 Gen2。这个陷阱被点名是大多数经不起推敲的"能用"说法的来源。
2. **不要信 sysfs 的 `max_link_speed`。** 两台机器的三张卡上它报告 `cur 5.0 GT/s / max 2.5 GT/s`——最大低于当前速率——而配置空间 `LnkCap` 正确读 `0x00456102`。预期这种错位；它不是故障。
3. **不要把 `nvidia-smi` 的 `PCIe Generation Max` 当任何证据。** 原厂卡自 2023 年起就报告 `Max: 2` 配 `Device Current: 1` 和 `Device Max: 1`，而 `LnkCap2` 只列出 2.5 GT/s。它只适合当指纹。
4. 三个诚实的字段是 `lspci -vvs <bdf> | grep LnkSta`、`/sys/bus/pci/devices/<bdf>/current_link_speed` 和 `nvidia-smi --query-gpu=pcie.link.gen.gpucurrent`。

社区的标配链路报告是公开的 `pcielink.sh` 诊断工具，它抓取内核、驱动、SEC2_DEBUG 行数、BDF、板卡与 GPU 部件号、VBIOS、**GPU 和主机桥两侧**完整的 LnkCap/LnkCap2/LnkCtl2/LnkSta/LnkSta2/DevCap2/DevCtl2/LnkCtl 集合、sysfs 速率与宽度、`nvidia-smi` 数据和 AER 计数器。确认卡上的观察身份：VBIOS `92.00.6D.00.0A` 和 `92.00.67.00.01`，BoardPN `900-11001-0108-000`，GPUPN `20C2-105-A1`，子系统 `0x158510DE`。

## 开放问题

> [!NOTE]
> **开放问题：Gen3 和 Gen4**
>
> `FUSE_PCIE_GEN23_DIS` 和 `FUSE_PCIE_GEN3_DIS` 都读 `0x00000001`，支持速率向量即使在 PHY 速率被强推到 Gen3 能力的 `0x00340036` 后仍被截在 `0x00000006`。"向量连续所以 Gen2/3/4 是一个问题"的论点是在这颗硅片上失败了，还是 Gen3 熔丝在下游独立执行，未决。最便宜未试的实验：通过同一个已经*尝试* `0x0082057c` 的 `xp3g` 表写 `0x00820580 = 0`，注意那个写入失败，所以预期 `booter FAILED to set` 和 `rd=0x00000001`。然后请求 TLS = 3。便宜，但先验很低。见[Gen3 和 Gen4](../frontier/pcie-gen3-gen4.md)。

> [!NOTE]
> **开放问题：`FUSE_PCIE_MAGIC_D` 可写吗？**
>
> 一份分析把 `0x00820520` 注为"(可写)"；一条净室链向它写 `0x00200000`；现场手册把它列为只读；分支补丁只读它。因为没有 Gen4 主机就无法测试 Gen4，这一直没被实际执行。读、写 `0x00200000`、读回、公布两个值。五分钟的活，没人干过。

> [!NOTE]
> **开放问题：x16 稳定吗？**
>
> 2026-07-26 的一次抓取是 Gen2 x16 的全部证据基础。没有烤机、没有随时间的 AER 计数器、没有第二台机器。

> [!NOTE]
> **开放问题：Gen2 卡上的可调整大小 BAR**
>
> 能力结构存在且已被抓取（`Capabilities: [bb0 v1] Physical Resizable BAR`，BAR0 16 MB、BAR1 64 MB、BAR3 32 MB，各一个支持大小）。开放的是 ReBAR 能否变得*可用*：即使卡报告 81920 MiB，BAR1 也钉在 64 MiB，而且没人在 Gen2 训练成功的卡上重新测试过。

## 另见

- [Gen2 软件解锁](../unlock/pcie-gen2.md)讲寄存器机制与分支代码
- [物理改造](../operations/physical-mods.md)讲电容返工步骤
- [熔丝与 OTP](fuses-and-otp.md)讲完整熔丝图
- [Gen3 和 Gen4](../frontier/pcie-gen3-gen4.md)讲未解决的一半
- [寄存器索引](../appendix/register-index.md)
- [术语表](../start/glossary.md)
