# 解锁的工作原理

**本页涵盖：** 完整的机制，从端到端、按启动时发生的顺序，以及每一步存在的原因。有能力的工程师应该能只读这一页就理解整个设计，无需打开源码。逐寄存器的细节见 [寄存器参考](register-reference.md)；Falcon 内部结构见 [Falcon 与 Booter](falcon-and-booter.md)；载荷构造见 [ROP 链](rop-chain.md)。

一句话版本：驱动把 GSP 签名缓冲区扩大到 `0x0000f800`，向其中填充精心构造的 Falcon ROP 载荷，对 NVIDIA 自签名的 SEC2 Booter Load 每个权限级掩码各重跑一次，以在 Heavy-Secure 权限下获得一次任意 BAR0 写入，打开四道 PLM，写入四个普通主机寄存器，恢复真实签名，然后让 GSP 以把新几何补进其上报的 static-info 的方式正常启动。

---

## 为什么必须以这种方式工作

三个实测约束决定了整个设计。每一个都是吃尽苦头才发现的。

**1. 显存几何经不起复位，但计算解锁可以。** 在功能级复位下的实测：

| 寄存器 | FLR 后存续？ |
|---|---|
| `SS0 0x0082381c` | **是** |
| `SS1 0x00823820` | **是** |
| `FEAT_OVR_PLM 0x00823804` | **是**（常开岛） |
| `CFG1 0x009a0204` | 否，恢复为 `0x02449000` |
| 逐 FBPA 的 CFG1、CSTATUS | 否 |
| `LMR 0x00100ce0` | 否，恢复为原厂值 |
| FB 几何 PLM | 否，它们重新上锁 |
| AON LMR 影子 `0x001180f0` | 否，恢复 |
| SEC2 复位 PLM 污染 | FLR **清除**（`0x8f` 变为 `0xff`） |

`FEAT_OVR_PLM` 是 26 寄存器调查中唯一标为常开的 PLM。这种不对称性是计算解锁比显存解锁早数周发布的唯一原因，也是显存解锁不能沿用旧的"触发、FLR、再从主机写入"计算配方的原因。

**2. 帧缓冲几何没有常开影子。** 人们花了数小时寻找这样一个影子，因为 SS0 有可找到的 AON 影子，且一篇已发表论文描述了这一概念。即使六道 FB 几何 PLM 加上 `FUSE_SS_PLM` 全部打开，CFG1、CSTATUS 和 LMR 在 FLR 后仍然恢复，并且从不冷启动持久。一次专门的 FLR 存续性映射运行得出结论：没有任何 PLM 能把 FBPA 配置移入常开域。因此几何必须在**每次**加载模块时重新应用。

**3. 原厂驱动会重新锁上无驱动工具打开的东西。** 在未加载驱动的情况下从主机写入 `0x009A0204`、`0x0082381C` 和 `0x00823804` 是有效的，写入明显落盘，但随后加载原厂驱动把 `0x00823804` 读回 `0xffffff8f`，节流分频器恢复为 5。这个失败正是驱动内方案要解决的问题。

再加一条顺序约束：**GSP-RM 在自己的启动期间把 LMR 视为主控。** CFG1 设为 40 GB 档但 LMR 保持原厂 `0x288` 时，GSP-RM 在 `kgspBootstrap` 期间把逐 FBPA 的 `CSTATUS_RAMAMOUNT` 从 `0x800` 恢复为 `0x200`。因此几何必须在真正的 Booter Load 启动 GSP-RM **之前**就位，而不是之后。

维护者得出的结论：全部在 `_kgspBootGspRm` 内完成，位置在驱动拥有它控制的签名缓冲区与真正的 Booter Load 运行之间。

---

## 第 0 步：解锁在 GA100 启动链中的位置

```text
Power on
  └─ GFW / DEVINIT        signed firmware from flash; reads the RAMCFG strap,
                          latches the L2 address map. No RM yet. Always locked.
  └─ CPU-side RM (nvidia.ko)
       ├─ kgspPopulateWprMeta      reads hardware geometry / LMR into WprMeta.fbSize,
       │                           decides WPR2 placement
       ├─ kgspPrepareForBootstrap  runs FWSEC / FRTS (the VBIOS devinit)
       └─ kgspBootstrap            ◀── THE UNLOCK RUNS HERE, inside _kgspBootGspRm
            └─ SEC2 Booter Load    signed Falcon ucode; carves WPR2, launches GSP-RM
  └─ GSP-RM                        closed RISC-V firmware, runs on the GPU
```

