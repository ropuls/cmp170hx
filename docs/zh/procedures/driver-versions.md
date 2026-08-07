# 驱动版本：支持哪些，为什么

## 本页涵盖

CMP 170HX 解锁构建针对哪些 NVIDIA 驱动版本、为什么列表这么短、如果你让安装器指向其他版本会怎样，以及未发布的移植分支对 595、590 和 580 到底提供了什么。

简短回答：**正式发布的 `master` 恰好支持两个版本，`610.43.03`（默认）和 `610.43.02`，按精确字符串匹配。其他任何版本构建都会硬失败。** 两者都在真实硬件上做过启动测试。610 以下的一切只存在于未发布的分支上，只经过源码验证，从未有人在本记录的范围内让 170HX 启动过。

注意那个容易绊倒人的区别：170HX 在普通原版 NVIDIA 驱动上运行得非常好。只是不会在它们上面**解锁**。能驱动和能解锁是两个独立的问题。

---

## `master` 上的受支持列表

`driver/VERSION` 包含两行，按此顺序：

```text
610.43.03
610.43.02
```

第一行是默认构建目标。`common/constants.yaml` 在 `driver_versions` 下镜像同样的两个版本。`install.sh` 和 `driver/build.sh` 都把 `driver/VERSION` 读入 `SUPPORTED_VERSIONS`，并调用精确字符串匹配的 `version_supported()`。没有范围检查、没有"610 或更新"的比较、没有模糊匹配。

如果你已安装的驱动不是这两个之一，安装会以下列信息终止：

```text
Installed driver is ${detected}, but cmpunlocker requires one of: 610.43.03,610.43.02.
```

### 已安装版本如何被检测

`install.sh` 依次尝试四个来源，在第一个得到版本的来源处停止：

| 顺序 | 来源 |
|---|---|
| 1 | `/proc/driver/nvidia/version` |
| 2 | `nvidia-smi --query-gpu=driver_version` |
| 3 | 探测 `/lib/firmware/nvidia/<supported>/` 目录 |
| 4 | `/lib/firmware/nvidia/` 下排序最高的目录 |

然后构建会下载匹配的上游源码压缩包：

```text
https://github.com/NVIDIA/open-gpu-kernel-modules/archive/refs/tags/${VERSION}.tar.gz
```

它缓存在 `driver/.build/` 下，每次运行都会干净地重新解压。cmpunlocker 仓库本身不携带任何 NVIDIA 代码。

> [!NOTE]
> **下载没有校验和**
>
> `build.sh` 用 `curl -L --fail` 获取压缩包，不验证任何东西。树中任何地方都没有记录的 SHA-256。在 `driver/VERSION` 或 `common/constants.yaml` 中为每个版本记录预期哈希是一个显而易见、但尚未实现的改进。

---

## 为什么是 610.43.0x

四个原因，按"硬约束"程度递减排列。

**补丁 hunk 锚定在那棵源码树上。** 六个补丁文件在 `set -euo pipefail` 下用 `patch -p1` 应用。任何一个 hunk 被拒绝都会中止构建。`kernel_gsp.c`、`g_kernel_gsp_nvoc.h`、`osinit.c`、`kernel_gsp_tu102.c` 和 `nv.c` 中的行号、周围上下文和结构布局在上游版本之间都会变动。参见 [the six driver patches](../unlock/driver-patches.md)。

**解锁必须是对开源内核模块的补丁。** 专有 NVIDIA 驱动"启动路径不同，不能以同样方式打补丁"。开源模块在 GA100 上也是仅 GSP 的：用 `NVreg_EnableGpuFirmware=0` 加载会直接以 `0x62` 固件初始化错误失败，所以这颗硅片没有 CPU-RM 逃生通道。

**610 是明确声明的底线。** 维护者被问到与第三方 P2P 驱动共存时的原话是"它需要是 **610 或以上**"。实际上 `master` 比这句话更严格：它拒绝任何不是两个白名单字符串之一的版本。

**两个版本都有现场验证。** 来自两台机器的独立运行时抓取：

