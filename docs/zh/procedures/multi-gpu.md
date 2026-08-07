# 运行多张卡

**本页涵盖：** 当一台主机有不止一张 CMP 170HX 时会发生什么：为什么正式发布的 `master` 明明是单卡安装器却能在多卡机器上工作、未发布的 `multiple-cards` 分支增加了什么（按 BDF 分类、`gpu_inventory` 文件、`mixed` 配置和 `SKIP_GEOMETRY_REWRITE`），以及只有机箱里有两张或更多 GPU 才会出现的失败模式。

先把关键结果说在前面：**解锁本身已经是逐 GPU 的。** 自提交 `7fe49b6` 起，打过补丁的 `nvidia.ko` 同时携带两种几何，并在 GSP 启动时从 `pGpu->idInfo.PCIDeviceID >> 16` 选择其一，所以机器里的每张卡都会以正确的容量独立解锁，无论安装器怎么想。`master` 上单卡的只是*安装器*的记账：它取第一条匹配的 `lspci` 行、从一次 `nvidia-smi` 读数猜一个配置、写一组元数据文件。`multiple-cards` 和 `Gen2` 分支用真正的逐设备清单取代了这种记账。

多 GPU 操作在实践中已被确认可用：一位运营者在 Proxmox 下直通了八张 8 GB CMP 170 卡，全部解锁。针对六卡机器的早期建议是先试 `master`，后来一位多 GPU 用户确认 master 工作良好。

---

## `master` 在多卡主机上会做什么

```bash
lspci -nn | grep -iE '10de:20b0|10de:20c2|10de:2082' | head -1
```

这个 `head -1` 就是全部故事。`install.sh` 记录一个 BDF 和一个设备 ID，然后调用 `detect_card_profile()`，它读取 `nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits | head -1`，再次取第一条，这次是 *nvidia-smi* 顺序而不是 *lspci* 顺序。两种顺序并不保证一致。

当前 `master` 上的后果：

| 场景 | 结果 |
|---|---|
| 4x 8 GB 卡 | 能用。四张全部解锁到 65536 MiB。配置元数据显示 `8gb`，碰巧是对的 |
| 4x 10 GB 卡 | 能用。四张全部解锁到 40960 MiB |
| 混合 8 GB + 10 GB | **每张卡的几何仍然正确**，因为它是 GSP 启动时按设备 ID 选择的。只有 `card_profile` / `unlock_geometry` 对抛硬币输掉的那种类型是错的 |
| CMP 卡加一张无关的 NVIDIA GPU | 配置可能从*另一张*卡检测。除非那张卡的容量落在全部四个检测窗口之外导致安装**终止**，否则仍然只是元数据错误 |
| 存在 `10de:20b0` 卡 | 只有当它是*第一条*匹配的 `lspci` 行时才被警告；如果它排在 `20c2` 或 `2082` 卡后面，`head -1` 会把它完全藏起来，不打印任何警告。无论哪种情况它都不会被解锁，因为驱动内门控只接受 `0x20C2` 和 `0x2082` |

> [!WARNING]
> **混合 GPU 主机上务必传 `--profile`**
>
> 一台同时有 RTX 3080 10 GB 和 8 GB CMP 170HX 的主机，至少有两个人复现过从 3080 检测出 "10GB" 并选择 10 GB 配置。另一份报告把另一个 CMP SKU（50HX）误检为 10 GB 170HX。在当前 `master` 上这只会弄错元数据文件，但显式传 `--profile=8gb` 或 `--profile=10gb` 的习惯不花任何代价，还能消除一整类令人困惑的输出。

---

## `multiple-cards` 分支

> [!WARNING]
> **实验性：未发布分支**
>
> `multiple-cards`（尖端 `b1cb6d8` "Added support for multiple cards"，提交于 2026-07-18，宣布于 2026-07-19）截至尖端 `cc872cb`（2026-07-23）**尚未**合并进 `master`。同样的安装器也通过提交 `2f27474` "Gen2 + multiple-card support" 折叠进了 `Gen2` 血统。本节的每一样东西都是分支代码。

### 按 BDF 分类

`detect_card_profile()` 被 `profile_from_devid()` 取代：

```bash
profile_from_devid() {
    case "$1" in
        20c2) echo "8gb" ;;
        2082) echo "10gb" ;;
        *) echo "unsupported" ;;
    esac
}

expected_mib_for_profile() {
    case "$1" in
        8gb) echo "65536" ;;
        10gb) echo "40960" ;;
        *) echo "" ;;
    esac
}
```

