# 在 CMP 170HX 上跑 LLM

**本页涵盖。** 哪些推理栈在解锁卡上可用、实测的提示处理与 token 生成速率（带模型、量化和上下文）、什么会坏以及为什么、如何跨卡扩展，以及决定每个多卡决策的那个问题：PCIe 带宽是不是瓶颈。原始计算和带宽数字在 [performance.md](performance.md)；时钟和功耗在 [tuning.md](tuning.md)。

**三个结果决定你几乎每一个选择。**

1. **vLLM 在单卡、单一模型家族上大约比 llama.cpp 快 1.8 倍**，也比 SGLang 快。这个比例来自一位测试者，且双方量化不对齐，所以当作方向而不是常数。
2. **跨卡必须用流水线并行；张量并行在 PCIe Gen1 x4 上是死路。** Qwen2.5-72B AWQ 的直接 A/B：TP2 预填充差 2.3-2.8 倍，解码只多 +23%。
3. **有记录的最佳多卡结果**是 744B 参数 MoE（GLM-5.2，40B 活跃）在 vLLM 流水线并行下跑 8 张解锁 64 GB 卡：**131k 上下文下预填充 2,675 t/s、解码 30.2 t/s**，整个会话零硬故障。

下面所有数据除非说明，都在 **PCIe Gen1 x4** 下测得。出货解锁器完全不含 PCIe 代码：`master` 上的 `common/constants.yaml` 只有 `driver_versions`、`gpu`、`compute` 和 `profiles` 键，整棵代码树没有任何 `pcie` 段。任何描述为「在已发布解锁上运行」的基准，跑的都是卡的原生链路。

> [!WARNING]
> **怎么读本页的数字**
>
> 这里几乎每个数字都来自**一位测试者、一张卡、一次会话**。独立复现的极少，而且有几行曾经看起来像独立的确认，结果只是同一张表的两行，或是对十分钟前附上的报告的一次聊天转述。凡是有一个以上来源的数字，正文会明确说明；没有说明就当作单一报告。模型、量化、上下文和条件逐数字列出，也是这个原因。

---

## 解锁给推理工作负载带来什么

计算解锁（FEAT PLM `0x00823804` 打开为 `0xffffffff`，然后 SS0 `0x0082381C` = `0x88888888`、SS1 `0x00823820` = `0x00000008`）才是让 tensor core 吞吐可用的东西。见 [compute-throttle.md](../unlock/compute-throttle.md)。

- **`llama.cpp` 用出厂 SM80 构建自动使用 GA100 tensor core**。无需补丁、无需标志。这就是几位测试者在未修改的最新构建（CUDA 架构列表里带 SM80 和 SM86）上复现出每秒数千 token 提示处理速率的原因。
- **解锁后的显存真正可用。** 一位在 64 GB 卡上跑 LLM 的测试者报告「一次崩溃都没有」。六张解锁到 40 GB 的 10 GB 卡以 4-bit 跑 Qwen 27B 和 Qwen 35B，无崩溃、无变砖；唯一的限制是散热——没有足够散热方案时只能跑约 10 分钟。见 [cooling.md](cooling.md)。
- **解锁后的快速 sanity check：** LM Studio 配小模型，在「E2B」小模型上预期约 **85 tokens/s**。这是某位测试者在一个未完全说明模型大小的条件下的数字，当作数量级检查而不是目标。严格的检查是用 BF16 吞吐对照 202 TFLOPS 上限；见 [verify.md](../procedures/verify.md)。

---

## 后端选择

| 后端 | 在这块硬件上的结论 | 证据 |
|---|---|---|
| **vLLM** | 在几乎所有实测配置中都是最好的。默认选择。 | 「约 1.8 倍快」出自那位在单卡上对 Qwen3.6 27B 两边都跑过的测试者。量化对齐未建立：llama.cpp 这边是 Q4_K_M，vLLM 那边的量化从未说明，在频道里被读作 q6。存档的 vLLM 表给出 Qwen3.6-27B 单流 62.4 t/s，对照 36.87 t/s 是 1.69 倍。也是唯一能可用地跑 GLM-5.2 的栈 |
| **llama.cpp** | 单卡和非 DSA 模型的跨卡都没问题。DSA 注意力模型不可用。 | 单卡 pp512 888.09 t/s；GLM-5.2 八卡预填充 141 t/s 对比 vLLM 的 2,675 |
| **ik_llama** | 在唯一一次受控对比中比主线 llama.cpp 慢 | pp512 296.36 对比 360.65；tg128 33.20 对比 33.10 |
| **SGLang** | 在单卡 Qwen3.6 27B int8 上输给 vLLM，出乎意料，而且完全跑不了 MTP | 带截图的正面对决 |
| **LM Studio / llama-swap** | 能用；适合做冒烟测试和服务 | 小模型约 85 t/s；35B A3B Q8 在 125 W 功耗上限下 60 t/s |
| **Vulkan** | 多卡死路 | ggml Vulkan 后端不支持 `VK_KHR_device_group`，所以所有卡间传输都走主机内存 |

### 为这张卡构建 llama.cpp

一个可复现的容器构建以 `build-llama-170hx.sh` 发布：

