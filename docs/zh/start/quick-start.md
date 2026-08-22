# 快速入门

**本页涵盖：** 从原厂 CMP 170HX 到已解锁卡的最短正确路径，使用发布版 `cmpunlocker` 的 `master` 分支。精确命令、每一步的预期输出，以及一张失败到页面的路由表。这里没有任何实验性内容：下面的一切都在发布版代码树里。

整个过程是：安装 nvidia-open `610.43.03`（或 `610.43.02`），运行 `sudo ./install.sh`，冷启动，检查 `nvidia-smi`。8 GB 卡最终得到 **65536 MiB**；10 GB 卡最终得到 **40960 MiB**。完整 SM 计算吞吐同时解锁。改动仅限寄存器：不写闪存，`sudo ./remove.sh --yes` 把卡还原到原厂状态。

> [!WARNING]
> **快速入门不会给你什么**
>
> - **没有 PCIe Gen2。** 发布版 `master` 只含补丁 `0001` 至 `0006`。Gen2 补丁（`0007-pcie-gen2.patch`、`0008-pcie-gen2-probe-retrain.patch`）在未发布分支上。你仍处于第 1 代、2.5 GT/s。见 [PCIe Gen2](../unlock/pcie-gen2.md)。
> - **没有 x16 链路宽度。** 卡出厂时第 4–15 通道的交流耦合电容未焊接，所以以 x4 训练。恢复 x16 需要手工焊接 24 颗 0402 220 nF X7R 电容。那是与链路速率物理上独立的成就，且永远不会改变 PCIe 代际。见[物理改装](../operations/physical-mods.md)。
> - **10 GB 卡上没有 80 GB。** 那个配置被构建、测试并因不稳定而放弃。见 [80 GB 档位](../frontier/80gb.md)。
> - **没有 ECC、没有 NVLink、没有对等直连。** ECC 和 NVLink 是 OTP 熔丝禁用的，无已知杠杆。对等直连同样缺失，但那是熔丝还是驱动闸门从未确定。见[对等直连](../frontier/p2p.md)。
> - **仅限 Linux。** 解锁依托 Linux 的 GSP 引导路径。Windows 是完全不同的驱动模型。
>
> 预期 Gen1 x4 下约 **0.85 GB/s** 的主机到设备带宽（实测，clpeak）。那是跳过上面两个硬件/分支项目的主要实际代价。

---

## 前置条件清单

| 要求 | 检查 | 备注 |
|---|---|---|
| x86-64 Linux、root | `id -u` 在 `sudo` 下返回 `0` | `install.sh` 以 `Run as root: sudo ./install.sh` 死掉 |
| 一块 CMP 170HX | `lspci -nn \| grep -iE '10de:20b0\|10de:20c2\|10de:2082'` | `20c2` = 8 GB SKU，`2082` = 10 GB SKU |
| **nvidia-open 610.43.03 或 610.43.02** | `cat /proc/driver/nvidia/version` | 精确字符串匹配。其他任何版本都中止安装 |
| 内核头文件 | `ls -d /lib/modules/$(uname -r)/build` | 软件包 `linux-headers-$(uname -r)` 或 `kernel-devel` |
| Secure Boot **已禁用** | `mokutil --sb-state` | 打补丁的模块未签名 |
| 网络访问 | 仅首次安装 | `build.sh` 下载匹配的原厂 `open-gpu-kernel-modules` 压缩包 |
| `python3`、`curl`、`patch`、`make`、C 工具链 | `command -v python3 curl patch make gcc` | `master` 上不使用 PyYAML，发布脚本中也没有显式的 GCC 版本检查 |
| 一个 initramfs 工具 | `update-initramfs`、`dracut` 或 `mkinitcpio` | 没有的话构建会警告，启动时原厂模块可能获胜 |
| 供电：1 × EPS 8-pin | 300 W 额定接口 | 需要 2 × PCIe 转 EPS 转接线。见[供电与电源](../operations/power-and-psu.md) |
| 散热：强制风冷 | 被动散热片，卡上无风扇 | 见[散热](../operations/cooling.md) |
| 一块有显示输出的 GPU，或一块能无头 POST 的主板 | 170HX 没有视频输出 | 至少有一块主板被报告仅装 170HX 时拒绝 POST |