由于冷启动总是运行带锁定 CMP strap 表的签名 DevInit，首次枚举总是显示原始容量、原始 Gen1 链路和节流在位。单纯的 `rmmod` / `modprobe` **不会**重跑 DevInit（没有断言 PERST），这就是为什么上一次加载写入的几何在驱动重载后存续、但经不起复位。

---

## 第 1 步：驱动加载并检测设备 ID

补丁 0001 在 `src/nvidia/src/kernel/gpu/gsp/kernel_gsp.c` 中添加一个门控辅助函数：

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

**为什么移位。** `pGpu->idInfo.PCIDeviceID` 把厂商和设备打包在一起；设备半部分在高 16 位。补丁 0001 到 0005 中的每个解锁点都以同样方式测试它。补丁 0006 是唯一例外，比较原始 `nv->pci_info.device_id`，因为它运行在 `kernel-open/nvidia/nv.c` 的 PCI probe 阶段，那时 `OBJGPU` 还不存在。

**为什么用运行时门控而不是编译时门控。** 两种几何配置编译进同一个模块：

```c
if (devId == 0x20C2) { cfg1Value = 0x02779000U; lmrValue = 0x0000020BU; }
else                 { cfg1Value = 0x02669000U; lmrValue = 0x0000028AU; }
```

`driver/build.sh` 仍含一段内联 Python 步骤，过去用于重写这些常量，但在 `master` 上它检测到两种几何都已存在，于是打印 `runtime device-id geometry (profile metadata=<label>)` 后退出，不做任何修改。因此被误检测的 `--profile` 只会写入错误的元数据，不会产生错误的几何。

任何其他 GA100 SKU，包括 `10de:20b0`，即使安装了补丁模块也走完全原厂的启动路径。

---

## 第 2 步：签名缓冲区扩大到 `0xf800`

`_kgspCreateSignatureMemdesc` 通常分配 `NV_ALIGN_UP(pGspFw->signatureSize, 256)` 字节，在此平台上为 4096。当设备门控为真时，它改为在 `ADDR_SYSMEM` 中分配 `SEC2_POSTBL_TIMING_SIGNATURE_SIZE = 0x0000f800ULL`（63,488 字节），按 256 字节对齐。在覆盖任何内容之前，真实签名字节被复制到两个新的 `KernelGsp` 字段中：

```c
NvU8 *pStockSignatureData;
NvU64 stockSignatureSize;
```

记录为 `SEC2_DEBUG: saved stock signature (4096 bytes)`。

**为什么恰好是 0xf800。** 这就是整个漏洞利用。这个漏洞是 **Booter 的 LS 签名校验中的无界 DMA**：IMEM `0x29C4` 处的 `booterVerifyLsSignatures_TU10X` 调用 `booterIssueDma_HAL`，其 DMEM 目标固定，传输长度直接取自 `WprMeta` 中的 `sizeOfSignature`，没有任何边界检查。DMA 目标是 **DMEM `0x0800`**，而 Falcon DMEM 为 64 KB。因此：

```text
0x0800 (DMA base) + 0xF800 (length) = 0x10000 (end of DMEM)
```

载荷与 DMEM `0x0800`..`0xFFFF` 一一对应，这是 Booter 运行时使用的全部区域，包括它的栈、保存的返回地址和栈金丝雀全局变量。精确选择 `0xf800` 使载荷最后一个字节落在 DMEM 最后一个字节上。（算术可自证：载荷在偏移 `0x5b40` 写入假金丝雀，而 `0x5b40 + 0x800 = 0x6340`，即独立反汇编出的金丝雀全局地址。）

**为什么这是 ROP 而不是代码执行。** Falcon 把指令放在 IMEM、数据放在 DMEM。溢出只到达 DMEM。它控制 Falcon 调用栈上的返回地址，因此攻击必须由签名 booter 镜像中已有的 gadget 构造。

**缓冲区不含什么。** 固件拼接时代的一个早期信念是，缓冲区必须以真实、有效的签名开头，因为把整个 `0xF800` 区域零填充会让原厂 booter 以邮箱 `0x31` 退出。正式载荷并不保留它：`_kgspSec2PostblTimingFillPayload()` 从偏移 0 起把每个 dword 写成 `0x000004a7`，从不把签名字节复制回来。保存的原厂字节只存在于 `pStockSignatureData` 中，直到第 7 步把它们放回去。邮箱 `0x31` 也是成功载荷通过时报告的，因此它本身不是签名有效性判定。

`_kgspCreateSignatureMemdesc` 还尝试在 `/lib/firmware/nvidia/ga100/gsp/dmem.bin` 上调用 `os_open_and_read_file()`，失败时记录 `SEC2_DEBUG: <path> not found (0x59), using built-in payload`。状态 `0x59` 是良性的、预期中的。

