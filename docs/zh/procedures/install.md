# 安装解锁

**本页涵盖：** 在 CMP 170HX 上安装正式发布的 `cmpunlocker` 驱动补丁的完整受支持流程：开始前必须满足什么、确切的命令、`install.sh` 和 `driver/build.sh` 每一步做什么、卡配置如何选择（以及何时强制指定）、为什么冷重启很重要，以及一次正确的运行在屏幕上和 `dmesg` 里应该是什么样。

简短版：安装 nvidia-open **610.43.03** 或 **610.43.02**，关闭 Secure Boot，安装内核头文件，然后从仓库克隆运行 `sudo ./install.sh` 并冷启动。脚本会为你的驱动版本下载 NVIDIA 原版 `open-gpu-kernel-modules` 压缩包，应用六个补丁，构建五个内核模块，并把它们安装到 `/lib/modules/$(uname -r)/updates/cmpunlocker/`。不会向卡的 VBIOS 写入任何东西，磁盘上也不会有固件文件被修改。8 GB 卡（`10de:20c2`）重启后报告 **65536 MiB**，10 GB 卡（`10de:2082`）报告 **40960 MiB**。

解锁本身在 [How the unlock works](../unlock/how-it-works.md) 中描述，补丁系列在 [Driver patches](../unlock/driver-patches.md) 中。本页只是操作流程。

---

## 前置条件

| 要求 | 详情 | 由谁强制 |
|---|---|---|
| 操作系统 | Linux，x86-64。此解锁**没有 Windows 路径**。 | 不检查；补丁只存在于 Linux GSP 启动路径 |
| 权限 | root（`sudo ./install.sh`） | `install.sh` 第 1 步、`build.sh` |
| 显卡 | `10de:20c2`（8 GB）或 `10de:2082`（10 GB）。`10de:20b0` 会被检测到但**不会**被解锁。 | `install.sh` 第 2 步（`lspci` grep）和驱动内设备门控 |
| 驱动 | nvidia-**open** `610.43.03`（默认）或 `610.43.02`，精确字符串匹配 | `install.sh` 第 4 步和 `driver/build.sh`，都对照 `driver/VERSION` |
| 内核头文件 | `/lib/modules/$(uname -r)/build` 必须存在 | `install.sh` 第 4 步和 `build.sh` |
| Secure Boot | 已禁用；补丁模块未签名 | `install.sh` 第 4 步，通过 `mokutil --sb-state` |
| 网络 | 首次安装时可达的 `github.com`，用于获取源码压缩包 | `build.sh` 中的 `curl -L --fail` |
| 工具链 | `python3`、`patch`、`make`、`curl`、可用的内核构建环境 | `build.sh` 只检查 `python3` |

实践中重要的说明：

- **nvidia-open，不是专有驱动。** 闭源驱动启动路径不同，不能用同样方式打补丁。卡在原版驱动下*能正常驱动*（一位测试者开箱即用 Ubuntu 24.04 上的 `nvidia-driver-570` 加 CUDA 12.8，Ubuntu 22.04 上的 `nvidia-driver-535-server` 也被报告可用），但能驱动和解锁是两回事。参见 [Driver versions](driver-versions.md)。
- **Secure Boot 检查是有条件的。** 它只在 `/sys/firmware/efi` 存在**且** `mokutil` 在 `PATH` 上时运行。在非 EFI 机器或未安装 `mokutil` 的机器上，检查会被静默跳过，你仍可能以内核拒绝加载的模块收场。`dmesg` 中的症状是 `nvidia: module verification failed: signature and/or required key missing - tainting kernel`。
- **没有 PyYAML，没有 GCC 版本检查。** `build.sh` 只用带标准库的普通 `python3`，不做任何编译器版本测试。网上流传的 "python3 with PyYAML / gcc 13+" 前置要求来自第三方 `unlock-cmp-170hx` 指南仓库，不是来自这些脚本；一个固定 `pyyaml>=5.1` 的遗留 `requirements.txt` 也放在六个 cmpunlocker 分支上，不过没有任何分支脚本 import 过 `yaml`。泄露的预构建包 README 只要求 root 权限和内核头文件。Ubuntu 26.04 LTS 内核 7.0.0-27-generic（`Gen2` 分支，扛过了多次重启）上有成功构建的报告。
- **在一台你坏得起的机器上做。** 裸机上的驱动补丁迭代破坏性很强，一位开发者报告每次部署砸了的 `nvidia.ko` 后都重装系统。参见 [Risks](../start/risks.md)。