> [!NOTE]
> **这张卡不解锁也能工作**
>
> 原厂 170HX 在普通发行版驱动上运行良好（`nvidia-driver-570` 加 Ubuntu 24.04 上的 CUDA 12.8 已确认）。`nvidia-smi` 叫它 `NVIDIA Graphics Device`、计算能力 8.0，因为驱动的 PCI ID 表里没有 `0x20C2` 的市场名。把它当作动任何东西之前的预检健全性检查。能被驱动和已被解锁是两回事。

---

## 第 0 步：确认你手里是哪张卡

```bash
lspci -nn | grep -iE '10de:20b0|10de:20c2|10de:2082'
nvidia-smi --query-gpu=memory.total,driver_version --format=csv
```

| PCI ID | 物理 | 解锁到 | CFG1 `0x009a0204` | LMR `0x00100ce0` |
|---|---|---|---|---|
| `10de:20c2` | 8 GB | **64 GB**（65536 MiB） | `0x02779000` | `0x0000020B` |
| `10de:2082` | 10 GB | **40 GB**（40960 MiB） | `0x02669000` | `0x0000028A` |
| `10de:20b0` | 视情况 | **什么都没有** | n/a | n/a |

`install.sh` 检测全部三个 ID，但驱动内闸门 `_kgspSec2PostblTimingEnabled()` 只接受 `0x20C2` 和 `0x2082`。`20b0` 卡干净安装然后永不解锁；安装器警告 `This card reports 0x20b0; install will continue, but unlock may not activate.`
更多细节：[识别你的卡](identify-your-card.md)。

---

## 第 1 步：安装 nvidia-open 610.43.0x

用你的发行版提供的任何方式，或 NVIDIA 的 `.run` 安装器，只要结果恰好是 `610.43.03` 或 `610.43.02` 的开源内核模块。继续前先验证：

```bash
cat /proc/driver/nvidia/version
# NVRM version: NVIDIA UNIX Open Kernel Module for x86_64  610.43.03  Release Build ...
nvidia-smi
```

`610.43.03` 是默认构建目标（`driver/VERSION` 的第一行）。

> [!NOTE]
> **开放问题**
>
> `610.43.02` 和 `610.43.03` 哪个更可靠被反复问起，从未得到回答。两个版本上都有成功解锁。`610.43.03` 只是排在列表第一个。

考虑把驱动包固定在 610，这样发行版升级不会悄悄把你移出受支持版本。见[驱动版本](../procedures/driver-versions.md)。

## 第 2 步：获取工具并运行

```bash
git clone https://github.com/amoghmunikote/cmpunlocker
cd cmpunlocker
sudo ./install.sh
```

只有当自动检测错误或 `nvidia-smi` 不可用时才强制指定配置：

```bash
sudo ./install.sh --profile=8gb     # 8 GB 卡  -> 64 GB 几何
sudo ./install.sh --profile=10gb    # 10 GB 卡 -> 40 GB 几何
```

自动检测读取 `nvidia-smi --query-gpu=memory.total` 并分档：
`>= 60000 MiB -> 8gb`（已解锁卡）、`35000-59999 -> 10gb`、`7680-8704 -> 8gb`、
`9728-10752 -> 10gb`。其他任何值都以 `Could not detect 8GB vs 10GB card` 中止。

**预期输出**，六个编号步骤，全部同时写入 `logs/install_YYYYmmdd_HHMMSS.log`：

```text
Step 1/6: Verifying root privileges
✓ Running as root
Step 2/6: Detecting CMP 170HX GPU
✓ GPU detected: 0000:0b:00.0 (10de:20c2)
Step 3/6: Selecting card memory profile
✓ Detected stock/reported memory 8192 MiB → profile 8gb
==> Unlock geometry: 64GB (CFG1=0x02779000 LMR=0x0000020B)
Step 4/6: Verifying nvidia-open (610.43.03,610.43.02)
✓ NVIDIA driver 610.43.03 is supported
✓ Kernel headers present for 6.8.0-136-generic
Step 5/6: Building and installing patched modules
...
Step 6/6: Done
Profile: 8gb → expect ~65536 MiB after unlock
```

你传的配置只是元数据。在发布版 `master` 上，两种几何都烘焙进打补丁的 `kernel_gsp.c`，在 GSP 启动时根据实时 PCI 设备 ID 选择，所以误检的配置会写错标签，但不可能产生错误的几何。

