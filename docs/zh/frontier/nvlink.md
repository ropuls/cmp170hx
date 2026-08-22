# NVLink：熔丝关闭，没有找到操作手段

**本页涵盖：** CMP 170HX 上 NVLink 的完整状态：证明它是在一次性可编程硅片中而非软件中被禁用的熔丝读数、所有被探测过的内容、每条被提出过的覆盖路径以及它们各自为什么关闭、物理连接器和桥接器状况，以及真正能推进这个问题的少量实验清单。

**头条结论：CMP 170HX 上的 NVLink 不工作，对语料中的任何人都从未工作过，而且从未在寄存器层面尝试过解锁。** 它被 OTP 熔丝禁用，而不是被软件禁用。cmpunlocker（正式发布的 `master` 或 12 个未发布分支快照中的任何一个）里没有一行代码触碰 NVLink。NVLink 在分支集中的全部存在形式只是两个 README 功能表里的一个词 `Planned`。

> [!NOTE]
> **开放问题**
>
> 这是该领域价值最高的未知数，而且桌面上没有任何可着手之处。两条覆盖路径都已关闭：`CTRL_OPT` 被 `FUSE_EN_SW_OVERRIDE` = `0x0` 关闭，`FEAT_OVR` 路线因该块中没有任何 NVLink 寄存器而关闭。正如 2026-07-20 的一份总结所说："Still unsolved rn, a bit harder as there's no fuse mask."（还没解决，因为没有熔丝掩码所以更难。）

**这项能力会是什么。** NVLink 的行为像 MMIO：链路远端的内存可以被映射进本地 GPU 的地址空间，并由 CUDA 内核或复制引擎驱动。来源：exploit 原作者，置信度中。从未在 170HX 上演示过，因为从来没有链路起来过。这种直接可寻址性正是跨桥接卡的内存池有任何价值的原因。

---

## 硅片说了什么

三个读数界定了局面，它们在两个 SKU 上至少五次独立回读之间相互一致，另有 15 卡 Ampere 参考群组。

| 寄存器 | 地址 | 170HX 值 | 含义 |
|---|---|---|---|
| `FUSE_NVLINK_DIS`（`OPT_NVLINK_DISABLE`） | `0x00820684` | `0x00000007` | `[2:0]` 禁用字段的三个位全部置位 |
| `STATUS_OPT_NVLINK`（只读镜像） | `0x00820DB8` | `0x00000007` | 芯片其余部分看到的有效状态 |
| `PTOP_SCAL_NUM_NVLINK` | `0x0002246C` | `0x0000000c` | die 扩展到 12 条链路，与每个 A100 完全一样 |

然后是说明硅片健康的三个：

| 寄存器 | 地址 | 170HX 值 | 含义 |
|---|---|---|---|
| `FUSE_NVLINK_DEFECTIVE` | `0x0082068C` | `0x00000000` | 不是良率修复。一项调查报告其 15 卡群组中每张返回了值的卡都是 `0`；A16 列读出的是 `BAR0` 占位符，ES 列为空 |
| `FUSE_NVLINK_DIS_CP`（禁用关键路径） | `0x00820688` | `0x00000000` | 关键路径层面未禁用 |
| `FUSE_NVLIPT_RST_DIS` | `0x00821100` | `0x00000000` | NVLink IP 复位条件未被禁用 |

还有覆盖四个覆盖机制相关寄存器的，其中后两个显示它是关闭的：

| 寄存器 | 地址 | 170HX 值 | 含义 |
|---|---|---|---|
| `CTRL_OPT_NVLINK`（有效，每链路位 15:0） | `0x008209B8` | `0x00000000` | 未设置 CTRL 覆盖；禁用通过 STATUS 路径到达 |
| `CTRL_OPT_PERLINK`（位 11:0） | `0x00820820` | `0x00000000` | 同上 |
| `FUSE_EN_SW_OVERRIDE` | `0x00820040` | `0x00000000` | 整个 `CTRL_OPT` 覆盖机制被熔丝禁用 |
| `FUSE_DIS_SW_OVR` | `0x00820084` | `0x00000001` | 从另一方向确认了上述内容 |

语料得出的结论：**这是刻意的产品划分，而不是报废捡料（salvage binning）。** 链路被物理构建，die 报告 12 条，没有一条被标记为缺陷，而禁用来自熔丝。

### 它不是挖矿 SKU 的限制

Drive A100 32 GB（PG199，`GA100-550F-A1`，`FUSE_PCIE_DEVIDA` = `0x000020bb`，`FUSE_PCIE_DEVIDB` = `0x000020fb`）读出完全相同的 `FUSE_NVLINK_DIS` = `0x00000007` 和 `STATUS_OPT_NVLINK` = `0x00000007`，在两块物理 PG199 板上测得。所有三个常规 A100 SKU 读出 `0x00000000`。任何把 NVLink 熔丝当作加密货币挖矿特定惩罚的理论都必须解释 Drive 部件。

### 写入安全按架构分裂，而不是按 SKU