---

## 第 3 步：写入载荷

`_kgspSec2PostblTimingFillPayload(buffer, writeAddr, writeValue)` 首先把缓冲区的每个 dword 填成 `SEC2_POSTBL_TIMING_FILL_DWORD = 0x000004a7`，然后覆盖以下槽位。DMEM 列就是载荷偏移加 `0x800`：

| 载荷偏移 | DMEM | 值 | 作用 |
|---|---|---|---|
| 全部 | `0x0800`-`0xFFFF` | `0x000004a7` | 填充 dword |
| `0x1100` | `0x1900` | `0x00000007` | `f100_field_save_restore` 门控；让 SEC2 复位 PLM 保持 `0xff` |
| `0x5b40` | `0x6340` | `0xc0deca7e` | **写入防护全局变量的假金丝雀** |
| `0xf754` | `0xFF54` | *writeValue* | 值参数 |
| `0xf758` | `0xFF58` | `0xc0deca7e` | 保存金丝雀槽 |
| `0xf75c` | `0xFF5C` | `0x00000cbd` | |
| `0xf76c` | `0xFF6C` | *writeAddr* | 地址参数 |
| `0xf774` | `0xFF74` | `0x00001fbd` | |
| `0xf780` | `0xFF80` | `0x00000000` | |
| `0xf788` | `0xFF88` | `0x000010aa` | **BAR0 主控写 gadget** |
| `0xf78c` | `0xFF8C` | `0x0000815a` | |
| `0xf790` | `0xFF90` | `0x00008e18` | |
| `0xf794` | `0xFF94` | `0xc0deca7e` | 保存金丝雀槽 |
| `0xf798` | `0xFF98` | `0x0000815a` | |
| `0xf79c` | `0xFF9C` | `0x00000000` | |
| `0xf7a0` | `0xFFA0` | `0xc0deca7e` | 保存金丝雀槽 |
| `0xf7a4` | `0xFFA4` | `0x00001fbd` | |
| `0xf7b0` | `0xFFB0` | `0x0000ffbc` | |
| `0xf7b8` | `0xFFB8` | `0x0000582d` | |
| `0xf7c4` | `0xFFC4` | `0xc0deca7e` | 保存金丝雀槽 |
| `0xf7c8` | `0xFFC8` | `0x00000cbd` | |
| `0xf7d8` | `0xFFD8` | `0x00000003` | |
| `0xf7e0` | `0xFFE0` | `0x00001fbd` | |
| `0xf7f4` | `0xFFF4` | `0x00000ccb` | 见下方开放问题 |
| `0xf7f8` | `0xFFF8` | `0x00007f2f` | 最外层槽 |

这个块在 `master` 和全部十二个存档分支中逐字节相同。魔法值 `0xc0deca7e` 在每个副本中恰好出现五次。

**为什么需要假金丝雀。** Booter 每次启动都生成新的随机栈金丝雀，保存在 DMEM `0x6340` 的全局变量中。每个受保护函数把它复制到栈帧边界并在退出时重新比较；不匹配就调用 `panic()`。因为该值每次启动都重新生成，无法离线猜测。载荷不需要猜测它：它用 `0xc0deca7e` 覆盖*全局变量*，并把同一常量写进每个重建的金丝雀槽，于是每个尾声比较都通过，展开静默进行。

**为什么是 `0x000010aa`。** 那是 `reg_write_indirect`，booter 自己的任意 BAR0 写入例程。它与 NVIDIA 的 `_acrlibBar0RegWrite_TU10X` 逐字节相同，驱动 Falcon CSB 空间中一个带互斥锁的间接邮箱：

```text
I[0x1c100] = target PRI address
I[0x1c200] = data
I[0x1c000] = 0x800000f2   (write; 0x800000f1 is read)
```

Booter 用自己的工作也走这条路径，这就是该原语被发现的方式。这一个 gadget 就是整个权限提升：它在 LEVEL2 上、在一个真实、已签名、已验证的 HS 镜像内执行。

**为什么每次触发只写一个寄存器。** 链条在退出时不得不重建 booter 的栈帧，每次写入花费一个 `0x18` 字节的帧。独立实现把每次触发的硬上限定在两到六个写入之间，而无驱动引擎拒绝构造一两个写入之外的载荷。正式驱动每次触发只做**一个**写入，然后直接重新触发，这更简单，也没有预算风险。

---

## 第 4 步：执行 Booter Load 以获得写入原语

每一轮驱动调用：

```c
kgspExecuteBooterLoad_HAL(pGpu, pKernelGsp,
    memdescGetPhysAddr(pKernelGsp->pWprMetaDescriptor, AT_GPU, 0));
```

