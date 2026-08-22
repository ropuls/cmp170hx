# 六个驱动补丁，逐个解析

## 本页涵盖

正式 CMP 170HX 解锁不是固件刷写、不是用户空间守护进程、也不是二进制修改。它是**六个编号补丁文件**，应用于 NVIDIA `open-gpu-kernel-modules` 610.43.03 或 610.43.02 版本的未修改检出，本地编译，安装为五个内核模块。本页逐个走查每个补丁：它改什么、在哪个源文件、为什么需要、以及去掉它会怎样失败。

整个解锁（权限级掩码打开、计算节流写入、显存几何写入）都在**补丁 0001** 中，位于 `src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c`。其他五个补丁的存在是因为突然声称拥有 64 GiB 帧缓冲的 GA100 会打破驱动中若干下游假设：致命断言触发、PRAMIN 窗口落出范围、物理内存分配器永远不知道新内存、复制引擎 scrubber 选错 PTE kind、RM 在客户端之间拆除软件状态。0002 到 0006 各自修复其中恰好一个。

如果本页只记一条命令：

```bash
sudo dmesg | grep SEC2_DEBUG
```

补丁集发出的几乎每一行都带那个前缀。唯一例外是补丁 0001 的降级 WPR2 警告 `WPR2 already up before GSP boot; continuing for recovery`，它按 `LEVEL_WARNING` 打印、没有 `SEC2_DEBUG:` 标签。

---

## 补丁系列一览

按文件名顺序、`patch -p1` 应用于一份新解压的原厂树。

| # | 文件 | 字节 | 行 | 主要源文件 | 作用 |
|---|---|---|---|---|---|
| 0001 | `0001-sec2-postbl-plm-ss-cfg.patch` | 19,278 | 463 | `gpu/gsp/kernel_gsp.c` | 漏洞利用：打开 PLM、写 SS0/SS1/CFG1/LMR、加宽 `fb_length` |
| 0002 | `0002-booter-verify.patch` | 3,901 | 87 | `gpu/gsp/arch/turing/kernel_gsp_tu102.c` | 软化四个致命断言，打印启动后回读证明 |
| 0003 | `0003-late-pma.patch` | 10,317 | 263 | `gpu/mem_mgr/mem_mgr.c`、`nvalloc/unix/src/osinit.c` | 把新内存注册给物理内存分配器 |
| 0004 | `0004-bar0-pramin-clamp.patch` | 841 | 20 | `gpu/bus/arch/maxwell/kern_bus_gm107.c` | 让 PRAMIN 窗口保持在可达 BAR0 空间内 |
| 0005 | `0005-ce-scrub-workarounds.patch` | 1,604 | 38 | `mem_mgr/arch/turing/mem_mgr_tu102.c`、`mem_mgr/mem_scrub.c` | 用普通 PTE kind 强制 scrubber 进入物理模式 |
| 0006 | `0006-persistent-sw-state.patch` | 584 | 19 | `kernel-open/nvidia/nv.c` | 在 PCI probe 设置持久软件状态标志 |

总计：**六个补丁 36,525 字节**，触及**十个不同源文件**。行数含 diff 头。字节数是仓库 blob 大小，LF 行尾。用 `core.autocrlf=true` 在 Windows 上检出的树会把每个文件按每行一字节膨胀，这就是常被引用的 37,415 字节总数来源。

该系列触及的完整文件清单：

```text
src/nvidia/generated/g_kernel_gsp_nvoc.h
src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c
src/nvidia/src/kernel/gpu/gsp/arch/turing/kernel_gsp_tu102.c
src/nvidia/arch/nvalloc/unix/src/osinit.c
src/nvidia/inc/kernel/gpu/mem_mgr/mem_mgr.h
src/nvidia/src/kernel/gpu/mem_mgr/mem_mgr.c
src/nvidia/src/kernel/gpu/bus/arch/maxwell/kern_bus_gm107.c
src/nvidia/src/kernel/gpu/mem_mgr/arch/turing/mem_mgr_tu102.c
src/nvidia/src/kernel/gpu/mem_mgr/mem_scrub.c
kernel-open/nvidia/nv.c
```

`driver/build.sh` 下载 `https://github.com/NVIDIA/open-gpu-kernel-modules/archive/refs/tags/${VERSION}.tar.gz`，缓存在 `driver/.build/` 下，每次运行删除并重新解压一棵干净树，然后循环：

```bash
for p in "${patches[@]}"; do patch -p1 < "${p}"; done
```

