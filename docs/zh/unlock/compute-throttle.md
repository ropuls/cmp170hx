# 计算节流及其解除方式

**本页涵盖：** NVIDIA 用来削弱 CMP 170HX 算术吞吐的机制、击破它的精确寄存器与值、为什么熔丝本身从不被碰、实测提升是多少、为什么名字最有希望的寄存器（`SM_ISSUE_RATE_MODIFIER`）是死路，以及为什么计算解锁在[显存解锁](memory-geometry.md)不行的情况下仍能挺过功能级复位。

**简短版。** 限制是一条**指令发射率分频器**，实现为 `OPT_SM_SPEED_SELECT` 块中的九个一次性可编程熔丝，在此 SKU 上全部设为其最大除数（除以 32），在每张 A100 上为零。熔丝无法更改，解锁也不尝试。它改为打开一个权限级掩码 `FEAT_OVR_PLM`（`0x00823804`），从仅 L3（`0xffffff8f`）到完全开放（`0xffffffff`），用 SEC2 Booter 作为特权写入原语，然后执行**两次普通主机寄存器写入**：

```c
GPU_REG_WR32(pGpu, 0x0082381cU, 0x88888888U);   /* SS0: FEATURE_OVERRIDE_SM_SPEED_SELECT   */
GPU_REG_WR32(pGpu, 0x00823820U, 0x00000008U);   /* SS1: FEATURE_OVERRIDE_SM_SPEED_SELECT_1 */
```

这两个 dword 就是整个计算解锁。FP32 从 0.395 TFLOPS 升到约 12.7 到 12.9 TFLOPS（约 **32 倍**），FP64 和每条张量路径随之恢复，覆盖因 `0x00823804` 位于常开岛而经得起 FLR。这里没有任何东西写入卡的 BIOS；补丁驱动每次 GSP 启动重新应用该序列。

---

## 第 1 层：熔丝

节流按算术单元。九个熔丝，一个共享权限级掩码。熔丝值 `0` 是满速；`5` 是最大除数，除以 32。`FUSE_SS_DP` 是 1 位熔丝，`0` 为满速、`1` 为降低，因此 `1` 是*它*的最大值。

| 熔丝 | 地址 | 管辖 | 170HX | A100 / A10 / A5000 / A6000 / DRIVE A100 | RTX 3080-3090 Ti |
|---|---|---|---|---|---|
| `FUSE_SS_DP` | `0x00820224` | FP64（1 位） | `0x00000001` | `0x00000000` | `0x00000000` |
| `FUSE_SS_FFMA` | `0x0082059c` | FP32 融合乘加 | `0x00000005` | `0x00000000` | `0x00000000` |
| `FUSE_SS_FMLA16` | `0x008207d4` | FP16 MLA | `0x00000005` | `0x00000000` | `0x00000000` |
| `FUSE_SS_FMLA32` | `0x008207d8` | FP32 MLA | `0x00000005` | `0x00000000` | `0x00000001` |
| `FUSE_SS_IMLA0` | `0x008207dc` | 整数 MLA 0，也是 DP4A | `0x00000005` | `0x00000000` | `0x00000000` |
| `FUSE_SS_IMLA1` | `0x008207e0` | 整数 MLA 1 | `0x00000005` | `0x00000000` | `0x00000000` |
| `FUSE_SS_IMLA2` | `0x008207e4` | 整数 MLA 2 | `0x00000005` | `0x00000000` | `0x00000000` |
| `FUSE_SS_IMLA3` | `0x008207e8` | 整数 MLA 3 | `0x00000005` | `0x00000000` | `0x00000000` |
| `FUSE_SS_IMLA4` | `0x008207ec` | 整数 MLA 4 | `0x00000005` | `0x00000000` | `0x00000001` |
| `FUSE_SS_PLM` | `0x008200fc` | 块上的共享 PLM | `0xffffffff` | `0xffffffff` | `0xffffffff` |

这些读数来自 2026-05-07 到 2026-07-27 之间两个 SKU 上至少五次独立的 170HX 探测，加一个 11 卡租用对照队列和两块物理 DRIVE A100（PG199）板。这些值是**产品线常量**：每台 170HX 都相同，这就是固定解锁配方安全的原因。对比下面的覆盖寄存器，它们是逐裸片的。

> [!NOTE]
> **一个常被重复的不精确说法**
>
> 说"全部 9 个速度选择熔丝为 `0x5`"的摘要不严谨。八个熔丝读 `0x5`；第九个 `FUSE_SS_DP` 读 `0x1`，因为它是 1 位字段，`0x1` 是它自己的最大值。

### 产品档签名

`FUSE_SS_FMLA32` 和 `FUSE_SS_IMLA4` 把探测过的 Ampere 队列分成恰好三档：