`OPT_SECURE_NVLINK_MASK_WR_SECURE` 在 `0x00820704` 处，在**每一个** GA100 部件上（两块 170HX、全部三个 A100 SKU、Drive A100）读出 `0x00000005`，在每个 GA10x 部件上读出 `0x00000085`。170HX 相对于普通 A100 并没有被特别锁定。

---

## 代码说了什么

在正式发布的 `master` 树中 grep `nvlink` 和每一个 NVLink 寄存器地址都返回空。`common/constants.yaml`、`driver/build.sh`、`driver/VERSION`、`install.sh`、`remove.sh`、`README.md` 以及六个补丁（`0001-sec2-postbl-plm-ss-cfg.patch` 到 `0006-persistent-sw-state.patch`）里都没有。`constants.yaml` 只声明两个驱动版本、设备 ID `20c2`/`2082`、计算值 `ss0: 0x88888888` / `ss1: 0x00000008` 和两个显存 profile。

未发布的分支也一样。在全部十二个（`80`、`Gen2`、`PG199`、`clanker/driver-port`、`debug-gen2`、`deced`、`docs`、`ecc`、`far`、`housekeeping`、`memory`、`multiple-cards`）中，NVLink 恰好出现一次，作为一行表格：

```markdown
| PCIe Gen2 x4 | Platform-dependent (no separate Root-port patch) |
| ECC | Planned |
| NVLink | Planned |
```

那一组行只在 `housekeeping` 和 `memory` 分支的 README 中。任何地方都没有 NVLink 逻辑。

---

## 两条覆盖路径，以及为什么两者都关闭

### 路径 A：`CTRL_OPT_NVLINK`

这是整个语料中被引用最多的"下一步"，而**从来没有人尝试过写入**。它读出 `0x00000000`，被文档记载为*有效的*每链路使能/禁用字段，并被描述为可写。它看起来就是那个杠杆。

它被一个强先验关闭，而不是被已执行的实验关闭：

- `FUSE_EN_SW_OVERRIDE` 在 `0x00820040` 处于 170HX 和所有数据中心 GA100 部件上 = `0x00000000`，而在所有消费级和工程样品部件上 = `0x00000001`。`CTRL_OPT` 覆盖机制本身在熔丝层面被禁用。
- `FUSE_DIS_SW_OVR` 在 `0x00820084` 处在所有卡上 = `0x00000001`。
- 在未签名 FwSec VBIOS 尾部（`0x43A00`-`0x47700`，MAC 校验范围外的 15,616 字节）偏移 `0x47341` 处找到的 25 项 `NV_FUSE_CTRL_OPT_*` 表，在 13 张被探测的 GA100 卡上读出全零，在此处是惰性的。

任何经由 `CTRL_OPT_NVLINK` 的计划都必须先击败 `FUSE_EN_SW_OVERRIDE`，而这样的机制不存在。

### 路径 B：`FEAT_OVR` 式攻击

它有吸引力，因为正式发布的计算解锁正好生活在这个寄存器块中，而且主覆盖开关熔丝 `FUSE_FEAT_OVR_DIS` 在 `0x008203F0` 处在所有卡上读出 `0x00000000`（也就是说，它**没有**被熔断）。推理过程是：如果计算限流可以在这里被覆盖，也许 NVLink 也可以。

它被直接排除，因为该块中没有 NVLink 寄存器。`0x00823800`-`0x0082382C` 的完整清单：

| 地址 | 名称 |
|---|---|
| `0x00823800` | `FEAT_OVR_ECC_PLM` |
| `0x00823804` | `FEAT_OVR_PLM` |
| `0x00823808` | `FEAT_OVR_QUADRO` |
| `0x0082380C` | `FEAT_OVR_ECC` |
| `0x00823810` | `FEAT_OVR_ECC_1` |
| `0x00823814` | `FEAT_READOUT_0` |
| `0x00823818` | `FEAT_READOUT_1` |
| `0x0082381C` | `FEAT_OVR_SM_SPD` |
| `0x00823820` | `FEAT_OVR_SM_SPD_1` |
| `0x00823824` | `FEAT_OVR_ROW_REMAP` |
| `0x00823828` | `FEAT_READOUT_2` |
| `0x0082382C` | `FEAT_OVR_ECC_2` |

十二个条目覆盖 ECC、Quadro 分类、SM 速率、行重映射和只读输出。没有任何可写的。在同一个块上做的 PCIe 尝试是一个有用的对照，它是探测结果而不是第二个寄存器：对 `0x00823800` 的高安全写入回读 `0xfffffe8e`，因此写入生效了，但 `OPT_GEN23`（`0x82057C`）保持 `0x1`，链路保持 Gen1。当时该结果被解读为 PCIe 覆盖使能被熔断**关闭**，尽管上面的清单中没有 PCIe 条目。`SM_SPD` 在 `0x0082381C` 是一个真实条目且被熔断**打开**，这就是为什么[计算解锁](../unlock/compute-throttle.md)经由该路径工作而 [PCIe 速率解锁](pcie-gen3-gen4.md)不行。

