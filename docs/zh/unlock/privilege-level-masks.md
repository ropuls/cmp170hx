# 权限级掩码（Privilege Level Masks）

**本页涵盖：** GA100 上 PLM 是什么、掩码位如何编码、正式解锁打开的恰好四个掩码以及每个打开到的精确值、为什么四个中有一个故意只部分打开、哪些掩码在功能级复位后存续、哪些不存续，以及未发布 PCIe 分支改用九掩码表。执行打开的机制在 [ROP 链](rop-chain.md)；它运行的微码在 [SEC2 Falcon 与 Booter Load 微码](falcon-and-booter.md)。

**关键结论放在最前。** 解锁恰好打开四个掩码，按固定顺序，其中只有三个打开到全一：

| 顺序 | 名称 | BAR0 地址 | 写入的值 |
|:---:|---|---|---|
| 0 | `WPR_CFG` | `0x001fa7cc` | **`0xfffff0ff`** |
| 1 | `FBPA` | `0x009a0148` | `0xffffffff` |
| 2 | `WPR` | `0x001fa7c4` | `0xffffffff` |
| 3 | `FEAT` | `0x00823804` | `0xffffffff` |

> [!CAUTION]
> **`WPR_CFG` 打开到 `0xfffff0ff`，不是 `0xffffffff`**
>
> `0x001fa7cc` 的位 [11:8] 被故意保持清除。项目 README 和 `docs` 分支都把解锁描述为"把 PLM 打开到 `0xffffffff`"，或声称成功解锁后所有 PLM 必须读 `0xffffffff`。这种措辞对第一个条目不精确，会让正确解锁在肉眼验证者看来像失败。显示 `0x001fa7cc = 0xfffff0ff` 且其余三个为 `0xffffffff` 的卡是**正确的**。

---

## 1. PLM 是什么

主机、SEC2 Falcon 和 GSP NVRISC-V 核心都通过 **PRI**（共享特权寄存器接口）访问 GPU 内部寄存器。主机通往 PRI 的窗口是 BAR0。PRI 不是平坦开放的：它的区域被 **权限级掩码（PLM）** 门控，每个受保护区域一个小子寄存器，声明哪些权限级可以读该区域、哪些可以写，以及哪些片上来源被允许发起这些访问。

