# 解锁一览

**本页涵盖：** CMP 170HX 解锁实际做了什么、不做什么，哪些部分已随正式版本发布、哪些只存在于未发布分支，两种按 SKU 区分的显存几何配置，以及后续深入阅读的入口。本页是整个解锁章节的入口。这里的每个数值都是权威值；更深入的页面提供证据。

## 核心结果

正式发布的 `cmpunlocker` 补丁集移除了 CMP 170HX 上的**两项**出厂限制，并增加一个便捷标志。它移除了 SM 速度选择（speed-select）节流，在 GA100 硅片上恢复约 30 倍的 FP32 FMA 和完整的张量吞吐，并重写帧缓冲几何，使 8 GB 卡枚举出 **65536 MiB（64 GB）**、10 GB 卡枚举出 **40960 MiB（40 GB）**。两者都在每次加载驱动时自动发生在 GSP 启动路径内，约一秒钟完成，无需焊接、无需刷 VBIOS、也无需伪造签名。

其机制是对 NVIDIA 自签名的 SEC2 Booter Load ucode 的一次纯数据攻击：驱动把 GSP 签名缓冲区从约 4 KB 扩大到 `0x0000f800`，向其中填充精心构造的 Falcon ROP 载荷，然后利用 Booter 签名校验路径中的一次无界 DMA 覆写 Falcon 的栈。这使每次 Booter 运行都能在 Heavy-Secure 权限下获得一次任意 BAR0 写入，该写入被用于打开四道权限级掩码（PLM）。之后，普通的主机寄存器写入完成实际解锁。没有开盖、没有提取密钥、也没有破坏 RSA 启动 ROM 校验。完整叙述参见 [工作原理](how-it-works.md)。

> [!CAUTION]
> **这会使一切保修失效，并可能丢失数据**
>
> 补丁过的内核模块未签名，因此必须关闭 Secure Boot，且内核处于 tainted 状态。对解锁后的卡超频会在不崩溃的情况下静默破坏内存。10 GB 卡上的 80 GB 配置会报告其无法可靠提供的容量。在运行超出标准配置的任何操作之前，请阅读 [风险](../start/risks.md) 和 [调优](../operations/tuning.md)。

## 状态表

| 能力 | 状态 | 所在位置 | 机制 |
|---|---|---|---|
| SM 速度选择节流已移除 | **已发布，稳定** | `master`，补丁 0001 | 打开 `FEAT_OVR_PLM 0x00823804` 后写 `SS0 0x0082381c = 0x88888888`、`SS1 0x00823820 = 0x00000008` |
| 显存几何 8 GB 到 64 GB | **已发布，稳定，生产可用** | `master`，补丁 0001 | `CFG1 0x009a0204 = 0x02779000`，`LMR 0x00100ce0 = 0x0000020B` |
| 显存几何 10 GB 到 40 GB | **已发布，稳定** | `master`，补丁 0001 | `CFG1 = 0x02669000`，`LMR = 0x0000028A` |
| 向 CUDA 通告的容量 | **已发布** | `master`，补丁 0001 + 0003 | GSP static-info `fb_length` 重写加一次延迟的 PMA 区域扩展 |
| 内置持久模式 | **已发布** | `master`，补丁 0006 | 在 PCI probe 时设置 `NV_FLAG_PERSISTENT_SW_STATE`；无需守护进程 |
| PCIe 第 1 代到第 2 代（5 GT/s） | **实验性，仅分支** | `debug-gen2`、`Gen2`、`far`、`deced`；补丁 0007 / 0008 | 25 次经 Booter 路由的寄存器写入加普通 BAR0 写入，随后对上游桥执行一次 retrain |
| 超过 x4 的 PCIe 宽度 | **仅硬件** | 完全不是软件 | 24 颗手工焊接的 0402 交流耦合电容 |
| MIG（多实例 GPU） | **社区发现，未合并** | 代码树中无 | `0x820840` 的位 0；一位研究者，三份相互印证的 `nvidia-smi` 输出 |
| 10 GB 卡到 80 GB | **尝试过并已放弃** | `80` 分支 | 实际使用超过约 40 GB 即不稳定 |
| PCIe 第 3 代 / 第 4 代 | **未实现** | 无处 | `FUSE_PCIE_GEN23_DIS` 和 `FUSE_PCIE_GEN3_DIS` 均读出 `0x00000001` |
| ECC | **未实现** | 无处 | 熔断关闭，未找到杠杆；名为 `ecc` 的分支不含任何 ECC 代码 |
| NVLink | **未实现** | 无处 | 熔断禁用（`FUSE_NVLINK_DIS`）；不存在 FEAT_OVR 条目 |
| 超过 70 个 SM | **未实现** | 无处 | 每条 GPC 禁用写入路径都被闩锁，包括 HS 权限写入 |
| 对等直连（P2P） | **不存在** | 无处 | 此卡上没有 |
| 更高频率 | **不属于解锁范围** | NVML，树外 | 通过 `nvmlDeviceSetGpcClkVfOffset` 设置 GPC 时钟 VF 偏移；参见 [调优](../operations/tuning.md) |