### DevInit 角度

DevInit 确实读取这颗熔丝。CMP DevInit 反汇编中 `0x1482xxxx`（MMIO `0x82xxxx`）访问的完整清单包括 `0x820684`，以及 `0x820C14`/`0x820D38`（FBIO/FBP floorsweep）、`0x82380C`/`0x823814`、`0x820520`（`MAGIC_D`）和 `0x820148`。任何来源都没有写入它，没有命名有效的覆盖，而且**没有人追踪过该值被读取后发生了什么**（置信度：中；依据：访问清单，而非完整追踪）。

---

## 死路

下面每一个都是有人认真追过的真实、合理的想法。

| # | 想法 | 为什么看似可行 | 怎么死的 |
|---|---|---|---|
| 1 | "NVLink 已经在启动日志里显示了，所以我们只需要一个桥接器" | `nvidia-nvlink: Nvlink Core is being initialized, major device number 236` 确实在每次启动时出现 | 该行由 `nvidia-nvlink.ko` 软件核心库在 `nvlink_linux.c:344` 发出，宣告它已加载。以 `DBG_INFO` 级别记录，在几乎所有 GPU 的几乎每次驱动加载时都会出现，发生在 GPU/GSP 启动之前的早期模块加载期间。`236` 是 `alloc_chrdev_region` 动态分配的字符设备主号，每次启动可能不同。唯一一次记录的 `nvidia-smi nvlink` 运行返回 "Device does not have or support Nvlink." |
| 2 | "HULK" 加密阻塞器 | 它是唯一已发布的解释，出现在一个项目相关的 gitbook 上，语气权威 | 网站维护者于 2026-07-20 否认了它（"This hasn't been updated in some time, don't rely on that"），页面自己的作者也称其过时。任何熔丝读数、VBIOS 转储或 DevInit 反汇编中都没有佐证有加密方案门控 NVLink |
| 3 | "`FUSE_NVLINK_PHYS_DMG = 0x1` 意味着链路被标记为损坏" | 寄存器名是 `OPT_SECURE_NVLINKS_PHYSICAL_DAMAGE_WR_SECURE`；置位的损坏标志会是单向门 | 在全部十四张被探测的 Ampere 卡（包括健康的 A100）上读出 `0x1` |
| 4 | "NVLink 是软件锁定的" | 其他几个 170HX 限制确实是固件侧的 | 禁用来自 OTP 熔丝 `0x00820684` 并被镜像进只读状态寄存器。记录它是因为它在语料最后一天 2026-07-27 仍在流传 |
| 5 | Titan V 类比：那里的 NVLink 被 VBIOS 禁用 | 一个真实的前例 | 在 170HX 上该值来自 OTP 熔丝，而不是 VBIOS 设置。机制不可迁移 |
| 6 | "有些 die 有可用的 VRAM 但 NVLink 块是坏的，所以才有捡料" | 这正是报废捡料通常的工作方式 | `FUSE_NVLINK_DEFECTIVE` 在每张被探测的 170HX 上 = `0x00000000`。那颗熔丝恰恰是会记录坏链路组的字段 |
| 7 | 按照 A100 原理图焊接缺失的 NVLink 元件 | 板卡匹配，候选元件已按位号识别 | 依次被下列因素阻塞：净室政策（原理图被提供并被拒绝）；三颗 GPU 到地的端接电阻没有可见走线，需要 boardview 或拆除 GPU 并做专业红外返修；`R976` 落在一颗至少 82 行球阵列封装的**芯片下方** `F51` 球上；以及决定性的——完美的返修后 `FUSE_NVLINK_DIS` 仍然是 `0x00000007` |
| 8 | 先表征 NVLink 信号完整性 | 对数十 GHz 差分接口来说是正确的工程顺序 | 唯一可用的 60 GHz 示波器被判定不足；租用足够设备一个月估计要几千。采纳的结论："Not like we need traceability on DIY nvlink boards. They either work or they don't."（DIY nvlink 板不需要可追溯性。要么能用要么不能用。） |
| 9 | A100 桥接器上的 Microchip SM806022 时钟发生器 | 一个真实、规格正确的元件（52.08333 MHz 晶振输入，两路 156.25 MHz 差分 HCSL 输出）确实出现在消费级 Ampere 桥接器上 | 直接检查官方 A100 桥接器：裸 PCB，无时钟发生器。命名它的拆解总结是机器从消费级桥接器材料生成的 |
| 10 | A100 桥接器包含一个存有设备 ID 的 EEPROM | 消费级桥接器确实带一个 | "a100 nvlink has neither eeprom or sig gen"，来自直接检查。消费级 EEPROM 被认为保存每板的产线末端阻抗表征，而不是 ID。SXM2 基板也确认："No, only traces"（没有，只有走线） |
| 11 | 便宜的 A100 4 卡有源 NVLink 背板 | 据报存在；可以直接解决拓扑问题 | NVIDIA 只文档化了成对的全部三桥 Ampere 拓扑，NVSwitch 只存在于 SXM 平台内部。识别出的唯一真实产品是一块无交换机的中国 4x SXM V100 背板，是另一代产品且接线未知 |
| 12 | 单槽 8 路 NVLink 背板 | 有真实的 PCB CAD 工作：网格中重复的 `NVLink_MiniCoolEdge_124pin` 封装、差分对布线、`SlimSAS_MCIO_8x` 连接器、铜皮中写着 "A100" | 没有制造板卡、没有链路起来、没有测带宽。8 路需要只在 SXM 中存在的 NVSwitch。而且熔丝仍然是 `0x7`。唯一的信号完整性输入是机器写的 EM 模拟器，预测走线"在 37ghz 下做了很多天线的事，但模拟器说它勉强能用" |
| 13 | 买一个桥接器，逆向它，制造副本 | 两位有数据中心硬件经验的人评估桥接器为完全被动，而且约 200 欧元一个的经济性很残酷 | 没人买、没人造、语料中从没有人手里有桥接器："I don't have a bridge to test"（我没有桥接器可测）。而且熔丝还挡着，做这事也没意义 |
| 14 | 从头制造 A100 中介层 | "the a100 interposer is pretty simple, just needs the connector"（a100 中介层很简单，只需要连接器），并有具体的信号完整性策略（Megtron 层压板、非标准卡间朝向以缩短路径） | 连接器需要批量订购，还有一个公开担忧：90 度面板安装版本可能受出口管制（提出边缘安装作为变通）。没有订购连接器、没有制造板卡。而且中介层会插进死的硅片里 |
| 15 | 把两块 PLX 背板面对面安装让边缘连接器对齐 | 完全绕开插槽间距 | 纯推测，从未画图、从未估算成本，被同一个熔丝阻塞 |
| 16 | 把 NVLink 当作多卡带宽问题的修复 | 基线是约 1 GB/s 的 PCIe Gen1 x4，张量并行被反复称为"没有 nvlink 就是浪费时间" | 被一次一手测量缓和：2x RTX 3090 带 NVLink 在 27B 模型 vLLM 张量并行下只显示约 10 % 吞吐提升。对应的反方论点——在 Gen1 x4 基线上相对增益会大得多——只是推理，在熔丝挡着时无法测试 |
| 17 | "CMP PCB 完全没有 NVLink 连接器" | 有人确实观察到一块带 NVLink 开口的导流罩盖在没有连接器的 PCB 上 | 该观察属于 CMP **90HX**，一块 GA102 RTX 3080 级板卡，同一无品牌制造商出品的 "RTX 3080 20GB" 兄弟卡确实用带 NVLink 连接器的 PCB。应用到 170HX 上它拆解证据相矛盾（置信度：中；这是内部调和，不是新观察） |

