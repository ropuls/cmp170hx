# 验证解锁

**本页涵盖：** 如何用证据而非希望来证明解锁确实生效：每个 SKU 的 `nvidia-smi` 应该显示什么、如何逐行解读 `SEC2_DEBUG` 内核日志、已安装的元数据文件能说明什么又不能说明什么、仅存在于分支上的 `verify.sh` 如何工作、如何确认多出来的显存是*真实*的而非别名映射（aliased），以及如何确认**算力**解锁——它与显存容量是完全独立的两个结果，需要各自的测量。

要点：8 GB 卡上 `nvidia-smi` 报告 **65536 MiB** 或 10 GB 卡上报告 **40960 MiB**，证明显存几何写入生效了。但它对算力什么都证明不了。算力由两次寄存器写入解锁（SS0 `0x0082381c` = `0x88888888`，SS1 `0x00823820` = `0x00000008`），这两次写入对 `nvidia-smi` 不可见，确认它们的唯一方法是回读 `SEC2_DEBUG` 日志或对吞吐量做基准测试。

---

## 三个层面，以及各自的证据

| 层面 | 是什么 | 主要证据 | 次要证据 |
|---|---|---|---|
| 显存**容量** | CFG1 + LMR 几何，加上 GSP `fb_length` 与 PMA 重写 | `nvidia-smi` 总显存 | `SEC2_DEBUG: POST-WRITE ... CFG1=... LMR=...` |
| 显存**真实性** | 报告的容量背后是独立的物理 DRAM，而非别名映射 | `check_fold.py` 报告 `REAL, NO FOLD` | 一次大型 `gpu_burn` 或 `cuda_memtest` 运行零错误 |
| 算力**吞吐** | FEAT PLM 打开后写入的 SS0/SS1 | `SEC2_DEBUG: POST-WRITE SS0=0x88888888 SS1=0x00000008` | 与锁定基线对比的 FP32/OpenCL 基准测试 |

一张卡可能通过其中一项而另一项失败。算力解锁能扛过函数级复位，而显存几何不行，这正是算力先于显存发布的原因。

---

## 各 SKU 的 `nvidia-smi` 预期

| 指标 | 8 GB 卡（`10de:20c2`） | 10 GB 卡（`10de:2082`） |
|---|---|---|
| 出厂 `memory.total` | 8192 MiB | 10240 MiB |
| 解锁后 `memory.total` | **65536 MiB** | **40960 MiB** |
| 写入的 CFG1 `0x009a0204` | `0x02779000` | `0x02669000` |
| 写入的 LMR `0x00100ce0` | `0x0000020B` | `0x0000028A` |
| 写入的 GSP `fb_length` | `0x0000001000000000`（64 GiB） | `0x0000000A00000000`（40 GiB） |
| 报告的产品名 | 原版驱动下为 `NVIDIA Graphics Device`，因为 PCI ID 表中没有市场名称 | 同上 |
| 计算能力（Compute capability） | 8.0 | 8.0 |
| SM 数量 | 70（4480 个 CUDA 核心） | 70 |
| PCIe 链路（出厂） | 第 1 代，上限 1，宽度 4 | 第 1 代，上限 1，宽度 4 |

```bash
nvidia-smi
nvidia-smi --query-gpu=name,memory.total,clocks.max.sm,pcie.link.gen.current,pcie.link.gen.max,pcie.link.width.current --format=csv
```

解读结果：

- **8 GB 卡显示 8192 MiB 意味着解锁没有生效。** 这是泄露发行版自带 README 中的失败排查界限，而且是对的：PLM 没有打开，因此它下游的一切都没有发生。
- **出厂容量与目标容量之间的任何数值都不是"部分解锁"。** 几何是一组从 PCI 设备 ID 选出的固定 CFG1 + LMR 组合；它要么生效，要么没有。

> [!CAUTION]
> **81920 MiB 不是成功**
>
> 10 GB 卡报告约 81920 MiB、CUDA 看到 85,545,582,592 字节（79.67 GiB），说明它跑的是实验性 80 GB 档，而不是正式发布的 40 GB 配置。`cudaMalloc` 分配 77 GiB 能成功，但触碰超过约 40 GB 的内核会导致致命 GPU 丢失，与功耗上限无关。已报告的错误码包括 Xid 31（被描述为无害）以及 CUDA 内存测试之后的 Xid 154；主要报告症状是卡死。Xid 31 的说法来自一位旁观者，并未被拥有故障卡的运营者证实为*那个*特征错误。80 GB 配置曾被尝试并已放弃。参见 [80 GB](../frontier/80gb.md)。提到它的每份文档都记录为不稳定；`80` 分支 README 里 "Working" 那一行是文档缺陷。