```bash
# base image: nvidia/cuda:13.3.0-devel-ubuntu26.04
# clones ggml-org/llama.cpp at a resolved master commit
cmake -B build \
  -DGGML_CUDA=ON \
  -DCMAKE_CUDA_ARCHITECTURES=80 \
  -DGGML_BACKEND_DL=ON \
  -DGGML_CPU_ALL_VARIANTS=ON \
  -DGGML_OPENMP=ON \
  -DGGML_CUDA_FA=ON \
  -DLLAMA_OPENSSL=ON \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/app
cmake --build build -j12          # Ninja
# verifies libggml-cuda.so has no missing ldd entries, tags the image by
# llama.cpp build number, then smoke-tests with --runtime=nvidia --gpus all:
nvidia-smi --query-gpu=name,memory.total,pcie.link.gen.current,pcie.link.gen.max --format=csv
llama-bench --list-devices
```

`-DCMAKE_CUDA_ARCHITECTURES=80` 是承重的标志：CUDA 能力 8.0 就是 GA100。

> [!CAUTION]
> **预编译的 `libggml-cuda.so` 需要 CUDA 13，没有它会静默失败**
>
> 预编译二进制链接 `libcudart.so.13` 和 `libcublas.so.13`。在只有 CUDA 12.4 的主机上，权重会**以纯 CPU 方式加载然后 OOM**，而不是干净地报错，这让诊断变得很难。有效修复是把 PyTorch 自带的 cu13 库目录加到 `LD_LIBRARY_PATH` 前面。

`llama.cpp` 还通过 `--split-mode tensor` 获得了后端无关的张量并行（上游 PR `ggml-org/llama.cpp#19378`），取消了 GPU 数量必须偶数/2 的幂的限制。PR 自己把该功能描述为「实验性的……尚未达到生产就绪」。vLLM 的张量并行仍要求偶数 GPU 数，而且在这张卡上张量并行本来就是错误策略（见下文）。

---

## 单卡实测吞吐

### vLLM，一张解锁 64 GB 卡

测试形态大约是 1,500 token 提示加 200 token 输出。

| 模型 | 解码 | 4 路并行的合计 | 预填充 |
|---|---|---|---|
| `cyankiwi/Qwen3.6-35B-A3B-AWQ-4bit`（MoE） | **113 tok/s** | **452 tok/s** | **1,700 tok/s** |
| `Qwen3-32B-AWQ`（稠密） | 52.9 tok/s | 205 tok/s | 1,755 tok/s |
| Qwen3.6-27B BF16 | 19.2 tok/s | 71.1 tok/s | 2,231 tok/s |
| Qwen3.6-27B-AWQ-INT4 | 58.5 tok/s | 214.8 tok/s | 2,044 tok/s |

### llama.cpp，一张解锁卡

| 基准 | 模型 / 条件 | 结果 |
|---|---|---|
| `llama-bench` pp512 | qwen35 27B Q4_K_M，15.65 GiB，26.90 B 参数，CUDA，`ngl 99`，一张卡 | **888.09 ± 24.69 t/s** |
| `llama-bench` tg128 | 同一张表的第二行，同一次运行，同一截图 | **36.87 ± 0.04 t/s** |
| 社区参考基准 | Llama-2 7B Q4_0，频道里被提议作为标准量化 | **pp 3106 t/s、tg 158 t/s**，单一数字报告。第二位测试者发了截图但没有引用数字，说数字「还行」；第三位说「7b 我跑 3333」，没有说明量化 |
| 稠密对比稀疏 | Gemma4 26B A4B Q8（稀疏）pp2048 3894.29 对比 Gemma4 31B Q8（稠密）pp2048 830.33，同一台机器同一构建 | 模型密度，不是配置错误 |
| 按模型的解码 | Qwen3.5 9B q4_k_xl 约 105 tok/s；Qwen3.6 27B q8 配 q8 KV 和 MTP depth 2，50 tok/s；Gemma4 26B A4B Q8 tg128 90.65 / tg512 90.10 / tg1024 89.85；Gemma4 31B Q8 tg128 27.07 / tg512 26.74 / tg1024 26.33 | 单卡 |

888.09 / 36.87 这一对是**一次运行**：同一张 `llama-bench` 表格里的一行，同一张截图。另一位参与者后来转发了同一段对话的截图，说这次运行与自己的数字吻合，但从没发过被吻合的数字，所以没有第二张表可比。

稠密 27B 级模型在提示处理上比 7B 慢一个数量级，这是模型密度，不是调优失败：稠密 Qwen3.6 27B q4_k_p 在未打补丁的 SM80/SM86 构建上，pp 约 500 t/s 和约 900 t/s 都有人报告过。

### 最严格的一次受控单卡基准

一张解锁到 40 GB 的 10 GB 卡，Gen1 x4，250 W 上限。主机：2x EPYC 7713、DDR4、Supermicro H12DSi-N6、内核 6.18.38。模型 `unsloth/Qwen3.6-27B-MTP-GGUF`（`UD-Q4_K_XL` 和 `UD-Q8_K_XL`），Q4 用 40960 MiB 中的 17464 MiB、Q8 用 35718 MiB，待机约 42 W。一位测试者、一张卡、一次会话，完整发布带误差棒；没人重跑过。

