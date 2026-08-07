# 显存几何：容量解锁的工作原理

**本页涵盖：** 让 CMP 170HX 报告并使用超出出厂帧缓冲的精确机制：承载几何的两个寄存器、按 SKU 的值、为什么这两个寄存器必须一致、把报告的内存变成可分配内存的驱动侧管道，以及持久性规则。物理基板（HBM 堆栈、分区、筛选、带宽）见 [内存子系统](../hardware/memory-subsystem.md)。

头条结论在此陈述一次，全文反复提及，因为搞混它是这个领域最常见的错误：

> **8 GB CMP 170HX（`10de:20c2`）解锁到 64 GB。10 GB CMP 170HX（`10de:2082`）解锁到 40 GB。** 绝不反过来。10 GB 卡的 80 GB 配置尝试过，发现实际使用超过约 40 GB 就不可用。

正式解锁器中整个显存几何机制就是**两次主机寄存器写入**：

| 寄存器 | 地址 | 作用 |
|---|---|---|
| `NV_PFB_FBPA_CFG1`，广播 | `0x009a0204` | 逐分区寻址深度（容量*档*） |
| `NV_PFB_PRI_MMU_LOCAL_MEMORY_RANGE`（LMR） | `0x00100ce0` | MMU 对帧缓冲总大小的声明 |

解锁中的其他一切都服务于让这两次写入落盘、让 GSP-RM 和 CPU-RM 相信它们、并让所得空间可分配。

---

## 按 SKU 的值表

这是权威表。其中每个数字在正式 `common/constants.yaml`、`driver/build.sh`、`install.sh` 和 `driver/patches/0001-sec2-postbl-plm-ss-cfg.patch` 内的硬编码分支之间一致。

| 项目 | 8 GB 卡 `10de:20c2` | 10 GB 卡 `10de:2082` |
|---|---|---|
| 原始容量 | 8192 MiB | 10240 MiB |
| **解锁后容量** | **65536 MiB（64 GB）** | **40960 MiB（40 GB）** |
| 原始 CFG1 `0x009a0204` | `0x02449000` | `0x02449000`（相同） |
| **解锁后 CFG1** | **`0x02779000`** | **`0x02669000`** |
| 原始 LMR `0x00100ce0` | `0x00000208` | `0x00000288` |
| **解锁后 LMR** | **`0x0000020B`** | **`0x0000028A`** |
| `targetFbBytes` / GSP `fb_length` | `0x0000001000000000`（64 GiB） | `0x0000000A00000000`（40 GiB） |
| 元数据中的 `unlocked_mib` | 65536 | 40960 |
| 原始逐 FBPA `CSTATUS_RAMAMOUNT` | `0x200`（512 MiB） | `0x200`（512 MiB） |
| 解锁后逐 FBPA `CSTATUS_RAMAMOUNT` | `0x1000`（4096 MiB） | `0x800`（2048 MiB） |
| 活动 FBPA | 16 | 20 |
| CFG1 档字节 [23:16] | `0x77` | `0x66` |
| `install.sh` 横幅 | `Unlock geometry: 64GB (CFG1=0x02779000 LMR=0x0000020B)` | `Unlock geometry: 40GB (CFG1=0x02669000 LMR=0x0000028A)` |

第三个设备 ID `10de:20b0` 会被安装器的 `lspci` 扫描检测到，但**不会**被解锁：驱动内门控只接受 `0x20C2` 和 `0x2082`，因此 `20b0` 卡会构建并安装一个解锁永不触发的驱动。master `README.md` 仍说解锁由 `0x20C2` 门控；这种措辞已过时，代码在全部六个补丁中按两个 ID 门控。

几何在**运行时按 PCI 设备 ID 选择**，而非编译时。两种配置编译进同一个模块：

```c
NvU32 devId = pGpu->idInfo.PCIDeviceID >> 16;

if (devId == 0x20C2) { cfg1Value = 0x02779000U; lmrValue = 0x0000020BU; }  /* 8 GB  -> 64 GB */
else                 { cfg1Value = 0x02669000U; lmrValue = 0x0000028AU; }  /* 10 GB -> 40 GB */
```

---

## CFG1 编码什么

CFG1 编码**每个内存分区的寻址深度**。它不编码容量，也不编码堆栈数量。

档字节位于位 `[23:16]`。每个半字节是偏移 8 的行地址计数：

| 档字节 | 行位 | 每 FBPA 容量 | 完整 CFG1 字 |
|---|---|---|---|
| `0x44` | 12 | 512 MiB | `0x02449000`（原厂，两个 SKU） |
| `0x66` | 14 | 2048 MiB | `0x02669000`（10 GB 卡到 40 GB） |
| `0x77` | 15 | 4096 MiB | `0x02779000`（8 GB 卡到 64 GB） |

因此总容量是 `档 × 活动 FBPA 数量`，FBPA 数量由熔丝决定、CFG1 不碰。**同一个字 `0x02779000` 在 16 分区 8 GB CMP 上给出 64 GB，在 20 分区 A100 上给出 80 GB。** 它是逐分区 strap，不是逐卡 strap。

该寄存器的探测目录字段解码是 `SUBP[1:0]`、`COL[15:12]`、`ROWA[19:16]`、`BANK[25:24]`。在观察过的每个 HBM 部件上，`COL` 保持 `0x9`、`BANK` 保持 `0b10`；只有行地址半字节移动。GDDR6 部件读出不同的 `COL` 值（A10/A5000/RTX 3090/3090 Ti 是 `0x4266b000`，A6000 是 `0x4277b000`，RTX 3080/3080 Ti 是 `0x4266a000`），这就是为什么 `0x9` 半字节是内存类型常量而不是"5 堆栈"标志。

**这些不是魔法常量。** `0x02779000` 字面就是真实 A100 PCIe 80 GB 硅片（PCI `0x20b5`）上测得的原厂 CFG1 值，连同 LMR `0x0000028b`。`0x02669000` 同样是 A100 PCIe 40 GB 和 A100 SXM4 40 GB 上的原厂值。解锁恢复真实的 A100 几何。参考 GA100 读出 `0x22779000`，只在位 29 上不同。该位**把每 FBPA 寻址深度减半**：2026-07-27 一台被驱动到 `CFG1 = 0x22779000` 的 170HX 让逐 FBPA `CSTATUS_RAMAMOUNT` 保持 `0x800`（2048 MiB）而不是 `0x1000`，记录为 `tier=0x77 HALVED -> 2048 MiB/FBPA x20 = 40960 MiB (40 GB)`；对照的解锁 8 GB 卡在档 `0x77` 解锁后每个活动 FBPA 读出 `0x00001000`。位 29 是否还有其他作用没有定论。