脚本在 `set -euo pipefail` 下运行，因此单个被拒 hunk 会中止构建。因为循环是对 `driver/patches/*.patch` 的普通 glob，作为 `0007-*.patch` 放入的第三方 diff 会与系列干净组合。该情况唯一记录在案的例子见 [P2P](../frontier/p2p.md)。

构建并安装五个模块到 `/lib/modules/$(uname -r)/updates/cmpunlocker/`：`nvidia.ko`、`nvidia-modeset.ko`、`nvidia-uvm.ko`、`nvidia-drm.ko`、`nvidia-peermem.ko`。只有 `nvidia.ko` 携带解锁代码；其余四个是原厂重建，存在是为了让整组有匹配的 `srcversion`。

---

## 六个补丁共用的设备门控

每个解锁点都把 PCI 设备 ID 的**高 16 位**对照两个已知 170HX SKU 测试。补丁 0001 在 `kernel_gsp.c` 添加共享辅助函数：

```c
#define SEC2_POSTBL_TIMING_CMP_170HX_8GB_PCI_DEVICE_ID   0x20C2
#define SEC2_POSTBL_TIMING_CMP_170HX_10GB_PCI_DEVICE_ID  0x2082

static NvBool _kgspSec2PostblTimingEnabled(OBJGPU *pGpu)
{
    NvU32 devId = pGpu->idInfo.PCIDeviceID >> 16;
    return (devId == SEC2_POSTBL_TIMING_CMP_170HX_8GB_PCI_DEVICE_ID ||
            devId == SEC2_POSTBL_TIMING_CMP_170HX_10GB_PCI_DEVICE_ID);
}
```

值得内化的后果：

- 任何其他 GA100 SKU，包括同一机箱里的 A100，在安装补丁模块的情况下走完全原厂路径。补丁在它上面是惰性的。
- `install.sh` 用 `lspci` grep `10de:20b0|10de:20c2|10de:2082`，因此 `20b0` 卡能成功安装、然后永不解锁，因为驱动内门控没有列出 `0x20B0`。安装器也这么说：`This card reports 0x${DEVID}; install will continue, but unlock may not activate.`
- 补丁 0006 是 `>> 16` 形式的唯一例外。它在 `kernel-open/nvidia/nv.c` 的 PCI probe 运行，那时 `OBJGPU` 还不存在，因此比较原始 `nv->pci_info.device_id`。

几何在**运行时从设备 ID 选择**，而非安装时。两种配置都烘进补丁 0001，因此安装器的 `--profile=` 只改打印横幅和元数据文件。见 [显存几何](memory-geometry.md)。

---

## 0001 `sec2-postbl-plm-ss-cfg`：漏洞利用

这才是补丁。其他都是支撑。它 19,278 字节、约 460 个 diff 行，做七项不同的修改。

### 1. GSP 对象上的两个新字段

插入 `src/nvidia/generated/g_kernel_gsp_nvoc.h`，紧跟 `MEMORY_DESCRIPTOR *pSignatureMemdesc;` 之后，hunk 锚点 `@@ -544,6 +544,8 @@`：

```c
NvU8 *pStockSignatureData;
NvU64 stockSignatureSize;
```

真实 GSP 固件签名在载荷覆盖缓冲区之前被复制进这些字段，记录为 `SEC2_DEBUG: saved stock signature (4096 bytes)`。

> [!CAUTION]
> **先恢复原厂 GSP 固件**
>
> 如果机器曾经跑过 cmpunlocker 的固件补丁前代方案，在安装驱动补丁前把 `gsp_tu10x.bin` 恢复到原厂。驱动把固件携带的任何签名保存为"原厂"。如果磁盘上的固件仍是补丁版，它会把**漏洞利用载荷**保存为原厂，之后干净的 GSP-RM 启动会 DMA 错误的 ROP 链。
>
> ```bash
> GSP_DIR=/lib/firmware/nvidia/610.43.03
> sudo cp $GSP_DIR/gsp_tu10x.bin.cmpunlocker.bak $GSP_DIR/gsp_tu10x.bin
> ```

### 2. 超大签名缓冲区

`_kgspCreateSignatureMemdesc` 通常分配 `NV_ALIGN_UP(pGspFw->signatureSize, 256)` 字节，实践中为 4096。设备门控为真时它改为在 `ADDR_SYSMEM` 中分配 `SEC2_POSTBL_TIMING_SIGNATURE_SIZE = 0x0000f800ULL`（63,488 字节），按 256 字节对齐，因为 SEC2 Booter 的 DMA 引擎要求。

