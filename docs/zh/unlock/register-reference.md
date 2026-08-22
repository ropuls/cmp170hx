# 寄存器参考

## 本页涵盖

本项目任何人在 NVIDIA CMP 170HX（GA100）上读过、写过或争论过的每一个硬件寄存器，按功能块组织，带地址、名称、作用、原厂值、存在时的解锁值、是否有[权限级掩码](privilege-level-masks.md)门控主机写入、内容是否经得起功能级复位（FLR），以及哪个补丁或工具碰它。

本页最重要的事实：**整个正式解锁是四道权限级掩码打开加四次普通主机寄存器写入。** 这里的其他一切都是背景、诊断或未发布工作。

| 步骤 | 地址 | 写入的值 | 机制 |
|---|---|---|---|
| PLM 打开 0 | `0x001fa7cc` | `0xfffff0ff` | Booter Load 重触发 |
| PLM 打开 1 | `0x009a0148` | `0xffffffff` | Booter Load 重触发 |
| PLM 打开 2 | `0x001fa7c4` | `0xffffffff` | Booter Load 重触发 |
| PLM 打开 3 | `0x00823804` | `0xffffffff` | Booter Load 重触发 |
| 解锁写入 0 | `0x0082381c` | `0x88888888` | 普通主机 `GPU_REG_WR32` |
| 解锁写入 1 | `0x00823820` | `0x00000008` | 普通主机 `GPU_REG_WR32` |
| 解锁写入 2 | `0x009a0204` | `0x02779000`（8 GB 卡）/ `0x02669000`（10 GB 卡） | 普通主机 `GPU_REG_WR32` |
| 解锁写入 3 | `0x00100ce0` | `0x0000020B`（8 GB 卡）/ `0x0000028A`（10 GB 卡） | 普通主机 `GPU_REG_WR32` |

以上一切直接读自正式 `master` 的 `driver/patches/0001-sec2-postbl-plm-ss-cfg.patch`。叙述版见 [解锁的工作原理](how-it-works.md)，逐补丁分解见 [驱动补丁](driver-patches.md)。

---

## 阅读约定

### 地址对比值