同一个字也存在于 VBIOS。至少从 Pascal 起，内存类型/大小/厂商由硬件引脚 STRAP0 到 STRAP2 选择的 16 个内存配置 strap 之一决定。CFG1 strap 表是 16 个 4 字节条目，小端排布 `00 90 TT 02`，即 `u32 = 0x02TT9000`，与寄存器值逐位相同。正式 CMP 硬件上只填充 strap 4。见 [VBIOS](../hardware/vbios.md)。

---

## LMR 编码什么

`0x00100ce0` 的 MMU 本地内存范围寄存器把帧缓冲总大小声明为幅值/刻度对：

```text
size_MiB = LOWER_MAG[9:4] << LOWER_SCALE[3:0]
```

等价地 `bytes = MAG << (SCALE + 20)`。`MAG` 按 SKU 恒定，等于**活动 FBPA 数量的两倍**；`SCALE` 才是解锁改变的东西。

| LMR 值 | MAG | SCALE | 解码为 | 状态 |
|---|---|---|---|---|
| `0x00000208` | 32 | 8 | 8192 MiB | 原厂，8 GB 卡 |
| `0x00000288` | 40 | 8 | 10240 MiB | 原厂，10 GB 卡 |
| `0x0000020B` | 32 | 11 | 65536 MiB | **正式，8 GB 卡到 64 GB** |
| `0x0000028A` | 40 | 10 | 40960 MiB | **正式，10 GB 卡到 40 GB** |
| `0x0000028B` | 40 | 11 | 81920 MiB | 对 80 GB 正确；实验性地从脚本触发，从未发布 |
| `0x0000028C` | 40 | 12 | 163840 MiB | 一个 PRAMIN 运行真的接受的笑话值 |

解锁在 **8 GB 卡上给刻度半字节 +3**、**10 GB 卡上 +2**。按 CFG1 的说法，8 GB 卡的增量是 `+0x00330000`（位 16、17、20、21），10 GB 卡是 `+0x00220000`（位 17、21）。

> [!NOTE]
> **开放问题：`LOWER_MAG` 是 [9:4] 的 6 位还是 [10:4] 的 7 位？**
>
> 实际使用中的一切都按 6 位工作。宽度从未从 `dev_fb.h` 读过。这是头文件查找，不是实验，也是 `0x28B` 对 `0x50A` 之争与干净答案之间最后的一道坎。

---

## 为什么 CFG1 和 LMR 必须匹配

它们是同一事实的两份独立声明，GPU 会互相核对。

硬件上的一次受控三方对比：

| 配置 | 结果 |
|---|---|
| 完全不写内存 | CPU-RM 在 `0x24` 失败（`kbusVerifyBar2`） |
| 40 GB CFG1 strap 配原厂 10 GB LMR（`0x288`） | 仍然 `0x24` |
| 40 GB CFG1 **加匹配的 LMR**（`0x28A`） | 到达 `0x25`（StateLoad） |

没有任何配置能在没有 LMR 的情况下到达 StateLoad。**仅有 CFG1 不够；LMR 是硬性前置条件。**

**GSP-RM 在自己的启动期间把 LMR 视为主控。** CFG1 设为 40 GB 档但 LMR 保持 `0x288` 时，GSP-RM 在 `kgspBootstrap` 期间把 `CSTATUS` 从 `0x800` 恢复为 `0x200`。LMR 一致地设为 `0x28A` 时，仪器化转储在包括 post-Bootstrap 在内的全部四个检查点读到 `CSTATUS=0x800 LMR=0x28a CFG1=0x2669000 WprMeta.fbSize=0xa00000000`。FWSEC 本身不会恢复几何。