---

## 物理状况

独立于熔丝，还有机械和元件装填两个问题。

- **金手指存在。** 170HX 复用 A100 板卡布局；NVLink 边缘金手指物理存在，且有三个桥接器连接器位置。由外部拆解于 2023-10-25 确定，卡主们同意。
- **导流罩挡住它们。** 任何桥接器就位前必须机加工或移除铝制盖板，与 Tesla P100 情况相同。NVIDIA 在 A100 上用橡胶覆盖连接器，桥接器夹在 A100 外壳上，因此给 170HX 装桥接器还需要弄一个带卡扣的 A100 外壳或制造等效件。存在一张用打磨机开孔的 P100 照片；其电气结果未知。据报道有一款 Bykski 水冷头会露出 NVLink 区域。
- **桥接器是傻的。** 官方 A100 NVLink 桥接器是裸被动 PCB：无时钟发生器、无 EEPROM、无 retimer、无包处理 ASIC。消费级 3090 SLI 桥接器*确实*带时钟发生器，据信是因为 NVIDIA 无法假设消费级主板提供相同的 PCIe 参考时钟。从 Ampere 到 H200-NVL 的所有桥接器都被评估为"傻桥接器"；交换机只出现在后来的世代。
- **你买不到第三方货。** 唯一生产过的第三方 Ampere 桥接器是已停产的 ElmorLabs NVB-3S，一个面向 RTX 3090、RTX A5000 和 RTX A6000 的 3 槽部件，不是 A100 部件。在两个中国市场上的市场调查只发现官方 2 槽和 3 槽桥接器，价格统一，暗示交易量极低。