`kgspExecuteBooterLoad_TU102` 在每次运行前执行 `kflcnReset(SEC2)`，因此 SEC2 在各轮之间不积累状态。Booter 加载，通过针对自身镜像（未改动、真实）的 RSA-3072 启动 ROM 检查，把自己解密进 HS 模式，开始验证 LS 签名，把 `0xf800` 字节 DMA 进 DMEM，然后沿着注入的返回地址而不是自己的返回地址展开。

> [!WARNING]
> **报告的状态总是失败，这是预期的**
>
> 每次载荷轮都记录
> `s_executeBooterUcode_TU102: Booter failed with non-zero error code: 0x31` 和
> `kgspExecuteBooterLoad_TU102: failed to execute Booter Load: 0xffff`，而寄存器写入仍然落盘。seccode 每次运行后都会在 mailbox0 中留下错误码，`mailbox0 != 0` 使 HAL 返回 `NV_ERR_GENERIC`（`0xffff`）。**寄存器回读是唯一有效的成功标准**，而这正是正式循环使用的标准。不要把 `status=0xffff` 当成问题。

一轮 Booter 大约耗时 **180 ms**。

---

## 第 5 步：按顺序打开四道 PLM

权限级掩码是逐寄存器块的门控：它决定哪些权限级可以读写它所覆盖的寄存器。在 PL0（内核驱动的普通主机 BAR0 写入）下，目标寄存器就是不可写的，而且重要的是，**失败是静默的**。早期流水线记录了三遍 `Write failed - wrote 0x2779000, read 0x2449000`，任何地方都没有报错。

正式的 `plmTable[]` 恰好有四个条目，按此顺序打开：

| 索引 | 名称 | 地址 | 写入的值 | 为什么需要 |
|---|---|---|---|---|
| 0 | WPR_CFG | `0x001fa7cc` | `0xfffff0ff` | 门控 booter 验证、驱动操作的 WPR 配置块 |
| 1 | FBPA | `0x009a0148` | `0xffffffff` | 门控 FBPA 孔径，包括 `CFG1 0x009a0204`。也是内置回退载荷目标 |
| 2 | WPR | `0x001fa7c4` | `0xffffffff` | 门控 WPR 区域寄存器 |
| 3 | FEAT | `0x00823804` | `0xffffffff` | `FEAT_OVR_PLM`。门控功能覆盖块，即 SS0 和 SS1 |

> [!WARNING]
> **WPR_CFG 打开到 `0xfffff0ff`，不是 `0xffffffff`**
>
> 发行版 README 和 `docs/ARCHITECTURE.md` 都说每个 PLM 应读 `0xffffffff`。代码为条目 0 写入并验证 `0xfffff0ff`。请把 `0xfffff0ff` 视为权威值：正式循环写入并对照检查的就是它。

每个条目**最多尝试两次**。尝试循环是：

```text
for each plmTable entry:
    for attempt in 1..2:
        GPU_REG_WR32(0x001fa824, wpr2Lo)      # re-arm WPR2 low
        GPU_REG_WR32(0x001fa828, wpr2Hi)      # re-arm WPR2 high
        kgspSec2PostblTimingRefillPayload(addr, value)
        kgspExecuteBooterLoad_HAL(...)         # returns 0xffff; ignored
        if GPU_REG_RD32(addr) == value: break  # readback is the verdict
restore wpr2Lo / wpr2Hi one final time
```

**为什么每次尝试前都必须重新武装 WPR2。** WPR2 由 ADDR_LO `0x001fa824` 和 ADDR_HI `0x001fa828` 控制。当 HI 为零时 `kgspIsWpr2Up()` 返回 false。每轮 Booter Load 划分 WPR2 并让它保持 up 状态；第二次 Booter Load 随后会以"WPR2 already up"中止。保存循环前的值并在每次尝试前重写，使寄存器回到驱动和 booter 都期望的状态。同样的问题也是补丁 0001 把 `_kgspBootGspRm` 中的致命路径降级的原因：

```c
/* stock */
NV_PRINTF(LEVEL_ERROR, "unexpected WPR2 already up, cannot proceed with booting GSP\n");
return NV_ERR_INVALID_STATE;

/* patched */
NV_PRINTF(LEVEL_WARNING, "WPR2 already up before GSP boot; continuing for recovery\n");
```

执行随后直接落入 `kgspPopulateWprMeta_HAL`。

