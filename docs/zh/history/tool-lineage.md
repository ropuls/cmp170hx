# 工具谱系：用什么，什么已死

## 本页涵盖

CMP 170HX 工具存在四代，且时间上重叠，所以"更新"不一定意味着"取代"。本页追溯从 2026 年 6 月的第一次手动 BAR0 poke 到正式驱动内补丁集及其上的分支结构的每一个工具，并直说哪些路径已死，以免有人走上去。

**如果你只想要简短答案：**

- 要**解锁一张卡**，用 `cmpunlocker` `master`：`install.sh` 加 `driver/build.sh` 加六个补丁。其余一切要么是历史，要么是实验。
- 要**测量一张卡**，用第 0 代只读仪器（`probe.sh`、`pcielink.sh`、`check_fold.py`、VBIOS 转储器）。它们从未被淘汰。
- **不要**使用第 1 代中的任何 Python ROP 解锁器、任何 GSP ELF 补丁器或任何 systemd 持久化守护进程。那一代中的每个持久化机制都被替换了。
- **不要**指望在仓库里找到测量工具。`master` 没有 `tools/` 目录。`probe.sh`、`pcielink.sh`、`check_fold.py`、`cuda_dbg.py`、A100 探测套件与 `refire_chain*.py` 脚本都是作为 gist 和频道附件在带外分发的。

---

## 谱系一览

| 代 | 时期 | 工具 | 状态 |
|---|---|---|---|
| **0：只读特征化** | 2026-05-31 至今 | `probe.sh`（mmio-probe）、`z1_dump_and_parse_vbios.sh` + `z2_parse_vbios_table.py`、`pcielink.sh`、A100 `probe.py`/`sweep.sh`、`cuda_dbg.py`、`check_fold.py`、`cuda_memtest` | **现行。** 从未被淘汰。这些是测量仪器，不是解锁器 |
| **1：手动 BAR0 poke 与 Python ROP 解锁器** | 2026-06-22 至 2026-07-17 | `deploy.py --path sec2-rop`、`deploy.py --path vbios-memory`、`load_custom_bin.py`、`unlc.py`、`stack_gen.py`、`patch_gsp.py`、`payload-lnject.py`、`scan_dmem.py`、`nuke.sh`、`b.sh`、`falcon_emulator.py`、systemd `cmp170hx-unlock.service` | **已废弃。** 这里每个持久化机制都被替换了 |
| **2：正式驱动补丁（`cmpunlocker`）** | 2026-07-14 至今 | `install.sh`、`driver/build.sh`、`driver/patches/0001`-`0006`、`remove.sh`、`common/constants.yaml`、`driver/VERSION` | **现行且规范。** 这就是 `master` 上发货的东西 |
| **3：免驱动 SEC2 refire chain** | 2026-07-22 至今 | `refire_chain.py`（v1）、`refire_chain_v2.py`、`refire_chain_v6.py` | **现行但实验性。** 一条并行的、不发货的路径；不属于 `cmpunlocker` |
| **4：未合并的功能分支** | 2026-07-18 至今 | `multiple-cards`、`clanker/driver-port`、`80`、`debug-gen2` 到 `Gen2` 到 `far` 到 `deced`，以及 `docs`、`ecc`、`housekeeping`、`memory`、`PG199` | **实验性。** 位于第 2 代之上 |

---

## 第 0 代：测量仪器

它们早于解锁，为熔丝特征化而建，至今仍是正确的工具。整个项目的主导纪律源于这里：**每次写入之后，用 `probe.sh` 读回寄存器，而不是相信工具声称的成功。**

### `probe.sh`（tools/mmio-probe）

一个自包含的 bash 加内联 Python 工具，只读 mmap `/sys/bus/pci/devices/<BDF>/resource0`，dump 大约 120 到 130 个命名寄存器加上 24 次 per-FBPA 读取。它**从不写 BAR0**。

```bash
./probe.sh [pci_id]      # 默认过滤 10de:
# 输出到 ${OUTDIR:-/tmp/mmio-probe-$(date +%s)}
```

| 属性 | 值 |
|---|---|
| 访问模式 | `os.open(..., os.O_RDONLY)`、`mmap.mmap(fd, 0, access=mmap.ACCESS_READ)`、`struct.unpack_from('<I', bar, off)` |
| 输出 | `registers.json`、`lspci.txt`、`nvidia-smi.txt`、`gpu-summary.csv`、`probe.log`，打包到 `/tmp/mmio-probe-$(hostname)-YYYYmmdd-HHMMSS.tar.gz` |
| `registers.json` 键 | `targets`（名称到偏移/值/原因）、`fbpa_capacity`、`fbpa_cfg0` |
| Per-FBPA 常量 | `FBPA_BASE = 0x900000`、`FBPA_STRIDE = 0x4000`、`CSTATUS_RAM = 0x20C`、`FBPA_COUNT_TO_PROBE = 24` |
| 派生地址 | fbpa00 CSTATUS_RAMAMOUNT `0x0090020C`，fbpa01 `0x0090420C`，fbpa23 `0x0095C20C`；偏移 `0x200` 处为 CFG0；广播 CFG0 `0x009A0200`、CFG1 `0x009A0204` |
| 可选 CUDA 步骤 | 用 `nvcc -arch=sm_70 -O2` 编译并运行 `sr_dump.cu`，以 `dump_sr<<<p.multiProcessorCount, 32>>>()` 启动，按 SM 报告 `%smid`、`%warpid`、`%nsmid`、`%nwarpid`、`%lanemask_eq`，因此 **SM 数是测出来的，不是报出来的**（170HX 上为 70）。若缺少 `nvcc` 则记一行日志跳过 |

`gpu-summary.csv` 捕获 `driver_version` 与 `vbios_version`，这正是让探测结果能绑定到特定 VBIOS 的东西。该工具基于 MODS/MATS 构建，预期可移植到其他卡，但需注意"寄存器可能在不同的范围等"。

读 BAR0 需要 root **外加** `CAP_SYS_RAWIO`，容器化 GPU 主机通常会丢弃该能力；此时探测会抛出 `cannot open .../resource0 (EPERM) even as root`。如果 `mmap` 以 EBUSY 或 EACCES 失败，说明 NVIDIA 驱动持有 BAR：

```bash
sudo systemctl stop nvidia-persistenced; sudo nvidia-smi -pm 0
# 或者更强制
echo <BDF> | sudo tee /sys/bus/pci/drivers/nvidia/unbind
```

GA100 BAR0 是一个 16 MiB PRI 孔径（`0x1000000`）；偏移 0 处的 `PMC_BOOT_0` 标识芯片：`0x170000a1` 为 GA100，`0xb72000a1` 为 GA102，`0xb74000a1` 为 GA104。