> [!NOTE]
> **开放问题：PCB 的 NVLink 区域是否装填了元件？**
>
> 这是该领域影响最大的开放问题，因为它决定熔丝绕过是否真的有用。证据倾向**未装填**：语料中唯一直接的 A100 对比 CMP 板卡比较报告元件缺失，而反方说法是原理图推断而非观察。
>
> **未装填：** 2023 年拆解声称"the gold fingers of the NV-Link interface exist, but the feature is unsupported with all components unpopulated on the PCB"（NV-Link 接口的金手指存在，但该功能不受支持，PCB 上所有元件未装填），并另外说"ICs related to the NV-Link interface are also missing"（与 NV-Link 接口相关的 IC 也缺失）。一位依据 A100 原理图工作的研究者识别出 GPU 上方五颗具体的未装填电阻（`R234` 000、`R237` NP、`R236` 1k、`R1024` 000、`R238` 000，全部在第 17 页），外加 `R976`、`R1029`、`R1030` 和三颗 GPU 到地端接电阻。另一位参与者回忆起"absent parts of power supply to nvlink"（nvlink 供电缺失元件）。
>
> 那份电阻清单来自直接对比两块板卡："they are populated on a genuine A100, but missing on CMP"（它们在真品 A100 上装填，在 CMP 上缺失）。这是语料中唯一这样的并排对比。
>
> **已装填：** 项目自己的 VBIOS 对比表声称"NVLink bridge, external bridge absent (PCB fully populated)"（NVLink 桥接器、外部桥接器缺失（PCB 完全装填）），这是项目文档行而非检查结果。在电阻清单发布*两小时前*，另一位研究者说"我不认为有任何缺失的 NVlink 元件。根据原理图，GPU die 直接连接到边缘连接器"，把混乱归因于桥接器含有有源元件"包括一个 ROM 芯片"。最后这个前提本身被证伪：下面的死路 #10 记录了直接检查发现 A100 桥接器上没有 EEPROM。
>
> **复杂的细节：** `R237` 在 A100 原理图本身中被标记为 **NP**（未装填），所以五颗中至少有一颗在真品 A100 上也被预期缺失。这展示了目测对比多么容易误导，也是结论是"倾向未装填、一次直接对比、未被反驳"而不是定案的原因。没有人为了记录给两块板卡的该区域拍过照。

---

## 拓扑与带宽，供将来需要时使用

记录下来以免有人重新推导，也因为一些流传的数字是错的。

| 数值 | 值 | 置信度 |
|---|---|---|
| A100 PCIe 支持拓扑 | 2 个 GPU，必须全部三个桥接器 | 高 |
| A100 每桥接器带宽 | 200 GB/s | 高 |
| A100 成对总计 | 600 GB/s | 高 |
| Ampere 端口结构 | 4 个子端口 x 4 条 lane，每 lane 50 Gbps，被表述为每端口 200 Gbps；4 x 4 x 50 是 800 Gbps，所以分解和数字不可能同时正确 | 中 |
| GA102（RTX 3090）第三代每链路 | 14.0625 GB/s 双向，四个 x4 链路 | 高 |
| GA102 总计 | 56.25 GB/s 双向，两个 GPU 之间 112.5 GB/s 总聚合 | 高 |
| NVSwitch | 仅 SXM 平台（例如 DGX）；8 路 | 高 |

有三个比值说法在流传，没有一个被干净地定案。频道在 A100 对比 3090 上定为 **3x**（600 对比 200 GB/s），但 NVIDIA 文档记载的 GA102 数字是 112.5 GB/s 总聚合，得出 **5.33x**。同一讨论中引用的 3090 的 200 GB/s 数字被描述为"200 GB/s 级桥接器降频"，这论证了 3x 比较用了错误的约定。两种读法都同意早先的"A100 的 NVLink 带宽是 3090 的 6 倍"说法是错的。什么能定案：明确说明 A100 的 600 GB/s 数字是单向总和还是总聚合。

> [!NOTE]
> **开放问题：2 路还是 4 路被动？**
>
> 三个连接器恰好是四节点全连接网格所需的节点度数，且每条边 200 GB/s 跨 3 条边就是每卡 600 GB/s 聚合，与成对数字算术上相同。所以 4 路在几何上自洽。**没有**被确定的是 NVIDIA 的驱动或固件会在 PCIe GA100 上把链路训练到三个不同对端。没有文档这么说，也没人演示过。两种说法说的是不同的事（几何形状对比受支持的配置），可能都对。

> [!WARNING]
> **不要按 320 GB 数字来规划装机**
>
> 一次 4 卡 NVLink 讨论引用了四张 10 GB 卡的 320 GB 内存池。那假设每卡 80 GB。正式发布解锁给 10 GB 卡 **40 GB**，所以四张池化 **160 GB**。四张解锁的 8 GB 卡池化 **256 GB**。80 GB 配置被尝试过并发现不稳定：参见[80 GB 尝试](80gb.md)。

---

## PCIe 对等直连备选方案

因为 NVLink 不可达，PCIe P2P 是今天唯一有任何机会能用的跨 GPU 加速路径。它不在 cmpunlocker 中：在 `master` 和每个分支中 grep `p2p` 和 `peer` 只返回 `build.sh` 安装列表中的原版 `nvidia-peermem.ko` 和 `0008` diff 中一行未修改的上下文（`nv_uvm_resume_P2P(pUuid)`）。任何分支都不包含 P2P 使能。

候选是 `tinygrad/open-gpu-kernel-modules` 的社区 fork，默认分支 `610.43.03-p2p`，**与 cmpunlocker 针对的驱动版本相同**。`HEAD~3` 是提交 `452cec62d827` "610.43.03"（2026-07-07），一个纯粹的 NVIDIA 发布导入。上面有三个提交：

