# PCIe Gen2 软件解锁

**本页涵盖：** 让 CMP 170HX 从 PCIe 第 1 代（2.5 GT/s）到第 2 代（5.0 GT/s）且无需硬件改装的完整寄存器机制，它如何在补丁 `0007` 和 `0008` 之间拆分、retrain 流程、IOMMU 依赖、modprobe 注册表键，以及合并前这些工作的分支谱系。它击败的硬件见 [PCIe 子系统](../hardware/pcie-subsystem.md)。

> [!NOTE]
> **已发布：Gen2 于 2026-07-29 合并到 `master`**
>
> Gen2 在第一个星期仅限分支。它在提交 `2e0a2c02`（"PCIe Gen 2 unlock!"）合并，`master` 现在在 `0001` 到 `0006` 之外携带 `0007-pcie-gen2.patch` 和 `0008-pcie-gen2-probe-retrain.patch`。master 的 README 列出 `PCIe Gen 2 speeds | Working`。多台独立机器已复现。
>
> 仅分支时期的两点细节仍然成立、值得知道：`common/constants.yaml` 仍然没有 `pcie:` 块，因此寄存器数据在补丁里而不是配置里；一位贡献者把该方法形容为"像脚本猛灌东西并希望它粘住"，这对一个步骤间无回读的 25 写序列是公允描述。它能工作，而且不优雅。

## 结果，放在最前

Gen2 使链路**速率**翻倍。它不碰链路**宽度**，任何分支中的代码都不读或写宽度字段。未焊接的 Gen2 补丁卡以 Gen2 x4 运行。

| | 原厂，无解锁 | 带解锁器 |
|---|---|---|
| `LnkCap` | `0x00456101`（最大 Gen1，x16） | `0x00456102`（最大 Gen2，x16） |
| `LnkCap2` 支持的速率 | `0x00000002`（仅 2.5 GT/s） | `0x00000006`（2.5 和 5.0 GT/s） |
| `LnkCtl2` 目标 | `0x0000` | `0x0002` |
| `LnkSta` | `0x1041`（Gen1，x4） | `0x1042`（Gen2，x4） |
| `LnkSta2` | `0x0000` | `0x0001` 或 `0x0000`，随机器 |
| `DevCap2` / `DevCtl2` | `0x00070803` / `0x1400` | `0x00070813` / `0x0400`（一台机器 `0x7410`） |
| `nvidia-smi` cur / max / width | 1, 1, 4 | 2, 2, 4 |
| 主机带宽 | 约 0.85 GB/s | 约 1.71 GB/s |
| AER correctable / nonfatal / fatal | 0 / 0 / 0 | 0 / 0 / 0 |

三台独立机器发布了 Gen2 状态的完整机器生成转储：驱动 610.43.02 配内核 5.15.0-186-generic、驱动 610.43.03 配内核 6.12.54-Unraid（双卡），以及驱动 610.43.03 配内核 6.12.0-hiveos（双卡）。

## 代码在哪里

| 分支 | `0007-pcie-gen2.patch` | `0008-pcie-gen2-probe-retrain.patch` | `tools/retrain.sh` | IOMMU 处理 | `RMPcieLinkSpeed` |
|---|---|---|---|---|---|
| `master`（正式） | 无 | 无 | 无 | 无 | 无 |
| `debug-gen2` | 有（hunk 头畸形） | 无 | 安装，自动发现 BDF，加 `cmpretrain.service` | 无 | `0x1` |
| `Gen2` | 有（头已修复） | 有 | 存在但从未安装，BDF 硬编码 | 自动 | `0x1` |
| `far` | 有 | 有 | 存在但从未安装，BDF 硬编码 | 自动 | `0x2` |
| `deced` | 有 | 有 | 存在但从未安装，再次自动发现 BDF | 自动 | `0x2` |

### 分支谱系

```text
# dates in committer-local time (-0700)
6621ffc  Effort on PCIe Gen 2                               2026-07-22
4bd6d4d  Fixed malformed patch                              2026-07-22
a9b2470  Delete requirements.txt
746d9f7  PCIe Gen 2 works!                                  2026-07-23   <- tip of debug-gen2
0901346  Fix malformed 0007-pcie-gen2 hunk line counts      2026-07-24
d88af88  Potential fix                                      2026-07-24
146da6f  Correct retraining                                 2026-07-24
2f27474  Gen2 + multiple-card support                       2026-07-24
7ea2c4f / 1605219 / bed923f / a14176b / e95784c
6a85e6c  IOMMU enablement as part of install script         2026-07-24
a4de322  (merge)                                            2026-07-26   <- tip of Gen2
8854d3e  Remove clamp link to Gen1                          2026-07-26   <- tip of far
2326599  Stupid mistake - it appears to be hardcoded        2026-07-27   <- tip of deced
```

"通告 Gen2 但不 retrain"（`Effort on PCIe Gen 2`，2026-07-22 22:02:43 -0700，即 2026-07-23 05:02 UTC）与"训练成功"（`PCIe Gen 2 works!`，2026-07-23 18:21:35 -0700，即 2026-07-24 01:21 UTC）之间相隔约二十小时。结果于 2026-07-24 00:59 公开宣布，数小时内被多位在不同硬件上的独立测试者复现。

`far` 是 `Gen2` 加恰好一个提交，其在整个树中的唯一内容变化是一行中的一个字符。`deced` 是 `far` 加一个提交，把 BDF 自动发现重新加回安装器删除的脚本。

## 机制

解锁分三个阶段。阶段 A 需要 SEC2 Booter 特权；阶段 B 只需要一道打开的 PLM；阶段 C 需要上游桥，完全无法在驱动 GSP 钩子内部完成。

```text
Phase A  25 Booter-routed writes   (0007, kernel_gsp.c)      opens PLMs, sets XP3G overrides
Phase B   6 plain BAR0 writes      (0007, in-GSP hunk)       clears DIS_G2, sets MAX_RATE
Phase C   root-port retrain        (0008 / retrain.sh / hammer)  actually changes the link speed
```

### 注入点

补丁 `0007` 把整个寄存器块注入 `src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c` 的 `@@ -4942,6 +4942,260 @@`，紧跟现有 `devId` 打印之后、`plmStatus = kgspSec2PostblTimingRebuildStockSignature(pGpu, pKernelGsp);` **之前**。因此它运行在 SEC2 post-bootloader 解锁窗口内，PLM 仍然打开、构造的 Falcon 签名载荷仍通过 Booter Load 提供任意 BAR0 写入原语。这与显存和计算解锁使用的权限提升相同；见 [Falcon 与 Booter](falcon-and-booter.md) 和 [工作原理](how-it-works.md)。