> [!WARNING]
> **广告中的 `/dev/mem` 回退并不存在**
>
> 头部注释第 9 行写着 `# Falls back to /dev/mem path if resource0 fails.`。resource0 解析块实际上以 `log "ERROR: cannot find resource0 for $PCI_BDF"; ... exit 2` 结束。这段代码路径从未被写过。

### VBIOS 工具

`z1_dump_and_parse_vbios.sh` 通过三条 sysfs 命令（`echo 1 > .../rom`、`cat`、`echo 0 >`）无损地 dump VBIOS，回退到无前缀的 sysfs 路径，再回退到 `nvflash --save`。它**对闪存只读**：不存在写路径。若没有可用 dump 方法则退出码 2，dump 为空则退出码 3。

`z2_parse_vbios_table.py` 通过四个魔术定位 ROM 结构：偏移 0 处的 `NVGI`、经 ROM 头指针 `+0x18` 的 `PCIR`、BIT 模式 `ff b8 42 49 54 00`，以及绝对 `0x2000` 处的 `RFRD`。CFG1 strap 表通过从 `0x30000` 到 `0xB0000` 的步长 1 扫描自动定位：寻找 16 个连续的 4 字节条目，其字节+2 在 `{0x44,0x55,0x66,0x77}` 中、字节+3 在 `{0x02,0x22}` 中。

> [!WARNING]
> **解析器的标签在四处过时**
>
> 它的 `extract_cfg1_strap_table` docstring 引用"A100 PCIe 中的 ~0x3FB18"，而对照表把它放在 `0x4285A`；`extract_rfrd` 把 RFRD 称为"power table"，而它是图像布局描述符，`field_0C` 是 MAC 校验过的范围大小而非功耗上限；`extract_fbpa_tier_table` 可能匹配到 CFG1 表本身并报告重复；`find_subsystem_id` 是一个桩。任何人逐字引用该工具的输出标签都会传播这四处错误。参见 [VBIOS](../hardware/vbios.md)。

### `pcielink.sh`

标准的 PCIe 现场报告收集器，也是任何链路相关问题报告的合适附件。它自动发现 `10de:20c2` / `10de:2082`（回退到任意 NVIDIA 3D 控制器），并在端点及其父桥两者上解码：

| Capability 偏移 | 字段 |
|---|---|
| `CAP_EXP+0c.l` | LnkCap |
| `CAP_EXP+2c.l` | LnkCap2 |
| `CAP_EXP+10.w` | LnkCtl |
| `CAP_EXP+12.w` | LnkSta |
| `CAP_EXP+24.l` | DevCap2 |
| `CAP_EXP+28.w` | DevCtl2 |
| `CAP_EXP+30.w` | LnkCtl2 |
| `CAP_EXP+32.w` | LnkSta2 |

加上 sysfs 链路速率与宽度、`nvidia-smi pcie.link.gen`、AER 计数器，以及带 OPT 熔丝三元组的 `SEC2_DEBUG` dmesg 行数。该工具在两台独立的双卡 Gen2 已解锁机器上打印了 `SEC2_DEBUG lines=152` 与 `OPT=00000001/00000001/16680000`，一台 HiveOS、一台 Unraid。

> [!NOTE]
> **行数不是可靠的跨构建指纹**
>
> 每个记录值都不同：归档的单卡 8 GB 捕获为 29，归档的双卡 Gen2 分支 `610.43.03` 日志为 134，报告工具为 34（Gen1 构建）与 80（Gen2 构建），两台双卡 Gen2 机器上的 `pcielink.sh` 为 152。不要把不匹配读成安装失败。

### A100 探测套件

用于构建 Gen2 差分的三步、可选写工作流：

```bash
python3 probe.py which
sudo python3 probe.py inventory --out a100_native.json   # 只读
sudo ./sweep.sh                                          # 强制 Gen1/2/3，通过 EXIT trap 自动恢复
sudo python3 probe.py write-test --confirm               # 写入后立即恢复
```

`write-test` 触碰 `0x880a8`（目标速率改为 2）、`0x8c044`（改为 `0x00000002`）与 `0x88088`（retrain 位 5），把每个分类为 `WROTE-OK` 或 `REJECTED(PLM?)`。掩码读哨兵是 `0xBADF5040`。套件中没有任何东西写熔丝或在重启后持久。请在 GPU 空闲时运行：sweep 会对活动链路做 retrain。

### 显存验证器

| 工具 | 证明什么 | 机制 |
|---|---|---|
| **`check_fold.py`** | **权威。** 解锁显存是否真实、未被别名（alias） | 分配全部空闲显存减去 2 GiB，用 PTX `sm_80` `fill` 内核把每个 64 KB 页写入自己的索引，再用 `chk` 内核读回每一页。必须是密集的：fold 在通道**交错**偏移处别名，所以 `LOW[0]` 映射到 `(40 GiB + interleave)` 而非 `(40 GiB + 0)`，稀疏探测会得到假阴性。通过 ctypes 使用 `libcuda`，用 `st.global.wt.u32` 与 `ld.global.cv.u32` 绕过缓存。输出 `REAL, NO FOLD` 或 `FOLD/mismatch @<pageindex>`；退出码 0 真实、1 fold、2 错误 |
| `cuda_dbg.py` | 更轻量的别名测试 | `cuMemGetInfo_v2`，然后 `cuMemAlloc_v2` 依次尝试 64、60、56、52、48、44、42 GiB 直到成功；在偏移 0 写 `0xAAAA0000`、在 40 GiB 写 `0xBBBB0000`，读偏移 0。读回 `0xBBBB0000` 意味着空间别名。它会泄漏分配，所以每次驱动加载只运行一次 |
| `cuda_memtest` 1.2.3 | 社区显存验证器 | 在第一个错误处退出。在解锁卡上报告 `global memory size=85545582592`。在 80 GB profile 下打印 `Attached to device 0 successfully.` 然后无限挂起，除非限制在 39 GB |

> [!CAUTION]
> **更早的 fold 检测工具链本身是坏的**
>
> 一次 SBR 回到一致原生状态（10240 MiB、驱动 610.43.03、CFG1 `0x02449000`）后的对照运行分配了 9 GiB 真正原生显存，却在五轮中报告"4608 chunks, 4608 corrupt/aliased"，也就是原生显存在"fold"——这不可能。同一工具链更早曾把 10 GiB 报告为完全别名：第 1 轮约 26.6 GB/s，第 2 至 5 轮 197 至 198 GB/s。这追溯性地使一批"40 GB 处折叠"结论失效。用 `check_fold.py`，不要用任何更早工具链的输出。

整个测试中使用的标准驱动拆除序列，至今仍然正确：

```bash
sudo modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia
echo 1 | sudo tee /sys/bus/pci/devices/<BDF>/reset
```

---

## 第 1 代：手动 BAR0 poke 与 Python ROP 解锁器

### 那个时代是什么样

2026-07-12 在驱动内补丁出现之前可用的手动流程：