| 构建 / 配置 | pp512 | pp2048 | pp8192 | tg128 | tg512 | tg2048 |
|---|---|---|---|---|---|---|
| llama.cpp b10095（e8e6c7af2），Q4，无 MTP | 360.65 | 564.30 | 722.49 | 33.10 | 32.67 | 30.50 |
| 同上，MTP（`--spec-type draft-mtp --spec-draft-n-max 2`） | 323.63 | 496.85 | 639.53 | 46.24（峰值 56.67） | 43.02（峰值 55.33） | 44.47（峰值 59.00） |
| ik_llama b4735（9d07d868），Q4，无 MTP | 296.36 | 544.72 | 649.61 | 33.20 | 34.23 | 31.93 |
| ik_llama，Q4，MTP（`--spec-type mtp:n_max=2,p_min=0.0`） | 203.38 | 315.87 | 336.61 | 41.11（峰值 47.00） | 38.26（峰值 47.67） | 35.82（峰值 46.67） |
| ik_llama，Q8，无 MTP | 271.49 | 584.31 | 697.18 | 26.36 | 27.27 | 25.79 |
| ik_llama，Q8，MTP | 203.84 | 328.81 | 363.25 | 38.15 | 37.69 | 36.78 |

### 单卡层面的功耗与能效

| 条件 | 结果 |
|---|---|
| Qwen3.6 27B `q6_k_xl`（41 GB 常驻），刻意弱化的主机（无 AVX2 CPU、卡在 250 W 降频、PCIe x4） | 约 26 tok/s，启用 MTP 后升到 **50 再到 55 tok/s** |
| `Qwen-AgentWorld-35B-A3B-Q8_0.gguf`，llama.cpp 经 llama-swap，**125 W 功耗上限** | 约 60 tok/s |
| Qwen3.6-35B-A3B Q8 配 MTP，**170 W** | 约 130 tok/s |
| vLLM，Qwen3.6 27B int8 | **每瓦 2.16 tok/s**，频道里被描述为「实际上不错的能效」 |

功耗上限和 MTP 都会大幅改变数字。见 [tuning.md](tuning.md)。

> [!NOTE]
> **未解决问题：27B 单卡解码数字没有对齐**
>
> 已发布和频道里的「Qwen 27B 级、一张解锁 64 GB 卡、vLLM」数字跨 **97 / 90 / 75 / 58.5 t/s**，外加 llama.cpp Q4_K_M 的 36.87 t/s。量化、MTP 状态、上下文长度和 vLLM 版本在各报告间都不同，从未被固定。照原样在单卡上跑一遍已发布的仓库配置并贴出参数，就能定案。

---

## 多 token 预测（MTP）

对同一个 35B MoE，MTP 在两个后端上的表现**正好相反**。

| 后端 | 无 MTP | 有 MTP | 变化 |
|---|---|---|---|
| llama.cpp（`unsloth/Qwen3.6-35B-A3B-MTP-GGUF`） | 108.4 tok/s | **131.3 tok/s** | **+21%** |
| vLLM（同类模型） | 147 tok/s | **113 tok/s** | **-23%**，尽管接受率 75% / 1.75 tokens |

在受控的 40 GB 机器上，MTP 买来约 **+40% 解码、约 -10% 提示处理**（tg128 33.10 到 46.24，pp8192 722.49 到 639.53）。它**只对单流有益**；另有警告说 MTP 在批大小增大时扩展不佳，这是推理而非实测。

vLLM 的倒退经过三个假设排查。MTP 头的量化被排除（`mtp` 在 `modules_to_not_convert` 里，fc、attention 和共享专家都是 BF16）。第二个怀疑是 FlashInfer cubin 不对。站得住的解释是 CPU 侧瓶颈：「它占用 CPU，而我们在 PCIe Gen1 4x 上，这成了瓶颈。GPU 计算利用率下降了 7%，显存占用也是。」建议的监控：

```bash
nvidia-smi dmon -s put      # watch sm, mem, rx/txpci
```

从未用修复证明过。

---

## 多卡：头条结果与配方

一个 744B 参数 MoE（GLM-5.2，40B 活跃）以 W4A16 对称量化在 **8 张解锁 64 GB 卡**上跑 vLLM 流水线并行，驱动 610.43.02，PCIe Gen1 x4（无电容改装，早于 Gen2 合并），租赁硬件。

| 指标 | 值 |
|---|---|
| 4k / 32k / 65k / 131k 上下文的预填充 | 665 / 1,497 / 2,342 / **2,675 t/s** |
| 解码（无 MTP） | **30.2 t/s** |
| KV 容量 | 0.92 显存利用率下 438,107 tokens（BF16 KV，MLA 下约 88-100 KB/token） |
| 模型加载时间 | 约 440-620 s |
| 故障 | 整个会话零硬故障 |

预填充随上下文**上升**，因为分块预填充加稀疏注意力。这与 llama.cpp 在同一模型上的表现相反，也是「栈的选择主导硬件」最清晰的一个信号。

