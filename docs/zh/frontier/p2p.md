# 对等直连与多卡

## 本页涵盖

两块 CMP 170HX 卡能否直接互相对话、第三方 `aikitoria/open-gpu-kernel-modules` P2P 补丁做什么、它如何以文档化的三提交 diff 叠在 [cmpunlocker](../unlock/driver-patches.md) 之上、IOMMU 和 BAR 大小与之有什么关系，以及还有什么未知。

**简短答案：这张卡上默认没有对等直连。** 没有正式发布的代码启用它，语料中的每个测量都报告它不可用。唯一一个可能改变这一点的第三方补丁确实能在解锁之上构建和加载——这本身就是一个有用的结果，因为它证明 cmpunlocker 构建系统可以与无关的驱动 diff 组合。此后一位构建者报告该补丁在 GA100 上"半工作"：对等*数据移动*以 6.25 GB/s 运行，而对等*同步*完全不起作用，这会让 NCCL 和每个其他集合通信库挂起。该报告**未经验证**，来自一台没有独立复现的机器。参见[未经验证的报告](#unverified-report-peer-dma-works-peer-synchronisation-does-not)。

**第二个简短答案：即使 P2P 工作，链路仍然是瓶颈。** 在 PCIe Gen1 x4（约 1.0 GB/s）下，项目大体一致同意的立场是 P2P 受带宽限制，在 Gen3 之前收益不大。项目在软件上走到的最远是 Gen2 x4，已于 2026-07-29 随 `master` 发布；Gen2 x16 已在两台机器上复现，且只出现在携带 24 电容焊接改造的卡上。参见 [PCIe Gen2](../unlock/pcie-gen2.md) 和 [Gen3/Gen4](pcie-gen3-gen4.md)。

---

## 实测基线："没有 P2P"长什么样

| 观察 | 结果 | 条件 |
|---|---|---|
| `torch.cuda.can_device_access_peer(i,j)` | **全部 56** 对都返回 `False`（P2P 能力对：0/56） | 8 张解锁卡，所有对，包括 `PIX` 组内 |
| ggml `-lv 5` 日志 | 零次 `peer` / `p2p` / `rpc` 出现 | 同一台机器 |
| `nvidia-smi nvlink` | `Device does not have or support Nvlink.` | 同一台机器，8 张解锁 64 GiB 卡，2026-07-24；语料中唯一的抓取 |
| MIG profile 列表 | `1g.64gb`，ID 0，63.00 GiB，70 SMs，5 CEs，**P2P No** | 解锁卡上 `nvidia-smi mig -lgip` 提供的唯一 profile |
| 扫描期间的活动链路 | Gen1 x4，约 1.0 GB/s，推理负载下不升速 | 设备最大报告为 Gen2 x16 |

缺失也可以从源码树中看到，而不仅仅在遥测中。在正式发布 `master` 和全部十二个未发布分支中 grep `p2p` 和 `peer` 恰好返回两类命中：`build.sh` 模块安装列表中的原版 `nvidia-peermem.ko` 文件名，以及 Gen2 分支 `0008-pcie-gen2-probe-retrain.patch` 内一行未修改的上下文（`nv_uvm_resume_P2P(pUuid)`）。**任何分支都不包含任何 P2P 使能。**

> [!NOTE]
> **`nvidia-peermem` 不是同一回事**
>
> `build.sh` 收集并安装五个模块：`nvidia.ko`、`nvidia-modeset.ko`、`nvidia-uvm.ko`、`nvidia-drm.ko` 和 `nvidia-peermem.ko`。`nvidia-peermem` 是让第三方 RDMA 硬件访问 GPU 显存的原版对等显存客户端。看到它被构建和加载（包括无害的 `Skipping BTF generation for ... nvidia-peermem.ko` 行）**不是** GPU 到 GPU 对等访问可用的证据。

### 为什么它重要：实测代价

在 8 卡、80 层模型（每 GPU 10 层）上使用 `-sm layer` 切分时，每个生成的 token 都要经历 **7 次 GPU 到 CPU RAM 到 GPU 的跳转**，其中一次在第 49 到 50 层转换处跨越 NUMA/套接字边界。后果以并发上限的形式显现：

| 并发用户 | 1 | 2 | 4 | 8 | 16 |
|---|---|---|---|---|---|
| 聚合 tok/s | 17.3 | 21.6 | 25.7 | 28.1 | 38.9 |
| 每用户 tok/s | 17.3 | 10.8 | 6.4 | 3.5 | 2.4 |
| 批量墙钟时间（s） | n/a | 11.9 | 20.0 | 36.5 | 52.6 |
| 相对 1 用户的扩展 | 1.00x | 1.25x | 1.49x | 1.62x | 2.25x |

这是**从 1 到 16 用户的 2.25x 聚合扩展**，来源报告把它归因于三个原因共同作用：没有 P2P/NVLink、跨 NUMA 管线跳转和链路。它是一台虚拟化主机上的一次扫描，链路读活动为 Gen1 x4，设备最大为 Gen2 x16。张量并行——最能从对等访问中受益的策略——在这些链路上是实测死路：

| 配置 | Prefill 1k / 4k / 16k（t/s） | Decode（t/s） |
|---|---|---|
| 1 卡 | 839 / 1,092 / 960 | 27.3 |
| PP2（管线） | 829 / 1,084 / 1,167 | 29.1 |
| TP2（张量） | 316 / 420 / 416 | 33.7 |

Qwen2.5-72B 密集 AWQ，vLLM，Gen1 x4。TP 在 prefill 上差 2.3-2.8 倍，换来 +23 % decode。更多见 [LLM 推理](../operations/llm-inference.md)。

---

## aikitoria fork

`github.com/aikitoria/open-gpu-kernel-modules` 是 `tinygrad/open-gpu-kernel-modules` 的 fork，创建于 2024-10-14。其默认分支是 **`610.43.03-p2p`**，与 cmpunlocker 针对的驱动版本相同，fork 中的分支名从 515 一直到 610.43.03。正是这种版本对齐让叠层变得可行：master 的 `build.sh` 对 `driver/VERSION`（`610.43.03`、`610.43.02`）中未列出的任何驱动版本都硬性失败。

该分支在一个纯 NVIDIA 发布导入之上有三个提交：

| 提交 | 主题 | 范围 |
|---|---|---|
| `452cec62d827` | `610.43.03`（基础导入，2026-07-07） | `README.md`、`kernel-open/Kbuild`、`dp_connectorimpl.cpp`、`nvBldVer.h`、`nvUnixVersion.h`、`version.mk` |
| `9fb650447c7b` | 组合 P2P 修改 | 8 个文件，**+83 / -28** |
| `52670f7fd6a7` | 实验性 hugepage `cudaHostRegister` | 7 个文件，**+383 / -97** |
| `2849449f8cd6` | README 更新 | **+245** |

P2P 提交本身触碰：

| 文件 | 差异 |
|---|---|
| `install.sh` | +7 |
| `kernel-open/nvidia-uvm/uvm_gpu.h` | +7 |
| `kernel-open/nvidia/nv-reg.h` | +1 / -1 |
| `src/nvidia/generated/g_kern_bus_nvoc.c` | +5 / -5 |
| `src/nvidia/src/kernel/gpu/bif/kernel_bif.c` | +3 / -3 |
| `src/nvidia/src/kernel/gpu/bus/arch/pascal/kern_bus_gp100.c` | +10 |
| `src/nvidia/src/kernel/mem_mgr/io_vaspace.c` | +11 / -10 |
| `src/nvidia/src/kernel/rmapi/nv_gpu_ops.c` | +39 / -9 |

**机制：** 它在没有 NVLink 的 GPU 上启用 **BAR1 对等直连**，有 NVLink 时回退到 NVLink。对于 PCIe 对，传输通过 DMA 直接写入另一块 GPU 的物理地址，而不是经过主机 RAM 反弹。

> [!WARNING]
> **实验性：GA100 不是受支持的配置**
>
> 分支 README 列出 RTX 3090（有 NVLink 时成对 NVLink，否则 PCIe BAR1）、RTX 4090（PCIe BAR1）和 RTX 5090（PCIe BAR1），并声明 P2P 在同一代的不同设备之间也工作。**GA100 不在该列表中，补丁从未在 170HX 上验证过。** 它修改的代码路径是 `kern_bus_gp100.c`（Pascal 及以后的总线代码）、`io_vaspace.c` 和 `nv_gpu_ops.c`，所以其中可能根本不存在可用的 GA100 分支。

---

## 三提交 diff 工作流

这是 2026-07-23 与一张可用构建截图一起发布的精确配方：

```bash
git clone https://github.com/aikitoria/open-gpu-kernel-modules open-gpu-kernel-modules-p2p
git -C open-gpu-kernel-modules-p2p diff --src-prefix=a/ HEAD~3 > ./cmpunlocker/driver/patches/0007-unlock-p2p.patch
cd ./cmpunlocker && sudo install.sh
```

### 为什么它能组合

`driver/build.sh` 删除并重新解包一棵干净的原版树，然后用 `patch -p1` 按 glob（字典序）顺序应用**每个**匹配 `driver/patches/*.patch` 的文件：

```bash
rm -rf "${SRC_DIR}"
# ... 重新解包 open-gpu-kernel-modules-${VERSION}.tar.gz ...
for p in "${patches[@]}"; do
    patch -p1 < "${p}"
done
```

脚本在 `set -euo pipefail` 下运行，所以失败的 hunk 会中止构建而不是产出半补丁模块。把第三方 diff 命名为 `0007-unlock-p2p.patch` 使它排在正式发布系列 `0001`-`0006` 之后，所以解锁先落地，P2P 改动应用在其上。`--src-prefix=a/` 保证 `patch -p1` 期望的 `a/` 和 `b/` 路径前缀。不带第二个修订的 `HEAD~3` 把工作树与三个提交之前的版本做 diff，产出一个包含全部三个提交的压缩补丁，而不是三个独立补丁。

机制是代码确认的。该具体 diff 与 610.43.0x 的兼容性由一位测试者报告，没有被独立复现，所以把配方当作中等置信度。

> [!CAUTION]
> **你同时也在安装实验性 hugepage 提交**
>
> `HEAD~3..HEAD` 包含 `52670f7fd6a7`，它为 1G-hugepage 背书的缓冲区加速 `cudaHostRegister` 并缩小这些映射的设备页表。其自己的作者记录它自动启用，且"this path skips some of the per-4K-page bookkeeping the stock driver performs, so it may misbehave in edge cases the stock driver handles correctly"（该路径跳过了原版驱动执行的部分每 4K 页簿记，所以在原版驱动正确处理的一些边缘情况下它可能出错）。它没有任何形式的 GA100 验证。要只取 P2P 改动，cherry-pick 或 format-patch **单独的 `9fb650447c7b`**，而不是整个范围。

### 配方实践说明

- 按转录，最后一行读作 `sudo install.sh`。master 的安装器以 `sudo ./install.sh` 调用；不带路径的 `install.sh` 只在 `.` 在 `PATH` 上时有效。
- fork 自己的 `install.sh` 改动（+7 行）被扫进 diff 但没有效果：它补丁的文件是一个原版 NVIDIA 安装脚本，cmpunlocker 的 `build.sh` 从不运行它。
- **文件名冲突隐患。** Gen2 现在在 `master` 中，已经使用 `0007-pcie-gen2.patch` 和 `0008-pcie-gen2-probe-retrain.patch`，所以 P2P diff 对任何当前 checkout 都必须编号 `0009` 或更后。这在 2026-07-29 之前是分支合并隐患；现在只是每个叠层补丁都必须遵守的编号规则。
- `build.sh` 用 `curl -L --fail` 获取上游 tarball，**不做任何校验和或签名验证**。在未经验证的 diff 之上再叠一层会加剧这一点。
- 安装后，`build.sh` 把 `/sys/module/nvidia/srcversion` 与已补丁 `nvidia.ko` 上的 `modinfo -F srcversion` 比较。不匹配意味着原版模块赢得了加载竞争，解锁和 P2P 补丁都没有生效。参见[验证](../procedures/verify.md)。

---

## 实测结果

几乎一个都没有，而这个缺口是本页唯一最重要的事。

| 数值 | 值 | 条件 | 置信度 |
|---|---|---|---|
| 任何 170HX 上的 `p2pBandwidthLatencyTest` | **未运行** | 没有人发布矩阵，带或不带补丁都没有 | n/a |
| 未补丁时报告有能力的 P2P 对 | 0/56 | 8 张解锁卡，PyTorch | 高 |
| P2P 补丁在 cmpunlocker 上构建并加载 | 是 | 一位测试者，2026-07-23，截图；机器还带 2x RTX 3090 | 中 |
| 对纯 170HX 对的效果 | 报告**无** | 一位测试者，未发布测试输出 | 低 |
| 参考：P2P 禁用带宽 | 42.69-43.91 GB/s | 9-GPU Blackwell 系统，Gen5 x16，**不是 170HX** | 高（对该系统） |
| 参考：P2P 启用带宽 | 55.59-56.58 GB/s | 同一系统 | 高（对该系统） |
| 参考：设备到自身 | 1611.24-1665.83 GB/s | 同一系统 | 高（对该系统） |

Blackwell 参考数字**不可迁移**。170HX 在 Gen1 x4 下（甚至 Gen2 x4 下）移动大约这些数字的三十分之一到六十分之一，而且该系统的驱动分支只把 3090/4090/5090 列为受支持。

> [!NOTE]
> **开放问题：补丁在纯 170HX 主机上做任何事吗？**
>
> 同一天有两份报告。一份记录在带截图的机器上得到"p2p + cmpunlock working"，该机器还带两张 RTX 3090。另一份记录成功构建后"it doesn't seem to take effect on the 170HX ... it only has an effect on them if there are other models of GPUs on the same machine"（它在 170HX 上似乎没有生效……只有当同一台机器上有其他型号的 GPU 时才对他们有效）。这两份可能实际上并不冲突：成功的机器正是负面报告所说的唯一能工作的混合型号场景。没有人以任何方式发布过 `simpleP2P` 或 `p2pBandwidthLatencyTest` 输出。**什么能定案：** 纯 170HX 双卡主机的连通性矩阵，带和不带叠层补丁。测试便宜，结果无歧义。

---

## 未经验证的报告：对等 DMA 工作，对等同步不工作

> [!CAUTION]
> **未经验证的社区声明**
>
> 本节的一切来自一台四卡机器上的单一构建者，发布时带日志，但从未被独立复现。它与上面的"效果未证实"立场矛盾。把它当作值得核查的线索，而不是结果。

该声明是：叠层的 `aikitoria` P2P 补丁确实在 GA100 上生效，但只生效一半：对等*数据移动*工作，对等*同步*不工作。

| 测试 | 报告结果 |
|---|---|
| `torch.cuda.can_device_access_peer(i,j)` | 4 卡主机上全部 12 个有序对返回 `True` |
| 跨卡 `cudaMemcpyPeer` | **6.25 GB/s**，对照同样拷贝经主机显存暂存的 5.70 GB/s |
| 跨进程 CUDA IPC 句柄共享 | 工作 |
| 任何 NCCL 集合 | 在传输连接处**挂起**：无错误、无超时、两块 GPU 都钉在 100 % |
| vLLM 自定义 all-reduce | **同样方式挂起** |

给出的解释是两半有不同的要求。对等拷贝是一个 DMA 引擎走一张映射。集合还需要一块 GPU 把标志写进另一块 GPU 的显存，并让第二块 GPU 上的内核自旋直到观察到该写入。据报道正是第二种模式不工作，这可以解释为什么裸拷贝成功而每个集合通信库都挂起而不是失败。

为它给出的机制也未经验证：

- `kbusIsPcieBar1P2PMappingSupported_GH100` 要求两块 GPU 上都有**静态 BAR1**，而静态 BAR1 要求 BAR1 在 512 MB 对齐偏移上跨越整个帧缓冲。在 170HX 上 BAR1 是 **64 MB**，所以该检查无法通过。参见 [BAR 大小](#bar-sizing-and-resizable-bar-limits)，它与本页其他东西阻塞的是同一个 64 MB 约束。
- 邮箱回退随后在 `kern_bus.c` 中失败于自己的对齐断言 `(base & RM_PAGE_MASK) == 0`，接着是 `kern_bus_gm200.c` 中的 `remoteWMBoxLocalAddr != ~0ULL`。
- 另外，报告者声称 cmpunlocker 自己的 `P2P` 分支把 `p2pOverride` 和 `pcieP2PType` 门控在 `_kbifInitRegistryOverrides` 内从 `pGpu->idInfo.PCIDeviceID` 读取的 `devId == 0x20C2` 之后，但该字段直到 `gpu.c` 更晚才被填充，所以门控从不打开。上游 `aikitoria` 提交 `9fb650447c7b` 无条件设置两者。

如果这站得住，实际后果狭窄但真实：手写的多 GPU 代码在卡间移动缓冲区并把协调留给**主机**的，可以使用对等 DMA，而每个集合通信库——因此每个主流推理服务器中的张量并行——都不行。报告者在多卡 vLLM 上的可用配置是 `NCCL_P2P_DISABLE=1` 加 `--disable-custom-all-reduce`，这恰恰是完全没有 P2P 补丁也能工作的配置。在那台机器上，补丁因此对推理没有买到任何东西。

**什么能定案。** 第二台机器跑三个测试：`can_device_access_peer`、定时 `cudaMemcpyPeer` 和任何 NCCL 集合。第三个是决定性的，而且对任何已有两张卡并构建好补丁的人来说是两分钟测试。

---

## IOMMU 交互

BAR1 对等 DMA 在另一设备处写入裸物理地址。只有当 IOMMU 不转换这些地址时才有效。

P2P 分支的文档化设置是：

```bash
# /etc/default/grub, GRUB_CMDLINE_LINUX_DEFAULT
amd_iommu=on iommu=pt        # AMD
intel_iommu=on iommu=pt      # Intel
sudo update-grub
# 安装 610.43.03 驱动，运行 ./install.sh，重启
```

README 直白地陈述要求：IOMMU 必须处于**直通**模式而非转换模式，否则 DMA 会走 IOMMU 页表且传输失败。

> [!CAUTION]
> **直通模式削弱 DMA 隔离**
>
> 同一份 README 警告该配置"如果你运行不受信任的软件或设备，是非常危险的"。`iommu=pt` 意味着设备以主机物理地址做 DMA，IOMMU 不监管它们。不要把它应用到多租户主机上。

**ACS 是问题的另一半。** 如果 P2P 被启用但很慢，根端口上的访问控制服务（Access Control Services）会迫使所有 GPU 到 GPU 的流量上行穿过 CPU 根复合体，这摧毁了补丁存在的意义。给出的补救措施按优先顺序：在 BIOS 中禁用 ACS；用 `pcie_acs_override=downstream,multifunction` 启动；或应用 ACS 覆盖内核补丁。注意 ACS 覆盖也正是破坏 IOMMU 组隔离的东西，所以这会加剧上面的警告。

对于 A/B 测试，3090 对可以被强制走 PCIe BAR1 路径而不是 NVLink：

```conf
# /etc/modprobe.d/nvidia.conf
options nvidia NVreg_RegistryDwords="RMForceP2PType=1"
```

### cmpunlocker 自己对 IOMMU 做什么

| 树 | IOMMU 处理 |
|---|---|
| `master`（正式发布） | **无**。`install.sh` 和 `remove.sh` 完全没有 `iommu` 或内核命令行处理 |
| Gen2 代码（现在在 `master` 中） | 把 `intel_iommu=on iommu=pt`（GenuineIntel）或 `amd_iommu=on iommu=pt`（AuthenticAMD）追加到 `/etc/default/grub` 或 `/etc/kernel/cmdline`，带 `--no-iommu` 退出选项 |

Gen2 安装器还在运行时用 `grep -qw iommu=pt /proc/cmdline && [[ -d /sys/class/iommu ]] && [[ -n "$(ls -A /sys/class/iommu)" ]]` 验证，打印 `IOMMU is already active in passthrough mode on the running kernel`（IOMMU 已在运行内核上以直通模式激活）或 `IOMMU passthrough takes effect after the next reboot`（IOMMU 直通将在下次重启后生效），外加提醒 BIOS 中还必须打开 VT-d / AMD-Vi / SVM。该分支的 `remove.sh` 从 `*.cmpunlocker.bak` 恢复，并打印 `Reverted IOMMU kernel parameters (effective after reboot)`（已恢复 IOMMU 内核参数（重启后生效）），或报告找不到 IOMMU 配置备份且内核命令行保持原样。那是提交 `6a85e6c` "IOMMU enablement as part of install script"，分支代码，不是正式发布。

实际后果：**Gen2 代码已经配置了 P2P 补丁要求的恰好内容**，这使 Gen2 加 P2P 成为任何人今天能组装的最接近预配置栈的东西。没有人组装过它。

一台测试机上的已验证直通启动，供比较：

```text
Linux 7.1.3-arch2-2, cmdline: intel_iommu=on iommu=pt nowatchdog nvme_load=YES
DMAR: IOMMU enabled
(four DRHD units)
iommu: Default domain type: Passthrough (set via kernel command line)
GPU at 0000:65:00.0, alone in IOMMU group 3
```

一份单独的 `lspci -vvv` 抓取显示 IOMMU 组 31 中 `0000:81:00.0` 处的一张卡。一张卡独处自己的组正是直通设置想要的，但它对根端口之间的 ACS 行为什么都没说，而后者才是支配 P2P 吞吐的部分。

无驱动[重触发链](../history/tool-lineage.md)出于不同原因有同类要求：它需要 `intel_iommu=off` **或** `iommu=pt`，这样当它把 hugepage 地址交给 Booter 时，DMA 物理地址等于主机物理地址。

---

## BAR 大小与可调整大小 BAR 限制

170HX 暴露三个 BAR 和一个实际上无法调整任何东西的可调整大小 BAR 能力。

| BAR | 大小 | 类型 | 观测区域基址 | ReBAR 支持大小 |
|---|---|---|---|---|
| BAR0 | 16 MB（`0x1000000`） | 32 位，不可预取 | `f0000000`（另一台主机上 `0xfa000000`） | 仅 16MB |
| BAR1 | **64 MB** | 64 位，可预取 | `20048000000` | 仅 64MB |
| BAR3 | 32 MB | 64 位，可预取 | `2004c000000` | 仅 32MB |

`lspci -vvv` 报告 `Capabilities: [bb0 v1] Physical Resizable BAR`，每个 BAR 恰好一个受支持大小，`nvidia-smi` 同意：`BAR1 Memory Usage Total: 64 MiB`。在解锁卡上创建的 MIG 实例报告 `0MiB / 64MiB` 共享 BAR1，同时带 `1MiB / 65053MiB` 显存。

**即使卡通告 81920 MiB 帧缓冲，BAR1 仍保持 64 MiB。** 因此这张卡上没有大 BAR 或全 VRAM 主机映射，这正是 [PRAMIN 窗口](../unlock/memory-geometry.md)对显存解锁重要的原因。

由于 aikitoria 补丁通过 **BAR1** 映射对等显存，这个 64 MiB 不可调整大小的窗口是悬在 GA100 整个方法之上的结构性问题。语料中没有任何来源确定驱动的 BAR1 P2P 路径能否在 64 MiB 窗口内运作，或它是否像消费级 4090/5090 设置那样假设大 BAR。没有人测试过。

### 正式发布的 BAR0/PRAMIN 钳制

`0004-bar0-pramin-clamp.patch` 是 20 行，适用于**两个**设备 ID。当 `devId == 0x20C2 || devId == 0x2082` 且 `Ram.fbAddrSpaceSizeMb > 0x2000`（8192 MB）时：

```c
offsetBar0 = (0x2000ULL << 20) - DRF_SIZE(NV_PRAMIN);
```

10 GB 卡在 10240 MB 时已经超过 `0x2000`，所以钳制在那里也生效，解锁到 40 GB 的 10 GB 卡得到基于 8192 MiB 的 PRAMIN 窗口，而不是基于 10240 MiB 的。这是一个刻意的两行规模常量，任何实验 BAR 行为的人都可以修改并重新构建。

### 可调整大小 BAR：什么已定案，什么没有

> [!NOTE]
> **开放问题：大 BAR / ReBAR / Above 4G Decoding**
>
> 该问题于 2026-07-22 14:11 发布，从未被回答。它重要，因为正式发布解锁刻意钳制 BAR0/PRAMIN 窗口，且卡通告的 ReBAR 能力似乎不提供替代大小。**下一步：** 启用 Above 4G Decoding 启动，并在 Gen2 训练成功的卡上读回 ReBAR 能力大小。如果能力结构真的为每个 BAR 列出单一支持大小，那么包括 `github.com/xCuri0/ReBarUEFI`（用于 UEFI 缺乏 ReBAR 支持的主机）在内的任何主机侧变通都无济于事。

### 多卡的 BAR 压力

一份二手报告描述了单台服务器中超过八块高显存 GPU 时出现 BAR 地址空间问题，没有捕获错误字符串或平台。重要的限定：解锁**不**扩大任何 BAR，所以 BAR 压力来自每设备可调整大小 BAR 窗口，而不是来自 64 GB 帧缓冲。128 lane 单插槽平台的 lane 算术给出大约七张卡 x16（无 NVMe 时八张）或同等聚合带宽下多得多的 x4 卡。运营者正在生产环境中跑 8 卡和 10 卡服务器。

---

## 多卡安装状态

P2P 天生是多卡话题，而 cmpunlocker 的多卡支持只在分支中。

| 能力 | `master` | `multiple-cards` / `Gen2` |
|---|---|---|
| 卡枚举 | `lspci -nn \| grep -iE '10de:20b0\|10de:20c2\|10de:2082' \| head -1`（只取第一个匹配） | `mapfile -t PCI_LINES`，每个匹配，五个并行数组（BDF、devid、profile、expected_mib、current_mib） |
| Profile | 按 `nvidia-smi memory.total` 阈值的 `8gb` / `10gb` | `profile_from_devid()`：`20c2 → 8gb`、`2082 → 10gb`，外加第三个 `mixed` profile |
| 清单文件 | 无 | `/lib/modules/$(uname -r)/updates/cmpunlocker/gpu_inventory`，每 GPU 一行，例如 `0000:0b:00.0 20c2 8gb 65536` |
| `verify.sh` | 不存在 | 每 GPU `OK` / `STOCK` / `MISSING` / `UNEXPECTED`，阈值 `>= 60000 MiB`（8gb）和 `35000-59999 MiB`（10gb） |

安装器限制对解锁本身是表面的：补丁 `0001` 在**每次** GSP 启动时读取 `pGpu->idInfo.PCIDeviceID` 并按设备选择几何，所以多卡主机被完全解锁，即使 master 的安装器只检查一张卡。一个构建就服务于同时带 8 GB 和 10 GB 卡的主机。

> [!CAUTION]
> **混合 GPU 主机误检测 profile**
>
> `detect_card_profile()` 读取 `nvidia-smi --query-gpu=memory.total ... | head -1`，这是 **nvidia-smi 顺序中的第一块 GPU**，而不是 `lspci` 找到的 CMP。一台带 RTX 3080 10 GB 和 8 GB 170HX 的主机从 3080 检测到 "10GB" 并选择错误的 profile。至少两位测试者复现；其他 CMP SKU 也被误检测为 10 GB 170HX 卡。**在混合主机上始终显式传 `--profile=8gb` 或 `--profile=10gb`。** 这尤其咬住 P2P 工作，因为混合型号主机正是 P2P 补丁被报告有任何效果的那一种配置。

这里值得携带两条进一步的多卡约束：

- **Proxmox 直通需要 SeaBIOS，而不是 UEFI/OVMF。** UEFI 产生看起来完全像 exploit 没生效的 RM 初始化/适配器失败。两个人独立地把非复现追溯到这一点。
- **`verify.sh` 从不检查 PCIe 代际**，即使在 Gen2 分支谱系上。在 `Gen2/verify.sh`、`far/verify.sh` 和 `deced/verify.sh` 中 grep "pcie" 返回零命中。链路状态必须用 `nvidia-smi` 或 `pcielink.sh` 手动检查。

完整安装路径见[多卡流程](../procedures/multi-gpu.md)。

---

## P2P 相对替代方案的定位

| 路径 | 状态 | 阻塞器 |
|---|---|---|
| NVLink | 熔丝禁用（`FUSE_NVLINK_DIS` `0x00820684` = `0x00000007`），从未起来 | OTP 熔丝加上被拆空的板卡元件；见 [NVLink](nvlink.md) |
| PCIe P2P，正式发布解锁 | 缺失 | 树中任何地方都没有代码 |
| PCIe P2P，叠层补丁 | 构建并加载。一份**未经验证**的报告称对等 DMA 6.25 GB/s 而对等同步仍坏 | 不受支持的配置；集合据报挂起 |
| 更快链路（Gen2 x4） | 自 2026-07-29 起随 `master` 发布 | 低于所述的张量并行阈值 |
| 更快链路（Gen2 x16） | 已在两台机器上复现，5.97 到 6.67 GB/s | 需要 24 电容焊接改造；90 分钟以上老化未测 |
| Gen3 / Gen4 | 未实现 | 评估为需要无人产出的 GSP 补丁 |

张量并行变得值得尝试的所述阈值是 **PCIe Gen2 x16 或 Gen3 x4**。解锁器交付 Gen2 **x4**，低于它。恢复 x16 是[物理改造](../operations/physical-mods.md)（24 x 0402 220 nF X7R 电容），不是软件改动，而且只改变 lane 数，从不改变链路代际。两种机制相互独立，不得混为一谈。

在那之前，工作指导不变：管线并行，而不是张量并行，以及用 MoE 模型减少每 token 的跨设备激活流量。

---

## 开放问题

> [!NOTE]
> **开放问题：P2P 问题集**
>
> 1. **叠层补丁是否在两张 170HX 卡之间启用 P2P？** 在纯 170HX 对上运行 `simpleP2P` 和 `p2pBandwidthLatencyTest`，带和不带补丁，并发布矩阵。该领域中没有什么比这更便宜或更决定性的了。
> 2. **补丁中是否根本存在 GA100 代码路径？** 被修改的文件是 Pascal 时代的总线代码加上 VA 空间和 RM API 层。对照 GA100 HAL 阅读 `kern_bus_gp100.c` 可以在没有硬件的情况下回答这个问题。
> 3. **BAR1 P2P 能否通过 64 MiB 不可调整大小的窗口工作？** 未确立。这可能是负面报告存在的原因。
> 4. **P2P 在 Gen1 x4 或 Gen2 x4 下值得吗？** 记录中的主要立场是"I would only implement P2P when we get at least PCIe Gen 3, otherwise it seems kind of a waste on these cards"（我只有在至少拿到 PCIe Gen 3 时才会实现 P2P，否则在这些卡上似乎是种浪费）。该前提目前未满足，也没有任何证据表明它可达。
> 5. **硅片中是否存在 P2P 能力位？** 记录中有一个建议：检查由 `0x00823804` 处 FEAT PLM 管辖的寄存器空间是否携带 P2P 能力位，因为解锁已经能到达该块。没人看过。
> 6. **tinygrad 谱系的设备表是否匹配 170HX？** 上游 P2P 驱动枚举从 A100 和 CMP 40HX 到 CMP 90HX，却漏掉 170HX，为 610.x 更新的 fork 也仍然漏掉它。未知解锁卡是否会被接受，或 `Graphics Device` 识别字符串是否会破坏设备匹配。把两个设备 ID 加进表并测试是琐碎改动。
> 7. **多卡、IOMMU 和 Gen2 是否应该合入 master，按什么顺序？** `multiple-cards` 安装器改动（`b1cb6d8`）是自包含的，可以单独落地；Gen2 分支把它们与未验证的 PCIe 寄存器写入捆绑在一起。

---

## 相关页面

- [PCIe 子系统](../hardware/pcie-subsystem.md)，了解链路、BAR 和配置空间细节
- [PCIe Gen2 解锁](../unlock/pcie-gen2.md) 和 [Gen3/Gen4](pcie-gen3-gen4.md)
- [NVLink](nvlink.md) 和 [NVLink 硬件](../hardware/nvlink-hardware.md)
- [驱动补丁](../unlock/driver-patches.md)，了解 `0001`-`0006` 系列和构建系统
- [多卡安装](../procedures/multi-gpu.md) 和[验证](../procedures/verify.md)
- [物理改造](../operations/physical-mods.md)，了解 x16 电容改造
- [LLM 推理](../operations/llm-inference.md)，了解并行度测量
- [状态板](status-board.md) 和[开放问题](open-questions.md)
- [词汇表](../start/glossary.md)