```text
run the ROP script -> FLR -> kill the NVIDIA driver -> FLR again -> run the SM unlock script
```

从 TTY 运行，使用开放内核模块 580.159.04，ROP payload 由 `patch_gsp.py` 拼接进 `gsp_tu10x.bin`。`unlc.py` 演示了正式补丁至今仍在使用两步模型：漏洞利用只需打开 FEAT PLM，之后 SS0 与 SS1 就是普通的主机写。

### 工具及其各自的死法

| 工具 | 角色 | 归宿 |
|---|---|---|
| `deploy.py --path sec2-rop` | 编排器，把一个 63,232 B 的 payload 写到 `/var/lib/cmp170hx/payload.bin`，安装一个 systemd 单元 | 被驱动内投递取代。其首个 release 还因把 `--verify` 传给 `load_custom_bin.py`（其 argparse 不接受）而中止（退出码 2，非硬件故障）。2026-06-24 修复 |
| `deploy.py --path vbios-memory` | 重写 VBIOS 中的 CFG1 strap 层级 | **从未工作。** 产生 `[vbios-memory] ERROR: Not a PCI Option ROM (bad magic at 0x00)`：它期望内部镜像，而不是原始 ROM dump。整个 VBIOS 显存路线在 2026-06-23 被放弃 |
| `load_custom_bin.py` | 加载 Falcon 二进制并 dump DMEM | 被并入 refire chain 的投递原语 |
| `unlc.py` | PLM 打开后的主机侧 SS0/SS1 写入器 | 其模型存活于补丁 0001 内部；脚本本身没有 |
| `stack_gen.py` | ROP 栈构建器 | 首个 release **把全部金丝雀槽位清零**，不可能工作：`D[0x6340]` 处的金丝雀必须复制到每个帧中，否则 `__stack_chk_fail`（`0x7dd9`）触发。常量是 `exit_addr 0x79e7`、`payload_size 0xF700`、`dma_target 0x0900`、`stack_start 0xf75c` |
| `patch_gsp.py`、`payload-lnject.py` | 对 `gsp_tu10x.bin` 做 ELF 手术：解析 ELF64 头（`e_shoff 0x28`、`e_shentsize 0x3A`、`e_shnum 0x3C`、`e_shstrndx 0x3E`），找到 `.fwsignature_ga100`，原地覆盖，把 `sh_size` 改为 `0xF800`，把 `.shstrtab` 追加到 EOF，重写 `e_shoff` | 被取代。`.fwsignature_ga100` 位于 580.159.04 blob 的文件偏移 `0x1D09F0F` 处 |
| `scan_dmem.py` | 更安全的 ELF 补丁变体，用 pyelftools，把替换节追加到 EOF 并尊重 `sh_addralign`，外加 DMEM 扫描 | 被取代 |
| `nuke.sh`、`b.sh` | PLM 持久性扫描（27 个候选地址，9 轮每轮 3 个，每轮两次 FLR，不加载驱动） | 它们的金丝雀字面量是 `CANARY_ADDR = 0x6340` 处的 `0xFACEB13D`，`DMA_TARGET = 0x0800` |
| `falcon_emulator.py` | 本地 Falcon 模拟 | 非承重；论文自己的模拟器从未发布 |
| `cmp170hx-unlock.service` | systemd 持久化，轮询 `/proc/driver/nvidia/gpus/<BDF>/clients`，每当新 CUDA 进程打开 GPU 时在 250 ms 内重新应用 | 被取代。守护进程把进程打开与重新应用竞争，且无法经受驱动重载 |
| `/opt/cmpunlocker/daemon/watchdog.py` | 替代守护进程设计 | 被取代 |

### 两次值得理解的取代

**磁盘上的 GSP 固件补丁变成了驱动内签名 memdesc。** 直到 2026-07-17，人们都相信 payload 必须拼进随附的 GSP ELF。当时存在三个独立的 ELF 补丁器。流水线把打过补丁的 blob 复制到 `/lib/firmware/nvidia/580.159.04/gsp_tu10x.bin`，加载驱动，验证 PLM 读到 `0xFFFFFFFF`，然后恢复原件。补丁 `0001` 以在 `0xf800` 分配 `pSignatureMemdesc` 并在内存中填充取代了这一切。没有 ELF 手术、没有需要备份或恢复的固件文件、没有把打过补丁的 blob 留在磁盘上的风险，payload 还可以在两次 Booter 点火之间重建。**残留：** `remove.sh` 仍会删除五种 `gsp_tu10x.bin.cmpunlocker.*` 后缀。

**用户态守护进程持久化变成了打过补丁的内核模块。** 解锁现在在每次打过补丁的模块启动 GSP 时运行于 `kgspBootstrap` 内部：没有守护进程、没有轮询、没有重新应用窗口。**残留：** `remove.sh` 仍会停止 `cmpunlocker` systemd 单元并 `pkill` 看门狗。

### 逐出 ROP 与配方目录

两个第 1 代工件值得作为技术而非工具记住。

原始 booter 栈从硅片上逐字恢复，使用由 gadget `0x7de9` 构建的逐出 ROP：该 gadget 把一个选定的 DMEM 字写入 SEC2 mailbox，所以每次启动泄漏一个 dword。DKMS 下约 **35 次启动、每次约 90 秒**，每趟约一小时。`D[0xFF74]` 以下的区域无法泄漏，因为 ROP 本身就在那里。由于金丝雀每次启动都会重新随机化，运行两次 dump 并做 diff 会揭示哪些槽位恰是金丝雀：一个限制变成了技术。

八种命名的 ROP 配方以参数化目录维护，区别在于 rejoin 点（`0x37b7` 对 `0x37cc`）、劫持 gadget、栈样式与 smash 大小（`0xF800`、`0xF810`、`0xF820`）：`rejoin_short_37cc`、`whole_stack_37b7`（守卫 `0xFACEB13D`）、`dummy_shift_37cc`、`srw_v1_37b7`、`srw_v2_37cc`、`waa_37cc`、`waa_37b7`、`waa_3747`。研究链标准化采用的多写模式是 `0x4d4(r0=addr,r1=val,RA=0x10b9) -> [0x10b9 write -> 0x10aa-epi] xN -> TERM`。那个 `0x10b9` 中途进入形式属于净室与免驱动工具：正式 payload 改种 `0x000010aa`，字符串 `10b9` 在正式树中无处出现。参见 [ROP 链](../unlock/rop-chain.md)。

---

## 间奏：第一个 `cmpunlocker` 是免驱动 Python

在 **2026-07-14T21:47:02-07:00** 与 **2026-07-18T19:11Z** 之间，公开 `cmpunlocker` 仓库不含任何形式的驱动补丁。它发货 `payload/build.py`、`payload/gsp_patch.py`、`payload/pipeline.py`、`payload/bar0.py`、`payload/driver.py`、`unlock/compute.py` 与一个 `daemon/` 看门狗。

