# 卸载与还原

**本页涵盖：** 如何干净地移除 cmpunlocker 驱动补丁、`remove.sh` 具体动了哪些东西、它刻意留下了什么、为什么硬件层面还原是安全的，以及一条被广泛复制但注定失败的指令（因为它所指的文件根本不存在）。

简短版：

```bash
sudo ./remove.sh --yes
```

这就是全部受支持的卸载方式。它会删除每个已安装内核下的 `/lib/modules/*/updates/cmpunlocker/`，重新运行 `depmod`，重建 initramfs，清理两个废弃的第一代设计的残留文件，并重新加载原版 NVIDIA 模块。下次冷启动后，显卡会恢复到出厂报告的 8192 MiB 或 10240 MiB；热重启不算复位，也没有证据表明它能清除几何（geometry）配置。

> [!CAUTION]
> **`uninstall.sh` 并不存在**
>
> `docs` 分支上的 `docs/INSTALLATION.md` 第 40 行指示运行 `sudo ./uninstall.sh --yes`。**仓库中任何地方都没有 `uninstall.sh`**，无论是 `master` 分支还是 `docs` 分支本身。运行它只会产生一个 shell 错误、什么都不做，有人把这种情况解读为"卸载程序静默失败了"。正确的命令是 `remove.sh --yes`。`docs` 分支还带有另外三处已知缺陷，不足为凭：参见 [Verify](verify.md#the-sec2_debug-dmesg-trail)。

---

## 为什么还原是安全的

这次解锁没有任何东西持久化在硬件中。没有 VBIOS 刷写、没有熔丝烧断、没有 EEPROM 写入，而且从出货设计以来，磁盘上也不再有固件文件被修改。解锁是由打过补丁的内核模块在每次启动 GSP 时执行的一系列易失性寄存器写入：

| 状态 | 能扛过函数级复位吗？ | 能扛过断电重启吗？ |
|---|---|---|
| SS0 `0x0082381c`、SS1 `0x00823820`、FEAT_OVR_PLM `0x00823804` | 能（常开岛，always-on island） | 不能 |
| CFG1 `0x009a0204`、各 FBPA 的 CFG1、CSTATUS、LMR `0x00100ce0`、FB 几何 PLM、AON LMR 影子 `0x001180f0` | 不能 | 不能 |

移除打过补丁的模块，这些写入就不再发生。这就是还原的全部机制。一位跑 HiveOS 的测试者报告，`remove.sh` 之后两块卡都恢复正常挖矿，这是称该改装的软件部分"无破坏性"的依据（目前仅此一例第一手报告）。

物理改装则是完全另一回事，本页的任何内容都**不会**撤销它们。如果显卡已经补焊了 PCIe AC 耦合电容，那就是焊死的硬件。参见 [Physical mods](../operations/physical-mods.md)。

---

## `remove.sh` 一步步做了什么

脚本拒绝在缺少 `--yes` 或 `-y` 的情况下运行。直接裸调用时，它会打印将要执行的操作摘要并以退出码 1 结束。

### 保护检查与第 1 步：root

`[[ "${EUID}" -eq 0 ]]`，否则以 `Run as root: sudo ./remove.sh --yes` 报错退出。输出会同时写入检出目录下的 `logs/remove_<YYYYmmdd_HHMMSS>.log`；若检出目录不可写则回退到 `/tmp`。

### 第 2/5 步：停止旧版 systemd 单元

停止并禁用 `cmpunlocker` 服务，删除 `/etc/systemd/system/cmpunlocker.service`，运行 `systemctl daemon-reload` 与 `reset-failed`，然后执行 `pkill -f /opt/cmpunlocker/daemon/watchdog.py`。

### 第 3/5 步：移除打过补丁的模块与旧版文件

- 对找到的每一个 `/lib/modules/*/updates/cmpunlocker` 目录（即**所有**已安装内核，不只是当前运行的内核）：`rm -rf`，然后 `depmod -a "${kernel}"`。
- 如果一个都没匹配到，会警告 `No patched kernel modules found`。
- 对每个涉及的内核重建 initramfs，让原版模块重新被打包进去，按可用顺序选择 `update-initramfs -u -k`、`dracut --force --kver` 或 `mkinitcpio -P`。这一步在卸载时和安装时同样重要：一个仍装着补丁模块的 initramfs 会继续加载它们。
- 删除每个 `gsp_tu10x.bin` 旁边的五个固件时代遗留文件：`.cmpunlocker.bak`、`.cmpunlocker.patched`、`.cmpunlocker.tmp`、`.cmpunlocker.cleanup`、`.cmpunlocker.pat`。
- 如果存在 `/opt/cmpunlocker` 则删除，否则警告 `/opt/cmpunlocker not found (ok for module-only installs)`。

> [!CAUTION]
> **这会删除你唯一的补丁时代 `gsp_tu10x.bin` 备份**
>
> 如果你正处于从固件补丁前代方案迁移的中途，且**尚未**恢复原版 GSP 固件，请在运行 `remove.sh` **之前**先恢复它。第 3 步会删除 `gsp_tu10x.bin.cmpunlocker.bak`，那是原始固件 blob 的副本。先恢复：
> `sudo cp /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin.cmpunlocker.bak /lib/firmware/nvidia/610.43.03/gsp_tu10x.bin`。

### 第 4/5 步：重新加载原版驱动

仅当 `lsmod` 显示存在 `nvidia` 模块时执行。按顺序：

1. 停止 `gdm3`、`sddm`、`lightdm`、`display-manager`，然后是 `nvidia-persistenced`。
2. `killall -9 Xorg Xwayland nvidia-persistenced`，等待 1 秒。
3. `modprobe -r nvidia_drm nvidia_uvm nvidia_modeset nvidia`（每个都忽略失败），等待 1 秒。
4. 如果仍有模块被加载，用 `rmmod -f` 强制卸载这四个模块。
5. `modprobe nvidia`，然后依次加载 `nvidia-modeset`、`nvidia-uvm`、`nvidia-drm`。失败时警告 `Could not reload NVIDIA driver, reboot to finish cleanup`。
6. 重启第一个原本启用的显示管理器。

> [!CAUTION]
> **第 4 步会终止你的图形会话**
>
> `remove.sh` 会停止显示管理器并用 `rmmod -f` 强制卸载模块。请从文本控制台或 SSH 运行，不要在你即将终止的桌面会话内的终端里运行。在无头计算节点上这无伤大雅；在工作站上，显示会消失，而且可能要等重启才会回来。

### 第 5 步：摘要

打印日志路径，如果 GPU 或显示不正常，则提示你 `sudo reboot`。

---

## `remove.sh` **不会**撤销什么

| 不处理的内容 | 为什么重要 | 手动操作 |
|---|---|---|
| 内核命令行 | `master` 的 `remove.sh` 完全不包含任何 `iommu` 或 `cmdline` 处理。IOMMU 配置只存在于 `Gen2`、`far` 和 `deced` 分支 | 如果你是从 `Gen2`、`far` 或 `deced` 安装的，请使用**同分支**的 `remove.sh`，它会从 `<file>.cmpunlocker.bak` 恢复并打印 `Reverted IOMMU kernel parameters (effective after reboot)`。改用 `master` 的 `remove.sh` 会让内核命令行被永久修改，并遗留一个孤儿文件 `/etc/default/grub.cmpunlocker.bak` |
| `/etc/modprobe.d/cmp-pcie-gen2.conf` | 由 Gen2 系安装器写入，内容为 `options nvidia NVreg_RegistryDwords="RmForceEnableGen2=1;RMPcieLinkSpeed=0x1"`（`far`/`deced` 上是 `0x2`）。`master` 从不创建也从不删除它 | `sudo rm /etc/modprobe.d/cmp-pcie-gen2.conf` 并重建 initramfs |
| `/usr/local/sbin/retrain.sh`、`cmpretrain.service` | 仅由 `debug-gen2` 分支安装。`Gen2` 安装器会移除它们；`master` 不知道它们的存在 | `sudo systemctl disable --now cmpretrain.service; sudo rm -f /usr/local/sbin/retrain.sh` |
| `/lib/firmware/nvidia/ga100/gsp/dmem.bin` | 如果你在那里放置了自定义负载覆盖文件，它会保留。未来的补丁版安装会把它读入负载缓冲区，而不是运行内置填充，但第一次 `kgspSec2PostblTimingRefillPayload()` 会在任何 Booter Load 消费它之前重写该缓冲区，所以在正式发布路径上这个文件没有效果 | 如果不是你有意放置的，就删除它 |
| `driver/.build/` 缓存 | 下载的 NVIDIA 源码压缩包和解压后的源码树，在检出目录里可能占用数百 MB | `rm -rf driver/.build` 或直接删除克隆 |
| `logs/` | 安装与卸载记录 | 保留它们；排查问题时很有用 |
| NVIDIA 驱动本身 | `remove.sh` 还原的是*补丁*，不是驱动包。nvidia-open 610.43.0x 仍保持安装状态 | 使用你的发行版包管理器 |
| 任何物理改动 | 电容改装、导流罩、供电转接线 | 不在范围内 |
| 显卡的 VBIOS | cmpunlocker 的任何部分都不会写入它 | 无需操作；参见 [VBIOS](../hardware/vbios.md) |

显卡的非易失状态中也没有任何需要还原的东西。每张被检查过的卡上，`0x008203f0` 处的 master kill 熔丝都读为 `0x00000000`（未熔断），解锁路径中没有任何操作会烧熔丝或写入 OTP。参见 [Fuses and OTP](../hardware/fuses-and-otp.md)。

---

## 验证还原

```bash
# Modules gone from every kernel
ls /lib/modules/*/updates/cmpunlocker 2>/dev/null   # expect: no output at all

# The stock module is what resolves and what is loaded
modprobe -n -v nvidia
cat /proc/driver/nvidia/version                      # should now say dvs-builder again

# Capacity back to stock (only after a cold boot)
nvidia-smi --query-gpu=memory.total --format=csv,noheader
#   8 GB card:  8192 MiB
#   10 GB card: 10240 MiB

# No unlock activity this boot
sudo dmesg | grep -c SEC2_DEBUG                      # expect 0 after a reboot
```

运行 `remove.sh` 后，显卡会一直报告解锁后的容量，直到冷启动。这是正常结果，不能证明补丁模块仍然驻留，因为几何寄存器能在驱动卸载并重新加载后存活。判断还原是否成功要看 `modprobe -n -v nvidia`、`/sys/module/nvidia/srcversion` 以及 `dmesg` 中没有 `SEC2_DEBUG` 行，而不是看 `memory.total`。如果热重启后解锁容量仍然存在，请彻底关机再试一次，然后才下结论：热重启不算复位。如果真正冷启动后仍然存在，请检查 initramfs 是否真的重建了：一个仍装着补丁版 `nvidia.ko` 的陈旧 initramfs 是常见原因，与安装侧同样的失败模式如出一辙。

---

## 切换分支前先卸载

维护者的原则是添加新安装之前先移除旧安装："In fact, I would always recommend to remove the old one before adding the new one." 一位克隆了 `Gen2` 分支并直接覆盖安装到现有安装之上的测试者报告说这样不行，先卸载再安装就好了。

这是建议而非铁律。至少另外两位测试者覆盖安装没有任何问题，非正式共识是大多数人"直接叠上去就完事"。失败是真实存在的但不普遍，也没人找到差异化的因素。先卸载再安装是受支持的路径：

```bash
cd /path/to/old-checkout && sudo ./remove.sh --yes
cd /path/to/new-checkout && sudo ./install.sh
sudo shutdown -h now      # cold boot
```

---

## 如果显卡是卡死状态而不是仅仅打了补丁

`remove.sh` 是针对健康系统的。如果显卡处于糟糕状态（启动失败导致 WPR2 保持开启、Booter 停在半途、`RmInitAdapter` 失败，或显卡已停止被枚举），卸载模块并不是正确的第一步。请前往 [Recovery](recovery.md)，那里涵盖通过 `/sys/bus/pci/devices/<BDF>/reset` 进行的函数级复位、`modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia` 的卸载顺序，以及只有冷启动才能清除状态的场景。

一个实际的多租户例子可以说明区别：某运营商的租客杀死了一个性能不佳的 `llama.cpp` 进程，留下大量幽灵进程，把驱动状态搞坏了。恢复需要由运营商执行主机重启，因为无法从容器内部重启这些卡。任何程度的卸载都帮不上忙。

---

## 相关页面

- [Install](install.md) 正向安装流程
- [Verify](verify.md) 健康安装应该是什么样，这样你才知道自己在移除什么
- [Troubleshooting](troubleshooting.md) 与 [Recovery](recovery.md)
- [Multi-GPU](multi-gpu.md)，其分支安装器会添加 `master` 的 `remove.sh` 不知道的文件