| 值 | 部件 |
|---|---|
| `0x00000000` | A100 SXM4 40G、A100 PCIe 40G、A100 PCIe 80G、A10、A5000、A6000、DRIVE A100 |
| `0x00000001` | RTX 3080、RTX 3080 Ti、RTX 3090、RTX 3090 Ti |
| `0x00000005` | **CMP 170HX，两台** |

`0x1` 档是众所周知的消费级 FP16 带 FP32 累加张量吞吐减半。CMP 节流不是特殊机制：它是同一机制调到了最大除数。

### 为什么不能直接改熔丝

四条路线被尝试并关闭：

- **写 `OPT_SM_SPEED_SELECT` 寄存器。** 它们是 OTP 熔丝*影子*，无论权限如何都只读。`FUSE_SS_PLM`（`0x008200fc`）在每张卡上都完全开放看起来像疏忽，但什么都得不到。
- **FUSECTRL 软件熔丝覆盖路径。** 在此部件上关闭：`NV_FUSE_FUSECTRL 0x00820000 = 0xe0040000`、`FUSE_EN_SW_OVERRIDE 0x00820040 = 0x00000000`、`ENABLE_FUSE_PROGRAM_STATUS 0x00820078 = 0x00000001`、`DISABLE_FUSE_PROGRAM_STATUS 0x0082007c = 0x00000000`、`BYPASS_FUSES_STATUS 0x00820080 = 0x00000000`、`DISABLE_SW_OVERRIDE_STATUS 0x00820084 = 0x00000001`。一张 GA10x 对照卡共享 FUSECTRL 值，但 `EN_SW_OVERRIDE = 0x00000001`，这证明寄存器可用、且在这里被故意关闭。
- **FECS 镜像。** `FECS_FEAT_OVERRIDE 0x00409664` 和 `FECS_FEAT_READOUT_1 0x00409668` 在全部十五张探测过的 Ampere 卡（包括未节流的）上返回 PRI 权限违规哨兵 `0xbadf5040`，因此该值是读块指示，不是数据。
- **物理重新熔断硅片。** 2024 年被点名为攻击路径，从未尝试，被覆盖寄存器变得无关紧要。

---

## 第 2 层：FEATURE_OVERRIDE 块

`0x00823800` 块是一组**优先级高于熔丝**的寄存器。对锁定卡上 `0x00823800` 到 `0x00823ffc` 的完整范围扫描只返回十三个活 dword；其他每个偏移返回 `0xbadf5040`。

| 寄存器 | 地址 | 原厂 170HX | 作用 |
|---|---|---|---|
| `FEATURE_OVERRIDE_ECC PLM` | `0x00823800` | `0xffffff8f` | ECC 覆盖组上的 PLM（与 `0x00823804` 不同的寄存器） |
| **`FEATURE_OVERRIDE PLM`（FEAT_OVR_PLM）** | **`0x00823804`** | **`0xffffff8f`** | **那道门。原厂仅 L3。常开岛** |
| `FEATURE_OVERRIDE_QUADRO` | `0x00823808` | 两台物理 170HX 上 `0x00000181` / `0x00000182`；其他转储读 `0x00100183`（原厂 PLM 范围扫描）和 `0x00000081`（解锁后探测）；A100 80 GB 读 `0x01000282` | 逐裸片，13 个分档差异之一，且无法解释。只读。为什么值在转储间不同是开放问题；见 [寄存器参考](register-reference.md) |
| `FEATURE_OVERRIDE_ECC` | `0x0082380c` | `0x00888888` | SM_LRF / L1 / LTC / DRAM / CBU ECC 控制 |
| `FEATURE_OVERRIDE_ECC_1` | `0x00823810` | `0x002aaaaa` | icache / FECS / GPCCS / PMU / HUBMMU ECC |
| `FEATURE_READOUT`（READOUT_0） | `0x00823814` | `0x00000233` | Quadro 位 [5:0] + ECC 状态 [31:12]，只读 |
| **`FEATURE_READOUT_1`** | **`0x00823818`** | **`0x016db6ed`** | **只读有效 SM 速度选择，全部九个单元** |
| **`FEATURE_OVERRIDE_SM_SPEED_SELECT`（SS0）** | **`0x0082381c`** | 逐裸片 | **IMLA0-3、FMLA16、FMLA32、FFMA、DP：八个 4 位字段** |
| **`FEATURE_OVERRIDE_SM_SPEED_SELECT_1`（SS1）** | **`0x00823820`** | 逐裸片 | **第九个字段，IMLA4** |
| `FEATURE_OVERRIDE_ROW_REMAPPER` | `0x00823824` | `0x00000000` / `0x00000001` | 在 `0x00823b00` 有自己的 PLM |
| `FEATURE_READOUT_2` | `0x00823828` | `0x00000000` | |
| `FEATURE_OVERRIDE_ECC_2` | `0x0082382c` | `0x0000000a` | LTC_CBC 和 SM_URF ECC |
| `FEAT2 PLM`（ROW_REMAPPER PLM） | `0x00823b00` | `0xffffff8f` | 只被 Gen2 家族分支打开 |

