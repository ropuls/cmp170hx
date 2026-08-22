# 实测性能

**本页涵盖。** CMP 170HX 所有可复现的吞吐数字：解锁前后各数据类型的计算速率、HBM 带宽、每种有人真正达到过的链路配置下的 PCIe 带宽，以及与真 A100 的全面对比。每一行都注明测试条件。调优杠杆（时钟偏移、功耗上限、配置档）在 [tuning.md](tuning.md)；推理专属数字在 [llm-inference.md](llm-inference.md)。

**头条结论。** 计算解锁是一次寄存器写入，不是时钟改动。写入 SS0 `0x0082381c` = `0x88888888` 和 SS1 `0x00823820` = `0x00000008`（先打开 `0x00823804` 的 FEAT PLM）把 FP32 从 **0.30-0.41 TFLOPS 提到 12.2-12.8 TFLOPS**，**26-32 倍**提升，时钟不变。出货解锁器没有任何代码碰核心时钟、显存时钟、功耗上限或 PCIe 链路速率，所以除非特别说明，下面每个数字都是出厂时钟下的数字。机制见 [compute-throttle.md](../unlock/compute-throttle.md)。

**证明机制的那个对照实验。** 同一张卡的 A/B 对比中，FP32 提升 30.7 倍，而显存带宽只从 **1592 GB/s 变到 1599 GB/s**，比例 1.0 倍。NVIDIA 限制的是指令发射速率，不是显存子系统。

---

## 怎么读这些数字

- **全程出厂时钟。** 持续 SM 时钟 1410 MHz（`nvidia-smi -pl 300` 下 1470 MHz），基础 1140 MHz。解锁后 `nvidia-smi` 报告的 `clocks.max.sm = 1935 MHz` 是一个*报告字段*，不是可达时钟；它是单一报告、从未复核过，而 VBIOS 表上限是 1695 MHz。把 1935 MHz 视为低置信度。
- **限流是按指令类别进行的。** 它不是全局乘数。在干净的 CMP 90HX 上，比例是 FP64 1/64、FP32 1/32、FP16 1x（未动）、INT32 1/2、INT8 1/16。170HX 遵循同样的模式，只是除数不同，这就是为什么一种数据类型看起来正常、另一种却掉 30 倍。
- **两个 INT8 数字都是真的。** tensor MMA 路径和库/OpenCL 路径差 7.6 倍。别取平均。
- **压缩是关着的。** 出货补丁 `0005-ce-scrub-workarounds.patch` 对 `0x20C2` 和 `0x2082` 强制 `*pteKind = NV_MMU_PTE_KIND_GENERIC_MEMORY`，而出厂返回 `NV_MMU_PTE_KIND_GENERIC_MEMORY_COMPRESSIBLE_DISABLE_PLC`。任何与真 A100 的带宽对比都必须注明这一点。
- **两个会产生垃圾数据的测量坑。** 用 `float` alpha/beta 指针的 `CUBLAS_COMPUTE_16F` 会立即返回并报告一个荒谬的 10748 TFLOPS：那是空操作，不是结果，所以用 fp32 累加的那些行。另外 CUDA 13 移除了 `cudaDeviceProp::clockRate` 和 `memoryClockRate`，改用 `cudaDeviceGetAttribute` 加 `cudaDevAttrClockRate` / `cudaDevAttrMemoryClockRate` 查询。

> [!WARNING]
> **冒充测量的理论峰值**
>
> 坊间流传的三个数字是工具计算的设备属性，不是运行结果。`1769.47 GB/sec` 恰好是 `864 MHz x 4 x 4096 bits / 8`；`12633.60 GFlops` 恰好是 `4480 x 2 x 1410 MHz`。两者都源自同一份 mixbench 转储中它们上方两行打印的时钟和总线宽度字段。**没有任何卡实测过 1769 GB/s。** 同一个 12633.6 GFLOPS 在另一份转储中以「4480 cores, 12.634 TFLOPs/s」再次出现，同样只是设备属性。

---

## 计算：锁定对比解锁

除注明外，所有行都在计算已解锁、出厂时钟的卡上测得。锁定列是同一块硅片在做 SS0/SS1 写入之前。