> [!CAUTION]
> **固件补丁时代的遗留状态**
>
> 如果这台机器曾经跑过 `cmpunlocker` 的**固件补丁前代方案**，在安装驱动补丁**之前**先把 `gsp_tu10x.bin` 恢复到原版：
>
> ```bash
> GSP_DIR=/lib/firmware/nvidia/610.43.03
> sudo cp "$GSP_DIR/gsp_tu10x.bin.cmpunlocker.bak" "$GSP_DIR/gsp_tu10x.bin"
> ```
>
> 打过补丁的驱动会在启动期间把固件的签名保存为"stock"。如果磁盘上的固件仍是补丁版，驱动会保存漏洞利用负载的签名，干净的 GSP-RM 启动随后会 DMA 错误的 ROP 链。之后要找的成功行是 `SEC2_DEBUG: saved stock signature (4096 bytes)`。

---

## 命令

```bash
git clone https://github.com/amoghmunikote/cmpunlocker
cd cmpunlocker
sudo ./install.sh
```

自动检测错误或 `nvidia-smi` 不可用时强制指定配置：

```bash
sudo ./install.sh --profile=8gb     # 8 GB physical card  -> 64 GB geometry
sudo ./install.sh --profile=10gb    # 10 GB physical card -> 40 GB geometry
sudo ./install.sh --help
```

只接受这三种标志形式（`--profile=8gb|8GB|10gb|10GB`、`-h`、`--help`）。任何其他参数都会以 `Unknown argument: <arg>` 退出 1。

所有输出都会同时写入检出目录内的 `logs/install_<YYYYmmdd_HHMMSS>.log`，所以要从一个可写目录运行。

---

## `install.sh` 一步步做了什么

脚本在 `set -euo pipefail` 下有六个编号步骤。

### 第 1/6 步：root

`[[ "${EUID}" -eq 0 ]]`，否则以 `Run as root: sudo ./install.sh` 报错退出。

### 第 2/6 步：GPU 检测

```bash
lspci -nn | grep -iE '10de:20b0|10de:20c2|10de:2082' | head -1
```

无匹配即致命：`No CMP 170HX GPU found (10de:20b0 / 10de:20c2 / 10de:2082)`。注意 `head -1`：**master 是单卡安装器**。它只记录第一条匹配的 BDF（总线、设备、功能地址）和那一行的设备 ID。机器里有多张卡时参见 [Multi-GPU](multi-gpu.md)。

如果检测到的设备 ID 既不是 `20c2` 也不是 `2082`，脚本会警告并**继续**：

```text
! In-driver unlock path is gated on PCI ID 0x20C2 / 0x2082.
! This card reports 0x20b0; install will continue, but unlock may not activate.
```

这是准确的。驱动内门控 `_kgspSec2PostblTimingEnabled()` 只接受 `0x20C2` 和 `0x2082`，所以 `20b0` 卡会得到完整补丁的模块，但永远不会为它触发。README 中旧的"unlock is `0x20C2`-gated"说法已经过时；自提交 `0f9aca5` "Unlock isn't gated anymore" 起 `0x2082` 就是一等目标。

### 第 3/6 步：卡配置

要么用 `--profile` 覆盖，要么用 `detect_card_profile()`，它读取 `nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits | head -1` 并映射四个窗口：

| 报告的 `memory.total` | 选中的配置 | 为什么有这个窗口 |
|---|---|---|
| `>= 60000` MiB | `8gb` | 在**已解锁**的 64 GB 卡上重装 |
| `35000`-`59999` MiB | `10gb` | 在已解锁的 40 GB 卡上重装 |
| `7680`-`8704` MiB | `8gb` | 出厂 8 GB 卡（8192 MiB） |
| `9728`-`10752` MiB | `10gb` | 出厂 10 GB 卡（10240 MiB） |
| 其他任何值 | 致命 | 打印 `unknown:<mib>`，然后 `Could not detect 8GB vs 10GB card. Re-run with --profile=8gb or --profile=10gb` |

然后横幅打印以下之一：