> [!CAUTION]
> **SS0 和 SS1 是逐裸片分档值。绝不要拿一张卡的读数当权威。**
>
> 队列中实测原厂 SS0：170HX `0x51261070`、另一台 170HX `0x10206152`、第三台 `0x71066125`、第四台 `0x12103060`；`0x20bb` GA100 参考板（未节流，`FEAT_READOUT_1` = 0）`0x53540175`；A100 SXM4 40G `0x10413004`；A100 PCIe 40G `0x14604062`；A100 PCIe 80G `0x72020072`；A10 `0x11303071`；A5000 `0x63573073`；A6000 `0x14170072`；RTX 3080 `0x03676064`；RTX 3080 Ti `0x10551033`；RTX 3090 `0x06740057`；RTX 3090 Ti `0x30403100`；DRIVE A100 `0x25045144`。同一 A100 80 GB 设备 ID 的两份存档转储互相矛盾（`0x00112011`/`0x00000002` 对比 `0x00343015`/`0x00000004`），因此这些是运行时状态，不是稳定熔丝状态。用 `FEATURE_READOUT_1`（`0x00823818`），不要用 SS0/SS1，作为参考目标。

`FEATURE_READOUT_1` 是唯一稳定且有意义的值：它在大约两台物理 170HX 卡上读出相同的 `0x016db6ed`（尽管它们的 SS0/SS1 不同），在每张 A100 和 DRIVE A100 上读出 `0x00000000`，在全部四张 RTX 30 系列部件上读出 `0x00400080`。**`0x00823818 == 0` 是现有最干净的"此卡是否已解锁"测试。**

### 半字节编码

每个 SS0 半字节最好读作 `[enable | 3 位速度]`。`0x8` 设置位 3（覆盖使能），位 [2:0] = 0（速度 0，满速）。因此 `0x88888888` 表示全部八个 SS0 单元"覆盖使能、满速"，`0x00000008` 对 SS1 中单独的 IMLA4 做同样的事。

> [!WARNING]
> **编码是推断的，没有文档**
>
> 语料中不存在 NVIDIA 对该字段布局的文档。该读法由三个观察支撑：存档中任何原厂转储都没有任何半字节大于等于 8，即在原厂硅片上覆盖使能位是清除的、字段内容为无关项；`0x00823818` 的有效读出在写入后变为零；性能结果匹配。它从未逐字段确认。在解锁卡上对 SS0 做单半字节扫描、观察 `0x00823818` 的哪些位移动，能同时定案编码和读出解码。

### 门控链

```text
FUSE_QUADRO_WR_SEC (0x0082038c) = 1
        permits
FEAT_OVR_PLM (0x00823804) to be opened from 0xffffff8f to 0xffffffff
        permits
PL0 host writes to SS0 (0x0082381c) and SS1 (0x00823820)
        which
outrank the OPT_SM_SPEED_SELECT fuses
```

在同一张卡上测到两个门控熔丝：`OPT_SECURE_FEATURE_OVERRIDE_QUADRO_WR_SECURE`（`0x0082038c`）= `0x00000001` 和 `OPT_SECURE_GSP`（`0x0082074c`）= `0x00000001`。

而在这一切之上：

> [!NOTE]
> **让这一切成为可能的那根熔丝**
>
> CMP 170HX 上 `0x008203f0` 的 `OPT_FEATURE_FUSES_OVERRIDE_DISABLE`（`FUSE_FEAT_OVR_DIS`）读出 `0x00000000`。探测注释它"MASTER KILL: if YES all overrides permanently locked"。如果 NVIDIA 熔断了那一根，本页每条路线都会被永久关闭。它在探测过的每张卡（包括 GA10x 对照）上都读零。

注意 `FEAT_OVR_PLM 0x00823804` 在**全部十五张**探测过的 Ampere 部件上读出 `0xffffff8f`（仅 L3），包括每张 A100。170HX 在这里并不特殊。解锁的全部难度是到达 L3 来改变它，这正是 [SEC2 Booter 路径](falcon-and-booter.md) 做的。只有 SS0 和 SS1 在 PL0 可被主机写；PLM 本身必须由高安全模式的 Falcon 写。正如一份分析所说，若非如此，任何 NVIDIA 卡都可以不经漏洞利用就被解锁。

---

## 正式代码实际做什么

来自 `master` 分支的 `driver/patches/0001-sec2-postbl-plm-ss-cfg.patch`，位于 GSP 引导路径内，由接受 `0x20C2`（8 GB）**和** `0x2082`（10 GB）的 `_kgspSec2PostblTimingEnabled()` 按 PCI 设备 ID 门控：