它的流水线：定位 `/lib/firmware/nvidia/*/gsp_tu10x.bin`，备份，构建 `0xF800` 字节 ROP payload，把它拼进 `.fwsignature_ga100` ELF 节，加载原厂模块，FLR 复位，激进卸载，再次 FLR 复位，从主机经 BAR0 写 SS0 `0x0082381C = 0x88888888` 与 SS1 `0x00823820 = 0x00000008`，然后恢复原厂固件。

它的 ROP 构建器每次写发出一个帧，使用至今仍在用的网格：

```yaml
dmem_layout:   { dma_target: 0x0800, payload_size: 0xF800, guard_addr: 0x6340, canary: 0xFACEB13D }
booter_addrs:  { bar0_write_gadget: 0x10B9 }
payload_frames:
  frame_start_addr: 0xFF48
  frame_stride:     0x18
  frame_field_offsets: { r0: 0x00, r1: 0x04, r2: 0x08, r3: 0x0C, saved_reg: 0x10, return_addr: 0x14 }
```

带一个返回 `0x0000810D` 的零化终结帧。它的三次写是 `0x009A0204 = 0x02779000`、`0x00100CE0 = 0x0000020B` 与 `0x00823804 = 0xFFFFFFFF`。

---

## 第 2 代：`cmpunlocker`，正式驱动补丁

这是规范工具。仓库标语："A tool to unlobotomize your NVIDIA card!"。

### `master` 实际包含什么

恰好八个顶层项：

```text
.github/pull_request_template.md
.gitignore
LICENSE
README.md
common/constants.yaml
driver/
install.sh
remove.sh
```

`master` 上**没有** `verify.sh`、**没有** `tools/` 目录、**没有** `probe.sh`、**没有** `requirements.txt`（2026-07-19 在 `7019bc2` 删除）也**没有**测试套件。

### `install.sh`

六步：root 检查、GPU 检测、profile 选择、驱动 / Secure Boot / 头文件检查、构建与安装、完成。一切都被 tee 到 `logs/install_$(date +%Y%m%d_%H%M%S).log`。

```bash
lspci -nn | grep -iE '10de:20b0|10de:20c2|10de:2082' | head -1
```

若无匹配则以 `No CMP 170HX GPU found (10de:20b0 / 10de:20c2 / 10de:2082)` 退出，并对任何其他设备 ID 警告 `In-driver unlock path is gated on PCI ID 0x20C2 / 0x2082.`。因此 `10de:20b0` 卡会安装但不解锁。

`detect_card_profile()` 读取 `nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits | head -1`，映射四个窗口：`>= 60000 MiB` 到 `8gb`（已解锁）、`35000-59999` 到 `10gb`、`7680-8704` 到 `8gb`、`9728-10752` 到 `10gb`。其余打印 `unknown:<mib>`，安装器退出并让你传 `--profile=8gb|10gb`。

> [!CAUTION]
> **混合 GPU 主机上自动检测不安全**
>
> `detect_card_profile()` 读取的是 **`nvidia-smi` 顺序中的第一块 GPU**，而不是 `lspci` 找到的 CMP。一台带 RTX 3080 10 GB 与 8 GB CMP 170HX 的系统会从 3080 检测到"10GB"并选错 profile。至少两名用户复现过；其他 CMP SKU 也曾在多 GPU 主机上被误检测为 10 GB 170HX 卡。**多 GPU 主机上始终显式传 `--profile`。**

Secure Boot 是硬门槛：如果 `/sys/firmware/efi` 存在、`mokutil` 存在且 `mokutil --sb-state` 报告已启用，安装器拒绝。驱动版本必须与 `driver/VERSION`（`610.43.03`、`610.43.02`）中的一行精确匹配，按序从 `/proc/driver/nvidia/version`、然后 `nvidia-smi --query-gpu=driver_version`、然后对 `/lib/firmware/nvidia/<version>/` 的目录探测、最后按最高排序的 `/lib/firmware/nvidia/*/` 检测。内核头文件必须存在于 `/lib/modules/$(uname -r)/build`。

### `driver/build.sh`

它从不随附 NVIDIA 代码。它用 `curl -L --fail` 下载 `https://github.com/NVIDIA/open-gpu-kernel-modules/archive/refs/tags/${VERSION}.tar.gz`，缓存在 `driver/.build/` 下，干净解包，用 `patch -p1` 应用每一个 `driver/patches/*.patch`，在清除 `_out` 与 `conftest` 后用 `make -j$(nproc) modules SYSSRC=/lib/modules/$(uname -r)/build` 构建，并把 `nvidia.ko`、`nvidia-modeset.ko`、`nvidia-uvm.ko`、`nvidia-drm.ko` 与 `nvidia-peermem.ko` 以 0644 模式安装到 `/lib/modules/$(uname -r)/updates/cmpunlocker/`。

它在那里写三个单行元数据文件：`driver_version`、`card_profile`（`8gb` 或 `10gb`）与 `unlock_geometry`（`64GB` 或 `40GB`），然后运行 `depmod -a "${KVER}"`，并通过 `update-initramfs -u -k`、`dracut --force --kver` 或 `mkinitcpio -P` 中第一个可用的重建 initramfs。

它还交叉检查打过补丁的模块是否胜出。重载前运行 `modprobe -n -v nvidia | awk '/insmod/ {print $2; exit}'`，若不在 `updates/cmpunlocker/` 下则警告 `Resolved nvidia.ko is not under updates/cmpunlocker/ - stock may still win`。重载后比较 `/sys/module/nvidia/srcversion` 与 `modinfo -F srcversion .../updates/cmpunlocker/nvidia.ko`，不匹配则警告 `Loaded nvidia srcversion (X) != patched (Y)`，清除 `reload_ok`，并建议冷重启加 `cat /proc/driver/nvidia/version  (should NOT say dvs-builder)`。

> [!WARNING]
> **`--profile` 不再选择几何**
>
> 在 `memory` 分支快照上，`--profile` 确实通过构建期 Python 正则重写选择 CFG1 与 LMR。在当前 `master` 上不是。补丁 `0001` 包含 `build.sh` 守卫寻找的全部六个标记，所以重写会打印 `runtime device-id geometry (profile metadata=<label>)` 并不编辑任何东西就退出。几何在 GSP 启动时按 `pGpu->idInfo.PCIDeviceID >> 16` 选择。`--profile` 现在只影响打印的横幅、`EXPECTED_MIB` 与元数据文件。2026-07-18 之前写的说明对此是错的。

### 六个补丁

| # | 文件 | 字节 |
|---|---|---|
| 0001 | `0001-sec2-postbl-plm-ss-cfg.patch` | 19,741 |
| 0002 | `0002-booter-verify.patch` | 3,988 |
| 0003 | `0003-late-pma.patch` | 10,580 |
| 0004 | `0004-bar0-pramin-clamp.patch` | 861 |
| 0005 | `0005-ce-scrub-workarounds.patch` | 1,642 |
| 0006 | `0006-persistent-sw-state.patch` | 603 |
| | **合计** | **37,415** |