### 阶段 A：23 条目 `xp3gTable`

每个条目是一对经 Booter 载荷原语推入的 `{address, value}`。每个条目代码恢复 WPR2 低/高位（`GPU_REG_WR32(pGpu, 0x001fa824U, wpr2Lo)` 和 `0x001fa828U, wpr2Hi`）、调用 `kgspSec2PostblTimingRefillPayload(pGpu, pKernelGsp, addr, value)`、调用 `kgspExecuteBooterLoad_HAL(...)`、回读目标并重试一次（`xattempt < 2`），失败打印 `SEC2_DEBUG: PCIe xp3g booter FAILED to set <name>`。

| # | 地址 | 名称 | 值 | 目的 |
|---|---|---|---|---|
| 1 | `0x0008e1b0` | `XP3G_PLM` | `0xffffffff` | 打开 PLM |
| 2 | `0x0008e1b4` | `XP3G_PLM4` | `0xffffffff` | 打开 PLM |
| 3 | `0x0008e1b8` | `XP3G_PLM8` | `0xffffffff` | 打开 PLM |
| 4 | `0x0008e1bc` | `XP3G_PLMC` | `0xffffffff` | 打开 PLM |
| 5 | `0x00088fe8` | `XVE_D0` | `0xffffffff` | 打开 PLM |
| 6 | `0x00088fec` | `XVE_D4` | `0xffffffff` | 打开 PLM |
| 7 | `0x00088ff0` | `XVE_D8` | `0xffffffff` | 打开 PLM |
| 8-17 | `0x008200d0`、`d4`、`d8`、`dc`、`e0`、`e4`、`e8`、`ec`、`f0`、`f4` | `OPTB_D0` .. `OPTB_F4`（**10** 个寄存器） | `0xffffffff` | 打开 PLM |
| 18 | `0x00823800` | `FEAT_OVR_ECC_PLM` | `0xffffffff` | 打开 PLM |
| 19 | `0x0082057c` | `OPT_GEN23` | `0x00000000` | 值写入，**总是失败** |
| 20 | `0x0008e120` | `XP3G_VAL0` | `0x00000000` | 值写入 |
| 21 | `0x0008e110` | `XP3G_OVR0` | `0x00000001` | 使能，槽 0 |
| 22 | `0x0008e12c` | `XP3G_VAL3` | `0x00200000` | 值写入，导出为 `opt_magic_a100` |
| 23 | `0x0008e11c` | `XP3G_OVR3` | `0x00000004` | 使能，槽 3 |

十八次 PLM 打开加五次值写入。还有两个寄存器在表**外**得到同样两次尝试的 Booter 处理，共 **25 次经 Booter 路由的写入**：

| 地址 | 名称 | 操作 | 硅片上的结果 |
|---|---|---|---|
| `0x0008860c` | `VSEC_DEVICE` | 设置位 0（`|= 1 << 0`） | **失败**：`pre=0x00000800 want=0x00000801`，回读两次 `0x00000800`，然后 `PCIe VSEC_DEVICE booter FAILED` |
| `0x0008841c` | `PRIV_MISC_1` | 设置位 11 和 13，清除位 12 和 14 | **首次尝试成功**：`0x20340500` 变为 `0x20342d00`，真正 BooterLoad 后仍读 `0x20342d00` |

XP3G 覆盖块是三个平行的四 dword 数组：状态基址 `0x0008e100`、覆盖使能基址 `0x0008e110`、值基址 `0x0008e120`，槽 *n* 在基址 + 4*n*，因此槽 3 是基址 + `0xC`。值总是在使能之前写，覆盖从不闩锁陈旧数据，使能编码每槽单热（槽 0 为 `0x1`，槽 3 为 `0x4`）。

> [!NOTE]
> **两个条目失败，Gen2 仍然工作**
>
> `OPT_GEN23` 是纯 OTP 熔丝感知反射，没有写端口；任何权限级的每次尝试都返回 `status=0xffff rd=0x00000001`。`VSEC_DEVICE` 也失败。发布补丁仍然尝试两者、在两者上都失败，链路仍然以 Gen2 训练。可工作的杠杆是 `CYA_0`、`LINK_CONFIG_0`、XP3G 覆盖和 `PRIV_MISC_1`。

> [!NOTE]
> **二手文档中的计数错误**
>
> 三个已发布计数是错的，可直接对照补丁核实：OPTB 运行是**十**个寄存器（`D0, D4, D8, DC, E0, E4, E8, EC, F0, F4`），不是十一也不是九；表在 **23** 条目表内打开 **18** 道 PLM，不是 22；晚期 hunk 位于 `kernel_gsp_tu102.c`、紧跟 Booter Load 返回之后，即 GSP-RM 运行**之前**，不是之后。

### 阶段 B：普通 BAR0 写入

XVE PLM `0x00088fe8` / `fec` / `ff0` 打开后，普通 `GPU_REG_WR32` 写入落盘。在那之前，priv ring 丢弃它们；对 `0x08c044` 的探测返回 priv 掩码哨兵 `0xbadf5040` 并被跳过，而 `0x0880a8` 干净写读。

| 寄存器 | 地址 | 操作 | 备注 |
|---|---|---|---|
| `VSEC_HIERARCHY` | `0x00088610` | `hier = (hier & ~(1U << 12)) | (1U << 0)` | 位 12 门控 `PRIV_MISC_1` 重新编程；修改前活值 `0x00001001` |
| `LINK_CTRL_2` | `0x000880a8` | `lc2 = (lc2 & ~0xFU) | 0x2`，然后 `lc2 = (lc2 & ~0x000F0000U) | 0x000F0000U` | `[3:0]` 中 Target Link Speed = 2，`[19:16]` 中 `0xF` |
| `CYA_0` | `0x0008c2c0` | `cya0 = cya0 & ~(1U << 2)` | 清除 `DIS_G2` 鸡位 |
| `LINK_CONFIG_0` | `0x0008c040` | `linkCfg = (linkCfg & ~0x000C0000U) | (0x2U << 18)` | `MAX_RATE` 字段 `[19:18]` 设为 2（5.0 GT/s）。CMP 原厂读 `0x800C4C00`，SPEED = 3 |
| `PL_LINK_RATE` | `0x0008c1c0` | `= 0x00240036` | 见下方注意事项 |
| LTSSM / `XVE_OVR` | `0x0008872c` | `= 0x00000006` | 日志行：`SEC2_DEBUG: PCIe XVE_OVR@8872c=0x%08x; skip mid-boot retrain` |

