# 调优已解锁的卡

**本页涵盖。** 把已解锁的 CMP 170HX 榨出最大价值：时钟和唯一好用的超频手段、功耗上限、持久模式、性能状态管理、实际能分配多少显存，以及工作负载级调优。实测基线吞吐在 [performance.md](performance.md)；推理专属调优在 [llm-inference.md](llm-inference.md)；散热硬件在 [cooling.md](cooling.md)。

**塑造这里一切的三个事实。**

1. **唯一好用的超频手段是通过 NVML 的 GPC 时钟 VF 偏移**（`nvmlDeviceSetGpcClkVfOffset`，范围 `[-1000 .. +1000]` MHz）。显存时钟 VF 偏移范围是 `[0 .. 0]`，驱动拒绝它。610.43.03 上的 `nvidia-smi` 只暴露负方向（`--set-vf-derate`），让超频看起来不可用，但 NVML 导出完整 API。
2. **这个偏移本质上是把降压表达成频率偏移。** 在钉住 1350 MHz SM 时钟时，从 +0 到 +300 把功耗从 **174.6 W 砍到 132.0 W（-24.4%）**，吞吐完全相同（179.7 对比 180.7 TFLOPS BF16）。调优这张卡最大的价值是省下的功耗，不是多出来的吞吐。
3. **超过约 +300 MHz，失败模式是静默数据损坏，不是崩溃。** 一次跑完并不代表设置是安全的。

> [!CAUTION]
> **1400 MHz 上限下超过 +300 MHz 偏移会出现静默显存损坏**
>
> 以全显存模式扫描作为把关，实测：**+250** @ 142.2 W 三次扫描 0 错误通过；**+300** @ 138.5 W 四次扫描 0 错误通过；**+325** @ 132.7 W 三次扫描给出 **6 个错误、然后 3 个、然后 0 个**；**+375** 在负载下发生 CUDA 设备故障。1400 MHz 上限下的安全窗口只比 +300 **宽一个 25 MHz 步**，过了它你得到的是坏数据而不是堆栈回溯。跑四次扫描而不是两次，两个设置测出来相等时优先留余量。

---

## 时钟

| 量 | 值 | 备注 |
|---|---|---|
| 基础时钟 | 1140 MHz | 禁用 GSP 时 CPU-RM 也跑在这个频率 |
| 出厂持续 SM 时钟 | **1410 MHz** | 权威值；所有持续测量都落在这里 |
| `-pl 300` 下的持续 SM 时钟 | **1470 MHz** | 300 W 需要 OC VBIOS，见下文 |
| 持续 SM 时钟，调优参考卡 | +0 时约 1425 MHz，+150 时约 1571 MHz，+300 时约 1647 MHz | 单卡 |
| VBIOS 表最大图形时钟 | 1695 MHz | 参考卡，VBIOS 92.00.6D.00.0A |
| 实际硅片上限 | +350 偏移时约 1604-1614 MHz | 交付的实际工作，不是报告数字 |
| `nvidia-smi` 报告的 `clocks.max.sm` | 1935 MHz | **只是报告字段，低置信度，不是运行时钟**；单一报告，从未复核 |
| 图形时钟步进 | 100 档，从 1695 MHz 以 15 MHz 递减到 210 MHz | `nvidia-smi -q`，580.159.04 / CUDA 13.0 |
| 显存时钟域 | 恰好**一个**条目，`Supported Clocks / Memory: 1728 MHz` | 没有可选的。*出厂* 8 GB 显存时钟未定论，见下面的框 |
| 核心时钟下限 | 210 MHz | 不会再低，这是待机功耗居高不下的原因之一 |

10 GB 卡**完全没有时钟余量**：`nvidia-smi -q` 在抓取时报告 Graphics 1140 MHz、SM 1140 MHz、Memory 1215 MHz，Max Clocks 为 Graphics 1410 MHz、SM 1410 MHz、Memory 1215 MHz（当前等于最大）。10 GB 卡在解锁后核心时钟偏移锁和显存时钟锁仍然保留。