```text
NVRM version: NVIDIA UNIX Open Kernel Module for x86_64 610.43.02 Release Build
              (dvs-builder@U22-I3-H05-01-2) Tue May 19 11:24:27 UTC 2026
GCC version:  gcc version 13.3.0 (Ubuntu 13.3.0-6ubuntu2~24.04.1)
kernel:       6.8.0-136-generic
```

> [!WARNING]
> **那次抓取来自解锁没有生效的机器**
>
> 把它当作 `610.43.02` 存在且能安装的证据，而不是它能解锁的证据。`dvs-builder` 构建字符串是 NVIDIA 自己的，所以那台机器上加载的是**原版**模块，不是补丁版；在同一台机器上 `verify.sh` 报告每块 GPU 都是 `MISSING`，`dmesg` 中也没有 `SEC2_DEBUG` 行。不要把这个代码块里的 gcc 或内核版本当作已知良好的构建环境。

另外还有一份 `NVIDIA-SMI 610.43.03 / KMD Version: 610.43.03 / CUDA UMD Version: 13.3`。流通中的一个打包好的 NixOS 模块硬断言 `config.hardware.nvidia.package.version == "610.43.03"`。

每个实验分支，包括整个 PCIe Gen2 血统和 80 GB 尝试，也只列出 `610.43.03` 和 `610.43.02`。移植分支是唯一例外。

---

## `610.43.02` 还是 `610.43.03`？

> [!NOTE]
> **未解决问题**
>
> 没有人回答过这个问题。"610.43.02 和 610.43.03 哪个更可靠？"这个问题在 2026-07-24 被直接在频道里问过，从未得到回答。两个版本上都有成功的解锁。`610.43.03` 只是因为是 `driver/VERSION` 的第一行才成为默认。
>
> 这个实验很简单，但没人做过：收集现有装机上的 `driver_version` 元数据文件和 `SEC2_DEBUG` PLM 打开成功率，然后比较。

实用建议：**用 `610.43.03`，默认版。** 如果一张卡在两个版本之一上不能干净地解锁，尝试另一个是便宜且合理的诊断步骤，但没有任何证据表明哪边会有效。

---

## 固定版本（Pinning）

> [!WARNING]
> **有理由的建议，不是实测结果**
>
> 针对未来 NVIDIA 驱动封堵漏洞的推荐长期缓解措施是**把驱动固定在 610**，就像 P100 和 V100 的运营者把驱动固定在 580 附近一样。至少有一位运营者已经把包固定下来作为预防措施。这是未被质疑的推理，而非已证明的需求：不存在会封堵的驱动，而且 GitHub 上已发布的开源内核模块无法召回。

---

## 原版驱动：能驱动，不解锁

170HX 在完全未打补丁的驱动下也能被枚举并运行 CUDA。Ubuntu 24.04 上带 CUDA 12.8 的 `nvidia-driver-570` 开箱即用，Ubuntu 22.04 上的 `nvidia-driver-535-server` 也被报告可用。`nvidia-smi` 把这张卡称为计算能力 8.0 的 `NVIDIA Graphics Device`，因为驱动的 PCI ID 表没有为 `0x20C2` 提供市场名称。这个命名怪癖是快速确认你在看一张 CMP 部件的方法。

在原版驱动下，卡是锁定的：出厂容量、出厂算力节流、PCIe 第 1 代 x4。

---

## 随版本一起的硬性要求

| 要求 | 详情 |
|---|---|
| Secure Boot | 必须**关闭**。补丁模块未签名；开启时 dmesg 显示 `nvidia: module verification failed: signature and/or required key missing - tainting kernel`。如果 `/sys/firmware/efi` 存在、`mokutil` 可用且 `mokutil --sb-state` 报告已启用，`install.sh` 会以 `Secure Boot is enabled. Disable it before installing unsigned patched modules.` 终止。在非 EFI 系统，或未安装 `mokutil` 的系统上，该检查会被静默跳过。 |
| 驱动家族 | 仅 nvidia-open。专有 blob 不能用同样方式打补丁。 |
| 操作系统 | **仅 Linux。** GSP 启动路径是 Linux 特有的；Windows WDDM 驱动根本不同。 |
| 内核头文件 | `/lib/modules/$(uname -r)/build` 必须存在。 |
| 工具链 | 需要 `python3`。`master` 上**不用 PyYAML**，发布脚本中**也没有任何显式 GCC 版本检查**。"需要 gcc 13+ 和 PyYAML" 来自第三方 `unlock-cmp-170hx` 指南仓库（外加六个分支上一个遗留的 `requirements.txt`），不是来自 cmpunlocker。泄露包的 README 只要求 root 和内核头文件。 |
| 网络 | 首次安装时需要，用于获取上游压缩包。 |