那个超大缓冲区就是整个漏洞利用载体：签名 Booter 用无界 DMA 读它，砸掉自己的栈。机制在 [ROP 链](rop-chain.md) 和 [Falcon 与 Booter](falcon-and-booter.md) 中讲述。

### 3. 载荷

`_kgspSec2PostblTimingFillPayload` 用 dword `SEC2_POSTBL_TIMING_FILL_DWORD = 0x000004a7` 填满整个缓冲区，然后覆盖恰好 **24 个 dword**，全部由逐字节辅助函数 `_kgspSec2PostblTimingPutU32` 以小端写：

| 偏移 | 值 | 作用 |
|---|---|---|
| `0x1100` | `0x00000007` | 栈/控制字 |
| `0x5b40` | `0xc0deca7e` | 假栈金丝雀 |
| `0xf754` | `writeValue` | **每轮 PLM 修补** |
| `0xf758` | `0xc0deca7e` | 金丝雀 |
| `0xf75c` | `0x00000cbd` | gadget |
| `0xf76c` | `writeAddr` | **每轮 PLM 修补** |
| `0xf774` | `0x00001fbd` | gadget |
| `0xf780` | `0x00000000` | |
| `0xf788` | `0x000010aa` | **BAR0 写 gadget** |
| `0xf78c` | `0x0000815a` | 尾 |
| `0xf790` | `0x00008e18` | 尾 |
| `0xf794` | `0xc0deca7e` | 金丝雀 |
| `0xf798` | `0x0000815a` | 尾 |
| `0xf79c` | `0x00000000` | |
| `0xf7a0` | `0xc0deca7e` | 金丝雀 |
| `0xf7a4` | `0x00001fbd` | gadget |
| `0xf7b0` | `0x0000ffbc` | 尾 |
| `0xf7b8` | `0x0000582d` | 尾 |
| `0xf7c4` | `0xc0deca7e` | 金丝雀 |
| `0xf7c8` | `0x00000cbd` | gadget |
| `0xf7d8` | `0x00000003` | |
| `0xf7e0` | `0x00001fbd` | gadget |
| `0xf7f4` | `0x00000ccb` | 尾 |
| `0xf7f8` | `0x00007f2f` | 尾 |

24 个中只有两个在轮次间变化：`0xf76c` 的 `writeAddr` 和 `0xf754` 的 `writeValue`。链条是通用单寄存器写原语，每个目标重新触发一次。

载荷可以从磁盘覆盖。`_kgspCreateSignatureMemdesc` 先尝试 `os_open_and_read_file("/lib/firmware/nvidia/ga100/gsp/dmem.bin", pSignatureVa, 0xf800)`。成功记录 `SEC2_DEBUG: loaded 63488 bytes from ...`。正常路径上该文件不存在，驱动记录良性的状态 `0x59` 并回退到内置填充：

```text
SEC2_DEBUG: /lib/firmware/nvidia/ga100/gsp/dmem.bin not found (0x59), using built-in payload
```

内置默认武装 `writeAddr = 0x009a0148`、`writeValue = 0xffffffff`，PLM 循环每轮立即覆盖。

> [!NOTE]
> **`0xFACEB13D` 不是正式金丝雀**
>
> 早期独立测试台在 `CANARY_ADDR = 0x6340`、`DMA_TARGET = 0x0800` 下用 `0xFACEB13D`。那是与正式 `0x5b40` 金丝雀（`0x5b40 + 0x0800 = 0x6340`）相同的槽、不同的字面量。读正式代码，预期 `0xc0deca7e`。

### 4. PLM 循环

插入块位于 `kernel_gsp.c` 的 `kgspInitRm` 路径中，紧跟 `kgspPrepareForBootstrap_HAL(...)` 返回**之后**，不在其内部。Hunk 锚点 `@@ -4821,6 +4844,117 @@`。

四个权限级掩码，每个最多两次尝试。PLM 是逐寄存器访问控制寄存器：它决定哪个权限级可以读或写它所守护的寄存器。见 [权限级掩码](privilege-level-masks.md)。

```c
/* plmTable[] entries, verbatim from patch 0001 */
{ 0x001fa7ccU, 0xfffff0ffU, "WPR_CFG" },
{ 0x009a0148U, 0xffffffffU, "FBPA"    },
{ 0x001fa7c4U, 0xffffffffU, "WPR"     },
{ 0x00823804U, 0xffffffffU, "FEAT"    },
```