**为什么 `FEAT_OVR_PLM` 能够被打开。** 门控链是 `FUSE_QUADRO_WR_SEC`（`0x0082038c`）= 1 允许打开 `0x00823804`；打开 `0x00823804` 允许 PL0 主机写入 SS0/SS1；功能覆盖寄存器的优先级高于熔丝。而整条链之所以存在，仅仅因为主熔断丝 `OPT_FEATURE_FUSES_OVERRIDE_DISABLE` 在 `0x008203f0` 处于 170HX 上读出 `0x00000000`。如果它已熔断，任何权限级都不会有软件路径。PLM 本身不是 PL0 可写的：它必须由处于 HS 模式的 Falcon 打开，"若非如此，任何 Nvidia 卡都可以不经任何漏洞利用就被解锁"。

正式树**不**向 `0x008200fc`（`FUSE_SS_PLM`，分支源码中称 `OPT_PLM`）写入任何内容。早期整合配方要求打开它；它本来就在原厂卡上读出 `0xffffffff`，因此打开它没有必要。Gen2 家族分支确实添加了它，作为把表从四项扩到九项的五个额外条目之一。

---

## 第 6 步：写入计算与几何寄存器

PLM 打开后，权限提升就结束了。四个普通主机写入完成整个解锁，不涉及任何漏洞利用：

```c
GPU_REG_WR32(pGpu, 0x0082381cU, 0x88888888U);   /* SS0 */
GPU_REG_WR32(pGpu, 0x00823820U, 0x00000008U);   /* SS1 */
GPU_REG_WR32(pGpu, 0x009a0204U, cfg1Value);     /* FBPA CFG1 broadcast */
GPU_REG_WR32(pGpu, 0x00100ce0U, lmrValue);      /* MMU local memory range */
```

随后回读记录为 `SEC2_DEBUG: POST-WRITE SS0=... SS1=... CFG1=... LMR=... (devId=0x%x)`。

### SS0 和 SS1：计算解除节流

`0x0082381c` 是 `NV_FUSE_FEATURE_OVERRIDE_SM_SPEED_SELECT`，保存 IMLA0-3、FMLA16、FMLA32、FFMA 和 DP 的八个 4 位字段。`0x00823820` 是 `..._SM_SPEED_SELECT_1`，保存第九个字段 IMLA4。每个半字节读作 `[enable | 3 位速度]`：`0x8` 设置位 3（覆盖使能），速度字段为 0（满速）。因此 `0x88888888` 表示全部八个 SS0 单元"覆盖使能、满速"，`0x00000008` 对单独的 IMLA4 做同样的事。

**为什么两个都要。** 只写一个不够；这一点在频道内被独立强调，并反映在每一对正式写入中。

**为什么熔丝保持熔断仍能生效。** 节流按算术单元以 OTP 熔断：`FUSE_SS_FFMA`、`FUSE_SS_FMLA16/32` 和 `FUSE_SS_IMLA0-4` 在 170HX 上全部读出 `0x5`（除以 32），在探测过的每个 A100、A10、A5000、A6000 和 Drive A100 上读出 `0x0`。成功解锁后这些熔丝影子**仍然读出 `0x5`**。有效速率由可写的覆盖仲裁而来，覆盖优先于熔丝，而不是直接由熔丝决定。确认方式是 `0x00823818` 处的 `FEAT_READOUT_1`，即全部九个单元的只读有效速度选择，从 `0x016db6ed` 降到 `0x00000000`。

### CFG1 和 LMR：几何

`0x009a0204` 是 FBPA CFG1 广播寄存器。它在位 [23:16] 的档字节编码每个内存分区的寻址深度：`0x44` 为原厂（12 行位，每个 FBPA 512 MiB）、`0x66`（14 行位，2048 MiB）、`0x77`（15 行位，4096 MiB）。总容量是寻址深度乘以熔丝决定的启用 FBPA 数量，而 CFG1 不碰后者。两个正式值字面就是真实 A100 部件的原厂 CFG1 字：`0x02779000` 是 A100 PCIe 80 GB 读出的，`0x02669000` 是 A100 PCIe 40 GB 和 SXM4 40 GB 读出的。解锁恢复真实的 A100 几何，而不是发明常量。

`0x00100ce0` 是 MMU 本地内存范围。它把帧缓冲总大小编码为：

```text
size_MiB = MAG[9:4] << SCALE[3:0]
```

| 值 | 解码 | 含义 |
|---|---|---|
| `0x00000208` | 32 << 8 | 8192 MiB（原厂，8 GB 卡） |
| `0x00000288` | 40 << 8 | 10240 MiB（原厂，10 GB 卡） |
| `0x0000020B` | 32 << 11 | 65536 MiB（8 GB 卡解锁） |
| `0x0000028A` | 40 << 10 | 40960 MiB（10 GB 卡解锁） |
| `0x0000028B` | 40 << 11 | 81920 MiB（80 GB 尝试） |