> [!NOTE]
> **未解决问题：出厂 8 GB 显存时钟未定论**
>
> 出厂 8 GB 显存时钟没有定论：1458 MHz（一次扫描和 TechPowerUp）、1728 MHz（`nvidia-smi -q` Supported Clocks，标注为「432 MHz x 4」）、1890 MHz（解锁 64 GB 后在 300 W 下跑 `gpu_burn` 时的 `nvtop`）。1215 MHz 是 10 GB 卡的，是可靠的。那个看似合理的调和方案（1458 出厂、1728 OC VBIOS、1890 超频 OC VBIOS）未被证实；一次原始的 FBPA PLL 读取就能定案。本页因此在不同地方分别印出 1728、1890 和 1215。

> [!NOTE]
> **未解决问题：8 GB OC VBIOS 上的 1470 MHz 上限无法解释**
>
> VBIOS 92.00.6D.00.0A 宣告最大客户睿频时钟为 graphics/SM 1695 MHz、memory 1728 MHz（标注为 432 MHz x 4）、video 1545 MHz。在 `mmapeak` 下，卡停在 **1470 MHz**，功耗上限 300 W，GPU-T 报告 `PerfCap: None`，GPU 只拉约 150 W。功耗和显式性能上限都解释不了。

显存时钟锁定直接被拒绝：

```console
$ nvidia-smi -lmc 1000
Setting locked Memory clocks is not supported for GPU 00000000:21:00.0.
```

那条路径上只有功耗上限和 GPC VF 偏移可调。

---

## 超频 / 降压手段

### 经过验证的调优配置档

下面所有行来自**同一张参考卡**（64 GB、驱动 610.43.03、VBIOS 92.00.6D.00.0A、PCIe Gen2 x4）。条件：持续 BF16 tensor GEMM，n=8192，10-15 秒稳定期，进程内采样 NVML 功耗。每一行都至少两次以 `mem_errors=0` 通过全显存模式扫描。

| 配置 | 偏移 | 时钟上限 | BF16 TFLOPS | 功耗 | GFLOPS/W | 对比出厂 |
|---|---|---|---|---|---|---|
| stock | +0 | 无 | 184.3 | 199.2 W | 925 | 基线 |
| dense | +250 | 1200 MHz | 160.8 | 120.2 W | 1337 | -13% 性能 / -40% 功耗 |
| **eff（默认）** | **+250** | **1350 MHz** | **180.3** | **132.0 W** | **1366** | **-2% 性能 / -34% 功耗** |
| match | +250 | 1400 MHz | 186.5 | 142.2 W | 1311 | 出厂吞吐，-29% 功耗 |
| balanced | +300 | 1470 MHz | 196.2 | 149.7 W | 1311 | +6% / -25% |
| perf | +350 | 1590 MHz | 212.2 | 181.2 W | 1171 | +15% / -9% |
| max | +350 | 1650 MHz | 215.3 | 186.1 W | 1157 | 出厂功耗下 +17% |

1400 MHz 上限下最高的*已验证*点是 **+250 到 +300**（+300 四次扫描 0 错误通过）。配套的 GFLOP/W 表里 1390 GFLOP/W 峰值在 1400/+350，从未经过扫描验证，夹在两次失败之间（+325 损坏、+375 故障）。1350/+300 的 1376 GFLOP/W 格点同类：一次跑完，从未被模式扫描把关。同一把 1350 梯子还包含 1350/+400，它两次扫描通过，后来一次返回 `mem_errors=1`。最高的**扫描验证**能效点是出货 `eff` 配置：**1366 GFLOP/W，+250 / 1350 MHz**（180.3 TFLOPS，132.0 W）。1650 上限下的能效从 +250 的 1067 GFLOP/W 到 +350 的约 1149 GFLOP/W；更高的数字只来自已经故障的偏移。`eff` 配置成为出厂默认，是因为它用三分之一的功耗只换来 2% 的吞吐损失。