补丁 `0001` 携带整个解锁：它把 `pSignatureMemdesc` 扩大到 `SEC2_POSTBL_TIMING_SIGNATURE_SIZE 0x0000f800ULL`（63,488 字节），用 `SEC2_POSTBL_TIMING_FILL_DWORD 0x000004a7U` 填充，在其中铺设 ROP 栈，并在每次 PLM 尝试时重新运行 `kgspExecuteBooterLoad_HAL`。**不对 `gsp_tu10x.bin` 做任何 ELF 手术。**

PLM 表是四项，每项最多尝试两次，WPR2 lo/hi（`0x001fa824`/`0x001fa828`）在循环前保存，每次尝试前与最后再写一次：

```c
{ 0x001fa7ccU, 0xfffff0ffU, "WPR_CFG" },   /* 注意：不是 0xffffffff */
{ 0x009a0148U, 0xffffffffU, "FBPA"    },
{ 0x001fa7c4U, 0xffffffffU, "WPR"     },
{ 0x00823804U, 0xffffffffU, "FEAT"    },
```

然后是四次普通主机寄存器写：

```c
GPU_REG_WR32(pGpu, 0x0082381cU, 0x88888888U);   /* SS0 */
GPU_REG_WR32(pGpu, 0x00823820U, 0x00000008U);   /* SS1 */
GPU_REG_WR32(pGpu, 0x009a0204U, cfg1Value);     /* 0x02779000 (20C2) / 0x02669000 (2082) */
GPU_REG_WR32(pGpu, 0x00100ce0U, lmrValue);      /* 0x0000020B (20C2) / 0x0000028A (2082) */
```

补丁 `0001` 还把原厂 `WPR2 already up` 硬错误放宽为 `NV_PRINTF(LEVEL_WARNING, "WPR2 already up before GSP boot; continuing for recovery\n")`，并把 `pGSCI->fb_length` 重写为 `0x0000001000000000ULL`（64 GB）或 `0x0000000A00000000ULL`（40 GB），外加最后 FB 区域的 `limit`、`reserved`、`supportCompressed`、`supportISO` 与 `performance = 20`。

内建 payload 可在运行时从 `/lib/firmware/nvidia/ga100/gsp/dmem.bin`（`SEC2_POSTBL_TIMING_DMEM_PATH`）覆盖，通过 `os_open_and_read_file` 加载 `0xf800` 字节。若缺失，驱动记录 `SEC2_DEBUG: <path> not found (0x%x), using built-in payload`（报告码 `0x59`，良性）并回退到编译进的填充，其默认单写是 `0x009a0148U = 0xffffffffU`。

> [!WARNING]
> **正式 payload 的标记字是 `0xc0deca7e`，不是 `0xFACEB13D`**
>
> `0xc0deca7e` 出现在 payload 偏移 `0x5b40`、`0xf758`、`0xf794`、`0xf7a0` 与 `0xf7c4`。更早的独立工具链在 `CANARY_ADDR = 0x6340`、`DMA_TARGET = 0x0800` 处使用 `0xFACEB13D`。`0x5b40 + 0x0800 = 0x6340`，所以是同一槽位、不同字面量。读正式代码时不要假定 `0xFACEB13D`。

正式驱动内栈与独立免驱动链共享**同一收尾配方**：在 payload 偏移 `0xf78c` 到 `0xf7f8`，补丁 0001 写非零 gadget 序列 `0x815a, 0x8e18, 0x815a, 0x1fbd, 0xffbc, 0x582d, 0xcbd, 0x3, 0x1fbd, 0xccb, 0x7f2f`，并把 payload 字 `0x1100 = 0x00000007`。

### `common/constants.yaml`

仓库中机器可读的基准真相，与补丁 0001 中的 C 完全一致：

```yaml
driver_versions: [610.43.03, 610.43.02]
gpu: { vendor_id: 10de, device_ids: [20c2, 2082] }
compute: { ss0: "0x88888888", ss1: "0x00000008" }
profiles:
  8gb:  { stock_mib: 8192,  unlocked_mib: 65536, cfg1: "0x02779000", lmr: "0x0000020B", fb_bytes: "0x0000001000000000" }
  10gb: { stock_mib: 10240, unlocked_mib: 40960, cfg1: "0x02669000", lmr: "0x0000028A", fb_bytes: "0x0000000A00000000" }
```

### `remove.sh`

卸载器是 `remove.sh`，需要 `--yes` 或 `-y`。**树中任何地方都没有 `uninstall.sh`**，尽管 `docs` 分支那么说。五步：停止并禁用遗留 `cmpunlocker` systemd 单元并 `pkill -f /opt/cmpunlocker/daemon/watchdog.py`；`rm -rf /lib/modules/*/updates/cmpunlocker` 并按内核 `depmod -a`；重建 initramfs；删除 `/lib/firmware/nvidia/*/gsp_tu10x.bin.cmpunlocker.{bak,patched,tmp,cleanup,pat}`；移除 `/opt/cmpunlocker`；然后停止显示管理器与 `nvidia-persistenced`，强制卸载四个模块并再次 `modprobe nvidia`。一位 HiveOS 上的测试者报告运行后两张卡都回到挖矿，这是"改装非破坏性"说法的基础（单次报告，中等置信度）。

操作流程参见 [安装](../procedures/install.md)、[验证](../procedures/verify.md) 与 [卸载](../procedures/uninstall.md)。

---

## 第 3 代：免驱动 SEC2 refire chain

> [!WARNING]
> **实验性**
>
> 这是一条并行路径，不属于 `cmpunlocker`，也不是任何人为了生产使用解锁卡应该运行的东西。它重要的原因是：它是唯一仍在追求"不修改驱动即可解锁"这一创始目标的路线。

`refire_chain_v6.py`（27,769 字节，2026-07-24 发布）在**不加载任何 NVIDIA 驱动**的情况下从用户态完成整个解锁，只用标准库（`os`、`sys`、`mmap`、`ctypes`、`struct`、`time`、`subprocess`）。它把 BAR0 映射为 16 MiB，把 SEC2 视为基址 `0x00840000`，复位 Falcon，把 NS 代码以不安全方式加载到 IMEM 0、HS 代码以 tag 寄存器安全加载到 `IMEM[ns]`，加载 DMEM，把 MAILBOX0/1 设为 WprMeta 物理地址，启动 CPU，然后反复溢出被签名 Booter 的签名读取 DMA。

操作流程：

```bash
echo 16 | sudo tee /proc/sys/vm/nr_hugepages
sudo rmmod nvidia_uvm nvidia
BDF=$(python3 -c 'import refire_chain_v6 as V; print(V.resolve_bdf())')
echo 1 | sudo tee /sys/bus/pci/devices/$BDF/reset
sudo python3 refire_chain_v6.py --all
```