> [!WARNING]
> **四个中三个目标是 `0xffffffff`，WPR_CFG 不是**
>
> `0x001fa7cc` 的 `WPR_CFG` 被故意打开到 **`0xfffff0ff`**。循环的成功谓词是 `if (regVal == plmTable[plmIdx].value)`，因此 `0xfffff0ff` 是**通过**。项目自己的 `docs/DEBUGGING.md` 说"All the PLMs must show `0xffffffff`"，`README.md` 说"Expected: PLMs opening to 0xffffffff"。两者都是松散措辞。不要把 `WPR_CFG=0xfffff0ff` 读作失败。

每次尝试周围驱动保存并恢复写保护区域 2 的边界。循环前读 `wpr2Lo = GPU_REG_RD32(pGpu, 0x001fa824U)` 和 `wpr2Hi = GPU_REG_RD32(pGpu, 0x001fa828U)`；每次尝试都写回两者、为那一对 `{address, value}` 重填载荷，然后调用：

```c
kgspExecuteBooterLoad_HAL(pGpu, pKernelGsp,
    memdescGetPhysAddr(pKernelGsp->pWprMetaDescriptor, AT_GPU, 0));
```

并回读目标寄存器。循环后最后再恢复一次 `wpr2Lo`/`wpr2Hi`。最坏情况是**在正常引导 Booter Load 之前执行八次 Booter Load**。

> [!NOTE]
> **每次载荷轮 `status=0xffff` 是预期的**
>
> `s_executeBooterUcode_TU102` 在每次运行后发现 seccode 留在 mailbox0 中的错误码，因此 `kgspExecuteBooterLoad_HAL` 在载荷轮上无条件返回 `NV_ERR_GENERIC`（`0xffff`）。那是注入链已经执行*之后*提出的无效签名抱怨。**寄存器回读是唯一有效的成功标准。** `kgspExecuteBooterLoad_TU102` 还在每次运行前执行 `kflcnReset(SEC2)`，因此 SEC2 在各轮之间不积累状态。早期轮次中 Booter 状态 `0x31` 也可容忍。

失败打印 `FAILED to open %s after 2 attempts`。成功打印：

```text
SEC2_DEBUG: PLMs: FEAT=0xffffffff FBPA=0xffffffff WPR=0xffffffff WPR_CFG=0xfffff0ff
```

### 5. 四次主机寄存器写入

PLM 打开后，主机 CPU **直接**写解锁寄存器。这一步不涉及漏洞利用，这是关键的架构洞见：ROP 链只需要买到写权限，之后一切都是普通 MMIO 存储。

```c
GPU_REG_WR32(pGpu, 0x0082381cU, 0x88888888U);   /* SS0 */
GPU_REG_WR32(pGpu, 0x00823820U, 0x00000008U);   /* SS1 */
GPU_REG_WR32(pGpu, 0x009a0204U, cfg1Value);     /* FBPA CFG1 */
GPU_REG_WR32(pGpu, 0x00100ce0U, lmrValue);      /* MMU LMR   */
```

| 设备 ID | 卡 | `cfg1Value` | `lmrValue` | `targetFbBytes` | 结果 |
|---|---|---|---|---|---|
| `0x20C2` | 8 GB | `0x02779000` | `0x0000020B` | `0x0000001000000000` | 65536 MiB |
| `0x2082` | 10 GB | `0x02669000` | `0x0000028A` | `0x0000000A00000000` | 40960 MiB |

SS0 和 SS1 对**两个 SKU 无条件**写入。`common/constants.yaml` 与代码一致（`ss0: "0x88888888"`、`ss1: "0x00000008"`）。

> [!CAUTION]
> **文档分支对 SS0 和 SS1 是错的**
>
> `docs/ARCHITECTURE.md` 声称 cmpunlocker 向 SS0 和 SS1 都写 `0xffffffff`，并展示匹配的预期 dmesg 行。它不写。用那些字符串验证解锁会让一张可工作的卡看起来坏掉。代码自 2026-07-18 起写 `0x88888888` / `0x00000008`。见 [计算节流](compute-throttle.md)。

随后一行回读：

```text
SEC2_DEBUG: POST-WRITE SS0=... SS1=... CFG1=... LMR=... (devId=0x%x)
```

### 6. 原厂签名重建与第二轮 WPR 元数据