| 数据类型 / 路径 | 锁定 | 解锁 | 比例 | 条件 |
|---|---|---|---|---|
| FP32 非 tensor | 0.30 / 0.40 / 0.41 TFLOPS | 12.58 TFLOPS | 28.4x-30.7x | 1024³ / 4096³ / 8192³ 改装扫描，同一张卡 |
| FP32 SGEMM | 393 Gflop/s | 12,233-12,256 Gflop/s | 31x | gpu-burn，峰值 67 C |
| FP32（租赁批次） | 0.39 TFLOPS | 12.6 TFLOPS | 32.3x | 8 张卡的均值，驱动 610.43.02 |
| FP64 非 tensor | 0.20 TFLOPS | 6.223-6.31 TFLOPS | 约 31x | 为 FP32 的 1/2，即完整无限制 GA100 速率 |
| FP64 tensor | 197 / 191 Gflop/s（论文） | 11.6-11.96 TFLOPS | 59x-62x（论文） | DGEMM / FP64 tensor 11,668 / 11,786 Gflop/s |
| TF32 tensor | 2.96-3.21 TFLOPS | 79-94 TFLOPS | 15x-28x | 所有数据类型中跨度最大 |
| FP16 tensor | 6.01 TFLOPS | 158.7-190 TFLOPS | 约 27x | 低端为 fp32 累加 |
| BF16 tensor | 6.41 TFLOPS | 164.4-192.7 TFLOPS | 约 28x | 上限 202.1 TFLOPS |
| INT8 tensor MMA | 1.60 TOPS | 335.0-335.6 TOPS | 不适用 | 逐指令微基准 |
| INT8 库 / OpenCL | 1.60 TOPS | 43.33-47.894 TOPS | 27.0x | torch、cuBLAS、OpenCL-Benchmark |
| INT4 tensor MMA | 不适用 | 320.2 TOPS | 不适用 | `mma_s4s4s32_8_8_32` |
| FP4 / FP6 / FP8 MMA | 不支持 | 不支持 | 不适用 | sm_80 的预期行为 |

### FP32，承重的那个数字

至少十几张不同卡上、七个工具，都落在 12.2-12.8 TFLOPS 区间内。

| 值 | 工具 / 条件 |
|---|---|
| 12.72 TFLOPS | torch GEMM 8192² |
| 12.76 TFLOPS | `gemm_probe.cu` n=8192，30 次迭代 |
| 12.58 TFLOPS | 8192³ 改装扫描 |
| 12,565.14 GFLOPS | clpeak，驱动 13.0 / CUDA 13.3 |
| 12.493 TFLOPs/s | OpenCL-Benchmark，10 GB 到 40 GB 卡 |
| 12.6 TFLOPS | 八张租赁卡的均值 |
| 12,233-12,256 Gflop/s | 论文表 2，gpu-burn |
| 12,229-12,254 Gflop/s | 持续烤机，268435456 B 缓冲，24 次迭代 |
| 11.1 TFLOPS | 逐指令标量 `fma_fp32` 微基准（下限） |

70-SM GA100 在 1410 MHz 的理论峰值是 12,633.6 GFLOPS，所以这张卡达到约 99% 的算术峰值。锁定状态下的独立测量是 0.3159 TFLOPS FFMA / 0.32 / 0.39 TFLOPS / 393 Gflop/s SGEMM / 367 GFLOPS clpeak，全部落在 1/32 发射速率模型内。

### FP64 有两个速率，绝不能混为一谈

非 tensor FP64 恰好是 **FP32 的 1/2**（OpenCL 6.223 TFLOPs/s、clpeak 6308.65 GFLOPS、DGEMM 约 6,200 GFLOPS、纯标量 `fma_fp64` 微基准 5.6 TFLOPS）。FP64 **tensor** 大约是这个的 **2 倍**（11.65 TFLOPS、clpeak WMMA fp64 8x8x4 11.96 TFLOPS、八张租赁卡上 11.6 TFLOPS、论文中 11,668-11,786 Gflop/s）。1/2 的比例正是完整无限制 GA100 的速率：FP64 是真正恢复的，不是部分的。

### Tensor 吞吐细节