这是**一份报告，不是两份。** 一位测试者租了九张卡、用八张做基准，附上 `170HX-benchmark-results.md`（131k 预填充 2,675 t/s、解码 30.2 t/s），十分钟后在聊天里把同一次运行概括为「预填充 2600 t/s、解码 30 t/s」。取整的和精确的是同一次会话。没有人在其他卡上复现过。

### 精确配方

```text
vllm==0.20.2  release wheel
  + PR #38476 python files applied as a diff onto site-packages
transformers 5.x                       (4.57 does not know glm_moe_dsa)
VLLM_ATTENTION_BACKEND=TRITON_MLA_SPARSE
VLLM_MEMORY_PROFILER_ESTIMATE_CUDAGRAPHS=0
--pipeline-parallel-size 8 --gpu-memory-utilization 0.90
block-size 64                          (auto-set by DEEPSEEK_V32_INDEXER)
quantisation: W4A16 symmetric
```

GLM-5.2 使用 DeepSeek 稀疏注意力（DSA），原生需要 Hopper 或 Blackwell。在 Ampere 上运行需要 vLLM PR #38476 的 `TRITON_MLA_SPARSE` 后端。

> [!CAUTION]
> **量化选择：这条路线上大多数已发布指南是错的**
>
> 对 vLLM 上的 GLM-5.2，MoE 内核**拒绝非对称量化**。
> 可用：`lowbitcoffee/GLM-5.2-W4A16`（对称，g128，388 GB）和 `QuantTrio/GLM-5.2-Int4-Int8Mix`。
> **失败：`cyankiwi/GLM-5.2-AWQ-INT4`**（非对称，g32），而这是大多数指南引用的量化。

> [!WARNING]
> **实验性：`VLLM_USE_PRECOMPILED` 在这里行不通**
>
> 应用 vLLM 补丁的常规方式是 `VLLM_USE_PRECOMPILED` 的可编辑安装。它不附带 `vllm._C`，会失败。用 0.20.2 release wheel，把 PR 的 python 文件以 diff 形式应用到 site-packages。

### 其他多卡结果

| 配置 | 模型 / 栈 | 结果 |
|---|---|---|
| 8x 64 GB（512 GiB），llama.cpp b10079，`-ngl 999 -c 4096 -np 1 --flash-attn on --no-context-shift --fit off --no-warmup --spec-type none`；虚拟化主机，GPU 名被遮蔽，链路活动 Gen1 x4 但设备最大 Gen2 x16 | GLM-5.2 UD-IQ2_M，239 GB 2-bit，约 224 GiB 常驻，约 6 分钟加载 | TG **17.33 tok/s**（17.31-17.37，SD 0.02）；PP **113.0 tok/s**（111.8-115.5，SD 1.01），连续十次运行 |
| 8x 64 GB，llama.cpp `-sm layer` | GLM-5.2 Q4_K_S GGUF | 512 / 4k / 16k 下预填充 **141 / 162 / 124 t/s**（随上下文退化），解码 **17.2 t/s** |
| 8x 64 GB，llama.cpp 按层切分，Gigabyte G292-Z20，Proxmox 直通 | GLM-5.2-Q4_K_XL 完全进显存（报告 320 GB 常驻） | 单流 **13-14 tok/s**，20 个并发会话下塌到 **3 tok/s** |
| 4x 64 GB（256 GB）经 AliExpress x4x4x4x4 分叉转接板，每卡 Gen1 x4，llama.cpp 按层/按行切分，无 MTP | unsloth `GLM-5.2-UD-IQ2_XXS` | **约 15 tok/s 解码**，**24.07 t/s 预填充**；日志细节见下 |
| 3x 40 GB（120 GB），llama.cpp，模型几乎全在 CPU 上：**一层**加上下文和计算缓冲在 GPU 上（40 GB 中各自占 18 GB） | unsloth `GLM-5.2-GGUF`，约 460 GB MoE | **pp2048 33.44 ± 0.37 t/s，tg512 5.90 ± 0.03 t/s**。对照纯 CPU 加 DDR4，测试者只报了相对变化，「TG 大约升了 60%」、「PP 大约降了 30%」；纯 CPU 的绝对值从未发布 |
| 7 卡租赁机，llama.cpp | GLM-5.2 | **预填充 121 t/s**，被判定不可用（提示处理约 25 分钟）；在解码前就终止了 |

4 卡服务器日志值得全文引用，因为它是档案中条件交代最完整的多卡抓取：

```text
n_ctx_slot = 65536, n_keep = 0
prompt eval:  13210.35 ms /  318 tokens (41.54 ms per token,  24.07 tokens/s)
       eval:   5235.97 ms /   67 tokens (78.15 ms per token,  12.80 tokens/s)
      total:  18446.32 ms /  385 tokens ; graphs reused 66
slot timings at n_decoded 100/148/196/244/292:
  tg    = 13.62 / 14.25 / 14.59 / 14.79 / 14.93 t/s
  tg_3s = 13.62 / 15.79 / 15.72 / 15.68 / 15.68 t/s
VRAM: 53G/64G, 60G/64G, 60G/64G, 56G/64G
prompt cache: 8192.000 MiB, 65536 tokens, 8589934592 est
```