完整流程见 [Install](install.md)，还原见 [Uninstall](uninstall.md)。

---

## 移植分支：`clanker/driver-port`

> [!WARNING]
> **实验性：已源码验证，从未启动测试**
>
> 595、590 和 580 支持是一个未发布的分支。它自己的 README 原文如此：
>
> > `595.71.05, 590.48.01, and 580.105.08 are source-verified (patches apply cleanly and the
> > unlock logic matches the 610.43.0x path) but have not yet been boot-tested on physical CMP
> > 170HX hardware.`
>
> 该分支于 2026-07-21 宣布，并明确征集测试者。**截至 2026-07-28，记录中没有任何成功确认。** 在有一张卡真正启动之前，把成功构建当作什么都不是。

分支尖端 `153cd6d`，2026-07-21。

### 它改了什么

结构上几乎没改。`driver/patches/` 变成四个按主版本号的子目录，每个持有同样的六个补丁文件名，`build.sh` 增加两行修改：

```diff
-PATCH_DIR="${SCRIPT_DIR}/patches"
+BRANCH="${VERSION%%.*}"
+PATCH_DIR="${SCRIPT_DIR}/patches/${BRANCH}"
```

分支上的 `install.sh` 与 `master` **逐字节相同**：单 GPU、`head -1`、没有 `verify.sh`、没有 `gpu_inventory`。如果你还想要多 GPU 或 PCIe Gen2，这个分支给不了你。参见 [Multi-GPU](multi-gpu.md)。

### 这是重新锚定练习，不是重写

每个寄存器值、PLM 条目、负载偏移、static-info 重写和 PMA 函数在全部四个目录中逐字符相同。具体来说：

- 补丁 `0004` 和 `0005` 在全部四个版本目录中逐字节相同（md5 相同）。
- 补丁 `0002` 和 `0006` 在 590 和 610 之间逐字节相同。
- `0003` 新增的 `+` 行在全部四个中相同。
- `0001` 新增的 `+` 行在 610 与 580/590/595 之间只差**恰好一个额外的空行**，其他什么都没有。

### 每个目录的补丁大小

| 目录 | 0001 | 0002 | 0003 | 0004 | 0005 | 0006 | 合计 |
|---|---|---|---|---|---|---|---|
| `580` | 19,700 | 3,957 | 10,377 | 861 | 1,642 | 497 | **37,034** |
| `590` | 19,647 | 3,988 | 10,377 | 861 | 1,642 | 603 | **37,118** |
| `595` | 19,638 | 3,957 | 10,364 | 861 | 1,642 | 531 | **36,993** |
| `610` | 19,741 | 3,988 | 10,580 | 861 | 1,642 | 603 | **37,415** |

`610` 目录是 **`master` 补丁集的逐字节副本**。移植中的任何东西都不改变正式发布路径。

### 移植必须吸收的上游差异

其中只有一个是语义性的；其余都是上下文和锚点漂移。