| 提交 | 内容 | 大小 |
|---|---|---|
| `9fb650447c7b` | 组合 P2P 修改 | 8 个文件，+83/-28 |
| `52670f7fd6a7` | 实验性 hugepage `cudaHostRegister` 加速 | 7 个文件，+383/-97 |
| `2849449f8cd6` | README | +245 |

P2P 提交触碰 `install.sh`（+7）、`kernel-open/nvidia-uvm/uvm_gpu.h`（+7）、`kernel-open/nvidia/nv-reg.h`（+1/-1）、`src/nvidia/generated/g_kern_bus_nvoc.c`（+5/-5）、`src/nvidia/src/kernel/gpu/bif/kernel_bif.c`（+3/-3）、`src/nvidia/src/kernel/gpu/bus/arch/pascal/kern_bus_gp100.c`（+10）、`src/nvidia/src/kernel/mem_mgr/io_vaspace.c`（+11/-10）和 `src/nvidia/src/kernel/rmapi/nv_gpu_ops.c`（+39/-9）。它在没有 NVLink 时启用 BAR1 P2P，有 NVLink 时回退到 NVLink；对于 PCIe 对，传输通过 DMA 直接写入另一块 GPU 的物理地址。

> [!WARNING]
> **实验性：GA100 不在支持列表中**
>
> 该分支列出 RTX 3090（有 NVLink 时成对 NVLink，否则 PCIe BAR1）、RTX 4090 和 RTX 5090。**GA100 不在该列表中，该补丁从未在 170HX 上测试过。** P2P 路径触碰 `kern_bus_gp100.c`、`io_vaspace.c` 和 `nv_gpu_ops.c`，所以 GA100 代码路径可能根本不存在。

> [!CAUTION]
> **只取 P2P 提交，不要取 hugepage 提交**
>
> `52670f7fd6a7` 声称对 1G-hugepage 背书的缓冲区把 `cudaHostRegister` 加速约 5000 倍，并缩小此类映射的设备页表。其作者声称它是自动启用的，且"this path skips some of the per-4K-page bookkeeping the stock driver performs, so it may misbehave in edge cases the stock driver handles correctly"（该路径跳过了原版驱动执行的部分每 4K 页簿记，所以在原版驱动正确处理的一些边缘情况下它可能出错）。把它当作独立于解锁补丁的不稳定源。

该分支记录的设置要求：`GRUB_CMDLINE_LINUX_DEFAULT` 中加 `amd_iommu=on iommu=pt` 或 `intel_iommu=on iommu=pt`，`update-grub`，安装 610.43.03 驱动，运行 `./install.sh`，重启。IOMMU 必须处于**直通**模式而非转换模式，否则 DMA 会走 IOMMU 页表且传输失败。README 明确警告这对"如果你运行不受信任的软件或设备"是"非常危险的"。如果 P2P 慢，根端口上的 ACS 会迫使所有 GPU 到 GPU 的流量穿过 CPU 根复合体；在 BIOS 中禁用它，或用 `pcie_acs_override=downstream,multifunction`，或用 ACS 覆盖内核补丁。

完整处理见 [P2P](p2p.md)。

---

## 什么才能真正推进此事

最易处理在前。只有前两个便宜。

### 1. 做那个没人做过的写入

在全部 31 份存档解锁器附件和每个净室产物中，NVLink 只以熔丝读出形式出现。没有探测脚本、没有覆盖尝试、没有记录的写入。在一块可牺牲卡上对 `CTRL_OPT_NVLINK`（`0x008209B8`）和 `CTRL_OPT_PERLINK`（`0x00820820`）做一次读-写-读探测，然后重读 `STATUS_OPT_NVLINK`（`0x00820DB8`），花一次会话。

> [!CAUTION]
> **只写在可牺牲卡上**
>
> 这些是安全熔丝影子寄存器。关于写入它们的一般谨慎正是没人写的原因。预期结果：写入被丢弃，状态保持 `0x00000007`。这个负面结果仍值得记录在案，因为目前语料甚至不能说它被尝试过。

### 2. 拍摄 NVLink 元件区域

一块去罩 170HX 在 `R234`、`R236`、`R237`、`R238`、`R976`、`R1024`、`R1029`、`R1030` 位号周围的高分辨率照片，与真品 A100 并排，外加从 NVLink 边缘金手指到 BGA 球 `F1` 和 `G1` 的通断检查（`R1029`/`R1030` 在芯片边缘连到那些球，可以用细线触达）。便宜、决定性、需要一张卡和拆一次罩。维护者于 2026-07-19 把这个命名为实际第一步，而它从未被做。

### 3. 追踪 DevInit 对 `0x820684` 的读取

`0x820684` 在 DevInit 访问清单上。没有人跟着读取走完反汇编，看结果是否被写入某处或只是被消费。如果在 OTP 和状态寄存器之间存在可伪造的消费方，就在这里。只被工作量阻塞，以及被阻塞 PCIe 熔丝层的同一堵墙阻塞。