MAG 按 SKU 恒定，等于启用 FBPA 数量的两倍。SCALE 才是解锁改变的东西。

> [!WARNING]
> **`0x40A` 和 `0x50A` 已被证伪，而非观察到**
>
> 两者都曾作为候选编码流传。`(0x40A >> 4) & 0x3F = 0`，`(0x50A >> 4) & 0x3F = 0x10`，所以在 6 位 MAG 字段下两者都无法解码；2026-07-11 在 10 GB 卡上尝试 `0x40A` 时两个寄存器都没有变化。`LOWER_MAG` 字段的精确宽度（[9:4] 的 6 位还是 [10:4] 的 7 位）从未从 `dev_fb.h` 中读出，仍是这里最后一个未决细节。

**为什么 LMR 是硬性前置条件，而不是优化。** 硬件上做过一次受控三方对比：不写内存时，CPU-RM 在 `kbusVerifyBar2` 处失败 `0x24`；40 GB CFG1 strap 配原厂 10 GB LMR 仍然失败 `0x24`；strap 配匹配的 LMR 才到 `0x25`（StateLoad）。没有任何配置能在没有 LMR 的情况下到达 StateLoad。而按照上面提到的 GSP-RM 行为，不一致的 CFG1/LMR 对会在 `kgspBootstrap` 期间被恢复。

**为什么一次广播写入就够。** `0x009A0000`-`0x009A3FFF` 是广播 FBPA 孔径；24 个逐实例镜像位于 `0x00900204 + n*0x4000`。正式驱动**不**写任何逐 FBPA 寄存器就产出可工作的卡，因为 devinit 随后运行并传播该值。在没有 devinit 的无驱动运行时环境中，广播单独不会移动 CSTATUS，必须手工写入所有逐 FBPA 实例。这种传播是否是 PRI priv-ring 硬件机制从未被直接仪器化验证。

---

## 第 7 步：重建原厂签名

`kgspSec2PostblTimingRebuildStockSignature()` 释放并销毁 `0xf800` 载荷 memdesc，用 `MEMDESC_FLAGS_ALLOC_IN_UNPROTECTED_MEMORY` 分配一个 `NV_ALIGN_UP(stockSignatureSize, 256)` 的新 memdesc，把 `pStockSignatureData` 复制回去，并把 `pWprMeta->sysmemAddrOfSignature` 和 `pWprMeta->sizeOfSignature` 重新指向新描述符。如果失败，`_kgspBootGspRm` 返回该状态，启动中止。

**为什么。** 下一次 Booter Load 才是真正的：它必须正确划分 WPR2、通过自己的签名验证并启动 GSP-RM。如果超大的载荷缓冲区仍然挂着，那次运行会再次溢出 DMEM，GSP 永远不会启动。恢复真正的 4096 字节签名及其真实长度，交给 Booter 的就是原厂驱动会给它的东西。

**为什么几何改变不会使签名失效。** 原厂 AES-MAC 覆盖的是静态 GSP 固件镜像（静止时），不是运行时 WPR 元数据，也不是硬件几何。WPR 元数据由驱动在运行时计算。早先与此相反的说法被明确收回，正式保存 / 注入 / 恢复流程就是经验性证明。

这一步也解释了一个现实陷阱：如果机器之前跑过固件补丁前代方案，磁盘上的 `gsp_tu10x.bin` 必须首先恢复到原厂。否则驱动会在第 2 步把*漏洞利用载荷*保存为"原厂"签名，干净的 GSP-RM 启动随后会 DMA 错误的 ROP 链。要寻找的成功行是 `SEC2_DEBUG: saved stock signature (4096 bytes)`。

---

## 第 8 步：重新计算 WPR 元数据

`kgspPopulateWprMeta_HAL()` 在几何写入和签名重建**之后**被**第二次**调用。第一次调用运行在原厂位置，根据旧的、较小的帧缓冲计算 WPR2 放置。第二次调用根据现在已生效的几何重新计算，记录：

```text
SEC2_DEBUG: WPR meta updated fbSize=0x0000001000000000 wprStart=... wprEnd=... heapOffset=... heapSize=...
```

没有它，驱动的 WPR2 放置和 booter 的划分会不一致，这正是设计定型前消耗数天调试的失败类别（`0x55`、`0x65`）。

---

## 第 9 步：GSP 正常启动

真正的 Booter Load 现在未修改地运行。补丁 0002 增加确认回读：

```text
SEC2_DEBUG: normal BooterLoad status=0x0
SEC2_DEBUG: POST-BooterLoad verify PLM=0xffffffff SS0=0x88888888 SS1=0x00000008 CFG1=0x02779000 LMR=0x0000020b
```