### 并发扩展很糟糕

只有一次并发扫描：8 卡 GLM-5.2 UD-IQ2_M 报告，`-np 16 -c 16384`、持续批处理、每用户 128 tokens。下面每一列都在该报告中制成表。

| 用户数 | 1 | 2 | 4 | 8 | 16 |
|---|---|---|---|---|---|
| 合计 | 17.3 | 21.6 | 25.7 | 28.1 | **38.9 tok/s** |
| 每用户 | 17.3 | 10.8 | 6.4 | 3.5 | **2.4 tok/s** |
| 批处理墙钟时间 | 不适用 | 11.9 s | 20.0 s | 36.5 s | 52.6 s |
| 对比 1 用户的扩展 | 1.00x | 1.25x | 1.49x | 1.62x | **2.25x** |

从 1 到 16 个用户只有 **2.25x**。报告把原因归结为三件事共同作用：没有 PCIe 或 NVLink 对等直连，所以每个 token 都要经主机内存中转七次；第 49 到 50 层过渡处有一个跨 NUMA 的流水线跳变；以及链路本身。注意主机是虚拟化或直通环境、GPU 名被遮蔽，链路读作**活动 Gen1 x4 但设备最大 Gen2 x16**，所以这一次扫描不是干净的出厂卡测量。

---

## 为什么 PCIe 是瓶颈，又为什么不是

这是档案中争议最大的问题，争论持续主要是因为双方描述的是不同配置。按配置陈述论点，就干净地化解了。

### 物理状况

- 出厂链路是 **Gen1 x4、约 1.0 GB/s**，而且在推理负载下**不会拉高**。
- **没有对等直连，也没有 NVLink。** 在 8 卡机上，`torch.cuda.can_device_access_peer(i,j)` 对全部 **56 对 GPU** 返回 `False`，即使在同一个 PIX 组内；一份 ggml `-lv 5` 日志里 peer/p2p/rpc 出现次数为零；`nvidia-smi nvlink` 报告「Device does not have or support Nvlink」。见 [p2p.md](../frontier/p2p.md) 和 [nvlink.md](../frontier/nvlink.md)。
- 因此，在 8 卡 80 层模型（每 GPU 10 层）按层切分时，每个生成的 token 要经过 **7 次 GPU 到 CPU 内存到 GPU 的跳转**，其中一次在第 49 到 50 层过渡处跨 NUMA/插座边界。

### 它不咬人的地方

**单卡。** 权重在加载后常驻显存。PCIe 成本是一次性的约 30 s 模型加载，之后预填充和解码以正常速度运行。把单卡从 x4 移到 x16，llama.cpp 的 pp 只从 439 变到 448、tg 从 81.91 变到 85.75。「PCIe 带宽无关紧要」的立场在**这个场景**、也只有这个场景是**对的**。

**扩散和图像/视频生成。** 计算受限且常驻显存，链路根本不进入画面。

### 它狠咬的地方

**预填充，不是解码。** 在两种不同的卡族上都有报告：一位 170HX 卸载用户发现提示处理在 CPU 上竟然*比 GPU 卸载更快*（三卡一层卸载那次测到 pp2048 33.44 t/s，注释「PP 因为 gen1 x4 链路限制降了约 30%」）；一位带大 MoE 模型的 CMP 100-210 多卡用户报告「解码的流水线并行没问题，杀死你的是预填充」。同一次运行中解码反向移动，「TG 对照 CPU 加 DDR4 大约升了 60%」，所以只有当解码主导你的工作负载时 GPU 卸载才值得。对比两边的百分比都是测试者引用；纯 CPU 的绝对值从未发布，所以不要把任何推导出的 CPU 数字当成实测。

**张量并行。** 决定性的 A/B，Qwen2.5-72B 稠密 AWQ 在 vLLM、Gen1 x4 上：

| 配置 | 1k / 4k / 16k 预填充 | 解码 |
|---|---|---|
| 1 张卡 | 839 / 1,092 / 960 t/s | 27.3 t/s |
| **PP2**（流水线） | 829 / 1,084 / **1,167** t/s | 29.1 t/s |
| **TP2**（张量） | **316 / 420 / 416** t/s | 33.7 t/s |

TP 的预填充**差 2.3-2.8 倍，解码只多 +23%**。GLM-5.2 八卡上同样的模式更糟：TP8 配 `enforce_eager` 在 4k / 16k / 32k 给出 382 / 435 / 629 t/s（比 PP8 差约 4 倍）和 **3.4 t/s 解码**；不带 `enforce_eager` 时 CUDA-graph 捕获崩溃（vLLM issue #48285）。多位操作者独立收敛到同一结论：「PCIe 1.0 x4 链路下张量并行是行不通的。」

**并发。** 上面从 1 到 16 个用户的 2.25x 上限。

### 为什么流水线并行能挺过窄链路

流水线并行只在阶段之间传送 token 或其激活/嵌入向量，每个阶段每 token 一次。张量并行拆分每一次矩阵乘法，因此需要在每一层内跨卡做一次 all-reduce，这对带宽**和**延迟都很苛刻（而延迟是加宽链路修不了的部分）。测量支持的互联需求排序是：