### 电压地板

在 1350 MHz 上限下，+250 以上一切功耗相同，所以正确选择是到达地板时**最低**的偏移：

| 偏移 | +150 | +200 | +250 | +300 | +350 | +400 | +450 |
|---|---|---|---|---|---|---|---|
| 功耗 | 146.0 W | 140.9 W | **132.4 W** | 131.3 W | 132.1 W | 131.5 W | 132.5 W |

1350 到 1400 MHz 之间没有更好的地板点：`+400/1380` 和 `+375/1395` 都在第一次运行就故障。这个 SKU 完全不报告电压遥测（`nvidia-smi -q -d VOLTAGE` 是空的），所以固定时钟下的瓦数是对电压唯一可用的代理。

### 故障与挂死边界

同一张参考卡。**+350 是 1650 MHz 上限下经过验证的最高偏移。** 安全偏移取决于上限：1400 MHz 上限下经过验证的最高偏移是 **+300**，因为 1400/+325 会静默损坏。

| 时钟上限 | 偏移 | TFLOPS | 功耗 | 结果 |
|---|---|---|---|---|
| 1650 | +350 | 214.7 | 187 W | 干净，2 次扫描加一次 selftest PASS（最高已验证） |
| 1650 | +355 | 215.0 | 183 W | 第三次运行故障，`illegal instruction` |
| 1650 | +360 | 217.3 | 182 W | 故障 |
| 1650 | +375 | 219.3 | 182 W | 这张卡上的最佳单次结果，然后下一次扫描故障加 1 个显存错误 |
| 1650 | +400 | 210.7 | 179 W | 干净但**更慢** |
| 1590 | +400 | 不适用 | 不适用 | HANG |
| 1700 | +375 | 不适用 | 不适用 | HANG，「GPU requires reset」，需要断电重启 |
| 任意 | +450 | 不适用 | 不适用 | 硬崩溃；热重启不总是够 |

观察到的故障字符串：`illegal instruction`、`illegal memory access`、`misaligned address`、`cublas 14`。

**为什么 +400 跑基准会比 +375 慢：** Ampere 的 NAFLL 有跌落检测，电压不足时会拉伸时钟。在 +400，请求的 VF 点离曲线足够远，拉伸器持续介入，所以时钟*读起来*是 1650 MHz，而交付的工作比 +375 低约 4%。在约 +355 到 +390 之间，部件以完整请求速度、过小的余量运行，间歇性故障正好住在这里。**偏移更高却跑得更慢是警告信号，不是胜利。**

### 合格化阶梯

每张卡的硅片差异足够大，一张卡上验证过的偏移不能假设到另一张上。有记录的流程：

1. 跑 `sudo nvml_oc`，确认 GPC 范围**不是** `[0..0]`。（这也兼作「卡确实解锁了」的最快测试。）
2. `sudo 170hx-oc stock`，然后 `sudo oc_eff 10` 拿基线。
3. 用 `oc_eff` 从 +150 阶梯上到 +300，每步都跑。
4. 在候选点跑全显存内存扫描并计算校验和（`170hx-test.sh --no-unlock`）。
5. 在第一次设备故障处停下，**回退整整一步，不是一个 bin**。
6. 逐卡记录结果。

### 真实工作负载的增益

超频在真实工作负载上买来的远少于 GEMM 微基准。llama.cpp 开 MTP 跑短聊天：

