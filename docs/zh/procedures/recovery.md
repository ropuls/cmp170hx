# 恢复（Recovery）

**本页涵盖：** 当 CMP 170HX 无法初始化时如何把它恢复到可用状态：从最便宜到最彻底的复位阶梯、冷启动做了而热重启没做的事、FLR 清除什么而只有次级总线复位（SBR）能清除什么、每一种复位后究竟哪些寄存器状态会存活、如何移除补丁模块并恢复原版系统，以及关于真实变砖风险有多大的诚实评估。

**简短回答。** 解锁只是写寄存器。它不烧熔丝、不写 VBIOS、不写 EEPROM，而且（在正式发布工具上）不写任何固件文件。它触碰的每一个几何寄存器都是易失的，断电即还原，这意味着**一张起不来的卡几乎总能通过断电恢复**。整个语料中从未确认过任何永久变砖。一份第一手报告与"无持久状态"模型相矛盾，已在下方 [Bricking risk](#bricking) 中如实记录；它至今无法解释。

如果你只有一分钟：关机，关掉 PSU 开关或拔掉电源，等 60 秒，再开机。这一个动作能解决绝大多数卡死。

---

## 1. 复位阶梯 { #reset-ladder }

从上往下走这份清单。每一级都比上一级更具破坏性，而且只有当上面一级失败时才值得尝试。

| # | 级别 | 命令 | 清除什么 |
|---|---|---|---|
| 1 | 重新加载驱动 | `modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia && modprobe nvidia` | 仅驱动侧软件状态 |
| 2 | 函数级复位（FLR） | `sudo rmmod nvidia_uvm nvidia`; `echo 1 \| sudo tee /sys/bus/pci/devices/$BDF/reset` | 引擎、WPR2、PL0 scratch、SEC2 reset-PLM 污染、Falcon IMEM 内容；**重新采样熔丝** |
| 3 | PCI 分离与重新扫描 | 见 [3.4](#detach-rescan) | BAR0 读 `0xffffffff` 的离总线卡；之后需要恢复总线主控 |
| 4 | 次级总线复位（SBR） | 在上游桥接器上发起 | FLR 清除的一切，**加上**常开（AON / PGC6 "GC6 island"）电源域 |
| 5 | 完全断电冷启动 | 关 PSU 开关或拔线，等 60 秒 | 一切，包括电容保持的状态 |
| 6 | 物理移除显卡 | 取出放置一段时间 | 最后手段；在一个无法解释的案例中用过一次 |

第 2 级完整命令：

```bash
sudo rmmod nvidia_uvm nvidia
echo 1 | sudo tee /sys/bus/pci/devices/$BDF/reset
```

第 5 级在实践中是主要的恢复工具。记录在案的第一手建议很直白："lots of cold boots required from the way this gets wedged"，并建议关机状态下拔掉显卡的 PCIe 供电。

**为什么阶梯里既有 FLR 又有 SBR。** FLR 复位引擎但**不**复位 AON 岛，所以根因在 AON scratch 的 `0x65` 卡死能扛过 FLR，而 SBR 能清除它。见 [FLR versus SBR](#flr-vs-sbr)。

*（置信度：阶梯本身高；AON 机制中等。）*

---

## 2. 冷启动与热重启 { #cold-boot }

**热重启不是复位。** 它会留下 WPR2 保持开启、板载电容保持充电。每一份发布流程中失败的解锁后"重启"都指**冷**重启，而且这个区别是承重的：同一个症状扛过五次 `reboot` 循环，常常在第一次真正断电后消失。

### 流程 { #cold-boot-procedure }

```bash
sudo systemctl poweroff
```

1. 等待完全关机。SSH 断开、风扇停转。
2. 关掉 PSU 开关，或拔掉电源线。
3. **等待 60 秒**，让电容放电、WPR2 复位。
4. 重新插上并开机。

60 秒等待在发布指南的每个阶段都被规定为硬性要求，它与正式发布补丁的 WPR2 保存-恢复行为一致。PLM 未打开和显存仍是 8 GB 两种症状都采用同样的流程。

一些卡在冷启动后还需要物理拔插 PCIe 供电线才正常。那是记录在案的不顺，不是已解释的机制。

### 什么时候冷启动是强制而非建议 { #cold-boot-required }

* 首次启动未解锁之后。明确地说，操作系统重启不够。
* 出现 `[WARN] Modules installed but the running driver is still stock` 之后。
* `GSP didn't boot` / 状态 `0x65` 失败之后。一位测试者确认仅删除旧内核模块不够。
* 首次安装补丁模块之后（仅分支的 `verify.sh` 在自己的失败字符串里就说了："Cold reboot if modules were just installed."）。
* 在过度配置的 80 GB 配置的多次尝试之间，一位测试者必须对整个系统做冷循环，而不只是重载驱动。

---

## 3. 复位详解 { #resets }

### 3.1 函数级复位（FLR） { #flr }

```bash
echo 1 | sudo tee /sys/bus/pci/devices/$BDF/reset
sleep 3
```

FLR 会清除 PL0 scratch 写入并**重新采样熔丝**。普通 `modprobe` 重载不清除配置空间写入；FLR 会。这一点用 `0x14A0` 的 PL0 scratchpad 作为代理测试过，并得到漏洞利用原作者证实。

**成功的 FLR 确实清除 WPR2。** `WPR2_LO` / `WPR2_HI` / GSP mailbox 的分阶段测量：

| 阶段 | WPR2_LO | WPR2_HI | GSP mailbox |
|---|---|---|---|
| 冷启动（WPR2 禁用） | `0x1FFFFE00` | `0x00000000` | `0x00000000` |
| ROP 触发后（Booter 设置好） | `0x01F77000` | `0x01FFEE00` | `0x8FAE1000` |
| FLR 之后 | `0x1FFFFE00` | `0x00000000` | `0x00000000` |
| 之后加载原版驱动 | `nvidia-smi` 正常，8192 MiB | | |

FLR 还清除 SEC2 reset-PLM 污染：`0x8f` 变回 `0xff`。

**FLR 把 Booter 从 Falcon IMEM 中移除。** FLR 后写入 DMEM 出现 `EXCI 0x0a (MISS_INS)` 就是这个意思：Booter 不再驻留，因为 FLR 把它移除了。

### 3.2 次级总线复位（SBR） { #flr-vs-sbr }

SBR 在上游桥接器上发起，而不是在设备上，它会掉电并重新初始化 FLR 不去碰的常开电源域。

**为什么 FLR 有时无法恢复 `0x65` 卡死。** `SECURE_SCRATCH_14`（`0x001180f8`）位于 PGC6 "GC6 island" 常开电源域中，标记为 RW-4R（priv-masked）。AON scratch 能扛过引擎复位和 FLR，所以未 DONE 的交接加上毒化的 PLM 和特权状态（正是它们让 Booter 自己对 `0x1180f8` 的 DIO 读返回 `0xdead5ec1`）会一路穿过 FLR 存活。SBR 掉电并重新初始化 AON 电源域，清除 scratch，让新的 Booter 能运行第 3 阶段并自己设置 DONE。

*（置信度：中等。"FLR 清不掉、SBR 能清掉"的经验模式被反复观察到；附带的 AON / GC6 描述未经验证。）*

### 3.3 每种复位会留下什么 { #state-persistence }

这张表是本页的核心。它同时解释了为什么解锁不持久，以及为什么卡很难被永久损坏。

| 状态 | 扛过 `modprobe` 重载？ | 扛过 FLR？ | 扛过 SBR？ | 扛过断电？ |
|---|---|---|---|---|
| SS0 `0x0082381c` = `0x88888888` | 是 | **是**（AON） | 未证实 | 否 |
| SS1 `0x00823820` = `0x00000008` | 是 | **是**（AON） | 未证实 | 否 |
| `FEAT_OVR_PLM` `0x00823804` | 是 | **是**（AON） | 未证实 | 否 |
| CFG1 `0x009a0204` | 是 | 否 | 否 | 否 |
| 各 FBPA 的 CFG1（`0x00900204 + n*0x4000`） | 是 | 否 | 否 | 否 |
| CSTATUS_RAMAMOUNT | 是 | 否 | 否 | 否 |
| MMU LMR `0x00100ce0` | 是 | 否 | 否 | 否 |
| FB 几何 PLM | 是 | 否 | 否 | 否 |
| AON LMR 影子 `0x001180f0` | 是 | 否 | 否 | 否 |
| WPR2 边界 `0x001fa824` / `0x001fa828` | 是 | **否**，复位为 `0x1FFFFE00` / `0x0` | 否 | 否 |
| SEC2 reset-PLM 污染（`0x8f`） | 是 | **否**，回到 `0xff` | 否 | 否 |
| `SECURE_SCRATCH_14` `0x001180f8`（AON） | 是 | **是** | **否** | 否 |
| Falcon IMEM 内容（Booter 驻留） | 是 | 否（`EXCI 0x0a`） | 否 | 否 |
| PL0 scratch（代理 `0x14A0`） | 是 | 否 | 否 | 否 |
| PCI `COMMAND.BusMaster` | 被 `rmmod nvidia` 清除 | 复位为默认 | 复位 | 复位 |
| 熔丝 | 是 | **重新采样**，值不变 | 重新采样 | 重新采样 |
| VBIOS、EEPROM、磁盘固件 | 是 | 是 | 是 | **是** |

**两个直接推论。**

*算力先于显存发布，正是因为 FLR 的不对称。* SS0、SS1 和功能覆盖 PLM 位于常开岛，扛得过 FLR；整个显存几何扛不过。这就是为什么旧 FLR 管线能在复位后保住算力解锁，却每次都丢掉几何。这也是为什么正式发布补丁在**一次** GSP 启动内打开 PLM 并写入几何，中间没有任何复位。参见 [Memory geometry](../unlock/memory-geometry.md) 和 [Compute throttle](../unlock/compute-throttle.md)。

*解锁写入的任何东西都扛不过断电。* 一张断电的卡回来就是出厂状态。这是下面变砖评估背后唯一最重要的事实。

### 3.4 显卡掉出总线 { #detach-rescan }

如果 BAR0 读 `0xffffffff`，卡已经离总线。分离并重新扫描，然后恢复总线主控：

```bash
echo 1 | sudo tee /sys/bus/pci/devices/$BDF/remove
echo 1 | sudo tee /sys/bus/pci/rescan
sudo setpci -s ${BDF#0000:} COMMAND=0x0546
```

`setpci` 这一步很重要。`0x0546` 设置了第 2 位（Bus Master）；`0x0102` 没有，总线主控关闭的卡在独立工具向它开火时会静默地什么都不做，日志里任何地方都没有 DMA 错误。完整失败模式见 [Bus mastering cleared](troubleshooting.md#bus-master)。

如果卡完全不再出现，而且冷启动后也从不出现，原因可能是硬件而非状态。完全诊断过的板级故障（一颗失效的 GS7155NVTD LDO 短路 `PS_5V_PGOOD`）及其维修见 [The card dropped off the PCIe bus](troubleshooting.md#off-bus)。

### 3.5 复位前拆除驱动 { #teardown }

驱动还持有设备时 FLR 不可靠。可用的拆除顺序是：

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

nvidia 模块经常无论如何都拒绝卸载，留下 `nvidia 15835136 2`，`drm` 被包括 `i915` 在内的七个用户持有。这条依赖链是为什么解锁工作要在无头主机或不使用 NVIDIA 显示的主机上做的实际原因。

### 3.6 一启动就卡死的卡 { #boot-pre-wedged }

如果机器无法关机，或者卡在你来得及干预之前又卡死了，那是因为驱动在自动加载并重新卡死它。断开显卡启动，或从引导加载器命令行黑名单屏蔽模块，然后清理：

```text
# GRUB kernel command line, one boot
modprobe.blacklist=nvidia,nvidia_uvm,nvidia_drm,nvidia_modeset
```

一张 10 GB 卡曾陷入不可中断睡眠状态，扛过了大约五次冷重启，还阻止 Ubuntu 关机。**原因是自动加载的补丁内核驱动，不是卡。** 断开显卡启动并清理后解决。
*（置信度：中等；根因由受影响的测试者在恢复后确认。）*

---

## 4. 移除补丁模块 { #remove }

### 4.1 受支持的路径 { #remove-sh }

```bash
sudo ./remove.sh --yes
```

`remove.sh` 拒绝在缺少 `--yes` 或 `-y` 的情况下运行。它执行五步，写入 `logs/remove_YYYYMMDD_HHMMSS.log`，如果仓库目录不可写则回退到 `/tmp`。它做的事：

* 停止显示管理器，如果 `modprobe -r` 失败则强制 `rmmod`。
* 停止、禁用并删除遗留的 `/etc/systemd/system/cmpunlocker.service`，杀掉任何残留的 `/opt/cmpunlocker/daemon/watchdog.py` 进程。两者都是一个废弃看门狗设计的遗迹，当前安装器从不创建它们。
* **在每个内核上**删除 `/lib/modules/*/updates/cmpunlocker/`，逐内核运行 `depmod -a`。
* 删除遗留的 `/opt/cmpunlocker` 安装目录。
* 删除固件补丁时代的遗留物：对每个 `/lib/firmware/nvidia/*/gsp_tu10x.bin` 移除 `.cmpunlocker.bak`、`.cmpunlocker.patched`、`.cmpunlocker.tmp`、`.cmpunlocker.cleanup` 和 `.cmpunlocker.pat`。
* 重建 initramfs 并重新加载原版模块。

> [!CAUTION]
> **没有 `uninstall.sh`**
>
> `docs` 分支上的文档引用了 `sudo ./uninstall.sh --yes`。**这样的脚本不存在**，无论 master 还是 docs 分支本身。正确的命令是 `sudo ./remove.sh --yes`，而且那个分支自己的 `ARCHITECTURE.md` 也是这么说的。

`master` 的 `remove.sh` **不**碰内核命令行。IOMMU 配置及其撤销存在于 `Gen2`、`far` 和 `deced` 分支上，那里的 `remove.sh` 恢复 `*.cmpunlocker.bak` 并打印 `Reverted IOMMU kernel parameters (effective after reboot)` 或 `No IOMMU config backup found, kernel command line left as-is`。

最后冷重启。然后确认原版模块回来了：

```bash
cat /proc/driver/nvidia/version          # should show a dvs-builder release build
sudo dmesg | grep SEC2_DEBUG             # should print nothing
nvidia-smi                               # 8192 MiB or 10240 MiB
```

参见 [Uninstall](uninstall.md)。

### 4.2 三级手动回滚 { #rollback-tiers }

每一级都以冷重启结束。只有前一级未能恢复可用的原版栈时才升级。

**第 1 级：手工撤销模块安装。**

```bash
sudo systemctl stop nvidia-persistenced
sudo modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia
sudo rm -rf /lib/modules/$(uname -r)/updates/cmpunlocker/
sudo depmod -a
# restore the backed-up stock nvidia.ko, then:
sudo apt install --reinstall nvidia-driver-610-open
```

**第 2 级：** `sudo ./remove.sh --yes`。

**第 3 级：** 完全移除 610 栈并安装 580。

### 4.3 撤销之前的实验 { #undo-experiments }

用于解锁开发的机器会积累会静默破坏干净安装的状态。重装前，全部撤销：

* 删除 `/etc/modprobe.d/blacklist-nvidia-manual.conf`。
* 移除任何 dpkg diverts。
* 恢复 `nvidia-lib-bak`。
* 从 `/lib/firmware/nvidia/610.43.03/` 和 `/lib/firmware/nvidia/580.173.02/` 下的 `.stock` / `.backup` / `.bak` 副本恢复原版 `gsp_tu10x.bin`。

> [!CAUTION]
> **陈旧的补丁版 `gsp_tu10x.bin` 会毒化驱动内解锁**
>
> 如果这台机器曾用过固件补丁前代方案，在运行驱动内补丁之前恢复原版 blob：
>
> ```bash
> GSP_DIR=/lib/firmware/nvidia/610.43.03
> sudo cp $GSP_DIR/gsp_tu10x.bin.cmpunlocker.bak $GSP_DIR/gsp_tu10x.bin
> ```
>
> 驱动在启动期间会把固件的签名保存为 "stock"。如果固件仍是补丁版，它保存的会是**漏洞利用负载**的签名，干净的 GSP-RM 启动随后会 DMA 错误的 ROP 链。之后要找的成功行是 `SEC2_DEBUG: saved stock signature (4096 bytes)`。

### 4.4 恢复版本匹配的固件目录 { #restore-firmware }

一个被删除或错配的 `/lib/firmware/nvidia/<version>/` 目录会产生看起来像硬件逐渐劣化的失败。一次多日无法复现的"模型劣化"最终发现是 `/lib/firmware/nvidia/580.159.03/{gsp_tu10x.bin, booter_*.bin}` 被删除，加上 `.04` 用户空间 `nvidia-smi` 在 `.03` 模块上不会触发 GPU 初始化。那种状态下 `SEC2 MBOX0 = 0x0` 意味着 Booter 根本没有加载。恢复版本匹配的固件目录后立刻复现了之前的工作状态。

保留任何驱动改动的 diff 或变更日志。重装全新驱动会静默丢弃每一个需要的注入。

### 4.5 回到一张全新、从未动过的原版卡 { #restore-stock }

硬件中没有任何需要撤销的东西。解锁是叠在主线 NVIDIA 驱动之上的补丁，不是固件替换，所以一旦补丁模块消失、机器断电重启过，卡就与你开始时的卡逐位相同：CFG1 `0x02449000`、LMR `0x00000208` 或 `0x00000288`、每个 FBPA 的 `CSTATUS_RAMAMOUNT` `0x200`、SS0 回到锁定值。

卡还能在**原版** Linux NVIDIA 驱动上完全无补丁地运行（Ubuntu 24.04 上 `nvidia-driver-570` 加 CUDA 12.8 开箱即用，Ubuntu 22.04 上的 `nvidia-driver-535-server` 也被报告可用），报告为 `NVIDIA Graphics Device`，计算能力 8.0。能驱动和解锁是两回事，确认前者是证明卡挺过了你做的任何事的好方法。

---

## 5. 真实变砖风险有多大？ { #bricking }

### 5.1 证据 { #bricking-evidence }

**整个语料中从未确认过任何永久变砖。** 具体说法及其处理：

| 说法 | 处理 |
|---|---|
| 一张卡在净室（cleanroom）工作中变砖 | 一个 LLM agent 的误判，因为它忘了这些卡可以复位 |
| "CMP 170HX 卡会被解锁或被 NVIDIA 毒化驱动变砖" | 不存在第一手报告。被引用的具体公开案例经评估是一张被推到 80 GB 的 10 GB 卡 |
| 主板 PCH 故障由 170HX 测试引起 | 原帖用了 "coincidentally" 这个词；没有建立任何因果机制。仅作为风险轶事记录 |
| 2026 年 7 月下旬涨价期间卖家描述"一批有缺陷" | 被用作对曾显示正常卡的列表的取消借口。在记录在案的案例中，没有任何有缺陷的卡被发货或被诊断 |

结构性论证比"没有报告"更强。正式发布的解锁：

* 只写易失寄存器，全部在断电后还原；
* 不烧任何熔丝（master kill 熔丝 `0x008203f0` 读为 `0x00000000`，从不会被写）；
* 不刷 VBIOS 或任何 EEPROM；
* 在正式发布工具上，不修改 `/lib/firmware` 下的任何文件（更早的固件补丁一代会，所以 `remove.sh` 仍会清理它）；
* 被 `remove.sh` 加一次断电完全还原。

### 5.2 唯一一条对不上的报告 { #bricking-contradiction }

> [!NOTE]
> **未解决问题**
>
> 一份第一手描述：一张 10 GB 卡卡死，三个 D 状态线程卡住，FLR、SBR、PCI 分离重连**甚至完全 PSU 断电冷启动**都清不掉："when I rebooted, the registers were still written, and the D-threads were still there... card booted pre-wedged"。最终恢复需要关掉电源开关、按住电源键（去掉条带？）并物理取出显卡数小时。
>
> 这一观察在频道里被质疑，至今无法解释。它**与**"改装不改变任何持久状态"这一其余证据充分支持的模型**矛盾**，这正是为什么值得解决而不是打发掉。当时提出的原因是"在无驱动负载投递后为 cpu rm 启动打过补丁的专有 blob"。同一份记录里还有另外两次卡死：一次由 agent 写入 `FUSE_SS_PLM` 风格寄存器引起（需要完全断电循环），一次来自"在 `0x10b9` 处输入 `0x10aa`"，硬变砖了一个测试台。
>
> 什么能定论：在这样的冷启动后立刻新鲜抓取寄存器值，并附一张电源状态的照片。

在那之前，诚实的表述是：**模型说断电循环总能赢，而且在每个可复现的案例里它也赢了，但一位可信的运营者报告了一个扛过一次的状态。**

### 5.3 真实风险到底是什么 { #real-risks }

离这项工作最近的人点名的残余风险并不稀奇：

1. **普通的二手硬件故障。** 这些是前矿卡。语料中唯一完全诊断的"看似永久"的故障是板上 3.3 V LDO 失效，与解锁完全无关，而且可以在元件级修复。见 [Card off the bus](troubleshooting.md#off-bus)。
2. **补丁集正在积极变化。** 破坏安装的是分支变动，不是硅片。
3. **HBM 的热损伤。** 长期欠冷却会劣化 HBM。记录中烤机失败的那张卡在 85 °C 下带显存超频运行；无错误的卡保持在 73 °C 以下。*（置信度：中等；不存在失效率数据。）* 参见 [Thermals](../hardware/thermals.md)。
4. **让卡跑在稳定几何之外。** 8 GB 卡在 64 GB 稳定且在生产中；10 GB 卡在 40 GB 稳定；10 GB 卡在 80 GB 能报告容量，但超过约 40 GB 就不可用：触碰更多内容的内核会导致致命 GPU 丢失，与功耗上限无关。报告的错误码包括 Xid 31（被描述为无害）和 CUDA 内存测试后的 Xid 154；主要报告症状是卡死，以及烤机错误。Xid 31 的说法来自一位旁观者，并未被拥有故障卡的运营者证实为*那个*特征错误。这毁的是工作负载，不是卡。参见 [80 GB](../frontier/80gb.md)。
5. **对活动任务的操作失误。** 对活动的多 GPU 任务 `kill -9` 会卡死主机 CUDA 运行时（约 32 个僵尸进程，`cuInit` 返回 999），需要主机重启。对验证内核 SIGKILL 可能以 Xid 45 卡死卡。在一个完整的 8 卡会话、数百个 60 秒健康采样中，工作负载正确驱动时硬故障数为 **0**。

> [!CAUTION]
> **真正的硬件风险在哪里**
>
> 这个项目里唯一有真正不可逆硬件风险的地方是**电容改装**：在一张 8 到 12 层板上、C1100 到 C1350 范围内手工焊接 24 颗 0402 220 nF X7R 元件，芯片要 420 °C 热风 2 分钟才能抬起。那是焊接风险，不是固件风险，单独在 [Physical mods](../operations/physical-mods.md) 中覆盖。另请注意电容改装只改通道**数量**。它永远不会改变 PCIe 代数。

### 5.4 实用的安全姿态 { #posture }

* 保持卡的原版行为可验证：开始前就知道它在未打补丁的原版驱动上能枚举并运行。
* 优先使用无头或不使用 NVIDIA 显示的主机，这样模块才能真正卸载。
* 可行时在虚拟机或容器里做解锁开发。一位开发者报告裸机上每次部署砸了的 `nvidia.ko` 后都需要重装操作系统。
* 把驱动固定在 610 作为长期预防措施，防止未来 NVIDIA 版本封堵漏洞，就像 P100 和 V100 用户固定在 580 附近一样。*（置信度：中等；有理由的建议，尚未需要，因为不存在会封堵的驱动。）*
* 不要对活动任务 `kill -9`。不要在你珍惜的硬件上跑过度配置的几何。
* 求助前抓取 `sudo dmesg | grep SEC2_DEBUG` 和最新的安装日志。见 [Escalation](troubleshooting.md#escalation)。

---

## 相关页面

* [Troubleshooting](troubleshooting.md)：从症状到原因到修复，索引式
* [Uninstall](uninstall.md)：`remove.sh` 完整版
* [Install](install.md)：受支持的安装流程
* [Verify](verify.md)：确认良好状态
* [Risks](../start/risks.md)：此评估的方向级版本
* [Privilege level masks](../unlock/privilege-level-masks.md)：哪些 PLM 是 AON，哪些不是
* [Memory geometry](../unlock/memory-geometry.md)：为什么几何扛不过复位
* [Register reference](../unlock/register-reference.md)：本页点名的每一个寄存器