`CYA_0` 位 2 在**四个**独立位置清除：`kernel_gsp.c` 的 GSP 内位置、`kernel_gsp_tu102.c` 的晚期位置、`tools/retrain.sh`，以及补丁 `0008` 的 `nv_cmp170hx_retrain_gen2()`。`retrain.sh` 把仍设置的位 2 视为硬中止（`retrain: DIS_G2 still set; skip`）。

`PRIV_MISC_1` 是配对的使能和值 CYA 覆盖。补丁宏是 `PCIE_GEN2_PRIV_MISC_1_GEN2_EN = ((1U << 11) | (1U << 13))` 和 `PCIE_GEN2_PRIV_MISC_1_GEN2_VAL = ((1U << 12) | (1U << 14))`，请求值 `(misc1 | GEN2_EN) & ~GEN2_VAL`：断言两个覆盖使能，把两个值位驱动到零。

> [!NOTE]
> **`PL_LINK_RATE 0x00240036` 不是必需的**
>
> 该写入只存在于 `0007` 的 GSP 内路径。`tools/retrain.sh` 和补丁 `0008` 都不碰 `0x0008c1c0`，而两者都产出 Gen2。A100 强制代际扫描还显示整个 XP_PL 家族（`0x8C044`、`0x8C048`、`0x8C04C`）在参考卡上每个代际都读 `0xbadf5040`，因此该家族从未对照可工作链路验证过。`0x00240036` 的各个位编码什么没有文档。命名注意：地址被 `#define` 为 `PCIE_GEN2_LTSSM_ADDR` 对应 `0x0008872c`，而日志字符串叫它 `XVE_OVR`；这种歧义在源码本身。

### 晚期 hunk

补丁 `0007` 在 `src/nvidia/src/kernel/gpu/gsp/arch/turing/kernel_gsp_tu102.c` 的 `@@ -611,6 +611,44 @@` 有第二个 hunk。它以**不**经 Booter 的普通 BAR0 访问重新应用四个写入：`PRIV_MISC_1`、清除 `CYA_0` 位 2、`LINK_CONFIG_0` `MAX_RATE = 2`，以及 `0x0008872c = 6`。其日志行带 `late` 后缀。它按以下门控：

```c
NvU32 lateDevId = pGpu->idInfo.PCIDeviceID >> 16;
if ((lateDevId == 0x20C2 || lateDevId == 0x2082) && status == NV_OK)
```

因此 `10de:20b0` 卡完全得不到 Gen2 处理，与项目其余部分的设备 ID 处理一致。见 [识别你的卡](../start/identify-your-card.md)。

### `0007` 定义的命名地址

```c
#define PCIE_GEN2_LINK_CAP_ADDR        0x00088084U
#define PCIE_GEN2_LINK_CAP2_ADDR       0x000880a4U
#define PCIE_GEN2_LINK_CTRL_2_ADDR     0x000880a8U
#define PCIE_GEN2_LINK_CTRL_STATUS_ADDR 0x00088088U
#define PCIE_GEN2_PL_LINK_RATE_ADDR    0x0008c1c0U
#define PCIE_GEN2_LTSSM_ADDR           0x0008872cU
#define PCIE_GEN2_VSEC_DEVICE_ADDR     0x0008860cU
#define PCIE_GEN2_VSEC_HIERARCHY_ADDR  0x00088610U
#define PCIE_GEN2_XP3G_OVR_BASE        0x0008e110U
#define PCIE_GEN2_XP3G_VAL_BASE        0x0008e120U
#define PCIE_GEN2_XP3G_STATUS_BASE     0x0008e100U
#define PCIE_GEN2_OPT_GEN23_ADDR       0x0082057cU
#define PCIE_GEN2_OPT_GEN3_ADDR        0x00820580U
#define PCIE_GEN2_OPT_MAGIC_ADDR       0x00820520U
#define PCIE_GEN2_PRIV_MISC_1_ADDR     0x0008841cU
#define PCIE_LINK_SPEED_OF(stat)       (((stat) >> 16) & 0xFU)
```

其中两个值是 `const NvU32` 声明而不是 `#define`：

```c
const NvU32 PCIE_GEN2_LINK_SPEED = 0x00000002U;
const NvU32 PCIE_GEN2_PL_LINK_RATE_VALUE = 0x00240036U;
```

`OPT_GEN3` 和 `OPT_MAGIC` 被读取并记录（在 `NV_PRINTF` 参数列表内，格式 `OPT=%08x/%08x/%08x` 对应 GEN23 / GEN3 / MAGIC），但**从不写入**。代码尝试写的唯一熔丝选项寄存器是 `OPT_GEN23`，而那次写入失败。任何代码路径都不会请求高于 2 的目标链路速率。

### PLM 表从四项增长到九项

正式 master 武装四个 PLM 条目。Gen2 家族分支（`Gen2`、`debug-gen2`、`far`、`deced`，四个在这点上逐字节相同）在 `0001-sec2-postbl-plm-ss-cfg.patch` 上再加五个，共九个：

| 索引 | 地址 | 名称 | 目标值 | 在正式 master 上？ |
|---|---|---|---|---|
| 0 | `0x001fa7cc` | `WPR_CFG` | `0xfffff0ff` | 是 |
| 1 | `0x009a0148` | `FBPA` | `0xffffffff` | 是 |
| 2 | `0x001fa7c4` | `WPR` | `0xffffffff` | 是 |
| 3 | `0x00823804` | `FEAT` | `0xffffffff` | 是 |
| 4 | `0x00088ff4` | `XVE` | `0xffffffff` | 仅 Gen2 家族 |
| 5 | `0x00088ab4` | `XVE_B` | `0xffffffff` | 仅 Gen2 家族 |
| 6 | `0x00088ff8` | `XVE_C` | `0xffffffff` | 仅 Gen2 家族 |
| 7 | `0x00823b00` | `FEAT2` | `0xffffffff` | 仅 Gen2 家族 |
| 8 | `0x008200fc` | `OPT_PLM`（净室工具也称 `FUSE_SS_PLM`） | `0xffffffff` | 仅 Gen2 家族 |

每个条目最多两次尝试，每次尝试前重新武装 WPR2 低/高位。除 `WPR_CFG` 为 `0xfffff0ff`（正确的例外）外，全部九个回读 `0xffffffff`。完整细节见 [权限级掩码](privilege-level-masks.md)。