| 模型 | 设置 | 解码 | 功耗 | 核心时钟 |
|---|---|---|---|---|
| Qwen3.6-35B-A3B-UD-Q8_K_XL | 出厂 | 130 t/s | 170 W | 1445 MHz |
| Qwen3.6-35B-A3B-UD-Q8_K_XL | +200 偏移 | 144 t/s | 185 W | 1650 MHz |
| Qwen3.6-27B-UD-Q8_K_XL | 出厂 | 55 t/s | 268 W | 1390 MHz |
| Qwen3.6-27B-UD-Q8_K_XL | +200 偏移 | 59 t/s | 287 W | 1565 MHz |

那是 **7-11% 的 token 生成增益**。第二位测试者独立报告 +200 MHz 是他们自己的稳定上限。

### 8 GB 卡能超频；10 GB 卡不能

| 步骤 | 时钟 | FP32 |
|---|---|---|
| 会话开始 | 1410 MHz | 12.08 TF/s |
| `nvidia-smi -pl 300` | 1470 MHz | 12.99 TF/s |
| 偏移 +60 | 1515 MHz | 13.40 TF/s |
| 偏移 +225 | 1695 MHz | **14.97 TF/s（+24%）** |

这在 8 GB 部件上可行，是因为那里的 VBIOS 条目 `0x47177` / `0x47179` 持有 `freqDelta = +/-1000`。两者在 A100 和 CMP 10 GB 上都读作 0。见 [vbios.md](../hardware/vbios.md)。

---

## 功耗上限

| 量 | 值 | 来源 |
|---|---|---|
| 默认 / 当前 / 请求功耗上限 | 250.00 W | `nvidia-smi -q`，解锁卡，驱动 610.43.02 |
| 最小功耗上限 | 100.00 W | 同一抓取 |
| 出厂 CMP VBIOS 的最大功耗上限 | **250.00 W** | 同一抓取 |
| NVIDIA 300 W「OC 挖矿」VBIOS 的最大功耗上限 | **300 W** | 30 分钟 `gpu_burn` 下观察到 `POW 278 / 300 W` |
| 插槽功耗上限（DevCap） | 75 W | 这张卡需要它的 EPS 接口 |
| 电源接口 | 1 x EPS 8-pin（额定 300 W），需要 2 x PCIe 转 EPS 转接线 | 见 [power-and-psu.md](power-and-psu.md) |

所以在出厂固件下 `nvidia-smi -pl` 只能**下调**，范围 100 W 到 250 W。没有 OC VBIOS 时出厂之上没有余量，而且那个 VBIOS 是 8 GB 卡的故事：显存解锁后，10 GB 卡被确认仍带核心时钟偏移锁和显存时钟锁，锁死在 1215 MHz。档案中没有任何人在 10 GB 卡上验证过 300 W VBIOS 与解锁的组合。

```bash
nvidia-smi -pl 160          # works; documented uses span 100/150/160/175/200/250 (and 300 on the OC VBIOS)
nvidia-smi -q -d POWER      # confirm Current / Min / Max / Default
```

反复出现的「这些卡没法限制功耗」的说法是**错的**。

### 抬高上限有用吗？

几乎没用。在同一测试者自己的 250 W 基线上，带更快显存 VBIOS 和大涡轮风扇的卡升到 300 W 后，BF16 从 **约 180 变到 185 TFLOPS（约 +2.8%）**，而且散热不是限制因素（核心和显存都低于 65 C）。得出的结论是：核心就是不想跑得更高。

反方向几乎免费。限到 **150 W 在原始吞吐压力测试中没有可测的吞吐损失**（单一来源，仅限该类负载），而上面整个 `eff` 配置之所以存在，就是因为功耗曲线急剧递减。hashcat DES 中，超频 VBIOS 卡在 190 W 给出 1800 MHash，出厂卡在 150 W 给出 1700 MHash：功耗 +26.7% 换性能 +5.9%，功耗增长速度约为性能的 4.5 倍。硅片漏电流随温度上升，热的时候曲线更糟。

### 这张卡实际拉多少