```text
==> Unlock geometry: 64GB (CFG1=0x02779000 LMR=0x0000020B)
==> Unlock geometry: 40GB (CFG1=0x02669000 LMR=0x0000028A)
```

> [!WARNING]
> **自动检测在混合 GPU 主机上不安全**
>
> `detect_card_profile()` 取 **`nvidia-smi` 顺序中的第一张 GPU**，它不一定是 `lspci` 找到的那张 CMP。一台同时有 RTX 3080 10 GB 和 8 GB CMP 170HX 的主机，至少两个人复现过从 3080 检测出 "10GB"。另一份报告把其他 CMP SKU（50HX）误检为 10 GB 170HX。在当前 `master` 上后果只是元数据错误，但在任何带其他 NVIDIA 卡的主机上，安全的习惯是**始终显式传 `--profile`**。如果第一张 GPU 报告的容量落在全部四个窗口之外（比如一张 24 GB 卡），安装会直接终止。

### 第 4/6 步：Secure Boot、驱动版本、头文件

- Secure Boot：如果 `/sys/firmware/efi` 存在、`mokutil` 可用且 `mokutil --sb-state` 匹配 `SecureBoot enabled`，以 `Secure Boot is enabled. Disable it before installing unsigned patched modules.` 终止。
- 驱动版本检测顺序：
  1. `/proc/driver/nvidia/version`
  2. `nvidia-smi --query-gpu=driver_version`
  3. 探测 `/lib/firmware/nvidia/<supported-version>/` 目录
  4. `/lib/firmware/nvidia/` 下排序最高的目录
- 检测到的字符串必须精确匹配 `driver/VERSION` 中的一行，否则：`Installed driver is <detected>, but cmpunlocker requires one of: 610.43.03,610.43.02.`
- `/lib/modules/$(uname -r)/build` 必须存在，否则 `Kernel headers missing for <kver>. Install linux-headers-<kver> or kernel-devel.`

### 第 5/6 步：构建并安装

`install.sh` 给 `driver/build.sh` chmod 并在环境中带 `CMPUNLOCKER_DRIVER_VERSION` 和 `CMPUNLOCKER_CARD_PROFILE` 执行它。见下一节。

### 第 6/6 步：后续步骤横幅

打印预期的解锁后容量，然后是四个编号的后续步骤：冷重启提醒（`sudo shutdown -h now`）和三条验证命令（`nvidia-smi`、`sudo dmesg | grep SEC2_DEBUG`、`nvidia-smi --query-gpu=clocks.max.sm --format=csv,noheader`），最后是安装日志的路径。

---

## `driver/build.sh` 做什么

1. **重新验证** root、对照 `driver/VERSION` 的版本、补丁目录、内核头文件，以及 `python3` 的存在。
2. **下载** `https://github.com/NVIDIA/open-gpu-kernel-modules/archive/refs/tags/${VERSION}.tar.gz`，用 `curl -L --fail` 下载到 `driver/.build/`（用 `CMPUNLOCKER_BUILD_DIR` 覆盖缓存位置）。缓存的压缩包会被复用。仓库不携带任何 NVIDIA 代码。
3. **每次干净解压**：`rm -rf "${SRC_DIR}"` 然后解压，所以之前失败的构建不能污染下一次。
4. **按 glob（字典序）顺序**用 `patch -p1` 应用 `driver/patches/*.patch` 中的每一个。正式发布系列是六个文件，总计 37,415 字节：

   | 补丁 | 字节 | 作用 |
   |---|---|---|
   | `0001-sec2-postbl-plm-ss-cfg.patch` | 19,741 | 整个解锁：负载、[PLM](../unlock/privilege-level-masks.md) 循环、SS0/SS1/CFG1/LMR 写入、`fb_length` 重写 |
   | `0002-booter-verify.patch` | 3,988 | 软失败四条启动断言，打印 BooterLoad 后回读 |
   | `0003-late-pma.patch` | 10,580 | 把 8 GiB 以上的新显存注册给物理内存分配器 |
   | `0004-bar0-pramin-clamp.patch` | 861 | 把 BAR0/PRAMIN 窗口钳制到出厂 8192 MB 偏移 |
   | `0005-ce-scrub-workarounds.patch` | 1,642 | 强制复制引擎清理器进入物理模式 |
   | `0006-persistent-sw-state.patch` | 603 | 设置 `NV_FLAG_PERSISTENT_SW_STATE`，取代旧看门狗守护进程 |

   因为循环在 `set -euo pipefail` 下是普通 glob，把第三方 diff 命名为 `0007-*.patch` 放进该目录就能干净地组合，任何失败的 hunk 都会中止构建。P2P 补丁就是这么叠加的（参见 [P2P](../frontier/p2p.md)）。