`kgspSec2PostblTimingRebuildStockSignature()` 用 `MEMDESC_FLAGS_ALLOC_IN_UNPROTECTED_MEMORY` 释放并以 `NV_ALIGN_UP(stockSignatureSize, 256)` 重建 `pSignatureMemdesc`，把保存的原厂签名复制回去，把 `pWprMeta->sysmemAddrOfSignature` 和 `pWprMeta->sizeOfSignature` 重置为新描述符。如果它返回非 `NV_OK` 的任何值，`_kgspBootGspRm` 传播该状态、启动以 `SEC2_DEBUG: rebuild stock signature failed: 0x%x` 中止。

然后**第二次**调用 `kgspPopulateWprMeta_HAL()`（它已经在 PLM 工作前的原厂位置被调用过）。第一次调用记录 `SEC2_DEBUG: WPR meta fbSize=... wprEnd=... heapSize=...`；第二次记录 `SEC2_DEBUG: WPR meta updated fbSize=... wprStart=... wprEnd=... heapOffset=... heapSize=...`。第二次调用让驱动的 WPR2 放置与现在扩大后的几何一致。

### 7. WPR2 降级与 GSP static-info 重写

还有两处修改完成该补丁。

"WPR2 already up"致命错误被**降级，而不是删除**。在现有 `if (kgspIsWpr2Up_HAL(...) && !pGpu->getProperty(pGpu, PDB_PROP_GPU_PREINITIALIZED_WPR_REGION))` 守卫内，锚点 `@@ -4805,14 +4820,22 @@`，两行 `NV_PRINTF(LEVEL_ERROR, ...)` 和 `return NV_ERR_INVALID_STATE;` 变成一行：

```c
NV_PRINTF(LEVEL_WARNING, "WPR2 already up before GSP boot; continuing for recovery\n");
```

执行直接落入 `kgspPopulateWprMeta_HAL`。这是必需的：解锁最多运行八次 Booter Load，而 Booter Load 让 WPR2 保持 up。

然后，在 `@@ -5164,6 +5285,53 @@`，`kgspInitRm` 收到 GSP static config info 之后，补丁重写它：`pGSCI->fb_length = targetFbBytes`，如果最后一个 FB 区域的 `limit` 低于 `targetFbBytes - 1` 就设置 `limit = targetFbBytes - 1`、`reserved = limit - base + 1`、`supportCompressed = NV_TRUE`、`supportISO = NV_TRUE`、`performance = 20`。记录为 `SEC2_DEBUG: static-info BEFORE/AFTER`。

### 没有 0001 会怎样

一切都坏。没有解锁。卡以 8192 或 10240 MiB、计算节流在位地原厂启动。

---

## 0002 `booter-verify`：断言与证明

只碰 `src/nvidia/src/kernel/gpu/gsp/arch/turing/kernel_gsp_tu102.c`。三个任务。

**它命名五个寄存器。** 这些 define 纯粹为本文件的日志：

```c
#define SEC2_DEBUG_PRI_FEATURE_OVERRIDE_PLM        0x00823804
#define SEC2_DEBUG_PRI_FEATURE_OVERRIDE_SM_SPEED   0x0082381c
#define SEC2_DEBUG_PRI_FEATURE_OVERRIDE_SM_SPEED_1 0x00823820
#define SEC2_DEBUG_PRI_FBPA_CFG1                   0x009a0204
#define SEC2_DEBUG_PRI_MMU_LMR                     0x00100ce0
```

注意名字：SS0 和 SS1 的 `FEATURE_OVERRIDE_SM_SPEED` 和 `_SM_SPEED_1`。这些是代码自己对计算节流寄存器的命名，读起来像时钟或吞吐控制，而不是簇使能位掩码。

**它在 `kgspBootstrap_TU102` 中把四个致命断言转换为记录日志的状态检查**：对 `pPreparedFwsecCmd != NULL` 的 `NV_ASSERT_OR_RETURN`、`NV_ASSERT_OK_OR_RETURN(kflcnReset_HAL)`、FWSEC 状态断言和 `NV_ASSERT_OK_OR_RETURN(kflcnResetIntoRiscv_HAL)`。

**它打印决定性证明行。** 在真实 `kgspExecuteBooterLoad_HAL` 之后，仅对两个设备 ID，它打印 `SEC2_DEBUG: normal BooterLoad status=0x%x`，然后只在状态为 `NV_OK` 时：

```text
SEC2_DEBUG: POST-BooterLoad verify PLM=... SS0=... SS1=... CFG1=... LMR=...
```

那是显示解锁挺过真正 GSP 启动（而不只是载荷轮）的回读。