| 工作负载 | 功耗 |
|---|---|
| 待机 | 27-46 W，取决于卡、温度和驻留状态；运行中的机器上典型约 42 W |
| 显存常驻模型的待机 | 从约 33 W 升到约 45 W（常驻 CUDA 上下文会抬高时钟） |
| `gpu_burn` FP32 / FP64 | 约 60 W |
| 带 tensor core 的 `gpu_burn` | 约 75 W，尖峰超过 100 W |
| CUTLASS BF16（形状优化） | 锁定卡峰值 186 W；解锁后 `mmapeak` 在 1470 MHz 只有约 150 W |
| hashcat（纯整数） | 160+ W |
| STREAM 式显存基准 | 160+ W |
| FluidX3D 禁用 FMA，FP32/FP16S | 180 W |
| llama.cpp，稳定 | 230-240 W |
| 扩散 | 250-260+ W |
| 300 W 上限下的 `gpu_burn` | 278 / 300 W |

> [!WARNING]
> **绝不要用 FP32 拷机验证散热或稳定性**
>
> 这张卡很难被压满。常规 FP32 拷机只到 60-75 W，而整数或显存基准能到 160+ W。用 hashcat、显存扫描、或 `gpu_burn -tc` 加真实负载验证，不要只用 FP32。

**把卡散得更凉会降低它的待机功耗**，这是漏电流反馈环里良性的一半，也最可能是不同测试者间 30 W 对 44 W 待机离散的解释。

对机架来说：20 张卡各约 30 W 待机，光是放着就约 600-700 W。一套六卡 llama.cpp 分层切分系统总功耗约 **600 W**，远低于 6 x 250 W，因为按层和流水线切分不会让所有 GPU 同时满载。主机平台选择主导：双路 Xeon 6200 加 Optane PMem 200 和 1.2 TB 内存待机 400-600 W，双路 EPYC 7713 加 1 TB DDR4 约 200-250 W，单路 EPYC 7D12 整机 80 W，EPYC 7261 单条内存 30 W。

---

## 持久模式和让设置存活

超频和功耗设置是**易失的**：每次开机和每次驱动重载后都要重新应用。参考部署用 systemd oneshot：

```ini
[Unit]
After=nvidia-persistenced.service gen2-hammer.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/170hx-oc eff
ExecStop=/usr/local/bin/170hx-oc stock
```

应用器以 PCI 设备 ID `0x20C2` 为守卫并遍历每张 GPU，所以插槽里非 170HX 的卡会被跳过并记日志而不是被超频。一条有代表性的日志行：

```text
170hx-oc: GPU 0 (…) profile=eff offset=+250 clk_max=1350 power_limit=300 W
```

关于 `nvidia-persistenced` 本身的说明：

- 打过补丁的模块对两个设备 ID 都设置 `NV_FLAG_PERSISTENT_SW_STATE`（`0006-persistent-sw-state.patch`），所以最后一个客户端关闭时 RM 不会拆掉软件状态。这实际上就是内置持久化，也是为什么不需要守护进程；不过新卡上 `nvidia-smi -q` 仍报告 `Persistence Mode: Disabled`。
- **解锁脚本要求停止所有 NVIDIA 服务。** `build.sh` 在热重载流程中会停掉 `nvidia-persistenced`。可靠地拆掉驱动意味着停掉显示管理器和持久化守护进程，不只是 `modprobe -r`。见 [troubleshooting.md](../procedures/troubleshooting.md)。

> [!CAUTION]
> **不要在解锁主机上把 `nvidia-pstated` 装成 systemd 服务**
>
> 解锁脚本需要杀掉每个 NVIDIA 服务，与 pstate 守护进程的交互未经过测试。如果想实验，用启动器运行。

---

## 性能状态（pstate）管理