5. **运行配置步骤。** 一个内联 Python 脚本检查打过补丁的 `kernel_gsp.c` 是否已经包含全部六个标记（`SEC2_POSTBL_TIMING_CMP_170HX_8GB_PCI_DEVICE_ID`、`..._10GB_PCI_DEVICE_ID`、`0x02779000U`、`0x02669000U`、`0x0000001000000000ULL`、`0x0000000A00000000ULL`）。在 `master` 上六个都存在，所以它打印 `runtime device-id geometry (profile metadata=64GB)` 并退出，不编辑任何东西。它下面的正则替换分支是为单 SKU 补丁准备的死代码遗留。
6. **写三个元数据文件**到 `/lib/modules/$(uname -r)/updates/cmpunlocker/`：`driver_version`、`card_profile`（`8gb` / `10gb`）、`unlock_geometry`（`64GB` / `40GB`）。**内核模块中没有任何代码读取它们。** 它们是为人类和 `verify.sh` 存在的。打过补丁的内核在启动时读取的唯一文件是可选的 `/lib/firmware/nvidia/ga100/gsp/dmem.bin`。
7. **构建**：`rm -rf src/nvidia/_out src/nvidia-modeset/_out kernel-open/conftest`、`make clean`，然后 `make -j$(nproc) modules SYSSRC=/lib/modules/$(uname -r)/build`。报告称现代 CPU 上构建时间 **2 到 5 分钟**。这个区间来自两份书面传播（泄露的预构建包 README 说 "~2-5 min"，一份流传的 40 GB 解锁指南说 "~5 minutes on a modern CPU"）；没有人贴出计时实测。
8. **安装五个模块**，模式 `0644`，到 `/lib/modules/$(uname -r)/updates/cmpunlocker/`：`nvidia.ko`、`nvidia-modeset.ko`、`nvidia-uvm.ko`、`nvidia-drm.ko`、`nvidia-peermem.ko`（用 `find` 找到，排除 `*/conftest/*`）。只有 `nvidia.ko` 携带解锁代码；其他四个是原版重建，随附以保证模块集版本一致。
9. **`depmod -a "${KVER}"`**。模块优先级是普通 depmod 排序：`updates/cmpunlocker/` > `updates/dkms/` > `kernel/drivers/`，这就是为什么不需要 `dpkg-divert`。
10. **重建 initramfs**，按可用顺序选择 `update-initramfs -u -k`、`dracut --force --kver`、`mkinitcpio -P`，否则警告 `No initramfs tool found, rebuild manually before rebooting`。`master` 的 `build.sh` 在这里没有注释，但分支副本（`memory`、`ecc`、`housekeeping`、`PG199`）逐字解释了推理：NVIDIA 经常从 initramfs 加载，如果只有 `updates/dkms` 被打包进去，那么即使 depmod 偏好 `updates/cmpunlocker`，原版模块也会在启动时胜出。这是"装了但显存仍显示出厂容量"的一条合理路径，但这是脚本的推理而非诊断出的现场故障：*initramfs*、*initrd*、*dracut* 和 *mkinitcpio* 这些词在聊天语料中一处都没出现。只有在 initramfs 步骤之后，`build.sh` 才运行其经验检查，确认补丁模块真的胜出：`modprobe -n -v nvidia | awk '/insmod/ {print $2; exit}'`，警告 `Resolved nvidia.ko is not under updates/cmpunlocker/, stock may still win`。
11. **尝试热重载**：停止 `nvidia-persistenced` 和 `nvidia-fabricmanager`，对 `nvidia_drm`、`nvidia_uvm`、`nvidia_modeset`、`nvidia` 执行 `modprobe -r`，然后重新加载。接着比较 `/sys/module/nvidia/srcversion` 与 `modinfo -F srcversion .../updates/cmpunlocker/nvidia.ko`，不匹配时警告 `Loaded nvidia srcversion (X) != patched (Y)` 并清除自己的成功标志。