然后安装器遍历 `lspci -nn | grep -iE '10de:20b0|10de:20c2|10de:2082'` 的**每一**行（用 `mapfile`，不是 `head -1`），构建五个并行数组：BDF、设备 ID、配置、期望 MiB、当前 MiB。当前 MiB 来自一次缓存的 `nvidia-smi --query-gpu=pci.bus_id,memory.total --format=csv,noheader,nounits` 查询，按总线 ID 而不是索引匹配。

总线 ID 通过共享的 `normalize_bus_id()` 比较，它把短格式 `BB:DD.F` 转小写并展开为 `0000:BB:DD.F`，所以 `lspci` 和 `nvidia-smi` 的拼写可以比较相等。同一个函数逐字存在于 `verify.sh` 中。

`20b0` 卡在这里被分类为 `unsupported` 并**跳过**，提示 `GPU <bdf> (10de:20b0), unlock path not gated for this ID; skipping`，这与 master 有行为差异（master 会警告并继续以该卡为选中对象）。如果检测到的所有卡都不受支持，分支安装器会以 `No unlockable CMP 170HX GPUs found (need 10de:20c2 and/or 10de:2082)` 终止。

典型的第 2 步输出：

```text
✓ GPU 0000:0b:00.0 (10de:20c2) → 8gb (current 8192 MiB, expect ~65536 MiB unlocked)
✓ GPU 0000:0c:00.0 (10de:20c2) → 8gb (current 8192 MiB, expect ~65536 MiB unlocked)
✓ GPU 0000:0d:00.0 (10de:2082) → 10gb (current 10240 MiB, expect ~40960 MiB unlocked)
==> Inventory: 3 unlockable (2× 8gb, 1× 10gb)
```

### `mixed` 配置

当 `COUNT_8GB > 0` 且 `COUNT_10GB > 0` 时，`CARD_PROFILE` 变成第三个值 `mixed`：

```text
✓ Mixed variants detected → profile mixed (runtime geometry by PCI ID)
==> Unlock geometry: 64GB for 20c2 / 40GB for 2082 (chosen at GSP boot per GPU)
```

在混合清单上，`--profile=` 覆盖会被**明确丢弃**，提示 `--profile=8gb ignored for mixed inventory; card_profile stays mixed (each card unlocks by PCI ID)`。在同质清单上覆盖会被尊重，但会警告它只是元数据。分支的帮助文本把这种降级说得很清楚：`Force 8GB metadata label (geometry is still chosen per PCI ID)`。

### `SKIP_GEOMETRY_REWRITE`

分支上的 `driver/build.sh` 增加了一个 case 和一个保护标志：

```bash
SKIP_GEOMETRY_REWRITE=0
case "${PROFILE}" in
    8gb|8GB)   CFG1="0x02779000"; LMR="0x0000020B"; FB_BYTES="0x0000001000000000"; UNLOCK_LABEL="64GB" ;;
    10gb|10GB) CFG1="0x02669000"; LMR="0x0000028A"; FB_BYTES="0x0000000A00000000"; UNLOCK_LABEL="40GB" ;;
    mixed|MIXED)
        PROFILE="mixed"
        CFG1="0x02779000"; LMR="0x0000020B"; FB_BYTES="0x0000001000000000"
        UNLOCK_LABEL="mixed"
        SKIP_GEOMETRY_REWRITE=1
        ;;
esac

if [[ "${SKIP_GEOMETRY_REWRITE}" -eq 1 ]]; then
    info "mixed profile: runtime device-id geometry (no build-time CFG1/LMR rewrite)"
else
    python3 - ... <<'PY'
    ...
fi
```

这段代码里有两件事值得注意：

1. 在 `mixed` 模式下 `CFG1` / `LMR` / `FB_BYTES` 变量仍然被赋值为 **8 GB** 的值，只是从未被使用。如果去掉标志*并且*重写步骤可达，它们就是混合主机原本会给每张卡烤进去的值；第 2 点解释了为什么它不可达。
2. `SKIP_GEOMETRY_REWRITE` 是现有安全网之上加的双保险。它跳过的内联 Python 步骤本就以六标记检查开始，检查两种烤入的几何，并以 `runtime device-id geometry (profile metadata=<label>)` 退出且不修改任何东西。在任何 `7fe49b6` 后代树上，这个重写无论如何都是空操作。只有有人重新引入单 SKU 补丁时，这个标志才有意义。

在该模式下，`unlock_geometry` 会写成字面量字符串 `mixed`，`card_profile` 写成 `mixed`。

### `gpu_inventory` 文件

`install.sh` 导出 `CMPUNLOCKER_GPU_INVENTORY`，`build.sh` 把它持久化到 `/lib/modules/$(uname -r)/updates/cmpunlocker/gpu_inventory`，每张可解锁 GPU 一行、空白分隔：