| 数据类型 | 值 | 工具 / 形状 |
|---|---|---|
| TF32 | 79.0 TFLOPS | 8 张租赁卡，torch GEMM |
| TF32 | 80.59 / 84.75 / 51.53 TFLOPS | 8192³ / 4096³ / 1024³ 改装扫描 |
| TF32 | 81.35 TFLOPS | torch GEMM 8192² |
| TF32 | 83.2 TFLOPS | `mma_tf32tf32f32_16_16_8` |
| TF32 | 88.9-91.9 TFLOPS | `gemm_probe.cu` n=8192 |
| TF32 | 89.69 TFLOPS | clpeak `mma.sync m16n8k8` |
| TF32 | 94,103 Gflop/s | 论文，gpu-burn，64 C |
| FP16（fp32 累加） | 158.7-160.0 TFLOPS | `gemm_probe.cu` n=8192 |
| FP16（fp32 累加） | 162.7 TFLOPS | 8 张租赁卡 |
| FP16 | 174.11 TFLOPS | 4096³ 改装扫描 |
| FP16 | 175.79 TFLOPS | torch GEMM 4096² |
| FP16（fp32 累加） | 179.1 TFLOPS | `mma_f16f16f32`，两种 tile 形状 |
| FP16（fp16 累加） | 180.2 / 180.3 TFLOPS | `mma_f16f16f16_16_16_16` / `_32_8_16` |
| FP16（fp16 累加） | 189.66 TFLOPS | clpeak `mma.sync m16n8k16` |
| BF16 | 164.4 TFLOPS | `mma_bf16bf16f32`，两种 tile 形状 |
| BF16 | 171.4 TFLOPS | 8 张租赁卡 |
| BF16 | 180.09 TFLOPS | torch GEMM 4096² |
| BF16 | 183.75 TFLOPS | 4096³ 改装扫描 |
| BF16 | 188.1-192.7 TFLOPS | `gemm_probe.cu` n=8192，fp32 累加 |
| BF16 上限 | 202.1 TFLOPS | 算术：2048 x 70 SM x 1410 MHz，已验证精确 |

两个离散没有解释，按原样记录。TF32 在七个工具间波动 19%，而 FP16 和 BF16 保持紧凑。FP16 用 fp16 累加持续读数*高于* FP16 用 fp32 累加，而在 A100 上两者应该同速；最可能的解释是 mmapeak 式微基准让操作数留在共享内存里，从而抬高了这张卡。

> [!NOTE]
> **未解决问题：INT4 测出来低于 INT8**
>
> `mma_s4s4s32_8_8_32` 返回 **320.2 TOPS**，而 INT8 是 335.0/335.6 TOPS。在 Ampere 上 INT4 tensor 吞吐应该大约是 INT8 的 2 倍。没人重跑过。INT8 这边是可靠的（两种 tile 形状一致）；INT4 这边只有一次运行。用更长的目标时间和不同的 tile 重跑 INT4 形状，是这个领域最便宜的开放线索。

> [!NOTE]
> **未解决问题：INT8 库路径比 INT8 tensor 路径低 7.6 倍**
>
> 直接 MMA 给出 335 TOPS；torch、cuBLAS 和 OpenCL 都落在 43-48 TOPS。要么这些库没有在这台设备上发射 IMMA，要么解锁给 INT8 留下了一个部分保留的发射速率限制。建议的测试是：用 INT8 输入跑一次显式 `CUBLAS_COMPUTE_32I` GEMM，对比原始 MMA 数字。

### 硅片到硅片的可复现性

来自同一家租赁的八张已解锁 64 GB 卡，在驱动 610.43.02、PCIe Gen1 x4 下单次会话基准测试，显示**每卡离散低于 2.5%**，并且全量字节比对 VRAM 完整性测试 **8/8 通过**。八张卡，一份报告。出厂卡，无电容改装，无 Gen2。

| FP16 | BF16 | TF32 | FP32（锁定） | FP64 | INT8 | HBM |
|---|---|---|---|---|---|---|
| 162.7 TFLOPS | 171.4 TFLOPS | 79.0 TFLOPS | 12.6 TFLOPS（0.39） | 11.6 TFLOPS | 44.1 TOPS | 1600 GB/s |

这是档案中证明解锁在各卡之间一致落地的最强证据。

### 解锁失败的特征

如果 BF16 回来只有约 **12 TFLOPS 而不是约 185**，说明 payload 没落地。一位用户报告 mixbench 12 BF16 TFLOPS、clpeak 367 GFLOPS、自写 GEMM 6.25 TFLOPS，被诊断为看到了「锁定模式下 tf32 的性能」——对照已知的 202 TFLOPS 上限，接受了诊断并重试。相比 202 TFLOPS 低一个数量级就是特征。见 [verify.md](../procedures/verify.md)。

还要注意：**游戏帧率不是有效的验证方法**：一位测试者测到计算解锁前后 FPS 完全相同，而 LLM 和 CUTLASS 吞吐明显变了。解锁针对的是 SM 发射速率，不是图形路径。

---

## 显存带宽