> [!NOTE]
> **`clocks.max.sm = 1935 MHz` 是报告字段，不是可达频率**
>
> `install.sh` 建议把检查 `clocks.max.sm` 作为验证的第 4 步，解锁后的卡确实报告 1935 MHz。请把这个数字当作**低置信度**，不要当作工作频率：VBIOS 表中的最大图形频率是 1695 MHz，实际硅片上限大约在 +350 偏移下的 1604-1614 MHz。所有持续测量都稳定在**1410 MHz** 标称值，或 `-pl 300` 下的**1470 MHz**。参见 [Tuning](../operations/tuning.md)。

---

## `SEC2_DEBUG` dmesg 轨迹

每次解锁操作都会以 `SEC2_DEBUG:` 前缀记录日志。此前缀，连同 `SEC2_DEBUG_PRI_*` 寄存器名和 `kgspSec2PostblTiming*` 函数名，在原版 610.43.03 源码中完全不存在；"PostBL Timing" 是一个虚构的、听起来很像 NVIDIA 的功能名。实际后果就是一条 grep：

```bash
sudo dmesg | grep SEC2_DEBUG
sudo dmesg | grep -c SEC2_DEBUG      # the count varies by build and card count, see below
```

> [!NOTE]
> **行数不是通过/失败的判据**
>
> 每一份存档中的行数都不同，没有哪一个是特征指纹。唯一一份存档的单卡 8 GB 抓取包含 **29** 行。唯一一份存档的双卡 Gen2 分支 `610.43.03` 启动日志包含 **134** 行。`pcielink.sh` 报告工具在两台独立双卡 Gen2 机器（一台 HiveOS 主机和一台 Unraid 主机）上都打印了 `SEC2_DEBUG lines=152`，另有 34（Gen1 构建）/ 80（Gen2 构建）的记载。不要把行数不一致读作安装失败。下面的寄存器回读行才是判据。

在健康启动中，日志行大致按此顺序输出。

| 日志行（格式） | 阶段 | 如何解读 |
|---|---|---|
| `SEC2_DEBUG: saved stock signature (4096 bytes)` | 负载覆盖签名缓冲区之前 | 如果缺失或大小不对，磁盘上的 GSP 固件可能仍是固件时代前代方案打过补丁的版本 |
| `SEC2_DEBUG: loaded 63488 bytes from /lib/firmware/nvidia/ga100/gsp/dmem.bin` | 负载来源 | 仅当你刻意放置了覆盖负载时出现 |
| `SEC2_DEBUG: <path> not found (0x59), using built-in payload` | 负载来源 | **正常。** `0x59` 无害；会使用内置负载，目标为 `0x009a0148 = 0xffffffff` |
| `SEC2_DEBUG: WPR meta fbSize=... wprEnd=... heapSize=...` | 第一次 `kgspPopulateWprMeta_HAL` | 解锁前的几何，所以这里的 `fbSize` 仍反映出厂容量 |
| 携带 `status=0xffff` 的逐 PLM 尝试行 | 四项 PLM 循环 | **每次负载通过时都应出现。** Booter 在负载运行后总会在 mailbox0 中留下错误 |
| `SEC2_DEBUG: PLMs: FEAT=0xffffffff FBPA=0xffffffff WPR=0xffffffff WPR_CFG=0xfffff0ff` | PLM 循环结果 | PLM 的最终判定。见下表 |
| `FAILED to open %s after 2 attempts` | PLM 循环失败 | 每个 PLM 最多尝试两次；此行指明哪一个没有打开 |
| `SEC2_DEBUG: POST-WRITE SS0=... SS1=... CFG1=... LMR=... (devId=0x%x)` | 主机寄存器写入 | **本页最有用的单行。** 把四个值对照上面的 SKU 表 |
| `SEC2_DEBUG: WPR meta updated fbSize=... wprStart=... wprEnd=... heapOffset=... heapSize=...` | 第二次 `kgspPopulateWprMeta_HAL` | 现在与扩大后的几何一致 |
| `SEC2_DEBUG: normal BooterLoad status=0x%x` | 真正的引导 Booter 运行 | 这次应该是 `NV_OK`，与负载通过不同 |
| `SEC2_DEBUG: POST-BooterLoad verify PLM=... SS0=... SS1=... CFG1=... LMR=...` | 引导后回读 | 仅当正常 BooterLoad 返回 `NV_OK` 时打印。**这就是解锁在真正的 GSP 启动中存活下来的证明** |
| `SEC2_DEBUG: static-info BEFORE` / `AFTER` | GSP 静态配置重写 | `fb_length` 和最后一个 FB 区域被加宽 |
| `SEC2_DEBUG_HEAP: fbAddrSpace=... mapRam=... fbTotal=... fbUsable=... heapTotal=... regionBytes=... publicBytes=... numRegions=...` | 堆创建之后 | PMA 工作的诊断信息 |
| `SEC2_DEBUG: late PMA extension status=0x%x` | 显存欺骗的第二阶段 | `0x0` 即成功。此处非零意味着额外显存从未向分配器注册，即使几何写入已经生效 |
| `SEC2_DEBUG: rebuild stock signature failed: 0x%x` | 仅失败时 | 如果无法恢复原版签名，整个初始化会中止 |

