# 故障排查（Troubleshooting）

**本页涵盖：** CMP 170HX 解锁的每一个有记录的失败模式，按你实际看到的症状索引：dmesg 字符串、Xid 编号、Booter 状态码、`RmInitAdapter` 三元组、`nvidia-smi` 读数、构建错误和主机级怪事。每条都给出症状、已确认的原因和修复方法，并在记录较薄的地方标注置信度。

**从这里开始。** 两条命令能回答大多数问题：

```bash
sudo dmesg | grep SEC2_DEBUG      # did the unlock path run, and what did it read back?
nvidia-smi                        # 8 GB card -> ~65536 MiB, 10 GB card -> ~40960 MiB
```

如果 `dmesg | grep SEC2_DEBUG` 什么都不打印，说明补丁模块从未运行：去 [Installed but still stock](#stock-memory)。如果它有输出且 PLM 行达到目标但显存仍是出厂容量，去 [Memory still shows stock size](#stock-memory) 并检查 initramfs。如果启动根本没走到那一步，去 [GSP boot failures](#gsp-boot)。

两条能避免大多数误报的规则：

1. **`WPR_CFG` 读 `0xfffff0ff` 是正确的。** 四个权限级掩码（PLM）中只有三个以 `0xffffffff` 为目标。见 [PLM readback values](#benign-wprcfg)。
2. **PLM 通过期间的 Booter 状态 `0x31` 和 `0xffff` 是预期的。** 解锁故意让那些运行失败。只有 `SEC2_DEBUG: normal BooterLoad status=0x0` 才重要。见 [Booter errors during the PLM passes](#benign-booter-31)。

---

## 症状索引 { #index }

| 你看到的 | 去哪里 |
|---|---|
| dmesg 中完全没有 `SEC2_DEBUG` 行 | [Installed but still stock](#stock-memory) |
| 安装后 `nvidia-smi` 显示 8192 MiB 或 10240 MiB | [Memory still shows stock size](#stock-memory) |
| `[WARN] Loaded nvidia srcversion (…) != patched (…)` | [srcversion mismatch](#srcversion-mismatch) |
| `Resolved nvidia.ko is not under updates/cmpunlocker/` | [Module resolution](#module-resolution) |
| `nvidia-smi`：driver/library version mismatch | [Version mismatch](#version-mismatch) |
| 解锁生效了，但没扛过关机 | [Unlock does not persist](#not-persistent) |
| 安装器退出但什么都没做 | [Installer refuses to run](#install-refuses) |
| `Could not detect 8GB vs 10GB card` | [Profile detection](#profile-detect) |
| `This card reports 0x…; install will continue` | [Third device ID `20b0`](#device-id-20b0) |
| `WPR_CFG=0xfffff0ff` 看起来不对 | [PLM readback values](#benign-wprcfg) |
| `Booter failed with non-zero error code: 0x31` | [Benign Booter errors](#benign-booter-31) |
| `dmem.bin not found (0x59)` | [Missing `dmem.bin`](#benign-0x59) |
| `Skipping BTF generation … vmlinux` | [Benign build noise](#benign-btf) |
| `[drm] No compatible format found` | [Benign DRM messages](#benign-drm) |
| `cudaHostRegister of 439781.26 MiB failed` | [Benign llama.cpp warning](#benign-cudahostregister) |
| 卡显示为通用 "NVIDIA display device" | [Generic enumeration](#benign-generic-device) |
| `CMP Gen2: PCIe retrain completed without Gen2 link (status=0x1042)` | [Gen2 retrain false negative](#benign-retrain-false-negative) |
| `unexpected WPR2 already up, cannot proceed with booting GSP` | [WPR2 already up](#wpr2-already-up) |
| `RmInitAdapter failed! (0x62:0x40:2028)` | [WPR2 already up](#wpr2-already-up) |
| `RmInitAdapter failed! (0x62:0x55:2028)` | [Status code catalogue](#codes-rminit) |
| `RmInitAdapter failed! (0x62:0x65:2028)` | [Status code catalogue](#codes-rminit) |
| `RmInitAdapter failed! (0x62:0x40:2674)` | [Second-GPU init failure](#rminit-2674) |
| `RmInitAdapter failed! (0x62:0xffff:2119)`，带 Booter `0x29` | [Dirty SEC2 exit](#rminit-2119) |
| `RmInitAdapter failed!` `0x24:0x72`、`BAR 0/BAR 2 failed.` | [BAR2 self-test failure](#bar2-0x72) |
| `GSP didn't boot`，状态 `0x65` | [GSP timeout `0x65`](#gsp-0x65) |
| Xid 119、60 秒、函数 4097 `GSP_INIT_DONE` | [Xid 119, 60 s](#xid119-60s) |
| Xid 119、6 秒、函数 103 `GSP_RM_ALLOC` | [Xid 119, 6 s](#xid119-6s) |
| Booter 错误 `0x35` | [Booter `0x35`](#booter-0x35) |
| PG199 / A100D 上 Booter 错误 `0x54` | [Booter `0x54`](#booter-0x54) |
| `rpc_result = 0xFFFF`、NULL `GSP-LOG[RM]` | [RM init stalls early](#rpc-ffff) |
| `falconMailbox 0:00000031`、`riscvPc 00000000` | [Falcon core dump](#falcon-coredump) |
| `0xbadfXXXX` 寄存器读 | [`0xbadf` taxonomy](#codes-badf) |
| 触发运行了、什么都没变、`resetPLM 0xff -> 0x8f` | [Bus mastering cleared](#bus-master) |
| `PLMs: 1/9 open (fired 8 closed)`、`resetPLM=0x00cf` | [Driver still loaded](#driver-loaded) |
| `modprobe -r nvidia` 拒绝、`nvidia 15835136 2` | [Module will not unload](#module-stuck) |
| DMEM 写入被静默丢弃 | [DMEM locked in HS](#dmem-locked) |
| CFG1 写入弹回 `0x02449000` | [FLR between PLM open and write](#flr-between) |
| 卡多日后"劣化"、`SEC2 MBOX0 = 0x0` | [Deleted firmware directory](#firmware-deleted) |
| 换内核后构建失败 | [Build failures](#build) |
| 安装解锁器后 PCIe 仍是第 1 代 | [Gen2 stays at Gen1](#gen2-still-gen1) |
| 黑屏、文本控制台、`cmpretrain.service` 失败 | [Black screens](#black-screen) |
| Xid 31、`FAULT_INFO_TYPE_REGION_VIOLATION` | [Xid 31](#xid31) |
| 杀掉 CUDA 任务后 Xid 45 | [Xid 45](#xid45) |
| 过度配置的卡上 Xid 154 | [Xid 154](#xid154) |
| vLLM 在 `gpu-memory-utilization 0.95` 崩溃 | [vLLM headroom](#vllm) |
| 到处 `cuInit` 返回 999 | [`cuInit` 999](#cuinit999) |
| gpu-burn 报告成千上万的内存错误 | [Burn-in errors](#burn-errors) |
| `nvidia-smi --gpu-reset`："GPU is being used by another process" | [Reset refuses](#gpu-reset-busy) |
| 解锁有效但 CUDA 不行 | [CUDA broken on one host](#cuda-clean-host) |
| 多 GPU 机器：每张卡都保持出厂 | [Silent multi-card failure](#multicard) |
| 装上卡后服务器无法 POST | [Host will not boot](#no-post) |
| 卡跑了一小时后从总线消失 | [Card off the bus](#off-bus) |

---

## 1. 健康启动应该是什么样 { #healthy-boot }

一次成功的解锁启动按此顺序打印这些 `SEC2_DEBUG` 行：

1. `SEC2_DEBUG: saved stock signature (4096 bytes)`，紧跟着 `SEC2_DEBUG: <path> not found (0x59), using built-in payload`
2. WPR meta 转储：`SEC2_DEBUG: WPR meta fbSize=… wprEnd=… heapSize=…`
3. `SEC2_DEBUG: saved WPR2 lo=0x%08x hi=0x%08x`
4. 四行 `SEC2_DEBUG: PLM[%u] %s(0x%x) attempt=%u status=0x%x reg=0x%08x`
5. `SEC2_DEBUG: PLMs: FEAT=… FBPA=… WPR=… WPR_CFG=…`
6. `SEC2_DEBUG: POST-WRITE SS0=… SS1=… CFG1=… LMR=… (devId=0x%x)`，仅失败时接着 `SEC2_DEBUG: rebuild stock signature failed: 0x%x`
7. `SEC2_DEBUG: WPR meta updated fbSize=… wprStart=… wprEnd=… heapOffset=… heapSize=…`
8. `SEC2_DEBUG: normal BooterLoad status=0x0`
9. `SEC2_DEBUG: POST-BooterLoad verify PLM=… SS0=… SS1=… CFG1=… LMR=…`
10. GSP static-info BEFORE/AFTER 对

两条签名打印都来自 `_kgspCreateSignatureMemdesc`，它在 `_kgspBootGspRm` 之前运行，所以它们开启轨迹而不是坐在序列中间。`POST-BooterLoad verify` 行是决定性证明：它是在**真正的** GSP 启动**之后**回读的，所以它显示解锁存活了下来。

**期望值。**

| 日志字段 | 期望 | 说明 |
|---|---|---|
| `PLM[0] WPR_CFG(0x1fa7cc)` | `reg=0xfffff0ff` | 不是 `0xffffffff` |
| `PLM[1] FBPA(0x9a0148)` | `reg=0xffffffff` | |
| `PLM[2] WPR(0x1fa7c4)` | `reg=0xffffffff` | |
| `PLM[3] FEAT(0x823804)` | `reg=0xffffffff` | 常开岛，扛得过 FLR |
| 任何 PLM 行上的 `status=` | `0xffff` | 预期；回读才是判定 |
| `SS0 (0x0082381c)` | `0x88888888` | 锁定卡读例如 `0x53540175` |
| `SS1 (0x00823820)` | `0x00000008` | |
| `CFG1 (0x009a0204)` | `0x02779000`（8 GB 卡）/ `0x02669000`（10 GB 卡） | 两者出厂都是 `0x02449000` |
| `LMR (0x00100ce0)` | `0x0000020B`（8 GB 卡）/ `0x0000028A`（10 GB 卡） | 出厂 `0x00000208` / `0x00000288` |
| `normal BooterLoad status` | `0x0` | 唯一必须为零的状态 |

`POST-WRITE` 中任何其他 CFG1/LMR 组合都意味着错误配置在起作用。这些值的含义见 [Memory geometry](../unlock/memory-geometry.md)。

存在三个日志标签，全部以 `LEVEL_ERROR` 发出，所以无需额外调试标志就会显示：

| 标签 | 由谁发出 | 内容 |
|---|---|---|
| `SEC2_DEBUG` | 补丁 0001、0002、0003 | PLM、寄存器和 Booter 阶段：0001 中 14 条日志字符串、0002 中 7 条，加上 0003 中的 `late PMA extension status=0x%x` |
| `SEC2_DEBUG_HEAP` | 补丁 0003 | `fbAddrSpace=%lluMB mapRam=%lluMB fbTotal=%lluMB fbUsable=0x%llx heapTotal=0x%llx regionBytes=0x%llx publicBytes=0x%llx numRegions=%u`（一条字符串） |
| `SEC2_DEBUG_LATE_PMA` | 补丁 0003 | 逐 FB 区域描述符加上 `pma_total 0x%llx->0x%llx pma_free 0x%llx->0x%llx`（10 条字符串） |

**完整验证块：**

```bash
nvidia-smi                                                     # ~65536 MiB or ~40960 MiB
nvidia-smi --query-gpu=memory.total,clocks.max.sm --format=csv
sudo dmesg | grep SEC2_DEBUG
cat /lib/modules/$(uname -r)/updates/cmpunlocker/card_profile  # 8gb or 10gb
cat /lib/modules/$(uname -r)/updates/cmpunlocker/driver_version
cat /lib/modules/$(uname -r)/updates/cmpunlocker/unlock_geometry
```

一份存档的好结果字面显示 `65536 MiB, 1935 MHz`。注意 `clocks.max.sm = 1935 MHz` 是**仅报告字段**，不是可达频率：持续 SM 频率是 1410 MHz，或 `-pl 300` 下 1470 MHz。参见 [Performance](../operations/performance.md)。

完整安装后清单见 [Verify](verify.md)。

---

## 2. 看起来像失败但不是的消息 { #benign }

### 2.1 PLM 通过期间的 Booter 错误 { #benign-booter-31 }

**症状。**

```text
s_executeBooterUcode_TU102: Booter failed with non-zero error code: 0x31
kgspExecuteBooterLoad_TU102: failed to execute Booter Load: 0xffff
```

紧接着一行 PLM 行显示 `reg=` 中的目标值。

**为什么没事。** 补丁 0001 故意用漏洞利用负载覆盖 GSP 签名缓冲区进行每次 PLM 通过，所以 Booter Load *本来就该*拒绝这些运行：签名抱怨提出时注入链已经执行完了。成功只看 PLM 寄存器回读，绝不看 Booter 状态。最坏情况是在真正的引导 Booter Load 之前有八条这样的记录（四个 PLM，每个最多两次尝试）。

**必须成功的行**是 `SEC2_DEBUG: normal BooterLoad status=0x0`。

### 2.2 `dmem.bin not found (0x59)` { #benign-0x59 }

```text
SEC2_DEBUG: /lib/firmware/nvidia/ga100/gsp/dmem.bin not found (0x59), using built-in payload
```

这是正常路径。外部 `dmem.bin` 是一个开发覆盖钩子，用 `os_open_and_read_file` 读取；`0x59` 是该函数的文件未找到状态。每一份存档的成功解锁启动都显示这一行。内置回退负载以 FBPA PLM 为目标（`writeAddr = 0x009a0148`、`writeValue = 0xffffffff`），PLM 循环随后逐次重写它。

它前面的行 `SEC2_DEBUG: saved stock signature (4096 bytes)` 确认此驱动上原版 GSP 签名是 4096 字节。

### 2.3 构建中的 `Skipping BTF generation` { #benign-btf }

```text
Skipping BTF generation for .../nvidia.ko due to unavailability of vmlinux
```

无害。BTF 是与解锁无关的内核调试元数据；模块仍然能构建和加载。它出现在 `nvidia-peermem.ko`、`nvidia-modeset.ko`、`nvidia-drm.ko`、`nvidia.ko` 和 `nvidia-uvm.ko` 上。之后要看的行是 `[ OK ] Patched NVIDIA modules loaded`。

### 2.4 DRM "no compatible format" 消息 { #benign-drm }

```text
[drm] Initialized nvidia-drm 0.0.0 20160202 ... on minor 1
[drm] No compatible format found
[drm] Cannot find any crtc or sizes
```

无害。CMP 170HX 没有显示输出。

### 2.5 llama.cpp `cudaHostRegister` 警告 { #benign-cudahostregister }

```text
ggml_cuda_host_malloc: cudaHostRegister of 439781.26 MiB failed: unknown error
```

不是致命的：加载继续，基准测试完成。要把它与真正的分配崩溃区分开，后者会直接杀死进程。

### 2.6 通用设备枚举 { #benign-generic-device }

卡在 Linux 监控工具（例如 Mission Center）中枚举为通用 "NVIDIA display device" 是正常的。在原版驱动上 `nvidia-smi` 也把它报告为计算能力 8.0 的 `NVIDIA Graphics Device`，因为驱动的 PCI ID 表没有为 `0x20C2` 提供市场名称。这是快速确认你在看一张 CMP 部件的方法。

### 2.7 Gen2 重训练"未以 Gen2 链路完成" { #benign-retrain-false-negative }

```text
CMP Gen2: PCIe retrain completed without Gen2 link (status=0x1042, ret=0)
```

这是一个**假阴性**：`0x1042` *就是*一个训练好的 Gen2 x4 链路。解码：速率字段 `[3:0] = 2`（5.0 GT/s），宽度字段 `[9:4] = 4`（x4）。驱动的成功测试还额外要求 `PCI_EXP_LNKSTA_DLLLA`（Data Link Layer Link Active，第 13 位，`0x2000`），而 `0x1042` 的第 13 位是清除的，所以检查失败，尽管链路确实在 Gen2。报告 `0x7042`（第 13 位置位）的主机从同一代码打印成功消息。两台主机上的四张卡显示了这种矛盾配对。

请相信以下之一：

```bash
nvidia-smi --query-gpu=pcie.link.gen.current --format=csv
cat /sys/bus/pci/devices/0000:$BDF/current_link_speed
```

> [!NOTE]
> **未解决问题**
>
> DLLLA 位读零是否表示这些主机之间真实的（尽管无害的）链路层差异，而不仅仅是报告假象，从未被调查过。

### 2.8 "所有 PLM 必须显示 `0xffffffff`" { #benign-wprcfg }

第三方文档（`docs/DEBUGGING.md`、`docs/ARCHITECTURE.md`，以及正式发布 README 中较温和的措辞）说每个 PLM 都应该读 `0xffffffff`。这是过度概括。正式发布的 `plmTable[]` 是：

```c
{ 0x001fa7ccU, 0xfffff0ffU, "WPR_CFG" },
{ 0x009a0148U, 0xffffffffU, "FBPA"    },
{ 0x001fa7c4U, 0xffffffffU, "WPR"     },
{ 0x00823804U, 0xffffffffU, "FEAT"    },
```

循环的成功谓词是 `if (regVal == plmTable[plmIdx].value)`。健康启动打印 `SEC2_DEBUG: PLMs: FEAT=0xffffffff FBPA=0xffffffff WPR=0xffffffff WPR_CFG=0xfffff0ff`。

---

## 3. 装了，但卡仍是出厂状态 { #stock-memory }

这是最常见的报告类别。解锁代码没问题；正在运行的不是补丁模块，或者它没有得到一次干净的启动机会来运行。

### 3.1 `nvidia-smi` 显示 8192 MiB 或 10240 MiB { #stock-size }

**原因。** 要么 PLM 解锁没生效，要么原版模块仍在加载。

**修复。** 检查 `sudo dmesg | grep SEC2_DEBUG`。

* **完全没有输出**意味着补丁模块从未运行。走完 3.2 到 3.5。
* **有输出、PLM 未达到目标**意味着解锁链运行了但失败：做一次完全断电关机（操作系统重启*不够*）并重试。见 [Cold boot](recovery.md#cold-boot)。
* **有输出、`POST-WRITE` 正确、显存仍是出厂**指向第二阶段显存管道而非寄存器写入。抓取 `SEC2_DEBUG_HEAP` 和 `SEC2_DEBUG_LATE_PMA` 行以及 `late PMA extension status=0x%x` 值。

泄露发行版的 README 用同样的分流：`nvidia-smi` 显示 65536 MiB 是成功判据，8192 MiB 意味着 PLM 解锁失败。它也指示**冷**重启而非热重启。

### 3.2 srcversion 不匹配 { #srcversion-mismatch }

**症状。**

```text
[WARN] Loaded nvidia srcversion (…) != patched (…)
[WARN] Modules installed but the running driver is still stock (or unload failed).
```

**原因。** 原版 `nvidia.ko` 仍然驻留且无法卸载。`build.sh` 尝试热重载（停止 `nvidia-persistenced` 和 `nvidia-fabricmanager`，对四个模块 `modprobe -r`，重新加载），并把 `/sys/module/nvidia/srcversion` 与已安装模块的 `modinfo -F srcversion` 交叉核对。

**修复。** 冷重启（`shutdown -h now`，然后开机），然后确认：

```bash
cat /proc/driver/nvidia/version      # must NOT say dvs-builder
sudo dmesg | grep SEC2_DEBUG         # must have output
```

一位测试者确认冷重启清除了它。

### 3.3 模块解析：原版仍胜出 { #module-resolution }

**症状。** `build.sh` 打印 `Resolved nvidia.ko is not under updates/cmpunlocker/, stock may still win`。

这是模块解析问题的最早信号。模块优先级是 `updates/cmpunlocker/` > `updates/dkms/` > `kernel/drivers/`，即普通 depmod 排序（这就是为什么不需要 `dpkg-divert`）。`build.sh` 运行 `depmod -a "${KVER}"`，然后用 `modprobe -n -v nvidia` 经验性地验证结果。

> [!CAUTION]
> **多 GPU 隐患**
>
> 在多 GPU 系统上，补丁版和原版 `nvidia.ko` 可能同时落在唯一的 `updates` depmod 搜索条目下，此时 **depmod 会随意挑选一个，并静默丢弃另一个**。一位测试者把一次多 GPU 失败精确根因定位到这一点，只在 updates 搜索路径中保留 cmpunlocker 变体，重启，然后确认多 GPU 正常工作。

### 3.4 initramfs 仍带着原版模块 { #initramfs }

`build.sh` 的分支副本（`memory`、`ecc`、`housekeeping`、`PG199`）逐字带着解释："NVIDIA often loads from initramfs. If only updates/dkms is packed there, stock modules win at boot even when updates/cmpunlocker is preferred by depmod." Master 删掉了注释但保留了行为：`build.sh` 按可用顺序调用 `update-initramfs -u -k "${KVER}"`、`dracut --force --kver "${KVER}"` 或 `mkinitcpio -P`；如果都没有，它警告 `No initramfs tool found, rebuild manually before rebooting`。

这是"装了但显存仍显示出厂容量"的一条合理路径，值得先排除因为它便宜，但这是脚本自己的推理而非诊断出的现场故障：语料中没有任何聊天报告提到 initramfs、initrd、dracut 或 mkinitcpio。如果你看到了那个警告，手工重建 initramfs 并冷启动。

### 3.5 维护者分流：三步 { #triage-three-step }

"安装完成但卡仍是出厂"的标准分流：

```bash
# Step 1 - did the build target the kernel you actually booted?
uname -r
ls -la /lib/modules/$(uname -r)/updates/cmpunlocker/
cat /lib/modules/$(uname -r)/updates/cmpunlocker/driver_version
cat /lib/modules/$(uname -r)/updates/cmpunlocker/card_profile
cat /lib/modules/$(uname -r)/updates/cmpunlocker/unlock_geometry

# Step 2 - is the running module the patched one?
modprobe -n -v nvidia
modinfo -F filename,srcversion,version nvidia
cat /sys/module/nvidia/srcversion
modinfo -F srcversion /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko

# Step 3 - what does the card and the log say?
nvidia-smi --query-gpu=name,memory.total,driver_version,pci.bus_id --format=csv
sudo dmesg | grep -E "SEC2_DEBUG|NVRM|nvidia"
cat /proc/driver/nvidia/version
```

第 1 步中目录缺失意味着构建针对的内核与你启动的不同。第 2 步中解析路径必须包含 `/updates/cmpunlocker/`，运行中的 srcversion 必须等于 cmpunlocker `.ko` 的 srcversion；不匹配意味着原版仍在运行。

注意三个元数据文件**只是元数据**。内核模块中没有任何代码读取它们：几何在运行时从 PCI 设备 ID 选择。误检的 `--profile` 会写错误的元数据，但**不会**产生错误的几何。

### 3.6 `nvidia-smi`：driver/library version mismatch { #version-mismatch }

**原因。** 之前的内核模块仍然驻留。

**修复，按安全性排序：** 重启；或重新加载内核模块；或安装匹配的 `nvidia-smi` 构建。禁用版本不匹配检查是掩盖问题，不是修复。

> [!CAUTION]
> **不匹配的 `nvidia-smi` 会静默地使每次测量失效**
>
> NVML 拒绝跨版本通信，所以通过不匹配二进制做的解锁验证毫无意义。一次持续多日的测量系列就这样被作废了（580.159.03 用户空间对不同的内核模块构建）。如果你的用户空间和模块版本不同，你记录过的每一个 `memory.total` 读数都无效。

### 3.7 解锁没扛过关机 { #not-persistent }

**原因。** 旧版 NVIDIA 驱动和/或旧版 `cmpunlocker` systemd 服务的残留。

**修复。** 移除所有旧内核模块**和**旧 `cmpunlocker` 服务，然后重装。正式发布的 `remove.sh` 现在两者都做：它停止、禁用并删除 `/etc/systemd/system/cmpunlocker.service`，杀掉 `/opt/cmpunlocker/daemon/watchdog.py`，移除 `/lib/modules/*/updates/cmpunlocker/`，逐内核运行 `depmod -a`，重建 initramfs，并重新加载原版模块。参见 [Uninstall](uninstall.md) 和 [Recovery](recovery.md)。

正式发布工具上不需要 systemd 守护进程：补丁 0006 为两个设备 ID 都设置 `NV_FLAG_PERSISTENT_SW_STATE`，这实际上是内置的持久模式。

### 3.8 安装器拒绝运行 { #install-refuses }

`install.sh` 在以下情况下硬失败，什么都不做：

| 条件 | 消息 / 行为 |
|---|---|
| 不是 root | 立即退出 |
| `lspci -nn` 中没有 `10de:20b0`、`10de:20c2` 或 `10de:2082` | 退出 |
| Secure Boot 已启用 | `Secure Boot is enabled. Disable it before installing unsigned patched modules.` |
| `/lib/modules/$(uname -r)/build` 缺少内核头文件 | 退出 |
| 检测到的驱动不在 `driver/VERSION` 中 | `Installed driver is ${detected}, but cmpunlocker requires one of: 610.43.03,610.43.02.` |
| 内存总量落在所有配置桶之外 | `Could not detect 8GB vs 10GB card` |

Secure Boot 门控只在 `/sys/firmware/efi` 存在**且** `mokutil` 在 PATH 上时运行；在非 EFI 系统或未安装 `mokutil` 的系统上，检查被静默跳过，未签名模块随后会以 `nvidia: module verification failed: signature and/or required key missing - tainting kernel` 加载失败。

驱动版本检测顺序：`/proc/driver/nvidia/version`，然后 `nvidia-smi --query-gpu=driver_version`，然后扫描 `/lib/firmware/nvidia/<supported>/`，然后是 `/lib/firmware/nvidia/` 下排序最高的目录。参见 [Driver versions](driver-versions.md)。

### 3.9 检测到错误的卡配置 { #profile-detect }

`detect_card_profile()` 读取出厂 `nvidia-smi memory.total` 并分桶：

| 报告的 `memory.total` | 配置 |
|---|---|
| ≥ 60000 MiB | `8gb`（已解锁的 64 GB 卡） |
| 35000 到 59999 MiB | `10gb`（已解锁的 40 GB 卡） |
| 7680 到 8704 MiB | `8gb` |
| 9728 到 10752 MiB | `10gb` |
| 其他任何值 | 致命 `Could not detect 8GB vs 10GB card` |

前两个区间存在是为了让在已解锁卡上重装时仍能检测到正确配置。如果检测错误或 `nvidia-smi` 不可用，强制指定：

```bash
sudo ./install.sh --profile=8gb    # or --profile=10gb
```

### 3.10 第三个设备 ID `10de:20b0` { #device-id-20b0 }

`install.sh` 会检测到 `10de:20b0` 但警告并继续：

```text
In-driver unlock path is gated on PCI ID 0x20C2 / 0x2082.
This card reports 0x20b0; install will continue, but unlock may not activate.
```

补丁 0001 和 0002 中每一个解锁动作**和每一条 `SEC2_DEBUG` 打印**都只受 `0x20C2` / `0x2082` 门控，通过 `_kgspSec2PostblTimingEnabled()`，它测试 `pGpu->idInfo.PCIDeviceID >> 16`。所以 `20b0` 卡能干净安装、走完全原版路径，`dmesg | grep SEC2_DEBUG` 应该什么都**不**打印。

> [!NOTE]
> **未解决问题**
>
> 一位带 A100 工程样品硅片（`20B0`、8192 MB、2048 位、4096 个 CUDA 核心、三星 8Hi HBM2）的测试者报告了 `NVRM initialization error`，*而且* `SEC2_DEBUG` 确认寄存器被写入了。原版构建不可能在 `20B0` 卡上打印那些行。要么运行的是加了 ES ID 的修改构建，要么那些行来自同一主机里的另一张卡。从记录无法判定。

---

## 4. GSP 启动失败 { #gsp-boot }

### 4.1 WPR2 已经处于开启状态 { #wpr2-already-up }

**症状。**

```text
NVRM: _kgspBootGspRm: unexpected WPR2 already up, cannot proceed with booting GSP
NVRM: (the GPU is likely in a bad state and may need to be reset)
NVRM: RmInitAdapter: Cannot initialize GSP firmware RM
NVRM: RmInitAdapter failed! (0x62:0x40:2028)
NVRM: rm_init_adapter failed, device minor number 0
```

以 `nvidia-smi` 的 `No devices were found` 结束。这是漏洞利用后占主导地位的失败，至少三位测试者报告了相同的现象。

**原因。** 之前的一次 Booter 运行编程了 WPR2 MMU 寄存器然后脱轨，所以下一次 modprobe 时驱动看到 WPR2 已经开启就拒绝继续。

**修复（净室时代）。** 完整拆除驱动，然后通过 `echo 1 > /sys/bus/pci/devices/0000:BDF/reset` 做 FLR，或冷断电。

正式发布补丁还会在 PLM 循环前从 `0x001fa824` / `0x001fa828` 保存一次 WPR2 lo/hi，在**每次** Booter Load 尝试前重写两个寄存器，并在循环结束后再次重写。它从不清除它们。

### 4.2 Xid 119、60 秒超时、函数 4097 { #xid119-60s }

**症状。**

```text
Xid 119: Timeout after 60s of waiting for RPC response from GPU0 GSP!
  Expected function 4097 (GSP_INIT_DONE)
GSP RPC buffer contains function 4098 (GSP_RUN_CPU_SEQUENCER)
kflcnWaitForHalt_TU102: Timeout waiting for Falcon to halt
NV_ERR_TIMEOUT (0x00000065)  from kflcnWaitForHalt_HAL at kernel_gsp.c:5386 (or :5449)
falconMailbox 0:00000031
... then: WPR2 already up
```

**原因。** GSP RISC-V 核心从未到达 RM 初始化：启动是**挂起**而非被拒绝。

**修复。** 复位以清除 WPR2，然后重试：先 FLR，FLR 清不掉再用 SBR 或冷断电。见 [Recovery](recovery.md)。

**支撑细节。** 前面的 `_threadNodeCheckTimeout` 显示 4000 ms 的 Falcon 停止超时；GSP 事件本身花了 59 秒。一次抓取中 CPU 到 GSP 的 RPC 历史只包含条目 0 `SET_REGISTRY` 和条目 -1 `GSP_SET_SYSTEM_INFO`，意味着 GPU 从未越过早期引导。在两台主机、两个内核和两个驱动构建（580.159.03 和 580.167.08）上捕获过。

### 4.3 Xid 119、6 秒超时、函数 103 { #xid119-6s }

**这是与 4.2 不同的失败**。用函数号和超时长度区分。

**症状。** Xid 119 带 6 秒超时和函数 103（`GSP_RM_ALLOC`），发生在部分成功的启动之后：GPU 达到运行状态（nvidia-drm 加载、`GSP_RM_CONTROL` 和 `FREE` RPC 在 224 到 5222 µs 内完成），然后每个 `nvidia-smi` 都挂起，大约每 6 秒对连续序列号（184、185、186）重复一次 Xid。

**修复。** 复位卡；该状态无法原地恢复。

### 4.4 `GSP didn't boot`、状态 `0x65` { #gsp-0x65 }

**症状。** dmesg 中出现 `GSP didn't boot`，状态 `0x65`。

**原因。** 精心构造的签名缓冲区 / Booter 序列让 GSP 无法启动。`0x65` 是驱动侧 `NV_ERR_TIMEOUT`。

**修复。** 完全断电循环并重试。对报告它的测试者来说，仅移除旧内核模块**不**够。

> [!CAUTION]
> **`0x65` 不是 `0x31`**
>
> `0x65` 是驱动侧 `NV_ERR_TIMEOUT`；`0x31` 是一个 mailbox 值。它们发生在不同阶段。决定性测试：WPR2 错误只来自寄存器写入，而一个完全没有写入的双加载过程仍会撞上 `0x65`。早期声称两个代码是同一回事的说法，在一小时内就被一次受控的零写入运行反驳了。

为什么 FLR 有时无法恢复 `0x65` 卡死，在 [Recovery](recovery.md#flr-vs-sbr) 中覆盖。

### 4.5 Booter 错误 `0x35` { #booter-0x35 }

**症状（仅独立 / 无驱动工具）。** Booter 返回 `0x35`。

**原因。** `regtable_rw_indexed` 在 DMEM `0x2383` 和 `0x8e08` 读取 DMEM 寄存器描述符表，发现全是零。原版签名只有 `0x1000` 字节，所以它的 DMA 只到达 DMEM `0x17FF`，那些表保持完整。漏洞利用负载必须是 `0xF800` 字节，这样它的帧才能到达 `0xF748` 的栈，使 DMA 覆盖 DMEM `0x0800` 到 `0xFFFF` 并把表清零。阶段 MAIN.6 在 DMA 之前读表（完整），MAIN.7 在之后验证（已清零），触发 `0x35`。

**修复（无驱动路径）。** 在负载偏移 `0x1B83`（DMEM `0x2383`）和 `0x8608`（DMEM `0x8E08`）处包含原始原版表内容。那些内容不存在于任何平面文件中：它们是引导加载程序在运行时从引导加载程序代码和/或引导描述符中的常量生成的，所以重建它们本身就是一个大子问题。应用修复后研究者立即报告 GSP-RM 启动了。

> [!NOTE]
> **正式发布路径上不可达**
>
> 正式发布的驱动内补丁永远不会撞上 `0x35`。`kgspSec2PostblTimingRebuildStockSignature()` 在真正的 GSP-RM 启动前恢复真实的 4096 字节签名，所以那次启动的 DMA 只到达 DMEM `0x17FF`，描述符表保持完整。正式发布负载中没有 `0x1B83`/`0x8608` 恢复，也不需要。*（置信度：中等；依据正式发布代码加上 2026-07-20 的根因分析推理，未独立插桩。）*

### 4.6 Booter 错误 `0x54` { #booter-0x54 }

> [!NOTE]
> **未解决问题**
>
> **症状（PG199 / A100D、`10DE:20BB`、出厂 32768 MiB）：**
>
> ```text
> kgspBootstrap_TU102: kflcnResetIntoRiscv 0x0
> s_executeBooterUcode_TU102: Booter failed 0x54
> ```
>
> 失败前达到的状态：`MMU_LMR 0x0000020a -> 0x0000020b`；`FBPA_CFG1` 出厂 `0x22779000`，一个变体清除第 29 位得到 `0x02779000`，另一个保留 `0x22779000`；`SS0 0x53540175` 和 `SS1 0x00000000` 故意保持不变；WPR2 `07f68000/07fefe00 -> 1ffffe00/0`；PLM `ffffff8f/0004cb8f -> opened`。WPR2 留在失败初始化状态，因为 GSP 从未完成初始化。
>
> **没人能说清 `0x54` 是什么意思。** 它只在 A100D / PG199 硬件上观察到，从未在 170HX 上。寄存器写入被证明确实落地，所以问题很窄：找到 Booter 状态枚举。注意名为 `PG199` 的分支不含 A100D 支持，所以这项工作在仓库之外。

### 4.7 `RmInitAdapter failed! (0x62:0x40:2674)` { #rminit-2674 }

**症状。** 多 GPU 机器上的一张 GPU 完全没有成功初始化，而同一机器上另一张 GPU 到达 Xid 119。

**原因。** 未确认。这是一个真实、可复现的签名。在 OEM BTC B250 挖矿主板、内核 6.8.0-134-generic、驱动 580.159.03、开启 ACS 变通方案的 Intel SPT PCH 根端口上观察到。

### 4.8 `RmInitAdapter failed! (0x62:0xffff:2119)`、脏 SEC2 退出 { #rminit-2119 }

**症状。**

```text
s_executeBooterUcode_TU102: Booter failed with non-zero error code: 0x29
_kgspBootGspRm: SEC2_DEBUG: FAILED to open FBPA_008 (0x9a0008) after 2 attempts reg=0xffffff8f
kgspInitRm_IMPL: Max GSP-RM boot attempts exceeded: 4/4
NVRM: RmInitAdapter failed! (0x62:0xffff:2119)
```

**原因。** PLM 被一次脏 SEC2 退出部分锁住。`reg=0xffffff8f` 是线索：`0x8f` 是 "secure_teardown ran" 标记值。Booter `0x29` 来自 `check_1180f8_nibbles`，它要求 `0x001180f8` 的传入高半字节为 0。

**可复现 A/B。** 不带几何或算力写入、也不写 `0x1180f8` 触发时走得更远，只在 `FBPA_00C (0x9a000c)` 失败；只加上 `0x1180f8 = 0x17100000` 写入就让 `FBPA_008` 和 `FBPA_00C` 都失败。

**修复。** 获得干净的 SEC2 退出：冷启动，然后不带额外写入重新触发。

### 4.9 `0x24:0x72`、`BAR 0/BAR 2 failed.` { #bar2-0x72 }

**症状。** `RmInitAdapter failed!` 带 `0x24:0x72` 或 `0x72`，日志字符串 `"BAR 0/BAR 2 failed."` 在 `journal.c:4081`，即 `NV_ERR_MEMORY_ERROR`。

**原因。** 不是显存损坏，也不是 SCP 加密失败。前后探测显示 BAR0 到 vidmem 路径仍返回写入的模式 `0xabcdabcd`。失败的是 `kbusVerifyBar2` 中第二次、BAR2 虚拟（MMU 翻译）测试，因为真正的 Booter 在 `0x2777000` 到 `0x27fee00` 处雕刻了 WPR2，并在其正常 ACR 工作中设置了 FBIF `0x800` 位，而驱动的 BAR2 测试缓冲区 / 实例块落在那个写保护区域里。

**修复。** 在退出 heavy-secure 模式的路上拆掉 WPR2：

```text
0x1FA824 = 0x1FFFFE00
0x1FA828 = 0x00000000
```

**逃生口，从未用过。** BAR2 自检可以通过 `PDB_PROP_GPU_BROKEN_FB`、`gpuIsCacheOnlyModeEnabled` 或 `kbusIsBar2TestSkipped` 跳过。这些是在 `0x24:0x72` 仍阻塞启动时在源码中识别的，但 WPR2 拆除先修好了根本原因，所以它们保持已记录但未尝试。

一次无驱动漏洞利用运行会产生同样的 `0x72` 映射：它把 GPU 的 BAR2/L2/MMU（POST/DEVINIT）状态留在 ACR 配置下，所以 CPU-RM 的内存自检失败。

### 4.10 RM 初始化以 `rpc_result = 0xFFFF` 停滞 { #rpc-ffff }

> [!WARNING]
> **实验性**
>
> 历史性的、净室时代的单负载 rejoin 路径。部分成功的 rejoin 可以到达 GSP-RM 初始化，却仍以 `RPC_HDR->rpc_result = 0xFFFF`（`NV_ERR_GENERIC`）和 NULL `GSP-LOG[RM]` 缓冲区停滞，意味着 RM 初始化非常早就失败。那种状态下 Booter 已经完成（WPR2 设置好、`BOOTVEC = 0xfd00`、`finalize_1180f8` 观察到 `0x17100000` 对照已知良好的 `0x11000000`），驱动因为 MBOX0 被破坏为 `0x31` 而重新断言 GSP boot-args，恢复了 `WprMeta.sizeOfSignature = 0x1000`，并因 HS 锁定的寄存器给出假阴性而绕过 `kflcnIsRiscvActive`。2026-07-07 记录为 "the current wall"；RM 侧根因从未被识别，整个方法被驱动内补丁取代。*（置信度：中等。）*

一个相关状态：失败的 GSP 交接表现为 Xid 119 / `GSP_INIT_DONE` 超时，mailbox0 = `0x31`、`finalize_1180f8 = 0x11000000`、`BOOTVEC = 0xfd00`。那里 Booter 完成了其认证路径，但 RISC-V GSP 从未启动，因为没有发出 BCR 写入。在 `0x37b7` 和 `0x37cc` 返回给出了相同结果。

成功完成整栈 rejoin 后的参考"良好着陆"状态，供对照：

| 可观察量 | 良好值 | 含义 |
|---|---|---|
| `finalize 0x1180f8` | `0x11000000` | `[31:28]` 中半字节 1 加上 authenticate 的第 24 位；第 26 位（`BOOT_STAGE_3_HANDOFF`）**未**设置 |
| `GSP_FALCON_MAILBOX0` | `0x31` | GSP-RM 活着 |
| `GSP BOOTVEC` | `0xfd00` | |
| SEC2 resetPLM | `0x8f` | `secure_teardown` 运行过 |
| SEC2 MBOX0 | `0x0` | `report_status` 写了 r0 = 0 |
| `RV_STATUS 0x111240` | `0x33` 或 `0x35` | RISC-V 核心运行中（从未启动时为 `0x0`） |

### 4.11 挂起启动的 Falcon 核心转储 { #falcon-coredump }

挂起启动的非破坏性 Falcon 核心转储读取：

```text
falconMailbox 0:00000031        # PC hijack succeeded
riscvPc       00000000          # RISC-V core idle
riscvCpuctl   00000010
riscv mailboxes 0,1,2,3 = 0
riscvIrqmask / riscvIrqdest / riscvPrivErrStat / riscvPrivErrInfo
  / riscvPrivErrAddr / riscvHubErrStat = 0
falconIrqstat 00000000
falconIrqmode 0000fc24
fbifInstblk   00000000
fbifCtl       00000190
fbifThrottle  80000064
fbifAchkBlk   0:a2286560 1:370b1788
fbifAchkCtl   0/0
fbifCg1       0000000f
```

**解读：** 溢出控制了 Booter，但 GSP 核心从未启动。漏洞利用让启动挂起，而不是被签名检查拒绝。

### 4.12 可诊断性：补丁 0002 增加了什么 { #patch-0002 }

补丁 0002 的存在正是为了让 GSP 引导失败可诊断。它把致命的 `NV_ASSERT_OK_OR_RETURN` 宏转换成记录状态检查，产生：

```text
SEC2_DEBUG: FWSEC cmd is NULL, aborting
SEC2_DEBUG: kflcnReset for FWSEC: 0x%x
SEC2_DEBUG: kflcnResetIntoRiscv: 0x%x
SEC2_DEBUG: FWSEC: pPreparedFwsecCmd=%p frtsSize=0x%x
SEC2_DEBUG: FWSEC status=0x%x
```

用户被要求粘贴进工单的大多数 `SEC2_DEBUG` 行都源于这里。

### 4.13 签名大小阳性对照 { #sigtest }

如果你看到这个，它是**阳性对照**，不是失败：

```text
_kgspCreateSignatureMemdesc: kgsp: TEST sig override active:
  orig first 4096 B + /tmp/sig tail, total 23360 B, orig size: 4096
kgspBootstrap_TU102: [sigtest] DEVICE IS UP: GSP booted and RISCV is active
  (Booter accepted the signature)
```

这次运行在 580.167.08 上演示了漏洞。60 秒后它仍然撞上常见的 Xid 119 / WPR2-already-up 路径。

正式发布的签名缓冲区是 `0xf800` 字节（`SEC2_POSTBL_TIMING_SIGNATURE_SIZE 0x0000f800ULL`，63,488 字节），**不是** `0xf700`。一次社区复现卡在 GSP 二进制中 `fwsignature_ga100` 段只有 `0x1000` 字节而硬编码负载是 `0xf700`；解决方案是停止打固件补丁，改为从驱动扩大 `pSignatureMemdesc`。

---

## 5. 状态码目录 { #codes }

### 5.1 Booter 与 GSP 状态码 { #codes-booter }

| 码 | 含义 | 说明 |
|---|---|---|
| `0x00` | SEC2 MAILBOX0 干净退出 / GSP-RM 干净启动 | |
| `0x2` | 无效签名 | |
| `0x29` | 坏的 finalize 半字节，来自 `check_1180f8_nibbles` | 要求 `0x1180f8` 的传入高半字节为 `0` |
| `0x31` | Booter 拒绝 / SEC2 MAILBOX0 中的默认状态 | **取决于上下文**，见下文 |
| `0x35` | DMEM 寄存器描述符表读为零 | 仅无驱动路径，见 [4.5](#booter-0x35) |
| `0x47` | 金丝雀（canary）不匹配恐慌 | |
| `0x54` | 仅在 A100D / PG199 上观察到 | 含义**未知**，见 [4.6](#booter-0x54) |
| `0x59` | 可选 `dmem.bin` 的文件未找到 | 无害，见 [2.2](#benign-0x59) |
| `0x60` | 观察到过渡到 `0xffff` | |
| `0x62` | 驱动侧 `NV_ERR_RESET_REQUIRED`；也是固件初始化失败状态 | RmInitAdapter 三元组的首字段 |
| `0x65` | 驱动侧 `NV_ERR_TIMEOUT` | 见 [4.4](#gsp-0x65) |
| `0x72` | `NV_ERR_MEMORY_ERROR`，BAR2 自检 | 见 [4.9](#bar2-0x72) |
| `0xfe` | CPU-RM ACR 检测到触发后的 SEC2 状态 | 只有 FLR 能清除它 *（置信度：中等）* |
| `0xffff` | Booter Load 失败 / GSP-RM 初始化失败 | 每次负载通过都预期 |
| `0xFFFFFFFF` | GSP mailbox 未读 | |
| `0x15` | Booter 的 `csb_write` 错误路径，报告进 SEC2 MAILBOX0 | |

从实时 dmesg 读取的码置信度高，`0x54` 和 `0xffff` 的机制归因置信度低。有一种解读把 `0xffff` 归因于 Booter 在几何改变后把 FB 顶部的 WPR2 雕刻进无后备区域；这未定论。

**Mailbox 地址。** SEC2 MAILBOX0 是 BAR0 `0x00840040`；GSP mailbox 是 `0x00110040`。

> [!NOTE]
> **未解决问题**
>
> **mailbox `0x31` 的含义从未定论。** 存在三种互不相容的解读：(a) "初始值 / 尚无写入"，**被明确撤回**，因为 `0x31` 后来被发现是写入值（驱动的 boot-args 物理地址，被破坏），而且健康 GSP 启动会把 `0x110040` 复位为 0；(b) "ACR 互斥锁被持有"，早期坚持的解读；(c) "SEC2 Booter 自己的成功签名"，那么驱动的 `0x65` 只是 SEC2 坐在 `0x8f` 拆除状态下导致的 60 秒完成等待超时。第四种用法在良好着陆状态中把 `GSP_FALCON_MAILBOX0 = 0x31` 读作 "GSP-RM alive"。**把 `0x31` 当作观察，而不是诊断。** 什么能定论：SEC2 Booter 自己的状态枚举，或一个在 ACR 互斥锁被证明空闲的情况下产生 `0x31` 的受控实验。

### 5.2 `RmInitAdapter` 三元组 { #codes-rminit }

首字段 `0x62` 是 `NV_ERR_RESET_REQUIRED`。

| 三元组 | 含义 |
|---|---|
| `(0x62:0x40:2028)` | WPR2 已经开启，见 [4.1](#wpr2-already-up) |
| `(0x62:0x55:2028)` | `DEVICE FAILED TO COME UP: RISCV not active after Booter Load` |
| `(0x62:0x65:2028)` | RmInitDone 超时 |
| `(0x62:0x40:2674)` | 第二张 GPU 上的初始化失败，根因未知，见 [4.7](#rminit-2674) |
| `(0x62:0xffff:2119)` | `0x29` / FBPA 打开路径，见 [4.8](#rminit-2119) |
| `0x24:0x72:1220` | 10 GB 卡冷启动下游阶段，在 `RmInitNvDevice` 中；与 BAR2 案例是不同阶段 |

### 5.3 SEC2 reset PLM 可观察值 { #codes-resetplm }

在地址 `0x8403C4` 报告。GSP 对应物是 `0x001103d0`。

| 值 | 含义 |
|---|---|
| `0xff` | 干净；总线主控健康 |
| `0x8f` | `secure_teardown` 运行过（位 `[6:4]` 从 `0x7` 变为 `0x0`） |
| `0x00cf` | 驱动仍加载的部分触发状态 |

*（置信度：中等。它被一致地用作许多次运行的可观察标记，但 `0x8403C4` 处寄存器的身份在频道里被质疑过，理由是地址不在熔丝列表上，而且从未被独立记录。参见 [Register reference](../unlock/register-reference.md)。）*

FLR 清除 SEC2 reset-PLM 污染：`0x8f` 变回 `0xff`。

### 5.4 `0xbadfXXXX` 读 { #codes-badf }

`0xbadfXXXX` 读是**权限或存在性失败，不是存储的数据**。

| 模式 | 含义 | 示例 |
|---|---|---|
| `0xbadf5040` | 读被权限级掩码阻止 | `FECS_FEAT_OVERRIDE 0x00409664`、`FECS_FEAT_READOUT_1 0x00409668`、第二个功能覆盖组 `0x00823830`-`0x0082383c` |
| `0xbadf1100` | PRI 目标不存在 | `PMC_BOOT_42 0x0000a800`、GA100 上 `FUSE_OPT_FBIO_OLD 0x00021c14` |
| `0xbadf20NN`（`0xbadf2010`-`0xbadf201b`） | 目标存在但 FBPA 分区被筛选（floorswept）掉 | 低字节编码实例 |
| `0xbadf1002` | GA10x 变体的不存在哨兵 | 在 `0x00021C14` |
| `0xbadf5108` | 从 PL0 读 AON 安全 scratch | `0x001180f8`、`0x001182d0` |
| `0xbadf` 前缀一般 | 特权阻止的回读 | 例如 GSP falcon 启动块 `0x110280`-`0x110298` |

*（置信度：三个主要家族高，确切分类措辞中等。）*

### 5.5 其他驱动错误码 { #codes-other }

**`NV_ERR_INSUFFICIENT_RESOURCES (0x1A)`** 出现在 CUDA 失败上，指向 WPR meta 第二遍没有拾取解锁后的容量。用 `dmesg | grep -E 'Xid|NVRM.*rror'` 检查。*（置信度：中等；来自发布的指南，无独立复现加修复。）*

**`nv_gpu_ops.c:11190` 处 `NVA06F_CTRL_CMD_STOP_CHANNEL` 的 `NV_ERR_RESET_REQUIRED (0x62)`** 出现在分配越过设备真实解码边界时（在过度配置的卡上于 40 GB 处观察到）：

```text
nvAssertOkFailedNoLog: Assertion failed: Reset required [NV_ERR_RESET_REQUIRED] (0x00000062)
  returned from pRmApi->Control(...)
```

边界以下卡没问题，分配越过边界后通道停止失败。

**FLR 后写入 DMEM 出现 `EXCI 0x0a (MISS_INS)`** 意味着 Booter 不再驻留 IMEM：FLR 把它移除了。

---

## 6. 静默失败与运维陷阱 { #silent }

### 6.1 `rmmod nvidia` 清除总线主控 { #bus-master }

> [!CAUTION]
> **语料中在运维上最重要的一条陷阱**
>
> **`rmmod nvidia` 会清除 PCI `COMMAND.BusMaster`。** SEC2 Booter 通过 DMA 从系统内存获取 ROP 负载，所以总线主控关闭时它什么都取不到，带着空负载运行，不执行任何 ROP 并错误退出。**日志中没有任何地方提到 DMA。** 每一次写入都只是弹回，唯一可见的痕迹是 `resetPLM` 从 `0xff` 变成 `0x8f`。

**诊断：**

```bash
setpci -s <bdf> COMMAND
# 0x0102 = broken (bus master bit clear)
# 0x0546 = good   (bit 2, Bus Master, set)
```

**修复。** 触发前重新启用总线主控：

```bash
sudo setpci -s <bdf> COMMAND=0x0546
```

重触发工具中的 `prepare()` 添加了一个 `ensure_bus_master()` 调用以便自愈。修复后，每次触发 `resetPLM` 都保持 `0xff`。

这条陷阱适用于独立 / 无驱动工具。正式发布的驱动内补丁在活动驱动内运行，那里总线主控按定义是开启的。

### 6.2 驱动仍加载时触发 { #driver-loaded }

**症状。**

```text
PLMs: 1/9 open (fired 8 closed)
resetPLM=0x00cf
PRE  CFG1=0x02449000 LMR=0x00000288
POST CFG1=0x02449000 LMR=0x00000288
decode=0x70000300
CSTATUS=0/24
WPR2=['0x2779000','0x27fee00']
STATE NOT CLEAN, FLR + re-fire
EXIT_CODE=1
```

**修复。** 卸载驱动。驱动卸载后，同样的命令给出 `PLMs: 9/9 open (fired 0 closed)`、`resetPLM=0x00ff`、`CSTATUS=20/24` 和 `READY`。失败和修复在同一硬件上几分钟内背靠背观察到。

### 6.3 正确拆除驱动 { #teardown }

仅 `modprobe -r` 不够。可用的顺序：

```bash
systemctl stop nvidia-persistenced      2>/dev/null || true
systemctl disable nvidia-persistenced   2>/dev/null || true
systemctl stop gdm3 sddm lightdm display-manager 2>/dev/null || true
killall -9 Xorg Xwayland nvidia-persistenced     2>/dev/null || true
sleep 2
modprobe -r nvidia-uvm      2>/dev/null || true
modprobe -r nvidia_drm      2>/dev/null || true
modprobe -r nvidia_modeset  2>/dev/null || true
modprobe -r nvidia          2>/dev/null || true
sleep 2
lsmod | grep -q nvidia && rmmod -f nvidia_uvm nvidia_drm nvidia_modeset nvidia
```

每一步都有保护，缺失服务不会让脚本在 `set -e` 下中止。接下来的 FLR 是：

```bash
echo 1 | sudo tee /sys/bus/pci/devices/0000:${PCI}/reset
sleep 3
```

这个测试台跑了九个解锁循环。

### 6.4 模块无法卸载 { #module-stuck }

nvidia 模块经常无论如何都拒绝卸载，留下：

```text
nvidia 15835136 2
drm    753664 7 drm_kms_helper,drm_display_helper,nvidia,drm_buddy,i915,ttm
```

通过 `drm` 对 `i915` 的依赖是为什么解锁工作要在无头主机或不使用 NVIDIA 显示的主机上做的实际原因。那台系统上的模块大小：`nvidia_modeset` 2248704、`nvidia_uvm` 2039808。*（置信度：中等；在单个运行日志中反复观察到。）*

### 6.5 DMEM 写入被静默丢弃 { #dmem-locked }

**原因。** 一旦 `nvidia.ko` 把 SEC Falcon 引导进 heavy-secure（HS）模式，`DMEM_PRIV_LEVEL_MASK`（`0x00840284`）写保护读为 0，所有 DMEM 写入都被丢弃。Falcon 处于 HS 时 DMEM 既不能读也不能写。

**检测。**

```text
mask    = read32(0x00840284)
rd_prot = mask & 0x7
wr_prot = (mask >> 4) & 0x7      # 0 means LOCKED
```

功能测试：通过 DMEMC0/DMEMD0 把 `0xDEADBEEF` 写入 DMEM`[0x000]` 并读回。

**修复。** 一次 ENGINE 复位：

```text
wr32(PSEC_ENGINE, 0x1); sleep 10 ms; wr32(PSEC_ENGINE, 0x0)
poll DMATRFCMD for IDLE && !FULL
poll DMACTL & 0x6 == 0                 # scrub complete
check SCP_CTL_P2PRX bit 3 (SFK_LOADED)
check KFUSE_LOAD_CTL bit 0 set, bit 1 clear
```

或断电循环，在加载 `nvidia.ko` 前运行。

### 6.6 与活动 CUDA 上下文一起触发 { #live-cuda-context }

与活动 CUDA 上下文一起触发解锁**确实**会打开 FB 几何 PLM（`0x00100b10`：`0xffffff8f -> 0xffffffff`），但随后会挂起 `nvidia-smi`，因为让 SEC2 停在 HS（spin-park）会破坏驱动的健康路径。恢复是 `FALCON_ENGINE` 复位，它在不碰 FB 内容的情况下清除 HS 状态。*（置信度：中等。）*

### 6.7 在 `0x82xxxx` 块之外写安全寄存器 { #resetplm-8f }

在 `0x82xxxx` 之外写任何安全寄存器都会把 SEC2 reset PLM 重新提升为 `0x8f`，这会阻塞原版 `kflcnReset` 并使第二次 Booter Load 以 `0x65` 失败。点名的肇事者：`0x1183A4`（容量 scratch）、`0x9A0204`（FBPA strap）、`0x1FA8xx`（WPR）。只有 `0x82xxxx` 写入豁免。**这就是算力容易解锁而显存难的原因。** *（置信度：中等；症状可复现，带一致的 `resetPLM=0x8f` 标记，但寄存器身份被质疑过。）*

### 6.8 用复位把 PLM 打开与几何写入分开 { #flr-between }

**症状。** 同一个流水线在算力 PLM 上成功，但 `0x009A0204` 的 CFG1 写入弹回，三次尝试都读回出厂 `0x2449000` 而不是 `0x2779000`，以 `Pipeline complete: 0/1 GPU(s) unlocked` 结束。

**原因。** FB 几何 PLM **不在**常开（AON）岛里，而功能覆盖 PLM `0x00823804` **在**。一个先打开 PLM、做 FLR、再写几何的分阶段流水线会在 FLR 之间丢掉 FB 几何 PLM 状态。

**修复。** 永远不要用复位把 PLM 打开与几何写入分开。正式发布补丁在**一次** GSP 启动内完成两者。见 [Recovery: what survives a reset](recovery.md#state-persistence)。

### 6.9 已删除或错配的固件 { #firmware-deleted }

**症状。** 一次多日无法复现的"模型劣化"，带 `SEC2 MBOX0 = 0x0`（Booter 根本没有加载）。

**原因。** `/lib/firmware/nvidia/580.159.03/{gsp_tu10x.bin, booter_*.bin}` 已被删除，而 `.04` 用户空间 `nvidia-smi` 在 `.03` 模块上不会触发 GPU 初始化。

**修复。** 恢复版本匹配的固件目录并使用版本匹配的 `nvidia-smi`。恢复固件后立即复现了之前的工作状态。

**当时记录的实际教训：** agent 修改驱动时保留 diff 或变更日志，因为重装全新驱动会静默丢弃每一个需要的注入。

### 6.10 磁盘上陈旧的补丁版 `gsp_tu10x.bin` { #stale-firmware }

如果这台机器曾用过 cmpunlocker 的**固件补丁前代方案**，运行驱动内补丁前必须把 `gsp_tu10x.bin` 恢复到原版：

```bash
GSP_DIR=/lib/firmware/nvidia/610.43.03
sudo cp $GSP_DIR/gsp_tu10x.bin.cmpunlocker.bak $GSP_DIR/gsp_tu10x.bin
```

**为什么。** 驱动在启动期间把固件的签名保存为 "stock"。如果固件仍是补丁版，它保存的会是**漏洞利用负载**，干净的 GSP-RM 启动随后会 DMA 错误的 ROP 链。成功行是 `SEC2_DEBUG: saved stock signature (4096 bytes)`。

### 6.11 重复驱动加载使错误码向前走 { #nondeterminism }

重复的 CPU-RM 驱动加载会渐进地清理脏设备并使错误码向前走。一次单变量对照显示单次 MMU 失效运行保持在 `0x24`，所以更早的 `0x24 -> 0x25` 前进来自**双重加载**（CPU-RM 自己的部分初始化清理状态），不是 MMU 写入。这也解释了观察到的非确定性：脏设备清理是非确定性的，每次触发结果都有噪声。*（置信度：中等；底层清理机制从未被确认。）*

---

## 7. 构建失败 { #build }

`build.sh` 在 `set -euo pipefail` 下运行，所以任何失败的 hunk 都会中止构建。它把 `https://github.com/NVIDIA/open-gpu-kernel-modules/archive/refs/tags/${VERSION}.tar.gz` 下载到 `driver/.build/`（缓存，可用 `CMPUNLOCKER_BUILD_DIR` 覆盖），每次运行删除并重新解压一棵干净树，用 `patch -p1` 按字典序应用 `patches/*.patch`，然后运行 `make -j$(nproc) modules SYSSRC=/lib/modules/$(uname -r)/build`。构建时间约 5 分钟 *（单一报告，取决于硬件）*。

| 失败 | 原因 | 修复 |
|---|---|---|
| `Installed driver is X, but cmpunlocker requires one of: 610.43.03,610.43.02.` | 精确字符串版本白名单 | 安装 610.43.03 或 610.43.02 nvidia-open |
| 内核头文件错误，`/lib/modules/$(uname -r)/build` 缺失 | 未为**正在运行的**内核安装头文件 | 安装匹配头文件，或启动你为之构建的内核 |
| 补丁 hunk 被拒绝 | 错误的上游压缩包，或陈旧的 `.build/` 树 | 脚本每次运行都会重新解压；检查你确实在以为的分支上 |
| `python3: command not found` | `build.sh` 需要 `python3` | 安装它。**master 上不使用 PyYAML**，发布脚本中**没有显式 GCC 版本检查** |
| 首次安装无网络 | 压缩包下载 | 预填充 `driver/.build/` |
| `No initramfs tool found, rebuild manually before rebooting` | `update-initramfs`、`dracut`、`mkinitcpio` 都没有 | 手工重建 initramfs，见 [3.4](#initramfs) |
| 通过 mainline 在 Ubuntu 上换内核后构建坏了 | 换内核破坏了 NVIDIA 610 open 驱动构建 | 使用发行版内核 |

值得显式说明的前置条件：Secure Boot 关闭（模块未签名）、nvidia-open 610.43.0x（专有驱动"启动路径不同，不能以同样方式打补丁"）、**仅 Linux**（GSP 启动路径是 Linux 特有的）、root，以及为运行中的内核编译的模块。构建安装**五个**模块（`nvidia.ko`、`nvidia-modeset.ko`、`nvidia-uvm.ko`、`nvidia-drm.ko`、`nvidia-peermem.ko`），模式 `0644`，到 `/lib/modules/$(uname -r)/updates/cmpunlocker/`。只有 `nvidia.ko` 携带解锁代码；其他四个是原版重建。

因为补丁按 glob 应用，把名为 `0007-*.patch` 的第三方 diff 放进 `driver/patches/` 就能与解锁系列干净组合。这是叠加 P2P 补丁的记录在案机制。参见 [Driver patches](../unlock/driver-patches.md)。

**安装前移除。** 切换分支时维护者的规则是"always remove the old one before adding the new one." 一位克隆了 Gen2 分支并覆盖安装到现有安装之上的测试者报告说这样不行，先卸载就修好了。这是建议，不是铁律：至少另外两位测试者覆盖安装成功了。先移除再安装是*受支持的*路径。*（置信度：中等；没人找到差异化的因素。）*

---

## 8. PCIe Gen2 问题 { #gen2 }

> [!WARNING]
> **实验性**
>
> **`master` 上不发布任何 PCIe Gen2 补丁。** 补丁 `0007-pcie-gen2.patch` 和 `0008-pcie-gen2-probe-retrain.patch`，加上 `tools/retrain.sh`，只存在于 `Gen2`、`far`、`debug-gen2`（仅 0007 和 `tools/retrain.sh`）和 `deced` 分支上。`verify.sh` 是独立工具，随 `Gen2`、`far`、`deced` 和 `multiple-cards` 发布，见 [11.2](#verify-sh)。本节的每一样东西都适用于实验分支。

记住**速率和宽度是两个独立的成就**。第 1 代到第 2 代是驱动和固件解锁。超过 x4 宽度需要物理焊接 24 颗 0402 X7R 电容到通道 4 到 15 上。两者互不影响。参见 [PCIe Gen2](../unlock/pcie-gen2.md) 和 [Physical mods](../operations/physical-mods.md)。

### 8.1 Gen2 在有些机器上工作，有些不行 { #gen2-hardcoded-bdf }

**根因：硬编码的 PCI 地址 `0a:00.0`。** 硬编码在*用户空间辅助工具* `tools/retrain.sh` 中，共三处（`SYS=/sys/bus/pci/devices/0000:0a:00.0`、`GPU, UP = "0a:00.0", "09:01.0"`、以及 `resource0` 路径），**不在**内核补丁中：`0008-pcie-gen2-probe-retrain.patch` 在 `Gen2` 和 `deced` 之间逐字节相同。

`deced` 分支（提交消息："Stupid mistake - it appears to be hardcoded"）用 `find_gpu_bdf()` 取代它，通过 `lspci -d 10de:20c2` / `lspci -d 10de:2082` 发现卡，等待最多 120 秒的 `resource0` 和 `nvidia-smi -L`，并用 `readlink -f` 推导上游桥接器。**`Gen2` 和 `far` 分支仍包含硬编码。**

### 8.2 安装后 PCIe 仍是第 1 代 { #gen2-still-gen1 }

**第一项检查：IOMMU 直通。** `Gen2`、`far` 和 `deced` 安装器都会通过 `/etc/default/grub` 或 `/etc/kernel/cmdline` 把 `intel_iommu=on iommu=pt`（Intel）或 `amd_iommu=on iommu=pt`（AMD）追加到内核命令行，替换冲突条目，把文件备份为 `*.cmpunlocker.bak`，重新生成引导配置，每个分支自己的 `remove.sh` 恢复它。`--no-iommu` 选择退出。Master 完全不碰这些，所以用你安装时用的那个分支卸载。IOMMU 还必须在 BIOS/UEFI 中启用（VT-d / AMD-Vi / SVM）。

**第二项检查：你的检出是最新的吗？** 2026-07-29 之前 Gen2 补丁只存在于分支上，用户因为待在 `master` 上而反复失败。Gen2 现在在 `master` 中，所以要排除的是早于那次合并的检出。

**第三：重训练可能中途退出了。** 独立 `retrain.sh` 在四种情况下带着打印的原因提前退出：

| 消息 | 条件 |
|---|---|
| `retrain: BAR0 dead; skip` | BAR0 或 CYA 读 `0xFFFFFFFF` |
| `retrain: DIS_G2 still set; skip` | `DIS_G2`（BAR0 `0x8c2c0` 第 2 位）仍设置 |
| `retrain: Cap Gen1; skip` | 链路能力低于 Gen2 |
| `retrain: preconditions failed; skip` | 写入后前置条件失败 |

如果 `nvidia-smi` 不可用、显存读 `[N/A]`、链路已经是第 2 代、或最大链路代数不是 2、3、4，它还会以状态 0 **静默**退出。

驱动内重训练在探测时运行，在 `msleep(50)` 后最多轮询 2 秒（20 次 × 100 ms）。它清除 BAR0 `0x8c2c0` 第 2 位（DIS_G2），把 `0x8c040` 位 `[19:18]` 强制为 2，向 `0x8872c` 写 `0x00000006`，在 GPU 和上游桥接器上都设置 `PCI_EXP_LNKCTL2_TLS_5_0GT`，然后在上游桥接器上设置 `PCI_EXP_LNKCTL_RL`（重训练链路）。它的失败打印：

```text
CMP Gen2: no upstream PCIe bridge; skipping link retrain
CMP Gen2: cannot map BAR0; skipping link retrain
CMP Gen2: PCIe capability access failed (%d); skipping link retrain
```

加上 [2.7](#benign-retrain-false-negative) 中描述的假阴性。

### 8.3 根端口不改变速率 { #gen2-root-port }

> [!WARNING]
> **实验性**
>
> 有记录但未经独立确认。如果 `sudo dmesg | grep "SEC2_DEBUG.*Root port"` 说 "upstream port not valid"，说明芯片组驱动没有枚举上游端口；建议的变通方案是 `setpci -s <root_port> <offset>.w=0002` 后跟一次链路重训练。如果 "Root port LnkCtl2" 显示写入生效但速率仍是 1，根端口可能不支持定向速率改变；建议的修复是通过设置第 5 位的 `setpci -s <root_port> <link_ctrl>.w` 做根端口发起的重训练。**语料中没有任何一个变通方案成功的测量。**

### 8.4 虚拟机中的 Gen2 { #gen2-vm }

显存和算力解锁在 Proxmox 直通下可用（一位运营者直通了八张 8 GB 卡，全部解锁）。**截至 2026-07-24，PCIe Gen2 链路速率改动在虚拟机中不工作**，维护者已承认。重训练序列需要的是配置空间还是被虚拟机管理程序拦截的链路层访问，尚未确定。

> [!NOTE]
> **未解决问题**
>
> 一台主机（ASUS X99-A、LGA2011）报告只有一个插槽是 Gen2 x1，其余都是 Gen1 x4，在尝试了 IOMMU/虚拟化设置和全部四个插槽之后。它于 2026-07-27 报告，正是硬编码 BDF 发现落地的那一天。插槽依赖正是硬编码 BDF 会产生的东西，所以这很可能已在 `deced` 分支上修复，但从未被确认。

### 8.5 `RMPcieLinkSpeed` 分裂 { #gen2-linkspeed }

> [!NOTE]
> **未解决问题**
>
> 两个分支家族发布不同的注册表值，每个作者都认为自己的是对的：`debug-gen2` 和 `Gen2` 写 `NVreg_RegistryDwords="RmForceEnableGen2=1;RMPcieLinkSpeed=0x1"`，而 `far` 和 `deced` 写 `…=0x2`（由一个标题为 "Remove clamp link to Gen1" 的提交引入）。README 声称 Gen2 可用的 `Gen2` 分支发布 `0x1`。**不存在 A/B 启动测试。** 两个值都不应被当作权威。在一张卡上做一次三方启动对比就能定论。

---

## 9. 黑屏 { #black-screen }

### 9.1 运行解锁脚本时黑屏 { #black-screen-script }

> [!NOTE]
> **未解决问题**
>
> 一位测试者在 2026-07-20 报告运行解锁脚本时黑屏。他自己未经证实的假设是装错了驱动，并且拒绝进一步调试。那个线程里的两位测试者都在限于 PCIe 第 3 代的主机上。什么都没确定。

### 9.2 一般的无头启动 { #headless }

CMP 170HX 没有视频输出，所以一些主板在它作为唯一卡时不会 POST（一块 ASRock X370-I 被报告拒绝）。规划一张第三张可显示的卡，或确认主板能无头启动。另见 [Host will not boot](#no-post)。

---

## 10. 运行时、CUDA 与工作负载失败 { #runtime }

### 10.1 Xid 31、MMU 故障、区域违规 { #xid31 }

**症状。** `Xid 31` MMU 故障带 `FAULT_INFO_TYPE_REGION_VIOLATION`；卡在重启前 CUDA 不可用。一次抓取显示：

```text
ENGINE GRAPHICS HUBCLIENT_FE  faulted @ 0x7fad_3a200000 ... ACCESS_TYPE_VIRT_WRITE
ENGINE CE2      HUBCLIENT_HSCE2 faulted @ 0xf_f7400000 ... ACCESS_TYPE_PHYS_WRITE
```

**原因。** 分配越过解锁窗口可用顶部的分配。物理地址 `0xf_f7400000` 是 63.86 GiB，正好在 64 GB 窗口顶部。

**修复。** 少卸载一层 LLM 到那张 GPU。恢复需要完整重启。*（建议的修复未确认被应用。）*

> [!NOTE]
> **Xid 31 本身不是 80 GB 的特征错误**
>
> 在 80 GB 下，触碰超过约 40 GB 的内核会导致致命 GPU 丢失，与功耗上限无关。报告的错误码包括 Xid 31（被描述为无害）和 CUDA 内存测试后的 Xid 154；主要报告症状是卡死。Xid 31 的说法来自一位旁观者，并未被拥有故障卡的运营者证实为*那个*特征错误。80 GB 图景其余已裁决的部分：报告约 81920 MiB / 85,545,582,592 字节，`cudaMalloc` 分配 77 GiB 成功。参见 [80 GB](../frontier/80gb.md)。

### 10.2 杀死 CUDA 任务后 Xid 45 { #xid45 }

用 SIGKILL 杀死活动的 CUDA 验证内核可能以 Xid 45 卡死卡并强制一次复位循环。发布工具带有警示：在**前台**运行，绝不中途 SIGKILL；内核启动之间按 Ctrl-C 可以。密集 fill/check 内核在 64 GB 卡上运行很长时间（超过一百万个 64 KB 页），所以后台运行再杀掉它们的诱惑是真实存在的。*（置信度：中等；表述为来之不易的运维警告，没有附 dmesg 抓取。）*

### 10.3 过度配置配置上的 Xid 154 { #xid154 }

Xid 154 是过度配置的 80 GB 配置在 CUDA 内存测试后的主要失败，把卡限制为每次触发一个 CUDA 上下文。一位测试者只能让 GSP-RM 工作，CPU-RM 不行；另一位必须在尝试之间冷循环整个系统，而不只是重载驱动。两人都同意显存物理上可被 CUDA 访问：未解决的问题是保留与稳定性，不是可寻址性。两位测试者在不同硬件上独立复现。

> [!NOTE]
> **未解决问题**
>
> 4 GB/通道解码重新启用和 CUDA `719` → Xid 45 → Xid 154 链是同一个原子故障的两个结果，无法分开。对 40 GB 以上页面的原子操作（由 UVM 持有在主机上，因为设备只解码 40 GB）会出错；CPU-RM 把页迁移上去并重新启用 4 GB/通道，但同一个故障毒化了 CUDA 上下文，表现为 `719 unspecified launch failure`，然后 Xid 45，然后 Xid 154。尝试干净交接（用一个小型 managed "keeper" 翻转解码、释放它、然后分配非 managed 的 77 GiB）持续出错。一个注意到但未测试的相关旋钮：`PDB_PROP_GPU_RECOVERY_SQUASH_XID154`。

### 10.4 vLLM 在高显存利用率下崩溃 { #vllm }

**症状。** vLLM 卡在 `gpu-memory-utilization 0.95` 崩溃。

**原因。** 解锁几何暴露 65052 MB，但实际可用只有 64733 MB，所以 0.95 的头寸很薄。

**修复。** 降到 0.9，恢复了卡。另一次长时间多卡会话只在 0.95 带超大上下文时看到一次瞬态 "GPU requires reset"，随后自愈。**指导：保持利用率在 0.90 或以下。** *（置信度：中高。）* 参见 [LLM inference](../operations/llm-inference.md)。

### 10.5 到处 `cuInit` 返回 999 { #cuinit999 }

**症状。** `nvidia-smi` 仍报告健康，而每个框架的 `cuInit` 都返回 999。

**原因。** 反复对活动的多 GPU 任务 `kill -9` 会留下约 32 个僵尸 CUDA 进程并卡死主机 CUDA 运行时。

**修复。** 主机重启。**这无法在容器内修复。** 不要对活动的多 GPU 任务 `kill -9`。

在同一个完整的 8 卡会话中，数百个 60 秒健康采样里硬故障数为 **0**。这是操作者引发的失败，不是硬件故障。

### 10.6 分配越过真正可用容量 { #alloc-crash }

即使 `nvidia-smi` 报告更大数字，分配越过真正可用容量也会让基准测试崩溃。三张各报告 81920 MiB 的卡上，`llama-server` 持有 37798 / 47400 / 53960 MiB 时把运行搞崩了。重启后同一机器每卡加载约 32 GB（27734 / 31758 / 32754 MiB），基准测试完成，结果与 10 GB → 40 GB 配置大致相同。

### 10.7 烤机下的内存错误 { #burn-errors }

**症状。** 解锁超过其稳定几何的卡在算力烤机开始后几分钟内累积内存错误：

```text
2.1%  proc'd: 777 (12153 Gflop/s)   errors: 24433  (WARNING!)  temps: 85 C
```

错误出现在头几分钟。12153 Gflop/s 数字显示算力解锁是活跃的。**稳定的 10 GB → 40 GB 配置干净通过 5 分钟 gpu-burn**，8 GB → 64 GB 配置稳定且在生产中。

> [!NOTE]
> **未解决问题**
>
> 那些 85 °C 的 gpu-burn 错误是热的还是显存超频造成的，从未定论。一种立场："too hot, dial it back"。另一种："85 °C is within spec"，核心和显存温度彼此相差几度；第三位观察者称之为 HBM 硬件错误。其他人报告两张保持在 73 °C 以下的卡零错误。分支作者实际采用的解决方案是降低显存倍频，而不是要求更好的散热。故障卡是三星显存部件。什么能定论：同一张卡、同一倍频、强制散热保持在 70 °C 以下。参见 [Thermals](../hardware/thermals.md)。

> [!WARNING]
> **实验性**
>
> 一个相关说法，以低置信度提出且被其作者自己打了折扣："normal stress tests don't load an unlocked card because the fuses rely on the math being thrown at them." 从未针对已知良好的工作负载测试。它很重要，因为它决定 gpu-burn 是否足够作为稳定性测试。背景：一位测试者在打补丁前用标准压力测试无法把卡推到 68 W 以上。

### 10.8 `nvidia-smi --gpu-reset` 拒绝 { #gpu-reset-busy }

> [!NOTE]
> **未解决问题**
>
> 没有进程持有 GPU 时 `nvidia-smi --gpu-reset` 仍以 "GPU is being used by another process" 失败。**未解决。** 下一步：用 `fuser -v /dev/nvidia*` 和 `lsof /dev/nvidia*` 枚举持有者，检查是否有泄漏的 `nvidia-persistenced` 或产生 `cuInit=999` 的那类僵尸 CUDA 进程。
>
> 同一窗口内报告的恢复不顺：冷启动后卡有时需要物理拔插 PCIe 供电线，而 CUDA 别名测试会泄漏其分配，所以两次运行之间需要 SBR 恢复加驱动重载。

### 10.9 解锁有效但 CUDA 不行 { #cuda-clean-host }

**解决一个案例的分流步骤：把卡移到干净主机。** 原主机的软件栈被怀疑是罪魁祸首；根因从未被确认。归咎于解锁之前，先在干净主机上测试卡。*（置信度：中等。）*

注意卡在**原版** Linux NVIDIA 驱动上完全无补丁地运行（Ubuntu 24.04 上 `nvidia-driver-570` 加 CUDA 12.8 开箱即用），所以"卡能不能驱动"和"卡有没有解锁"是可以分别测试的两个独立问题。

### 10.10 纯粹作为运行时变通方案存在的补丁 { #runtime-patches }

三个正式发布补丁的存在只为修复解锁后的运行时故障。如果你在调试运行时故障，要知道这些已经应用了：

| 补丁 | 作用 |
|---|---|
| 0004 `bar0-pramin-clamp` | 每当 `0x20C2`/`0x2082` 上 `fbAddrSpaceSizeMb > 0x2000` 时，把 BAR0 PRAMIN 窗口钳制回基于出厂 8 GB 的偏移 `(0x2000ULL << 20) - DRF_SIZE(NV_PRAMIN)`，使窗口在几何改变后不会落到真实孔径之外。注意 10 GB 卡在 10240 MB 时已超过 `0x2000`，所以那里钳制也会介入 |
| 0005 `ce-scrub-workarounds` | 强制 `*pteKind = NV_MMU_PTE_KIND_GENERIC_MEMORY`（而不是 `..._COMPRESSIBLE_DISABLE_PLC`），并为此类卡禁用基于 VAS 的 CE 清理器路径 |
| 0006 `persistent-sw-state` | 为两个设备 ID 都设置 `NV_FLAG_PERSISTENT_SW_STATE`，使 RM 在最后一个客户端关闭时不拆除软件状态 |

---

## 11. 多卡机器 { #multicard }

### 11.1 解锁在多 GPU 机器上静默什么都不做 { #multicard-silent }

**症状。** 热重启和冷重启后全部五张 8 GB 卡保持出厂；验证器对每个 BDF（01:00.0、05:00.0、06:00.0、07:00.0、12:00.0）报告 `MISSING` 和 `✗ 0000:01:00.0: not found in nvidia-smi`，每个 `20c2 / 8gb` 预期约 65536 MiB，加上 `! No SEC2_DEBUG lines in dmesg`。

**原因。** 早期工具没有多卡处理。同一个人的单卡机器用同样的驱动就工作。

**修复。** 对于双卡 HiveOS 案例：`remove.sh`、重启、重装。两张卡随后都以 40 GB 起来。之后增加了 `multiple-cards` 分支和 `verify.sh`，但**尚未合并进 master**：master 的 `install.sh` 仍然通过 `head -1` 只取第一条匹配的 `lspci` 行。

另见 [depmod picking one module arbitrarily](#module-resolution)，这是另一种外表相同的多 GPU 失败。

### 11.2 解读 `verify.sh` 输出 { #verify-sh }

> [!WARNING]
> **实验性**
>
> `verify.sh` 只存在于 `deced`、`multiple-cards`、`Gen2` 和 `far` 分支上。三条值得识别的诊断字符串：
>
> * `<bdf>: not found in nvidia-smi`，状态 `MISSING`
> * `No SEC2_DEBUG lines in dmesg (logs may have rotated; unlock can still be OK if memory is unlocked)`
> * 致命的 `<N> GPU(s) failed unlock verification. Cold reboot if modules were just installed.`
>
> 它把设备 ID 映射到配置（`20c2 -> 8gb -> 65536 MiB`、`2082 -> 10gb -> 40960 MiB`），并从 `/lib/modules/$(uname -r)/updates/cmpunlocker/gpu_inventory` 读取清单。

### 11.3 HiveOS 十卡案例 { #hiveos }

> [!NOTE]
> **未解决问题**
>
> HiveOS beta 24.04 带十张 CMP 170HX 和 nvidia 610.43.03：`install.sh` 干净完成，但冷重启后补丁模块没有加载。除了重跑安装之外没有尝试别的，也没有发布修复。最有希望的下一步：运行三步分流（[3.5](#triage-three-step)），特别是比较 `/sys/module/nvidia/srcversion` 与 cmpunlocker `.ko`，并检查 HiveOS 自己的驱动包是否会重装覆盖补丁模块，或 initramfs / DKMS 排序是否把原版放在前面。两个候选都被点名但未确认。

参见 [Multi-GPU](multi-gpu.md)。

---

## 12. 主机与硬件级失败 { #host }

### 12.1 装上卡后服务器无法启动 { #no-post }

**症状。** 无蜂鸣码、无主板诊断 LED、"No display adapter, press F1" 已禁用。

**记录案例中的原因：与 GPU 无关。** 更换 PCI 插槽重命名了网络接口（从一个 `XXX5XX` 到一个 `XXX6XX` 的可预测名称），所以机器无头启动了但没有 IP。

**修复。** 修复网络配置。

**诊断时给出的一般建议：** 使用正确的电源线（ATX/EPS 型接头不带转接器，PCIe 接头带转接器），一次试一张卡，硬件改动后预期两到三次重启。卡需要一根 EPS 8-pin（额定 300 W），需要一个 2 × PCIe 转 EPS 转接器。参见 [Power and PSU](../operations/power-and-psu.md)。

### 12.2 虚拟机 { #vm }

**Proxmox 直通需要 SeaBIOS，不要 UEFI/OVMF。** UEFI 会产生模拟漏洞利用根本没生效的 RM 初始化/适配器失败。一个人在意识到自己旧的可用 VM 是 SeaBIOS 之前，在 "rm init adapt failures" 上花了大量时间；第二位成员立刻认出这就是自己无法复现的原因。

至少部分漏洞利用开发是针对直通给 QEMU Q35 VM 的 GPU 而不是裸机做的（`QEMU Standard PC (Q35 + ICH9, 2009)`、BIOS `rel-1.17.0-0-gb52ca86e094d-prebuilt.qemu.org 04/01/2014`、GPU 在 `0000:01:00`、Ubuntu 内核 6.8.0-136-generic、nvidia-modeset 580.159.03）。那里的崩溃发出坏帧指针栈展开警告，故障路径穿过 `nvidia_drm`/`nvidia_modeset`（`EnumerateGpus -> AllocateDevice -> nvkms_open_gpu`）。

显存和算力解锁在 VM 中工作；PCIe Gen2 不工作，见 [8.4](#gen2-vm)。

### 12.3 卡掉出 PCIe 总线 { #off-bus }

**症状。** 一张卡跑了一小时，然后永久掉出 PCIe 总线，不再被检测到。如果 BAR0 读 `0xffffffff`，卡已离总线。先试 [Recovery](recovery.md#reset-ladder) 中的恢复阶梯，包括 `echo 1 > /sys/bus/pci/devices/$BDF/remove` 后跟 `echo 1 > /sys/bus/pci/rescan`。

如果它再也不回来，原因可能是硬件。A100/170HX 级硬件上有一例完全诊断的实例：

**原因。** 一颗失效的 GS7155NVTD 3.3 V LDO 把 `PS_5V_PGOOD` 网络短路到 5 欧姆，阻止了 MP1475DJ 5 V 转换器启动。表现为打嗝模式保护：几十微秒后重试的几十纳秒瞬间 SW 节点脉冲。

**诊断路径。** 12 V 输入电感对地读高阻抗，插槽 12 V 无短路（排除大核心短路）；核心侧输出电感无电压、无开关；拆下 MP1475DJ 后，其空焊盘的引脚 1（Power Good）对地测得 5 欧姆。

**修复。** 更换 MP1475DJ 和 GS7155NVTD，然后从 U816 跳接 `PS_5V_PGOOD`。结果：3V3_SEQ 恢复，开关恢复，NVVDD 1.0 V 和 PEXVDD 恢复，GA100 在 PCIe 上被重新检测到。`PS_5V_PGOOD` 馈给排序 PEXVDD、NVVDD、1V35 和 1V8 的 SN74LV1T08 AND 门，并启用产生 3V3_SEQ 的 LDO。

> [!CAUTION]
> **信任新焊接的 GS7155NVTD 之前先做台架测试**
>
> 把 7.68 千欧反馈电阻换成 20 千欧以把输出从 3.3 V 重新编程为 1.8 V，在 5 V 轨上注入 3.3 V，确认稳压 1.8 V，然后恢复 7.68 千欧元件。要防御的危险是**开路反馈引脚**（QFN 上可能的冷焊失效），它会让 LDO 看到永久欠压并把输出驱动到最大，把全部 5 V 放到 3.3 V 轨上，毁掉几乎所有 3.3 V 逻辑。这块 8 到 12 层板的返工基准：任何芯片可移除前需要 420 °C 热风 2 分钟。GS7155NVTD 是 GSTEK QFN 部件，完整数据手册受 NDA 保护。

### 12.4 到货时的卡况 { #dirty-cards }

前矿卡到货时很脏：厚灰、锈蚀的 PCIe 挡板、散热器内的盐壳、裸露且无接头盖的金手指。使用前需要清洁、重新涂硅脂和换导热垫。**外观状况不是解锁失败的预测指标：** 一张明显肮脏的卡首次尝试就干净解锁到 64 GB。*（置信度：多次独立开箱的状况报告高；"不是预测指标"的结论中等，基于一个样本。从未发布过批次级解锁良率。）*

> [!WARNING]
> **实验性**
>
> 一个未被质疑但无数据支撑的立场认为，长期欠冷却的 HBM "should be dead by now unless it's had a very low operating time"，而且 HBM 一旦超过安全温度会快速劣化。许多卡可能是近零小时的，因为 CMP 170HX 在 9 月发布、市场到 11 月已无利可图。没有失效率或温度数据支持或反驳这一点。

2026 年 7 月下旬涨价期间卖家的"defective batch"说法**不是**真实硬件缺陷群体的证据：它被用作对曾显示正常卡的列表的取消借口。在记录在案的案例中，没有任何有缺陷的卡被实际发货或被诊断。

### 12.5 扛过冷启动的不可中断睡眠卡死 { #d-state }

**症状。** 一张 10 GB 卡陷入"不可中断睡眠"状态，扛过了大约五次冷重启，并阻止 Ubuntu 关机。

**原因。** 自动加载的补丁内核驱动，不是卡。

**修复。** 断开显卡启动（或 `blacklist nvidia`），然后清理。*（置信度：中等；根因由受影响的测试者在恢复后确认。）*

这个问题的更难变体在 [Recovery](recovery.md#bricking) 中被诚实讨论。

---

## 13. 升级与报告 { #escalation }

`install.sh` 把带时间戳的日志写到 `logs/install_YYYYMMDD_HHMMSS.log`；`remove.sh` 写 `logs/remove_YYYYMMDD_HHMMSS.log`。仓库目录不可写时 `remove.sh` 回退到 `/tmp`；`install.sh` 不回退，而是在启动时中止。**任何支持请求都附上最新的安装日志。**

一份有用的报告包含：

1. 操作系统与版本、内核（`uname -r`）
2. GPU 型号与驱动版本
3. 整个主机的 `lspci -nn`
4. 完整的 `sudo dmesg | grep SEC2_DEBUG`
5. 最新安装日志
6. `cat /lib/modules/$(uname -r)/updates/cmpunlocker/{driver_version,card_profile,unlock_geometry}`

响应是单运营者且缓慢：第一张有记录的 Gen2 工单等了约 10.5 小时才得到首次回复（06:21 开出，16:59 回复）。

---

## 14. 没有已知修复的症状 { #unsolved }

> [!NOTE]
> **未解决问题**
>
> 这些被记录下来，以免被当作新问题重新发现。没有一项有已发布的解决方案。
>
> * **冷重启后 `NVRM initialization error`**，在 Ubuntu 24.04、内核 6.8.0-111-generic、驱动 610.43.03 上。冷重启（之前为同一位测试者清除了 srcversion 不匹配）没有帮助。`SEC2_DEBUG` 行存在且寄存器正在被写入，这指向寄存器写入后的初始化失败而非失败的解锁链，但没人跟进。下一步：抓取 `SEC2_DEBUG` 块**之后**的完整 dmesg，包括 `normal BooterLoad status` 行和任何 `RmInitAdapter` 三元组，以把失败定位在序列中。
> * **运行漏洞利用后立即首次 `insmod` 普通驱动时内核恐慌加重启。** 2026-07-01 被问过一次，从未回答。下一步：抓取恐慌（串口控制台或 `pstore`）；可类比的 QEMU 抓取中故障路径穿过 `nvidia_drm`/`nvidia_modeset`，所以触发前卸载它们是便宜的第一测试。
> * **缺少 iGPU 或 BMC 显示设备会让 GSP 不高兴吗？** 一个观察，无确认、无反驳、无错误字符串。在一台机器上、BIOS 中禁用 BMC 显示设备做一次 A/B 就能回答。
> * **80 GB 不稳定：`cuda_memtest` 在重启后立即一次性通过全部 80 GB，之后每次重试都失败。** 功耗限制到 100 W 和供电假设都已被排除。重启依赖指向内存训练或刷新状态。这是 80 GB 配置上最具体的剩余线索。参见 [80 GB](../frontier/80gb.md)。
> * **Ubuntu 与 Arch 之间的显存解锁失败。** 一种解读：两个 PCIe 设备之间的内存地址冲突，一个非 170HX、非 2080 的设备（推测是 M.2 SSD）试图读取 IOMMU 拒绝的地址。受影响的测试者自己的解读：Ubuntu 安装只是配置错误。只有变通方案（在不同 M.2 SSD 上装不同 OS）被验证。当时推荐的第一诊断：`lspci -s 06:00.0`。
> * **PLM 的期望数量。** 独立工具报告 "9/9 open"，一位评审者预期 "0 or 26, not 1"。正式发布的驱动内路径恰好打开 4 个。这些是不同的 PLM 清单，但记录中没有任何东西把 9 项或 26 项列表映射到正式发布的 4 项 `plmTable`。

整个项目未解决项的完整列表见 [Open questions](../frontier/open-questions.md) 和 [status board](../frontier/status-board.md)。

---

## 相关页面

* [Recovery](recovery.md)：冷启动、FLR、SBR 以及什么真正持久
* [Verify](verify.md)：完整安装后清单
* [Install](install.md)：受支持的流程
* [Uninstall](uninstall.md)：`remove.sh` 与手动回滚
* [Driver versions](driver-versions.md)：哪些版本受支持并经过启动测试
* [Multi-GPU](multi-gpu.md)：多卡安装
* [Privilege level masks](../unlock/privilege-level-masks.md)：PLM 表做什么
* [Register reference](../unlock/register-reference.md)：本页点名的每一个寄存器
* [Dead ends](../history/dead-ends.md)：被尝试并反驳的假设