| 量 | 值 | 条件 |
|---|---|---|
| 理论峰值 | 1555.2 GB/s = 1448.4 GiB/s | 1215 MHz DDR x 5120-bit；两个数字是同一数量的不同单位 |
| 真实扫描的峰值行 | 1448 GiB/s | 共 79.3 GiB / 空闲 79.0 |
| 实测，8 张租赁 64 GB 卡 | 1600 GB/s | 八张完全一致 |
| 出厂对比改装，同一张卡 | 1592 到 1599 GB/s（1.0x） | 256 MB 工作集，解锁对照行 |
| OpenCL 对齐读 / 写 | 1305.86 / 1521.62 GB/s | 10 GB 到 40 GB 卡，驱动 580.159.03 |
| OpenCL 未对齐读 / 写 | 789.82 / 161.76 GB/s | 同一张卡 |
| 一次 OpenCL 测试 | 1333 GB/s | 出厂 VBIOS |
| 2023 外部评测上限 | 1355 GB/s | 用于其屋顶线屋脊点 |

> [!NOTE]
> **不存在唯一权威的 HBM 数字**
>
> 实测 HBM 带宽跨 **1305.86 GB/s 到 1600 GB/s**，没有一种方法论能调和这个范围。部分调和：8 GB 卡带 4 个更快的 HBM2e 堆栈（4096-bit），10 GB 卡带 5 个更慢的 HBM2 堆栈（5120-bit），而且读写/未对齐模式在*同一个工具内*就差近 10 倍。引用范围，别引用点估计。另外，「1493 GB/s 数据手册」和「1555 GB/s 满睿频理论值」是不同量（前者是 A100 40 GB PCIe 数据手册数字，后者是 1215 MHz x 5120-bit 算出来的），流传文档中经常混用。

### 8 GiB 偏移之上的 79% 平台

在一张被烧到被放弃的 80 GB 几何的 10 GB 卡上实测，1 GB memset 扫描：

| 偏移 | 带宽 | 峰值百分比 | 时间 |
|---|---|---|---|
| 0 GB | 1416 GiB/s | 98% | 0.70-0.71 ms |
| 1 GB | 1422 GiB/s | 98% | 0.70-0.71 ms |
| 2 GB | 1416 GiB/s | 98% | 0.70-0.71 ms |
| 4 GB | 1419 GiB/s | 98% | 0.70-0.71 ms |
| 8 GB 到 76 GB | 1147-1151 GiB/s（平坦） | 79% | 约 0.87 ms |

块足够大时差距完全消失：32 GB 块时，扫描在偏移 0 给出 1452 GiB/s（100%）、偏移 40 GB 给出 1443 GiB/s（100%），而 1 GB 块时是 1419 对比 1149 GiB/s。

两条出货代码的事实与之相关，但没有定案：

- `0004-bar0-pramin-clamp.patch` 在 `devId` 为 `0x20C2` 或 `0x2082` 且 `fbAddrSpaceSizeMb > 0x2000` 时，把 BAR0 窗口钉在 `(0x2000ULL << 20) - DRF_SIZE(NV_PRAMIN)`。出货代码里确实存在一个恰好在观察到的台阶偏移处的 8 GiB 不连续，但 PRAMIN 是 CPU 侧的 BAR0 窗口，而那次扫描是设备侧的 memset，所以因果关系**没有**建立。
- `0001-sec2-postbl-plm-ss-cfg.patch` 把最后一个 FB 区域扩展到 `targetFbBytes - 1`，带 `supportCompressed = NV_TRUE`、`supportISO = NV_TRUE` 和 `performance = 20`。因此资源管理器把整个解锁范围建模为均匀性能内存，没有任何办法偏好快速区域。

> [!NOTE]
> **未解决问题：平台在出货几何上存在吗？**
>
> 那次扫描只在被放弃的 80 GB 配置上跑过。在出货的 8 GB 到 64 GB 卡上重复完全相同的偏移和块扫描，就能确定这是任何解锁几何的属性，还是超规格尝试（over-fire）的产物。见 [80gb.md](../frontier/80gb.md)。

---

## 各链路配置下的 PCIe 带宽

两个独立机制，绝不混为一谈。**链路速率**（Gen1 到 Gen2）是驱动/固件改动，只存在于未发布分支。**链路宽度**（x4 到 x16）是物理板卡改装：16 条通道中有 12 条出厂没贴交流耦合电容，补上意味着手工焊 24 颗 0402 元件。见 [pcie-gen2.md](../unlock/pcie-gen2.md) 和 [physical-mods.md](physical-mods.md)。