### `constants.yaml`

Gen2 分支添加一个 `pcie:` 块，恰好这些键：

```yaml
pcie:
  target_gen: 2
  link_speed_gen2: "0x2"
  xve_link_control_status: "0x00088088"
  xve_link_control_2: "0x000880a8"
  pl_link_rate_addr: "0x0008c1c0"
  pl_link_rate_value: "0x00240036"
  vsec_hierarchy_addr: "0x00088610"
  vsec_device_addr: "0x0008860c"
  xp_fuse_override_base: "0x0008e110"
  xp_fuse_override_val_base: "0x0008e120"
  opt_gen23_addr: "0x0082057c"
  opt_magic_a100: "0x00200000"
```

机制核心的五个寄存器**不在** yaml 中：`CYA_0` `0x0008c2c0`、`LINK_CONFIG_0` `0x0008c040`、`PRIV_MISC_1` `0x0008841c`、`LINK_CAP` `0x00088084` 和 `0x0008872c`。同一提交还删除了 8gb 和 10gb 配置块中的 `comment:` 行。正式 master 完全没有 `pcie:` 块。

## 补丁 0007 对比补丁 0008

它们是**互补的，不是替代**。

| | `0007-pcie-gen2.patch` | `0008-pcie-gen2-probe-retrain.patch` |
|---|---|---|
| 触及文件 | `kernel_gsp.c`、`kernel_gsp_tu102.c` | `kernel-open/nvidia/nv.c` |
| 所需特权 | SEC2 Booter（写 PLM 保护寄存器） | 除三个已解锁 BAR0 寄存器和标准 PCIe 能力访问外无需任何 |
| 达成什么 | 把 `LINK_CAP` / `LinkCap2` 提高到 Gen2 | 从上游桥触发实际链路 retrain |
| 不能做什么 | Retrain（明确拒绝："skip mid-boot retrain"） | 单独提高 `LINK_CAP` |
| 分支 | `debug-gen2`、`Gen2`、`far`、`deced` | `Gen2`、`far`、`deced` |
| Hunk 大小 | 260 行（254 增）和 44 行（38 增） | 增加 include 加一个函数和一个调用点 |

`driver/build.sh` 按文件名顺序应用：

```bash
patches=("${PATCH_DIR}"/*.patch)
for p in "${patches[@]}"; do patch -p1 < "${p}"; done
```

### 补丁 0008 详情

`nv_cmp170hx_retrain_gen2()` 被加到 `kernel-open/nvidia/nv.c`，连同 include `<linux/delay.h>`、`<linux/io.h>`、`<linux/pci.h>` 和 `<uapi/linux/pci_regs.h>`。调用插在 `nv->flags |= NV_FLAG_PERSISTENT_SW_STATE;` 之后、`(void)rm_get_gpu_uuid_raw(sp, nv);` 之前。

```text
return unless gpu->device is 0x20c2 or 0x2082
pci_upstream_bridge(gpu)                 -> bail if NULL
ioremap(pci_resource_start(gpu, 0), 0x90000)   /* 576 KiB, just enough to reach 0x8c2c0 */
clear CYA_0 bit 2            at BAR0 0x8c2c0
set MAX_RATE = 2             at BAR0 0x8c040   ((v & ~0x000c0000) | (2 << 18))
write 0x00000006             to  BAR0 0x8872c, read back to flush posted writes
iounmap
msleep(50)
set PCI_EXP_LNKCTL2_TLS_5_0GT on BOTH the GPU and the upstream bridge
set PCI_EXP_LNKCTL_RL         on the UPSTREAM BRIDGE
poll LnkSta 20 times at msleep(100)      /* 2.05 s worst case */
```

> [!CAUTION]
> **0008 的成功测试在这张卡上永远不会通过**
>
> 谓词是：
>
> ```c
> if (!ret && (link_status & PCI_EXP_LNKSTA_DLLLA) &&
>     ((link_status & PCI_EXP_LNKSTA_CLS) >= PCI_EXP_LNKSTA_CLS_5_0GB))
> ```
>
> `PCI_EXP_LNKSTA_DLLLA` 是位 13（`0x2000`）。Gen2 训练的 170HX 读 `LnkSta = 0x1042`，而 `0x1042 & 0x2000 = 0`，因此即使 `0x1042 & 0xF = 2` 表示 5.0 GT/s，谓词也失败。该位在这条端口上**永远**无法设置：DLL Link Active Reporting Capable 是 `LnkCap` 位 20，而 GPU 的 `LnkCap = 0x00456102` 位 20 是清除的。上游根端口确实报告它（`LnkCap 0x007b7905`、`LnkSta 0x7042`），但 `0008` 从**GPU** 读 `LnkSta`。
>
> 后果：在每台可工作的 Gen2 170HX 上，补丁 `0008` 烧满 20 × 100 ms，然后以 `NV_DBG_ERRORS` 打印 `CMP Gen2: PCIe retrain completed without Gen2 link (status=0x1042, ret=0)`。该消息是假阴性。它已经误导至少一份下游分析得出 `0008` "跑得太晚"的结论。

日志级别约定加剧了它。在 `0008` 中，成功以 `NV_DBG_INFO` 打印，而四条失败路径都以 `NV_DBG_ERRORS` 打印；对比 `0007` 连常规前后转储都以 `LEVEL_ERROR` 发出，使它们能挺过默认 dmesg 过滤。**不要靠 dmesg 判断 Gen2 是否工作。** 读 `nvidia-smi --query-gpu=pcie.link.gen.current` 或 `lspci` 的 `LnkSta`。

## Retrain

Retrain 是实际改变链路速率的步骤，必须从**上游桥的** Retrain Link 位驱动，绝不可能从 GPU。Link Control 的位 5（`0x20`）只在下游端口上有意义。语料中每个实现都这么做：

| 实现 | Retrain 调用 |
|---|---|
| `debug-gen2` `retrain.sh` | `pci_write(up, cap + 0x10, 2, ctl | 0x20)` |
| `Gen2` / `far` / `deced` `retrain.sh` | `setpci -s <UP> CAP_EXP+10.w=<cur|0x20>` |
| 补丁 `0008` | `pcie_capability_write_word(upstream, PCI_EXP_LNKCTL, upstream_ctl | PCI_EXP_LNKCTL_RL)` |
| 独立 hammer 脚本 | `setpci -s "${rp}" "CAP_EXP+0x10.w=...|0x20"` |