### 正确解读 PLM 行

| PLM | 地址 | 期望值 | 说明 |
|---|---|---|---|
| WPR_CFG | `0x001fa7cc` | `0xfffff0ff` | **不是** `0xffffffff`。这是代码写入*并*检查的值 |
| FBPA | `0x009a0148` | `0xffffffff` | 也是内置负载的默认目标 |
| WPR | `0x001fa7c4` | `0xffffffff` | |
| FEAT | `0x00823804` | `0xffffffff` | 出厂为 `0xffffff8f`；常开，扛得过函数级复位 |

### 日志缺失时

环形缓冲区会轮转。显存正确的卡缺少 `SEC2_DEBUG` 轨迹是警告而非失败，`verify.sh` 也是这么对待的。要强制生成新轨迹，冷启动后立即 grep，或增大内核日志缓冲区。

---

## 已安装的元数据文件

```bash
cat /lib/modules/$(uname -r)/updates/cmpunlocker/card_profile      # 8gb | 10gb | mixed
cat /lib/modules/$(uname -r)/updates/cmpunlocker/unlock_geometry   # 64GB | 40GB | mixed
cat /lib/modules/$(uname -r)/updates/cmpunlocker/driver_version    # e.g. 610.43.03
cat /lib/modules/$(uname -r)/updates/cmpunlocker/gpu_inventory     # branch-only, see multi-gpu.md
```

这些是由 `build.sh` 写入的单行文件。**内核模块中没有任何代码读取它们。** 它们记录的是安装器*以为*的东西，而不是驱动*实际做*的事。一台 8 GB 卡以 65536 MiB 启动的机器上出现 `card_profile` 为 `10gb`，是元数据 bug，不是解锁 bug，因为几何是在 GSP 启动时从 PCI 设备 ID 选出的。打过补丁的内核在启动时读取的唯一文件是可选的 `/lib/firmware/nvidia/ga100/gsp/dmem.bin`。

另外两个有用的检查：

```bash
cat /proc/driver/nvidia/version        # should NOT say dvs-builder if the patched module is live
cat /sys/module/nvidia/srcversion      # compare with:
modinfo -F srcversion /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko
```

`srcversion` 不匹配意味着正在运行的是原版模块。这与 [Multi-GPU](multi-gpu.md) 中描述的多卡 depmod 歧义属于同一类失败。

---

## `verify.sh`

> [!WARNING]
> **实验性：仅分支脚本**
>
> `verify.sh` 在 `master` 上**不存在**。它只随 `multiple-cards`、`Gen2`、`far` 和 `deced` 分支发布。`master` 上也没有 `tools/` 目录和测试套件。

`verify.sh` 是一个多 GPU 安装后检查器。它要求有 `nvidia-smi`，缓存 `nvidia-smi --query-gpu=pci.bus_id,memory.total --format=csv,noheader,nounits`，然后枚举 GPU：优先使用已安装的 `gpu_inventory` 文件，否则回退到 `lspci -nn | grep -iE '10de:20c2|10de:2082'`。

对每块 GPU，它按报告容量分类：

| 配置 | `is_unlocked_memory` | `is_stock_memory` |
|---|---|---|
| `8gb` | `>= 60000` MiB | `7680`-`8704` MiB |
| `10gb` | `35000`-`59999` MiB | `9728`-`10752` MiB |

并打印四种状态之一：