```c
static const struct { NvU32 addr; NvU32 value; const char *name; } plmTable[] = {
    { 0x001fa7ccU, 0xfffff0ffU, "WPR_CFG" },
    { 0x009a0148U, 0xffffffffU, "FBPA" },
    { 0x001fa7c4U, 0xffffffffU, "WPR" },
    { 0x00823804U, 0xffffffffU, "FEAT" },
};

NvU32 wpr2Lo = GPU_REG_RD32(pGpu, 0x001fa824U);
NvU32 wpr2Hi = GPU_REG_RD32(pGpu, 0x001fa828U);

for (plmIdx = 0; plmIdx < 4; plmIdx++)
{
    NvBool opened = NV_FALSE;
    for (attempt = 0; attempt < 2 && !opened; attempt++)
    {
        GPU_REG_WR32(pGpu, 0x001fa824U, wpr2Lo);        /* re-arm WPR2 before every attempt */
        GPU_REG_WR32(pGpu, 0x001fa828U, wpr2Hi);

        plmStatus = kgspSec2PostblTimingRefillPayload(pGpu, pKernelGsp,
            plmTable[plmIdx].addr, plmTable[plmIdx].value);
        if (plmStatus != NV_OK)
            continue;

        plmStatus = kgspExecuteBooterLoad_HAL(pGpu, pKernelGsp,
            memdescGetPhysAddr(pKernelGsp->pWprMetaDescriptor, AT_GPU, 0));

        NvU32 regVal = GPU_REG_RD32(pGpu, plmTable[plmIdx].addr);
        if (regVal == plmTable[plmIdx].value)
            opened = NV_TRUE;
    }
}
```

然后，PLM 打开后：

```c
GPU_REG_WR32(pGpu, 0x0082381cU, 0x88888888U);   /* SS0  */
GPU_REG_WR32(pGpu, 0x00823820U, 0x00000008U);   /* SS1  */
GPU_REG_WR32(pGpu, 0x009a0204U, cfg1Value);     /* CFG1 (memory geometry) */
GPU_REG_WR32(pGpu, 0x00100ce0U, lmrValue);      /* LMR  (memory geometry) */
```

值得注意的点：

- 载荷携带**一个**（地址，值）对，Booter Load 对**每个 PLM 重新触发一次**，**每个最多 2 次尝试**，WPR2 边界 `0x001fa824` / `0x001fa828` 在每次尝试周围保存并恢复。成功由**回读**判定，而不是 Booter 状态——后者无论是否成功每次都返回 `0xffff`。
- 四个 PLM 中只有**三个**到 `0xffffffff`。`WPR_CFG 0x001fa7cc` 被写 `0xfffff0ff`。任何说"所有 PLM 必须显示 `0xffffffff`"的文档都是松散措辞。
- **两个 SKU 的 SS0 和 SS1 相同。** 只有 `cfg1Value` 和 `lmrValue` 按设备 ID 选择。见 [显存几何](memory-geometry.md)。
- 正式顺序是 SS0、SS1、CFG1、LMR，随后一行回读日志。
- SS0 **和** SS1 都必须写。只写一个不够。
- `common/constants.yaml` 记录 `compute: ss0: "0x88888888"` / `ss1: "0x00000008"`，但 `install.sh` 和 `driver/build.sh` 都不读该文件。值硬编码在补丁里。把 YAML 当作碰巧与代码一致的文档。
- SS0/SS1 在全部十二个未发布分支中逐字节相同。没有任何分支实验过不同的计算值；所有计算实验都早于值定案。

---

## 验证解锁

第二个正式补丁 `0002-booter-verify.patch` 定义项目自己认为决定性的规范五寄存器集，并在每次 Booter Load 后记录：

| 符号 | 地址 |
|---|---|
| `SEC2_DEBUG_PRI_FEATURE_OVERRIDE_PLM` | `0x00823804` |
| `SEC2_DEBUG_PRI_FEATURE_OVERRIDE_SM_SPEED` | `0x0082381c` |
| `SEC2_DEBUG_PRI_FEATURE_OVERRIDE_SM_SPEED_1` | `0x00823820` |
| `SEC2_DEBUG_PRI_FBPA_CFG1` | `0x009a0204` |
| `SEC2_DEBUG_PRI_MMU_LMR` | `0x00100ce0` |

```bash
sudo dmesg | grep SEC2_DEBUG
```

一台成功的 8 GB 卡打印：

```text
SEC2_DEBUG: POST-WRITE SS0=0x88888888 SS1=0x00000008 CFG1=0x02779000 LMR=0x0000020b (devId=0x20c2)
```

8 GB 卡解锁后的寄存器状态，对比锁定对照卡：