模式：`--compute`（只关闭 SM 与张量节流，常开，FLR 黏性）、`--memory 40`（真实且稳定的档位）、`--memory 80`（80 GB 档位）、`--pcie-gen2`（仅 LnkCap2 上限）、`--pcie-retrain`。环境覆盖：`CMP_BDF`、`CMP_BOOTER_IMG`、`CMP_BOOTER_SIG`。`--all` 让卡处于 READY 状态，可无 FLR 加载驱动。除 `--pcie-retrain`（纯主机写）外，每个模式都需要 GPU 解绑与 hugepages。

前置条件严格：root；GPU 与任何 NVIDIA 驱动**解绑**；一个带签名的 GA100 `booter_load` HS ucode 镜像（约 60,160 字节，384 字节 RSA-3072-PSS 签名烤在 `0x8900`）；16 个 hugepages；内核命令行 `intel_iommu=off` 或 `iommu=pt`，使 DMA 物理地址即主机物理地址。它分配一个物理连续的 2 MiB hugepage，`mlock` 它，经 `/proc/self/pagemap` 解析物理地址（位 63 必须置位，否则 `page not present (need hugepages)`），并调用一段手工汇编的 clflush 加 mfence 桩（`0F AE 3F 48 83 C7 40 48 83 EE 40 7F F3 0F AE F0 C3`），因为"sig-DMA 是非一致的，必须命中 RAM 而非 CPU 缓存"。

值得知道的内部细节：`stage_radix3()` 必须运行，否则 Booter 的签名前 DMA 以原因 `0x9` 失败。它分配 `0x6000` 字节并写一条三级链（`[0x0000] = phys+0x1000`、`[0x1000] = phys+0x2000`、`[0x2000] = phys+0x3000`），然后 flush。WprMeta 模板是一个从真实 10 GB 启动捕获的 256 字节结构，只覆盖签名指针（`0x48`）、签名大小（`0x50` = `0xF800`）、radix3 指针（`0x10`）、radix3 大小（`0x18`）、bootloader 指针（`0x20`）与 bootloader 大小（`0x28`）。它的前两个字是 WPR 描述符魔术 `0x371a60b3` 与 `0xdc3aae21`。

### 版本谱系

| 版本 | 变化 |
|---|---|
| v1（`refire_chain.py`） | 每个 PLM 硬编码一个紧凑的两写 payload |
| v2 | payload 变成通用写引擎，接受扁平 `[(addr, value), ...]` 列表，零 WprMeta 或几何知识。WprMeta 在投递层只作为签名 DMA 溢出触发器构建一次。投递原语 `Bar0, alloc, flush, reset_sec2, load_booter, wpr_meta, start_wait, stage_radix3, geometry, fire, PATCHLOC` 逐字复用自硬件验证过的 v1。payload 大小 `0xF800`，入口收尾常量 `TAIL0 = 0x815a` |
| v6 | 添加上述模式标志、BDF 解析与环境覆盖 |

> [!CAUTION]
> **可移植性限制：仅 10 GB 卡**
>
> 已发布链携带从 **10 GB** 启动捕获的 WprMeta 模板。它不能未经修改应用于 `0x20C2` 8 GB 卡。制作 8 GB 模板是一个记录在案的开任务：在正常驱动 GSP 启动期间从 8 GB 卡捕获 `pWprMeta` 并替换。反正只覆盖六个字段，风险低，但还没人做。

---

## 第 4 代：未合并分支

捕获了十二个未发布分支快照（**加上发货的 `master` 共十三棵树**）：`80`、`Gen2`、`PG199`、`clanker/driver-port`、`debug-gen2`、`deced`、`docs`、`ecc`、`far`、`housekeeping`、`memory`、`multiple-cards`。远端存在十六个未发布分支引用；`code-simplification`、`dual-geometry-fix`、`fix` 与 `v0.1` 未被快照，本 wiki 任何地方都不分析。参见 [方法论](../appendix/methodology.md)。

| 分支 | Tip | 增加什么 | 结论 |
|---|---|---|---|
| `memory` | 2026-07-18 | 最初的驱动内显存解锁，单一烤入几何 | 合并到 `master` |
| `housekeeping` | 2026-07-18 | `43c762d "Add 2082 (10GB) device support to all patches"`；还删除了 `.ai/CONTEXT.md` agent 指令文件 | 修复后合并 |
| `ecc` | `bb4d669`，2026-07-18 | 一个提交，"Fixed dual geometry support"。**不含任何 ECC 代码** | 已合并。ECC 熔丝关闭且无已知杠杆 |
| `multiple-cards` | `b1cb6d8`，2026-07-18（07-19 宣布） | 把 `detect_card_profile()` 换成 `profile_from_devid()`（`20c2` 到 `8gb`，`2082` 到 `10gb`，否则不支持），遍历**每一条**匹配的 `lspci` 行，构建五个并行数组，添加第三个 `mixed` profile 设置 `SKIP_GEOMETRY_REWRITE=1`。导出 `CMPUNLOCKER_GPU_INVENTORY`，持久化为 `/lib/modules/$(uname -r)/updates/cmpunlocker/gpu_inventory`，每 GPU 一行，形式 `0000:0b:00.0 20c2 8gb 65536`。添加 `verify.sh` | 未合并。参见 [多 GPU](../procedures/multi-gpu.md) |
| `80` | `3c53aca`，2026-07-19 | 把补丁 0001 的 10 GB 分支重写为 `cfg1Value = 0x02779000U`（**8 GB** 卡的 CFG1），配 `lmrValue = 0x0000028AU` 与 `targetFbBytes = 0x0000001400000000ULL` | **不要使用。** 不稳定 |
| `clanker/driver-port` | `153cd6d`，2026-07-21 | 按分支补丁目录 `driver/patches/{580,590,595,610}/`，由 `BRANCH="${VERSION%%.*}"` 选择。`driver/VERSION` 列出十二个版本但 `constants.yaml` 五个，这是公认的内部不一致。其 `install.sh` 与 master **字节级相同** | 未合并，从未在 610 以下启动测试 |
| `debug-gen2` | `746d9f7 "PCIe Gen 2 works!"`，2026-07-23 | 补丁 0001-0007，外加以 systemd oneshot 安装的 `tools/retrain.sh` 与 `tools/cmpretrain.service` | 被 `Gen2` 取代 |
| `Gen2` | `2f27474`，2026-07-24；tip `a4de322`，2026-07-26 | `2f27474 "Gen2 + multiple-card support"` 添加 `0008-pcie-gen2-probe-retrain.patch`、多卡支持与 `verify.sh`，并**删除** `tools/cmpretrain.service`（`tools/retrain.sh` 保留）。tip `a4de322` 是对 `master` 的纯合并，只触碰 `.github/pull_request_template.md` | 现行 Gen2 基 |
| `far` | `8854d3e "Remove clamp link to Gen1"`，2026-07-26 | 相对 `Gen2` 恰好改动一行：`RMPcieLinkSpeed` 从 `0x1` 到 `0x2` | |
| `deced` | `2326599`，2026-07-27 | 把 `tools/retrain.sh` 中硬编码的 BDF 替换为 `find_gpu_bdf()`。**归档中最新的 Gen2 树** | |
| `docs` | `651b6d5`，2026-07-27 | 七个文档提交 | **非权威。** 见下文 |
| `PG199` | | Drive A100 对比快照 | 仅参考 |