第二行只在状态为 `NV_OK` 时打印，是解锁挺过真正 GSP 启动的决定性证据。它也是排查报告时唯一要索取的一行。补丁 0002 另外把 `kgspBootstrap_TU102` 中的四个致命断言转换为记录日志的状态检查，使瞬时失败产生诊断信息而不是死适配器。

---

## 第 10 步：修补 GSP static info，使新容量被通告

打开几何寄存器改变硬件解码的内容。它不改变 GSP-RM *上报*给驱动的内容。因此补丁 0001 在 `kgspInitRm` 收到 static config info 后重写它，以相同的两个设备 ID 门控：

- `pGSCI->fb_length` 设为 `targetFbBytes`：`0x20C2` 为 `0x0000001000000000`（64 GiB），`0x2082` 为 `0x0000000A00000000`（40 GiB）。
- 如果最后一个 FB 区域的 `limit` 低于 `targetFbBytes - 1`，该区域被加宽：`limit = targetFbBytes - 1`、`reserved = limit - base + 1`、`supportCompressed = NV_TRUE`、`supportISO = NV_TRUE`、`performance = 20`。

记录为 `SEC2_DEBUG: static-info BEFORE` / `AFTER`。没有它，驱动根本不会上报加宽后的大小。

---

## 第 11 步：让额外容量真正可分配

还有四个补丁弥合"区域存在"与"CUDA 能用"之间的差距。

**补丁 0003，延迟 PMA 扩展。** `memmgrSec2DebugLateExtendHighPmaRegion()` 从 `osinit.c` 中的钩子在堆创建后运行。它挑选 `bRsvdRegion && !bInternalHeap && limit >= 8 GiB` 的最高 FB 区域，用 `pmaRegisterRegion` 注册 `[max(base, 8 GiB), limit]`，然后把候选区域从 8 GiB 起拆成新的公开 `FB_REGION_DESCRIPTOR`，或原地取消保留，随后调用 `memmgrRegenerateFbRegionPriority`。如果需要拆分且 `numFBRegions >= MAX_FB_REGIONS`，它返回 `NV_ERR_INSUFFICIENT_RESOURCES`。记录为 `SEC2_DEBUG: late PMA extension status=0x0`。

注意 `stockFbBytes = 0x200000000ULL`（8 GiB）被硬编码为**两种配置**的拆分点，包括真实原厂大小为 `0x280000000` 的 10 GB 卡。同一常量在补丁 0001 中被声明但从未使用。

**补丁 0004，PRAMIN 钳制。** `kern_bus_gm107.c` 中一个十行 hunk。在原厂 `offsetBar0 = (Ram.fbAddrSpaceSizeMb << 20) - DRF_SIZE(NV_PRAMIN);` 之后增加：如果设备 ID 匹配且 `Ram.fbAddrSpaceSizeMb > 0x2000`，重新计算为 `(0x2000ULL << 20) - DRF_SIZE(NV_PRAMIN)`。**为什么：** PRAMIN 窗口偏移由帧缓冲大小推导。按 65536 MB 计算会落到可达 BAR0 空间之外。把它钳回原厂 8 GiB 地址空间可保持孔径可用。

**补丁 0005，复制引擎 scrub 规避。** 两个 hunk。在 `mem_mgr_tu102.c` 中，scrubber PTE-kind 辅助函数对两个设备 ID 提前返回 `NV_MMU_PTE_KIND_GENERIC_MEMORY`，而不是 `NV_MMU_PTE_KIND_GENERIC_MEMORY_COMPRESSIBLE_DISABLE_PLC`。在 `mem_scrub.c` 中，`memmgrUseVasForCeMemoryOps()` 门控增加设备 ID 排除，使 `DRF_DEF(0050, _CEUTILS_FLAGS, _VIRTUAL_MODE, _TRUE)` 永远不会被设置，scrubber 以物理模式运行。

**补丁 0006，持久软件状态。** `kernel-open/nvidia/nv.c` 中 PCI probe 处的九行代码，对任一设备 ID 设置 `nv->flags |= NV_FLAG_PERSISTENT_SW_STATE`。该标志原本就为 SR-IOV 虚拟功能存在，被复用使 RM 在最后一个客户端关闭时不拆除软件状态。这实际上就是内置持久模式，也是正式设计不需要 systemd 守护进程的原因。（`remove.sh` 仍然清理当前安装器从不创建的遗留 `cmpunlocker.service` 和 `watchdog.py`：它们是已放弃看门狗设计的遗迹。）

---

## 完整时序