### 4. 把 3 位禁用字段对照 12 条物理链路解码

`FUSE_NVLINK_DIS[2:0]` = `0x7` 对照 `PTOP_SCAL_NUM_NVLINK` = 12，而 `STATUS_OPT_NVLINK` 被注释为 16 位字段却也读出 `0x00000007`。**工作假设，未确认：** 每组四条的三个链路*组*（12 = 3 x 4），这将解释反复出现的"all groups"措辞和 RTX 3080 对照其 `PTOP_SCAL_NUM_NVLINK` 为 `0x4` 时的 `0x1`。语料中没有任何东西确认这一点。什么能定案：探测一块已知部分 NVLink floorsweep 的 A100，或找到 NVIDIA 关于 GA100 上 `NV_FUSE_OPT_NVLINK_DISABLE` 字段宽度的文档。

### 5. 坐上一个桥接器，看会发生什么

没人运行过的唯一实证测试。语料中从来没有人同时拥有一张 170HX 和一个 A100 NVLink 桥接器。一个桥接器、一次导流罩改造（或一块露出该区域的水冷头），然后 `nvidia-smi nvlink` 和 dmesg。鉴于熔丝，预期为负面，但语料目前甚至无法确认连接器对齐正确。

### 6. 中介层制造

在项目 5 返回正面结果之前没有意义。降级优先。

### 7. 真正的熔丝绕过

桌面上没有任何可着手之处。进展还额外被潜在贡献者没有卡阻塞："I wanted to work on it but I cant get any cards. So you have to wait until someone else figures it out."（我想做但它拿不到卡。所以你得等别人搞明白。）

---

## 立场是如何移动的

| 时期 | 曾相信 | 被取代为 |
|---|---|---|
| 2023-10-25 至 2026-05-07 | NVLink 不受支持是因为硬件缺失（拆解：金手指存在，元件未装填） | 一个有实测的熔丝故事：die 扩展到 12 条链路，没有一条被标记为缺陷，禁用是读出 `0x7` 的 OTP 熔丝。对于硅片相信什么，直接 BAR0 回读胜过照片拆解。注意两者**并非**互斥；装填问题仍然开放 |
| 2026-05-31 | 这颗熔丝可能是值得攻击的挖矿 SKU 限制 | Drive A100 32 GB 读出相同的 `0x7`/`0x7`。这是通用 GA100 划分 |
| 2026-05-31 起 | "NVLink killed, CTRL_OPT override path under investigation"（NVLink 被干掉，CTRL_OPT 覆盖路径调查中）（仍在 VBIOS 对比表中印着） | 同一文档内被 `FUSE_EN_SW_OVERRIDE` = `0x0` 取代："CTRL_OPT fuse override disabled, cannot be changed, inert on 170HX"（CTRL_OPT 熔丝覆盖已禁用、无法更改、在 170HX 上惰性）。参考表内部不一致；熔丝测量胜出 |
| 2026-07-07 至 2026-07-10 | A100 存在便宜的 4 卡有源 NVLink 背板 | 只有成对拓扑；NVSwitch 仅限 SXM |
| 2026-07-18 至 2026-07-21 | A100 桥接器含有源电路（时钟发生器、EEPROM） | 直接检查：裸 PCB |
| 2026-07-19 | "it has triple (200GB/s?) NVLink, so PCIe is a non-issue"（它有三路 (200GB/s?) NVLink，所以 PCIe 不是问题） | 同一天在被问"doesn't work though, right?"（但它不工作，对吧？）后自我撤回 |
| 2026-07-20 | 阻塞器是"用名为 HULK 的加密破解某种安全设计架构" | 被网站维护者和页面自己的作者否认。从未发布替代解释 |
| 2026-07-19 至 2026-07-27 | "worth trying, probably just a bridge"（值得一试，可能只需要一个桥接器） | "Might need to consider the state of NVLink, it's a lot harder than I thought to get working"（也许需要考虑 NVLink 的状态，让它工作比我想的难得多）。实际第一步被重新定义为重新设计外壳以获得物理访问，然后拍摄元件区域。被问到选择一个月还是一年时，答案是研究完成前不能说任何定论 |

---

## 实测值