| 寄存器 | 地址 | 锁定 | 解锁 |
|---|---|---|---|
| `FEAT_OVR_PLM` | `0x00823804` | `0xffffff8f` | `0xffffffff` |
| SS0 | `0x0082381c` | 逐裸片，如 `0x12103060` | `0x88888888` |
| SS1 | `0x00823820` | 逐裸片，如 `0x00000003` | `0x00000008` |
| `FEAT_READOUT_1` | `0x00823818` | `0x016db6ed` | `0x00000000` |
| `FUSE_SS_FFMA` 及同类 | `0x0082059c` 等 | `0x00000005` | **`0x00000005`，不变** |

最后一行是这是覆盖而非熔丝编辑的硬确认：成功解锁后熔丝影子仍读 `5`（DP 仍读 `1`），而有效读出为零。

> [!WARNING]
> **`clocks.max.sm` 不是好的验证信号**
>
> `install.sh` 打印 `nvidia-smi --query-gpu=clocks.max.sm --format=csv,noheader` 作为计算验证步骤，一份解锁后报告从它读到 `1935 MHz`。每项持续测量都反驳把它当作运行时钟读：VBIOS 表最大图形时钟是 1695 MHz，实际硅片天花板在 +350 偏移下约 1604 到 1614 MHz。持续 SM 频率是 **1410 MHz**（`-pl 300` 下 1470 MHz）。把 1935 MHz 视为报告字段、单一报告、低置信度。更好的功能检查是 NVML GPC 时钟 VF 偏移范围回来是 `[-1000 .. +1000]` 而不是 `[0 .. 0]`；见 [调优](../operations/tuning.md)。

---

## 实测提升

锁定 FP32 融合乘加吞吐测得 **394.77 GFLOPS**，且在 float、float2、float4、float8 和 float16 之间*相同*。跨向量宽度的这种完美平坦正是固定指令发射限制的特征，而不是带宽或占用限制。算术精确闭合：4480 通道 1410 MHz 的理论 FP32 是 12.634 TFLOPS，12.634 / 32 = 394.8 GFLOPS。

这个五位数字**不是**社区测量。它来自 2023 年对一张原厂、从未解锁卡的公开 clpeak 评测，也是匹配的无 FMA 对照（float 6285.48 GFLOPS，约快 16 倍）和锁定 FP64 182.72 GFLOPS 的来源。社区记录只带同一数量的四舍五入转述：8 卡基准文章里的 0.39 TFLOPS，以及把 12.28 / 32 *反算*而非实测的 0.38 TFLOPS。除非直接引用那份外部 clpeak 运行，否则引用为**约 0.39 TFLOPS**。

### 首张完整前后对照表（2026-07-06，单卡）

以渲染图而非工具输出发布，与私有验证频道中第一份"I have compute unlock working"报告一起。来源附件：`archive/cleanroom/1523499947490541640_PNG_image.png`。

| 数据类型 | 节流 | 解锁 | 比率 |
|---|---|---|---|
| FP64 | 0.20 TF/s | 12.91 TF/s | 63.0 倍 |
| FP32 IEEE | 0.41 TF/s | 12.69 TF/s | 31.0 倍 |
| INT8 | 1.63 TOP/s | 50.50 TOP/s | 30.9 倍 |
| BF16 | 6.40 TF/s | 184.86 TF/s | 28.9 倍 |
| FP16 | 6.52 TF/s | 153.92 TF/s | 23.6 倍 |
| INT4 | 11.55 TOP/s | 259.34 TOP/s | 22.5 倍 |
| INT1 | 46.16 TOP/s | 1038.89 TOP/s | 22.5 倍 |
| TF32 | 90.72 TF/s | 90.09 TF/s | 1.0 倍，标为"untouched"（有争议） |

> [!NOTE]
> **为什么这个日期比时间线解锁里程碑早六天**
>
> [timeline.md](../history/timeline.md) 把"compute unlock works on hardware"标到 **2026-07-12**。那是首次*社区复现*解锁。这张表早于它，因为私有验证频道在 **2026-07-06 01:24** 已有计算可用，同一条消息还报告 INT4 和 INT8 经 CUTLASS TN 形状调优达到 300 和 600。两个日期不冲突；它们分别标记私有首次点亮和公开复现。
>
> 以一张图应得的谨慎对待这些数字：没有命名工具、没有说明时钟或 flop 计数约定，八行中没有任何一行被逐位复现。节流列在种类上由外部 clpeak 评测佐证（锁定 FP64 182.72 GFLOPS 对比这里的 0.20 TF/s；锁定 FP32 394.77 GFLOPS 对比 0.41 TF/s），解锁的 FP16 和 BF16 行落在后来 8 卡区间的内部。TF32 行被直接争议。

### 独立确认