| 状态 | 含义 |
|---|---|
| `OK` | 处于解锁区间 |
| `STOCK` | 仍是锁定容量 |
| `MISSING` | 该 BDF 完全不在 `nvidia-smi` 输出中 |
| `UNEXPECTED` | 对该配置既不是出厂也不是解锁的容量 |

然后它 grep `dmesg` 中的 `SEC2_DEBUG`，打印最后八条匹配行作为样本，打印已安装的 `card_profile` 和 `unlock_geometry`，最后以以下之一结束：

```text
✓ All 4 unlockable GPU(s) report unlocked memory
```

并以 0 退出，或者：

```text
✗ 1 GPU(s) failed unlock verification. Cold reboot if modules were just installed.
```

并以非零退出。

两个已知缺口：

- **`verify.sh` 从不检查 PCIe Gen2**，即使在 Gen2 分支血统上也是如此。对 `Gen2/verify.sh`、`far/verify.sh` 和 `deced/verify.sh` grep "pcie" 是零命中。链路验证完全靠手动；参见 [PCIe Gen2](../unlock/pcie-gen2.md)。
- **`verify.sh` 从不检查算力。** 显存容量是唯一的通过判据。

---

## 确认显存是真实的，而非别名映射

报告的容量和可用容量是两个不同的主张。要排除的失败模式是**折叠（fold）**：地址空间回绕，使高地址别名映射到低地址。

`check_fold.py` 是权威测试。它不在仓库中：和 `cuda_dbg.py` 一样，它是通过 gist 或频道附件带外分发的，所以要单独获取，不要指望克隆会提供它。它分配除 2 GiB 外的全部空闲显存，用 PTX `sm_80` 的 `fill` 内核把每个 64 KB 页写入自己的索引，然后用 `chk` 内核把每一页读回来，使用 `st.global.wt.u32` 存储和 `ld.global.cv.u32` 加载来绕过缓存。它必须是密集探测，因为折叠发生在通道**交织（interleave）**偏移处：`LOW[0]` 映射到 `(40 GiB + interleave)` 而不是 `(40 GiB + 0)`，所以稀疏探测会给出假阴性。

| 输出 | 退出码 | 含义 |
|---|---|---|
| `REAL, NO FOLD` | 0 | 容量背后是独立的物理 DRAM |
| `FOLD/mismatch @<pageindex>` | 1 | 在该页检测到别名映射 |
| error | 2 | 测试环境问题 |

更轻和更重的替代方案：

- `cuda_dbg.py` 是一个快速别名测试：`cuMemGetInfo_v2`，然后在 64、60、56、52、48、44、42 GiB 依次尝试 `cuMemAlloc_v2` 直到成功，接着用 `cuMemsetD32_v2` 在偏移 0 写入 `0xAAAA0000`、在 40 GiB 处写入 `0xBBBB0000`，并读回偏移 0。在偏移 0 读到 `0xBBBB0000` 意味着空间发生别名映射。它会泄漏其分配，所以每次驱动加载只运行一次。
- `cuda_memtest` 1.2.3 是维护者推荐的社区验证器；它在第一个错误处退出。在 80 GB 配置上，它打印 `Attached to device 0 successfully.` 然后无限挂起，除非把分配上限设为 39 GB。这个挂起是 `80` 分支 README 中 "Working" 主张的主要反证，而不是一个弱信号。
- 泄露发行版的 README 建议在 64 GB 卡上运行 `./gpu_burn -m 63500 -d 30`，期望零内存错误。

> [!WARNING]
> **折叠测试工具产生过假阳性**
>
> 一个早期的折叠/别名测试工具把*原生的、未解锁的*显存报告为折叠：在一次复位到一致原生状态（10240 MiB、驱动 610.43.03、CFG1 `0x02449000`）后的对照运行中，它分配了 9 GiB 真正原生的显存，并在五轮中报告 "4608 chunks, 4608 corrupt/aliased"，这是不可能的。这追溯性地使一批早期"40 GB 处折叠"的结论失效。请相信 `check_fold.py` 的密集方法，并把任何临时脚本的折叠结果视为未经证实，直到原生对照运行干净通过。

---

## 确认算力吞吐——与容量相区别

算力解锁是 FEAT PLM 打开后对 SS0 和 SS1 的一对写入。它们是来自主机 CPU 的 `GPU_REG_WR32` 调用，PLM 打开后没有任何漏洞利用参与，而且对**两个 SKU 都是无条件执行**。