`pci_upstream_bridge()` 返回 NULL 时 `0008` 以 `CMP Gen2: no upstream PCIe bridge; skipping link retrain` 退出，而 `debug-gen2` 的 systemd 单元字面名为 `Description=CMP 170HX PCIe Gen2 upstream soft retrain`。

### 手动主机侧流程

```bash
# 1. find the upstream root port for your GPU
lspci -tv

# 2. set Target Link Speed = 2 (Gen2) in LNKCTL2 at CAP_EXP+0x30
sudo setpci -s 64:00.0 CAP_EXP+0x30.L=2

# 3. set the Retrain Link bit (0x20) in LNKCTL at CAP_EXP+0x10, preserving current bits
sudo setpci -s 64:00.0 CAP_EXP+0x10.w=$(( LnkCtl | 0x20 ))

# 4. verify from the GPU
sudo lspci -vv -s <gpu_bdf> | grep -E "LnkCap:|LnkSta:"
```

`CAP_EXP+0x30.L=0x4` 形式目标是 Gen4，在这张卡上从未成功。内核 6.x 及以后在 bwctrl 服务中包含 `pcie_set_target_speed()`，但它**未导出**，因此 LNKCTL2 和 LNKCTL 写入必须手工发出。

### `retrain.sh` 序列

`Correct retraining`（`146da6f`）相对 `debug-gen2` 重新排序了脚本，后者先做 BAR0 写入：

```text
pre-state dump
  -> setpci -s <UP> CAP_EXP+30.w=<(cur & ~0xF) | 0x2>   (LNKCTL2 TLS = 2, on UP and GPU)
  -> sleep 0.2
  -> reopen BAR0, clear DIS_G2 at 0x8C2C0, set MAX_RATE = 2 at 0x8C040
  -> sleep 0.05
  -> verify
  -> setpci -s <UP> CAP_EXP+10.w=<cur | 0x20>            (Retrain Link)
  -> sleep 2.0                                            (up from 1.5 in debug-gen2)
  -> read CAP_EXP+12.w, print "retrain: speed_after=<sta & 0xF>"
```

提前退出前置条件：`nvidia-smi` 的 `memory.total` 为空或 `[N/A]`；`pcie.link.gen.current` 已是 2；`pcie.link.gen.max` 不在 {2, 3, 4}；BAR0 或 CYA 读 `0xFFFFFFFF`；`DIS_G2` 仍设置；`LINK_CAP` 速率半字节低于 2；以及 BAR0 写入后，"not alive or DIS_G2 set or mx != 2"。只有 Python 块内的检查打印跳过行。前三个在 shell 包装器中运行、是带无输出的裸 `exit 0`，因此完全静默的运行是正常的，不是脚本未启动的证据。

每个实现都先等驱动起来。`debug-gen2` 用 systemd `ExecStartPre=/bin/sleep 15` 加 `for _ in $(seq 1 60); do nvidia-smi -L && break; sleep 1; done`。`Gen2`、`far` 和 `deced` 在 `resource0` 存在和 `nvidia-smi -L` 上轮询 `for i in $(seq 1 120)`。`0008` 在 probe 内运行，因此只需要 `msleep(50)` 加 20 × `msleep(100)`。

> [!WARNING]
> **实验性：`tools/retrain.sh` 在 Gen2、far 和 deced 上是死代码**
>
> 这些分支发布的脚本被它们自己的安装器从 `/usr/local/sbin` 删除。grep 它们的安装器中的 `retrain` 只返回删除块。任何地方都没有 `install -m 0755 tools/retrain.sh`。要用它必须由 root 手工运行。更糟的是，在 `Gen2` 和 `far` 上脚本硬编码一位开发者的 PCI 地址（`SYS=/sys/bus/pci/devices/0000:0a:00.0`、`GPU, UP = "0a:00.0", "09:01.0"`、`PATH = "/sys/bus/pci/devices/0000:0a:00.0/resource0"`），在任何其他机器上静默瞄准错误的设备。这是回归：`debug-gen2` 自动发现两者。`deced`（`2326599`）恢复发现，用
>
> ```bash
> find_gpu_bdf() {
>   for id in 10de:20c2 10de:2082; do
>     lspci -d "$id" -D 2>/dev/null | head -1 | cut -d' ' -f1
>   done | head -1
> }
> UP_BDF="$(basename "$(dirname "$(readlink -f "/sys/bus/pci/devices/$GPU_BDF")")")"
> ```
>
> 并在 120 次等待迭代的每一次重新轮询 `find_gpu_bdf`。脚本行数：`debug-gen2` 138、`Gen2` 106、`far` 106、`deced` 115。

### 独立早期启动 hammer

一个独立的社区设置脚本走相反路线，尽可能早运行，因为"晚等于没有"。它的模型：在 `0007` 的 `CYA_0`、`LINK_CONFIG_0` 和 `VSEC_DEVICE` 写入保持期间，端点**瞬时**通告 `LnkCap2 = 0x06`；窗口在启动后约 8 到 14 秒、GSP 引导期间打开，在 RM 清除 `VSEC_DEVICE` 位 0 时关闭。陈述的关键洞见是能力不需要持久，因为在窗口打开时以 Gen2 训练的链路在窗口关闭后保持训练。

实现：`/usr/local/sbin/cmp170hx-gen2-hammer` 以 `SLEEP_S=0.05` 循环 `MAX_ITER=600`（30 秒覆盖），每一轮把两端 LnkCtl2 目标设为 Gen2 并切换**根端口的** Retrain 位。它通常在第 30 次迭代左右、约 1.5 秒内成功。它的单元用 `DefaultDependencies=no`、`After=sysinit.target`、`Before=multi-user.target`、`Type=oneshot`、`TimeoutStartSec=120`、`WantedBy=sysinit.target`，并用 `strings /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko | grep -q 'SEC2_DEBUG: PCIe'` 健全检查安装的驱动，记录到 `/var/log/cmp170hx-gen2.log`。

> [!NOTE]
> **开放问题：Gen2 窗口是瞬时的吗？**
>
> hammer 的瞬时窗口模型与一份存档稳态转储矛盾（内核 6.12.0-hiveos、驱动 610.43.03、两张 `10de:20c2` 卡），后者在**启动完成之后**读 `LnkCap2 = 0x00000006` 和 `LnkCap = 0x00456102`。`0007` 自己的 dmesg 还显示 `VSEC_DEVICE` 写入失败，因此 RM 本应清除的位可能从未被设置。双方都是一手。什么能定案：从启动早期到 60 秒、每 100 ms 对 `setpci -s <bdf> CAP_EXP+0x2c.l` 做带时间戳轮询，分别在 AMD CachyOS 主机和 HiveOS 主机上。在那之前，把瞬时窗口当作一台主机的观察，而不是卡的性质。