### 没有 0002 会怎样

控制流实际不变：替代品仍在 FWSEC 或任一次 Falcon 复位返回非 `NV_OK` 时 `return status`，与断言完全一样。0002 买到的是诊断。没有它，没有任何东西说出四个步骤中哪个失败、以什么状态失败，而且唯一的启动后验证行消失——那是每个故障排查流程索要的行。见 [验证](../procedures/verify.md)。

---

## 0003 `late-pma`：让内存可分配

补丁 0001 告诉 GSP-RM 帧缓冲变大了。这不同于让额外内存可分配。物理内存分配器（PMA）仍必须被告知。补丁 0003 有**四个 hunk** 加一个钩子。

1. `src/nvidia/inc/kernel/gpu/mem_mgr/mem_mgr.h` 中 `memmgrSec2DebugLateExtendHighPmaRegion` 的前向声明。
2. `kmemsysPostHeapCreate_HAL` 之后的一条诊断：`SEC2_DEBUG_HEAP: fbAddrSpace=... mapRam=... fbTotal=... fbUsable=... heapTotal=... regionBytes=... publicBytes=... numRegions=...`
3. 约 180 行的 `memmgrSec2DebugLateExtendHighPmaRegion()` 本身。
4. 另一个 CeUtils 虚拟模式排除，追加到 `mem_mgr.c` 的 compbit-backing 条件。第一个在补丁 0005（`mem_scrub.c`）；全系列共两处。补丁 0005 的第二个 hunk 是 PTE-kind 覆盖（`NV_MMU_PTE_KIND_GENERIC_MEMORY`），不是虚拟模式守卫。

外加 `src/nvidia/arch/nvalloc/unix/src/osinit.c` 中的钩子，它在 `RmInitAdapter` 后期调用它并记录 `SEC2_DEBUG: late PMA extension status=0x%x`。

函数本身：

```text
stockFbBytes = 0x200000000ULL           /* 8 GiB, hard-coded for BOTH SKUs */
candidate    = highest-limit Ram.fbRegion[] entry with
               bRsvdRegion && !bInternalHeap && limit >= stockFbBytes
pmaRegion    = [ NV_MAX(base, stockFbBytes), limit ], bSupportCompressed = NV_TRUE
early-out    if pmaIsPmaManaged already covers the range
pmaRegisterRegion(pPma, numPmaRegions, NV_FALSE, &pmaRegion, 0, NULL)
then, only on NV_OK:
  split   -> append public FB_REGION_DESCRIPTOR [8 GiB, limit] with
             bRsvdRegion = NV_FALSE, bInternalHeap = NV_FALSE,
             bSupportCompressed = NV_FALSE; clamp the reserved region to stockFbBytes - 1
  or
  in-place-> clear bRsvdRegion
finally  memmgrRegenerateFbRegionPriority()
```

如果需要拆分且 `numFBRegions >= MAX_FB_REGIONS`，它返回 `NV_ERR_INSUFFICIENT_RESOURCES` 而不是拆分。

> [!NOTE]
> **`stockFbBytes` 在 10 GB 卡上也是 8 GiB**
>
> `0x200000000ULL` 对两种配置都硬编码。因此 `0x2082` 卡上的边界是 8 GiB，而不是卡的原生 10 GiB。这在正式代码中，不是本 wiki 的笔误。

### 没有 0003 会怎样

额外帧缓冲被报告但从不交给 PMA，因此区域保持保留，超过原厂大小的分配失败。这就是"nvidia-smi 显示 65536 MiB"与"你实际能分配 63 GB"的区别。

---

## 0004 `bar0-pramin-clamp`：让 PRAMIN 可达

最小而有趣的补丁：`src/nvidia/src/kernel/gpu/bus/arch/maxwell/kern_bus_gm107.c` 中一个十行 hunk。PRAMIN 是 BAR0 中的滑动窗口，CPU 经它到达帧缓冲内存。原厂代码把它放在帧缓冲顶部：

```c
offsetBar0 = (pMemoryManager->Ram.fbAddrSpaceSizeMb << 20) - DRF_SIZE(NV_PRAMIN);
```

补丁为 `0x20C2` 或 `0x2082` 且 `Ram.fbAddrSpaceSizeMb > 0x2000`（8192 MB）时增加：

```c
offsetBar0 = (0x2000ULL << 20) - DRF_SIZE(NV_PRAMIN);
```

注意阈值是 8192 MB，因此钳制在原厂 10 GB 卡上也会启动，不只是解锁卡。