解锁刻意不碰两件事：**时钟频率**和 **PCIe 总线速率**。频道内的权威表述是"计算限制解除，总线速率不动"，这与正式机制一致——它不写入时钟表，也不写入 PCIe 配置块。

## 两种 SKU 配置

显存几何在**运行时按 PCI 设备 ID 选择**，而非编译时选择。两种配置编译进同一个模块，`master` 上的 `driver/build.sh` 完全不改写源码。

| 项目 | 8 GB 卡 | 10 GB 卡 |
|---|---|---|
| PCI ID | `10de:20c2`（`0x20C2`） | `10de:2082`（`0x2082`） |
| 原始容量 | 8192 MiB | 10240 MiB |
| 解锁后容量 | **65536 MiB（64 GB）** | **40960 MiB（40 GB）** |
| 原始 `CFG1 0x009a0204` | `0x02449000` | `0x02449000` |
| 解锁后 `CFG1` | `0x02779000` | `0x02669000` |
| 原始 `LMR 0x00100ce0` | `0x00000208` | `0x00000288` |
| 解锁后 `LMR` | `0x0000020B` | `0x0000028A` |
| `targetFbBytes` / `fb_length` | `0x0000001000000000`（64 GiB） | `0x0000000A00000000`（40 GiB） |
| 启用的 FBPA / FBP | 16 个 FBPA，8 个 FBP | 20 个 FBPA，10 个 FBP |
| 内存总线 | 4096 位 | 5120 位 |
| `SS0` / `SS1` | `0x88888888` / `0x00000008` | 相同 |
| SM 数量 | 70（CC 8.0，4480 个 CUDA 核心） | 70（相同） |
| GPC 时钟偏移余量 | VBIOS `0x47177` / `0x47179` 保存 `freqDelta = ±1000` | 两者读出均为 0 |

> [!WARNING]
> **切勿混淆两种配置**
>
> 8 GB 变 64 GB。10 GB 变 40 GB。把 8 GB 几何套用到 10 GB 卡上是已记录的失败模式，而 10 GB 卡的 80 GB 配置经过试验后被发现不稳定。参见 [显存几何](memory-geometry.md)。

还有**第三个**设备 ID `10de:20b0`，会被 `install.sh` 的 `lspci` 扫描匹配到，但**不会**被解锁：驱动内门控 `_kgspSec2PostblTimingEnabled()` 只接受 `0x20C2` 和 `0x2082`。这类卡可以干净安装、走原始启动路径、从不触发。任何暗示解锁仅由 `0x20C2` 门控的 README 措辞都已过时。

## 已发布与实验性的区别

**正式发布的 `master`** 是应用于未修改 `open-gpu-kernel-modules` 压缩包的六个编号补丁，共 37,415 字节：

| 补丁 | 大小 | 作用 |
|---|---|---|
| `0001-sec2-postbl-plm-ss-cfg.patch` | 19,741 B | 整个解锁：签名扩大、载荷、PLM 循环、寄存器写入、签名重建、static-info 重写 |
| `0002-booter-verify.patch` | 3,988 B | 把四个致命断言降级，并增加 `POST-BooterLoad verify` 回读 |
| `0003-late-pma.patch` | 10,580 B | 把 8 GiB 以上的帧缓冲注册到 PMA，使其可分配 |
| `0004-bar0-pramin-clamp.patch` | 861 B | 让 PRAMIN 窗口保持在可达的 BAR0 空间内 |
| `0005-ce-scrub-workarounds.patch` | 1,642 B | 强制复制引擎的 scrubber 进入物理模式 |
| `0006-persistent-sw-state.patch` | 603 B | 设置持久软件状态标志 |