> [!WARNING]
> **下载的压缩包没有完整性检查**
>
> `build.sh` 用 `curl -L --fail` 获取 NVIDIA 标签压缩包并缓存，树中任何地方都没有校验和或签名验证。在不可信网络上，请在首次构建前自行验证缓存的压缩包。

---

## 冷重启

热重启不够，热重载在一般情况下也不够。整个项目里的指示，包括泄露的预构建发行版自己的 README，都是**冷**重启：完全断电，然后开机。

```bash
sudo shutdown -h now
# then power on
```

原因，按坑人频率排序：

1. 解锁运行在补丁模块的 GSP 引导内部。如果因为热重载失败或 initramfs 仍装着原版模块，正在运行的 `nvidia.ko` 仍是原版，解锁就永远不会执行。
2. 使用中的模块（X11、显示管理器、persistenced 守护进程、CUDA 进程）会阻塞 `modprobe -r`，`build.sh` 打印 `Could not unload nvidia modules (in use), cold reboot required`。
3. 显存几何**不能**扛过函数级复位或断电重启，所以干净冷启动是补丁驱动从头重新应用一切的明确定义状态。只有 `0x00823804` 处的 SS0、SS1 和 FEAT PLM 活在常开岛上。

如果热重载成功了，`build.sh` 会说明，你可以立即验证。如果没有，脚本会自己打印恢复指示。

---

## 一次正确的运行应该是什么样

以下内容由脚本自己的字面输出字符串拼成（不是一份完整抓取记录），所以把可变部分当作占位符。

```text
╔════════════════════════════════════════╗
║               cmpunlocker              ║
╚════════════════════════════════════════╝

━━━ Step 1/6: Verifying root privileges ━━━
✓ Running as root

━━━ Step 2/6: Detecting CMP 170HX GPU ━━━
✓ GPU detected: 0000:0b:00.0 (10de:20c2)

━━━ Step 3/6: Selecting card memory profile ━━━
✓ Detected stock/reported memory 8192 MiB → profile 8gb
==> Unlock geometry: 64GB (CFG1=0x02779000 LMR=0x0000020B)

━━━ Step 4/6: Verifying nvidia-open (610.43.03,610.43.02) ━━━
✓ NVIDIA driver 610.43.03 is supported
✓ Kernel headers present for 6.8.0-136-generic

━━━ Step 5/6: Building and installing patched modules ━━━
[INFO]  Building against open-gpu-kernel-modules 610.43.03
[ OK ]  Using cached tarball .../driver/.build/open-gpu-kernel-modules-610.43.03.tar.gz
[INFO]  Applying unlock patches...
[INFO]    0001-sec2-postbl-plm-ss-cfg.patch
...
[ OK ]  All patches applied
runtime device-id geometry (profile metadata=64GB)
[ OK ]  Memory profile 8gb: CFG1=0x02779000 LMR=0x0000020B fb=0x0000001000000000 (64GB)
[ OK ]  Modules built
[ OK ]  Installed nvidia.ko
[ OK ]  depmod complete
[ OK ]  initramfs rebuilt
[INFO]  modprobe will load: /lib/modules/6.8.0-136-generic/updates/cmpunlocker/nvidia.ko
[ OK ]  Patched NVIDIA modules loaded
[ OK ]  Build and install finished. Verify with: nvidia-smi

━━━ Step 6/6: Done ━━━
Profile: 8gb → expect ~65536 MiB after unlock
```

启动后立即看到的决定性证据在内核日志里：

```bash
sudo dmesg | grep SEC2_DEBUG
```

一次健康的 8 GB 解锁会产生这些行。作为规模参考，存档的单卡 8 GB 抓取总共包含 **29** 行 `SEC2_DEBUG`，存档的双卡 Gen2 分支启动日志包含 **134** 行：