**张量 >> 专家 > 流水线 > 数据。**

只有数据并行能在出厂 170HX 链路上不受损地运行。早期流传过一个两卡流水线并行在 PCIe 1.0 x4 链路上提示处理 **1.56 倍**增益的说法，但那是二手的：来自一个报告者无法分享的私人群组，没有模型、量化或配置。当作轶事，不是测量。流水线并行不会显著改善 token 生成速度；它买来的是容量和预填充，不是解码。实测 PP2 相对单卡的解码增益是 27.3 到 29.1 t/s，16k 预填充从 960 升到 1,167 t/s。

### 张量并行的门槛，以及为什么 Gen2 x4 达不到

所述门槛是 **PCIe Gen2 x16 或 Gen3 x4**：「除非我们至少解锁 PCIE 2 16x 或 PCIE 3 4x，否则张量并行免谈。」解锁器交付的是 **Gen2 x4**（约 2 GB/s），低于这个门槛。聊天里提到的「Gen 2 x4 lane unlock」是用词不当：`Gen2/_DIFF_vs_master.patch` 里没有任何 lane、width 或 x16 处理。恢复 x16 是**物理**改装，24 颗手工焊接的 0402 电容。见 [physical-mods.md](physical-mods.md) 和 [pcie-gen2.md](../unlock/pcie-gen2.md)。

Gen2 x4 买来什么，来自**同一位测试者的两次运行**，都不是有明确方法论、受控的 A/B。两次都值得带着它们的条件读。

**更干净的一次：单卡、模型完全常驻显存。** 一张解锁到 40 GB 的 10 GB 卡，`unsloth/Qwen3.6-27B-MTP-UD-Q8_K_XL` 跑在开 MTP 的 ik_llama 上，测试者描述为「其他所有因素不变」：

| 测试 | Gen1 x4（2026-07-22） | Gen2 x4（2026-07-27） |
|---|---|---|
| pp512 | 203.84 ± 12.10 | **277.84 ± 19.81** |
| pp2048 | 328.81 ± 8.27 | **449.41 ± 13.44** |
| pp8192 | 363.25 ± 14.93 | **493.86 ± 16.92** |
| tg128 | 38.15 ± 0.20 | **41.52 ± 1.89** |
| tg512 | 37.69 ± 1.59 | **40.12 ± 1.52** |
| tg2048 | 36.78 ± 1.43 | **37.90 ± 0.80** |

测试者概括为「PP 大涨」，说「TG 也有不错的提升」，并解释不了为什么一个完全常驻显存的模型会动，猜测是 MTP 的 CPU 侧调度。两个注意点：两次运行相隔五天而不是背靠背，而且第二次运行的既定目的是测量 SlimSAS 转接路径，不是链路速率。

**多卡那次，不是一对对等的对比。** 一个三 GPU 运行，模型几乎全在 CPU 上（一张卡上只有一层加上下文和计算缓冲）：2026-07-20 在 Gen1 x4 下 pp2048 33.44 ± 0.37 t/s，2026-07-24 在测试者标注为「gen2 x4 尝试」下 48.22 ± 1.36 t/s；tg512 5.90 到 6.39；首响应时间 61,253 到 42,510 ms。这对数据常被引用的百分比增量（预填充 +44.2%、延迟 -30.6%）是这里算的，不是测试者说的，而且两次运行量化不对齐：后一次标着 `GLM-5.2-GGUF-Q4`，前一次是未标注的 `GLM-5.2-GGUF`。同一台机器在 Gen2 x4 下跑 `GLM-5.2-UD-Q2_K_XL` 给出 pp2048 49.00 ± 1.08、tg512 6.81 ± 0.06。测试者自己的判断很谨慎：「gen2 x4 下 PP 至少不比原来差，但我觉得还是被带宽钉着。」

两边的方向：预填充和延迟改善，解码动得少得多。这是流水线并行模型预测的形状，但两次运行都没把链路速率干净地隔离出来。

> [!NOTE]
> **未解决问题：没人重跑过 Gen2 x4 下的并行 A/B**
>
> 这个领域的所有并行对比都在 Gen1 x4 下跑的。上面两组 Gen2 x4 推理数据来自一位测试者，而且都不是流水线对张量的对比。其他人反复说「还没试 pcie 2.0」。现成最干净的单一变量实验是上面那套 4 卡分叉机（配置完整记录），装 Gen2 代码跑完全相同的 GLM-5.2 UD-IQ2_XXS 负载。

> [!NOTE]
> **未解决问题：权重常驻后通道数还重要吗？**
>
> 被直接问过，只得到观点（「任何额外带宽都求之不得」「用 MoE 模型」「PCI-e 3.0 x16 对多 GPU 绰绰有余」）。它被硬件卡住：提问的人手边没有 x16 卡。语料中唯一一个长上下文预填充对宽度的数字——64k 上下文下从 x16 移到 x8 大约从 6,000 掉到 3,000 t/s——是在一张不同的、非 170HX 卡上测的，不能平移。前面引用的 CMP x4 对 x16 llama.cpp 对比是短上下文、单卡，所以也定不了长上下文的案。

### 确实有依据的缓解措施