| 链路配置 | 如何达到 | 实测带宽 | 工具 / 条件 | 置信度 |
|---|---|---|---|---|
| Gen1 x4（出厂，出货解锁） | 无需任何操作 | 发送 0.80 / 接收 0.84 / 双向 0.81 GB/s | 一张 OpenCL-Benchmark 截图，来自外部硬件组转述，10 GB 到 40 GB 卡；工具把链路打印为「Gen1 x16」 | 中 |
| Gen1 x4 | 无需任何操作 | 约 0.85 GB/s | clpeak | 高 |
| Gen1 x4，推理负载下 | 无需任何操作 | 约 1.0 GB/s，没有拉高 | 8 卡机，设备报告最大 Gen2 x16 | 高 |
| Gen1 x16 | 仅电容改装 | 2.88 GB/s（标称约 4 GB/s） | 一份报告、一张改装卡、工具未命名；测量者预期 3.2 GB/s，被告知差距是 Gen1 信号开销 | 中 |
| Gen2 x4 | 仅 `Gen2` 系列分支 | 发送 1.68 / 接收 1.71 GB/s | OpenCL-Benchmark，一张存档截图，未改装卡；安装脚本独立预测「约 0.85 到约 1.7 GB/s，正好 2 倍」 | 中 |
| 改装过电容但协商到 x8 的卡上从 Gen1 → Gen2 | 分支**加**部分电容改装 | 1.67 → 3.24 GB/s | 一次 A/B、单卡、Asus Prime Z370 / i3-8100 / 8 GB RAM。不是 Gen2 x4 数字：3.24 GB/s 高于约 2.0 GB/s 的 Gen2 x4 上限，因为链路是 x8 | 中 |
| Gen2 x4（厂商说法） | 随附包 README | 「2 GB/s」 | 没有独立日志，在出货 `master` 上不可达 | 低 |
| Gen2 x16 | 分支**加**完整 24 电容改装 | 6.63-6.67 GB/s（线速率 8 GB/s 的约 83%）；nvtop TX 7.061 GiB/s | `ocl_pcie_bw` | 中 |

> [!NOTE]
> **Gen2 x16 带宽数字**
>
> 两套机器发布过抓取记录：2026-07-26 一张卡 6.63 到 6.67 GB/s，以及四张卡各 5.97 GB/s、90 分钟零 AER 错误。没人发布过长期烤机。Gen2 本身在 `master` 里，所以这些就是装了解锁器的电容改装卡应该达到的数字。

还有一点相关：**24 颗电容只贴 12 颗会得到 x8**，因为宽度协商回退到下一个合法宽度（16、8、4、1）。改装后得到 x8 意味着焊接不完整或有桥连，不是另一个硬件限制。

### PCIe 实际让你损失多少

PCIe 敏感性取决于引擎和拓扑，两个头条结果并不冲突。

| 场景 | 变化 | 结果 |
|---|---|---|
| 单卡，llama.cpp | x4 到 x16（同代） | pp 439 到 448，tg 81.91 到 85.75（约 +2%） |
| 三卡，llama.cpp | x4 到 x16 | pp 441 到 461，tg 86 到 89 |
| 三卡，GLM-5.2，模型几乎全在 CPU 上（GPU 上只有一层加缓冲） | Gen1 x4 那次，到后来的「gen2 x4 尝试」 | pp2048 33.44 ± 0.37 到 48.22 ± 1.36 t/s（**+44.2%**），tg512 5.90 ± 0.03 到 6.39 ± 0.09，首响应时间 61,253.57 ± 675.94 ms 到 42,510.25 ± 1,217.69 ms（**-30.6%**）。两个百分比都是这里从两次单独发布的运行算出来的，不是测试者自己说的 |
| 单卡，`Qwen3.6-27B-MTP-UD-Q8_K_XL` 跑在带 MTP 的 ik_llama 上，模型完全常驻显存 | Gen1 x4 到 Gen2 x4，「其他所有因素不变」 | pp2048 328.81 到 449.41 t/s，pp8192 363.25 到 493.86，tg128 38.15 到 41.52，tg512 37.69 到 40.12。与上一行同一位测试者，两次运行相隔五天 |
| 单卡，模型加载 | Gen1 x4 | 约 30 s，一次性成本 |
| 图形（BeamNG.drive） | Gen1 x16 对比 x4，已做电容改装 | 15 fps 对比 5 fps，哪种「都很糟糕」 |