| 测量 | 值 | 条件 |
|---|---|---|
| SGEMM FP32 | 12.28 TFLOPS | 2026-07-12，验证频道外首次报告完整 SM 解锁，cc 8.0；约一分钟后被一次独立的 gpu-burn 运行佐证：12229 Gflop/s，0 错误，62 C |
| DGEMM FP64 | 11.48 TFLOPS | 同一运行（张量 DMMA 路径，见下方） |
| FP32，OpenCL-Benchmark | 12.890 TFLOPs/s | 64 GB 解锁卡，驱动 610.43.03 |
| FP64，OpenCL-Benchmark | 6.421 TFLOPs/s（FP32 的 1/2） | 同一运行 |
| FP16，OpenCL-Benchmark | 48.740 TFLOPs/s（FP32 的 4 倍） | 同一运行 |
| INT8，OpenCL-Benchmark | 49.362 TIOPs/s | 同一运行 |
| FP32 非张量 | 12.6 到 12.76 TFLOPS | 2026-07-27，一台调优卡加 8 卡租赁 |
| FP16 张量 | 158.7 到 162.7 TFLOPS | 同一轮 |
| BF16 张量 | 171.4 到 192.7 TFLOPS | 同一轮 |
| TF32 张量 | 79.0 到 91.9 TFLOPS | 同一轮 |
| INT8 | 44.1 TOPS | 同一轮，仍被门控 |

### FP64 区间是两条路径，不是争议

约 6.3 和约 12 TFLOPS FP64 之间的表面冲突**已解决**，由 2026-07-15 一份单次运行打印两个数字的 clpeak 转储解决（sm_80，70 SMs，7890 MB，驱动 13.0）：

| 路径 | 指令 | 实测 |
|---|---|---|
| FP64 非张量 | 普通 `double` FMA | **6.31 TFLOPS**（`double : 6308.65` GFLOPS） |
| FP64 张量 | `wmma`/`mma` `fp64xfp64+fp64` 8x8x4（DMMA） | **11.96 TFLOPS**（`wmma_fp64 : 11.96`） |

非张量数字是架构性 1:2 速率：同一运行 FP32 的一半（`float : 12565.14` GFLOPS）。张量数字是 GA100 暴露的第二条 FP64 数据路径，也是 11.48 到 12.91 TFLOPS 簇的来源。因此 OpenCL-Benchmark 的 6.421 TFLOPs/s 和 DGEMM 的 11.48 TFLOPS 从未测量同一件事，也不涉及 flop 计数错误。

表述为：**FP64 非张量约 6.3 TFLOPS，FP64 张量约 12 TFLOPS。** 两者都被解锁完全恢复。同一转储也是张量行一般意义上最干净的单次运行来源：`wmma_fp16` 179.19、`fp16_f16acc` 189.66、`wmma_bf16` 179.19、`wmma_tf32` 89.69 TFLOPS。

### 解锁前张量核心崩溃

针对 A800 对照的周期级测量展示了节流对 `mma.sync` 做了什么：

| Warps | 170HX（节流） | A800 对照 |
|---|---|---|
| 1 | 256.40 周期 | 24.64 周期 |
| 4 | 256.34 周期 | 24.55 周期 |
| 5 | 374.65 周期 | |
| 8 | 513.83 周期 | |
| 16 | 1026.20 周期 | |
| 32 | 2039.46 周期 | 71.45 周期 |

墙钟吞吐在任何占用下从不超过约 0.082 TFLOPs，对比 A800 的 1.807910 TFLOPs。约每指令 10 倍惩罚，加上每 SM 并行 `mma.sync` 硬限制 4 warps。

---

## 解锁不改变什么

- **它不增加 SM。** 前后都是 70 个 SM，`smid` 0..69 无缺口。卡已经在其硅片熔丝下限。见 [GA100 硅片](../hardware/ga100-silicon.md)。
- **它不提高时钟。** 频道内权威表述是"计算限制解除，总线速率不动"。超频是单独的 NVML 杠杆；见 [调优](../operations/tuning.md)。
- **它不改变 PCIe 链路速率或宽度。** Gen2 在未发布分支上（[PCIe Gen2](pcie-gen2.md)），宽度是焊接活（[物理改装](../operations/physical-mods.md)）。
- **它不恢复 INT8 / IMMA。** 解锁 INT8 测得 44.1 TOPS，同一张卡上比 FP16 约慢 **3.7 倍**，而在 A100 上 INT8 比 FP16 约快 **2 倍**。IMLA 熔丝读同样的 `0x5`，SS0 覆盖半字节设置相同，但实测 IMMA 速率不跟随。对推理的实际影响：使用 W4A16（AWQ 或 GPTQ，INT4 权重加 BF16 激活），完全避开 W8A8；KV 缓存必须 BF16。见 [LLM 推理](../operations/llm-inference.md)。
- **标量 FP16 从未被节流**，即使在锁定卡上：GA100 以 4 倍其 FP32 fma 速率运行 16 位 hfma，锁定卡测约 42 到 50 TFLOPS 标量 FP16（mixbench 41869 GFLOPS；OpenCL half2-fma 约 48 到 50 TFLOPS）。这就是锁定卡原本已可用于 LLM token 生成的原因，也是 `FUSE_SS_FMLA16` 读 `0x5` 之下一个悬而未决的谜。
- **HBM 带宽和 L2 不受影响。** 同卡 A/B 测得原厂 1592 GB/s 对比改装 1599 GB/s，比率 1.0 倍，在 FP32 移动 30.7 倍的同一张表中。完整 32 MB L2 和约 12.5 TIOPS 的 INT32 在原厂同样不受限制。这些共同界定节流触及的范围：FP32 FFMA、DP、DP4A 和张量 MMA 路径。