## Modprobe 注册表键

`install.sh` 第 5b 步写 `/etc/modprobe.d/cmp-pcie-gen2.conf`：

```text
options nvidia NVreg_RegistryDwords="RmForceEnableGen2=1;RMPcieLinkSpeed=0x1"
```

净室工具把该键记录为承重，在一条标注为 2026-07-24 卡上确认的 docstring 中：`"REQUIRES: driver loaded with NVreg_RegistryDwords=\"RmForceEnableGen2=1;RMPcieLinkSpeed=0x1\" (else the RM re-clamps Gen1 every retrain)."` 另外，同一设置脚本把 `RmForceEnableGen2` 列入"已测试并确认不必要"的清单，而且没人展示过该键独自做任何事。

> [!NOTE]
> **开放问题：`RMPcieLinkSpeed=0x1` 还是 `0x2`？**
>
> 两种写法都随分支发布。`debug-gen2`（`install.sh:191`）和 `Gen2`（`install.sh:280`）写 `0x1`；`far`（`install.sh:280`）和 `deced`（`install.sh:280`）写 `0x2`，由提交 `8854d3e` "Remove clamp link to Gen1" 引入。注意 `Gen2` 分支——那个 README 声称 Gen2 "Working ✓" 的——发布 `0x1`，卡上确认也是用 `0x1` 做的。两种读法内部都自洽，取决于键是"钳制到第 N 代"还是"使能到第 N 代"。不存在 A/B 启动测试。两个值都不应呈现为权威。什么能定案：同一张卡、同一内核启动三次，分别无键、`0x1`、`0x2`，每次贴出 `LnkSta`。便宜且决定性。

## IOMMU 启用

从提交 `6a85e6c`（2026-07-24）起安装器自动配置 IOMMU 直通。在那之前，忘记它曾是 Gen2 结果失败的最常见单一原因。

`install.sh` 读 `/proc/cpuinfo`，为 `GenuineIntel` 选 `intel_iommu=on iommu=pt`、为 `AuthenticAMD` 选 `amd_iommu=on iommu=pt`，剥离任何现有 `intel_iommu=*`、`amd_iommu=*` 或 `iommu=*` token，重写 `/etc/default/grub`（`GRUB_CMDLINE_LINUX_DEFAULT`，回退 `GRUB_CMDLINE_LINUX`）或 `/etc/kernel/cmdline`，带 `*.cmpunlocker.bak` 备份。`--no-iommu` 退出。它警告 IOMMU 也必须在 BIOS 或 UEFI 中启用（VT-d、AMD-Vi 或 SVM）。`remove.sh` 恢复备份，打印 `Reverted IOMMU kernel parameters (effective after reboot)` 或警告 `No IOMMU config backup found - kernel command line left as-is`。

`debug-gen2` 完全没有 IOMMU 处理，正式 master 也没有。`6a85e6c` 之前的手动配方：

```bash
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"/' /etc/default/grub
sudo update-grub
sudo reboot
dmesg | grep -i iommu
```

> [!NOTE]
> **开放问题：IOMMU 直通真的必需吗？**
>
> 支持：多位测试者从"拿到 64GB 内存，但还是 PCIe 1"经一次 grub 修改后成功；维护者的 `DEBUGGING.md` 把"安装后 PCIe 仍在 Gen1"的*唯一*补救给了它；安装器现在自动做。反对：独立设置脚本把 `iommu=pt` 和 VT-d 列入"已测试并确认不必要"，其确认主机加上 AMD HiveOS 成功案例**完全没做 grub 修改**。对 `iomem=relaxed` 还存在直接矛盾的单一测试者报告：一位测试者卡在 2.5 GT/s "直到我在 grub 里折腾 iommu 配置 / 因为 mmap 在失败"，另一位跑 `intel_iommu=on iommu=pt iomem=relaxed` 什么也没得到。一个合理但未证明的调和是 `iomem=relaxed` 只对用户空间 `mmap` 基 retrain 重要、IOMMU 模式只对某些芯片组重要。什么能定案：在相同软件上做 {IOMMU 关、开、pt} × {Intel、AMD} × {用户空间 hammer、驱动内 0008} 矩阵。

## 验证

```bash
# expect "2, 2"
nvidia-smi --query-gpu=pcie.link.gen.current,pcie.link.gen.max --format=csv

# expect "LnkSta: Speed 5GT/s"
sudo lspci -d 10de:20c2 -vv | grep -E 'LnkCap:|LnkSta:'
```

三条规则，按重要性：

1. **用 `LnkSta` 验证，绝不用 `LnkCap`。** `LnkCap` 可以在链路仍在 Gen1 训练时读作 Gen2。那个陷阱是多数经不起推敲的"it works"说法的已知来源。
2. **预期 sysfs 撒谎。** 在 Gen2 训练的卡上 `/sys/bus/pci/devices/<bdf>/max_link_speed` 可能仍读 `2.5 GT/s`，而 `current_link_speed` 读 `5.0 GT/s`。一台机器报告一致的 `max 5.0 GT/s`；两台机器上的三张卡报告不匹配。这是预期的，不是故障。
3. **主机带宽约翻倍、从约 0.85 GB/s 到约 1.7 GB/s，才是真正的证明。**

`Gen2/verify.sh` **只检查显存几何**，完全不含 PCIe 检查，尽管分支 README 列出 `| PCIe Gen2 link (5GT/s, Device Max >= 2) | Working ✓ |`。见 [验证](../procedures/verify.md)。

## 诊断字符串