调和：单卡工作大多是带宽局部的（权重常驻，只有模型加载过链路），而多卡和 CPU 卸载的预填充受链路限制。+44.2% 那一行沿用的来源备注：Gen2 后的运行用了标着 Q4 的量化，Gen2 前那次没标，所以提示处理增量可能被高估。同一台机器在 Gen2 x4 下用 Q2_K_XL 量化给出 pp2048 49.00 ± 1.08、tg512 6.81 ± 0.06。单卡那一行让图景更复杂而不是定案：那个模型完全常驻显存，测试者和频道里任何人都解释不了为什么链路速率会动它。两行都是同一位测试者；没有独立的 Gen2 x4 推理测量。完整表格和条件见 [llm-inference.md](llm-inference.md)。

> [!NOTE]
> **未解决问题：没人测过多卡 LLM 机器上的 Gen2 x16**
>
> Gen2 x16 带宽数字和 Gen1 到 Gen2 推理运行是在不同系统上测的，从未合并。在同一台机器、同一个模型上跑 {Gen1, Gen2} x {x4, x16} 的 2x2 矩阵，就能同时了结这个和宽度对代数之争。

---

## 热学与功耗，简版

完整处理见 [thermals.md](../hardware/thermals.md)、[cooling.md](cooling.md) 和 [power-and-psu.md](power-and-psu.md)；基准测试需要知道的在这里。

| 观察 | 值 |
|---|---|
| 持续满速 GEMM 烤机 | 12,229-12,254 Gflop/s 平稳，核心在约 30-40 s 内从 62 到 64 到 69 到 71 到 73 C，零错误 |
| 峰值负载温度（论文） | FP32 67 C，FP64 tensor 和 TF32 tensor 64 C；全能力部件只在约 85 C 以上降频 |
| 默认 `gpu_burn` | 约 70 C ± 2 |
| 待机功耗 / 出厂上限 | 受控机器上约 42 W / 250 W |
| hashcat 对比 FP32 拷机下的功耗 | 160+ W 对比 60-75 W（2023，锁定卡） |

**实用规则：绝不要用常规 FP32 拷机验证稳定性或散热。** 这张卡很难被压满。整数和显存基准比 FP32 工具拉出多得多的功耗。

公认的解锁后验证配方：

```bash
# github.com/wilicc/gpu-burn
make COMPUTE=80
./gpu_burn -tc -m 90% 1200          # 20 minutes, tensor cores, 90% of VRAM
# variant used by one distributed package, expecting 0 memory errors:
./gpu_burn -m 63500 -d 30
```

报告的干净运行：调好的一张单卡 30 分钟；四张 8 GB 到 64 GB 卡各 2 小时无任何不稳定；一张 10 GB 到 40 GB 卡 5 分钟通过。先确保散热足够。

> [!CAUTION]
> **绝不要像它能工作那样去验证 80 GB 几何**
>
> 把 10 GB 卡烧到 80 GB 会产生 gpu-burn 错误，已被独立复现。10 GB 卡因此出厂就是 40 GB。见 [80gb.md](../frontier/80gb.md)。

显存超频据报道能买来约 **+2.5%**（默认 gpu_burn 平均 12,180 Gflop/s，对比持续 12,472-12,485 Gflop/s），代价是约 5 C（70 C ± 2 对比 75-77 C）。注意：存档的分支集合里没有叫 `mem_overclock` 的分支，也没有任何存档分支包含任何时钟、睿频或 p-state 代码：对全部 13 棵树 grep overclock/memclk/pstate/boost 全部为空。这个结果只能作为一份无法对照代码核实的测试者报告。

---

## 与 A100 的对比

170HX 携带**完整的 GA100 裸片**（826 mm²，`PMC_BOOT_0` = `0x170000a1`，与全部三个 A100 SKU 以及 Drive A100 相同），被筛选保留裸片 70 个 SM。所以诚实的对比是「同一架构、SM 更少、I/O 更差、没有 ECC、没有 NVLink」。