`master` 上支持的驱动版本恰好是 **`610.43.03`（默认）和 `610.43.02`**，按精确字符串匹配；其他任何版本构建都会硬性失败。仅限 Linux、仅限 nvidia-open、Secure Boot 关闭。参见 [驱动版本](../procedures/driver-versions.md) 和 [安装](../procedures/install.md)。

存在**十二个未发布的分支快照**（连同 `master` 共十三个树）：`80`、`Gen2`、`PG199`、`clanker_driver-port`、`debug-gen2`、`deced`、`docs`、`ecc`、`far`、`housekeeping`、`memory`、`multiple-cards`。

> [!WARNING]
> **实验性**
>
> **PCIe Gen2** 只随 `Gen2` 家族发布。补丁 `0007-pcie-gen2.patch` 存在于 `debug-gen2`、`Gen2`、`far` 和 `deced`；`0008-pcie-gen2-probe-retrain.patch` 存在于 `Gen2`、`far` 和 `deced`。Gen2 家族的 PLM 表从四个条目增加到九个。Gen2 不确定、在虚拟机直通下不工作，且四个分支中的两个（`Gen2` 和 `debug-gen2`）把 `RMPcieLinkSpeed` 设为 Gen1 枚举值 `0x1`，而 `far` 和 `deced` 设为 `0x2`；至今没有一次 A/B 启动测试能确定哪个值正确。参见 [PCIe Gen2](pcie-gen2.md)。

> [!WARNING]
> **实验性**
>
> **`clanker_driver-port`** 分支增加了 `580/`、`590/`、`595/` 和 `610/` 补丁目录。每个寄存器值和载荷偏移都与 `master` 逐字符相同，`610` 目录是它的逐字节副本。595 / 590 / 580 移植版**仅做过源码验证**：补丁能干净应用，但没有人报告过成功启动。

> [!WARNING]
> **`memory` 分支是单设备，且硬编码 8 GB 配置**
>
> `memory` 早于双几何支持。它的补丁 `0001` 硬编码了 `SEC2_POSTBL_TIMING_CMP_170HX_PCI_DEVICE_ID 0x20C2`、`cfg1Value = 0x02779000U` 和 `lmrValue = 0x0000020BU`，没有设备 ID 分支，也没有 10 GB 路径：`0x2082` 卡在该分支上完全不会被解锁。不要指望从它构建出运行时配置选择。

`ecc` 分支不含任何 ECC 实现：六个驱动补丁与 `master` 逐字节相同。`docs` 分支的 `ARCHITECTURE.md` 是文档缺陷，不应引用：它把 SS0/SS1 称为"Suspension State"寄存器，声称两者都被写成 `0xffffffff`，把 PLM 展开为"Program Logic Modules"，并打印代码中根本不存在的日志行。

## 解锁实际买到什么（实测）

日期为 2026-07-06 的行来自第一份私有"计算解锁成功"报告中发布的一张渲染图，而非具名工具输出。出处说明见 [compute-throttle.md](compute-throttle.md)。

| 指标 | 锁定 | 解锁 | 备注 |
|---|---|---|---|
| FP32 IEEE（2026-07-06，单卡） | 0.41 TF/s | 12.69 TF/s | 31.0 倍；理论上限 12.63 TFLOPS（4480 x 2 x 1410 MHz） |
| FP64 非张量 | 0.20 TF/s | 6.2-6.31 TF/s | FP32 的 1/2，即完整 GA100 速率 |
| FP64 张量（DMMA） | 无 | 11.5-12.9 TF/s | 约为非张量速率的 2 倍。两个 FP64 数字并不矛盾：一次 clpeak 运行并排打印了 `double : 6308.65` GFLOPS 和 `wmma_fp64 : 11.96` TFLOPS |
| BF16 张量（2026-07-27，8 张租用卡） | 6.40 TF/s | 164.4-192.7 TF/s | 跨八张租用卡加一台调优过的参考机 |
| FP16 张量（2026-07-27，8 张租用卡） | 6.52 TF/s | 158.7-190 TF/s | |
| INT8（2026-07-06，单卡） | 1.63 TOP/s | 50.50 TOP/s | 30.9 倍。后来一轮 8 张租用卡测试（2026-07-27）在库路径上测得 44.1 TOPS，但没有对应的锁定基线；INT8 *张量*路径测得 335 TOPS |
| FP16 标量（非张量） | 约 42-50 TFLOPS | 不变 | 从未被节流，这就是锁定卡原本已可用于 token 生成的原因 |
| INT32 | 约 12.5 TIOPS | 不变 | 从未被节流 |
| HBM 带宽 | 1305.86-1600 GB/s | 不变 | 是跨工具和访问模式的区间，不是单一数值 |
| 持续 SM 频率 | 1410 MHz | 不变 | `-pl 300` 下为 1470 MHz；`clocks.max.sm = 1935 MHz` 只是被报告的字段，置信度低 |