| 差异 | 610 | 595 | 590 | 580 |
|---|---|---|---|---|
| `_kgspCreateSignatureMemdesc` 中的 Memdesc 标志 | 以 `if (confComputeForceUnprotAlloc(pGpu))` 为条件 | `MEMDESC_FLAGS_ALLOC_IN_UNPROTECTED_MEMORY` 无条件 | 同 595 | 同 595 |
| `osinit.c` 中的 late-PMA 钩子上下文 | 跟在 `goto shutdown;` 后面 | 跟在 `goto shutdown;` 后面 | 跟在 `consoleDisabled = NV_FALSE;` 后面 | 跟在 `consoleDisabled = NV_FALSE;` 后面 |
| GSP static-info 尾部上下文 | `NV_ASSERT_OK_OR_GOTO(status, kgspInitGspTraceCrashBuffer(...), done);` | 存在 | **不存在** | 存在 |
| Static-info hunk 锚点 | `@@ -5164` | `@@ -5070` | `@@ -4065` | `@@ -4198` |
| `KernelGsp` 字段插入锚点 | `@@ -544,6 +544,8 @@` | `@@ -541` | `@@ -525` | `@@ -524` |
| 插入点之后的字段 | `GspSystemInfo *pSystemInfo; NvU32 regTableSize; PACKED_REGISTRY_TABLE *pRegTable;` | 同 610 | `LIBOS_LOG_DECODE logDecode; LIBOS_LOG_DECODE logDecodeVgpuPartition[48]; RM_LIBOS_LOG_MEM rmLibosLogMem[7];` | 同 590 |
| 补丁 0006 尾部上下文 | `(void)rm_get_gpu_uuid_raw(sp, nv);` | 同 610 | 同 610 | `{ const NvU8 *uuid = rm_get_gpu_uuid_raw(sp, nv);` |
| 补丁 0006 锚点 | `@@ -1521` | `@@ -1531` | `@@ -1521` | `@@ -1481` |
| 补丁 0002 相邻符号 | `void kgspConfigureFalcon_TU102(` | `static NvBool _kgspIsProcessorSuspended(OBJGPU *pGpu, void *pVoid);` | 同 610 | 同 595 |
| 补丁 0002 锚点 | `@@ -57` / `@@ -545` / `@@ -565` | `@@ -55` / `@@ -500` / `@@ -520` | 同 610 | `@@ -54` / `@@ -516` / `@@ -536` |

非保护分配（unprotected-allocation）差异是唯一的行为差异，它让 610 之前的源码树略微更宽松，而不是更严格。

### 版本列表内部不一致

> [!CAUTION]
> **十二个白名单版本中有七个没有经过验证的补丁锚点**
>
> 该分支的 `driver/VERSION` 列出了**十二个**版本：
>
> ```text
> 610.43.03  610.43.02
> 595.71.05  595.58.03  595.45.04
> 590.48.01
> 580.105.08 580.95.05  580.82.09  580.82.07  580.76.05  580.65.06
> ```
>
> 但只存在**四个**补丁目录，而 `build.sh` 通过 `BRANCH="${VERSION%%.*}"` 选择，也就是**只按主版本号**。所以 `595.45.04` 是用 `595.71.05` 的 hunk 打补丁，`580.65.06` 是用 `580.105.08` 的 hunk 打补丁。十二个中有五个有某种证据：`610.43.03` 和 `610.43.02` 经过启动测试，`595.71.05`、`590.48.01` 和 `580.105.08` 是分支 README 所称源码验证的三个。其余七个（`595.58.03`、`595.45.04`、`580.95.05`、`580.82.09`、`580.82.07`、`580.76.05`、`580.65.06`）完全依赖 `patch -p1` 的模糊匹配。
>
> 与此同时，同一分支上的 `common/constants.yaml` 只列出**五个**版本（`610.43.03`、`610.43.02`、`595.71.05`、`590.48.01`、`580.105.08`），与 `VERSION` 不一致。`install.sh` 接受全部十二个，所以用户无需做任何不寻常的事就能进入未验证状态。
>
> 这里的失败风险是基于阅读代码的合理推断，不是观察到的补丁拒绝。这个测试纯粹离线、机械：下载七个额外压缩包中的每一个，对主版本补丁目录运行 `patch -p1 --dry-run`。不需要硬件。

---

## 你应该跑哪个？