来自一次完整 8 GB dmesg 捕获的时间线，从驱动加载到卡可用：

| 时间 | 事件 |
|---|---|
| 11.13 s | 保存原厂签名 |
| 11.32 s | PLM[0] WPR_CFG 打开 |
| 11.50 s | PLM[1] FBPA 打开 |
| 11.68 s | PLM[2] WPR 打开 |
| 11.86 s | PLM[3] FEAT 打开 |
| 11.86 s | POST-WRITE（SS0、SS1、CFG1、LMR）和 WPR 元数据更新 |
| 12.07 s | 正常 BooterLoad `status=0x0` |
| 12.07 s | POST-BooterLoad verify：`PLM=0xffffffff SS0=0x88888888 SS1=0x00000008 CFG1=0x02779000 LMR=0x0000020b` |
| 12.64 s | 堆创建 |
| 12.72 s | 延迟 PMA 扩展 `status=0x0` |

约一秒墙钟时间，四轮 Booter，每轮约 180 ms。

解锁所做的一切都可以用以下命令查看：

```bash
sudo dmesg | grep SEC2_DEBUG
```

`SEC2_DEBUG_PRI_*` 寄存器名、`kgspSec2PostblTiming*` 函数名和 `SEC2_DEBUG:` 前缀在原厂 610.43.03 源码中任何地方都不出现。"PostBL Timing" 是一个虚构的功能名，被两份独立审查解读为故意把漏洞利用代码伪装成制造或调试功能。

---

## 之后什么会存续

| 事件 | 计算解锁 | 显存几何 |
|---|---|---|
| 卸载 / 重载驱动，无复位 | 存续 | **存续**（寄存器仍读出解锁值） |
| FLR（`echo 1 > /sys/bus/pci/devices/<bdf>/reset`） | 存续 | **丢失** |
| 断电 / 冷启动 | 丢失 | 丢失 |

因为没有任何东西在冷启动后持久，整个序列在每次加载模块时重跑。这不是实现的局限；它是帧缓冲几何没有常开影子的直接后果。

---

## PCIe Gen2 在这个叙述中的位置

> [!WARNING]
> **实验性，仅分支**
>
> `0007-pcie-gen2.patch` 把它的整个寄存器块注入 `kernel_gsp.c` 的 `@@ -4942,6 +4942,260 @@` 处，紧跟在 `devId` 打印之后、调用 `kgspSec2PostblTimingRebuildStockSignature()` **之前**。因此它恰好运行在第 5、6 步描述的窗口内：PLM 仍然打开、构造的签名载荷仍提供任意 BAR0 写入原语。它把 23 条 `xp3gTable` 加两个寄存器经 Booter 推入（25 次经 Booter 路由的写入），然后做普通主机 BAR0 写入，再把实际链路 retrain 留给补丁 0008 或用户空间。完整细节见 [PCIe Gen2](pcie-gen2.md)。

---

## 机制本身的开放问题

> [!NOTE]
> **开放问题**
>
> **链条真的会执行 `0x0ccb` 吗？** 有一条记录的硬约束：任何 ROP 退出路径都不得经过 `regtable_rw_indexed (0x0ccb)`，因为 `0xF800` 载荷线性砸掉了它在 DMEM `0x2383` 和 `0x8e08` 索引的描述符表，而 2026-07-06 的隔离矩阵显示每条携带写入的 rejoin 链都死在那里。然而正式载荷把 `0x00000ccb` 放在 DMEM `0xFFF4` 并且能工作。下一步是对从 `0xFF54` 起的展开做单步或仿真，记录 `0xFFF4` 是否被弹入 PC，还是只是最外层帧中一个存活通过的保存槽。

> [!NOTE]
> **开放问题**
>
> **载荷偏移 `0x1100` 的 `0x00000007` 除了复位 PLM 还有别的用吗？** 它的主要作用已定论：DMEM `0x1900` 是经 IMEM `0x1d3b` 到达的 `f100_field_save_restore` 槽，`0x7` 使经过 `secure_teardown` 的退出把 SEC2 复位 PLM 留在 `0xff`，而不是常见的 `0x8f` 污染。它是否还有任何进一步的作用从未被证实。

> [!NOTE]
> **开放问题**
>
> **`0x008200FC` 可写吗，冷卡上它读出什么？** 一次扫描报告 `0xffffffff`，另一次报告 `0x000003FF`。九 PLM Gen2 分支的尝试返回 `status=0xffff`，没有记录回读。

更多见 [开放问题](../frontier/open-questions.md) 和 [状态板](../frontier/status-board.md)。通往这里的路上尝试过并失败的内容，见 [死路](../history/dead-ends.md)。