两行看起来吓人实则无碍的构建日志：
`Skipping BTF generation for .../nvidia*.ko due to unavailability of vmlinux`（内核调试元数据，无关紧要），以及 `[drm] No compatible format found`（卡没有显示输出）。

## 第 3 步：冷启动

`build.sh` 会尝试热重载模块。如果成功你可以跳过这步，但冷启动才是可靠路径，也是安装器推荐的。

```bash
sudo shutdown -h now
```

然后在电源处断电或拔掉电源线，等待 **60 秒**让电容放电、WPR2 清除，再上电。热重启会让 WPR2 保持，不等价。

## 第 4 步：验证

```bash
nvidia-smi
# 8 GB 卡：  ~65536 MiB
# 10 GB 卡： ~40960 MiB

sudo dmesg | grep SEC2_DEBUG
cat /lib/modules/$(uname -r)/updates/cmpunlocker/card_profile   # 8gb or 10gb
```

健康的 `SEC2_DEBUG` 轨迹按顺序打印：WPR 元数据转储、`saved WPR2 lo=... hi=...`、四行 `PLM[n] ...`、一行 `PLMs:` 汇总、`POST-WRITE` 行、`WPR meta updated` 行、`normal BooterLoad status=0x0`、最后的 `POST-BooterLoad verify`，然后是静态信息的前后配对。
PLM 行的形状如下（一行存档原文逐字引用，其余同格式）：

```text
SEC2_DEBUG: PLM[3] FEAT(0x823804) attempt=0 status=0xffff reg=0xffffffff
```

预期回读：

| 行 | 寄存器 | 预期值 |
|---|---|---|
| `PLM[0] WPR_CFG` | `0x001fa7cc` | `0xfffff0ff`（**不是** `0xffffffff`） |
| `PLM[1] FBPA` | `0x009a0148` | `0xffffffff` |
| `PLM[2] WPR` | `0x001fa7c4` | `0xffffffff` |
| `PLM[3] FEAT` | `0x00823804` | `0xffffffff` |
| `POST-WRITE SS0` | `0x0082381c` | `0x88888888` |
| `POST-WRITE SS1` | `0x00823820` | `0x00000008` |
| `POST-WRITE CFG1` | `0x009a0204` | `0x02779000`（8 GB）/ `0x02669000`（10 GB） |
| `POST-WRITE LMR` | `0x00100ce0` | `0x0000020B`（8 GB）/ `0x0000028A`（10 GB） |

> [!NOTE]
> **忽略三行吓人的输出**
>
> - 每行 PLM 上的 `status=0xffff` 是**正常的**。载荷 Booter 运行本来就该被拒绝；成功与否以寄存器回读为准，绝不看状态。来自 `s_executeBooterUcode_TU102` 的 `0x31` 是同一个故事。
> - `SEC2_DEBUG: /lib/firmware/nvidia/ga100/gsp/dmem.bin not found (0x59), using built-in payload` 是正常路径。那个文件是开发覆盖钩子。
> - 声称"所有 PLM 必须显示 `0xffffffff`"的第三方文档是错的。`WPR_CFG` 按设计就是 `0xfffff0ff`。
>
> 唯一**必须**读零的是 `SEC2_DEBUG: normal BooterLoad status=0x0`。

计算解锁以吞吐而非时钟字段确认。解锁卡上的持续 SM 时钟是 1410 MHz（`nvidia-smi -pl 300` 下 1470 MHz）。`nvidia-smi --query-gpu=clocks.max.sm` 报告 1935 MHz，但那是低置信度的报告最大值字段，不是可达时钟：VBIOS 表最大值是 1695 MHz。改用真实基准。见[性能](../operations/performance.md)。

---

## 如果失败，去这里