| 寄存器 | 地址 | 锁定 | 解锁 |
|---|---|---|---|
| SS0（`SEC2_DEBUG_PRI_FEATURE_OVERRIDE_SM_SPEED`） | `0x0082381c` | 例如 `0x53540175` | `0x88888888` |
| SS1（`..._SM_SPEED_1`） | `0x00823820` | | `0x00000008` |
| FEAT_OVR_PLM | `0x00823804` | `0xffffff8f` | `0xffffffff` |

### 第 1 步：回读它们

`POST-WRITE` 和 `POST-BooterLoad verify` 两行都带有 SS0 和 SS1。如果在正常 BooterLoad 之后它们读为 `0x88888888` 和 `0x00000008`，本次启动算力已解锁。这是最便宜、最直接的确认方式，即使在显存解锁失败的卡上也有效。

### 第 2 步：测量吞吐

寄存器回读证明写入生效了；但不能证明硅片行为真的不同。健康解锁卡的参考数据：

| 指标 | 数值 | 说明 |
|---|---|---|
| SM 数量 | 70 | 用 PTX `%smid` 转储器实测，而非仅仅报告值 |
| 理论 FP32 | 12.63 TFLOPS | 4480 核心 x 2 x 1410 MHz |
| 持续 SM 频率 | 1410 MHz，`-pl 300` 下 1470 MHz | 基础 1140 MHz |
| TDP / 最大软件功耗上限 | 250 W / 300 W | 300 W 仅在带 NVIDIA OC 挖矿 VBIOS 的卡上；原版 CMP VBIOS 的 `nvidia-smi -pl` 范围为 100-250 W |
| HBM 带宽，实测 | 一个**区间**，1305.86-1600 GB/s | 取决于工具与访问模式；没有唯一的权威数值 |
| HBM 理论峰值 | 1555.2 GB/s（1448.4 GiB/s） | 1215 MHz DDR x 5120 位 |

用算力基准测试，而不是显存测试：项目自己的概念验证截图用的是 [OpenCL-Benchmark](https://github.com/ProjectPhysX/OpenCL-Benchmark)，`clpeak` 和 `mixbench` 也在使用中。请在同一张卡上、解锁前后、同一驱动、同一功耗上限下用同一基准对比。参见 [Performance](../operations/performance.md)。

### 关于 FMA 锁定的说明

与 SS0/SS1 无关，此部件的 FP32 融合乘加吞吐受到限制，但编译器标志可以绕开：`nvcc -fmad=false`、`#pragma OPENCL FP_CONTRACT OFF` 加上对 `fma()`/`mad()` 的宏遮蔽（OpenCL），或 SYCL 的 clang `-ffp-contract=off`。2023 年一个**锁定卡**的 FluidX3D 案例在去掉 FMA 后达到 7,681 MLUPs/s，提升 3.4 倍；另一份 2023 年报告用同样方法把锁定卡的 FP32 从 0.395 提升到 6.285 TFLOPS。这些都是解锁前的数据。SS0/SS1 写入之后，FP32 FFMA 不再受限（普通构建下 12.2-12.8 TFLOPS），no-FMA/no-DP4A 补丁不再需要。**基准测试接近 6.25 TFLOPS 的卡是解锁失败的标志，而不是 FMA 收缩的产物**。参见 [Performance](../operations/performance.md)。

---

## 完整的验证清单

> [!WARNING]
> **`check_fold.py` 和 `cuda_dbg.py` 不在仓库中**
>
> 两者都是通过 gist 和频道附件带外发布的，而不是通过仓库，必须单独获取。克隆拿不到任何一个。

```bash
# 1. Right module is live
cat /proc/driver/nvidia/version                       # not dvs-builder
cat /sys/module/nvidia/srcversion
modinfo -F srcversion /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko

# 2. Unlock executed this boot
sudo dmesg | grep SEC2_DEBUG | grep -E 'PLMs|POST-WRITE|POST-BooterLoad|late PMA'

# 3. Capacity
nvidia-smi --query-gpu=memory.total --format=csv,noheader

# 4. Capacity is real
python3 -u check_fold.py <BDF>                         # expect: REAL, NO FOLD   (out-of-band script, not in the repo)

# 5. Compute
#    read SS0/SS1 from the POST-WRITE line, then benchmark FP32 against the locked baseline

# 6. Multi-card rigs (branch script)
sudo ./verify.sh
```

如果任何一步失败，[Troubleshooting](troubleshooting.md) 按症状组织，[Recovery](recovery.md) 涵盖卡死情况。