### 仅分支工具

`verify.sh` 是一个**仅分支**的多 GPU 安装后检查器，`master` 上不存在。它优先用已安装的 `gpu_inventory`，否则经 `lspci -nn | grep -iE '10de:20c2|10de:2082'` 枚举。`is_unlocked_memory` 对 `8gb` 接受 `>= 60000 MiB`、对 `10gb` 接受 `35000..59999 MiB`；`is_stock_memory` 接受 `7680..8704` 与 `9728..10752`。每 GPU 状态为 `OK`、`STOCK`、`MISSING` 或 `UNEXPECTED`。缺失 `SEC2_DEBUG` dmesg 轨迹是警告而非失败，因为环形缓冲会轮转。

> [!NOTE]
> **未解决问题**
>
> **`verify.sh` 从不检查 PCIe Gen2，即使在 Gen2 分支谱系上也不检查。** 在 `Gen2/verify.sh`、`far/verify.sh` 与 `deced/verify.sh` 中 grep "pcie" 得零命中。Gen2 验证完全留给用户手工运行 `nvidia-smi`。修复很小：查询 `nvidia-smi --query-gpu=pcie.link.gen.current,pcie.link.gen.max`，或复用 `pcielink.sh` 的 `CAP_EXP+12.w` LnkSta 解码。

### Gen2 谱系取代

**用户态 retrain 脚本变成了驱动内探测期 retrain。** `debug-gen2` 安装 `/usr/local/sbin/retrain.sh` 与 `cmpretrain.service`（`Type=oneshot`、`ExecStartPre=/bin/sleep 15`、`WantedBy=multi-user.target`）。从 `Gen2` 起，`0008-pcie-gen2-probe-retrain.patch` 在 `kernel-open/nvidia/nv.c` 中添加 `nv_cmp170hx_retrain_gen2()`，以 `gpu->device == 0x20c2 || gpu->device == 0x2082` 为门槛，遍历 `pci_upstream_bridge(gpu)` 并在探测期 retrain。安装器主动禁用 `cmpretrain.service` 与 `cmp-gen2-retrain.service`，`rm -f` 辅助脚本，打印 `Removed legacy PCIe retrain helpers`。`multi-user.target` 之后一个睡 15 秒的 oneshot 很脆弱，无法在驱动声明设备之前运行。

Gen2 谱系安装器还写 `/etc/modprobe.d/cmp-pcie-gen2.conf` 并配置 IOMMU，向内核命令行追加 `intel_iommu=on iommu=pt` 或 `amd_iommu=on iommu=pt`，除非传了 `--no-iommu`。

> [!CAUTION]
> **`Gen2` 安装一个 Gen1 clamp**
>
> `debug-gen2` 与 `Gen2` 写 `options nvidia NVreg_RegistryDwords="RmForceEnableGen2=1;RMPcieLinkSpeed=0x1"`，把链路钉在 Gen1，同时却在尝试启用 Gen2。`far` 与 `deced` 写 `0x2`。哪个值正确**真的未定案**：两种写法都发货，作者各自相信自己的对，且不存在 A/B 启动测试。在一张卡上做一次三方启动对比即可定案。

> [!CAUTION]
> **不要跟随 `docs` 分支**
>
> `docs/INSTALLATION.md` 第 40 行说 `sudo ./uninstall.sh --yes`（没有这样的文件）；`docs/ARCHITECTURE.md` 第 81-82 行声称 `SEC2_DEBUG: SS0 = 0xffffffff` / `SS1 = 0xffffffff`，而正式代码写的是 `0x88888888` 与 `0x00000008`；`docs/DEBUGGING.md` 第 15 行说"所有 PLM 必须显示 `0xffffffff`"，而 WPR_CFG 在 `0x001fa7cc` 被打开为 `0xfffff0ff`。该分支还编造了代码中无处可寻的缩写展开。

---

## 社区 fork 与相邻工具

发布后数天内至少有六个公开仓库 fork 或重实现了解锁。

| 仓库 | 性质 |
|---|---|
| `amoghmunikote/cmpunlocker` | 上述参考实现 |
| 若干个人 fork，其中一个带 `combined-multiple-cards-gen2` 分支 | Fork；一个把 Gen2 工作与多卡支持结合。按本 wiki 的匿名化政策省略所有者姓名 |
| `abobasixseven/unlock-cmp-170hx` | **不是写稿。** 一个 AI agent 执行提示词：只有 `README.md` 与 `cmp90_compute_unlock_prompt.md`，都以 `EXECUTE STEP BY STEP: 5 (preparation) -> 6 (installation) -> 6.5 (cold reboot) -> 7 (verification)` 之类的行结尾，通篇硬编码特定主目录。其寄存器表与正式补丁匹配，但它的散文与 PCIe Gen2 章节是二手总结，不是一手测量 |
| `theneocorp/cmppatcher` | 一条真正不同的路线：直接补 NVIDIA 驱动**二进制**，使补丁跨驱动更新持久。报告 3D 加速与 FP32 FMA 绕过 |

相邻的、非解锁工具：

| 工具 | 用途 |
|---|---|
| `CMPGPU-patch-script`（`optimize-cmp-cuda.py`） | 交互式 llama.cpp 源码补丁器，五组独立优化项，每组默认 `n`：`fp32_fma_flag`（给 CUDA_FLAGS 加 `-fmad=false`）、`fp32_fma_split`（把 `fmaf(...)` 重写为 `__fadd_rn(__fmul_rn(...), ...)`）、`math_intrinsics`、`dp2a`、`fp16_bf16_cuda_core`。跨七个文件十一个 PatchSpec 条目；`.cmp-bak` 备份；`--dry-run`、`--no-backup`、`--restore`。其 README 警告在非 170HX 的 CC 8.x 设备上性能可能**下降** |
| `170tune`（`/usr/local/bin/170hx-oc`） | 调校与资质测试工具链，测量、设门槛并恢复时钟与电压设置，把"跑完一个基准"当作"什么也证明不了"。附带 26,987 字节的调校指南。其设置在重启后是否持久是一个未决问题。参见 [调校](../operations/tuning.md) |
| `cmp170hx-gen2-setup.sh`（12,389 B，2026-07-26） | 独立 Gen2 设置工件，不同于 `Gen2` 分支的驱动内路线，与一份 `PCIE_GEN1_LOCK.md` 分析一起发布 |
| `unlock_host_610.sh` | nvidia-open 610.43.03 的主机侧脚本：从 `vfio-pci` 解绑，清除 `driver_override`，杀掉 `nvidia-persistenced`，卸载四个模块，`modprobe ecdh_generic ecc ecdsa_generic`（模块有加密依赖），`insmod kernel-open/nvidia.ko`，断言 `/sys/module/nvidia/version == 610.43.03`，`insmod nvidia-uvm.ko`，把 BDF echo 进 `/sys/bus/pci/drivers/nvidia/bind`，然后 `mknod` 设备节点。绑定触发 `RmInit`，因而触发驱动内解锁 |