| 数值 | 值 | 条件 | 置信度 |
|---|---|---|---|
| `FUSE_NVLINK_DIS` `0x00820684` | `0x00000007` | 两块 170HX；Drive A100 32GB（PG199） | 高 |
| 同上 | `0x00000000` | A100 SXM4 40G、A100 PCIe 40G、A100 PCIe 80G、A10、A5000、A6000、RTX 3090、RTX 3090 Ti | 高 |
| 同上 | `0x00000001` | RTX 3080、RTX 3080 Ti | 高 |
| `STATUS_OPT_NVLINK` `0x00820DB8`（RO） | `0x00000007` | 两块 170HX；Drive A100 | 高 |
| `FUSE_NVLINK_DEFECTIVE` `0x0082068C` | `0x00000000` | 每张被探测的卡；15 卡调查中每张返回值的卡都是 `0`，A16 和 ES 列除外 | 高 |
| `FUSE_NVLINK_DIS_CP` `0x00820688` | `0x00000000` | 每张被探测的卡 | 高 |
| `OPT_SECURE_NVLINK_MASK_WR_SECURE` `0x00820704` | `0x00000005` GA100 / `0x00000085` GA10x | 干净的架构分裂 | 高 |
| `OPT_SECURE_NVLINKS_PHYSICAL_DAMAGE_WR_SECURE` `0x00820BD4` | `0x00000001` | 全部 14 张被探测的 Ampere 卡上统一 | 高 |
| `FUSE_NVLIPT_RST_DIS` `0x00821100` | `0x00000000` | 每张被探测的卡 | 高 |
| `CTRL_OPT_NVLINK` `0x008209B8` | `0x00000000` | 每张被探测的卡，包括 170HX | 高 |
| `CTRL_OPT_PERLINK` `0x00820820` | `0x00000000` | 170HX | 高 |
| `PTOP_SCAL_NUM_NVLINK` `0x0002246C` | `0x0000000c`（12） | 两块 170HX、所有 A100 SKU、Drive A100 | 高 |
| 同上 | `0x00000004`（4） | A10、A5000、A6000、RTX 3080/3080 Ti/3090/3090 Ti | 高 |
| 同上 | `0x00000000` | 仅 A16 | 中 |
| `FUSE_EN_SW_OVERRIDE` `0x00820040` | 170HX 和数据中心 GA100 上 `0x00000000` / 消费级和 ES 上 `0x00000001` | | 高 |
| `FUSE_DIS_SW_OVR` `0x00820084` | `0x00000001` | 所有卡 | 高 |
| `FUSE_FEAT_OVR_DIS` `0x008203F0` | `0x00000000` | 所有卡；主覆盖开关熔丝**未**熔断 | 高 |
| 未签名 FwSec VBIOS 尾部 | `0x43A00`-`0x47700`，MAC 范围外 15,616 字节 | 在 `0x47341` 处持有 25 项 `NV_FUSE_CTRL_OPT_*` 表，在 13 张 GA100 卡上全零 | 高 |
| DevInit 读取 NVLink 熔丝 | `0x820684` 出现在 `0x1482xxxx` 访问清单中 | 只读，从不写入 | 中 |
| `nvidia-smi nvlink` 输出 | "Device does not have or support Nvlink." | 一台租用的 8 卡 64 GiB 主机，2026-07-24，GPU 名称被遮蔽；语料中唯一的抓取 | 中 |
| dmesg NVLink 行 | `nvidia-nvlink: Nvlink Core is being initialized, major device number 236` | 良性，软件核心加载 | 高 |
| 正式发布 `master` 中的 NVLink 引用 | 0 | 全树 grep | 高 |
| 全部 12 个分支中的 NVLink 引用 | 1 个词 `Planned`，出现在两个 README 表中 | 任何地方都没有代码 | 高 |
| 4x 解锁 10 GB 卡池 | 160 GB（4 x 40960 MiB） | 正式发布 `constants.yaml` | 高 |
| 4x 解锁 8 GB 卡池 | 256 GB（4 x 65536 MiB） | 正式发布 `constants.yaml` | 高 |
| A100 桥接器市价 | 约 200 欧元一个，且受支持的 A100 对需要全部三个 | 2026-07-26 市场核查 | 中 |
| NVLink 走线频率估计 | 37 GHz（机器写的 EM 模拟器）对比约 60 GHz（二手消息） | 冲突；两者都不是从 50 Gbps lane 速率推导的 | 低 |
| 2x RTX 3090 vLLM TP、27B 模型 | 带 NVLink 约 10 % 吞吐提升 | 一手、单一测试者 | 中 |
| 2x RTX 3090 vLLM、发布的第三方数据 | 715 对比 483 t/s 输出；6,790 对比 4,583 t/s 吞吐 | 模型、量化和批处理设置未说明，因此与上面不可比 | 中 |

> [!NOTE]
> **群组注意事项**
>
> 参考表中 A16 列对每个 NVLink 熔丝行读出占位符 `BAR0`。上面"on all cards"（所有卡）的表述应理解为熔丝行排除 A16。A16 是唯一报告零 NVLink 扩展能力的 Ampere 部件，但其实际的禁用熔丝状态从未被捕获。

---

## 参见

- [NVLink 硬件](../hardware/nvlink-hardware.md)，了解连接器和板卡细节
- [熔丝与 OTP](../hardware/fuses-and-otp.md)，了解完整熔丝群和方法论
- [计算限流](../unlock/compute-throttle.md)，了解确实有效的 `FEAT_OVR` 路径
- [PCIe Gen3 与 Gen4](pcie-gen3-gen4.md)，了解另一个熔丝门控前沿
- [P2P](p2p.md)、[状态板](status-board.md)、[开放问题](open-questions.md)