| 来源 | 字符串 | 含义 |
|---|---|---|
| `0007` | `SEC2_DEBUG: PCIe xp3g booter FAILED to set <name>` | 一次 PLM 或熔丝写入没有经 Booter 生效 |
| `0007` | `SEC2_DEBUG: PCIe VSEC_DEVICE booter FAILED` | 在这颗硅片上预期；Gen2 仍然工作 |
| `0007` | `SEC2_DEBUG: PCIe PRIV_MISC_1 booter FAILED` | 不预期；该写入通常首次尝试成功 |
| `0007` | `SEC2_DEBUG: PCIe CYA_0 after clear DIS_G2: 0x%08x (bit2=%u)` | 信息性；`bit2` 应读 0 |
| `0007` | `SEC2_DEBUG: PCIe XVE_OVR@8872c=0x%08x; skip mid-boot retrain` | 信息性且刻意 |
| `0008` | `CMP Gen2: no upstream PCIe bridge; skipping link retrain` | 卡不在能 retrain 它的桥后面 |
| `0008` | `CMP Gen2: cannot map BAR0; skipping link retrain` | `ioremap` 失败 |
| `0008` | `CMP Gen2: PCIe capability access failed (<ret>); skipping link retrain` | 配置访问错误 |
| `0008` | `CMP Gen2: PCIe retrain completed without Gen2 link (status=0x1042, ret=0)` | **假阴性。** `0x1042` *就是* Gen2 |
| `0008` | `CMP Gen2: PCIe link trained to Gen<n>` | 成功，但以 `NV_DBG_INFO` 发出、通常被过滤掉 |
| `retrain.sh` | `BAR0 dead; skip` / `DIS_G2 still set; skip` / `Cap Gen<n>; skip` / `BAR0 dead after TLS; skip` / `preconditions failed; skip` | 提前退出 |

用 `sudo dmesg | grep SEC2_DEBUG` 读全部。记录计数：无 PCIe 补丁的 Gen1 构建 34 行，Gen2 构建 80 行，610.43.03 上两台独立双卡 Gen2 机器的 `pcielink.sh` 报告 `SEC2_DEBUG lines=152`，每次输出都带 `OPT=00000001/00000001/16680000`。唯一存档的原始双卡 Gen2 分支 `610.43.03` dmesg 含 134 行，唯一存档的单卡捕获含 29 行。

> [!NOTE]
> **行数不是可靠的跨构建指纹**
>
> 记录值有 29、34、80、134 和 152，取决于构建、分支和卡数。不要把不匹配读作安装失败。

> [!NOTE]
> **Booter 运行状态总是 `0xffff`**
>
> `kgspExecuteBooterLoad_HAL` 对每次载荷运行都返回 `0xffff`，无论写入是否落盘。每次运行后 seccode 错误码位于 mailbox0，`mailbox0 != 0` 使 `s_executeBooterUcode_TU102` 产生 `NV_ERR_GENERIC`。对载荷运行，这是 priv 定序器脚本已运行*之后*提出的预期"无效签名"抱怨。**寄存器回读是唯一有效的成功标准。** 对真正的 BooterLoad，`mailbox0 != 0` 是真实失败。

## 要求与约束

- **驱动**：nvidia-open `610.43.03`（默认）或 `610.43.02`，精确匹配。`debug-gen2` 和 `Gen2` 的 `driver/VERSION` 与正式 master 相同，其他任何版本构建硬性失败。因为 `0007` 补 `kernel_gsp.c` 和 `kernel_gsp_tu102.c`、`0008` 补 `kernel-open/nvidia/nv.c`，Gen2 工作与这两个版本紧密绑定。见 [驱动版本](../procedures/driver-versions.md)。
- **Secure Boot 必须关闭。** 如果 `mokutil --sb-state` 报告 `SecureBoot enabled`，`install.sh` 会死。
- **设备 ID** 必须是 `10de:20c2` 或 `10de:2082`。`10de:20b0` 卡能安装但得到 `unlock path not gated for this ID; skipping`。
- **裸机或 Oculink。** 直通 VM 通告 Gen2 但不训练；Thunderbolt 3 扩展坞破坏整个解锁，不只是 PCIe。

### 持久性

冷启动总是运行来自 flash、带锁定 CMP 表的签名 DevInit，因此首次枚举总是 Gen1。单纯 `rmmod` 和 `modprobe` **不会**重跑 DevInit（没有 PERST），因此补丁每次 GSP 启动重新触发并恢复寄存器值，但 retrain 必须在每次重载后重新触发。完整复位路径（PERST、`nvidia-smi --gpu-reset`、`echo 1 > /sys/bus/pci/devices/<bdf>/reset`）重跑签名 DevInit 并丢弃修复。这个模型与每个观察一致，但未通过直接前后 PERST 测量确认。

Gen2 家族的 `remove.sh` 清理整个足迹：禁用并复位失败 `cmpretrain.service` 和 `cmp-gen2-retrain.service`、移除两个单元文件、`/usr/local/sbin/retrain.sh`、`/usr/local/sbin/cmp-gen2-retrain.sh` 和 `/etc/modprobe.d/cmp-pcie-gen2.conf`（`Removed PCIe Gen2 helpers`），然后恢复 `*.cmpunlocker.bak` 内核命令行备份。

## 已知失败模式

| 症状 | 状态 |
|---|---|
| `install.sh` 后立即 `nvidia-smi` 报告 `2, 2`，但重启后可复现地回到 `1, 1` | 未解释。每次重跑 `install.sh` 都能恢复。一位 NixOS 用户通过在内核层面应用补丁、使它每次启动都重新应用来绕过。工作假说：*补丁过的*模块实际不是启动时加载的那个。责怪 retrain 之前先查 `modinfo` 并找重启后的 `SEC2_DEBUG: PCIe` 行 |
| 一台 Intel 平台永远到不了 Gen2 | 一台带四个 PCIe 5.0 x16 槽的 ASUS W890 SAGE，Ubuntu 24.04，内核 7.0.0-28-generic，双卡。试过：`Gen2` 分支、`debug-gen2`、外部 fork、含 `intel_iommu=on iommu=pt iomem=relaxed` 的 grub 行、焊接卡和未改卡、槽 1 和 4。每次都：`LnkSta: Speed 2.5GT/s (downgraded), Width x4 (downgraded)`，而 `LnkCap` 正确通告 5 GT/s。内核版本被另一位测试者排除：他把 CachyOS 回滚到 6.12-LTS，没有变化。对照成功案例：AMD、HiveOS Ubuntu 22.04、内核 6.12.0、完全没做 grub 修改 |
| Proxmox 或 VFIO 下的访客 VM | 通告能力，训练不发生。Retrain 必须从**主机**在物理根端口驱动，因为访客访问不到真实上游桥 |
| Thunderbolt 3 | Booter Load 直接失败（`0x15` / `0xffff`），因此这是计算和显存失败，不是 PCIe 失败。用 Oculink |
| 首个公开补丁无法应用 | `kernel_gsp_tu102.c` 上 `patch: **** malformed patch at line 264`。两个 hunk 头夸大了行数：`@@ -4942,6 +4942,323 @@` 应为 `260`，`@@ -611,6 +611,50 @@` 应为 `44`。当天由 `0901346` 修复。`debug-gen2` 的 `0007` 与 `Gen2` 的完整 diff 恰好显示两行不同，都是 hunk 头；补丁主体在四个分支间逐字节相同。它能活下来只因为 `build.sh` 用宽松的 `patch -p1` 而不是 `git apply` |