- **优先用 MoE 模型。** 它们减少每 token 的跨设备激活流量，这是针对没有 NVLink 的推荐缓解。是推理而非基准隔离，但与所有实测一致。
- **永远用流水线并行。**
- **模型放得下就在一张卡上批处理**，而不是跨卡分片。

---

## 什么会坏

| 症状 | 原因 | 修复 |
|---|---|---|
| 权重以纯 CPU 方式加载然后 OOM | 预编译 `libggml-cuda.so` 需要 `libcudart.so.13` / `libcublas.so.13`；主机是 CUDA 12.4 | 把 PyTorch 自带的 cu13 库目录加到 `LD_LIBRARY_PATH` 前面 |
| vLLM 导入失败，没有 `vllm._C` | `VLLM_USE_PRECOMPILED` 可编辑安装 | 0.20.2 release wheel + 把 PR #38476 的 python diff 应用到 site-packages |
| vLLM 不认识 `glm_moe_dsa` | `transformers` 4.57 | `transformers` 5.x |
| GLM-5.2 MoE 内核拒绝量化 | 非对称量化（`cyankiwi/GLM-5.2-AWQ-INT4`，asym g32） | 对称量化：`lowbitcoffee/GLM-5.2-W4A16`、`QuantTrio/GLM-5.2-Int4-Int8Mix` |
| GLM-5.2 预填充塌到约 120-160 t/s | llama.cpp 没有 DSA 支持，回退到稠密注意力（llama.cpp issue #24730） | 用带 `TRITON_MLA_SPARSE` 的 vLLM |
| vLLM TP8 在 CUDA-graph 捕获时崩溃 | vLLM issue #48285 | `enforce_eager` 能避开崩溃但预填充贵约 4 倍；改用 PP |
| MTP + 流水线并行拒绝运行 | vLLM 中目前不兼容 | 未知；MTP + TP8 直接 OOM |
| SGLang 跑不了 MTP | 「sglang doesnt like mtp」 | MTP 用 vLLM 或 llama.cpp |
| 加载器把 RSS 钉满并抖动磁盘直到 OOM | llama.cpp 加载期的计算图扫描抖动系统内存 | 更多主机内存（见下文）；容器内 `swapon` 被禁 |
| 模型加载在约 20 GB 后卡住 | 80 GB 几何 | 回退到出货 40 GB 配置 |

> [!CAUTION]
> **80 GB 配置给你的是更少的可用内存，不是更多**
>
> 在实验性 `80` 分支下，模型加载在约 20 GB 后卡住，连之前能装进 40 GB 解锁的模型都开始加载不了；另一位测试者在 40-60 GB 区间看到失败。回退到 40 GB 几何恢复了正常的加载。还要注意：该分支的 `constants.yaml` 宣告 `lmr: 0x0000028B`，但构建从不读那个文件：`80/driver/build.sh` 第 93 行设置 `LMR="0x0000028A"`，所以每个跑过该分支的测试者实际编程的是 CFG1 `0x02779000` + LMR `0x0000028A` + `fb_length 0x0000001400000000`，一个三向不一致，它本身很可能就是不稳定的原因。见 [80gb.md](../frontier/80gb.md)。

---

## 模型尺寸与主机要求

| 问题 | 答案 |
|---|---|
| Qwen3.6 27B bf16 装得进 64 GB 卡吗？ | 可以，大约 **54-56 GB**，几乎不留 KV 余量。同一模型的 Q4_K_M 量化是 **18-24 GB** |
| 超过 q4 值得吗？ | 频道里的判断（有未指明基准支持）：不明显。64 GB 卡上 KV 余量更重要 |
| 需要多少主机内存？ | 超大型模型**至少约 256 GB**。一个 GLM-5.2 4-bit 467 GB 模型在 88 GiB 内存的主机上**即使有 512 GiB 显存也加载不了**：权重达到约 431 GiB 显存平台（405 GiB 模型加 KV/开销），但加载器把 RSS 钉在 **87.6 GB** 并持续以约 **820 MB/s** 反复读盘直到 OOM。在原生 `llama-server` 和 Unsloth studio 运行、以及 `-c 1024` / `-c 8192`、no-warmup 和批处理调优下都复现 |
| 模型加载时间？ | 单卡 Gen1 约 30 s；239 GB 模型跨 8 卡约 6 分钟；GLM-5.2 在 vLLM 下约 440-620 s。走 RPC，>=500B 模型 Q4-6 要 20-60 分钟，Kimi K3 级模型跨 170HX 卡要 4-6 小时 |
| Kimi K3 需要几张卡？ | 约 1.4T 权重以 MXFP4（e2m1）加 MXFP8 激活，是**每权重 4.25 bit**（4 给权重，0.25 给每 32 权重一个 8-bit scale），所以约 **744 GB 权重，十二张 64 GB 卡的量**。频道里的估计是随口说的、偏高：「那么…… 25 张卡 XD」只针对纯流水线并行，还有「大概 32，算上低效、kv 缓存和一个合理的并行配置」。没人调和过这个算术，而且那个估计的 tp8 那一半在 Gen1 x4 下不成立。GA100 通过 Marlin 内核处理 group scales |