| 症状或精确消息 | 可能原因 | 去哪里 |
|---|---|---|
| `No CMP 170HX GPU found (10de:20b0 / 10de:20c2 / 10de:2082)` | 卡未枚举、主板无头未 POST、插接或供电 | [识别你的卡](identify-your-card.md)、[排障](../procedures/troubleshooting.md) |
| `Installed driver is X, but cmpunlocker requires one of: 610.43.03,610.43.02.` | 驱动版本不受支持 | [驱动版本](../procedures/driver-versions.md) |
| `Secure Boot is enabled. Disable it before installing unsigned patched modules.` | Secure Boot 开启 | [安装](../procedures/install.md) |
| `Kernel headers missing for <kver>` | 没有 `linux-headers-$(uname -r)` | [安装](../procedures/install.md) |
| `Could not detect 8GB vs 10GB card` | `nvidia-smi` 缺失或 `memory.total` 超出范围 | 用 `--profile=8gb` 或 `--profile=10gb` 重跑 |
| 构建在下载期间停止 | 无网络，或压缩包标签不可达 | [安装](../procedures/install.md) |
| `Resolved nvidia.ko is not under updates/cmpunlocker/, stock may still win` | depmod 解析或 initramfs 仍持有原厂模块 | [排障](../procedures/troubleshooting.md) |
| `Loaded nvidia srcversion (X) != patched (Y)` | 热重载后原厂模块仍在内存 | 冷启动，然后[排障](../procedures/troubleshooting.md) |
| 重启后 `nvidia-smi` 仍显示 8192 / 10240 MiB | PLM 打开未生效，或原厂模块在运行 | 完整断电冷循环，然后[排障](../procedures/troubleshooting.md) |
| **完全**没有 `SEC2_DEBUG` 行 | 打补丁的模块从未运行 | [排障](../procedures/troubleshooting.md)、[验证](../procedures/verify.md) |
| `WPR2 already up` / `RmInitAdapter failed! (0x62:0x40:2028)` / `No devices were found` | GSP 引导让 WPR2 保持编程状态 | [恢复](../procedures/recovery.md) |
| Xid 119、`Timeout after 60s ... Expected function 4097 (GSP_INIT_DONE)` | GSP 从未到达 RM 初始化 | [恢复](../procedures/recovery.md) |
| `nvidia-smi` 报告 "driver/library version mismatch" | 用户态与已加载模块不匹配 | [排障](../procedures/troubleshooting.md) |
| 多 GPU 盒子里没有卡解锁 | 原厂和打补丁的 `nvidia.ko` 同时存在于唯一的 `updates` 搜索路径下，depmod 任意加载其中一个 | [多卡](../procedures/multi-gpu.md) |
| Xid 31、`FAULT_INFO_TYPE_REGION_VIOLATION`、卡直到重启前不可用 | 分配越过窗口可用顶部 | [LLM 推理](../operations/llm-inference.md)、[排障](../procedures/troubleshooting.md) |
| 链路仍报告 Gen1 x4 | `master` 上符合预期：那里不发布 Gen2 补丁 | [PCIe Gen2](../unlock/pcie-gen2.md) |

开支持工单前，收集 `sudo dmesg | grep SEC2_DEBUG` 和最新的 `logs/install_*.log`。预期缓慢的单人响应。

---

## 回滚

```bash
sudo ./remove.sh --yes
```

卸载器是 `remove.sh`，要求 `--yes` 或 `-y`。**没有 `uninstall.sh`**，尽管某个分支的 `INSTALLATION.md` 这么说。它删除每个内核下的 `/lib/modules/*/updates/cmpunlocker/`，重跑 `depmod`，重建 initramfs，清除遗留的 systemd 和 `/opt/cmpunlocker` 残留，并重载原厂模块。如果 GPU 没有干净回来，重启。一位测试者报告之后两张卡都恢复正常挖矿，这正是称该改动无破坏性的依据。从未确认过永久变砖。完整细节：[卸载](../procedures/uninstall.md)。

在分支间切换时，维护者的建议是先卸载再安装。那是建议而非硬性规则：一位测试者遇到一个先卸载就修好的失败，至少另外两人直接在原基础上安装成功。

---

## 接下来去哪里

- [正确验证](../procedures/verify.md)，包括解锁的显存是否真实、不是别名折叠。
- 在不可替换的卡上跑之前读[风险](risks.md)。
- [显存几何](../unlock/memory-geometry.md)和[计算限速](../unlock/compute-throttle.md)：那四次寄存器写入实际做什么。
- [驱动补丁](../unlock/driver-patches.md)：六个补丁逐 hunk 通读。
- [术语表](glossary.md)：PLM、WPR2、SEC2、GSP-RM、FBPA、LMR 和 CFG1。