```text
BDF              devid  profile  expected_mib
0000:0b:00.0     20c2   8gb      65536
0000:0c:00.0     20c2   8gb      65536
0000:0d:00.0     2082   10gb     40960
```

真实文件没有表头行；上面的列是为可读性标注的。如果该变量为空，`build.sh` 会把文件截断为零字节，而不是留下陈旧文件。

与其他三个元数据文件一样，**内核模块中没有任何代码读取它。** 它唯一的消费者是 `verify.sh`，它优先于实时的 `lspci` 枚举使用它，这样一张掉出总线的卡会被报告为 `MISSING`，而不是从检查中静默消失。

### 多卡机器上的 `verify.sh`

```bash
sudo ./verify.sh
```

如果 `gpu_inventory` 可读且非空则从中枚举，否则回退到 `lspci -nn | grep -iE '10de:20c2|10de:2082'`。对每张 GPU 按窗口 `>= 60000` MiB（8gb）和 `35000`-`59999` MiB（10gb）打印 `OK`、`STOCK`、`MISSING` 或 `UNEXPECTED`，然后总结：

```text
✓ All 3 unlockable GPU(s) report unlocked memory
```

或以 `<n> GPU(s) failed unlock verification. Cold reboot if modules were just installed.` 失败。完整细节，包括它不检查的两件事，见 [Verify](verify.md#verifysh)。

---

## 已知的多卡失败模式

### 1. depmod 静默地只挑一个 `nvidia.ko`

本页最有价值的一条。打过补丁的和原版的 `nvidia.ko` 可能同时落在唯一的 `updates` depmod 搜索条目下，此时 **depmod 会随意挑选一个，并静默丢弃另一个**。一位测试者把一次多 GPU 失败精确根因定位到这一点，只在 updates 搜索路径中保留 cmpunlocker 变体，重启，然后确认多 GPU 正常工作。

这与 `build.sh` 警告的 `srcversion` 不匹配属于同一类失败：正在运行的模块不是补丁版，所以没有卡会解锁。用以下方式诊断：

```bash
modprobe -n -v nvidia | awk '/insmod/ {print $2; exit}'
find /lib/modules/$(uname -r)/updates -name 'nvidia.ko'
cat /sys/module/nvidia/srcversion
modinfo -F srcversion /lib/modules/$(uname -r)/updates/cmpunlocker/nvidia.ko
```

第一条命令中任何不在 `updates/cmpunlocker/` 下的东西，或第二条命令中多于一个 `nvidia.ko`，就是这个 bug。

### 2. 陈旧的 initramfs 完全压过 depmod

与单卡情况相同，但在机器上更糟，因为部分结果看起来像逐卡问题而不是模块加载问题。`build.sh` 会自己重建 initramfs，做不到时警告 `No initramfs tool found, rebuild manually before rebooting`。如果同一启动中某些卡解锁而另一些没有，这*不是*原因；如果*全部*都没有，那很可能就是。

### 3. 从错误的 GPU 误检配置

上文已述。在 `master` 上这只是元数据错误，除非它导致安装终止。

### 4. `verify.sh` 的 lspci 回退会丢掉 `0x20B0`

`install.sh` grep `10de:20b0|10de:20c2|10de:2082` 然后警告或跳过 `20b0`，但 `verify.sh` 的回退路径只 grep `10de:20c2|10de:2082`。含 `20b0` 卡的机器在安装器和验证器中会显示不同的设备数量。无害，但令人困惑。

### 5. 虚拟化限制

- **Proxmox 直通可用**，对显存和算力都是：八张 8 GB 卡直通后全部解锁。
- **使用 SeaBIOS，不要用 UEFI/OVMF。** UEFI 会产生 RM 初始化和适配器失败，伪装成漏洞利用根本没生效。这是第一手根因定位的，并立即被第二个人证实——他的"无法复现"最后发现是同一原因。
- **截至 2026-07-24，PCIe Gen2 链路训练在虚拟机中不工作**，维护者承认这是一个待调试的开放项。

### 6. 多租户使用中的主机级卡死

一位运营者的租客杀死了一个性能不佳的 `llama.cpp` 运行（约 121 t/s），留下的幽灵进程破坏了驱动状态。恢复需要运营者执行主机重启，因为无法从 Docker 容器内部重启这些卡。在任何出租机器上都应规划带外重启通道。参见 [Recovery](recovery.md)。

### 7. 是互连问题，不是安装器问题

好几份"多卡很慢"的报告是链路带宽问题，不是解锁问题：

- 每张卡默认都是第 1 代 x4。升到第 2 代是未发布分支上的软件改动；超过 x4 宽度需要焊接 AC 耦合电容。这是两个完全独立的成就。参见 [PCIe subsystem](../hardware/pcie-subsystem.md) 和 [PCIe Gen2](../unlock/pcie-gen2.md)。
- **NVLink 被熔丝关闭**，这张卡上也没有 P2P。`llama-server --split-mode row` 曾与层拆分命令一起流传，但标注为 "benchmark-only on these links"，与张量并行式拆分在第 1 代 x4 上不可行的判断一致。
- 一条常被引用的经验法则（多 GPU LLM 推理服务中"x4 有 10-30% 提升，x8 或更好才理想"）只是作为经验法则给出的，**从未在 170HX 上测量过**。请把它当作低置信度。参见 [LLM inference](../operations/llm-inference.md)。

### 8. P2P 叠加

`aikitoria` P2P 补丁可以叠加到 cmpunlocker 上，把它的 diff 作为 `0007-unlock-p2p.patch` 放进 `driver/patches/`，因为 `build.sh` 按 glob 顺序应用每一个 `*.patch`。它在纯 170HX 系统上有没有用仍未解决：一位测试者报告 "It doesn't seem to take effect on the 170HX... It only has an effect on them if there are other models of GPUs on the same machine"，而另一位测试者同一天报告在一台也带两张 RTX 3090 的机器上 P2P 加 cmpunlocker 工作正常——这正是第一份报告所说唯一有效果的混合型号场景。双方都同意 P2P 受带宽限制，在第 1 代 x4 上收益甚微。参见 [P2P](../frontier/p2p.md)。

---

## 今天为多卡机器推荐的做法

1. 安装前先盘点硬件：

   ```bash
   lspci -nn | grep -iE '10de:20b0|10de:20c2|10de:2082'
   nvidia-smi --query-gpu=pci.bus_id,name,memory.total --format=csv
   ```

   确认每张卡的设备 ID。参见 [Identify your card](../start/identify-your-card.md)。

2. 如果所有卡类型相同，从 `master` 安装并显式指定配置：

   ```bash
   sudo ./install.sh --profile=8gb     # or --profile=10gb
   ```

   对于真正的混合 8 GB + 10 GB 机器，`master` 仍然会对每张卡产生正确几何；你放弃的只是准确的元数据和 `verify.sh`。如果你需要这些，用 `multiple-cards` 分支并接受它是未发布的。

3. 冷启动。`sudo shutdown -h now`，然后开机。

4. 逐张验证**每一**张卡，而不仅仅是第一张：

   ```bash
   nvidia-smi --query-gpu=pci.bus_id,memory.total --format=csv
   sudo dmesg | grep 'POST-WRITE'      # one line per unlocked GPU, with its devId
   ```

   `POST-WRITE` 行携带 `(devId=0x...)`，所以混合 SKU 的机器应该同时显示 `CFG1=0x02779000 LMR=0x0000020b` 和 `CFG1=0x02669000 LMR=0x0000028a` 两行。

5. 如果恰好一张卡不对，怀疑那张卡（插槽、供电、转接卡）。如果全部都不对，怀疑模块加载（失败模式 1 和 2）。

---

## 合并状态

> [!NOTE]
> **未解决问题：多卡、IOMMU 和 Gen2 应该以什么顺序合并到 master？**
>
> `multiple-cards`（`b1cb6d8`，独立）和 `Gen2` 血统（把前者折叠进去）截至 `cc872cb` 都未合并。障碍是打包：整体合并 `Gen2` 会把实验性的 PCIe 链路重训练补丁（`0007-pcie-gen2.patch`、`0008-pcie-gen2-probe-retrain.patch`）及其未验证的寄存器写入拖进稳定路径。多卡安装器的改动是自包含的，可以单独 cherry-pick，而且 `mixed` 配置已经能工作，因为 master 的补丁 0001 把两种几何都烤进去了。另外，`clanker/driver-port`（580/590/595/610 支持）和 Gen2 血统是独立开发的、从未合并，所以今天选一个意味着放弃另一个。参见 [Status board](../frontier/status-board.md) 和 [Driver versions](driver-versions.md)。

---

## 相关页面

- [Install](install.md)、[Verify](verify.md)、[Uninstall](uninstall.md)
- [Troubleshooting](troubleshooting.md) 以症状优先诊断
- [Driver patches](../unlock/driver-patches.md) 让逐卡几何生效的设备 ID 门控
- [PCIe subsystem](../hardware/pcie-subsystem.md) 链路实际能承载什么