> [!NOTE]
> **在这个区间，显存不是模型选择的唯一约束**
>
> 从 40 GB 升到 84 GB 只让一位用户用更长的上下文跑同一个 27B 模型。另外两人附和：「即使 8x64 加 512GB，deepseek pro 之类的 LLM 还是跑不了」，「我觉得 27b 大概撑到 200gb 左右」。这是关于这个尺寸下模型生态的说法，不是关于硬件的。反方意见是更大的量化和未量化上下文。

---

## 这张卡和其他硬件的对比

| 参照 | 数字 | 备注 |
|---|---|---|
| RTX 3090，Qwen 27B q4_k_m | 开 MTP 60 tok/s，不开 40，预填充约 1,200 t/s | 频道里用的对比基线 |
| 170HX 对比 3090 | **稠密**模型单流解码大致 3090 级，显存大得多；MoE 预填充**高于** 3090；MoE 解码**低于**它，因为解码带宽受限 | 「3090 级」的说法有争议，双方对不同的模型类别可能都对 |
| 两张 170HX 对比一张 RTX 5090，图像和视频 | 「慢一点点，但功耗相同，还能在两个独立的 comfyui 容器里跑并发任务」 | 一位测试者、租赁硬件、只有截图、从未复现：低置信度 |
| A100，GLM-5.2 | 55 tok/s | 二手、无配置、低置信度 |

---

## 非 LLM 的 CUDA 工作负载

扩散和图像生成是强适配：INT8 卷积「在 cmp170 的 ComfyUI 里又快又好」，一位从 Pascal 过来的用户称它「快得像鬼」。机理是工作负载计算受限且完全装进显存，所以 Gen1 x4 链路不咬人，而且扩散对低精度噪声的容忍度比语言模型高，因为错误不会以同样方式累积。一个注意点：扩散 transformer 权重的离群值比 LLM 权重更差，所以 W8A8 式量化不是自动安全的。实测扩散数字在 [performance.md](performance.md) 列成表。

视频生成可用：单张解锁卡上的 LTX 2.3 在「零优化」的情况下约 **2 分钟生成约 30 秒片段**，卡在 USB 控制涡轮风扇下持续 250 W 且保持低于 65 C。那份报告没有分辨率、帧数或步数，所以置信度低到中。

一台六 GPU 10 GB 到 40 GB 主机被验证服务**五个并发 vLLM OpenAI 兼容端点**（GPU 0-4，各一个模型：`Qwen/Qwen2.5-7B-Instruct-AWQ`、`cyankiwi/Qwen3-Coder-30B-A3B-Instruct-AWQ-4bit`、`Qwen/Qwen2.5-32B-Instruct-AWQ`、`Qwen/Qwen2.5-VL-32B-Instruct-AWQ`、`Qwen/Qwen2.5-Omni-7B-AWQ`），加 GPU 5 上的 ComfyUI。这明确是一次并发冒烟测试，不是基准。

> [!NOTE]
> **未解决问题：3D Gaussian splat 训练**
>
> 预测差而不是实测差。Splat 很少有稠密矩阵乘法，通常也不用低精度格式，所以它们跑在标准 CUDA 核心上，而这张卡 FP32 约 12 TFLOPS，大致是 3060 的水平。没有人跑过真正的 splat 基准。

---

## 未解决问题

1. **让 MTP 在 vLLM 里和流水线并行共存。** 基准报告预测如果 RFC #44697 中未合并的修复落地，PP8 解码「约 1.7 倍（30 到约 50 t/s）」；同一位作者后来在聊天里说 MTP「应该能把 GLM 5.2 推到接近 45 t/s」。两者都是从实测 30.2 t/s 出发的推算，不是测量。
2. **在已记录的 4 卡分叉机上于 PCIe Gen2 x4 重跑并行 A/B。**
3. **用数字回答 Gen2 下 x4 对 x16。** 被硬件卡住。
4. **照原样跑一个已发布配置，对齐约 27B 的单卡解码数字。**
5. **解释 vLLM 的长上下文倒退：** 一位测试者看到 vLLM 在约 130k 上下文时掉到 22 tok/s，而同一张卡上、两边都开 MTP 的 llama.cpp 保持在 48 tok/s，且正常上下文时 vLLM 领先 90 对 60。八卡结果显示 vLLM 预填充随上下文*改善*，所以很可能是配置原因。没人复现过。
6. **确定 P2P 到底能不能开启。** 解锁器已经能到达 SEC2/PLM 寄存器；P2P 能力位是否落在 `0x00823804` 的 FEAT PLM 管辖的同一片空间里，从未被检查过。
7. **Colibri 式专家放置**，直接在 CPU 上从内存执行非常驻 MoE 专家，而不是通过慢链路把它们传上去。提示性的先验证据：在这条链路上，预填充本来就已经比 GPU 卸载更快。
8. **显存解锁是否通过刷新冲突略微拖慢本已不受限的计算。** 没人跑过前后对比，因为安装器把两个解锁一起应用。

见 [open-questions.md](../frontier/open-questions.md)、[multi-gpu.md](../procedures/multi-gpu.md) 和 [dead-ends.md](../history/dead-ends.md)。