```text
SEC2_DEBUG: saved stock signature (4096 bytes)
SEC2_DEBUG: /lib/firmware/nvidia/ga100/gsp/dmem.bin not found (0x59), using built-in payload
SEC2_DEBUG: PLMs: FEAT=0xffffffff FBPA=0xffffffff WPR=0xffffffff WPR_CFG=0xfffff0ff
SEC2_DEBUG: POST-WRITE SS0=0x88888888 SS1=0x00000008 CFG1=0x02779000 LMR=0x0000020b (devId=0x20c2)
SEC2_DEBUG: late PMA extension status=0x0
SEC2_DEBUG: POST-BooterLoad verify PLM=... SS0=0x88888888 SS1=0x00000008 CFG1=0x02779000 LMR=0x0000020b
```

> [!NOTE]
> **不要把行数当作通过/失败测试**
>
> 行数不是可靠的跨构建指纹。记录值：存档单卡 8 GB 抓取 **29**、存档双卡 Gen2 分支 `610.43.03` 日志 **134**、报告工具给出 34（Gen1 构建）和 80（Gen2 构建），`pcielink.sh` 在两台独立双卡 Gen2 机器上打印 `SEC2_DEBUG lines=152`。不要把行数不一致读作安装失败。上面的寄存器回读行才是判据。

三件经常吓到首次安装者的事都是正常的：

- `WPR_CFG=0xfffff0ff` 是**通过**。四个 PLM 中只有三个以 `0xffffffff` 为目标。
- 每次负载通过时，逐次尝试的 Booter 状态 `0xffff` 都是预期的，无论成功与否。寄存器回读是唯一有效的成功判据。
- `dmem.bin` 的 `not found (0x59)` 无害；使用内置负载。

下一步阅读 [Verifying the unlock](verify.md) 获取完整日志解码和显存与算力的区别。如果有问题，去 [Troubleshooting](troubleshooting.md)。

---

## 重装、升级与切换分支

维护者声明的规则是切换分支时**先移除，再安装**："In fact, I would always recommend to remove the old one before adding the new one." 一位克隆了 `Gen2` 分支并覆盖安装到现有安装之上的测试者报告说这样不行，先卸载就修好了。至少另外两位测试者覆盖安装没有问题，非正式共识是大多数人"直接叠上去就完事"。失败是真实存在的但不普遍，没有人找到差异化的因素。先移除再安装是*受支持的*路径：

```bash
sudo ./remove.sh --yes     # in the OLD checkout
sudo ./install.sh          # in the NEW checkout
```

参见 [Uninstalling](uninstall.md)。

---

## 特定环境的说明

> [!WARNING]
> **实验性：虚拟化**
>
> 显存和算力解锁在 **Proxmox GPU 直通**下可用：一位运营者直通了八张 8 GB CMP 170 卡，全部解锁。记录在案的两个限制：
>
> - 使用 **SeaBIOS，不要用 UEFI/OVMF**。UEFI 会产生 RM 初始化和适配器失败，看起来就像漏洞利用根本没生效。这是第一手根因定位的，并立即被一位一直无法复现结果的人证实。
> - 截至 2026-07-24，PCIe Gen2 链路速率改动在虚拟机中**不工作**，维护者承认这是一个待调试的开放项。参见 [PCIe Gen2](../unlock/pcie-gen2.md)。

> [!NOTE]
> **未解决问题：缺少显示设备会让 GSP 不高兴吗？**
>
> 一位运营者观察到，与有显示设备的系统相比，没有 iGPU 也没有 BMC 显示设备的系统上 GSP 似乎更不高兴。没有人回应确认、反驳或错误字符串。在一台机器上、BIOS 中禁用 BMC 显示设备做一次 A/B，抓取 `dmesg`，就能定论。

Windows 对这个补丁是死路：解锁是针对 Linux 开源内核模块实现的，GSP 启动路径是 Linux 特有的。Windows 机器可以用 GRID 或数据中心驱动加注入的硬件 ID 来*驱动* 170HX，但那让你得到一张能用的卡，不是解锁的卡。

---

## 相关页面

- [Quick start](../start/quick-start.md) 精简版
- [Identify your card](../start/identify-your-card.md) 先确认你的是哪个 SKU
- [Verify](verify.md)、[Troubleshooting](troubleshooting.md)、[Recovery](recovery.md)
- [Multi-GPU](multi-gpu.md) 如果机器里不止一张卡
- [Driver versions](driver-versions.md) 仅 610 的约束和未发布的移植
- [Driver patches](../unlock/driver-patches.md) 每个补丁实际改了什么
