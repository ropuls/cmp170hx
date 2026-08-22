# 识别你的卡

**本页涵盖：** 如何确定你手里的到底是哪一款 CMP 170HX，从而确定适用哪个解锁配置。`lspci` 设备 ID、子系统 ID、板卡和 GPU 部件号、解锁前后 `nvidia-smi` 报告什么、散热器下面的芯片标记、区分两个 SKU 的寄存器级指纹，以及 `install.sh` 安装时运行的确切配置检测阶梯。最后是一张从你所观察到你所得到的决策表。

**关键结果，一句话。** 170HX 有两个 SKU，解锁到不同容量：**8 GB 卡，PCI ID `10de:20c2`，解锁到 64 GB**；**10 GB 卡，PCI ID `10de:2082`，解锁到 40 GB**。千万别搞混。第三个 ID `10de:20b0` 会被安装器检测到，但**不是** 170HX，也不会解锁。本页其余内容都是为这一条区别做佐证。

最快的答案：

```bash
lspci -nn | grep -i nvidia
```

如果方括号里的配对读作 `[10de:20c2]`，你有一张 8 GB 卡，将解锁到 64 GB。如果读作 `[10de:2082]`，你有一张 10 GB 卡，将解锁到 40 GB。

---

## 1. `lspci`：权威识别

两个 SKU 在同一个人类可读名称下枚举，所以单看名称什么也说明不了。方括号里的厂商:设备配对才是关键。

```bash
lspci -nn | grep -i '10de:20c2\|10de:2082\|10de:20b0'
```

8 GB 卡上的预期输出：

```text
0a:00.0 3D controller [0302]: NVIDIA Corporation GA100 [CMP 170HX] [10de:20c2] (rev a1)
```

10 GB 卡上的预期输出：

```text
81:00.0 3D controller [0302]: NVIDIA Corporation GA100 [CMP 170HX] [10de:2082] (rev a1)
```

注意类别：它是 `3D controller`，不是 `VGA compatible controller`，因为这张卡没有显示输出。`rev a1` 是 GA100 硅片版本，两个 SKU 相同。

至于子系统 ID——第二个独立确认——用详细输出：

```bash
sudo lspci -nn -vv -s 0a:00.0 | head -8
```

```text
0a:00.0 3D controller [0302]: NVIDIA Corporation GA100 [CMP 170HX] [10de:20c2] (rev a1)
        Subsystem: NVIDIA Corporation GA100 [CMP 170HX] [10de:1585]
        Control: I/O- Mem+ BusMaster+ SpecCyc- MemWINV- VGAMon- FastB2B- DisINTx+
        Capabilities: [60] Power Management version 3
```

子系统 `10de:1585` 是 8 GB 卡。子系统 `10de:1557` 是 10 GB 卡。子系统 ID 是这张卡上唯一**可由 VBIOS 设置**的标识符，所以把它当作佐证而非证据。主设备 ID 熔进芯片：`FUSE_DEVID_SW_OVR_DIS`（`0x00820584`）在每张探测过的卡上都读 1，软件无法覆盖它，跳线电阻也改不了它。

sysfs 无需 root 也能给出同样的两个值：

```bash
BDF=0000:0a:00.0
cat /sys/bus/pci/devices/$BDF/device            # 0x20c2
cat /sys/bus/pci/devices/$BDF/subsystem_device  # 0x1585
```

---

## 2. `nvidia-smi`：报告的显存和部件号

`nvidia-smi` 不报告有用的产品名。两个 SKU 都显示为 **`NVIDIA Graphics Device`**，Linux 监控工具显示一个通用的 "NVIDIA display device" 条目是正常行为。尺寸字段和部件号才有用。

```bash
nvidia-smi --query-gpu=name,memory.total,pci.device_id,pci.sub_device_id,vbios_version \
           --format=csv
```

原厂 8 GB 卡：

```text
name, memory.total [MiB], pci.device_id, pci.sub_device_id, vbios_version
NVIDIA Graphics Device, 8192 MiB, 0x20C210DE, 0x158510DE, 92.00.67.00.01
```

原厂 10 GB 卡：

```text
NVIDIA Graphics Device, 10240 MiB, 0x208210DE, 0x155710DE, 92.00.66.00.02
```

解锁成功后，同一命令在 8 GB 卡上报告 **65536 MiB**，在 10 GB 卡上报告 **40960 MiB**。输出中其他任何内容都不变。

板卡和 GPU 部件号在完整查询中：

```bash
nvidia-smi -q | grep -E 'Board Part Number|GPU Part Number|VBIOS Version|Bus Id'
```