还要注意，看到补丁应用错误中出现 `kernel_gsp_tu102.c` **不**意味着补丁针对 Turing。`_TU102` 后缀的 GSP 函数正是 170HX 执行的函数；它们出现在可工作 Ampere 卡上的 Booter 失败消息中（`s_executeBooterUcode_TU102`、`kgspExecuteBooterLoad_TU102`、`kgspBootstrap_TU102`）。

不要把这个 `0007` 与净室线的 `0007-pcie-gen4-shadow.patch` 混淆，后者被弃置到启动循环中。同名不同补丁。

## 实测 Gen2 结果

| 项目 | 值 | 条件 | 置信度 |
|---|---|---|---|
| 主机带宽，Gen2 x4 | 1.68 GB/s 发送，1.71 GB/s 接收 | OpenCL-Benchmark，一份存档截图，一张未改卡 | 中 |
| 主机带宽，Gen2 x4 | 约 1.71 GB/s | 设置脚本自己的预测，"约 0.85 到约 1.71 GB/s，恰好 2 倍"，在一台 AMD B650M / CachyOS 主机上验证。不是独立测量 | 低 |
| Gen1 x4 到 Gen2 x4，一次 A/B | 1.67 到 3.24 GB/s | OpenCL，在只协商到 x8 的改装卡上 | 中 |
| Gen2 x16 | 6.63 到 6.67 GB/s | `ocl_pcie_bw`，一台机器，2026-07-26，完整 24 电容改装 | 中 |
| pp512 | 203.84 到 277.84 t/s | Q8 ik_llama 带 MTP，10 GB 卡解锁到 40 GB，`--spec-type mtp:n_max=2,p_min=0.0`，其余不变 | 高 |
| pp2048 | 328.81 到 449.41 t/s | 同一 A/B | 高 |
| pp8192 | 363.25 到 493.86 ± 16.92 t/s | 同一 A/B | 高 |
| tg128 | 38.15 到 41.52 ± 1.89 t/s | 同一 A/B | 高 |
| tg512 | 37.69 到 40.12 t/s | 同一 A/B | 高 |
| tg2048 | 36.78 到 37.90 t/s | 同一 A/B | 高 |
| Gen2 下 AER 计数器 | 0 / 0 / 0 | 两张 `0x20c2` 卡，内核 6.12.0-hiveos | 高 |
| Gen2 去加重 | -3.5 dB | `LnkSta2`，首份确认的 Gen2 捕获 | 中 |

Prefill 收益明显；token 生成几乎不动。这与算术一致：5120 隐藏维度下 fp16 激活每 token 每跳 10,240 字节，因此解码流量远未接近链路天花板。见 [LLM 推理](../operations/llm-inference.md)。

## 开放问题

> [!NOTE]
> **开放问题：修复 0008 的成功谓词**
>
> 整个 PCIe 领域最易处理的一项，也是一行改动。去掉 `PCI_EXP_LNKSTA_DLLLA` 项，或让它以 `PCI_EXP_LNKCAP_DLLLARC` 为条件，或从上游桥而不是端点读 `LnkSta`。另外把成功打印提高到 `NV_DBG_ERRORS`，与 `0007` 的约定一致。

> [!NOTE]
> **开放问题：0008 是足够、不必要，还是主动误导？**
>
> 三位独立测试者报告 `0008` 修复了多卡 Gen2 并消除了崩溃。独立设置脚本断言 `0008` 在驱动 probe 时运行、大约在能力窗口关闭后三秒，并把它列入"已测试并确认不必要"。DLLLA 缺陷使**双方**都复杂化：`0008` 的失败消息在这张卡上无条件发出，因此它不是 retrain 失败的证据；同样，三位测试者可能是在看那些经另一条路线到达 Gen2 的卡上的 `nvidia-smi`。什么能定案：装带 `0008`、无 hammer 服务的 Gen2 分支，在已知 hammer 能成功的主机上冷启动后查 `pcie.link.gen.current`。

> [!NOTE]
> **开放问题：为什么有些用户得到 Gen2、有些没有？**
>
> `pcielink.sh` 报告专门被传播，以便把内核、驱动、序列号、板卡部件号和 VBIOS 与成功失败关联。两个 VBIOS 版本已经在其他方面相同的 `900-11001-0108-000` 板上出现：`92.00.6D.00.0A` 和 `92.00.67.00.01`。下一步：按 VBIOS 和根端口型号制表，并测试记录的冷启动依赖（一次净室运行在冷启动后需要重新打开 27 道 PCIe PLM 中的 18 道）。

> [!NOTE]
> **开放问题：把 Gen2 合并到 master**
>
> 树中可见的阻碍：`0007` 是一个大而带调试仪表、全程以 `LEVEL_ERROR` 记录的 hunk；`tools/retrain.sh` 在 `Gen2` 和 `far` 上是死代码；`constants.yaml` 漏掉机制依赖的五个寄存器；`verify.sh` 完全不检查 PCIe。多卡、IOMMU 和 Gen2 工作是否合并、以什么顺序，尚未决定。多卡安装器改动自包含，可以单独落地。

> [!NOTE]
> **开放问题：一个无法解释的早期 Gen2 说法**
>
> 一份验证过的净室交流、日期 2026-07-05，把 Gen3 说法修正为"只有 2.0 有"，把 Gen2 当作已完成，这比复现结果早三周，并直接与 2026-07-07 一条仍说"We still need something for PCIe 2.0"的消息矛盾。要么更早的独立结果从未传播，要么时间戳归属有误。只有原始消息元数据能定案。那条消息的技术内容（Gen3 被熔丝门控）与其他一切一致、可以依赖；日期不能。

## 另见

- [PCIe 子系统](../hardware/pcie-subsystem.md)，熔丝、DevInit 表和宽度上限
- [Gen3 与 Gen4](../frontier/pcie-gen3-gen4.md)，未解决的一半
- [权限级掩码](privilege-level-masks.md)，九条目 PLM 表
- [Falcon 与 Booter](falcon-and-booter.md)，`0007` 依靠的写入原语
- [驱动补丁](driver-patches.md)，`0001` 到 `0008` 完整清单
- [寄存器参考](register-reference.md) 和 [寄存器索引](../appendix/register-index.md)
- [故障排查](../procedures/troubleshooting.md)
- [状态板](../frontier/status-board.md)