---

## 为什么 `SM_ISSUE_RATE_MODIFIER`（`0x00504204`）不是节流

这是整个领域最诱人的假线索，值得直说：**`0x00504204` 不是 CMP 节流寄存器，正式解锁从不碰它。** 对正式树做仓库级 grep `0x504204` 返回零命中。

证据：

| 观察 | 详情 |
|---|---|
| 170HX 上它读 `0x00000005` | 恰好是节流熔丝值，所以诱人 |
| A100 SXM4 40G、A100 PCIe 40G 和 80G、A10、A5000、A6000、RTX 3080 / 3080 Ti / 3090 / 3090 Ti 和 DRIVE A100 上它也读 `0x00000005` | 全速部件，同一个值 |
| 96 SM `0x20bb` GA100（每个 `FUSE_SS_*` 读 `0`）上它读 `0x00000005` | 决定性反测量，2026-07-27 |
| 它主机可写，清零后无性能变化 | 熔丝参考表中记录的空结果 |
| GA10x 对照（`0x2484`）那里读 `0x00000007` | 该值在任何部件上都不跟随节流 |
| 驱动前，170HX 在该偏移返回 `0xbadf1201` | 与全部五个相邻 SKED 寄存器相同 |

该寄存器确实有真实的 NVIDIA 侧消费者。对 GSP 固件的逆向发现一个 VA `0x01607b78` 的 init 函数，读取注册表键 `RMOverrideSmSpeedSelect`，把一个 present 标志和一个覆盖 dword 存入 GPU 配置结构，在 VA `0x01155dcc` 和 VA `0x01175a48` 到 `0x01175b2c` 的四个辅助函数消费，present 标志检查在 `0x014853e4` 和 `0x01491f34`。那个覆盖流入 PROD_DIFF 列表，最终指向 `SM_ISSUE_RATE_MODIFIER`，经 HAL 抽象到达（`0x504204` 在固件中甚至不作为字面量出现）。**名字是对的；目标寄存器错了。**

一个相关且有教益的死路：在 GSP 固件内部伪造 `speed_select` 熔丝值，让 PROD_DIFF 编程 `SM_ISSUE_RATE_MODIFIER = 0`。对 `gsp_ga10x.bin` 的十四个固件补丁加十二处 `nvidia.ko` 修改把 FFMA 从 0.3159 TFLOPS 移到 0.3146 TFLOPS，0.4% 的增量被称为测量噪声。它因两个独立原因失败：FECS 通过一个横跨 `0x20000000` 到 `0x23050000` 的 priv 窗口到达 GPU 寄存器，而 `SM_ISSUE_RATE_MODIFIER` 所在的 SM 寄存器空间（`0x20504xxx`）完全不在其中，因此即使 PROD_DIFF 列表完美 FECS 也物理上写不到它；而且 GSP-RM 本来就是 NVIDIA 签名的。

> [!NOTE]
> **开放问题：`0x00504204` 对已解锁卡施加任何残余限制吗？**
>
> 没人跑过明显的 A/B：在**SS0/SS1 已设置**的卡上把 `0x00504204` 写为零并重跑基准套件。寄存器主机可写，ROP 工具链中有写入原语，答案是或否。这是计算领域最易处理的开放问题。第二个相关未知数是 GA100 上该偏移的 `0xbadf1201` 是否意味着"权限阻止"还是"未解码"：整个 `0x00504xxx` 和 `0x00407xxx` 孔径在 170HX 上返回同一哨兵，而 GA10x 对照处处返回真实值，这指向地址解码差异而非逐寄存器块。`0x20bb` GA100 读真实 `0x00000005` 使问题复杂化。

---

## 为什么计算经得起 FLR

`0x00823804` 的 `FEAT_OVR_PLM` 位于**常开（AON）岛**。它是 26 寄存器 PLM 调查中唯一标为 AON 的 PLM，而帧缓冲几何 PLM 一个都不是。一旦打开，它跨功能级复位保持打开，经它写入的 SS0/SS1 值保持写入。

| FLR 下的行为 | 寄存器 |
|---|---|
| **存续** | SS0 `0x0082381c`、SS1 `0x00823820`、`FEAT_OVR_PLM` `0x00823804` |
| **不存续** | CFG1 `0x009a0204`、逐 FBPA CFG1、CSTATUS、LMR `0x00100ce0`、FB 几何 PLM（重新锁定）、AON LMR 影子 `0x001180f0`（恢复） |
| **FLR 清除** | SEC2 复位 PLM 污染（`0x8f` 回到 `0xff`） |