```text
    VBIOS Version                         : 92.00.6D.00.0A
    Board Part Number                     : 900-11001-0108-000
    GPU Part Number                       : 20C2-105-A1
```

> [!CAUTION]
> **不匹配的 `nvidia-smi` 会使本页的每个读数失效**
>
> NVML 拒绝跨驱动版本对话，所以通过不匹配的 `nvidia-smi` 二进制读出的 `memory.total` 毫无意义。有一个持续多天的测量系列正是这样被作废的：用 580.159.03 用户态去跑一个不同的内核模块构建。如果 `nvidia-smi` 打印 "driver/library version mismatch"，先修好它，再相信它说的任何话。见[排障](../procedures/troubleshooting.md#version-mismatch)。

---

## 3. 物理标记

如果卡已从机器中取出，或散热器已拆下，可以读三处标记。

| 标记 | 8 GB 卡 | 10 GB 卡 |
|---|---|---|
| ASIC（芯片）标记 | `GA100-105F-A1` | `GA100-105A-A1` |
| 板卡部件号 | `900-11001-0108-000` | `900-11001-0105-000` |
| GPU 部件号 | `20C2-105-A1` | `2082-105-A1` |
| PCB 丝印，金手指上方 | `180-11001-DAAA-B15`（也见过 `180-11001-DAAA-B35`、`180-11001-DAAA-045`） | 同一板卡家族 |
| 板 ID | 未记录 | `0x8100` |

两个丝印字符串是同一板卡家族；末尾字段是版本或变体代码，两者都已在 USB 显微镜下拍过照。170HX 封装上的图例还读作 `NVIDIA / B KR 2120A1 / TBSG42.M0W e1`。作为对照，与 170HX 并排拍过照的一块零售 Tesla A100 40 GB 芯片标记为 `GA100-883AA-A1`，所以中间字段 `-105x-` 就是"这是 CMP"的标志。

> [!WARNING]
> **实验性：此处没有 10 GB 芯片标记的照片**
>
> 8 GB 卡上的 `GA100-105F-A1` 由一张拆解照片和 TechPowerUp 数据库确认，两个独立来源对变体字符串意见一致。10 GB 卡的 `GA100-105A-A1` 来自文档化的规格表，本语料中没有任何地方独立拍过照。读芯片还意味着拆散热器，所以这是本页最不实用的识别途径：用 `lspci`。

板卡和 GPU 部件号是更有用的物理标识符，因为 `nvidia-smi` 无需拆机就能报告它们，而且在两台主机上检视的全部四张 8 GB 卡上它们都相同。

---

## 4. VBIOS 版本，以及为什么它们不决定任何事

```bash
nvidia-smi --query-gpu=vbios_version --format=csv,noheader
```

| VBIOS | SKU | 构建日期 | 备注 |
|---|---|---|---|
| `92.00.67.00.01` | 8 GB（`0x20C2`，子系统 `0x1585`） | 2021-05-14 | 原厂量产镜像，显存字段 364 MHz，250 W |
| `92.00.6D.00.0A` | 8 GB | 2022-04-07 | 300 W "OC 挖矿"镜像，显存字段 432 MHz，允许核心时钟偏移 |
| `92.00.6D.00.09` | 8 GB | 2021-11-01 | 带 300 W 上限但不带显存超频。不在 TechPowerUp 收藏中。*（置信度：中高；一位研究者持有该文件。）* |
| `92.00.66.00.02` | 10 GB（`0x2082`，子系统 `0x1557`） | 2021-04-23 | 唯一的 10 GB 镜像 |

**VBIOS 版本对解锁是否生效没有任何影响。** 这是一位核心研究者直接断言的，并由一次四卡双主机对比独立佐证：两张卡在 `92.00.67.00.01` 上、两张在 `92.00.6D.00.0A` 上，产生了完全相同的解锁和 Gen2 结果。完整镜像清单见 [VBIOS](../hardware/vbios.md)，包括哪些流传镜像被贴错标签、哪些绝不能刷。

---

## 5. 寄存器级指纹

这些需要 BAR0 访问，用于确认含糊情况或交叉核对探测转储，不做日常识别。

| 寄存器 | 8 GB（`0x20C2`） | 10 GB（`0x2082`） |
|---|---|---|
| `PMC_BOOT_0` | `0x170000a1` | `0x170000a1`（所有 GA100） |
| `FUSE_PCIE_DEVIDA` `0x008204d8` | `0x000020c2` | `0x00002082` |
| `FUSE_PCIE_DEVIDB` `0x0082056c` | **有争议**：2026-07-19 对一块 `0x20c2` 卡的一次探测读 `0x000020c2`；`DEVIDB = DEVIDA + 0x40` 规则预测 `0x00002102`（见[板卡与变体](../hardware/board-and-variants.md)） | `0x000020c2` |
| `FUSE_SKU_ID` `0x00821060` | `0x80` | `0x68` |
| `OPT_GPC_DISABLE` `0x00820350` | **逐芯片而非逐 SKU：不要用于识别** | **逐芯片而非逐 SKU：不要用于识别** |
| `NV_PTOP_FS4` `0x0002241c` | `0x00000000` | `0x00000081` |
| 原厂 CFG1 `0x009a0204` | `0x02449000` | `0x02449000`（两者相同） |
| 原厂 LMR `0x00100ce0` | `0x00000208` | `0x00000288` |
| 原厂每 FBPA `CSTATUS_RAMAMOUNT` | `0x200`（512 MiB） | `0x200`（512 MiB） |
| HBM `MRS_2` `0x009a0334` | `0x00200019` | `0x002000cf` |
| HBM `MRS_WL_RL` `0x009a0338` | `0x003000eb` | `0x003000ea` |
| `FBPA_HBM_CFG0` `0x009a038c` | `0x000000a7` | `0x000000a7` |

注意**原厂 CFG1 在两个 SKU 上都是 `0x02449000`**，所以单靠 CFG1 无法判断你手里是哪张卡；LMR 可以。`NV_PTOP_FS4` 是最干净的单寄存器分水岭，bit 0 是 `GEN2_PCIE`、bit 7 是 `GEN2_PCIE_SPEED`，这使得 8 GB 卡读 `0x00000000` 成为更有意思的那一半。

> [!WARNING]
> **`OPT_GPC_DISABLE` 不是 SKU 指纹**
>
> GPC 熔断筛选掩码逐芯片而异，而非逐 SKU。在两个 SKU 的多张 170HX 卡上观察到的值包括 `0x13`、`0x15`、`0x23`、`0x25`、`0x45`、`0x85`、`0xa8` 和 `0xd0`，而它们全都仍然枚举 70 个 SM。绝不要硬编码筛选值，也不要从筛选值推断 SKU。改用 `FUSE_SKU_ID` `0x00821060`（8 GB 卡上为 `0x80`，10 GB 卡上为 `0x68`）。

由 SKU 决定的架构差异：

| 属性 | 8 GB 卡 | 10 GB 卡 |
|---|---|---|
| 总线宽度 | 4096 位 | 5120 位 |
| HBM 堆栈 | **未解决。** 一块开盖的 8 GB 卡可见六个堆栈；一份芯片照片来源称六个中有两个是空壳。两种读数都与实测总线宽度兼容，所以总线宽度无法定案 | 报告 5 个 |
| 活动 FBPA | 24 个中的 16 个（8 个 FBP） | 24 个中的 20 个（10 个 FBP） |
| 每 FBPA 容量，原厂 | 512 MiB（`_CSTATUS_RAMAMOUNT` = `0x200`，CFG1 层级 `0x44`） | 512 MiB（相同） |
| 每 FBPA 容量，解锁后 | 4096 MiB（`0x1000`，层级 `0x77`） | 2048 MiB（`0x800`，层级 `0x66`） |
| 显存时钟 | **未解决**，见下方框 | 1215 MHz，当前值等于最大值，无余量 |
| 解锁到 | **64 GB** | **40 GB** |

> [!NOTE]
> **开放问题：原厂 8 GB 显存时钟**
>
> 原厂 8 GB 显存时钟未解决：1458 MHz（一次扫描和 TechPowerUp）、1728 MHz（`nvidia-smi -q` 的 Supported Clocks，注为 "432 MHz × 4"）、1890 MHz（解锁 64 GB 的 `gpu_burn` 在 300 W 下运行时的 `nvtop`）。1215 MHz 是 10 GB 卡的，可靠。看似合理的整合——原厂 1458 / OC VBIOS 1728 / 超频 OC VBIOS 1890——尚未被证实，且 1728 MHz 这个数值被单独归因于 POST 时的 FWSEC devinit，而非 OC VBIOS，因为八份转储的 ROM 中任何一份都没有 Memory Clock Table。一次原始 FBPA PLL 读取即可定案。

哪些具体 FBPA 被筛选逐卡而异。一份 10 GB 转储读 `FBP_DEFECTIVE` = `0x840`（FBP6、FBP11），与 A100 PCIe 40/80 GB 部件同模式，但该读数背后的卡身份有争议（一份来源把它归于 8 GB 卡），且另一份 10 GB 探测读 `OPT_FBP_DISABLE` = `0x00000009`（FBP 0 和 3）。中等置信度，单份转储。10 GB 卡上每次干净的解锁点火都报告 `CSTATUS=20/24`。
见[显存子系统](../hardware/memory-subsystem.md)和[熔丝与 OTP](../hardware/fuses-and-otp.md)。

---

## 6. 第三个设备 ID：`10de:20b0`

`install.sh` 搜索三个 ID：

```bash
lspci -nn | grep -iE '10de:20b0|10de:20c2|10de:2082' | head -1
```

但驱动内闸门 `_kgspSec2PostblTimingEnabled()` 只接受 **`0x20C2` 和 `0x2082`**。因此 `20b0` 卡会干净地安装但永不解锁。安装器会这样说并继续：

```text
! In-driver unlock path is gated on PCI ID 0x20C2 / 0x2082.
! This card reports 0x20b0; install will continue, but unlock may not activate.
```

`0x20b0` 是 A100 SXM4 40 GB 设备 ID，也被一块 A100 工程样品（8192 MB、2048 位、4096 个 CUDA 核心、三星 8Hi HBM2）携带。README 里较老的"解锁受 `0x20C2` 门控"措辞已过时：自 "Unlock isn't gated anymore" 提交起，`0x2082` 就是一等目标。

两条相关说明：

* 补丁 0001 和 0002 中的每条 `SEC2_DEBUG` 打印都门控在同样的两个设备 ID 上，所以一块 `20b0` 卡上的原厂构建应该在 `dmesg` 中**什么都不打印**。一份关于 `20b0` 工程样品上出现 SEC2_DEBUG 行的报告未解决，最可能是修改过的构建或同一主机里的第二张卡。
* `0x20BB` 是 Drive A100 / PG199 部件（原厂 32768 MiB）。名为 `PG199` 的分支什么都没加：它的代码树与 `ecc` 分支逐字节相同，提交列表为空，其中任何地方都不提 PG199、`0x20BB` 或 A100D。它不加任何检测、不含任何 A100D 支持，也不改 `lspci` 搜索或驱动内闸门。不存在 PG199 解锁。

见[排障](../procedures/troubleshooting.md#device-id-20b0)。

---

## 7. 安装时的配置检测阶梯

`install.sh` 在 6 步中的第 3 步选择配置。要么你传 `--profile`，要么 `detect_card_profile()` 读取报告的显存并分档：

```bash
nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits | head -1
```

| 报告的 `memory.total` | 选中的配置 | 窗口为何存在 |
|---|---|---|
| `>= 60000` MiB | `8gb` | 在**已解锁**的 64 GB 卡上重装 |
| `35000`–`59999` MiB | `10gb` | 在已解锁的 40 GB 卡上重装 |
| `7680`–`8704` MiB | `8gb` | 原厂 8 GB 卡（8192 MiB） |
| `9728`–`10752` MiB | `10gb` | 原厂 10 GB 卡（10240 MiB） |
| 其他任何值 | **致命** | 打印 `unknown:<mib>`，然后 `Could not detect 8GB vs 10GB card. Re-run with --profile=8gb or --profile=10gb` |

然后横幅打印以下之一：

```text
==> Unlock geometry: 64GB (CFG1=0x02779000 LMR=0x0000020B)
==> Unlock geometry: 40GB (CFG1=0x02669000 LMR=0x0000028A)
```

> [!WARNING]
> **混合 GPU 主机上自动检测不安全**
>
> `detect_card_profile()` 取 `nvidia-smi` 顺序中的**第一块 GPU**，它不一定是 `lspci` 找到的那块 CMP。至少两个人复现过：一台带 RTX 3080 10 GB 和一块 8 GB CMP 170HX 的主机从 3080 检测出 "10GB"；另一份报告里一块 CMP 50HX 被误检为 10 GB 170HX。如果第一块 GPU 报告的尺寸在所有四个窗口之外——例如一块 24 GB 卡——安装直接死掉。
>
> 任何带不止一块 NVIDIA 卡的主机都务必显式传 `--profile`：
>
> ```bash
> sudo ./install.sh --profile=8gb     # 8 GB 物理卡  -> 64 GB
> sudo ./install.sh --profile=10gb    # 10 GB 物理卡 -> 40 GB
> ```

安装后，记录的配置可以这样读：

```bash
cat /lib/modules/$(uname -r)/updates/cmpunlocker/card_profile      # 8gb or 10gb
cat /lib/modules/$(uname -r)/updates/cmpunlocker/unlock_geometry   # 64GB or 40GB
cat /lib/modules/$(uname -r)/updates/cmpunlocker/driver_version
```

内核模块里没有任何东西读这三个文件。它们为人类和 `verify.sh` 存在，后者把 `20c2 -> 8gb -> 65536 MiB`、`2082 -> 10gb -> 40960 MiB` 映射起来。注意 `verify.sh` **不随 `master` 发布**：它只存在于 `multiple-cards`、`Gen2`、`far` 和 `deced` 分支上，而且它从 `lspci` 推导那个映射，读 `card_profile` 和 `unlock_geometry` 只是为了打印它们。

---

## 8. 决策表

| 你观察到的 | 卡 | 配置 | 解锁后 `nvidia-smi` | 写入的 CFG1 / LMR |
|---|---|---|---|---|
| `[10de:20c2]`，子系统 `10de:1585`，8192 MiB | 8 GB CMP 170HX | `8gb` | **65536 MiB** | `0x02779000` / `0x0000020B` |
| `[10de:2082]`，子系统 `10de:1557`，10240 MiB | 10 GB CMP 170HX | `10gb` | **40960 MiB** | `0x02669000` / `0x0000028A` |
| `[10de:20c2]`，已报告 65536 MiB | 8 GB，已解锁 | `8gb` | 不变 | 每次启动重新应用 |
| `[10de:2082]`，已报告 40960 MiB | 10 GB，已解锁 | `10gb` | 不变 | 每次启动重新应用 |
| `[10de:20b0]` | A100 SXM4 40 GB 或 A100 工程样品 | 安装、警告 | **无变化** | 无；驱动内闸门拒绝 |
| `[10de:20bb]`，32768 MiB | Drive A100 / PG199 | 根本检测不到 | **无变化** | 无；不存在 PG199 解锁 |
| 任何其他 `10de:` ID | 不是 170HX | 安装器退出 | n/a | n/a |
| `[10de:2082]` 强制使用 80 GB 几何 | 10 GB，超配 | 存档 `80` 分支 | 报告 81920 MiB | `0x02779000` / `0x0000028A` |

> [!CAUTION]
> **80 GB 一行不是受支持的选项**
>
> 存档的 `80` 分支把 10 GB 卡编程为报告 81920 MiB，但该卡约 40 GB 以上不可用：几分钟内出现卡死、Xid 154 和烧机错误。每个提到它的来源都把它呈现为不稳定或已否决，且它与功耗上限无关。它实际编程的是一个三处不一致的组合（CFG1 `0x02779000` 配 LMR `0x0000028A` 和 80 GiB `fb_length`），这本身很可能就是原因。见 [80 GB](../frontier/80gb.md)。

---

## 9. 如果还是不确定

跑全部三项检查并比对。它们应当一致；如果不一致，`lspci` 设备 ID 胜出，因为它熔进芯片，VBIOS 或跳线都改不了。

```bash
# 1. 熔断设备 ID：权威
lspci -nn | grep -iE '10de:(20b0|20c2|2082)'

# 2. 子系统 ID 和部件号：佐证（子系统可由 VBIOS 设置）
nvidia-smi --query-gpu=pci.sub_device_id,vbios_version --format=csv,noheader
nvidia-smi -q | grep -E 'Board Part Number|GPU Part Number'

# 3. 报告的尺寸：安装器自动检测会看到什么
nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits
```

一张能在未打补丁的原厂驱动上枚举并运行、报告 `NVIDIA Graphics Device` 和计算能力 8.0 的卡，就是健康的 170HX，无论它看起来多脏。退役矿卡带着厚厚的灰尘、锈蚀的挡板和散热器里的盐结晶而来，而外观状况从未预测过解锁失败：一张肉眼可见很脏的卡第一次尝试就干净地解锁到 64 GB。

---

## 相关页面

* [这是什么卡](what-is-this-card.md)：入门级概览
* [风险](risks.md)：开始前阅读
* [快速入门](quick-start.md)和[安装](../procedures/install.md)
* [验证](../procedures/verify.md)：确认解锁真的落地
* [多卡](../procedures/multi-gpu.md)：为什么安装器里的 `head -1` 在矿机上很重要
* [板卡与变体](../hardware/board-and-variants.md)：物理板卡详解
* [VBIOS](../hardware/vbios.md)：每个已知镜像及其各自改动
* [显存几何](../unlock/memory-geometry.md)：CFG1 和 LMR 实际做什么
* [熔丝与 OTP](../hardware/fuses-and-otp.md)：跨变体熔丝表
* [术语表](glossary.md)