### 没有 0004 会怎样

PRAMIN 孔径按 65536 MB 计算，落出可达 BAR0 空间之外。GA100 BAR0 是 16 MiB PRI 孔径；放在 64 GiB 地址空间顶部的窗口根本无法经它寻址。

---

## 0005 `ce-scrub-workarounds`：物理模式清扫

`master` 上两个 hunk。

在 `mem_mgr_tu102.c` 中，scrubber 的 PTE-kind 辅助函数用 `ENG_GET_GPU(pMemoryManager)` 获取 GPU，并对两个设备 ID 提前返回：

```c
*pteKind = NV_MMU_PTE_KIND_GENERIC_MEMORY;   /* instead of */
/*         NV_MMU_PTE_KIND_GENERIC_MEMORY_COMPRESSIBLE_DISABLE_PLC */
```

在 `mem_scrub.c` 中，`if (memmgrUseVasForCeMemoryOps(pMemoryManager))` 守卫增加：

```c
&& ((pGpu->idInfo.PCIDeviceID >> 16) != 0x20C2
 && (pGpu->idInfo.PCIDeviceID >> 16) != 0x2082)
```

因此 `DRF_DEF(0050, _CEUTILS_FLAGS, _VIRTUAL_MODE, _TRUE)` 永远不会被设置，复制引擎 scrubber 以**物理**模式运行。另一个虚拟模式排除在补丁 0003 中。

### 没有 0005 会怎样

scrubber 会用支持压缩的 PTE kind 在 MMU 表不是为这种几何构建的帧缓冲上尝试虚拟模式复制引擎操作。这是你最可能想跳过、但最不该跳过的补丁。

---

## 0006 `persistent-sw-state`：无需守护进程

总共十九行，其中九行是新增代码，位于 `kernel-open/nvidia/nv.c` 的 `@@ -1521,6 +1521,15 @@`，插在 IRQ 设置错误路径之后、`(void)rm_get_gpu_uuid_raw(sp, nv);` 之前：

```c
if (nv->pci_info.device_id == 0x20C2)
{
    nv->flags |= NV_FLAG_PERSISTENT_SW_STATE;
}
else if (nv->pci_info.device_id == 0x2082)
{
    nv->flags |= NV_FLAG_PERSISTENT_SW_STATE;
}
```

两个分支相同，本可以是一个条件。表面瑕疵，不是 bug。

`NV_FLAG_PERSISTENT_SW_STATE` 是 `nv.h` 中为 SR-IOV 虚拟功能设计的现有标志。在这里被复用，使 RM 在最后一个客户端关闭时不拆除软件状态。它实际上就是内置持久模式，也是当前设计不需要 systemd 看门狗的原因。早期几代工具确实需要一个；`remove.sh` 仍然清理当前安装器从不创建的遗留 `cmpunlocker.service` 和 `/opt/cmpunlocker/daemon/watchdog.py`。见 [工具谱系](../history/tool-lineage.md)。

### 没有 0006 会怎样

RM 在最后一个客户端退出时拆除 GPU 软件状态。在几何于驱动加载期间建立的卡上，这种拆除-重建循环正是你在 CUDA 进程之间不想要的东西。

---

## 阅读一次成功启动

```bash
sudo dmesg | grep SEC2_DEBUG
```

预期序列，按顺序：

| 行 | 来自 | 含义 |
|---|---|---|
| `saved stock signature (4096 bytes)` | 0001 | 真实签名被复制到一旁 |
| `<dmem.bin path> not found (0x59), using built-in payload` | 0001 | 正常，无覆盖文件 |
| `WPR meta fbSize=... wprEnd=... heapSize=...` | 0001 | 第一轮 WPR 元数据 |
| 逐 PLM 行，`status=0xffff` | 0001 | 每次载荷轮都预期 |
| `PLMs: FEAT=0xffffffff FBPA=0xffffffff WPR=0xffffffff WPR_CFG=0xfffff0ff` | 0001 | 四个全部打开 |
| `POST-WRITE SS0=... SS1=... CFG1=... LMR=... (devId=0x...)` | 0001 | 解锁寄存器已写 |
| `WPR meta updated fbSize=...` | 0001 | 第二轮 WPR 元数据 |
| `normal BooterLoad status=0x0` | 0002 | 真正的启动成功 |
| `POST-BooterLoad verify PLM=... SS0=... SS1=... CFG1=... LMR=...` | 0002 | 挺过真正启动 |
| `static-info BEFORE/AFTER` | 0001 | `fb_length` 加宽，GSP 返回其静态配置后 |
| `late PMA extension status=0x0` | 0003 | 新内存可分配 |