**170HX 只暴露 P0。** `NvAPI_GPU_SetForcePstate` 返回 `NVAPI_ERROR`，社区里那个在双 P-state 卡（P100、V100）上有效的 `nvidia-pstated` fork 在 170HX 上试过，**没有任何变化**。该守护进程的默认值，供参考：`iterationsBeforeSwitch = 30`、`performanceStateHigh = 16`、`performanceStateLow = 8`、`sleepInterval = 100`、`temperatureThreshold = 80`。

这是这张卡的问题，不是工具的故障：`nvidia-pstated` 能把 **CMP 90HX 待机从 75 W 降到 5 W**，跨多 GPU 配置工作，重启后保持。

> [!NOTE]
> **未解决问题：一个还是两个 P-state？**
>
> 一种说法称 170HX 有两个 P-state；每一份已发布的抓取都只显示一个默认 P0，这正是 NvAPI 调用失败的原因。无论哪种，实际后果相同。`nvidia-smi -q -d PERFORMANCE` 列出受支持的 P-state 集合就能定案。

待机功耗唯一没试过的线索是刷 PCIe A100 逻辑，那里确实有几个 P-state。没人试过，而且这张卡的 VBIOS 工作受签名约束；见 [vbios.md](../hardware/vbios.md)。

---

## 显存分配上限与实用可用显存

| 卡 | 出厂 | 解锁 | 工具报告什么 |
|---|---|---|---|
| 8 GB（`10de:20c2`） | 8192 MiB | **65536 MiB（64 GB）** | `nvidia-smi` 65536 MiB；`gpu_burn`「Initialized device 0 with 65052 MB of memory (64733 MB available, using 58259 MB of it)」 |
| 10 GB（`10de:2082`） | 10240 MiB | **40960 MiB（40 GB）** | 一次抓取报告 40459 MB；一次受控运行用了 40960 MiB 中的 17464 MiB |
| 烧到 80 GB 的 10 GB | 不适用 | 报告约 81920 MiB / 79.7 GiB | **约 40 GB 以上不可用** |

实用分配指引：

- 给驱动和上下文开销预算 64 GB 中约 **1 GB**：上面的 gpu_burn 抓取显示工具拿 90% 之前总 65052 MB、可用 64733 MB。
- **vLLM**：8 卡 GLM-5.2 配方用 `--gpu-memory-utilization 0.90`，实际达到 0.92 利用率，得到 438,107 tokens 的 KV。把利用率保持在 0.90 或以下：0.95 崩过一张卡，因为解锁几何暴露 65052 MB，实际只有 64733 MB 可用，0.95 时余量很薄。配方还设置 `VLLM_MEMORY_PROFILER_ESTIMATE_CUDAGRAPHS=0`。
- **llama.cpp**：4 卡机器上观察到的稳态驻留是 53G/64G、60G/64G、60G/64G 和 56G/64G，提示缓存 8192.000 MiB / 65536 tokens。
- RM 把整个解锁范围建模为**均匀性能内存**（扩展区域 `supportCompressed = NV_TRUE`、`supportISO = NV_TRUE`、`performance = 20`），所以分配策略无法偏好快速区域，即使存在。8 GiB 偏移之上实测的带宽台阶（之下是峰值的 98%，之上平坦 79%，32 GB 块时完全消失）对分配器不可见。见 [performance.md](performance.md)。
- 出货 `master` 把两个设备 ID 的 BAR0/PRAMIN 窗口都夹在 8 GiB（`0004-bar0-pramin-clamp.patch`）。这是 CPU 侧窗口，不是设备侧分配限制，但它是基于 PRAMIN 的工具只看到前 8 GiB 的原因。
- 安装器的**配置自动检测**读 `nvidia-smi --query-gpu=memory.total`，把 `>= 60000 MiB` 映射到 8 GB 配置、`35000-59999 MiB` 映射到 10 GB 配置，所以已解锁的卡能正确重检测。出厂窗口是 7680-8704 MiB 和 9728-10752 MiB，带 ±512 MiB 的保留 FB 容差。