这由一次专门 FLR 存续性扫描（`plm_flr_survival_20260716.sh` 加 `fire_vram_featovr_sweep.sh`）确立，并由另一位测试者两天前独立佐证。

```bash
# Function Level Reset, as used by every unlock harness
echo 1 | sudo tee /sys/bus/pci/devices/0000:${PCI}/reset
```

**这种不对称性是计算解锁先于显存解锁发布的唯一原因。** 计算写入在常开域是粘性的；显存几何写入在第一次复位时丢失，这就是显存路径需要双加载、无 FLR 工作流的原因。

两个常被混淆的澄清：

- **寄存器本身是易失的。** 断电就丢失。正式驱动补丁改变的不是硬件行为，而是补丁模块在设备 `0x20C2` 或 `0x2082` 的**每次 GSP 引导**时重新应用整个 PLM 打开加 SS0/SS1 序列。因此面向用户的说法是"跨重启持久"，而硬件层面的"断电后什么都不存续"在其下仍然成立。
- **无驱动加载写 SS0/SS1 然后加载原厂驱动不工作。** 写入明显落盘，但原厂驱动重新锁定 PLM：`0x00823804` 回读 `0xffffff8f`，节流分频器回到 `5`。那个失败模式正是驱动内 GSP 引导路径方法存在的原因。

---

## 剩余开放问题

> [!NOTE]
> **本领域的开放问题**
>
> 1. **`0x00504204` 在解锁卡上要紧吗？** 见上方。一次 A/B 定案。
> 2. **为什么 INT8 / IMMA 仍被门控？** IMLA 熔丝读 `0x5`，覆盖半字节与 FMA 的设置相同，但实测 IMMA 不跟随。下一步：把 `0x00823818` 转储与逐数据类型微基准并列，看有效 IMLA 字段是否真的是零，并在 `SM_SPEED_SELECT` 块之外寻找单独的 DP4A/IMMA 门。
> 3. **隔离 SS1 对 FP64 的影响。** "SS1 nerfs 64-bit compute"的说法严格说是 2026-07-14 的一个未测试预测，碰巧放在正确的 FP64 测量旁边。一个移除 `0x00823820` 写入的单行构建，再跑 OpenCL FP64 测试，就能给出答案（若信念正确，预期 6.421 对比接近 0.19 TFLOPs/s）。
> 4. **解码 `FEATURE_READOUT_1`（`0x00823818`）。** 对原厂 `0x016db6ed` 做朴素九乘三位 LSB 优先解包得到 `[5,5,3,3,3,3,3,3,1]`，与熔丝不匹配（均匀 5、DP 为 1，预测 `0x01b6db6d`）。要么字段顺序或宽度假设错了，要么读出是仲裁后的有效速率。无论解码如何，`== 0` 仍是实用成功测试。
> 5. **为什么 `FUSE_SS_FMLA16 = 0x5` 看起来不节流 FP16？** 可能因为 FMLA16 管辖与打包半 CUDA 核心路径不同的张量/MLA 路径，但没人曾在同一张卡上、两种状态下分别测量 FP16 标量和 FP16 张量。
> 6. **TF32 在原厂被节流吗？** 一张表说 `90.72 → 90.09 TF/s`（未触及）；另一张、不同卡上，在 1024³、4096³ 和 8192³ 下说 `2.96 → 51.53`、`3.01 → 84.75` 和 `3.21 → 80.59 TFLOPS`。两者不可能都对。在由 `0x00823818 != 0x00000000` 确认锁定的卡上跑一次 TF32 GEMM 就能定案。
> 7. **`0x008200fc` 可写吗，冷启动时它读什么？** 一次扫描 `0xffffffff`、另一次 `0x000003ff`，九 PLM 分支尝试返回 `status=0xffff`、未记录回读。该寄存器在净室工具中叫 `FUSE_SS_PLM`、在分支源码中叫 `OPT_PLM`；它们是同一个寄存器。

---

## 相关页面

- [解锁的工作原理（端到端）](how-it-works.md)
- [SEC2 Falcon 与 Booter 原语](falcon-and-booter.md)
- [权限级掩码](privilege-level-masks.md)
- [显存几何解锁](memory-geometry.md)
- [驱动补丁](driver-patches.md)
- [完整寄存器参考](register-reference.md)
- [GA100 硅片与筛选](../hardware/ga100-silicon.md)
- [熔丝与 OTP](../hardware/fuses-and-otp.md)
- [验证流程](../procedures/verify.md)
- [性能](../operations/performance.md) 和 [调优](../operations/tuning.md)
- [词汇表](../start/glossary.md)