这是存档中最常见的困惑来源，因此它有自己的规则：**`Address` 列中的一切都是 BAR0 字节偏移；`Value` 列中的一切都是 32 位数据。** 若干解锁值本身看起来像合理的地址，若干载荷槽包含根本不是 BAR0 寄存器的 Falcon IMEM 地址。底部 [不是 BAR0 地址的数字](#numbers-that-are-not-bar0-addresses) 一节列出每一种这类陷阱。

本页所有 BAR0 偏移都是相对 BAR0（区域 0）起点的绝对偏移。手工读一个：

```bash
# BAR0 offset 0x009a0204 on the GPU at 0000:05:00.0
sudo dd if=/sys/bus/pci/devices/0000:05:00.0/resource0 \
        bs=4 count=1 skip=$((0x009a0204 / 4)) 2>/dev/null | xxd -e -g4
```

### 列含义

- **PLM-gated**：对这个寄存器的 PL0（普通主机）写入是否在权限级掩码从重型安全（HS）Falcon 上下文打开之前被静默丢弃。"Silently" 是字面的：早期流水线记录了三遍 `Write failed - wrote 0x2779000, read 0x2449000`，任何地方都没有报错。
- **FLR**：值是否经得起 `echo 1 > /sys/bus/pci/devices/<bdf>/reset`。整张映射中只有三个寄存器已知能存续，那种不对称性是计算解锁比显存解锁早数周发布的原因。
- **Touched by**：正式补丁、未发布分支补丁或净室工具。"read only" 意思是项目里有东西为诊断读它，但没有任何东西写它。

### 哨兵值

读出以下之一的不是数据：

| 哨兵 | 含义 |
|---|---|
| `0xbadf0000`（与 `0xffff0000` 相与） | 通用 PRI 毒值，Booter 自己的 `csb_read` 会测试它 |
| `0xbadf1100` | 在 SEC2 注入点看到的 `0x85080` / `0x85084` 上的 PRI 毒值，以及 `0x00021c14` / `0x0000a800`（GA100 上缺失的寄存器） |
| `0xbadf1201` | 无驱动加载的 170HX 上 `0x00504204` 和整个 `0x00407xxx` SKED 块返回 |
| `0xbadf2010`、`0xbadf2011`、`0xbadf2013`、`0xbadf2017`、`0xbadf2018` | 对筛选（禁用）FBPA 的 CSTATUS 读取 |
| `0xbadf5040` | 权限违规哨兵；`0x00409664` / `0x00409668` 在探测过的每张 Ampere 卡上返回它，XVE 影子对非特权读者返回它 |
| `0xbadf5108` | 主机 PL0 读安全 PLM |
| `0xdead5ec1` | 对 `0x001180f8` 的 PLM 毒化读取；注意该模式中位 26 已读作 1，产生假 BOOT_STAGE_3 "DONE" |
| 整个孔径 `0xffffffff` | BAR0 映射已死（卡掉出总线），不是解锁的 PLM |

---

## SEC2 Falcon（BAR0 + `0x840000`）

SEC2 是漏洞利用劫持其签名 `booter_load` ucode 的安全协处理器。它是 Falcon v4 核心，不是 RISC-V（`FALCON_HWCFG2` 位 10 读出 0）。见 [Falcon 与 Booter](falcon-and-booter.md)。

| 地址 | 名称 | 功能 | 备注 |
|---|---|---|---|
| `0x00840040` | `FALCON_MAILBOX0` | Booter 运行后的状态字 | 每次携带载荷的运行 `0x31`；目标寄存器的回读是唯一有效判定 |
| `0x00840044` | `FALCON_MAILBOX1` | 第二个邮箱 | |
| `0x0084007c` | `SFTRESET` | 软复位；写 1 并回读 | 只在 `SCTL` HSMODE（位 1）被设置时 |
| `0x00840084` | `FALCON_RM` | Falcon 资源管理器 scratch | |
| `0x008400f4` | `FALCON_HWCFG2` | 位 10 = RISCV | SEC2 上读出 0（Falcon 核心） |
| `0x00840100` | `FALCON_CPUCTL` | 位 1 = STARTCPU 脉冲，位 4 = HALTED（RO） | |
| `0x00840104` | `FALCON_BOOTVEC` | 启动向量 | |
| `0x0084010c` | `FALCON_DMACTL` | 轮询直到 scrub 位 `0x6` 清除 | `0xffffffff` 读意味着窗口尚未响应，不是失败 |
| `0x00840110` | `FALCON_DMATRFBASE` | DMA 基址 | |
| `0x00840114` | `FALCON_DMATRFMOFFS` | DMEM/IMEM 偏移 | |
| `0x00840118` | `FALCON_DMATRFCMD` | DMA 命令 | |
| `0x0084011c` | `FALCON_DMATRFFBOFFS` | FB 偏移 | |
| `0x00840128` | `FALCON_DMATRFBASE1` | DMA 基址高位 | |
| `0x00840180` / `0x00840184` | `IMEMC` / `IMEMD` | IMEM 端口，自动递增，每 256 字节 tag | 安全 tag 位是 `1 << 28`；用于在 0..`0x8700` 范围把已加载的 booter 读出来 |
| `0x00840240` | `SCTL` | 安全控制；HSMODE = 位 1，`AUTH_EN` = `1 << 14` | 引擎复位后观察到 `0x3000` 到 `0x3002` |
| `0x00840284` | SEC2 `DMEM_PLM` | DMEM 权限掩码 | LS 模式下 `0xff`（完全开放） |
| `0x008403c0` | `FALCON_ENGINE` | 位 0 = RESET；脉冲 1 后 0 | 引擎复位门是 `(resetPLM & 0x77) == 0x77` |
| `0x008403c4` | **SEC2 复位 PLM** | 决定 SEC2 能否再次复位 | `0xff` 干净、`booter_unload` 成功后 `0xdf`、`secure_teardown` 后 `0x8f`（阻止 SFTRESET）。`reset_allowed = {0xff, 0xdf}`。**FLR 清除到 `0xff`。** 每个净室触发工具把它读作就绪门 |
| `0x00840480` / `0x00840484` | SEC2 触发后状态 | `0` 到 `0x1` 和 `0` 到 `0x11100` | HS 退出副作用，从不恢复 |
| `0x00840530` | `SCP_P2PRX` | 无驱动复位期间轮询位 3 | |
| `0x008411ec` | `KFUSE_CTL` | 轮询位 0 设置、位 1 清除 | |

> [!NOTE]
> **Falcon 内部地址是另一空间**
>
> 运行中的 Booter 内部，`I[0x1c100]` / `I[0x1c200]` / `I[0x1c000]` 是 Falcon CSB 空间中的 BAR0 主控地址、数据和命令端口（`0x800000f2` = 写，`0x800000f1` = 读），`I[0x1c300]` 是以 `0x1312d00`（20,000,000）播种的看门狗。`I[0x12000]`、`I[0x12100]`、`I[0x12400]` 和 `I[0x12600]` 是 Booter 在 `main()` 早期编程的 Falcon 本地孔径/PLM 字。这些都不是 BAR0 偏移，主机也戳不到任何一个。同样，`0x9100` 是 `FALCON_CSBERRSTAT`，其位 31 是每次 CSB 访问后的失败关闭故障标志。见 [ROP 链](rop-chain.md)。

## GSP RISC-V 与 BSI 安全 scratch（BAR0 + `0x110000` / `0x118000`）

| 地址 | 名称 | 功能 | 备注 |
|---|---|---|---|
| `0x00110624` | GSP `FBIF_CTL` | 孔径控制 | Booter `reg_init (0x68ed)` 写 `0x90`（ALLOW_PHYS_NO_CTX 位 7 加位 4） |
| `0x00110684` | GSP FBIF 伴生 | | `reg_init` 写 `1` |
| `0x0011126c` | GSP RISC-V 伴生 | | `reg_init` 写 `1` |
| `0x00111240` | `RISCV_STATUS` | GSP 核心状态 | `0x35` = RISC-V 活动；`0x0` = 从未启动 |
| `0x00111268` | `RISCV_CPUCTL` | GSP 核心控制 | |
| `0x00118244` / `0x00118248` | WPR 暂存对 | 被 `booter_load_wpr_main (0x22ba)` 读取后清零 | |
| `0x0011824c` / `0x00118250` | memcfg 交接 | 由 `memcfg_program (0x79cc)` 写；`memcfg_apply_poll` 只在 `0x11824c` 位 0 被设置时运行 | |
| `0x001180f0` | AON LMR 影子 | 内存范围值的常开影子 | **FLR 后恢复**；不是持久性杠杆 |
| `0x001180f8` | `NV_PGC6_BSI_SECURE_SCRATCH_14` | 位 26 = `BOOT_STAGE_3_HANDOFF`（INIT 0，DONE 1） | GPU 侧由 HS 上下文中的 SEC2 设置；主机驱动只轮询它。启动挂起 `0x65` 就是该轮询超时。Booter 自己的入口检查要求位 [31:28] == 0（否则错误 `0x29`）和位 [27:24] == 0（否则 `0x88`）。**正式链从不写这个寄存器。** |
| `0x001182d0` | AON 安全 scratch | | PL3 可达 |
| `0x00118f78` | 辅助 scratch | | 每张调查过的卡都读 `0x00000000` |

> [!NOTE]
> **开放问题：正确的 `0x001180f8` 交接值**
>
> `0x11000000`、`0x13100000` 和 `0x17100000` 都被提出过。`0x17100000` 被实测满足 Booter 的 `0x29` 检查和主机 DONE 轮询，但写它也让 `FBPA_008` 和 `FBPA_00C` 的 Booter PLM 打开都失败。正式链通过从不碰该寄存器绕开整个问题。见 [开放问题](../frontier/open-questions.md)。

## WPR 块（`0x001fa7xx` / `0x001fa8xx`）

写保护区域。解锁必须在每次 Booter 重触发周围保存并恢复 WPR2，因为否则第二次 `booter_load` 会以"WPR2 already up"中止。

| 地址 | 名称 | 功能 | 原厂 | 谁碰 |
|---|---|---|---|---|
| `0x001fa7c4` | `WPR_PLM` | WPR 区域寄存器上的权限掩码 | `0x0004cb8f` | 正式补丁 0001，PLM 索引 2，打开到 `0xffffffff` |
| `0x001fa7c8` | `MMU_LOCK` PLM | 写半字节 `0x8` = 仅 L3/HS | `0x0004cb8f` | 只读 |
| `0x001fa7cc` | `WPR_CFG_PLM`（WPR 掩码 PLM） | WPR 允许掩码上的权限掩码 | `0x0004cb8f` | 正式补丁 0001，PLM 索引 0，打开到 **`0xfffff0ff`**，不是 `0xffffffff` |
| `0x001fa814` | WPR 读允许掩码 | 位 [7:4] 中的模式字段 | | Booter `fbif_set_bit800 (0x8307)` 在掩码 `0x0ffff8ff` 下设置位 `0x800` |
| `0x001fa818` | WPR 写允许掩码 | 同上 | | 同上 |
| `0x001fa81c` | `WPR1_ADDR_LO` | 位 [31:4] 中的值，`<< 12` | | 被净室 re-fire 链清除 |
| `0x001fa820` | `WPR1_ADDR_HI` | | | 同上 |
| `0x001fa824` | `WPR2_ADDR_LO` | 位 [31:4] 中的值，`<< 12` | 空/INIT = `0x0fffffff` | **正式补丁在每次 Booter 尝试前保存并重新武装**；净室 teardown 值是 `0x1ffffe00` |
| `0x001fa828` | `WPR2_ADDR_HI` | `HI = 0` 使 `kgspIsWpr2Up()` 返回 false | 空/INIT = `0` | 同上 |
| `0x001fa82c` / `0x001fa830` | memlock 范围低/高 | | AHESASC 后 `0x1ffffff0` / `0x00000000`（空） | 只读 |

野外实测 WPR2 值：10 GB 卡上划分出 `0x02777000`，在 10 GB 对比 40 GB A/B 的两支都是；40 GB 净室触发后 `[0x1ffffe00, 0x027fee00]`；PG199 参考板上 `07f68000/07fefe00`。RM 每次启动把 WPR2 重新分配到新的 FB 地址，这就是为什么静态烘入的 WprMeta 以 `0xf0000000` 收尾（失败），而活值给出 `0x11000000`。

---

## 权限级掩码：完整目录

PLM 是守护另一个寄存器或寄存器组的半字节编码权限 32 位字。`0xffffff8f` 是常见锁定状态（写仅第 3 级）；`0xffffffff` 完全开放；`0x0004cb8f` 是 WPR 对的锁定状态。编码见 [权限级掩码](privilege-level-masks.md)。

### 正式 `master` 写的（四个条目，此顺序，每个最多两次尝试）

| 索引 | 地址 | 代码中的标签 | 写入的值 | 备注 |
|---|---|---|---|---|
| 0 | `0x001fa7cc` | `WPR_CFG` | `0xfffff0ff` | 例外：**不是** `0xffffffff` |
| 1 | `0x009a0148` | `FBPA` | `0xffffffff` | 也是内置回退载荷目标 |
| 2 | `0x001fa7c4` | `WPR` | `0xffffffff` | |
| 3 | `0x00823804` | `FEAT` | `0xffffffff` | 唯一的 AON 条目；经得起 FLR |

每次尝试从循环前保存的值重新武装 `0x001fa824` / `0x001fa828`，为那一对（地址，值）重填整个 `0xf800` 字节载荷，触发 Booter Load，并回读目标。成功纯粹由回读相等定义。8 GB 卡上 2026-07-19 dmesg 捕获的时序：PLM[0] 在 11.32 s、PLM[1] 11.50 s、PLM[2] 11.68 s、PLM[3] 11.86 s，每轮 Booter 约 180 ms。

### Gen2 家族分支添加的（共九个条目）

分支 `Gen2`、`debug-gen2`、`far` 和 `deced` 携带相同修改的补丁 0001，其 `plmTable[]` 有九行、循环边界是 `plmIdx < 9`。五个额外条目，全部写 `0xffffffff`：

| 地址 | 标签 | 目的 |
|---|---|---|
| `0x00088ff4` | `XVE` | XVE 配置空间影子 PLM；因为否则主机读 PCIe 影子会返回 `0xbadf5040` |
| `0x00088ab4` | `XVE_B` | 第二个 XVE 能力 PLM |
| `0x00088ff8` | `XVE_C` | 第三个 XVE 能力 PLM |
| `0x00823b00` | `FEAT2`（行重映射器 PLM） | 原厂 `0xffffff8f`；一次 HS 内扫描在 FLR 后读它 `0xffffffff`，因此它可能 AON，但打开它没让几何持久 |
| `0x008200fc` | `OPT_PLM`（别名 `FUSE_SS_PLM`） | 一次扫描中原厂卡已读 `0xffffffff`、另一次读 `0x000003ff`；**正式 master 从不写** |

> [!WARNING]
> **实验性**
>
> 九条目表只存在于未发布分支。全部九个被报告在一台双卡机器的两张 GPU 上首次尝试即成功。见 [PCIe Gen2](pcie-gen2.md)。

### FB 几何 PLM 集（仅净室工具）

`refire_chain_v9.py` 用 `FB_GEO_PLMS = [0x00100b10, 0x009a0148, 0x009a014c, 0x009a0008, 0x009a000c]`。最早 HS 配方用加 `0x00100b38` 的六条目变体。打开其中任何一个都需要 L3。

**这些没有一个把 FB 几何移入常开域。** 一个专门存续脚本（`geo_flr_survival_map_20260716.sh`）打开全部六个加 `FUSE_SS_PLM`，发现 CFG1、CSTATUS 和 LMR 在 FLR 上仍然恢复。那个负面结果就是正式设计每次加载模块在 GSP 启动路径内重新应用几何、而不是试图让它粘住的原因。

### 26 寄存器 PLM 调查

`nuke.sh` 以冷启动基线加九轮 FLR 分类这个候选集：

```text
0x008200D0  0x008200D4  0x008200D8  0x008200DC  0x008200E0  0x008200E4
0x008200E8  0x008200EC  0x008200F0  0x008200F4  0x008200FC
0x00823800  0x00823804  0x00823B00
0x009A0008  0x009A000C  0x009A0148  0x009A014C  0x009A0168  0x009A03F0
0x009A0554  0x009A0BFC
0x00100B10  0x00100B38  0x00100B84  0x00100B9C
```

580.159.04 上无驱动加载、两次 FLR 后扫描的实测回读：

| 寄存器 | 值 | 读法 |
|---|---|---|
| `0x8200D4`、`D8`、`E0`、`E4`、`E8`、`EC`、`F0`、`F4`、`FC`、`0x823800`、`0x823804`、`0x823B00` | 开放 | 报告 UNLOCKED |
| `0x8200D0`、`0x8200DC` | `0xffffff8f` | 锁定 |
| `0x9A0008`、`0x9A000C`、`0x9A0148`、`0x9A014C`、`0x9A03F0` | `0xffffff8f` | 锁定 |
| `0x9A0168`、`0x9A0554`、`0x00100B9C` | `0xffffffcf` | 锁定 |
| `0x9A0BFC` | `0x00000000` | |
| `0x00100B10`、`0x00100B38` | `0xffffff8f` | 锁定 |
| `0x00100B84` | `0xffffff88` | 锁定 |

解锁后加载**原厂**驱动，只有 `0x00823804` 保持打开。`0x001fa7cc` 和 `0x001fa7c4` 重新锁定到 `0x0004cb8f`、`0x009a0148` 读 `0xffffff8f`，对 `0x009a0008`、`0x00100b10` 和 `0x00100b38` 的主机写测试读 `0xfffffe8e`。

---

## 显存几何：FBPA 与 MMU

完整叙述在 [显存几何](memory-geometry.md)；硬件背景在 [内存子系统](../hardware/memory-subsystem.md)。

### 解锁实际写的两个寄存器

| 地址 | 名称 | 功能 | 原厂（两个 SKU） | 8 GB 卡解锁 | 10 GB 卡解锁 | PLM-gated | FLR |
|---|---|---|---|---|---|---|---|
| `0x009a0204` | `NV_PFB_FBPA_CFG1`（广播） | 每个内存分区的寻址深度 | `0x02449000` | `0x02779000` | `0x02669000` | 是，经 `0x009a0148` | **否** |
| `0x00100ce0` | MMU 本地内存范围（LMR） | MMU 看到的 FB 总大小 | `0x00000208`（8 GB）/ `0x00000288`（10 GB） | `0x0000020B` | `0x0000028A` | 是 | **否** |

两者都由补丁 0001 在 SS0/SS1 之后无条件写，运行时从 `pGpu->idInfo.PCIDeviceID >> 16` 选择：

```c
if (devId == SEC2_POSTBL_TIMING_CMP_170HX_8GB_PCI_DEVICE_ID) {   /* 0x20C2 */
    cfg1Value = 0x02779000U;  // 8GB card: 64GB unlock
    lmrValue  = 0x0000020BU;
} else {
    cfg1Value = 0x02669000U;  // 10GB card: 40GB unlock
    lmrValue  = 0x0000028AU;
}
GPU_REG_WR32(pGpu, 0x0082381cU, 0x88888888U);
GPU_REG_WR32(pGpu, 0x00823820U, 0x00000008U);
GPU_REG_WR32(pGpu, 0x009a0204U, cfg1Value);
GPU_REG_WR32(pGpu, 0x00100ce0U, lmrValue);
```

**CFG1 编码。** 位 [23:16] 的档字节是杠杆：`0x44` 原厂（12 行位，每 FBPA 512 MiB）、`0x66`（14 行位，每 FBPA 2048 MiB）、`0x77`（15 行位，每 FBPA 4096 MiB）。总容量是寻址深度乘以熔丝决定的活动 FBPA 数量；CFG1 不改变存在多少 FBPA。探测目录字段解码是 `SUBP[1:0]`、`COL[15:12]`、`ROWA[19:16]`、`BANK[25:24]`；在观察过的每个 HBM 部件上，COL 保持 `0x9`、BANK 保持 `0b10`。GDDR6 部件读出不同的 COL 半字节，这就是为什么 `0x9` 是内存类型常量而不是"5 堆栈"标志。

`0x02779000` 字面就是原厂 A100 PCIe 80 GB CFG1 字，`0x02669000` 是原厂 A100 PCIe 40 GB 和 SXM4 40 GB 字。解锁恢复真实的 A100 几何，而不是发明常量。

**LMR 编码。** `size_MiB = MAG[9:4] << SCALE[3:0]`，等价 `bytes = MAG << (SCALE + 20)`。MAG 按 SKU 恒定（8 GB 卡 32，10 GB 卡 40），等于活动 FBPA 数量的两倍。SCALE 才是解锁改变的。

| LMR | MAG | SCALE | 解码为 | 状态 |
|---|---|---|---|---|
| `0x00000208` | 32 | 8 | 8192 MiB | 原厂 8 GB 卡 |
| `0x00000288` | 40 | 8 | 10240 MiB | 原厂 10 GB 卡 |
| `0x0000020B` | 32 | 11 | 65536 MiB | 正式 8 GB 配置 |
| `0x0000028A` | 40 | 10 | 40960 MiB | 正式 10 GB 配置 |
| `0x0000028B` | 40 | 11 | 81920 MiB | `80` 分支上的惰性元数据，但被净室 refire 脚本真实触发过，见下方 |
| `0x0000020A` | 32 | 10 | 32768 MiB | PG199 Drive 参考板上的原厂值 |

**仅有 CFG1 不够。** 一次受控三方对比：完全不写内存给出 CPU-RM 失败 `0x24`（`kbusVerifyBar2`）；40 GB CFG1 配原厂 10 GB LMR 仍然 `0x24`；CFG1 加匹配的 LMR 到达 `0x25`（StateLoad）。GSP-RM 另外把 LMR 视为主控：CFG1 在 40 GB 但 LMR 保持 `0x288` 时，`kgspBootstrap` 把 CSTATUS 从 `0x800` 恢复为 `0x200`。

### FBPA 广播孔径（`0x009a0000` 到 `0x009a3fff`）

| 地址 | 名称 | 原厂 / 观察 | 备注 |
|---|---|---|---|
| `0x009a0008` | FB 几何 PLM | `0xffffff8f` | 在净室 `FB_GEO_PLMS` 清单中 |
| `0x009a000c` | FB 几何 PLM | `0xffffff8f` | 同上 |
| `0x009a0148` | **FBPA PLM** | `0xffffff8f` | 被正式补丁 0001 打开到 `0xffffffff` |
| `0x009a014c` | FB 几何 PLM | `0xffffff8f` | 净室清单 |
| `0x009a0164` | `FBPA_NUM_ACTIVE`（`NUM_ACTIVE_FBPS`） | 8 GB 卡上 `0x00000008` | |
| `0x009a0168` | PLM 候选 | `0xffffffcf` | 仅调查 |
| `0x009a0200` | `FBPA_CFG0_BROADCAST` | 170HX 和 A100 40G/80G 上 `0x07981800`；参考 GA100/Drive 部件上 `0x06981800` | 每个活动逐 FBPA 实例相同 |
| `0x009a0204` | `FBPA_CFG1_BROADCAST` | 见上方 | 参考 GA100 和 A100 32 GB Drive 读 `0x22779000`，与 CMP 目标只在位 29 上不同 |
| `0x009a020c` | `FBPA_CSTATUS` 广播 | 解锁 170HX 上 `0x00001000`，对比 A100 80 GB 上 `0x00000fff` | |
| `0x009a0224` | `TIMING1` | `0x12050d12`（R2W 18，W2R 13，R2P 5，W2P 18） | 编程的 CONFIG 值 |
| `0x009a0290` | `CONFIG0` | `0x1255b93c` | 位 31 `USE_TIMING_REGS` = 0 |
| `0x009a0294` | `CONFIG1` | `0x38d4841b` | CL 27，WL 8，RD_RCD 18，WR_RCD 13，QPOP_OFFSET 14 |
| `0x009a0298` | `CONFIG2` | `0x88130b11` | tWR 19，W2R_BUS 8，R2W_BUS 8，RPRE 1，WPRE 1，CDLR 11 |
| `0x009a02b0` | `TIMING0_GEN` | tRC 60，tRFC 441，tRAS 42 | 生成影子，实际生效的那个 |
| `0x009a02b4` | `TIMING1_GEN` | R2W 29，W2R 20，W2P 28 | |
| `0x009a02b8` | `TIMING2_GEN` | RD_RCD 18，WR_RCD 13，RRD 6 | |
| `0x009a02c0` | `TIMING4_GEN` | FAW 21 | 原始 `TIMING4` 持有陈旧的 FAW 40 |
| `0x009a02d8` | `TIMING9_GEN` | CCDL 4，CCDS 2 | |
| `0x009a02e0` | `TIMING16_GEN` | RP 18 | |
| `0x009a0300` | `FBPA_MRS_0` | 170HX / A100 / Drive A100 上 `0x00000003` | GDDR6 部件不同 |
| `0x009a0304` | `FBPA_MRS_1` | 每张卡 `0x00100000` | |
| `0x009a0320` | `FBPA_MRS_8`（MR8 密度） | 全部 15 张卡 `0x00200000` | **不是**容量限制 |
| `0x009a0334` | `FBPA_MRS_2` | `0x00200019`（8 GB CMP）、`0x002000cf`（10 GB CMP 和 A100 40G） | |
| `0x009a0338` | `FBPA_MRS_WL_RL` | `0x003000eb`（8 GB CMP）、`0x003000ea`（10 GB CMP） | |
| `0x009a038c` | `FBPA_HBM_CFG0` | 170HX / A100 上 `0x000000a7`；Drive A100 上 `0x000000a6` | `dual_rank[0]`、`dual_rank_bank[1]`、`SID_VAL[11]` |
| `0x009a03f0` | PLM 候选 | `0xffffff8f` | 仅调查 |
| `0x009a0470` | `FBPA_ECC_CTRL` | `0`，`MASTER_EN` 只读 | 见 [ECC](../frontier/ecc.md) |
| `0x009a0554` | PLM 候选 | `0xffffffcf` | 仅调查 |
| `0x009a0838` / `0x009a083c` | `FBPA_VEND_ID_C0` / `C1` | 全部 15 张卡 `0x00000000` | 厂商 ID 不在这里暴露 |
| `0x009a0974` | `FBPA_TRAINING_STATUS` | `0x00000000` = FINISHED | SUBP0[1:0]、SUBP1[3:2]，值 2 = ERROR |
| `0x009a0bfc` | PLM 候选 | `0x00000000` | 仅调查 |
| `0x009a3cb4` / `b8` / `bc` | `I1500_INSTR` / `MODE` / `DATA` | `0x0000000f` / `0x00000008` / `0x40000000` | IEEE 1500 HBM 测试端口 |
| `0x009a3cc0` / `cc4` / `cc8` | `I1500_SHADOW_WIR` / `WDR` / `STATUS` | `0x000000f0`（RO）/ 逐裸片 / `0x00000000`（空闲） | 8 GB 卡上 `WDR` 是 `0x8000f000`，10 GB 卡上是 `0x8273ff83` |

### 逐 FBPA 单播孔径

每个 FBPA 寄存器的实例 *n* 位于 `0x00900000 + n*0x4000`，n = 0..23：

| 寄存器 | 地址 | 备注 |
|---|---|---|
| 逐 FBPA `CFG0` | `0x00900200 + n*0x4000` | 每个活动实例、两个 SKU 都是 `0x07981800` |
| 逐 FBPA `CFG1` | `0x00900204 + n*0x4000` | 只在无 devinit 的无驱动上下文中必须手写 |
| 逐 FBPA `CSTATUS_RAMAMOUNT` | `0x0090020c + n*0x4000` | 验证目标：原厂 `0x200`，40 GB 档 `0x800`，64/80 GB 档 `0x1000` |

> [!NOTE]
> **步长是 `0x4000`，不是 `0x400`**
>
> 一份已裁决文档带一个掉零笔误。步长是 `0x4000`，由广播孔径是 `0x009a0000`-`0x009a3fff`（恰好 `0x4000` 宽）佐证。

逐 FBPA 容量按 LMR SCALE 为 `2^(SCALE+1)` MiB，恰好是 CSTATUS_RAMAMOUNT 报告的值。交叉检查：`0x800` x 20 个 FBPA = 40960 MiB；`0x1000` x 16 个 FBPA = 65536 MiB。被筛选的 FBPA 返回 `0xbadf20xx` 哨兵。

**在正式驱动路径中一次广播写入就够**，正式驱动完全不写逐 FBPA 寄存器。在无 devinit 的无驱动运行时中广播单独不移动 CSTATUS，必须手工写全部 24 个实例。广播是 PRI priv-ring 硬件机制还是软件步骤从未被直接仪器化。

### MMU / FB hub

| 地址 | 名称 | 值 | 备注 |
|---|---|---|---|
| `0x00100800` | `FBHUB_NUM_ACTIVE_LTCS` | `10de:20c2` 上 `0x00000010`（16）；`10de:2082` 上 `0x00000014`（20） | A100 PCIe 40G/80G 上也是 `0x14` |
| `0x00100b10` | FB 几何 PLM | `0xffffff8f` | 净室 `FB_GEO_PLMS` 条目 |
| `0x00100b38` | FB 几何 PLM | `0xffffff8f` | 仅六 PLM 变体 |
| `0x00100b84` | PLM 候选 | `0xffffff88` | 仅调查 |
| `0x00100b90` | `FBHUB_MEM_PART_BCFG0` | `0x00000603` | 每张卡都匹配已记录 init |
| `0x00100b98` | `SYSMEM_HSHUB_CONNECTION_CFG` | `0x00000003`（BOTH，PCIe 路由） | |
| `0x00100b9c` | PLM 候选 | `0xffffffcf` | 仅调查 |
| `0x00100ce0` | MMU 本地内存范围 | 见上方 | |
| `0x00100ec0` | `MMU_NUM_ACTIVE_LTCS` | 10 GB SKU 和全部三个 A100 SKU 上 `0x05001414`；8 GB SKU 报告 `0x04001410` | 按 SKU 的拆分是开放问题，不是异议；`...1410` 与 16 个 LTC 一致，`...1414` 与 20 个一致 |

### L2 / LTC

| 地址 | 名称 | 值 | 备注 |
|---|---|---|---|
| `0x0017e22c` | L2/LTC 地址映射寄存器 | 原生 `0x00280404` | 从没有被任何东西编程，40 GB 仍然工作 |
| `0x0017e2a0` / `0x0017e2a4` | 逐 LTC 解码 | | 被净室 v8 工具瞄准 |
| `0x001402b4` | LTC 伴生 | 尝试写 `0x00a00030` | 没有移动 40 GiB 折叠 |

净室 v7 到 v8 的改动把 `DECODE_VAL` 从 `0x60000300`（每通道 2 GB）驱动到 `0x10000300`（每通道 4 GB）。在 170HX 上该值全程保持 `0x70000300`，仍然无法解释。见 [80 GB 问题](../frontier/80gb.md)。

---

## 功能覆盖与计算（`0x008238xx`）

这个块是计算解锁。叙述在 [计算节流](compute-throttle.md)。

| 地址 | 名称 | 原厂 170HX | 解锁 | PLM-gated | FLR | 谁碰 |
|---|---|---|---|---|---|---|
| `0x00823800` | `FEAT_OVR_ECC_PLM` | `0xffffff8f`（A100 SXM4 40G 单独读 `0x0000abcf`） | 打开时 `0xffffffff` | 是 PLM | 未知 | Gen2 分支 `xp3gTable`，master 从不 |
| `0x00823804` | **`FEAT_OVR_PLM`** | `0xffffff8f` | `0xffffffff` | 是 PLM | **存续**（AON 岛） | 正式补丁 0001，PLM 索引 3 |
| `0x00823808` | `FEAT_OVR_QUADRO` | **逐裸片且无法解释：见下方说明** | | | | 只读 |
| `0x0082380c` | `FEAT_OVR_ECC` | `0x00888888` | | | | 只读 |
| `0x00823810` | `FEAT_OVR_ECC_1` | `0x002aaaaa` | | | | 只读 |
| `0x00823814` | `FEAT_READOUT_0` | `0x00000233`（RO）；参考 GA100 板读 `0xef8ff100` | | RO | | 字段布局无文档 |
| `0x00823818` | **`FEAT_READOUT_1`** | `0x016db6ed` | **`0x00000000`** | RO | | 最干净的"此卡是否已解锁"测试 |
| `0x0082381c` | **`FEAT_OVR_SM_SPEED_SELECT`（SS0）** | 各异：`0x53540175`、`0x12103060` | `0x88888888` | 是，经 `0x00823804` | **存续** | 正式补丁 0001 |
| `0x00823820` | **`FEAT_OVR_SM_SPEED_SELECT_1`（SS1）** | 各异：`0x00000000`、`0x00000003` | `0x00000008` | 是 | **存续** | 正式补丁 0001 |
| `0x00823824` | `FEAT_OVR_ROW_REMAP` | 两个 170HX SKU 上 `0x00000000` | | | | 只读 |
| `0x00823828` | `FEAT_READOUT_2` | 170HX 上 `0x00000000`；所有 A100 和 Drive 部件上 `0x00000007` | | RO | | 只读 |
| `0x0082382c` | `FEAT_READOUT_2`（一份转储中的别名） | `0x0000000a` | | RO | | 两份转储之间命名未定 |
| `0x00823b00` | 行重映射器 PLM（`FEAT2`） | `0xffffff8f` | 打开时 `0xffffffff` | 是 PLM | 一次扫描在 FLR 后读它打开 | 仅 Gen2 家族补丁 0001 |

> [!NOTE]
> **开放问题：`0x00823808` `FEAT_OVR_QUADRO` 每次转储读法不同**
>
> 逐裸片且无法解释。观察到：`0x00100183`（原厂，PLM 范围扫描，中）、`0x00000081`（解锁后探测，中）、`0x00000181` / `0x00000182`（两台物理 170HX，高，13 个分档差异之一）、`0x01000282`（A100 80 GB）。只读。**开放问题：** 为什么值在三份转储间不同；解锁或驱动中的什么东西可能在碰 Quadro 对比消费级分类字，那可能是驱动可见功能类的杠杆。下一步：在一张卡上正式序列每个阶段前后重读该寄存器。

**SS0/SS1 编码什么。** `0x0082381c` 保存 IMLA0-3、FMLA16、FMLA32、FFMA 和 DP 的八个 4 位字段；`0x00823820` 保存第九个字段 IMLA4。每个半字节最好读作 `[enable | 3 位速度]`，因此 `0x8` 表示"覆盖使能，速度 0 = 满速"。`0x88888888` 因此对全部八个 SS0 单元说"覆盖使能、满速"，`0x00000008` 对 IMLA4 做同样的事。存档中任何原厂转储都没有任何半字节大于等于 `8`。这个编码是推断的，没有文档。

> [!CAUTION]
> **不要把 SS0/SS1 回读用作逐部件参考**
>
> 它们是运行时状态，不是熔丝状态。同一 A100 80 GB 设备 ID 的两份存档转储互相矛盾（`0x00112011`/`0x00000002` 对比 `0x00343015`/`0x00000004`）。改用 `0x00823818 == 0`。

两个写入都必需；只写一个不够。两者在两个 SKU 和全部十二个未发布分支中相同：没有任何分支实验过不同的计算值。

---

## 熔丝与 OTP 影子（`0x0082xxxx`）

除非注明，这些是只读熔丝感知反射。见 [熔丝与 OTP](../hardware/fuses-and-otp.md)。

### 熔丝控制

| 地址 | 名称 | 170HX | 备注 |
|---|---|---|---|
| `0x00820000` | `FUSE_FUSECTRL` | `0xe0040000` | 队列中全部 15 张卡相同 |
| `0x00820040` | `FUSE_EN_SW_OVERRIDE` | `0x00000000` | 消费级和工程样品部件上 `0x00000001`；170HX 上可写且持久，但不改变任何可观察的东西 |
| `0x00820078` | `FUSE_EN_PROGRAM` | `0x00000001` | 全部 15 张卡 |
| `0x0082007c` | `FUSE_DIS_PROGRAM` | `0x00000000` | GA10x 上 `0xbadf5040` |
| `0x00820080` | `FUSE_BYPASS_STATUS` | `0x00000000` | GA10x 上 `0xbadf5040` |
| `0x00820084` | `FUSE_DIS_SW_OVR` | `0x00000001` | 全部 15 张卡；HS 写入被弹回 |
| `0x0082038c` | `FUSE_QUADRO_WR_SEC`（`OPT_SECURE_FEATURE_OVERRIDE_QUADRO_WR_SECURE`） | `0x00000001` | 允许 `0x00823804` 被打开 |
| `0x008203f0` | **`FUSE_FEAT_OVR_DIS`（`OPT_FEATURE_FUSES_OVERRIDE_DISABLE`）** | `0x00000000` | **主熔断丝，未熔断。这一个零就是整个解锁存在的原因。** |
| `0x008203f4` | `OPT_INTERNAL_SKU` | `0` | |
| `0x0082074c` | `FUSE_OPT_SECURE_GSP` | `0x00000001` | 全部 15 张卡 |
| `0x00820618` | `FUSE_FBPA_MEM_WR_SEC` | `0x00000001` | 全部 15 张卡 |
| `0x00821060` | `OPT_SKU_ID` | `0x00000068`（10 GB，`0x2082`）；`0x00000080`（8 GB，`0x20C2`） | 两台物理探测单元都是 10 GB、读 `0x68`；`0x80` 值来自对 8 GB 卡的净室硅片读取 |

### SM 速度选择熔丝（节流本身）

全部在 170HX 上读 `0x00000005`（3 位字段，0 = 满速，5 = 除以 32），在 A100 SXM4 40 GB、A100 PCIe 40/80 GB、A10、A5000、A6000、Drive A100 和 96 SM `0x20bb` DRIVE-PG199-PROD 部件上读 `0x00000000`。**它们不被解锁改变**：覆盖凌驾于它们。

| 地址 | 名称 | 170HX | 备注 |
|---|---|---|---|
| `0x008200fc` | `FUSE_SS_PLM` / `OPT_PLM` | 一次扫描 `0xffffffff`、另一次 `0x000003ff` | 一个寄存器的两个名字：`OPT_PLM` 是分支代码标签，`FUSE_SS_PLM` 是净室工具名。master 不写 |
| `0x00820224` | `FUSE_SS_DP` | `0x00000001` | 独立 1 位熔丝（0 满，1 降低） |
| `0x0082059c` | `FUSE_SS_FFMA` | `0x00000005` | |
| `0x008207d4` | `FUSE_SS_FMLA16` | `0x00000005` | |
| `0x008207d8` | `FUSE_SS_FMLA32` | `0x00000005` | RTX 3070 读 1 |
| `0x008207dc` | `FUSE_SS_IMLA0` | `0x00000005` | |
| `0x008207e0` | `FUSE_SS_IMLA1` | `0x00000005` | |
| `0x008207e4` | `FUSE_SS_IMLA2` | `0x00000005` | |
| `0x008207e8` | `FUSE_SS_IMLA3` | `0x00000005` | |
| `0x008207ec` | `FUSE_SS_IMLA4` | `0x00000005` | RTX 3070 读 1 |

### PCIe 熔丝

| 地址 | 名称 | 170HX | 探测过的其他每个 Ampere 部件 | 备注 |
|---|---|---|---|---|
| `0x0082057c` | `FUSE_PCIE_GEN23_DIS`（`OPT_PCIE_BOOT_GEN23_DISABLE`） | `0x00000001` | `0x00000000` | **硬只读。** 从主机、HS ROP 和 Booter 载荷尝试；总是以 `rd=0x00000001` 失败。Gen2 仍然工作 |
| `0x00820580` | `FUSE_PCIE_GEN3_DIS`（`OPT_PCIE_BOOT_GEN3_DISABLE`） | `0x00000001` | `0x00000000` | |
| `0x00820520` | `FUSE_PCIE_MAGIC_D` | `0x16680000`（位 25 设置 = `GEN4_SPEED_DISABLED`，NVIDIA bug 2220334） | A100 和 Drive GA100 上 `0x00200000` | 可写性有争议 |
| `0x00820584` | `FUSE_DEVID_SW_OVR_DIS` | `0x00000001` | `0x00000001` | |
| `0x00820394` | `OPT_PCIE_LANE_DISABLE` | `0x00000000` | `0x00000000` | 证明 x4 宽度是板级的，不是熔断的 |
| `0x0082082c` | `CTRL_OPT_PCIE_LANE` | `0x00000000` | `0x00000000` | |
| `0x00820c2c` | `STATUS_OPT_PCIE_LANE` | `0x00000000` | `0x00000000` | |
| `0x008204d8` | `OPT_PCIE_DEVIDA` | `0x000020c2`（8 GB）、`0x00002082`（10 GB） | A100 `0x20b2` | SKU id 熔丝 |
| `0x0082056c` | `OPT_PCIE_DEVIDB` | 两个 SKU 都 `0x000020c2` | A100 `0x20f2`；PG199 `0x000020fb` | 10 GB 卡上 DEVIDA 和 DEVIDB 不一致 |
| `0x00820148` | OTP 备用位 | `0x00000000` | | 永远不可设置 |

> [!NOTE]
> **开放问题：`0x00820520` 可写吗？**
>
> 一份分析注释位 25"（可写）"；一个净室工具把 `0x00200000` 写给它作为可工作 Gen2 链的一部分；PCIe 字段手册列它只读；正式 Gen2 补丁只读它。没人发布过写后回读。Gen4 反正无法测试，因为没有在做的人有 Gen4 主机。见 [Gen3 与 Gen4](../frontier/pcie-gen3-gen4.md)。

### NVLink 熔丝

| 地址 | 名称 | 170HX | 备注 |
|---|---|---|---|
| `0x00820684` | `FUSE_NVLINK_DIS`（`OPT_NVLINK_DISABLE`） | `0x00000007`（[2:0] 全部三位置位） | A100 SXM4 40G / PCIe 40G / PCIe 80G / A10 / A5000 / A6000 / RTX 3090 / 3090 Ti 上 `0x00000000`；RTX 3080 / 3080 Ti 上 `0x00000001`；Drive A100 也是 `0x00000007` |
| `0x00820db8` | `STATUS_OPT_NVLINK` | `0x00000007`（RO 镜像） | 与 Drive A100 共享 |
| `0x008209b8` | `CTRL_OPT_NVLINK` | `0x00000000`（位 15:0，每链路） | 覆盖影子在每张探测过的卡上读零 |
| `0x00820820` | `CTRL_OPT_PERLINK` | | 从未写测 |

`0x00823800`-`0x0082382c` 块中任何地方都不存在 NVLink 的 `FEAT_OVR` 条目，任何分支都不含 NVLink 代码。见 [NVLink](../frontier/nvlink.md)。

### 筛选熔丝

逐卡，不逐 SKU。每台调查过的 170HX 无论哪些 GPC 关闭都落在 70 SM。

| 地址 | 名称 | 观察 | 备注 |
|---|---|---|---|
| `0x00820350` | `OPT_GPC_DISABLE` | 四张不同卡上 `0x85`、`0x45`、`0x13`、`0xa8` | HS 写入被弹回，值被闩锁 |
| `0x00820364` | `OPT_FBP_DISABLE` | `0x00000840`（10 GB 卡）、`0x00000852`（社区转储）、`0x00000009` / `0x00000180`（两台） | |
| `0x00820368` | `OPT_FBPA_DISABLE` | `0x000000c3`（10 GB 卡：FBPA 00/01/06/07 关，20 活动）、`0x00c0330c`（8 GB 卡：FBPA 02/03/08/09/12/13/22/23 关，16 活动） | |
| `0x0082036c` | `OPT_FBIO_DISABLE` | 镜像 `0x00820368` | |
| `0x008202c4` | `OPT_ROP_L2_DISABLE` | 镜像 `0x00820368` | |
| `0x00820398` | `OPT_SPARE_FS` | `0x00000000` | |
| `0x008205c4` | `OPT_GPC_DEFECTIVE` | 若干 DISABLE 三位置位的卡上 `0x00000000`；一台 10 GB 卡上 `0x81` | "disabled" 和 "defective" 是分开的掩码：一些被禁的 GPC 是物理完好的硅片 |
| `0x008205cc` | `OPT_FBP_DEFECTIVE` | `0x00000840`（10 GB 卡） | |
| `0x008205d0` / `0x008205d4` / `0x008205e8` | `OPT_FBPA_DEFECTIVE` / `FBIO_DEFECTIVE` / `ROP_L2_DEFECTIVE` | 各 `0x00c03000` | |
| `0x00820818` | `CTRL_OPT_FBPA` | `0x00000000` | 无覆盖存在 |
| `0x00820838 + i*4` | `FUSE_CTRL_OPT_TPC_GPC(i)` | `0x00000000` | **仅移除（减法）**：写它永远不加回一个 TPC |
| `0x00820938` | `CTRL_OPT_FBP` | `0x00000000` | |
| `0x00820840` | MIG 使能 | 原厂 `0`；设置位 0 启用 MIG 并被报告持久 | **不在正式树中**；仓库级 grep `0x820840` 无结果 |
| `0x00820c00` | `STATUS_HALF_FBPA` | `0` | 没有可恢复的半容量熔丝 |
| `0x00820c14` | `STATUS_OPT_FBIO` | `0x00c0330c`（8 GB 卡） | |
| `0x00820c18` | `STATUS_OPT_FBPA` | `0x00c0330c` / `0x000000c3` | **这是正确地址；`0x00820c14` 是 FBIO** |
| `0x00820c1c` | `STATUS_OPT_GPC` | 总是镜像 `0x00820350` | |
| `0x00820c38 + i*4` | `FUSE_STATUS_OPT_TPC_GPC(i)` | 一张卡上 GPC0/3/5 = `0xff`，其他 = `0x01` | |
| `0x00820d38` | `STATUS_FBP` | 一台机器上 `0x00000180` | |

### 拓扑标量（`0x0002xxxx`）

只读，描述完整 GA100 裸片，不是筛选后的部件。

| 地址 | 名称 | 170HX |
|---|---|---|
| `0x0002241c` | `NV_PTOP_FS4` | 8 GB 卡（`0x20c2`）上 `0x00000000`；10 GB 卡（`0x2082`）上 `0x00000081`，A100 80 GB、RTX 3070、GA10x 和 Drive `0x20bb` 上也是。位 0 是 `GEN2_PCIE`，位 7 是 `GEN2_PCIE_SPEED` |
| `0x00022430` | `PTOP_SCAL_NUM_GPCS` | `0x00000008`（GA10x 读 7） |
| `0x00022434` | `PTOP_SCAL_TPC_PER_GPC` | `0x00000008` |
| `0x00022454` | `PTOP_SCAL_NUM_LTCS` | `0x00000018`（24） |
| `0x00022458` | `PTOP_SCAL_FBPA_PER_FBP` | `0x00000002`（RTX 3090 读 1） |
| `0x0002246c` | `PTOP_SCAL_NUM_NVLINK` | `0x0000000c`（12）；GA10x 读 4 |
| `0x00022470` | `PTOP_FS_STATUS` | `0x0000003f`；位0 TPC，位1 GPC，位2 FBP，位3 ROP，位4 FBIO |
| `0x00120078` | `RING_ENUM_GPC` | 每台 170HX 上 `5`；任何写尝试从未移动 |
| `0x00001404` | `PBUS_SW_SCRATCH(1)` | `0x20042000`，所有调查卡上位 14 = 0 |
| `0x00000000` | `PMC_BOOT_0` | 每个有效 GA100 上 `0x170000a1`；GA10x 对照读 `0xb74000a1` |
| `0x008204bc` | `OPT_SLT_REV` | 由 `ga100_topology_report.py` 读取 |

---

## PCIe：XVE、XP3G 与 XP-PL

本节一切都是**未发布分支材料**。正式 `master` 只含补丁 `0001` 到 `0006`，`constants.yaml` 没有 `pcie:` 块，对其安装器、移除器、README 和构建脚本做 `gen2|gen 2|pcie|iommu|retrain|RMPcieLinkSpeed` 的不区分大小写 grep 返回零命中。见 [PCIe Gen2](pcie-gen2.md) 和 [PCIe 子系统](../hardware/pcie-subsystem.md)。

> [!WARNING]
> **实验性**
>
> 补丁 `0007-pcie-gen2.patch` 存在于分支 `debug-gen2`、`Gen2`、`far` 和 `deced`；`0008-pcie-gen2-probe-retrain.patch` 在 `Gen2`、`far` 和 `deced`。这里没有任何东西合并到 master。

### XVE 配置空间影子（BAR0 基址 `0x88000`）

配置读取每次访问都从该影子新鲜到来，这就是为什么运行时重写它、然后强制 retrain 能让主机重读修正后的能力。PCIe Express 能力位于配置偏移 `0x78`，因此配置 `0x78 + X` 映射到 BAR0 `0x88078 + X`。

| 地址 | 配置偏移 | 名称 | 原厂 | 0007 后 | 备注 |
|---|---|---|---|---|---|
| `0x00088084` | CAP_EXP+0x0c | `LINK_CAP`（LnkCap） | `0x00456101` | `0x00456102` | 位 20（DLL Link Active Reporting Capable）**清除**，这破坏了 0008 的成功测试 |
| `0x00088088` | CAP_EXP+0x10 | `LINK_CTRL_STATUS` | LnkSta `0x1041` | `0x1042` | `PCIE_LINK_SPEED_OF(stat) = ((stat) >> 16) & 0xF` |
| `0x000880a4` | CAP_EXP+0x2c | `LINK_CAP2`（LnkCap2） | `0x00000002`（仅 2.5 GT/s） | `0x00000006`（G1/G2） | 硬件只读，标 `R-EVF`：`setpci` 写入被静默丢弃 |
| `0x000880a8` | CAP_EXP+0x30 | `LINK_CTRL_2`（LnkCtl2） | `0x0000` / 寄存器读 `0x00000001` | 位[3:0] = `0x2`，位[19:16] = `0xF` | A100 这里读 `0x001f0004` |
| `0x0008841c` | | `PRIV_MISC_1` | `0x20340500` | `0x20342d00` | 设置位 11 和 13，清除位 12 和 14；**首次尝试成功且挺过 BooterLoad** |
| `0x0008860c` | | `VSEC_DEVICE` | `0x00000800` | 想要 `0x00000801` | **写入在硅片上失败两次**；回读保持 `0x00000800` |
| `0x00088610` | | `VSEC_HIERARCHY` | `0x00001001` | 清除位 12，设置位 0 | Booter 阶段后的普通主机写入 |
| `0x0008872c` | | LTSSM / `XVE_OVR` | | `0x00000006` | 补丁自己的日志叫它"skip mid-boot retrain"。值 `0x2` 和 `0xa` 在 VFIO 下暴露额外 Gen2 行为，但最终卡死 QEMU 功能 |
| `0x00088ab4` | | `XVE_B` PLM | | `0xffffffff` | Gen2 家族 PLM 表 |
| `0x00088ce4` | | 未命名 | `0x0000003f` | | A100 读 `0x00000014` |
| `0x00088fe8` / `0x00088fec` / `0x00088ff0` | | `XVE_D0` / `D4` / `D8` PLM | | `0xffffffff` | `xp3gTable` 条目 |
| `0x00088ff4` | | `XVE` PLM | | `0xffffffff` | Gen2 家族 PLM 表 |
| `0x00088ff8` | | `XVE_C` PLM | | `0xffffffff` | Gen2 家族 PLM 表 |

### XP3G PHY 速率覆盖块（`0x0008e1xx`）

| 地址 | 名称 | 0007 写入的值 | 备注 |
|---|---|---|---|
| `0x0008e100` | `XP3G_STATUS` 基址 | （读） | |
| `0x0008e110` | `XP3G_OVR0` | `0x00000001` | 更早独立探测中观察到回读 `0x00000004` |
| `0x0008e11c` | `XP3G_OVR3` | `0x00000004` | |
| `0x0008e120` | `XP3G_VAL0` | `0x00000000` | |
| `0x0008e12c` | `XP3G_VAL3` | `0x00200000` | |
| `0x0008e1b0` | `XP3G_PLM` | `0xffffffff` | 干净打开；回读 `0xffffffff` |
| `0x0008e1b4` | `XP3G_PLM4` | `0xffffffff` | |
| `0x0008e1b8` | `XP3G_PLM8` | `0xffffffff` | |
| `0x0008e1bc` | `XP3G_PLMC` | `0xffffffff` | |

一次带 PLM 打开的孤立 XP3G 覆盖把速率字段驱动到 Gen3 能力的 `0x00340036`，链路仍以 Gen1 训练。那驳斥了 XP3G 作为独立杠杆，但它是后来奏效组合的一个组成部分。

### XP-PL 链路配置块（`0x0008cxxx`）

在 Booter 阶段后以**普通主机 BAR0 写入**、无权限提升地写：

| 地址 | 名称 | 操作 | 备注 |
|---|---|---|---|
| `0x0008c040` | `LINK_CONFIG_0` | 位[19:18] `MAX_RATE` = `0x2` | 读改写：清除掩码 `0x000c0000`，然后 OR 进 `2 << 18` |
| `0x0008c044` / `0x0008c048` / `0x0008c04c` | LINK_CONFIG 簇 | （HS 写入被拒） | 与可工作的三个*不同*的簇 |
| `0x0008c080` | 链路 WIDTH | A100 读 `0x00001010` | |
| `0x0008c1c0` | `PL_LINK_RATE` | `= 0x00240036` | A100 读 `0x00040036` |
| `0x0008c2c0` | `CYA_0` | 清除位 2（`DIS_G2`） | 中心杠杆 |

### 0007 写的 OPTB PLM 块

十个寄存器，`0x008200d0`、`d4`、`d8`、`dc`、`e0`、`e4`、`e8`、`ec`、`f0`、`f4`，全部设为 `0xffffffff`，加 `0x00823800` `FEAT_OVR_ECC_PLM` 和 `0x0082057c` `OPT_GEN23`（尝试 `0x00000000`，总是失败）。

### `0007` 的写入计数

`xp3gTable` 有 **23** 个条目：18 次 PLM 打开加 5 次值写入。`VSEC_DEVICE 0x0008860c` 和 `PRIV_MISC_1 0x0008841c` 在表外处理，共 **25 次经 Booter 路由的写入**。每次两次尝试，每次前重新武装 `0x001fa824` / `0x001fa828`。然后六次普通主机写入：`0x00088610`、`0x000880a8`、`0x0008c2c0`、`0x0008c040`、`0x0008c1c0` 和 `0x0008872c`。

### 从注入点被 PROT 墙挡或毒化的寄存器

| 地址 | 行为 |
|---|---|
| `0x00088070`、`0x0008808c`、`0x00088090` | 读返回 0，写被忽略 |
| `0x00085080`、`0x00085084` | 读 `0xbadf1100`；"GSP 写 `0x85084`"为真，但在注入点永远到达不了的特权上 |
| `0x00409664`、`0x00409668` | 每张 Ampere 卡（包括未节流的）上 `0xbadf5040` |

> [!NOTE]
> **不要重复 `0x8808c` 作为 LnkCap2 镜像**
>
> 一份字段手册把 `NV_XVE_LINK_CAPABILITIES_2` 列为"cfg 0xA4 / BAR0 镜像 `0x8808c`"，这在内部自相矛盾。XVE 镜像基址在 `0x88000` 时，配置 `0xA4` 映射到 `0x880a4`，补丁 0007 和独立 `pcielink.sh` 用的都是它。

---

## 图形、SKED 与 FECS：调查过，从未使用

正式树中没有任何东西碰这些。对 `0x504204`、`0x8200fc`、`0x82038c`、`0x8203f0`、`0x823818`、`0x820224`、`0x82059c` 和 `0x820840` 做仓库级 grep 返回零命中。

| 地址 | 名称 | 170HX | 结论 |
|---|---|---|---|
| `0x00407000` | `SKED_HW_BLK` | `0x00004042`（驱动前 `0xbadf1201`） | |
| `0x00407010` | `SKED_PM_UNK10` | `0x00000000` | |
| `0x00407020` | `SKED_TRAP` | `0x00000000` | |
| `0x00407024` | `SKED_TRAP_EN` | `0x3dfffffc`，与 A100 相同；RTX 3090 读 `0xbdfffffc`（仅位 31） | |
| `0x00407054` | `SKED_UNK54` | `0x60000600`（驱动前）或 `0x600000c0`；**A100 和 RTX 3090 上都是 0** | GSP 固件中被引用最多的未记录 SKED 寄存器（13 处引用），也是 13 卡队列中唯一 170HX 非零、对照为零的寄存器。从未写测。驱动在 GR init 期间清除它 |
| `0x00408970` | `gpcMask` | `0xdc`，每次强制尝试后重新断言 | 死路 |
| `0x00409664` | `FECS_FEAT_OVERRIDE` | `0xbadf5040` | 每张 Ampere 卡读阻塞，因此该值不携带信息 |
| `0x00409668` | `FECS_FEAT_READOUT_1` | `0xbadf5040` | 同上 |
| `0x00504204` | `SM_ISSUE_RATE_MODIFIER` | 有驱动 `0x00000005`，无驱动 `0xbadf1201` | **不是**节流：在 13 张对照 Ampere 卡和一个速度选择熔丝全零的 96 SM `0x20bb` GA100 上读 `0x00000005`。主机可写；清零它什么都不改变 |

> [!NOTE]
> **开放问题：`0x00504204` 对已解锁卡施加残余限制吗？**
>
> 没人跑过明显的 A/B：在解锁卡上把它写为零并重跑基准套件。写入原语已经存在。

---

## 载荷偏移表

Booter 的 LS 签名验证（IMEM `0x29c4` 的 `booterVerifyLsSignatures_TU10X`）执行一次长度直接取自 `WprMeta` 中 `sizeOfSignature` 的无界 DMA。驱动把它设为 `SEC2_POSTBL_TIMING_SIGNATURE_SIZE = 0x0000f800`（63,488 字节），DMA 目标是 DMEM `0x0800`，因此载荷与 DMEM `0x0800`..`0xffff` 一一对应（`0x0800 + 0xf800 = 0x10000`，恰好 64 KB DMEM 顶部）。**DMEM 地址 = 载荷偏移 + `0x800`。**

缓冲区每个 dword 首先填 `SEC2_POSTBL_TIMING_FILL_DWORD = 0x000004a7`（15,872 个 dword），然后恰好覆盖 24 个槽：

| 载荷偏移 | DMEM | 值 | 作用 |
|---|---|---|---|
| 全部 | `0x0800`-`0xffff` | `0x000004a7` | 背景填充 dword |
| `0x1100` | `0x1900` | `0x00000007` | 目的未识别 |
| `0x5b40` | `0x6340` | `0xc0deca7e` | **写入栈防护全局的假金丝雀** |
| `0xf754` | `0xff54` | *writeValue* | 值参数，最低尾槽 |
| `0xf758` | `0xff58` | `0xc0deca7e` | 保存金丝雀槽 |
| `0xf75c` | `0xff5c` | `0x00000cbd` | |
| `0xf76c` | `0xff6c` | *writeAddr* | 地址参数 |
| `0xf774` | `0xff74` | `0x00001fbd` | |
| `0xf780` | `0xff80` | `0x00000000` | |
| `0xf788` | `0xff88` | `0x000010aa` | **BAR0 主控写 gadget（`reg_write_indirect`）** |
| `0xf78c` | `0xff8c` | `0x0000815a` | |
| `0xf790` | `0xff90` | `0x00008e18` | |
| `0xf794` | `0xff94` | `0xc0deca7e` | 保存金丝雀槽 |
| `0xf798` | `0xff98` | `0x0000815a` | |
| `0xf79c` | `0xff9c` | `0x00000000` | |
| `0xf7a0` | `0xffa0` | `0xc0deca7e` | 保存金丝雀槽 |
| `0xf7a4` | `0xffa4` | `0x00001fbd` | |
| `0xf7b0` | `0xffb0` | `0x0000ffbc` | |
| `0xf7b8` | `0xffb8` | `0x0000582d` | |
| `0xf7c4` | `0xffc4` | `0xc0deca7e` | 保存金丝雀槽 |
| `0xf7c8` | `0xffc8` | `0x00000cbd` | |
| `0xf7d8` | `0xffd8` | `0x00000003` | |
| `0xf7e0` | `0xffe0` | `0x00001fbd` | |
| `0xf7f4` | `0xfff4` | `0x00000ccb` | 见下方开放问题 |
| `0xf7f8` | `0xfff8` | `0x00007f2f` | 最外层槽 |

这张表在**正式 `master` 和全部十二个存档分支中逐字节相同**（经校验和与 grep `0xc0deca7eU` 验证，它在每个副本中恰好出现五次）。

**表中引用的承重 DMEM 地址：**

| DMEM | 含义 |
|---|---|
| `0x0100` 及以下 | 这里什么都没分配，杀掉了"在低 DMEM 分阶段布置 mega-ROP"的想法 |
| `0x0530` | DMA/引擎配置描述符 |
| `0x0600` | `WprMeta`，256 字节结构 |
| `0x06fc` | Booter 在 IMEM `0x27fa` 的 `r4 == 0` 分支上存 `0xa0a0a0a0` 的地方；与 `0x1fa824`/`0x1fa828` WPR2 寄存器**无关** |
| `0x0800` | DMA 签名缓冲区基址，即载荷偏移 0 |
| `0x103c` 起 | 密码学会话描述符 |
| `0x2383`、`0x8e08` | 寄存器描述符表，被载荷线性砸掉 |
| `0x6340` | **栈金丝雀全局**，十进制 25408 |
| `0x8700` | booter 代码/数据末尾 |
| `0xffec` | 喂给 `main` 退出状态、决定 `secure_teardown` 是否运行的槽 |

> [!NOTE]
> **开放问题：DMEM `0xfff4` 的 `0x00000ccb`**
>
> `0x0ccb` 是 `regtable_rw_indexed`，它索引的正是被载荷砸掉的 DMEM `0x2383` 和 `0x8e08` 描述符表，2026-07-06 的隔离矩阵显示每条携带写入的 rejoin 链都死在 `0xccb`。然而正式载荷在 `0xfff4` 植入 `0x00000ccb`，解锁可证明地工作。通过追踪展开期间 `0xfff4` 是否曾载入 PC，或它是否只是从不返回穿过的帧中一个存活通过的保存槽，可以定案。

> [!NOTE]
> **开放问题：无法解释的载荷常量**
>
> DMEM `0x1900` 的 `0x00000007`、`0xffd8` 的 `0x00000003`、`0x0000582d`、`0x0000ffbc`、`0x00008e18`、`0x0000815a`（两次）、`0x00000cbd`（两次）、`0x00001fbd`（三次）、`0x00007f2f`，以及填充 dword `0x000004a7` 本身。ROP 文章命名了相邻 gadget 家族（`0x1fb9`、`0x1fca`、`0x814e`、`0x8173`、`0x7f82`），因此这些很可能是同一个尾的翻译。对带注释反汇编过一遍应该能解决全部。

### 载荷覆盖钩子

`SEC2_POSTBL_TIMING_DMEM_PATH = "/lib/firmware/nvidia/ga100/gsp/dmem.bin"` 被读进新创建的 `0xf800` 缓冲区，缺失时回退到内置模板（预填 `writeAddr 0x009a0148`、`writeValue 0xffffffff`）。缺失以状态 `0x59` 报告，是良性的。

### 存档中的载荷变体

| 变体 | 大小 | DMA 基址 | 金丝雀值 | 金丝雀槽 |
|---|---|---|---|---|
| 正式 `master` 和全部 12 个分支 | `0xf800` = 63,488 B | `0x0800` | `0xc0deca7e` | `0x6340`、`0xff58`、`0xff94`、`0xffa0`、`0xffc4` |
| 净室 ROP 文章（2026-07-07/08/13） | `0xf800` | `0x0800` | `0xfaceb13d` | `0x6340`、`0xff58`、`0xff94`、`0xffdc`、`0xfff4` |
| 已被取代的 `builder.py` / `patcher.py` | `0xf700` = 63,232 B | `0x0900` | `0x2c20` 处 `0xdead2c20` | 生产镜像偏移，无一复用 |

金丝雀**值**任意，只要在防护全局和每个保存副本上统一；**地址 `0x6340` 是承重事实**。一条给出 `0x6440` 的一次性消息是口误：`0x5b40 + 0x900 = 0x6440` 是用更早记录 DMA 基址 `0x0900` 得到的。

---

## 不是 BAR0 地址的数字 {#numbers-that-are-not-bar0-addresses}

这里每一项都至少在存档中被误认过一次寄存器地址。

| 数字 | 实际是什么 |
|---|---|
| `0x02449000`、`0x02669000`、`0x02779000` | FBPA CFG1 **值**（原厂、40 GB 档、64/80 GB 档）。聊天中流传的半字节移位拼写 `0x24490000`、`0x26690000`、`0x27790000` 是抄录口误 |
| `0x00000208`、`0x0000020B`、`0x00000288`、`0x0000028A`、`0x0000028B` | MMU LMR **值** |
| `0x0000001000000000`、`0x0000000A00000000`、`0x0000001400000000`、`0x0000000200000000` | 64 GiB、40 GiB、80 GiB 和 8 GiB 字节计数（`targetFbBytes` / `fb_length` / `stockFbBytes`） |
| `0x88888888`、`0x00000008` | SS0 和 SS1 **值** |
| `0x0000f800`、`0x000004a7`、`0xc0deca7e`、`0xfaceb13d` | 载荷大小、填充 dword、金丝雀值 |
| `0x000010aa`、`0x000010b9`、`0x00001196`、`0x00001064`、`0x00008224`、`0x00008264`、`0x00008262`、`0x00007f82`、`0x0000814e`、`0x00008137`、`0x00008173`、`0x00008117`、`0x00008119`、`0x0000810d`、`0x00000ccb`、`0x00001fbd`、`0x00007f2f` | **Booter 内的 Falcon IMEM 地址**，不是 BAR0 偏移。`0x10aa` 是 `reg_write_indirect`；从 `0x10b9` 进入跳过 `r10`/`r11` 复制 |
| `0x0001c000`、`0x0001c100`、`0x0001c200`、`0x0001c300`、`0x00009100`、`0x00012000` | Falcon CSB 空间（`I[...]`），主机不可达 |
| `0x00200000` | `FUSE_PCIE_MAGIC_D` 的 A100/Drive 值，另外是写入 `XP3G_VAL3` 的值 |
| `0x00240036`、`0x00340036` | `PL_LINK_RATE` 值（发布的 Gen2 那个，以及什么都没训练的 Gen3 能力那个） |
| `0x1ffffe00` | WPR2_LO teardown **值** |
| `0x800000f1`、`0x800000f2` | BAR0 主控读和写**命令字** |
| `0x1312d00` | BAR0 主控看门狗种子，十进制 20,000,000 |
| `0x0000abcf`、`0x0004cb8f`、`0xffffff8f`、`0xfffffe8e`、`0xffffffcf`、`0xffffff88`、`0xfffff0ff` | PLM **内容** |
| `0x170000a1` | 识别 GA100 硅片的 `PMC_BOOT_0` 值 |

---

## 交叉参考：哪个补丁或工具碰哪个寄存器

| 补丁 / 工具 | 写入的寄存器 | 读取的寄存器 |
|---|---|---|
| `0001-sec2-postbl-plm-ss-cfg.patch`（master） | `0x001fa7cc`、`0x009a0148`、`0x001fa7c4`、`0x00823804`（PLM）；`0x001fa824`、`0x001fa828`（重新武装）；`0x0082381c`、`0x00823820`、`0x009a0204`、`0x00100ce0`（解锁） | 上述全部，加 GSP static-info `fb_length` 和最后一个 FB 区域的 `limit`、`reserved`、`supportCompressed`、`supportISO`、`performance`（= 20） |
| `0002-booter-verify.patch` | 无 | `0x00823804`、`0x0082381c`、`0x00823820`、`0x009a0204`、`0x00100ce0`（项目自己的规范五寄存器验证行） |
| `0003-late-pma.patch` | 无 | 用 `0x200000000`（8 GiB）作为原厂区域与延迟 PMA 扩展之间的拆分点，**对两个 SKU 都是**，包括真实原厂大小为 `0x280000000` 的 10 GB 卡 |
| `0004-bar0-pramin-clamp.patch` | 无 | 当 `0x20C2` 和 `0x2082` 上 `Ram.fbAddrSpaceSizeMb > 0x2000` 时把 PRAMIN 窗口钳制到 `(0x2000ULL << 20) - DRF_SIZE(NV_PRAMIN)` |
| `0005-ce-scrub-workarounds.patch` | 无 | 在两个设备 ID 上强制 `NV_MMU_PTE_KIND_GENERIC_MEMORY` 并禁用虚拟模式 CE 清扫 |
| `0006-persistent-sw-state.patch` | 无 | 为 `0x20C2` 和 `0x2082` 设置 `NV_FLAG_PERSISTENT_SW_STATE` |
| `0007-pcie-gen2.patch`（分支） | 23 条目 `xp3gTable`，加 `0x0008860c`、`0x0008841c`、`0x00088610`、`0x000880a8`、`0x0008c2c0`、`0x0008c040`、`0x0008c1c0`、`0x0008872c` | `0x00088084`、`0x000880a4`、`0x00088088`、`0x0082057c`、`0x00820580`、`0x00820520`、`0x0008e1b0` |
| `0008-pcie-gen2-probe-retrain.patch`（分支） | BAR0 `0x8c2c0`、`0x8c040`、`0x8872c`；GPU 和桥上的 PCIe 能力 `LNKCTL2`；仅桥上的 `LNKCTL` Retrain Link | GPU `LNKSTA`，以 100 ms 轮询 20 次 |
| `80` 分支补丁 `0001` | 与 master 相同，除了 10 GB 路径上 `cfg1Value = 0x02779000U` 和 `targetFbBytes = 0x0000001400000000ULL` | |
| Gen2 家族补丁 `0001` | master 的四道 PLM 加 `0x00088ff4`、`0x00088ab4`、`0x00088ff8`、`0x00823b00`、`0x008200fc` | |
| `probe.sh` / `ga100_topology_report.py` | 无 | `0x00000000`、`0x00820350`、`0x00820c1c`、`0x008205c4`、`0x00120078`、`0x00022430`、`0x00001404`、`0x00118f78`、`0x008204d8`、`0x008204bc`、逐 GPC `OPT_DISABLE` / `RECONFIG` / `CTRL_OPT` / `STATUS` / `RECONF_OVR`；`FBPA_BASE = 0x900000`、`FBPA_STRIDE = 0x4000` |
| `nuke.sh` | 每轮三写 ROP，所有目标 `0xffffffff` | 26 PLM 候选集 |
| `refire_chain_v2/v6/v9.py` | `0x001fa824`（teardown `0x1ffffe00`）、FB 几何 PLM 集、`0x009a0204`、一个变体中 `0x00820520` | `0x009a0204`、作为就绪门的 `0x008403c4` |
| `geo_flr_survival_map_20260716.sh`、`plm_flr_survival_20260716.sh`、`fire_vram_featovr_sweep.sh` | 打开 PLM，然后 FLR | 整个 26 寄存器 PLM 调查 |

---

## FLR 存续表

这是塑造整个项目的不对称性：计算比显存早数周发布，因为计算状态在常开岛、显存状态不在。

| 寄存器 | 经得起 FLR？ | 证据 |
|---|---|---|
| `0x0082381c`（SS0） | **是** | 10 GB 卡上前后实测 |
| `0x00823820`（SS1） | **是** | 同上 |
| `0x00823804`（`FEAT_OVR_PLM`） | **是** | 26 寄存器调查中唯一标为 AON 的 PLM |
| `0x00823b00`（行重映射器 PLM） | 可能 | 一次 HS 内扫描在 FLR 后读 `0xffffffff`，但打开它没让几何持久 |
| `0x009a0204`（CFG1） | **否** | `0x02779000` 恢复到 `0x02449000` |
| 逐 FBPA CFG1 | **否** | 同一扫描 |
| 逐 FBPA CSTATUS | **否** | 同上 |
| `0x00100ce0`（LMR） | **否** | `0x20b` 恢复到 `0x288` |
| FB 几何 PLM | **否**（它们重新锁定） | |
| `0x001180f0`（AON LMR 影子） | **否**（虽是 AON 仍恢复） | |
| `0x008403c4`（SEC2 复位 PLM） | **清除**到 `0xff` | FLR 移除 `0x8f` HS 污染 |

几何**确实**经得起无二次总线复位的卸载重载：卸载后 `0x009a0204` 仍读 `0x02669000`、`0x00100ce0` 仍读 `0x0000028a`，新加载再次枚举 40960 MiB。完整复位路径（PERST、`nvidia-smi --gpu-reset`、`echo 1 > /sys/bus/pci/devices/<bdf>/reset`）重跑带锁定 CMP 表的签名 DevInit，丢弃一切。

---

## 可能伴随这些寄存器出现的状态码

| 代码 | 哪里 | 含义 |
|---|---|---|
| `0xffff` | `kgspExecuteBooterLoad_TU102` 返回 | **每次**载荷运行都返回，无论成功与否。寄存器回读是唯一有效判定 |
| `0x31` | SEC2 `MAILBOX0` | 驱动植入的参数在跳过 `report_status` 的裸退出路径上未被触碰，**不是** Booter 错误码 |
| `0x15` | Booter 状态 | CSB 访问错误 |
| `0x29` | Booter 状态 | `check_1180f8_nibbles (0x1c75)` 门处 `0x001180f8` 位 [31:28] 非零 |
| `0x88` | Booter 状态 | `check_1180f8_2724 (0x1ba3)` 处 `0x001180f8` 位 [27:24] 非零 |
| `0x5` | Booter 状态 | `wpr_region_check (0x28ac)` 失败，包括空区域 |
| `0x47` | Booter 状态 | 栈检查失败 |
| `0x59` | 驱动日志 | `dmem.bin` 缺失，良性 |
| `0x62` | CPU-RM | `NV_ERR_RESET_REQUIRED`；`RmInitAdapter` 三元组 `(0x62:0x40:2028)` 是 WPR2-already-up 情况。作为 Booter `MAILBOX0` 码，`0x62` 属于 PKA 路径 |
| `0x65` | 驱动 | `NV_ERR_TIMEOUT`，阶段 3 交接轮询超时 |
| `0x96` | Booter 状态 | 正常 |
| `0x24` / `0x25` | CPU-RM | `kbusVerifyBar2` 失败 / 到达 StateLoad；CFG1 对比 LMR 的判别器 |
| `Xid 31`、`Xid 154`、`Xid 119` | 内核 | `80` 分支上约 40 GB 以上致命 GPU 丢失；80 GB 下多上下文失败；GSP RPC 超时 |

每个怎么处理见 [故障排查](../procedures/troubleshooting.md)。

---

## 相关页面

- [解锁的工作原理](how-it-works.md) 和 [概览](overview.md)
- [权限级掩码](privilege-level-masks.md)、[Falcon 与 Booter](falcon-and-booter.md)、[ROP 链](rop-chain.md)
- [显存几何](memory-geometry.md)、[计算节流](compute-throttle.md)、[PCIe Gen2](pcie-gen2.md)、[驱动补丁](driver-patches.md)
- [熔丝与 OTP](../hardware/fuses-and-otp.md)、[内存子系统](../hardware/memory-subsystem.md)、[PCIe 子系统](../hardware/pcie-subsystem.md)
- [词汇表](../start/glossary.md)，PLM、FLR、WPR、FBPA、LMR、AON、HS、PL0/PL3
- [寄存器索引](../appendix/register-index.md)，扁平字母序地址清单