| 属性 | CMP 170HX（解锁） | A100 40 GB 参考 | 备注 |
|---|---|---|---|
| 裸片 | GA100，826 mm² | GA100，同一裸片 | 两者 `PMC_BOOT_0` 都是 `0x170000a1` |
| SM / CUDA 核心 | 70 / 4480 | 108 / 6912（A100 SXM4 40 GB） | 5 个活动 GPC、35 个 TPC |
| 计算能力 | 8.0（sm_80） | 8.0 | 相同 ISA，无 FP8，无 NVFP4 |
| L2 缓存 | 32 MB（32768 KB） | 不适用 | TechPowerUp 给 170HX 写的 8 MB 是错的，有延迟尖峰测量佐证 |
| 容量 | 64 GB（8 GB SKU）或 40 GB（10 GB SKU） | 40 GB | 见 [memory-geometry.md](../unlock/memory-geometry.md) |
| 总线宽度 | 4096-bit（8 GB，4 堆栈）/ 5120-bit（10 GB，5 堆栈） | 5120-bit | GPU-Z 在 A100-PCIE-40GB 上报告 5120-bit、1555.2 GB/s、1215 MHz 显存 |
| HBM 带宽 | 实测 1305.86-1600 GB/s | 数据手册 1493 GB/s | 八卡 1600 GB/s 高于 A100 40 GB 数据手册数字 |
| 主机链路 | 出厂 Gen1 x4；分支上 Gen2 x4；焊接后才 x16 | PCIe 4.0 x16 | 最大的一项差距 |
| NVLink / P2P | 熔断关闭，未找到杠杆；56 对 GPU 中 0 对报告对等访问 | 有 | 见 [nvlink.md](../frontier/nvlink.md)、[p2p.md](../frontier/p2p.md) |
| ECC | 熔断关闭，无遥测 | 开 | 见 [ecc.md](../frontier/ecc.md) |
| 显存压缩 | 被出货补丁强制关闭 | 开 | 影响任何带宽对比 |
| MIG | 只有 `1g.64gb` 配置存在；标准 A100 配置被拒绝 | 全套配置 | |

### 应用级对比

下面 FluidX3D 和 hashcat 行是 2023 年在**锁定**卡上测的，早于寄存器解锁存在。FluidX3D 行用了无 FMA 源码变通；hashcat 未修改直接跑，因为它是 FP32 FMA 限流够不到的整数工作。它们是档案中仅有的、指名对照 A100 的整应用对比，而且对解锁卡而言是下限而不是上限。

| 工作负载 | CMP 170HX | A100 | 比例 |
|---|---|---|---|
| FluidX3D FP32/FP32，无 FMA | 7681 MLUPs/s @ 1175 GB/s（458 steps/s） | 8526 MLUPs/s（A100 40 GB PCIe） | 90.1% |
| FluidX3D FP32/FP16S，无 FMA | 12386 MLUPs/s @ 954 GB/s（738 steps/s） | 16035 MLUPs/s | 77.2%（且比 RTX 4090 的 11091 高 +11.7%） |
| hashcat MD5 | 43930.0 MH/s（53.01 ms）@ Accel:64 Loops:512 Thr:1024 Vec:1 | 约 64900 MH/s | 67.7%（也比 RTX 3080 的 54000.1 MH/s 慢） |
| GLM-5.2 解码，8 路流水线并行 | 8 张解锁 64 GB 卡上 30.2 t/s | 流传的参考配方目标是 8x A100 80 GB 约 40 t/s | 见 [llm-inference.md](llm-inference.md) |

FluidX3D 行的说明：锁定卡上启用 FMA 时，同一内核是*计算*受限的，2276 MLUPs/s、只有 348 GB/s。降到 FP32/FP16S 把显存流量减半到 173 GB/s，吞吐仍停在 2250 MLUPs/s。带宽减半而吞吐持平，正是限流症的诊断特征。移除 FMA 后内核重新变成显存受限，达到 1175 GB/s，是该评测 1355 GB/s 上限的 87%。

单 A100 的 GLM-5.2 **55 tok/s** 数字作为对比基线流传。它是二手的、没带配置，置信度评为低。

### 屋顶线选择规则

来自 2023 年外部评测，仍是判断一个内核是否适合这张卡**锁定**状态的最佳指引：默认 FMA 下算数强度低于 **0.3 FLOPs/byte** 有用，禁用 FMA 后低于 **4.6 FLOPs/byte** 有用。这些屋脊点来自 394 GFLOPS 和 6250 GFLOPS 对 1355 GB/s 实测上限（恰好 0.291 和 4.61）。计算解锁后 FP32 屋脊大约以同样的 30 倍外移，所以该规则不再有约束力：对大多数内核，解锁卡就是一台正常的显存受限 GA100。

规则起作用的两个实例：