| 情况 | 建议 |
|---|---|
| 常规安装、一张卡、想让它工作 | `master` 上的 **610.43.03**。这是唯一有广泛第一手确认的组合。 |
| 一张卡，610.43.03 行为异常 | 试试 **610.43.02**。两者都在白名单里，都产生过成功解锁。 |
| 多张 170HX 卡 | `master` 能用，并已在多 GPU 主机上确认过，包括 Proxmox 直通下的 8 卡。注意 `install.sh` 的自动检测隐患，显式传 `--profile`。参见 [Multi-GPU](multi-gpu.md)。 |
| 你需要 PCIe Gen2 | 仅分支，且仅 610。参见 [PCIe Gen2](../unlock/pcie-gen2.md)。 |
| 你被另一个应用固定在了 595、590 或 580 | 移植分支是你唯一的选择，而且你会是第一个让它启动的人。在一台你坏得起机器上做，无论结果如何都报告 `POST-BooterLoad verify` 行。 |
| 你想让 170HX 与 Volta 或 Maxwell 卡共存 | 这正是 580 移植的动机：580 覆盖从 980 Ti 到 A100 的一切。移植分支以源码形式回答了这个问题，除此之外没有别处。 |

> [!CAUTION]
> **在裸机上做驱动补丁开发是破坏性的**
>
> 一位开发者报告说每次部署砸了的 `nvidia.ko` 后都需要重装操作系统。公认的补救措施是在虚拟机或容器里测试修改过的驱动。对于 Proxmox 直通，请使用 **SeaBIOS，而不是 UEFI/OVMF**：UEFI 会产生 RM 初始化和适配器失败，看起来就像漏洞利用根本没生效，这种误诊至少让两个人浪费了大量时间。

---

## 切换版本或分支

受支持的路径是**先移除，再安装**。维护者的原话："In fact, I would always recommend to remove the old one before adding the new one."

```bash
sudo ./remove.sh --yes
```

没有 `uninstall.sh`，无论 `master` 还是 `docs` 分支，尽管 `docs/INSTALLATION.md` 这么说。

话虽如此，这是建议而非铁律。一位克隆了不同分支并覆盖安装到现有安装之上的测试者报告说这样不行，先卸载就修好了；至少另外两位测试者覆盖安装没有问题，非正式共识是大多数人直接叠上去。没有人找到差异化的因素。

任何安装后，三个元数据文件会被写在模块旁边，位于 `/lib/modules/$(uname -r)/updates/cmpunlocker/`：

| 文件 | 内容 |
|---|---|
| `driver_version` | 例如 `610.43.03` |
| `card_profile` | `8gb` 或 `10gb` |
| `unlock_geometry` | `64GB` 或 `40GB` |

**内核模块中没有任何代码读取它们。** 它们只是安装时的记账。打过补丁的内核在启动时读取的唯一文件是可选的 `/lib/firmware/nvidia/ga100/gsp/dmem.bin`。如果你需要知道实际加载的是哪个版本，读 `cat /proc/driver/nvidia/version`（它应该**不是** `dvs-builder`），并用 `sudo dmesg | grep SEC2_DEBUG` 确认。

---

## 本页的未解决问题

> [!NOTE]
> **未解决问题**
>
> 1. **610.43.02 和 610.43.03 哪个更可靠？** 被反复问起，从未回答。
> 2. **595 / 590 / 580 移植到底能不能启动？** 每个分支只要一位测试者报告 `dmesg | grep SEC2_DEBUG` 和 `POST-BooterLoad verify` 行就能定论。
> 3. **移植分支 `VERSION` 中七个未验证的小版本能不能应用？** 可以用 `patch -p1 --dry-run` 离线回答。
> 4. **移植分支会不会与 Gen2 或多卡血统合并。** 它们是独立开发的。现在选一个意味着放弃另一个。合并结构上很简单，因为移植只改 `PATCH_DIR` 的计算，但需要在 580、590 和 595 源码上重新生成 Gen2 补丁 `0007` 和 `0008`。
> 5. **WSL 和 HiveOS 支持。** 两者都被问过，都没被回答，两边都没有证据。

---

## 相关页面

- [The six driver patches](../unlock/driver-patches.md)
- [Install](install.md) 与 [Verify](verify.md)
- [Troubleshooting](troubleshooting.md)
- [Multi-GPU](multi-gpu.md)
- [PCIe Gen2](../unlock/pcie-gen2.md)
- [Open questions](../frontier/open-questions.md)