### FMA 变通方案家族

FP32 FMA 封锁在编译期绕过，而非由任何解锁解决：OpenCL 经 `#pragma OPENCL FP_CONTRACT OFF` 加对 `fma()` 与 `mad()` 的宏遮蔽，CUDA 经 `nvcc -fmad=false`，SYCL 经 clang `-ffp-contract=off`。显式调用与对 `a * b + c` 的隐式收缩都必须抑制，数值后果是两次舍入而不是一次。

---

## Booter 提取工具链

只有当你自己研究漏洞利用时才需要，解锁卡不需要。

可读 Booter 反汇编的有文档配方：

```text
extract the debug binary from the NVIDIA .ko with the Nouveau extraction tool
  -> decrypt with rijndael-tool using NVIDIA's public test key
  -> check it is not compressed (NVIDIA uses a compressor called binHex)
  -> disassemble with envytools (envydis, target fuc5)
  -> annotate
```

`nouveau/extract-firmware-nouveau.py` 必须为 GA100 打补丁，因为生成的 C 数组名变成了 `kgspBinArchiveBooter{LOAD}Ucode_{GPU}_BINDATA_LABEL_IMAGE_{fuse.upper()}_data` 的形式。原厂脚本通过 `--debug-fused` 键选择 prod 或 debug ucode，默认为 prod，并且需要来自匹配**闭源**驱动包的固件 `.bin` 文件，其版本列在 `version.mk` 中，*不是*开放分支的版本号。

在开放驱动中，带签名的 HS ucode 位于 `src/nvidia/generated/g_bindata_kgspGetBinArchiveBooterLoadUcode_GA100.c`，含三个归档条目：`..._IMAGE_PROD`（用 NVIDIA 自己的 bindata 压缩器压缩，不是普通 zlib）、`..._SIG_PROD`（未压缩，384 字节数组）与 `..._PATCH_LOC`（4 字节 = `0x8900`）。镜像约 60,160 字节。

另一种提取从已加载的原厂驱动上经基址 `0x840000` 的 SEC2 Falcon 窗口实时 dump booter：写 `IMEMC (0x840180) = off | (1 << 25)` 启用自增读，并循环读 `IMEMD (0x840184)`，`off = 0 ... 0x8700`；DMEM 同样经 `DMEMC (0x8401c0)` / `DMEMD (0x8401c4)`。置信度**中**：流程具体、寄存器地址看起来正确，但从未发布过捕获的 dump，而且读取必须在驱动 PIO 加载 booter 之后、重用 SEC2 之前立即进行。

生产 booter 无法修改甚至无法读取：它用强密钥加密。因此漏洞利用通过栈改变*执行流*而非代码，也不需要重新签名。

> [!NOTE]
> **未解决问题**
>
> `envydis` 的 `fuc5` 目标能成功反汇编 GA100 booter，尽管 envytools 表名义上把 `fuc6` 分配给 GP102 及以后的部件。envytools 已约 8 年未更新；`envyhooks` 被建议为继任者但缺乏同等功能；`faucon` 只针对 fuc5。170HX 的 SEC2 正式是 fuc5 还是 fuc6 未定。下一步：对同一镜像做一次 fuc5 与一次 fuc6 解码并做 diff，寻找只有一个目标能连贯解码的指令。

---

## 已废弃路径，一览

不要跟随其中任何一条。

| 死路 | 原因 | 改用 |
|---|---|---|
| 在磁盘上补丁 `gsp_tu10x.bin`（`patch_gsp.py`、`payload-lnject.py`、`scan_dmem.py`） | 被驱动内签名 memdesc 取代；会在磁盘上留下打过补丁的 blob | `cmpunlocker` 补丁 `0001` |
| systemd 持久化（`cmp170hx-unlock.service`、`watchdog.py`） | 把进程打开与重新应用竞争；无法经受驱动重载 | `/lib/modules/.../updates/cmpunlocker/` 中的打过补丁模块 |
| `deploy.py --path vbios-memory` | 从未工作；当时修改 VBIOS 会得到无法工作的设备 | 寄存器级解锁 |
| `stack_gen.py` v1 | 清空全部金丝雀槽位；`0x7dd9` 处 `__stack_chk_fail` 触发 | 任何后来的 payload 构建器 |
| 手动五步 TTY 流程 | 2026-07-18 被取代 | `sudo ./install.sh` |
| `--profile` 作为几何选择器 | 在 `master` 上降级为元数据标签 | 几何在 GSP 启动时按 PCI ID 选择 |
| `sudo ./uninstall.sh --yes` | 没有这样的文件 | `sudo ./remove.sh --yes` |
| `Gen2`/`far`/`deced` 上的 `tools/retrain.sh` | 死代码；安装器删除它，补丁 0008 在内核内 retrain | 补丁 `0008` |
| `80` 分支 | 约 40 GB 以上不稳定 | 10 GB 卡上 40 GB |
| `docs` 分支 | 有文档记载的事实错误 | 本 wiki 与源码 |
| `check_fold.py` 之外的任何 fold 工具链 | 更早的工具链把原生显存报告为折叠 | `check_fold.py` |
| 把 `gsp_tu10x.bin` 当作 booter 解密 | 它是 GSP RISC-V ELF payload，不是 SEC2 Falcon booter。Ghidra 从它产出约 100 MB 的 C，objdump 产出约 1.5 GB 的汇编 | 反汇编 `booter_load_ga100_*.bin`，约 25 kB，约 390 kB 汇编 |

---

## 参见

- [项目时间线](timeline.md)，以上每次取代背后的日期
- [净室与出处问题](clean-room-and-provenance.md)
- [死路](dead-ends.md)
- [六个驱动补丁](../unlock/driver-patches.md) 与 [ROP 链](../unlock/rop-chain.md)
- [安装](../procedures/install.md)、[验证](../procedures/verify.md)、[故障排查](../procedures/troubleshooting.md)
- [寄存器参考](../unlock/register-reference.md) 与 [寄存器索引](../appendix/register-index.md)