一份来自 610.43.03 上可工作双卡 8 GB 启动的参考捕获包含 **152** 行 `SEC2_DEBUG`（中等置信度，单次捕获）。缺少轨迹本身不能证明失败，因为内核环形缓冲区会轮转。

> [!NOTE]
> **行数不是可靠的跨构建指纹**
>
> 34（Gen1 构建）/ 80（Gen2 构建）以高置信度记录，而另一次 Gen2 分支 610.43.03 启动以中等置信度数到 152。不要把不匹配读作安装失败。

---

## 命名，以及它为什么看起来像 NVIDIA 功能

补丁用虚构的 NVIDIA 风格名字伪装自己。`SEC2_DEBUG_PRI_*`、`kgspSec2PostblTiming*` 和 `SEC2_DEBUG:` 日志前缀在原厂 610.43.03 源码中**任何地方都不出现**。"PostBL Timing" 是听起来合理但虚构的功能名。两份独立审查把它解读为故意把漏洞利用代码伪装成制造或调试功能，并作为反对 NVIDIA 内部作者的证据，因为拥有合法内部权限的人不需要伪装。实际后果是有用的：一次 grep 就能在启动日志中找到每一行解锁内容。

> [!NOTE]
> **该忽略的虚构缩写展开**
>
> `docs` 分支把 PLM 展开为"Program Logic Modules"、SS0/SS1 为"Suspension State"寄存器、PMA 为"Power Management Array"，并引入代码中任何地方都不出现的"SEC2 Booter PMM"。四个全错。PLM 是权限级掩码；PMA 是物理内存分配器（`pmaRegisterRegion`、`pmaGetFreeMemory`、`PMA_REGION_DESCRIPTOR`、`pmaIsPmaManaged`）；代码自己对 SS0/SS1 的命名是 `FEATURE_OVERRIDE_SM_SPEED` 和 `_SM_SPEED_1`。不要传播这些展开。

---

## 正式系列**不**包含什么

若干广为流传的说法描述的是 cmpunlocker `master` 之外的工件。核实六个补丁全部缺席：

| 说法 | 现实 |
|---|---|
| `gpuValidateRegOps` 被桩成 `return NV_OK` | 不存在。`subdevice_ctrl_gpu_regops.c` 完全没有改动。预发布 `patch.diff`、显然还有泄露包确实携带这种对寄存器操作权限验证的无条件绕过；发布前被删除。那是真正的安全改进，不是清理。 |
| `CMP170HX_WPR2_SAFE_LIMIT 0x0A00000000ULL` 对 `pWprMeta->fbSize` 的钳制 | 提过，从未发布。发布设计加宽 `fb_length` 和最后一个 FB 区域，只钳制 BAR0/PRAMIN 窗口。 |
| 把除 `0x00000000`/`0x00000031` 外的任何邮箱值都视为存活的容差 | 提过，从未发布。字面量 `0x00000031` 在仓库中任何地方都不出现。 |
| `0x00100ce4` LMR 锁清除/重锁规避 | 正式代码中任何地方都不出现。记录为一份 40 GB 指南中未测试的应急方案，其片段还针对错误的文件（`kernel_gsp_tu102.c`；LMR 写在 `kernel_gsp.c`）。 |
| 对 `gsp_tu10x.bin` 的任何 ELF 手术 | 无。那是第 1 代方案，2026-07-18 被替换。 |
| PCIe Gen2 补丁 `0007`/`0008` | 不在 `master`。仅分支。见 [PCIe Gen2](pcie-gen2.md)。 |
| `kflcnIsRiscvActive` 绕过 | 六个补丁中任何一个都不存在。 |

Master 的补丁集就是 `0001` 到 `0006`，没有别的。

---

## 相关页面

- [解锁的工作原理](how-it-works.md)，端到端启动故事
- [权限级掩码](privilege-level-masks.md)，四个 PLM 目标的细节
- [显存几何](memory-geometry.md)，CFG1、LMR 与容量算术
- [计算节流](compute-throttle.md)，SS0 和 SS1
- [驱动版本](../procedures/driver-versions.md)，为什么是 610.43.0x、后移植做什么
- [寄存器参考](register-reference.md) 和 [寄存器索引](../appendix/register-index.md)
- [故障排查](../procedures/troubleshooting.md)，`SEC2_DEBUG` 轨迹提前停止时