- 一个 0.25 FLOPs/byte 的 SYCL FDTD 内核在锁定卡上**完全不需要**变通：10110 MC/s 和 1156992 MiB/s（16777216 格 x 1000 时间步，1.66 s），约为 Radeon VII / Instinct MI50 的 6000 MC/s 的 1.5 倍，用的是未修改源码。它达到的 181.98 GFLOPS 从未接近 394 GFLOPS 的限流上限。注意 1156992 MiB/s 是二进制单位，所以看起来比别处的十进制 GB/s 数字小。
- 1.7 FLOPs/byte 的 FluidX3D（153 B 流量上 261 FP32 + 102 INT32 操作，每格更新 363 个操作）位于锁定屋脊之上，正是 FMA 变通值得用的地方。

---

## 工具与证据标准

社区对一个声称解锁的标准证据是 `ProjectPhysX/OpenCL-Benchmark` 截图。AI 写的摘要被明确拒绝作为证据。

| 工具 | 用途 | 注意 |
|---|---|---|
| `ProjectPhysX/OpenCL-Benchmark` | 解锁证明工件，全设备转储 | **完全不测 tensor core**；只读它的输出会得出「这张卡 12.5 TFLOPS」而错过约 190 TFLOPS 的 tensor 路径 |
| mixbench（CUDA） | 计算扫描 | 在测量旁边打印理论峰值；见上面的警告 |
| cuBLAS tensor 测试 / torch GEMM 扫描 | 代表性 tensor 数字 | torch GEMM 从 HBM 读，所以比 MMA 微基准更有代表性 |
| `ReinForce-II/mmapeak` | 逐指令 MMA 扫描 | 乐观：操作数留在共享内存 |
| `gemm_probe.cu` | 已发布最高的 FP32 和 BF16 数字 | 它的 TF32（88.9-91.9）低于论文的 94.1 |
| clpeak | 全设备转储 | 明确**不适合**测量 FMA/DP4A 补丁 |
| gpu-burn | 稳定性和持续 flops | 用 `make COMPUTE=80` 构建 |
| `ocl_pcie_bw` | PCIe 带宽 | Gen2 x16 数字背后的工具 |

---

## 非 LLM 工作负载结果

| 工作负载 | 结果 | 条件 |
|---|---|---|
| SDXL 1024²，30 步 | 4.73 s（6.35 it/s，10.5 GB），对比 RTX 3090 的 7.59 s（3.95 it/s）= **1.60x** | 相同脚本 |
| Wan2.1-T2V，81 帧 @ 480p | 73.4 s（0.91 s/帧，18.5 GB），对比 132.8 s = **1.81x** | 相同脚本 |
| LTX-Video，81 帧 | 11.0 s（0.14 s/帧，15.9 GB），对比 20.0 s = **1.82x** | 相同脚本 |
| Wan2.1，129 帧 @ 720p | 1,485 s 用 33.3 GB，对比 3090 的 24 GB **OOM** | 相同脚本 |
| pearlhash 挖矿 | 解锁后 3 TH 到 147 TH（约 49x，一位测试者）；wildrig 更新后 200 W 下 140-170「th」 | Pearl 网络的总算力在解锁器发布后翻了一倍 |
| Gravity 基准，5 万颗小行星 | 18 FPS | 最好报告值；解锁状态未说明 |
| FurMark | 56 fps | 显存解锁前 |

扩散是突出的非 LLM 适配场景：工作负载计算受限且完全装进显存，所以 Gen1 x4 链路根本不会咬人。

> [!NOTE]
> **未解决问题：挖矿单位无法解读**
>
> 「200 W 下 140-170 th」没有单位展开，也没有第二个每卡来源。网络级翻倍是扎实的；每卡数字照字面无法使用。

---

## 这个领域的未解决问题

1. 为什么 INT4 测出来低于 INT8。
2. 为什么 INT8 库路径比 INT8 tensor 路径低 7.6 倍。
3. 8 GiB 之上 79% 峰值带宽平台是否适用于出货的 64 GB 和 40 GB 几何。
4. 在 CMP 100-210 上看到的 `n_ubatch` 调度悬崖（关 flash attention 时 pp512 从 353.59 到 977.20，开启时 380.96 到 1159.39，一个标志带来 3.04 倍）是否也存在于 70-SM 的 170HX 上。这是档案中最便宜的未测试线索：用 `llama-bench` 把 `n_ubatch` 从 48 扫到 80。
5. 为什么 BF16 和 FP16 紧凑时 TF32 却跨 79-94 TFLOPS。
6. Gen2 x16 带宽结果能否转化为多卡 LLM 吞吐。
7. 两张 170HX 卡之间实际可用的 P2P 带宽是多少，如果有的话。

完整清单见 [open-questions.md](../frontier/open-questions.md) 和 [dead-ends.md](../history/dead-ends.md)。