> [!CAUTION]
> **80 GB 几何不是更多显存，是更少**
>
> 烧到 80 GB 的 10 GB 卡报告约 81920 MiB 和 85,545,582,592 字节，`cudaMalloc` 77 GiB 甚至能成功，但接触超过约 40 GB 的内核会造成致命的 GPU 丢失，与功耗上限无关。报告的 Xid 码包括 Xid 31（被描述为无害）和 CUDA 显存测试后的 Xid 154；主导的报告症状是挂死。Xid 31 的说法来自一位旁观者，并未被持有故障卡的操作者证实为*那个*特征。一位测试者模型加载在约 20 GB 后卡住，另一位带多张卡的测试者在 40 到 60 GB 区间看到失败；无论哪种，之前能装进 40 GB 解锁的模型都开始加载不了。物理 DRAM 是存在的（一次 PRAMIN 遍历证明有 80 个不同的 GiB），在这个分支上这堵墙表现得像地址解码。脚本驱动的连贯寄存器组确实能到达 40 GiB 之后的真实内存，但它没有出货，而且大约每次烧写只能交付一个 CUDA 上下文。**8 GB 卡去 64 GB；10 GB 卡去 40 GB。** 见 [80gb.md](../frontier/80gb.md) 和 [memory-geometry.md](../unlock/memory-geometry.md)。

> [!WARNING]
> **没有 ECC，也没有 ECC 遥测**
>
> ECC 已熔断关闭且没有已知杠杆，所以边缘超频没有错误计数器的安全网。这正是上面的合格化阶梯要以带计算校验和的全显存模式扫描把关的原因。见 [ecc.md](../frontier/ecc.md)。

---

## 工作负载调优

### 重要的构建和启动参数

| 设置 | 值 | 为什么 |
|---|---|---|
| CUDA 架构 | `-DCMAKE_CUDA_ARCHITECTURES=80` | 计算能力 8.0 就是 GA100；SM86 也行但 80 才是对的 |
| 后端 | 支持你模型的地方用 vLLM | 「约 1.8 倍」于 llama.cpp，来自一位测试者 Qwen3.6 27B 单卡、量化不对齐 |
| 跨卡并行 | 流水线，绝不用张量 | Gen1 x4 下 TP 预填充差 2.3-2.8 倍 |
| MTP | 单流 llama.cpp 开，35B MoE 的 vLLM 关 | 一个后端 +21%，另一个 -23% |
| 量化 | q4 级是 64 GB 卡上的实际甜点 | bf16 Qwen3.6 27B 是 54-56 GB，不留 KV 余量 |
| 模型家族 | 多卡优先 MoE | 每 token 跨设备激活流量更少 |

### 屋顶线指引

对**锁定**卡，2023 年的选择规则仍然成立：默认 FMA 下算数强度低于 **0.3 FLOPs/byte** 时有用，禁用 FMA 后低于 **4.6 FLOPs/byte** 有用（屋脊点来自 394 和 6250 GFLOPS 对 1355 GB/s 实测上限）。计算解锁后 FP32 约涨 30 倍而带宽不变，所以屋脊以同样的倍数外移，规则不再有约束力：对大多数内核，解锁卡表现得像一台普通的显存受限 GA100。（最后这句是从两个权威数字推导的，不是单独测量的结果。）

### 最便宜的未测试调优线索

在 CMP 100-210 上，设 `n_ubatch 56` 让 llama.cpp pp512 从 353.59 升到 **977.20 t/s**（关 flash attention）和从 380.96 升到 **1159.39 t/s**（开），一个标志带来 **3.04 倍**，而 tg128 基本不变。小增益一直保持到 uBatch 62，然后性能崩溃。那张卡有 84 个 SM 中的 68 个，所以 70-SM 170HX 上对应的调优点会在略低于 70 的地方。