成功信号是 `0x00823818` 处的 `FEAT_READOUT_1` 读出 `0x00000000`。原厂 170HX 读出 `0x016db6ed`。这个单一寄存器是现有最干净的"此卡是否已解锁"测试。

> [!NOTE]
> **开放问题**
>
> 解锁后 INT8 / IMMA 仍然被门控，尽管 IMLA 覆盖半字节与 FMLA 和 FFMA 的设置完全相同。在 A100 上，INT8 比 FP16 快约 2 倍；在解锁的 170HX 上则慢 3.7 倍。对推理的实际影响：使用 W4A16（AWQ、GPTQ），完全避开 W8A8。参见 [LLM 推理](../operations/llm-inference.md)。

## 它为什么能生效

三点事实支撑一切：

1. **主熔断丝（master kill fuse）未熔断。** CMP 170HX 上 `0x008203f0` 处的 `OPT_FEATURE_FUSES_OVERRIDE_DISABLE` 读出 `0x00000000`。如果它已熔断，所有功能覆盖都会被永久锁定，任何软件路径都不存在。
2. **GA100 加载 Turing 代固件。** GSP 镜像是 `gsp_tu10x.bin`，SEC2 booter 是 Turing 血统的 `booter_load`，它带有无界 DMA 漏洞。GA100 的 `booter_load` 二进制在驱动分支 580 到 610 之间逐位相同。
3. **调试版和生产版 booter 镜像包含相同的明文代码。** 只有 AES 密钥不同，而调试密钥是编号的非机密测试密钥，因此无需任何泄露源码就能读取生产版 HS 代码。

该漏洞利用是对厂商签名 blob 的一次纯数据攻击，其验证密钥被熔断进不可变的硅片中，因此易受攻击的 booter 无法通过驱动更新撤销。这是对信任模型的推断，而非演示过的结论。

## 深入页面地图

| 页面 | 涵盖内容 |
|---|---|
| [工作原理](how-it-works.md) | 按启动顺序的完整端到端机制，以及为什么每一步都必要 |
| [Falcon 与 Booter](falcon-and-booter.md) | SEC2 硬件接口、booter 提取与解密、内部结构、漏洞 |
| [ROP 链](rop-chain.md) | 载荷布局、gadget、栈金丝雀破解、终结符、写入预算 |
| [权限级掩码](privilege-level-masks.md) | PLM 是什么、四项正式表、九项 Gen2 表、FLR 后存续 |
| [显存几何](memory-geometry.md) | CFG1 与 LMR 编码、逐 FBPA 传播、为什么 LMR 必需、80 GB 之墙 |
| [计算节流](compute-throttle.md) | SS0/SS1 语义、速度选择熔丝、门控链、实测吞吐 |
| [驱动补丁](driver-patches.md) | 六个补丁逐 hunk 解析、`install.sh`、`build.sh`、`remove.sh`、版本移植 |
| [PCIe Gen2](pcie-gen2.md) | 补丁 0007 与 0008、`xp3gTable`、retrain、分支历史 |
| [寄存器参考](register-reference.md) | 每个地址、每个实测值，按 SKU 和按对照部件 |

相关材料：[PCIe 子系统](../hardware/pcie-subsystem.md) 讲宽度上限和电容改装，[物理改装](../operations/physical-mods.md) 讲焊接本身，[验证](../procedures/verify.md) 讲确认解锁成功，[故障排查](../procedures/troubleshooting.md) 讲未触发时怎么办，[状态板](../frontier/status-board.md) 讲仍未解决的问题。