这正是未合并 `80` 分支撞上的失败：见下方 [80 GB 尝试](#the-80-gb-attempt-and-why-it-is-incoherent)。

---

## 正式写入序列

解锁由 `_kgspSec2PostblTimingEnabled()` 门控，它读取 `NvU32 devId = pGpu->idInfo.PCIDeviceID >> 16;`，对 `0x20C2` **或** `0x2082` 返回真。

### 第 1 步：打开四个 PLM

在 FB 几何权限级掩码打开之前，主机（PL0）对 CFG1 的写入**被静默丢弃**。早期流水线记录了三遍 `Write failed - wrote 0x2779000, read 0x2449000`，没有任何报错。原因是熔丝 `OPT_SECURE_FBPA_MEM_WR_SECURE`（`0x00820618`）= `1`，它把 FBPA 内存配置写入限制为特权代码。

正式 `plmTable[]` **恰好有四个条目**，按此顺序打开，每个经一轮 Booter Load 最多重试两次，保存的 WPR2 LO/HI 对在每次尝试前重写、循环后再写一次：

| 顺序 | 地址 | 写入的值 | 标签 |
|---|---|---|---|
| 0 | `0x001fa7cc` | **`0xfffff0ff`** | `WPR_CFG` |
| 1 | `0x009a0148` | `0xffffffff` | `FBPA` |
| 2 | `0x001fa7c4` | `0xffffffff` | `WPR` |
| 3 | `0x00823804` | `0xffffffff` | `FEAT` |

注意 WPR_CFG 目标是 `0xfffff0ff`，**不是** `0xffffffff`。README 和 `DEBUGGING.md` 说"所有 PLM 必须显示 `0xffffffff`"的文字是松散措辞。原厂值是 FEAT 和 FBPA 的 `0xffffff8f`、WPR 对的 `0x0004cb8f`。

如果某个 PLM 两次尝试都失败，驱动记录 `SEC2_DEBUG: FAILED to open <name> after 2 attempts` 并**无论如何继续**做几何写入。WPR2 LO/HI（`0x001fa824` / `0x001fa828`）只被保存和恢复；正式驱动从不把它们设为新值。见 [权限级掩码](privilege-level-masks.md) 和 [ROP 链](rop-chain.md)。

> [!NOTE]
> **命名有争议，地址没有**
>
> 寄存器目录工作把 `0x001fa7c4` 命名为 `NV_PFB_PRI_MMU_LOCAL_MEMORY_RANGE__PRIV_LEVEL_MASK`，即 LMR PLM，而正式 `plmTable` 把它标为"WPR"、把 `0x001fa7cc` 标为"WPR_CFG"。地址和值没有争议，功能结果相同。以地址为准。`0x001fa7c0` **不是** LMR PLM，在正式树中任何地方都不出现；把它放在多写链首位会使 ROP 链故障。

### 第 2 步：四次主机寄存器写入

```c
GPU_REG_WR32(pGpu, 0x0082381cU, 0x88888888U);  /* SS0: compute throttle off      */
GPU_REG_WR32(pGpu, 0x00823820U, 0x00000008U);  /* SS1: compute throttle off      */
GPU_REG_WR32(pGpu, 0x009a0204U, cfg1Value);    /* FBPA CFG1, BROADCAST alias     */
GPU_REG_WR32(pGpu, 0x00100ce0U, lmrValue);     /* MMU local memory range         */
```

CFG1 在 LMR **之前**写入。四个随后全部回读并打印：

```text
SEC2_DEBUG: POST-WRITE SS0=0x88888888 SS1=0x00000008 CFG1=0x02779000 LMR=0x0000020b (devId=0x20c2)
```

前两次写入是 [计算解锁](compute-throttle.md)，对两个 SKU 无条件发出。这里包含它们只是因为它们共享同一窗口。

### 第 3 步：仅广播

**正式驱动只向广播别名 `0x009a0204` 写 CFG1。** 它不遍历 `0x00900204 + n*0x4000` 的 24 个逐 FBPA 实例，仓库级 grep 所有补丁、脚本和 YAML 文件中这些地址返回零命中。因此任何读正式安装器 dmesg 的人只会看到一个 CFG1 值，而不是二十个。

这依赖上下文，区别很重要：

| 上下文 | 什么就够 |
|---|---|
| 驱动 / devinit 路径中（正式工具） | 向 `0x009a0204` 一次广播写入 |
| 无 devinit 的无驱动运行时（净室 refire 链） | 广播单独不会移动 CSTATUS；**必须手工写全部 24 个逐 FBPA CFG1 实例**，位于 `0x00900204 + n*0x4000`，通过读 `0x0090020C + n*0x4000` 的 `CSTATUS_RAMAMOUNT` 验证 |

五 PLM `FB_GEO_PLMS = [0x00100b10, 0x009a0148, 0x009a014c, 0x009a0008, 0x009a000c]` 清单和 24 实例循环属于**独立的未发布无驱动工具链**，不属于正式安装器。读任何文章时把两条路径分开。见 [工具谱系](../history/tool-lineage.md)。

> [!NOTE]
> **开放问题：广播是 PRI priv-ring 硬件机制吗？**
>
> 一位原本相信广播是 GSP-RM 中软件步骤的研究者划掉了这个信念，改提 priv ring，但它从未被直接仪器化。它很重要，因为它决定一次写入是在每个上下文中都保证足够，还是只在 devinit 跟随其后的情况下才够。实验：在 FB 几何 PLM 打开、无 devinit 的情况下只写广播，然后读全部 24 个逐 FBPA CFG1 镜像，看值是否即使 CSTATUS 不动也会传播。

### 第 4 步：重建原厂签名并启动 GSP

postbl 路径把 GSP 签名缓冲区替换为 `SEC2_POSTBL_TIMING_SIGNATURE_SIZE = 0x0000f800` 字节（63,488）的 ROP 载荷，填 dword `0x000004a7`，在固定偏移（`0x1100`、`0x5b40` 和 `0xf754` 到 `0xf7f8`，以 `0x00007f2f` 结束）覆盖。如果 `/lib/firmware/nvidia/ga100/gsp/dmem.bin` 存在则加载它；缺失则用带单次默认写入 `0x009a0148 = 0xffffffff` 的内置回退载荷，缺失报告为 `0x59`，是良性的。

GSP 启动前，`kgspSec2PostblTimingRebuildStockSignature()` 释放超大的载荷 memdesc、分配 `NV_ALIGN_UP(stockSignatureSize, 256)`、把保存的原厂签名复制回去，并更新 `pWprMeta->sysmemAddrOfSignature` 和 `sizeOfSignature`。如果重建失败，GSP 启动被中止。正是这一步让原本正常的 GSP-RM 启动落在已改变的几何之上。

然后重跑 `kgspPopulateWprMeta_HAL`，使 WPR 元数据匹配新帧缓冲：

| 字段 | 之前（8 GB 卡） | 之后（64 GB） |
|---|---|---|
| `fbSize` | `0x0000000200000000` | `0x0000001000000000` |
| `wprEnd` | `0x00000001fff00000` | `0x0000000ffff00000` |
| `wprStart` | （不适用） | `0x0000000ff7400000` |
| `heapOffset` | （不适用） | `0x0000000ff7500000` |
| `heapSize` | `0x0000000006900000` | `0x0000000006e00000` |
| 保存的 WPR2 | `lo=0x1ffffe00 hi=0x00000000` | 不变 |

在 10 GB 卡上对应的 `fbSize` 转变是 `0x0000000280000000` 到 `0x0000000a00000000`。

补丁 `0001` 还**无条件**把上游"unexpected WPR2 already up, cannot proceed with booting GSP"硬失败（`return NV_ERR_INVALID_STATE`）降级为警告（`WPR2 already up before GSP boot; continuing for recovery`），使遗留脏状态的卡仍能启动。

> [!CAUTION]
> **那个 WPR2 降级不按 CMP 设备 ID 门控**
>
> 它适用于补丁模块驱动的**每张 GPU**。在混合系统上，补丁模块会在无关硬件上静默越过真正恶劣的 WPR2 状态。不要把这些模块装在你关心其他 GPU 的机器上。

---

## 让空间成真，而不只是被报告

只有寄存器几何，`nvidia-smi` 里只是一个数字。再有四个补丁把它变成 CUDA 可以分配的内存。

### GSP static info 与 FB 区域（补丁 `0001`）

`kgspInitRm` 之后，对 devId `0x20C2` 或 `0x2082`：

- `pGSCI->fb_length` 被 `targetFbBytes` 覆盖（8 GB 卡是 `0x0000001000000000`，10 GB 卡是 `0x0000000A00000000`）。
- 如果 `0 < numRegions <= NV2080_CTRL_CMD_FB_GET_FB_REGION_INFO_MAX_ENTRIES`，驱动取 `fbRegion[numRegions-1]`，当 `limit < targetFbBytes - 1` 时设置 `limit = targetFbBytes - 1`、`reserved = limit - base + 1`、`supportCompressed = NV_TRUE`、`supportISO = NV_TRUE`、`performance = 20`。

记录为 `SEC2_DEBUG: static-info BEFORE/AFTER: fb_length=... numRegions=...`。**没有它驱动不会上报加宽的大小**，因为启用 GSP 固件时 CPU-RM 不自己定帧缓冲大小：它通过 `GspStaticConfigInfo` 的 RPC 从 GSP-RM 接收 `fbSize`。

64 GB 启动时观察到：`static-info BEFORE: fb_length=0x1000000000 numRegions=5`，最后 `region[4] base=0xff7300000 limit=0xfffffffff reserved=0x8d00000`。

### 延迟 PMA 扩展（补丁 `0003`）

`memmgrSec2DebugLateExtendHighPmaRegion()` 在 GPU 初始化后从 `osinit.c` 调用，按两个设备 ID 门控。它就是把报告的内存变成**可分配**内存的步骤。它扫描 `Ram.fbRegion[]` 找满足 `bRsvdRegion && !bInternalHeap && limit >= stockFbBytes && base <= limit` 的最高 limit 区域，用 `base = NV_MAX(candidate->base, 0x200000000)` 和 `bSupportCompressed = NV_TRUE` 构建 `PMA_REGION_DESCRIPTOR`，并调用 `pmaRegisterRegion(pPma, numPmaRegions, NV_FALSE, &pmaRegion, 0, NULL)`。

成功时它拆分或取消保留候选区域：

- 如果 `candidate->base < 0x200000000`，追加一个新公开 FB 区域（`base = 0x200000000`、`limit` = 旧 limit、`rsvdSize = 0`、`bRsvdRegion = NV_FALSE`、`bInternalHeap = NV_FALSE`、`bSupportCompressed = NV_FALSE`），把原区域截断到 `limit = 0x1ffffffff`、钳制其 `rsvdSize`、递增 `numFBRegions`，然后调用 `memmgrRegenerateFbRegionPriority()`。
- 否则原地清除 `bRsvdRegion`、`rsvdSize`、`bInternalHeap` 和 `bSupportCompressed`。

提前退出：PMA 未初始化时 `NV_OK`（`SEC2_DEBUG_LATE_PMA: no PMA, skipped`）、范围为空时、或 `pmaIsPmaManaged()` 已覆盖时；需要拆分但 `numFBRegions >= MAX_FB_REGIONS` 时 `NV_ERR_INSUFFICIENT_RESOURCES`。

64 GB 卡上的实测效果：

```text
SEC2_DEBUG_LATE_PMA: registering candidate=6 base=0xff7300000 limit=0xfffffffff ... pma_region_id=1
SEC2_DEBUG_LATE_PMA: status=0x0 pma_total 0xfd8f50000->0xfe1c50000 pma_free 0xfd8f50000->0xfe1c50000
```

那个增量是 `0x8d00000` 字节 = 147,849,216 字节 = 恰好 **141.0 MiB**。"约 +136 MiB"和"约 141 MiB"都在流传；141.0 MiB 是这个增量，而约 136 MiB 的数字指 WPR 划分，那是另一回事。

解锁 64 GB 卡上的完整 FB 区域布局，七个区域：

| 区域 | 基址 | 上限 | 标志 |
|---|---|---|---|
| 0 | `0x0` | `0x1007ffff` | rsvd=1，rsvdSize `0x10080000` |
| 1 | `0x10080000` | `0xfe8fcffff` | rsvd=0 |
| 2 | `0xfe8fd0000` | `0xff42dffff` | rsvd=0，rsvdSize `0xb310000`，intHeap=1 |
| 3 | `0xff42e0000` | `0xff430ffff` | rsvd=1，intHeap=1 |
| 4 | `0xff4310000` | `0xff720ffff` | rsvd=1，rsvdSize `0x2f00000` |
| 5 | `0xff7210000` | `0xff72fffff` | rsvd=1 |
| 6 | `0xff7300000` | `0xfffffffff` | rsvd=1，rsvdSize `0x8d00000`，**扩展候选** |

同一启动上的堆汇总：

```text
SEC2_DEBUG_HEAP: fbAddrSpace=65536MB mapRam=0MB fbTotal=65536MB fbUsable=0xfe4260000
                 heapTotal=0x1000000000 regionBytes=0x1000000000 publicBytes=0xfd8f50000 numRegions=7
```

### BAR0/PRAMIN 钳制（补丁 `0004`）

在 `kern_bus_gm107.c` 中，对 devId `0x20C2` 或 `0x2082` 且 `Ram.fbAddrSpaceSizeMb > 0x2000`（8192 MB）时，`offsetBar0` 被强制为：

```c
offsetBar0 = (0x2000ULL << 20) - DRF_SIZE(NV_PRAMIN);
```

即 PRAMIN 窗口被钉回**原厂 8 GiB** 地址空间，而不是跟随扩大后的空间。这对任何用 PRAMIN 探测高物理内存的人都很重要：解锁后 PRAMIN 默认够不到新空间的顶部。补丁文件在 master 和全部四个驱动系列移植目录之间逐字节相同（md5 `8e6a2b1c03df6d3388243db82ebbb9b4`）。

### CE scrub 规避（补丁 `0005`，加 `0003` 中一个 hunk）

压缩在解锁卡上被刻意在三处禁用，全部按两个设备 ID 门控：

1. 在 `mem_mgr_tu102.c` 中，scrubber 的 PTE-kind 选择器返回 `NV_MMU_PTE_KIND_GENERIC_MEMORY` 而不是默认的 `NV_MMU_PTE_KIND_GENERIC_MEMORY_COMPRESSIBLE_DISABLE_PLC`。
2. 在 `mem_scrub.c` 中，CeUtils 门控变成 `if (memmgrUseVasForCeMemoryOps(pMemoryManager) && ((pGpu->idInfo.PCIDeviceID >> 16) != 0x20C2 && (pGpu->idInfo.PCIDeviceID >> 16) != 0x2082))`，让 CeUtils 保持物理而非虚拟模式。
3. 在 `mem_mgr.c` 中（补丁 `0003` 的第三个 hunk），同一个设备 ID 排除被加到由 `bUseRawModeComptaglineAllocation` / `bOneToOneComptagLineAllocation` 门控的 `ceUtilsParams.flags |= DRF_DEF(0050_CEUTILS, _FLAGS, _VIRTUAL_MODE, _TRUE)` 路径。

没有这些，复制引擎 scrubber 会在加宽空间里绊到压缩分配。

### 持久软件状态（补丁 `0006`）

`NV_FLAG_PERSISTENT_SW_STATE` 在 `nv.c` 中对两个设备 ID 设置，位于两个主体相同的独立 `if`/`else if` 分支中。

### 正式代码中的一个真实不对称

`stockFbBytes = 0x200000000ULL /* 8GB */` 在补丁 `0001` 和 `0003` 中都被硬编码，并用于**两个**设备 ID，包括真实原厂大小为 `0x280000000` 的 10 GB 卡。PRAMIN 钳制同样对照 `0x2000` MB。在补丁 `0001` 中该变量被声明但从未引用（死代码）；在补丁 `0003` 中它是原厂区域与延迟 PMA 扩展区域之间的拆分点。

> [!NOTE]
> **开放问题：8 GiB 的 `stockFbBytes` 在 10 GB 卡上有影响吗？**
>
> 解锁在 `0x2082` 上可证明地工作，因此任何影响都是微妙的。检查纯属读日志：在 40 GB 解锁的 10 GB 卡上读 `SEC2_DEBUG_LATE_PMA: region[...]` 和 `SEC2_DEBUG_HEAP:` dmesg 行，确认 `publicBytes` 覆盖完整 40 GiB，而不是丢失 8 到 10 GiB 的切片。没人贴过。

---

## 时序：整个过程约一秒钟

来自 8 GB 卡升级到 64 GB 的完整 dmesg 捕获：

```text
11.13 s  stock signature saved
11.32 s  PLM[0] WPR_CFG
11.50 s  PLM[1] FBPA
11.68 s  PLM[2] WPR
11.86 s  PLM[3] FEAT              (about 180 ms per Booter pass)
11.86 s  POST-WRITE and WPR-meta update
12.07 s  normal BooterLoad status=0x0
         POST-BooterLoad verify PLM=0xffffffff SS0=0x88888888 SS1=0x00000008
                                CFG1=0x02779000 LMR=0x0000020b
12.64 s  heap creation
12.72 s  late PMA extension status=0x0
```

注意每次重触发运行无论成功与否 `BooterLoad` 都报告 `status=0xffff`；回读是唯一判定。

---

## 持久性：几何对比计算

这是显存解锁最重要的操作性属性，也是计算解锁先于显存解锁发布的原因。

| 事件 | 计算（SS0 `0x0082381c`、SS1 `0x00823820`、FEAT PLM `0x00823804`） | 显存几何（CFG1、逐 FBPA CFG1、CSTATUS、LMR、FB 几何 PLM、AON LMR 影子 `0x001180f0`） |
|---|---|---|
| 卸载并重载驱动，无 SBR | 存续 | **存续** |
| 功能级复位（FLR） | **存续**（常开岛） | **恢复** |
| 重启 / 断电 | 恢复 | 恢复 |
| DEVINIT | 恢复 | 恢复 |

10 GB 卡上一次 FLR 的实测：CFG1 `0x9A0204` 写入 `0x2779000` 恢复到 `0x2449000`；LMR `0x100CE0` 写入 `0x20b` 恢复到 `0x288`；而 SS0 `0x82381C` = `0x88888888` 和 SS1 `0x823820` = `0x8` 都存续。SEC2 复位 PLM 污染也被 FLR 清除（`0x8f` 到 `0xff`）。

几何**确实**经得起无 SBR 的卸载重载：卸载后寄存器仍读 `0x009a0204` = `0x02669000` 和 `0x00100ce0` = `0x0000028a`，新加载在 610.43.03 上再次枚举出 40960 MiB。

### 没有 PLM 能把几何移入常开域

一次 **11 PLM 的 HS 内几何存续性扫描**穷尽地定案了这一点。对全部 11 个候选，HS 内 FLR 前状态相同（`CFG1=0x2669000 CSTATUS0=0x800 LMR=0x28a amap=0x200404 resetPLM=0x8f`），每个 FLR 后读数都恢复到 `CFG1=0x2449000 CSTATUS0=0x200 LMR=0x288 amap=0x280404`。只有 `0x008200fc`、`0x00823800`、`0x00823804` 和 `0x00823b00` 保持打开（`PLM=0xffffffff`，AON = 是）；`0x00100b10`、`0x00100b38`、`0x009a0148`、`0x009a014c`、`0x009a0008`、`0x009a000c` 和 `0x00118128` 都重新锁定到 `0xffffff8f`。卡恢复到 `boot0=0x170000a1 resetPLM=0xff`。扫描的标题是"welp, no dice"。

**这就是正式设计在每次加载模块时于 GSP 启动路径内重新应用几何、而不是刷入或闩锁永久状态的结构原因。** 正式工具打开的四道 PLM 中只有 FEAT（`0x00823804`）经得起 FLR，因此 FLR 后从主机侧发出的 CFG1/LMR 写入会被阻止。几何写入必须与 PLM 打开发生在同一个无 FLR 窗口内。

### 推论：驱动不能静默撤销它

GA100 上 `kmemsysReadUsableFbSize_GP102` 是只读的，开源 CPU 侧 RM 从 LMR `0x00100ce0` 而不是 L2 amap 计算 `fbSize`。一个"原厂 RM 在启动时读 `0x44` 档、把 CFG1/CSTATUS 重写回原厂档并重新锁定 FB 几何 PLM"的假说被**测量驳斥**：完整干净驱动启动后、以及无 SBR 卸载重载后，`0x009a0204` 仍读 `0x02669000`、`0x00100ce0` 仍读 `0x0000028a`。观察到的恢复都追溯到操作者触发的 FLR。

---

## 验证它落地

按可信度排序，最可信的放最后：

1. **`nvidia-smi --query-gpu=memory.total` 什么也证明不了。** 驱动可以被补丁成打印任何数字而内存悄悄折叠。多人正是被这一点误导，包括一个在 64 GB 几何之上从 `nv-linux.c` 把 `fb_size` 强推到约 80 GB 的早期流程。
2. **寄存器回读。** `0x0090020C + n*0x4000` 的 `CSTATUS_RAMAMOUNT` 是最便宜可靠的检查：40 GB 档为 `0x800`，64 GB 档为 `0x1000`。预期被筛选掉的分区返回 `0xbadf20NN`，因此 10 GB 卡 24 中 20、8 GB 卡 24 中 16 才是正确的全过结果，而不是 24 中 24。无驱动链用 `CSTATUS == 0x800` 作为验证谓词。
3. **密集折叠测试。** 写入每个页面自己的索引，回读每个页面。如果地址线缺失，内存折叠、两个地址持有相同数据，而传统 memtest 对同一区域写读不会抓住。

> [!CAUTION]
> **不要对折叠测试做稀疏采样**
>
> 稀疏探测（每 N MB 一个词）在折叠卡上产生**假阴性**，因为折叠在通道交错偏移上别名，而不是相同字节偏移：`LOW[0]` 映射到 `40GiB + interleave`，而不是 `40GiB + 0`，因此稀疏测试写入一个伙伴、检查另一个地址。参考检查器分配全部空闲 VRAM 减 2 GiB，通过 PTX 内核写每个 64 KiB 页自己的索引，回读每一页，真实退出 0、折叠退出 1。还要先写完所有数据再回读全部：交错读写会引起争用，大约 48 GB 起变得非常慢。并且边跑边驱逐 L2，否则读由缓存服务。任何在采用驱逐方法之前得到的折叠结果都应丢弃。测试期间 `SIGKILL` 一个活 CUDA 内核可能以 **Xid 45** 卡死显卡并被迫复位循环。

一次成功的 8 GB 到 64 GB 解锁呈现为：

```text
NVIDIA-SMI 610.43.03   Driver Version: 610.43.03   CUDA Version: 13.0
NVIDIA CMP 170HX        0MiB / 65536MiB      34W / 250W     42C     P0
```

CUDA 经 ctypes 返回 `cuInit` 0、`cuDeviceGetCount` 1、`cuDeviceGetName` `NVIDIA CMP 170HX`、`cuDeviceTotalMem` 64.0 GB、属性 75/76 给出计算能力 8.0。卡名也可能显示 `Unknown`，这是正常的。

> [!WARNING]
> **`clocks.max.sm = 1935 MHz` 是报告字段，不是可达频率**
>
> 它出现在与容量相同的 `nvidia-smi` 查询中，常被当作 64 GB 签名的一部分引用。VBIOS 表最大图形时钟是 1695 MHz，实际硅片天花板在 +350 偏移下约为 1604 到 1614 MHz。持续 SM 频率标称 1410 MHz（`-pl 300` 下 1470 MHz）。把 1935 MHz 视为低置信度，见 [计算节流](compute-throttle.md)。

存档窗口结束时的稳定性结论：**8 GB 到 64 GB 稳定且生产使用中；10 GB 到 40 GB 稳定；10 GB 到 80 GB 报告大小但在约 40 GB 以上不可用。** 一台 64 GB 卡约一小时后以零错误通过 `gpu_burn`，多位独立拥有者复现，频道内没有 8 GB 卡 64 GB 解锁失败的案例。一台 10 GB 卡在 40 GB 下通过了 5 分钟 `gpu-burn`、零不匹配的 30 GiB CUDA 写/回读烧机，以及无折叠的 37 GiB 带标签自驱逐折叠测试。

---

## 打包：构建如何选择配置

`install.sh detect_card_profile()` 是在 `nvidia-smi --query-gpu=memory.total` 上的四档阶梯，而不是 PCI ID：

| 报告的 MiB | 配置 | 原因 |
|---|---|---|
| `>= 60000` | `8gb` | 已解锁的 64 GB 卡，重装会选择同一配置 |
| `>= 35000` 且 `< 60000` | `10gb` | 已解锁的 40 GB 卡 |
| `7680` 到 `8704` | `8gb` | 原厂窗口，为保留 FB 预留 ±512 MiB 容差 |
| `9728` 到 `10752` | `10gb` | 原厂窗口 |
| 其他 | 失败 | 打印 `unknown:<mib>` 并告诉你传 `--profile=8gb` 或 `--profile=10gb` |

如果 `nvidia-smi` 缺失或返回非数字，函数立即返回 1。

**`driver/build.sh` 不读 `common/constants.yaml`。** 树中没有任何脚本、补丁或 Makefile 读该文件。构建脚本在 bash `case` 中按配置硬编码自己的 CFG1/LMR/fb_bytes，然后对 `kernel_gsp.c` 运行一个 Python 重写器。因为 master 的补丁 `0001` 已包含重写器检查的全部六个标记（`..._8GB_PCI_DEVICE_ID`、`..._10GB_PCI_DEVICE_ID`、`0x02779000U`、`0x02669000U`、`0x0000001000000000ULL`、`0x0000000A00000000ULL`），重写器总是提前退出并打印 `runtime device-id geometry (profile metadata=<label>)`。**在 master 上，构建时几何重写从不触发。** bash 配置只影响标签、预期大小消息和 `card_profile` 标记文件。

`constants.yaml` 是文档。它的内容碰巧在 master 上正确，因此从未发布错误值，但该文件没有权威性，单独编辑它不会改变任何编译产物。读分支时任何人都必须查 `build.sh` 和补丁，而不是 YAML。

安装输出路径：`/lib/modules/$(uname -r)/updates/cmpunlocker/{driver_version,card_profile,unlock_geometry}`。卸载是 `sudo ./remove.sh --yes`；正式仓库中没有 `uninstall.sh`。

---

## 80 GB 尝试，以及它为什么不自洽 {#the-80-gb-attempt-and-why-it-is-incoherent}

> [!CAUTION]
> **`80` 分支未合并、不稳定、内部自相矛盾**
>
> 记录在这里是为了让发现它的人理解它实际编程了什么。不要在你需要的卡上运行它。

未合并的 `80` 分支瞄准让 10 GB 卡达到 81920 MiB。两个提交：`02ce75c` "Trying an 80GB unlock instead of 40GB" 和 `3c53aca` "Correct LMR for 80GB"。它的 README 声称 "Memory geometry (64GB on 8GB cards, 80GB on 10GB cards) | Working ✓"。该主张是假的，一两天内就被分支自己的测试者证伪。把它记录为文档缺陷，而不是结果。

**它实际的不同点：** 恰好一个补丁文件、两行。补丁 `0002` 到 `0006` 与 master 逐字节相同。`0001-sec2-postbl-plm-ss-cfg.patch` 只在 `cfg1Value = 0x02669000U` 变为 `0x02779000U`、`targetFbBytes ... 0x0000000A00000000ULL` 变为 `0x0000001400000000ULL` 上不同。加 `build.sh`、`install.sh` 和 `constants.yaml`。（"没有补丁文件与 master 不同"的说法是错的。）

**它实际编程了什么**，逐行代码验证：

| 层 | 值 | 解码为 |
|---|---|---|
| CFG1 `0x009a0204` | `0x02779000`（档 `0x77`） | 每 FBPA 4096 MiB × 20 活动 = **81920 MiB** |
| LMR `0x00100ce0` | `0x0000028A` | 40 << 10 = **40960 MiB** |
| `targetFbBytes` / GSP `fb_length` | `0x0000001400000000` | **80 GiB** |

这是一个**三方不一致**，按上面的 CFG1/LMR 一致性规则，与 2026-07-13 实验让 GSP-RM 恢复几何的 CFG1/LMR 失配属于同类。那个实验用了不同的一对——CFG1 `0x02669000` 配原厂 `0x288` LMR——因此机制类似而非相同。

`80/common/constants.yaml` 确实携带 `lmr: "0x0000028B"` 和 `unlocked_mib: 81920`，但 `build.sh` 从不读该文件。`80/driver/build.sh` 第 93 行设置 `LMR="0x0000028A"`，`80/install.sh` 第 138 行打印 `Unlock geometry: 80GB (CFG1=0x02779000 LMR=0x0000028A)`，分支的补丁 `0001` 烘入 `lmrValue = 0x0000028AU`。构建时重写也被短路：双设备门控测试 `0x02779000U`、`0x0000020BU`、`0x0000028AU`、`0x0000001000000000ULL` 和 `0x0000001400000000ULL`，全部存在，因此 Python 重写器在替换任何东西之前就退出。**提交 `3c53aca` "Correct LMR for 80GB" 只改了惰性元数据。** 每个在提交前后运行过 `80` 分支的测试者编程的都是 CFG1 `0x02779000` + LMR `0x0000028A` + `fb_length` 80 GiB。

该分支还加了 master 没有的安装器阶梯 `if (( mem_mib >= 75000 )); then echo "10gb"`，放在 `>= 60000` 测试之前，因为否则 81920 MiB 卡会被重新检测为 8 GB 卡。master 没有该阶梯，因为 master 从不产出 81920 MiB 卡。

**失败签名很精确，但它的每个组成部分都来自单一报告者，没有任何一个被独立复现。** `nvidia-smi` 显示约 81920 MiB，CUDA 报告 `global memory size=85545582592` 字节（79.67 GiB）。77 GiB 的 `cudaMalloc` 和 `cudaMemset` 都成功。一位测试者报告 `cuda_memtest` 在重启后立即完成一次、之后每次运行都失败，除非分配被限制在 39 GB，否则在 `Attached to device 0 successfully.` 后挂起；第二位操作者在同样测试后看到 Xid 154，但对第一位操作者的错误明确说"我不知道[他们]遇到了什么错误"。触及约 40 GB 以上内存的内核导致致命 GPU 丢失，通常需要完整重启。报告的 Xid 码包括 Xid 31（被描述为无害）和 CUDA 内存测试后的 Xid 154；主导报告症状是挂起。Xid 31 单独被旁观者建议，未被故障卡操作者佐证为*那个*签名。一位测试者的模型加载在 20 GB 以上失败，另一位在 40 到 60 GB 区间失败。失败**与功耗上限无关**。一个小组报告在 80 GB 的 `gpu-burn` 运行中有 2,796 个错误，而同一张卡在 40 GB 下运行干净。该配置每次驱动加载也只能工作一次，而且在至少一台系统上需要冷断电而不是驱动重载才能再次触发。

注意折叠边界恰好落在 **40 GiB**，与该分支实际编程的 LMR 匹配。下面的无驱动结果让这个匹配看起来是因果的而非巧合。

> [!WARNING]
> **实验性：一致三元组被触发过，折叠消失了**
>
> 一致集合**不是**通过重建分支达到的。它是在 2026-07-23 到 2026-07-27 之间由一个净室 refire 脚本达到的，记录 `CFG1=0x02779000 LMR=0x0000028b CST=20/24 resetPLM=0x00ff`，L2 解码 `0x10000300`，GSP-RM 和 CPU-RM 下都报告 81920 MiB。随后的一次密集带标签写/回读在 77.5 GiB 上返回 310 块中 310 块正确，**无折叠**，之后一次运行在原厂启动时序下到达 72 GiB。限制是真实的：每次触发约一个 CUDA 上下文后 Xid 154、边界以上约 79% 峰值带宽、顶部约 2 GiB 未测试，且只有两位操作者。它不发布，也不是安装路径。
>
> **正式 master 给 10 GB 卡 40 GB，40 GB 是受支持的配置。** 在 `driver/patches/0001-sec2-postbl-plm-ss-cfg.patch` 中把 `lmrValue = 0x0000028BU`、并在 `driver/build.sh`（不只是不被读取的 `constants.yaml`）中把 `LMR="0x0000028B"` 后重建分支，仍未尝试，而这正是能告诉你驱动能否承载触发脚本所能承载几何的方法。见 [80 GB 问题](../frontier/80gb.md)。

中间档也从未被尝试。逐通道档是粗粒度的（512 / 2048 / 4096 MiB），因此 10 GB 卡上的 48、56 或 64 GB 无法单靠档位到达；它需要 CFG1 钉在档 `0x77`、驱动可见大小通过 `targetFbBytes` 和延迟 PMA 区域上限钳制。在频道内直接问 10 GB 卡能否解锁到稳定的 60 GB，答案是"我不相信有人试过"。

---

## 值得知道的死路

- **刷不同 VBIOS。** "16 GB" 170HX 镜像（TechPowerUp 239457）和 10 GB 镜像（268984）都刷到了接受未签名和不匹配 ROM 的工程样品 GA100 上。两种情况下板仍报告 8 GB。容量不是加载哪个 ROM 的函数；它跟随 strap 选择的 CFG1 字。另外，把 8 GB VBIOS 刷到 10 GB 卡上让卡无法启动，归因于设备 ID 不匹配。
- **MAC 伪造的 VBIOS 显存解锁。** 它需要在 `0x41D53`（250 W 170HX）或 `0x41F53`（300 W）翻转一个字节。把 `44` 翻到 `66` 只能到 40 GB 几何；到 64 GB 需要 `44` 翻到 `77`。没有实现 MAC 伪造，字节级映射置信度中等、从未实测。
- **L2/LTC amap `0x0017e22c` 作为大于 10 GB 的门。** 这是团队一个多星期的根因工作模型。写下的当天就被证伪：一次运行在 `0x17e22c` 全程保持其原生 `0x00280404`、从未编程的情况下达到真实 40 GB。正式驱动在 master 或 12 个未发布分支快照的任何补丁中**完全不含** `0x17Exxxx` 地址。声称正式 `plmTable` 写 LTC 解码簇（`0x17E2B4`/`A0`/`E4`/`FC`）就是假的。
- **"神秘 PLM" `SEC2_DEBUG_PRI_FBPA_CFG1 0x009a0204`。** `0x009a0204` 是 CFG1 数据寄存器本身，不是 PLM。FBPA PLM 是 `0x009a0148`。
- **一秒寄存器重应用循环。** 一个第三方提交每秒重写几何寄存器。两位资深评审者和一位从未需要它的测试者否决了它：该循环与驱动争抢计算重定时，与显存解锁毫无关系。
- **`ecc` 分支。** 不含 ECC 代码。单一提交，"Fixed dual geometry support"，补丁目录与 master 逐字节相同。
- **"LMR 在 `0x1183A4`。"** 那是 GP102 的本地内存范围正确位置。验证过的 GA100 地址是 `0x00100ce0`。
- **半字节移位的抄录。** `0x26690000`、`0x27790000` 和 `0x24490000` 都在聊天中流传。验证过的形式是 `0x02669000`、`0x02779000` 和 `0x02449000`。
- **`docs` 分支。** 它把 CFG1/LMR 表弄对了，但发明了一个缩写，把 LMR 展开为"LM Request"。寄存器是 `NV_PFB_PRI_MMU_LOCAL_MEMORY_RANGE`，Local Memory **Range**。同一分支还把 SS0/SS1 误述为 `0xffffffff`/`0xffffffff`；正确值是 `0x88888888` 和 `0x00000008`。

完整目录见 [死路](../history/dead-ends.md)。

---

## 到达同一寄存器的替代路线

正式安装器是补丁内核模块，但不是唯一被演示过的路径。

> [!WARNING]
> **实验性：无驱动可重触发 ROP 链**
>
> 一条可重入 SEC2 ROP 链在硬件上被证明能让 CFG1 = `0x02669000` 存续进入干净、未修改的 GSP-RM 驱动启动，无 FLR、无 SBR（`BooterLoad 0x0`）。两个使能技巧：每次触发把有用写入与一次 WPR2_LO teardown 配对（`0x1fa824` 设为 `0x1ffffe00`、`0x1fa828` 设为 `0x00000000`，即起始大于结束的空区域），因为每次 booter 运行都重新划分 WPR2，保持 up 会污染下一次溢出；以及 ACR 互斥锁经干净 `0x7f2f` 尾释放，留下 `resetPLM = 0xff`。就绪谓词：广播 CFG1 等于目标**且** SEC2 复位 PLM（BAR0 `0x8403c4`，falcon 偏移 `0x3c4`）读出 `0xff`。这条链中，WPR2_HI 必须作为**最后一次**触发清除，在主机 CFG1 写入之后，因为 `kgspIsWpr2Up_HAL` 读 WPR2_HI 的 VAL 字段，否则原厂驱动会以 `NV_ERR_INVALID_STATE` 退出。这个工具链不在正式仓库中。

> [!WARNING]
> **实验性：未修改驱动的 Python 解锁器**
>
> 一个在驱动加载前运行的脚本在相反结论被定为不可能后的第二天被演示：无需补丁 `.ko`、无需安装器。原则上它是最干净的已知路线。它不在正式树或任何存档分支中，因此正式产品仍是补丁模块路径。

> [!NOTE]
> **泄露的概念验证有何不同**
>
> 泄露包在 GSP-RM 加载前直接修补主机内存中的 `WprMeta` 结构，这就是它必然随附修改过的开源内核模块的原因。净室方法改为打开相关 PLM 并直接写几何寄存器。中等置信度：这是持有两个工件的人陈述的，未被独立重新推导。

驱动补丁之前的历史社区 stage-1 poke 集是五次写入：`0x009A0204` = `0x02669000`、`0x00100CE0` = `0x0000028A`、`0x00823804` = `0xFFFFFFFF`、`0x0082381C` = `0x88888888`、`0x00823820` = `0x00000008`。生成器在单次溢出中交付最多五次写入，并用无害的 `(0x000014A0, 0)` 条目把更短的清单填充到恰好五个，使退出帧偏移保持不动。正式驱动在一个方面不同：它通过一轮 Booter 打开 `0x00823804`，而不是从主机直接写。

---

## 另见

- [内存子系统](../hardware/memory-subsystem.md)，物理分区与档位
- [权限级掩码](privilege-level-masks.md)
- [ROP 链](rop-chain.md) 和 [Falcon 与 Booter](falcon-and-booter.md)
- [驱动补丁](driver-patches.md)，按顺序的全部六个补丁
- [计算节流](compute-throttle.md)，同一窗口的 SS0/SS1 半边
- [寄存器参考](register-reference.md)
- [验证](../procedures/verify.md) 和 [故障排查](../procedures/troubleshooting.md)
- [80 GB 问题](../frontier/80gb.md)
- [词汇表](../start/glossary.md)