> [!NOTE]
> **未解决问题：uBatch 悬崖在 170HX 上存在吗？**
>
> 没人跑过扫描。在一张计算解锁的 170HX 上用 `llama-bench` 把 `n_ubatch` 从 48 扫到 80，只是一个下午的活，也是档案中最大的未测试上涨空间。

### 验证一个调优点

```bash
# 1. compute stability and sustained flops
make COMPUTE=80                       # github.com/wilicc/gpu-burn
./gpu_burn -tc -m 90% 1200

# 2. memory integrity: the gate that catches silent corruption
./170hx-test.sh --no-unlock           # full-VRAM pattern sweep + compute checksum

# 3. link and geometry sanity
nvidia-smi --query-gpu=memory.total,clocks.max.sm,pcie.link.gen.current,pcie.link.gen.max --format=csv
```

在解锁的 8 GB 到 64 GB 卡上、300 W 上限下，一次干净的 30 分钟 `gpu_burn` 长这样：225 次迭代，检查点保持 **12,472-12,485 GFLOP/s**、`errors: 0`，温度只从 75 C 升到 77 C，实时遥测 `PCIe GEN 1@ 4x`、`GPU 1440MHz MEM 1890MHz TEMP 76C FAN N/A POW 278 / 300 W`、`GPU 100% MEM 57.534Gi/64.000Gi`，结束于 `Tested 1 GPUs: GPU 0: OK`。

散热通常不是约束：持续 GEMM 烤机在约 25-30 秒内核心从 62 升到 73 C 时 flops 保持平坦，全能力部件只在约 85 C 以上降频。真正要紧的是待机功耗和漏电流一起上升，所以更好的散热双倍划算。见 [cooling.md](cooling.md) 和 [thermals.md](../hardware/thermals.md)。

### 能效参考点

| 指标 | 值 | 条件 |
|---|---|---|
| 实测最佳 GFLOPS/W | 1390 GFLOP/W | 上限 1400 MHz，偏移 +350。**从未扫描验证，且夹在 1400/+325 CORRUPT 和 1400/+375 fault 之间。不是工作点。** |
| 次佳实测 | 1376 GFLOP/W | 上限 1350 MHz，偏移 +300。**单次跑完，从未被模式扫描把关。不是工作点。** |
| 最佳*扫描验证* GFLOPS/W | 1366 GFLOP/W | `eff`（出厂默认），+250 / 1350 MHz，132.0 W 下 180.3 TFLOPS；至少两次以 `mem_errors=0` 通过全显存模式扫描 |
| 出厂 | 925 GFLOP/W | 199.2 W 下 184.3 TFLOPS |
| LLM 服务 | 每瓦 2.16 tok/s | vLLM，Qwen3.6 27B int8，单卡 |
| 显存超频（报告） | flops +2.5% 换约 +5 C | 任何存档分支都不存在时钟或 p-state 代码，所以无法对照代码核实 |

---

## 一行一行列出不工作的东西

- 显存时钟锁定（`nvidia-smi -lmc`）：被驱动拒绝。
- 显存 VF 偏移：范围是 `[0 .. 0]`。
- 出厂 VBIOS 上把功耗上限抬到 250 W 以上：不提供。
- P-state 强制（`nvidia-pstated`、`NvAPI_GPU_SetForcePstate`）：卡只暴露 P0。
- 10 GB 卡上的核心时钟偏移：它的 VBIOS 里 `freqDelta` 是 0。
- 超过上限专属已验证最大值的偏移：故障、挂死或静默损坏。1650 MHz 上限下该最大值是 +350，但 1400 MHz 上限下只有 **+300**。
- 打游戏，任何调优点都不行：电容改装下 BeamNG.drive 在 Gen1 x16 是 15 fps、x4 是 5 fps，「哪种都很糟糕」。这不是游戏卡。
- ECC 作为安全网：已熔断关闭。

见 [open-questions.md](../frontier/open-questions.md) 和 [dead-ends.md](../history/dead-ends.md)。