Falcon 家族权限级运行 L0 到 L3，映射到 [falcon-and-booter.md](falcon-and-booter.md#2-the-falcon-security-model) 描述的 Falcon 执行模式：

| 级别 | 谁 | 典型范围 |
|---|---|---|
| L0 | 经 BAR0 的主机，以及非安全 Falcon 代码 | 普通寄存器 |
| L1 / L2 | 轻安全 Falcon 上下文 | 中间 |
| L3 | 重型安全 Falcon 代码 | 一切，包括 PLM 本身 |

整个解锁的存在是因为改变卡显存几何和计算节流的寄存器被只允许 L3 写入的掩码门控，而在这颗裸片上到达 L3 的唯一方式是运行在签名重型安全微码内部。一旦 PLM 从 HS 内部被重写为允许 L0 写入，普通主机 BAR0 写入无需更多漏洞利用就能工作。这个枢轴就是正式解锁的整个架构。

> [!WARNING]
> **PLM 意思是 Privilege Level Mask**
>
> 项目自己的 `docs` 分支（`cmpunlocker-branches/docs/docs/ARCHITECTURE.md`）把 PLM 展开为"Program Logic Modules"，还发明了"PMM（Permute Mask Model）"、"LMR（LM Request）"、"SS0/SS1（Suspension State）"和"PMA（Power Management Array）"。这些都是杜撰。NVIDIA 头文件中底层寄存器名是 `PRIV_LEVEL_MASK`。该文档的任何内容都不应带进对这块硬件的描述。见 [死路](../history/dead-ends.md)。

---

## 2. 位编码

本项目全程使用的编码，与裸片上每个观测值一致：

| 字段 | 位 | 含义 |
|---|---|---|
| `READ_PROTECTION` | [3:0] | 哪些级别可以读 |
| `WRITE_PROTECTION` | [7:4] | 哪些级别可以写 |
| `SOURCE_ENABLE` | [23:8] | 哪些片上来源可以发起访问 |

常见值：

| 值 | 读法 |
|---|---|
| `0xFFFFFF8F` | 所有级别可读，写仅 L3，所有来源使能。**这颗裸片上的锁定基线。** |
| `0xFFFFFFFF` | 完全开放：所有级别读写，所有来源。 |
| `0xFFFFFFCF` | 另一个观察到的写锁定模式。 |
| `0x0004CB8F` | 解锁后原厂驱动加载时 `WPR` 和 `WPR_CFG` 被重新锁定的值。 |
| `0xFFFFFE8E` | 原厂驱动加载后在 `0x009A0008`、`0x00100B10` 和 `0x00100B38` 上观察到。 |
| `0x0000008F` / `0x000000DF` | 在 SEC2 复位 PLM 上看到的仅低字节模式。 |

对精确三字段分解的置信度作为工作模型是高的：它与每个观察到的基线、每个观察到的解锁后值以及正式解锁器选择 `0xFFFFFFFF` 都一致。它是从行为推断的，不是从已发布头文件读的。

编码的两个实际后果：

- **`0x000000FF` 不是"开放"。** 它只设置 READ 和 WRITE 半字节，留下 `SOURCE_ENABLE = 0`，会挡住一切。ROP v2 正是因此把 `0x000000FF` 写到 `0x00823804`，没有产生任何有用效果；之后的每个载荷和正式补丁都写 `0xFFFFFFFF`。
- **PLM 拒绝越界值。** 即使其他掩码打开，向 PLM 写任意值（如 `0xff`）也会被弹回。因此暴力尝试 PLM 值受硬件接受的编码限制，而不是整个 32 位空间。至少两个人、两张不同卡直接观察到。

---

## 3. 解锁打开的四个掩码

### 3.1 表，与源码中完全一致

```c
static const struct {
    NvU32 addr;
    NvU32 value;
    const char *name;
} plmTable[] = {
    { 0x001fa7ccU, 0xfffff0ffU, "WPR_CFG" },
    { 0x009a0148U, 0xffffffffU, "FBPA"    },
    { 0x001fa7c4U, 0xffffffffU, "WPR"     },
    { 0x00823804U, 0xffffffffU, "FEAT"    },
};
```

这张表在正式 `master` 和每个存档分支中逐字节相同，除了扩展它的四个 Gen2 家族分支（第 6 节）。

### 3.2 每个门控什么

| 名称 | 地址 | 门控 | 解锁为什么需要它 |
|---|---|---|---|
| `WPR_CFG` | `0x001fa7cc` | WPR 掩码/配置寄存器 `0x001fa814` / `0x001fa818`，以及 WPR 区域块整体 | 写保护区域机制必须在重复 Booter 触发之间可重新武装 |
| `FBPA` | `0x009a0148` | FBPA（帧缓冲分区）寄存器孔径，包括 `0x009a0204` 的广播 CFG1 | 触发后主机 PL0 必须能写 CFG1。见 [显存几何](memory-geometry.md)。 |
| `WPR` | `0x001fa7c4` | WPR1/WPR2 地址寄存器 `0x001fa81c`-`0x001fa828` | 每次触发前主机必须重新武装 WPR2 低/高位 |
| `FEAT` | `0x00823804` | 功能覆盖块 `0x00823800`-`0x00823FFC`，包括 SS0 `0x0082381c` 和 SS1 `0x00823820` | 主机 PL0 必须能写计算节流覆盖。见 [计算节流](compute-throttle.md)。 |

`FEAT` 对持久性最有意思，因为它位于常开岛。见第 5 节。

### 3.3 原厂值

| 寄存器 | 原厂读数 | 备注 |
|---|---|---|
| `0x00823804` `FEAT_OVR_PLM` | `0xffffff8f` | 在探测过的**每个** Ampere 卡上读数相同：两台 170HX、A100 SXM4 40G、A100 PCIe 40G、A100 PCIe 80G、A10、A5000、A6000、RTX 3080 / 3080 Ti / 3090 / 3090 Ti 和 Drive A100 |
| `0x00823800` `FEAT_OVR_ECC_PLM` | `0xffffff8f` | A100 SXM4 40G 单独读出 `0x0000abcf`，原因不明 |
| `0x00823B00`（行重映射器 PLM） | `0xFFFFFF8F` | |
| `0x009a0148` `FBPA` | `0xFFFFFF8F` | |
| `0x001fa7c4` / `0x001fa7cc` | 锁定 | 原厂驱动加载后重新锁定到 `0x0004CB8F` |

整个 `0x823800`-`0x823FFC` 窗口中只有三个寄存器在锁定卡上读出 `0xFFFFFF8F`：`0x823800`、`0x823804` 和 `0x823B00`。这个共享值正是把它们识别为守护该块的 PLM 的依据。不过它们不是仅有的可读 dword：2026-07-16 对锁定卡整个窗口的范围扫描在 PL0 返回十二个活 dword，即连续运行 `0x823800`-`0x82382C` 加 `0x823B00`，而直到 `0x823FFC` 的每个其他 dword 都是 `0xBADF5040`。

---

## 4. 如何执行打开

四个条目的机制相同，值得细读，因为成功标准不是日志行暗示的那样。

```c
savedWpr2Lo = GPU_REG_RD32(pGpu, 0x001fa824);
savedWpr2Hi = GPU_REG_RD32(pGpu, 0x001fa828);
/* SEC2_DEBUG: saved WPR2 lo=0x%08x hi=0x%08x */

for (plmIdx = 0; plmIdx < 4; plmIdx++) {
    opened = NV_FALSE;
    for (attempt = 0; attempt < 2 && !opened; attempt++) {
        GPU_REG_WR32(pGpu, 0x001fa824, savedWpr2Lo);
        GPU_REG_WR32(pGpu, 0x001fa828, savedWpr2Hi);
        kgspSec2PostblTimingRefillPayload(pGpu, pKernelGsp,
                                          plmTable[plmIdx].addr,
                                          plmTable[plmIdx].value);
        kgspExecuteBooterLoad_HAL(pGpu, pKernelGsp,
            memdescGetPhysAddr(pKernelGsp->pWprMetaDescriptor, AT_GPU, 0));
        regVal = GPU_REG_RD32(pGpu, plmTable[plmIdx].addr);
        opened = (regVal == plmTable[plmIdx].value);   /* exact equality */
    }
    /* SEC2_DEBUG: FAILED to open <name> after 2 attempts */
}
GPU_REG_WR32(pGpu, 0x001fa824, savedWpr2Lo);
GPU_REG_WR32(pGpu, 0x001fa828, savedWpr2Hi);
```

要点：

- **每次尝试一次 Booter Load 触发，每次触发一次任意 BAR0 写入。** 载荷只携带一对 `(writeAddr, writeValue)`，因此打开四个掩码花四到八次触发。
- **每次尝试前都重新武装 WPR2 低/高位**，最多八次，循环后再一次，因为每次触发都重新划分 WPR2，否则后续 Booter Load 会以"WPR2 already up"中止。正式驱动恢复*保存的*对；它不写无驱动工具写的那组常量 `0x1FFFFE00` / `0`。
- **成功是精确回读相等**，不是 Booter 状态。每次触发无论结果如何都报告 `status=0xffff` 并记录 `Booter failed with non-zero error code: 0x31`。寄存器回读是唯一有效判定。
- **一个条目失败不会中止循环。** 它记录并继续。

正式 `master` 每次尝试记一行，循环后记一行汇总：

```text
SEC2_DEBUG: PLM[%u] %s(0x%x) attempt=%u status=0x%x reg=0x%08x
SEC2_DEBUG: PLMs: FEAT=0x%08x FBPA=0x%08x WPR=0x%08x WPR_CFG=0x%08x
```

真实行读作 `SEC2_DEBUG: PLM[3] FEAT(0x823804) attempt=0 status=0xffff reg=0xffffffff`，索引到名称映射为 `PLM[0]=WPR_CFG, PLM[1]=FBPA, PLM[2]=WPR, PLM[3]=FEAT`。未发布分支用相同的逐条目格式。

fork 线程中流传的较短 `SEC2_DEBUG: PLM FEAT before:` / `PLM FEAT after:` 对**不是**正式字符串：grep 正式仓库找不到这样的文本。它引用的值（锁定 `0xFFFFFF8F`、打开 `0xFFFFFFFF`）对 `FEAT` 是对的，但不要把这一行本身当作 `dmesg` 里要找的东西。

如果掩码事后仍读锁定值，说明打开没有发生，记录在案的补救是再冷启动一次。见 [故障排查](../procedures/troubleshooting.md)。

### 4.1 循环后发生什么

四个掩码打开后，驱动在 PL0 做四个普通主机 `GPU_REG_WR32()` 调用，不再需要漏洞利用：

| 寄存器 | 值 | 目的 |
|---|---|---|
| `0x0082381c`（SS0） | `0x88888888` | 计算节流 |
| `0x00823820`（SS1） | `0x00000008` | 计算节流 |
| `0x009a0204`（CFG1） | `0x02779000`（8 GB 卡）/ `0x02669000`（10 GB 卡） | 显存几何 |
| `0x00100ce0`（LMR） | `0x0000020B`（8 GB 卡）/ `0x0000028A`（10 GB 卡） | 显存几何 |

> [!WARNING]
> **`docs` 分支对 SS0 和 SS1 是错的**
>
> `ARCHITECTURE.md` 声称 `SEC2_DEBUG: SS0 = 0xffffffff` 和 `SS1 = 0xffffffff`，并打印一行代码中任何地方都不存在的预期日志 `SEC2_DEBUG: Executing unlock sequence...`。正式代码写 SS0 = `0x88888888`、SS1 = `0x00000008`。同一文档中的几何表（8 GB 到 64 GB 用 `0x02779000`/`0x0000020B`，10 GB 到 40 GB 用 `0x02669000`/`0x0000028A`）倒是正确的。

---

## 5. 持久性：哪些掩码在复位后存续

这种不对称性是 PLM 集合中影响最大的属性，也是计算解锁先于显存解锁发布的原因。

| 寄存器 | FLR 后存续？ | 备注 |
|---|---|---|
| `FEAT` `0x00823804` | **是** | 常开（AON）电源岛 |
| SS0 `0x0082381c`、SS1 `0x00823820` | **是** | AON |
| `WPR` `0x001fa7c4`、`WPR_CFG` `0x001fa7cc` | 否 | 原厂驱动加载后重新锁定到 `0x0004CB8F` |
| `FBPA` `0x009a0148` | 否 | 重新读出 `0xFFFFFF8F` |
| CFG1 `0x009a0204`、逐 FBPA CFG1、`CSTATUS`、LMR `0x00100ce0` | 否 | 几何**不**存续 |
| AON LMR 影子 `0x001180f0` | 否 | |
| SEC2 复位 PLM `0x008403C4` | 污染被清除：`0x8f` 回到 `0xff` | FLR 是唯一清除它的东西 |

解锁后加载原厂驱动，只有 `0x00823804` 的 `FEAT` 保持解锁。580.159.04 上无驱动加载、两次 FLR 后的一次更大扫描报告 `0x8200D4`、`0x8200D8`、`0x8200E0`、`0x8200E4`、`0x8200E8`、`0x8200EC`、`0x8200F0`、`0x8200F4`、`0x8200FC`、`0x823800`、`0x823804` 和 `0x823B00` 为 UNLOCKED，而 `0x8200D0` 和 `0x8200DC` 读 `0xFFFFFF8F`、`0x9A0008` / `0x9A000C` / `0x9A0148` / `0x9A014C` / `0x9A03F0` 读 `0xFFFFFF8F`、`0x9A0168` / `0x9A0554` / `0x100B9C` 读 `0xFFFFFFCF`、`0x9A0BFC` 读 `0x00000000`、`0x100B10` / `0x100B38` 读 `0xFFFFFF8F`、`0x100B84` 读 `0xFFFFFF88`。

从那两次扫描得出的区分：持久 PLM 位于常开电源岛上；复位 PLM 每次启动都需要重新解锁。

> [!NOTE]
> **开放问题：系统性 AON 分类**
>
> 一个名为 `nuke.sh` 的实验完整规定了工作：每轮构建一个三写 ROP 载荷、补 GSP、加载驱动、FLR、杀驱动、再 FLR，在无驱动加载下读 26 个候选 PLM，先做冷启动基线，跑九轮。候选集：`0x008200D0, D4, D8, DC, E0, E4, E8, EC, F0, F4, FC`；`0x00823800`、`0x00823804`、`0x00823B00`；`0x009A0008, 000C, 0148, 014C, 0168, 03F0, 0554, 0BFC`；`0x00100B10, B38, B84, B9C`。方法和候选清单有记录；结果分类表没有。

因为几何经不起 FLR，"攻击、FLR、重载干净驱动"这招对计算解锁有效，却无法承载显存解锁：清除 GSP-RM 损伤的 FLR 同时清除 LMR 写入。这个约束迫使采用驱动内、同次加载的设计。

---

## 6. 未发布分支上的九掩码表

> [!WARNING]
> **实验性**
>
> 四个未发布分支 `Gen2`、`debug-gen2`、`far` 和 `deced` 把表从四个条目扩展到**九个**，循环 `plmIdx < 9`。这段代码不在 `master` 上、没有正式消费者，而且最关键条目的回读结果在源码中没有记录。见 [PCIe Gen2](pcie-gen2.md)。

| 顺序 | 名称 | 地址 | 值 | 状态 |
|:---:|---|---|---|---|
| 0 | `WPR_CFG` | `0x001fa7cc` | `0xfffff0ff` | 同正式 |
| 1 | `FBPA` | `0x009a0148` | `0xffffffff` | 同正式 |
| 2 | `WPR` | `0x001fa7c4` | `0xffffffff` | 同正式 |
| 3 | `FEAT` | `0x00823804` | `0xffffffff` | 同正式 |
| 4 | `XVE` | `0x00088ff4` | `0xffffffff` | 新增 |
| 5 | `XVE_B` | `0x00088ab4` | `0xffffffff` | 新增 |
| 6 | `XVE_C` | `0x00088ff8` | `0xffffffff` | 新增 |
| 7 | `FEAT2` | `0x00823b00` | `0xffffffff` | 新增；也是行重映射器 PLM |
| 8 | `OPT_PLM` | `0x008200fc` | `0xffffffff` | 新增 |

三个 XVE 掩码是必需的，因为 PCIe 影子寄存器受 PLM 保护、禁止主机读取：主机读取返回 `0xbadf5040`。

最终无驱动工具用同一个九条目清单，报告一次启动中两张 GPU 上全部九个首次尝试即成功，记录为 `PLM[n] NAME(addr) attempt=0 status=0xffff reg=0xffffffff`，前四个与正式表逐字节交叉验证，包括部分值 `0xfffff0ff`。

那个结果与另一条具体观察存在张力：

> [!NOTE]
> **开放问题：`FEAT2` `0x00823b00` 真的能打开吗？**
>
> 一位研究者在 2026-07-22 报告 `0x00823b00` **拒绝** SEC2 链，因为它的 `SOURCE_ENABLE` 字段没有白名单 sec2-HS，而且 SEC2 ROP 只能打开 `SOURCE_ENABLE` 允许的掩码。同期另一份陈述："还有其他允许 PLM 打开的原语。并非所有 L3 访问都平等。经 bar0 或经 sec2 `iowrs` 的 Regops 不同。"分支代码记录逐条目回读，因此一张卡上跑一次启动就能定案。

### 6.1 `0x008200FC`：两个名字、三种读数、无定论

> [!NOTE]
> **开放问题**
>
> `0x008200FC` 处的寄存器在分支源码中叫 `OPT_PLM`、在净室工具中叫 `FUSE_SS_PLM`。**它们是同一个寄存器**，wiki 在一个条目上携带两个别名。它读出什么、是否可写，尚未定案：
>
> | 日期 | 报告 |
> |---|---|
> | 2026-07-09 | "PLM = `0x000003FF`（目标 `0xFFFFFFFF`）……FUSE 写入失败，寄存器看起来物理只读。主机直接 `writel` 也被封顶在 `0x3FF`。" |
> | 2026-07-16 | "整个 Ampere 产品线读出 `0xffffffff`（所有级别开放）" |
> | 2026-07-23/24 | 九 PLM 工具包含它，报告 `PLM[8] OPT_PLM(0x8200fc) attempt=0 status=0xffff reg=0xffffffff`，即成功 |
>
> 可能的解释：卡状态差异、命名混淆，或该寄存器只在其他掩码打开后才可写。通过在同一台机器上冷启动、任何解锁前读 `0x008200FC`，然后九个打开每个之后各读一次来定案。
>
> 早期 ROP 链（v2 和 v3）写 `0x008200FC = 0xFFFFFFFF`，写入失败。它被正确诊断为不必要：可工作的计算解锁只用 `FEAT_OVR_PLM` `0x00823804` 加 SS0/SS1。**正式 `master` 不写它。**

---

## 7. 有多少个 PLM

远超最初的估计。计数走了 1、然后 3、然后"5 个重要，大概 10 到 15"，触发日志最终在不同集合中枚举出 9、10 和 27 个不同的掩码。最早的估计来自一份公开 GA100 熔丝和寄存器 gist，写于"我们甚至不知道 PLM 是什么"之前。

一次只读调查在 170HX 上编目了 **26 个不同的 PLM 寄存器**，并确立了第 2 节的位编码。另一次独立枚举在一张卡上统计了社区驱动启动时链打开的 **27 个掩码**：`FEAT`、`FBPA`、`WPR`、`WPR_CFG`、六个 `XVE` 掩码、`FUSE_FAM_A`、十个 `FUSE_PLM` 掩码和六个 `XP_PL` 掩码。置信度：中等，一手寄存器级报告，第二位研究者佐证。

每个 PLM 位于它所保护的寄存器附近。打开一个允许所有权限级 L0 到 L3 写入。

> [!NOTE]
> **开放问题：寄存器的 PLM 地址能否从寄存器自己的地址推导？**
>
> 直接问过，从未回答。正式集合（`FEAT 0x00823804`、`FBPA 0x009a0148`、`WPR 0x001fa7c4`、`WPR_CFG 0x001fa7cc`）相对它所守护的寄存器不遵循明显偏移规则，尽管"都放在寄存器附近"的观察成立。下一步：对一块孔径做全 PLM 扫描，看掩码是否占据每个块的固定子范围。

> [!NOTE]
> **开放问题：LMR 的 PLM 在哪里？**
>
> 没有人定位到门控 LMR `0x00100CE0` 的掩码。有人贴出了一张候选 FBHUB 表（`0x100B10 = 0xFFFFFF8F`、`0x100B38 = 0xFFFFFF8F`、`0x100B84 = 0xFFFFFF88`、`0x100B9C = 0xFFFFFFCF`），但贴主否认了它，后来报告找不到。正式驱动仍然在打开 `WPR_CFG`、`FBPA`、`WPR` 和 `FEAT` 后成功从主机写 LMR，因此**这四个之一已经门控它**。对正式表做四路消融可以识别出哪一个。

> [!NOTE]
> **开放问题：显存解锁实际需要多少个 FBPA 侧掩码**
>
> 一个立场认为需要四个 FBPA 掩码（`0x9A0148`、`0x9A014C`、`0x9A0008`、`0x9A000C`）加 LMR 加复位 PLM，`0x100b10` 被证明不必要。第二个立场被有力主张：CFG1 和 LMR 单独就够，FBPA 掩码由 CFG1 广播自动设置。正式代码部分定案：它恰好打开**一个** FBPA 掩码 `0x009a0148`，然后从主机写 CFG1 和 LMR，因此"四个"和"零个"都不是正式答案。它与第三个数据点冲突：无驱动 `geometry_chain()` 打开**五个** FB 几何掩码，包括 `0x100b10`。通过在同一张卡上逐条目消融两个清单来定案。
>
> 一个相关测量缩小了问题：**对 `0x009A0204` CFG1 的一次重型安全广播写入传播到全部 20 个逐 FBPA `CSTATUS` 寄存器**（每个活动 FBPA 上 `0x200` 到 `0x800`），因此 HS 完全绕过 FBPA 掩码。打开它们从来只为了 `0x00900204 + n*0x4000` 处的主机 PL0 逐 FBPA 写入。

---

## 8. SEC2 复位 PLM `0x008403C4`

这在上面的意义上是一个 PLM，但不是解锁打开的。它门控 SEC2 Falcon 自己的复位控制（SEC2 + `0x3c0` 的 `FALCON_ENGINE`），也是无驱动工具拥有整套"clean SEC2"纪律的原因。

| 值 | 状态 |
|---|---|
| `0xff` | 完全开放。干净空闲、SBR 后。主机 PL0 复位会生效。 |
| `0xdf` | 原厂驱动 GSP 启动 teardown 后的正常工作状态。仍允许复位。 |
| `0xcf` | 驱动的 GSP-prime 重新锁定 PLM 后（位 4 清除）。 |
| `0x8f` | 重型安全退出污染。写锁定到安全源；PL0 复位写入被弹回。 |

`reset_allowed = resetPLM in {0xff, 0xdf}`，`0xdf = 0x8f | 0x50`。`0x8f` 在 HS 到 NS 退出转换时由硬件闩锁，不是任何 booter 指令写入的。一旦它读出 `0x8f`，主机 PL0 无法向它写 `0xff`、`0xdf` 或 `0xffffffff`；只有 HS 写入能降低它，只有 FLR 或 SBR 能把它清回 `0xff`。在该状态下 `0x8f` 以错误对 `0x62:0x55` 阻止 GSP-RM 启动，并阻止 PL0 发出的 SEC2 `SFTRESET`。

主机侧流程检查的引擎复位门是 `(value & 0x77) == 0x77`。

> [!NOTE]
> **正式驱动从不碰它**
>
> grep 正式仓库找不到对 `0x008403c4` 的任何引用。整套 clean-SEC2 纪律属于无驱动工具。驱动内路径改经驱动自己的 `kflcnReset`/FWSEC 序列重新触发 Booter Load，因此它从不需要主机发出的 SFTRESET。对该解释的置信度：中等，是推断，因为没人陈述过。通过在正式驱动下 4 到 8 轮 PLM 每次前后读 `0x008403C4` 来定案。

完整细节，包括正式载荷无论如何让它保持 `0xff` 的 `D[0x1900] = 7` 机制，在 [falcon-and-booter.md](falcon-and-booter.md#10-leaving-heavy-secure-mode-and-the-reset-plm) 和 [rop-chain.md](rop-chain.md#73-why-the-exit-is-clean)。

---

## 9. 为什么掩码是唯一杠杆

值得说明 PLM 方案取代了什么，因为熔丝证据干净地关闭了替代方案。在 15 张 Ampere 卡上实测：

| 熔丝 | 地址 | 读数 | 后果 |
|---|---|---|---|
| `FUSE_QUADRO_WR_SEC` | `0x0082038C` | 全部 15 张 `0x00000001` | 门控 `0x823804` 功能覆盖 PLM 的自密封熔丝**处处已熔断** |
| `FUSE_FEAT_OVR_DIS` | `0x008203F0` | 全部 15 张 `0x00000000` | 会永久锁定所有功能覆盖的主熔断开关**未熔断**。这就是一切能工作的原因。 |
| `FUSE_EN_SW_OVERRIDE` | `0x00820040` | 170HX、全部三个 A100 SKU 和 Drive A100 上 `0x00000000`；每个消费级和 ES 部件上 `0x00000001` | CTRL_OPT 软件熔丝覆盖路径在熔丝级被禁用，因此未签名 FwSec 尾中的 25 条目 CTRL_OPT 表在这些卡上是惰性的 |
| `FUSE_OPT_SECURE_GSP` | `0x0082074C` | 全部 15 张 `0x00000001` | GSP 调试被禁用，GSP 只接受签名生产固件，这就是解锁必须走签名缓冲区路线的原因 |
| `FUSE_DIS_SW_OVR` | `0x00820084` | 全部 15 张 `0x00000001` | 非 HS 可写：2026-07-27 在 8 GB 卡上用两个活控件探测，值被弹回。仍然开放的问题更窄：考虑到 `DIS_SW_OVR = 1` 在覆盖可用的消费级卡上也读 `1`，它是否真的*锁定* `FUSE_EN_SW_OVERRIDE` |
| `FUSECTRL` | `0x00820000` | 全部 15 张 `0xe0040000` | |
| `FEATURE_OVERRIDE_QUADRO` | `0x00823808` | 逐裸片且无法解释：`0x00100183`（原厂 PLM 范围扫描）、`0x00000081`（**解锁后**探测，非原厂读数）、`0x00000181` / `0x00000182`（两台物理 170HX）、`0x01000282`（A100 80 GB） | 只读。为什么值在转储间不同是开放问题；见 [寄存器参考](register-reference.md) |

2026-05-31 关于仅主机寄存器写入"CONFIRMED DEAD"的结论作为诊断完全仍然有效：`FEAT_OVR_PLM` 和 `FEAT_OVR_ECC_PLM` 都在 `0xffffff8f`（仅第 3 级）、`FUSE_QUADRO_WR_SEC = 1` 密封该掩码、`FUSE_EN_SW_OVERRIDE = 0` 禁用 CTRL_OPT 覆盖表，且 `FECS_FEAT_OVERRIDE` 读取返回 `0xbadf5040`。那份文档缺少的是到达第 3 级的方法，而且它正确预测了答案会是"需要 Falcon HS"。

完整熔丝图见 [熔丝与 OTP](../hardware/fuses-and-otp.md)。

> [!NOTE]
> **值得保留的负面结果**
>
> 从非安全 Hello-World ucode 经 priv 总线运行 LMR 和 CFG1 写入被尝试过，不工作，与 NVIDIA 自己的 Falcon 安全文档预测完全一致：NS 限制寄存器和物理内存访问，这些寄存器上的掩码要求最高级别。后来一次看起来确认 NS 完全够不到外部 BAR0 的无驱动探测被**其作者自己撤回**，因为失败的测试用了与 Falcon 本地 DMEM 别名的 `D[0x14000000]` 窗口，因此从未探测 BAR0。没有报告替代测量。

> [!NOTE]
> **开放问题：掩码打开后 PL0 能到达几何寄存器吗？**
>
> 赌注很大。因为 `0x00823804` 的 SS0/SS1 掩码是常开的、经 FLR 保持打开，如果主机 PL0 在一次一次性 HS 打开后能到达 `0x00900204 + n*0x4000` 的 CFG1、LMR `0x00100CE0` 和 SS0/SS1，那么存在一条无需更多重型安全工作的永久路径。下一步：用正确映射的外部孔径窗口重跑被撤回的探测，逐个检查每个寄存器。

---

## 10. 手工验证一张卡

解锁后要回读的四个值，以及正确卡显示的内容：

```text
0x001fa7cc  ->  0xfffff0ff     WPR_CFG   (NOT 0xffffffff)
0x009a0148  ->  0xffffffff     FBPA
0x001fa7c4  ->  0xffffffff     WPR
0x00823804  ->  0xffffffff     FEAT
```

以及这些掩码启用的四次解锁写入：

```text
0x0082381c  ->  0x88888888     SS0
0x00823820  ->  0x00000008     SS1
0x009a0204  ->  0x02779000  (8 GB card)  /  0x02669000  (10 GB card)
0x00100ce0  ->  0x0000020b  (8 GB card)  /  0x0000028a  (10 GB card)
```

注意后两个经不起 FLR 或断电，只有 `0x00823804`、`0x0082381c` 和 `0x00823820` 能存续。见 [验证](../procedures/verify.md)。

---

## 相关页面

- [SEC2 Falcon 与 Booter Load 微码](falcon-and-booter.md)
- [ROP 链](rop-chain.md)
- [显存几何](memory-geometry.md) 和 [计算节流](compute-throttle.md)
- [驱动补丁](driver-patches.md)
- [PCIe Gen2](pcie-gen2.md)
- [熔丝与 OTP](../hardware/fuses-and-otp.md)
- [寄存器参考](register-reference.md) 和 [寄存器索引](../appendix/register-index.md)
- [词汇表](../start/glossary.md)
