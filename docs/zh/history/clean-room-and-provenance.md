# 净室与出处问题

## 本页涵盖

CMP 170HX 解锁是在一套明确的净室协议下开发的，而该协议存在的唯一意义，是要回答最终产出的代码是否可能在未使用 2022 年 2 月 LAPSUS$ 窃取的 NVIDIA 资料的情况下被写出来。本页记录净室是如何组织的、规则是什么、允许的输入是什么、当时的出处评估结论及其推理、后来的字节级对比确立了什么，以及一份泄露的私有概念验证如何在 2026-07-18 进入画面。

两个结果主导着以下所有内容。

1. **正式 ROP payload 中的每一个常量，除了一个之外，都能追溯到一份早于正式补丁的、带日期的公开或净室衍生工件。** gadget 地址来自 2026-07-01 发布的解密 debug-booter 反汇编，gadget 语义来自 2026-07-10 发布的自动生成图谱，DMEM 栈帧网格来自 2026-07-15T04:47Z 的公开 git 提交，缓冲、守卫、填充与大小常量来自 2026 年 6 月的学术预印本。例外是种植在 payload 偏移 `0x1100`（`D[0x1900]`）处的 `0x00000007`，没有任何早于补丁的带日期工件能解释它。payload 中没有任何其他内容需要泄露材料才能推导。
2. **正式 `cmpunlocker` 驱动补丁集逐行就是泄露的 `patch.diff`**，添加了 10 GB 双几何支持、移除了一处恶意改动，在 `patch.diff` 发布 70 分钟后被采纳。原则上可推导与实际上的确推导不是一回事，而源材料无法定案到底发生了哪一种。

本页如实报告两者，陈述双方各自的论点，然后就此打住。它不提供法律结论，全页不点名任何人。代码本身参见 [六个驱动补丁](../unlock/driver-patches.md) 与 [ROP 链](../unlock/rop-chain.md)；日期参见 [项目时间线](timeline.md)。

---

## 为什么会有净室

解锁并非始于净室。它始于 2026 年 3 月一个公开的 GitHub issue 跟踪器，2026 年 4 月迁到一个 Discord 服务器，到 2026 年 5 月已经产出了决定性的密码学结果，使 AES 加密、RSA 签名的 `booter_load` 代码得以被读取。2026 年 6 月，一条能在该代码内跳转到任意地址的 ROP 链被演示并公开宣布。随后开发转入一个**七人**私人小组，产出了概念验证、日期为 2026 年 6 月的学术预印本，以及两份内部**驱动修改指南**（一份针对计算，一份针对显存）。

该私人小组做了两个决定，塑造了之后的一切：

- **发表论文，扣住漏洞利用代码**，等待独立复现。
- **删除原始 Discord 服务器**，理由是它可能含有从 NVIDIA 泄露的资料。

净室是为了从头复现被扣住的结果而创建的，只使用出处可以证明的输入。它要管理的是 LAPSUS$ 泄露事件：按后来出处评估的描述，这是 2022 年 2 月至 3 月的一次事件，泄露了约 **1 TB** 的 NVIDIA 数据，包括 GPU 驱动源码、内部硬件文档与固件签名密钥。泄露缓存至今仍可公开定位（频道内点名了 Internet Archive），这正是需要一条规则而非一种假设的原因。

---

## 规则集

2026-06-27 起作为频道政策声明、并在整个期间以删除与封禁威胁执行的管理标准：

1. **不得讨论任何 NVIDIA 机密。**
2. **秘密知识只有在能证明同一信息可从公开来源推导时方可接受。**
3. **发布泄露或非法材料者封禁。** 这明确涵盖泄露源码、泄露原理图，以及任何从被删除的早期 Discord 带过来的文件。

2026 年 6 月的预印本被指定为**唯一干净输入文档**，理由有二：它发布在学术出版站点上，而且它已发送给 NVIDIA。

规则 2 是承重的那条，也是 2026-07-18 事件施加压力最大的那条。参见下文 [两种解读，未定案](#两种解读未定案)。

---

## 净室与脏室：设想与实际

第一天就提议两队分工。声明的组织原则是常规的：**脏室**团队做逆向工程，产出不含任何非法内容的文档；一个什么都没看过的**净团队**仅凭这些文档重新实现结果。

那个分工大体被放弃了。实际成形的是**频道分工，而非团队分工**：

| 设想 | 实际发生 |
|---|---|
| 脏室团队，隔离，做 RE 并编写规格说明 | 源集合中没有文档把"脏室"当作运作团队使用 |
| 净团队，零暴露，仅凭这些规格说明实现 | 同一些人在多个频道间工作 |
| 靠团队成员身份强制隔离 | 靠频道主题与可接纳性规则强制隔离 |

实际存在的频道有一个只携带工作值的 `#general-how-to-cleanroom` 操作指南频道，以及更深入的技术频道，承载寄存器扫描、Falcon 退出路径分析与 DMEM 栈探索。对这一描述的置信度为**中**：意图在归档中被逐字引用，频道结构可直接观察，但两队协议没有任何证据表明曾被实际配备人员。

---

## 干净输入语料

净室允许自己依赖的一切，以及各自的理由：

| 输入 | 为何被视为干净 |
|---|---|
| **《A Canary in the Crypto Mine: Defeating Stack Protection in a GPU Secure Coprocessor》**，2026 年 6 月，Zenodo 记录 `20916112`，ResearchGate 出版物 `408132536`，16 页 | 发表在学术出版站点上并向厂商披露。被指定为唯一干净输入文档。2026-06-26 传阅；2026-07-16T06:07:12Z 以 `main.pdf` 发布到净室服务器 |
| `NVIDIA/open-gpu-kernel-modules`（tag `610.43.02`、`610.43.03`，以及更早的 `580.x`） | NVIDIA 自己发布的源码 |
| **debug** `booter_load` 二进制 | 作为 NVIDIA 自己的 `.ko` 内的 C 数组编译，并以 `g_bindata_kgspGetBinArchiveBooterLoadUcode_GA100.c` 发布在 open-gpu-kernel-modules 树中。用 Nouveau 固件提取工具（为 GA100 打过补丁）提取。小组花了两天走通这条路径 |
| NVIDIA **公开的 AES-128-ECB 测试密钥** | 来自 NVIDIA 公开的 Jetson Secure Boot 文档。密钥构造是 MD5 初始化向量 `...0123456789abcdef...`，密钥号作为最后一个字节。一个不含任何 NVIDIA 材料的自包含公有领域 Rijndael 解密器与密钥验证器（`rijndael-tool.zip`）在频道内发布 |
| **GA100 熔丝与寄存器参考表**：15 张 Ampere 卡上的 120 个寄存器 | 完全来自对参与者拥有或租用的硬件做只读 MMIO 探测。参见 [熔丝与 OTP](../hardware/fuses-and-otp.md) |
| 卡自身以及 A100 对照件的 BAR0 寄存器读取 | 对自有硬件的所有者侧测量 |

频道内就加密达成的框架是尖锐的：NVIDIA 的错**不是**使用了平凡的测试密钥，而是**在 debug 与生产分支里交付了恰好同一个二进制**，并签名了一个含严重漏洞的二进制。

### 15 卡差分语料

寄存器层面的出处论证依赖这张表。它比较由 `tools/mmio-probe/probe.sh` 读取的 120 个寄存器，范围涵盖：

- **2 × CMP 170HX 10 GB**，物理卡，2026-05-05 与 2026-05-07 探测
- **11 张通过 GPU 租赁提供商租来的卡**：A100 SXM4 40G、A100 PCIe 40G、A100 PCIe 80G、A10、A16、A5000、A6000、RTX 3080、RTX 3080 Ti、RTX 3090、RTX 3090 Ti。（熔丝表中有一列是工程样品件，但任何一行都不带值。）
- **2 × Drive A100 32 GB**（`GA100-550F-A1`，PG199），物理卡，2026-05-31 探测

恰好五个寄存器组区分同一硅片的 170HX 与 A100：SM 速率选择、PCIe 启动代次、NVLink 禁用、ECC 启用与 FBPA CFG1 几何。

两张物理 170HX 也被证明在关键处寄存器相同：**120 个寄存器中 107 个字节级相同**，全部 13 处差异都是逐芯片分档伪影（floorsweep 掩码及其 FBIO/STATUS 镜像、每单元 `FEAT_OVR_SM_SPD` 编码、`FEAT_OVR_QUADRO`、HBM 硅片身份寄存器，以及跟随 floorsweep 的 per-FBPA 回读）。每一条限制熔丝都完全一致。正是这一结果授权把从一张卡推导出的配方推广到另一张卡。

---

## 寄存器值的出处标准

2026-07-02 在频道内被接受，作为发布目标寄存器集的理由：

> 显存几何来自 VBIOS 与对 BAR0 地址空间的读取，其余一切都可以通过把 A100 的 BAR0 输出与 170HX 的做 diff、并把一切改成 A100 的值来推导。

生成干净 `booter_load` 汇编只用了 NVIDIA 开源驱动数据与 NVIDIA 公开测试密钥。120 寄存器跨变体熔丝表是该主张的独立支撑：它直接展示 A100 对 170HX 的差值，不引用任何内部文档。

---

## 承重技术事实：debug 在关键处等于生产

整套净室路线依赖一个经验结果。如果 `booter_load` 的 `-debug` 编译携带额外字节，那么从可读的 debug 反汇编推导出的每个 gadget 地址都会在生产硅片上偏移而无用，得到正确偏移的唯一途径就只剩下生产二进制（无法解密）或泄露源码。

该关切在 2026-07-02 被提出，并在同一时期以两种方式被驳倒：

- debug 与生产二进制**恰好同样大小**。
- 一条完全由 debug 反汇编构建的 ROP 链**在生产硅片上正确执行**。

> [!NOTE]
> **未解决问题**
>
> "同样大小加一条成功的链"是实例证明，不是构造证明。源集合中没有文档记录对 `IMAGE_DBG` 与 `IMAGE_PROD` blob 的实际字节级比较，而 bindata 归档使这种比较轻而易举。定案方式：对 `g_bindata_kgspGetBinArchiveBooterLoadUcode_GA100.c` 中的两个条目做哈希。还没有人发布那个哈希。

一个相关的争论是：触碰 GSP 签名本身是否破坏干净性。一派认为篡改签名使结果按定义不干净。反方立场无人反驳：面向返回的编程（ROP）复用已存在于被签名二进制中的代码、不需要访问 NVIDIA 源码，因此由合法解密二进制构建的链是干净的，唯一可能不干净的元素是 payload 本身——如果它被复制而非推导的话。生产 booter 从不被修改：每个可用工具都加载未修改的签名镜像，只把分离的 384 字节签名在 `PATCH_LOC 0x8900` 处拼回去。

---

## 2026-07-18 出处评估

一份标题为 **"Assessment: Is patch.diff Derived from LAPSUS$-Leaked Information?"** 的文档发布于 **2026-07-18T18:40:16Z**，恰好是 `patch.diff` 本身于 **2026-07-18T18:01:15Z** 发布后 **39 分钟**。两个时间戳都是从 Discord 消息 snowflake 解码到毫秒。

**范围。** 针对 `NVIDIA/open-gpu-kernel-modules` tag `610.43.03` 的一个补丁。

**干净源语料，恰好三项，不多不少：**

1. `paper.md`（Canary 预印本）
2. `how_to.discord.html` 与 `discussion.discord.html`，净室服务器两个频道的导出记录
3. `NVIDIA/open-gpu-kernel-modules` tag `610.43.03`

评估者没有可用的泄露缓存对比，评估本身也记录这是其结论的局限。

**方法。** 把补丁与这三份来源对比，然后把补丁干净地应用到克隆的仓库上并在上下文中检查。

**结论，逐字引用：**"现有证据不支持该补丁衍生自 LAPSUS$ 泄露信息的结论。"

### 评估归为干净的内容

| 类别 | 内容 | 记录的结论 |
|---|---|---|
| **来自论文的干净内容**（九个概念） | 栈金丝雀引用字漏洞（§5.1-5.3，Thesis 1）；通过无界签名长度复制造成的 DMA 溢出（§3.2、§5.5）；通过被签名 booter 进入 SEC2 Falcon HS 模式（§2.2、§5.4）；统一填充值 **V = `0x4a7`**（§5.5，模拟器 trace）；溢出签名大小 **SIGSZ = `0xf800`**（§5.5）；PLM 解锁作为 HS 代码执行后的枢轴（§6.1）；feature-override 影子寄存器概念（§2.1）；HS 起的 WPR2 拆除（§8.5）；以 GPU 为防御者、主机被攻破的倒置威胁模型（§2.3） | "干净。这些概念在论文中有完整文档，不需要任何特殊访问。" |
| **来自开放驱动树的干净内容**（内核内部 API） | `memdescCreate`、`memdescMapInternal`、`memdescFlushCpuCaches`、`memdescGetSize`、`memdescGetPhysAddr`；`pmaRegisterRegion`、`pmaGetRegionInfo`、`pmaGetFreeMemory`、`pmaGetTotalMemory`；`MEMDESC_FLAGS_ALLOC_IN_UNPROTECTED_MEMORY`；`os_open_and_read_file`；`kgspExecuteBooterLoad_HAL`、`kgspPopulateWprMeta_HAL`；`FB_REGION_DESCRIPTOR`、`PMA_REGION_DESCRIPTOR`；`NV_FLAG_PERSISTENT_SW_STATE`；`GPU_REG_RD32`/`GPU_REG_WR32`；`NV2080_CTRL_CMD_FB_GET_FB_REGION_INFO_PARAMS` | "干净。任何称职的内核模块开发者读源码都能发现这些。" 每个列出的符号都存在于正式补丁集中 |
| **干净但较早**（八项，见于日期为 2026-07-09 至 2026-07-17 的公开记录） | feature-override 地址 `0x823804`、`0x82381c`、`0x823820`、`0x9a0204`、`0x100ce0`；WPR2 寄存器 `0x1fa824`/`0x1fa828` 与 WPR2 carve 作为阻塞项；FB 几何 PLM 地址 `0x100b10`/`0x100b38` 与 `0x9a0148`/`0x9a014C`/`0x9a0108`/`0x9a010C`；`0x8403C4` 作为 `resetPLM`；三种命名的退出策略（`secure_teardown`、提前退出 `0x8117`、`multiwrite_then_mutexfree_cleanexit`）；某些寄存器的 FLR 持久性；一个用于直接控制 SEC2 的非安全 ucode loader；从 `D[0xFFC4]` 到 `D[0xFFF0]` 的 DMEM 栈探索 | "干净，但有重叠。" |

该表中有一处引用是错误的。"HS 起的 WPR2 拆除"被映射到论文 §8.5；§8.5 的标题是"FLR 上的持久性"，论证常开岛中持有的 override 值如何把瞬时漏洞利用变成持久状态。它不含 WPR2 拆除，也没有 ROP 讨论。论文中唯一的 ROP 引用是 §5.5 中的一处引用。实际后果是：正式补丁在每次 Booter 传递前后显式保存与恢复 WPR2 这件事，并不像该表暗示的那样被论文覆盖。评估另行且正确地归因了公开记录中的 WPR2 处理，所以这是引用错误而非实质错误。

### 评估关于外部来源的三个论点

这些被记录为论点，而非事实。

1. **否定证据。** 补丁不含 NVIDIA 内部代码注释、不含泄露构建中出现的暗示性变量名、未使用泄露的签名密钥（漏洞利用不伪造任何签名），也没有任何通过 BAR0 探测同样无法发现的内部专有寄存器名。论文的道德声明直接佐证了关键一点："我们没有提取签名密钥，也没有伪造签名。"
2. **大锤论点。** `patch.diff` 在 `subdevice_ctrl_gpu_regops.c` 的 `gpuValidateRegOps` 中把 `return NV_OK;` 作为第一条语句插入，使原函数体成为死代码。它是无条件的，影响所有 GPU 而非仅 CMP 170HX，并完全禁用控制面板的寄存器读写校验。由此推出的推断是："大锤式做法表明这是一个需要快速绕过、不关心对其他 GPU 或安全性的附带损害的外部开发者…… 内部 NVIDIA 工程师或持有泄露文档的人更可能做外科手术式的改动。"
3. **伪装命名。** `SEC2_DEBUG_PRI_*`、`kgspSec2PostblTiming*` 与 `SEC2_DEBUG` 日志前缀在 NVIDIA 代码库中不存在，而 "PostBL Timing" 是一个听起来合理但虚构的功能名。这套方案读起来像试图把漏洞利用代码伪装成合法的制造或调试功能，而"这对拥有合法访问权的人来说没有必要。" 这些命名逐字出现在正式代码中，`0002-booter-verify.patch` 定义了 `SEC2_DEBUG_PRI_FEATURE_OVERRIDE_PLM 0x00823804`、`SEC2_DEBUG_PRI_FEATURE_OVERRIDE_SM_SPEED 0x0082381c`、`SEC2_DEBUG_PRI_FEATURE_OVERRIDE_SM_SPEED_1 0x00823820`、`SEC2_DEBUG_PRI_FBPA_CFG1 0x009a0204` 与 `SEC2_DEBUG_PRI_MMU_LMR 0x00100ce0`。

> [!CAUTION]
> **`gpuValidateRegOps` 绕过不在正式工具中**
>
> 该无条件绕过是真实且严重的：任何拥有 `NV_GPU_REG_OP` 访问权的进程都可以在机器内的任何 NVIDIA GPU 上读写任意寄存器。它**只存在于泄露的 `patch.diff` 中**。正式 `cmpunlocker` 补丁集完全不触碰 `subdevice_ctrl_gpu_regops.c`，字符串 `gpuValidateRegOps` 在 `master` 或十二个未发布分支中的任何一个都不出现。任何把此漏洞归给 `cmpunlocker` 的文字都是错的。

### 唯一的残余关切

两项被评为 HIGH 且未清除：完整 ROP 链 DMEM 字节偏移（`0x1100`、`0x5b40`、`0xf754` 至 `0xf7f8`），以及把 `writeAddr`/`writeValue` 经由 DMEM 栈槽映射的 gadget 链——被描述为 `_kgspSec2PostblTimingFillPayload` 在 `0xf800` 字节签名缓冲内的精确字节偏移处写入 **24 个特定的 32 位值**。

理由有四条：

| # | 论点 |
|---|---|
| (a) | 论文描述了统一填充 ROP 链的概念，但未发布版本特定的 DMEM 布局 |
| (b) | 这些偏移是 booter 版本特定的，弄错会得到崩溃（`MB0=0x31`、`IMEM_MISS_INS` 或金丝雀失败）而不是可用的漏洞利用 |
| (c) | 社区当时探索的据信是另一个偏移范围 |
| (d) | 推导它们需要一个周期精确的 Falcon 模拟器加上特定的 booter 二进制、NVIDIA 关于栈帧布局的内部文档，或泄露的 booter 源码 |

评估自己声明的缓解措施是：论文的模拟器方法论足以复现该分析。底线，逐字引用："ROP 链偏移是唯一需要在没有泄露文档的情况下做大量独立工作才能产出的元素，而论文明确描述了如何做这项工作。"

计数是对的：正式 payload 恰好包含 **24** 次 `_kgspSec2PostblTimingPutU32` 调用，位于 payload 偏移 `0x1100`、`0x5b40`、`0xf754`、`0xf758`、`0xf75c`、`0xf76c`、`0xf774`、`0xf780`、`0xf788`、`0xf78c`、`0xf790`、`0xf794`、`0xf798`、`0xf79c`、`0xf7a0`、`0xf7a4`、`0xf7b0`、`0xf7b8`、`0xf7c4`、`0xf7c8`、`0xf7d8`、`0xf7e0`、`0xf7f4`、`0xf7f8`。

---

## 后续分析对残余关切的结论

关切 (c) 与 (d) 站不住。以下更正来自重新阅读归档工件与正式源码，而非引自评估。

### (c) 是基址偏移框架伪影

payload 被 DMA 到 DMEM `0x800`，所以 **payload 偏移 + `0x800` = DMEM 地址**。这两个"不同"的范围是同一个范围：

| 补丁偏移 | DMEM 地址 | 评估自己表中的状态 |
|---|---|---|
| `0xf754` | `D[0xFF54]` | 标记 HIGH |
| `0xf76c` | `D[0xFF6C]` | 标记 HIGH |
| `0xf7c4` | `D[0xFFC4]` | 列为**干净但较早** |
| `0xf7f8` | `D[0xFFF8]` | 标记 HIGH |

干净但较早清单本就包含 `D[0xFFC4]` 至 `D[0xFFF0]`，这覆盖了评估标记为 HIGH 的 22 个栈槽中的四个。按评估自己的账目，它标记为 HIGH 的部分范围本就已被标记为干净。评估把补丁偏移引为可疑、把同一槽位的 DMEM 地址引为干净。

### (d) 被带日期的公开工件驳倒

正式 ROP 链中的每个代码地址都是净室自己解密的 debug-booter 反汇编中的一条指令边界，该反汇编发布在补丁之前**十七天**。

| 工件 | 发布时间 | 提供什么 |
|---|---|---|
| `booter_load_ga100_dbg_seccode.fuc5.asm`（545,149 B） | 2026-07-01T12:40:37Z | 原始 envydis 输出；每个链地址 `0x0cbd`、`0x0ccb`、`0x10aa`、`0x10b9`、`0x1fbd`、`0x582d`、`0x7f2f`、`0x815a`、`0x0d66`、`0x04d4` 恰好匹配一条指令行 |
| `...annotated.fuc5.asm` | 2026-07-03T17:12:52Z | 按函数横幅 |
| `...annotated.fuc5_v2.asm`（607,702 B，11,875 行） | 2026-07-09T03:03:21Z | 每条 `lcall` 带内联注释标明被调用者 |
| **寄存器 Gadget 图谱** | 2026-07-10T13:40:14Z | 由该反汇编机器生成。把 `0x0cbd` 列为"`$r10 <- $r0`，canary(r15==r9)，via-call，`mpopaddret $r3 0x4`"，把 `0x1fbd` 列为"`$r11 <- $r10`，canary(r15==r9)，via-call，`mpopaddret $r2 0x4`"，恰好是它们在正式链中扮演的角色，包括产生帧步长的 `mpopaddret` 收尾 |
| `cmpunlocker` 初始提交 `9b9fb2f`，`common/constants.yaml` | 2026-07-14T21:47:02-07:00 = 2026-07-15T04:47:02Z | `dmem_layout: dma_target 0x0800, payload_size 0xF800, guard_addr 0x6340, canary 0xFACEB13D`；`booter_addrs: bar0_write_gadget 0x10B9`；`payload_frames: frame_start_addr 0xFF48, frame_stride 0x18, frame_field_offsets {r0 0x00, r1 0x04, r2 0x08, r3 0x0C, saved_reg 0x10, return_addr 0x14}` |
| `ROP_CHAINS_1180f8_nibble_writeup_20260715.md` | 2026-07-15T18:48:10Z | 同一网格的文字版："通过轻量 `0x10b9` 自链做 N 次 BAR0-master 写，**每次写 +0x18 DMEM**"，列出 `D[0xFF50]`、`D[0xFF54]`、`D[0xFF5C]`、`D[0xFF68]`、`D[0xFF6C]`、`D[0xFF74]`、`D[0xFF80]`、`D[0xFF84]` |

**正式补丁中全部 22 个栈槽偏移都恰好落在该六字段、`0x18` 步长网格的某个命名字段上，零未对齐命中。** 24 个值中剩余的两个是守卫字（`0x5b40` 映射到 `D[0x6340]`）与 `0x1100`（映射到 `D[0x1900]`）。

其余的非 gadget 常量也有了解释：`0xf800`、`0x800`、`0x6340` 与 `0x4a7` 来自论文的模拟器 trace，在评估自己的"来自论文的干净内容"清单上；`0xc0deca7e` 是论文发布的守卫桩值；`0x5b40 = 0x6340 - 0x800` 是对论文印出的两个数字做算术；`D[0xFFB0]` 处的 `0x0000ffbc` 是一个自引用的 DMEM 栈指针，指向帧网格；`D[0xFF90]` 处的 `0x00008e18` 位于 booter 代码镜像之外（反汇编结束于 `0x86ff`），指向带注释清单在指令行 `0x0d39`、`0x0da1` 与 `0x0e1b` 处记载的寄存器描述符表区域 `0x8e04`/`0x8e08`。

### 净室自己的链相关但不是复制

净室 Python 解锁器与正式 C 链共享缓冲基址 `0x800`、大小 `0xF800`、守卫地址 `0x6340` 与 `0xFF48`/`0x18` 六字段帧网格。它们在两个地方明显不同：

| | 净室 Python（`payload/build.py`，提交 `9b9fb2f`） | 正式 C（`0001-sec2-postbl-plm-ss-cfg.patch`） |
|---|---|---|
| 金丝雀字面量 | `0xFACEB13D`（项目代号） | `0xc0deca7e`（论文发布的桩值） |
| 链形态 | 一个自链 gadget `0x10B9`，每次写一个帧 | 更长的链，经 `0x0cbd`、`0x1fbd`、`0x815a`、`0x582d` |
| 终结符 | `0x0000810D` | `0x00000ccb`（ACR mutex release）然后 `0x00007f2f`，正是公开记录中命名的 `multiwrite_then_mutexfree_cleanexit` 策略 |

漏洞利用的代号 **FACEB13D**，读作 "fake bird"，指的是必须被击败的栈守卫金丝雀，而不是 Falcon。被列举的障碍包括：通过隐蔽实现安全、栈金丝雀、安全级别 L0 至 L3、不可变启动 ROM、安全协处理器、代码的 AES 加密与代码的 RSA 签名。

---

## 泄露的概念验证

### 再分发包

在净室计算解锁器发布并克隆到 GitHub 后大约三天，一个"Chinese unlock"出现在俄语 Telegram 上。按当时的评估，它是泄露的私有概念验证，而非独立工作。

包结构，经多名独立审阅者检查：

```text
cmp170hx-unlock-610.43.03.zip
├── install.sh                              # 检查后评估为安全
├── NVIDIA-Linux-x86_64-610.43.03.run       # 与官方安装程序字节级相同
├── open-gpu-kernel-modules-610.43.03/      # 打过补丁的源码 + 预编译二进制
└── README.txt                              # 无关紧要
```

把随附源码与 `NVIDIA/open-gpu-kernel-modules` tag `610.43.03` 做 diff 产生 `patch.diff`，**35,867 字节、887 行、11 个文件**。每处修改都局限于开放内核模块组件；没有任何闭源二进制被改动。推荐的安全处理是删除随附的开放模块文件夹，`git clone` 上游，应用 `patch.diff`，重新编译，然后才运行 `install.sh`。

归档大小的报告不一致（一个同时给出内部 `.run` 为 461.5 MB 的账号报告 537.2 MB，另一个报告约 520 MB），一个来源还出现第二个文件名 `cmp170hx-unlock-610.43.03.tar.zst`。两个文件名可能都是真的，同一 payload 被再分发两次。

同时持有两件工件的人报告，diff 与私有驱动修改指南中的代码**逐词相同**。置信度：归档结构与 diff 大小**高**（多名审阅者，diff 文件本身已归档）；"泄露而非重新发现"的归因**中**，它只依赖争议归因一侧的一次字节比较。

关于再分发解锁器的另外两点，来自独立检查：它写入的权限级掩码表与公开仓库完全相同，不解锁任何额外功能，**不**启用 PCIe Gen2，且只识别 8 GB 卡（"currently this unlocker only supports 8G cards and can't recognize the 10G card"）。

### 泄露的 shell 脚本

另外，一个计算解锁 shell 脚本以 `CMP170HX_Compute_Unlock_v8_3.sh` 之名公开泄露，2026-07-14 发布到一个公开 GitHub 仓库并很快被删除。其作者描述它为"just the compute only logic that was posted here, with some minor modifications to attempt to run on multiple GPU's vs 1. Nothing new sadly"，实现方式是把注入块按卡复制、硬编码 PCIe ID。它不含任何显存解锁内容。

### 博客说法

一篇报道此事的中文博客声称两名黑客独立解锁了显存，并展示了一张团队 `booter_load` 代码的截图，其中的函数名与注释完全不同。不同的名字既与独立的反汇编加注释过程一致（净室自己的名字正是这样产生的），也与对复制材料的重新注释一致。

> [!NOTE]
> **未解决问题**
>
> 那张截图是否用公开提示独立解密，从未定案。下一步：把截图的指令地址与 `booter_load_ga100_dbg_seccode.fuc5.asm` 对比。如果它们匹配 debug 构建，作者就必须用公开的 Jetson 测试密钥解密它，那就是干净路径。

---

## 70 分钟采纳窗口

这是当时的评估不可能知道的部分，它由 git 作者时间戳、`diff -Naur` 头部 mtime 与解码的消息 snowflake 确立。

| 时间（UTC） | 事件 |
|---|---|
| 2026-07-18T18:01:15Z | `patch.diff` 发布到 `#general-how-to-cleanroom` |
| 2026-07-18T18:26:26Z | 正式 `cmpunlocker` 补丁集中的每个文件都带有该 `diff -Naur` 头部 mtime（`2026-07-18 11:26:26 -0700`）。一棵树，写于同一瞬间，发布后 25 分钟 |
| 2026-07-18T18:40:16Z | 出处评估发布，就在窗口中间 |
| 2026-07-18T19:11:01Z | `06fabf2 "WORKING MEMORY UNLOCK"` 在 `memory` 分支上作者时间，发布后 **70 分钟** |
| 2026-07-18T20:51:36Z | `6b7d9ee "FULL WORKING THING"` |
| 2026-07-18T21:46:49Z | `e4026e5 "Memory working!"` 合并到 `master` |

推导方向没有歧义：正式仓库在 `06fabf2` 之前**没有任何形式的驱动补丁**，而 `patch.diff` 只支持 8 GB `0x20C2` 卡。

### 两者实际差异

把归档 `patch.diff` 的每一行新增与 `driver/patches/0001` 至 `0006` 的拼接逐行对比：

| 度量 | 值 |
|---|---|
| `patch.diff` | 35,867 B，887 行，11 个文件 |
| `cmpunlocker` 补丁集 | 890 行，6 个补丁文件，10 个目标文件 |
| 两者间字节级相同的新增行 | **638** |
| 仅 `patch.diff` 有的行 | **19** |
| 仅 `cmpunlocker` 有的行 | **43** |

19 行 `patch.diff` 独有的行，每一行要么是 `cmpunlocker` 改为按 profile 处理的仅 8 GB 硬编码形式，要么是 `cmpunlocker` 附加了设备 ID 的日志行，要么是 `gpuValidateRegOps` 中的那一行 `+    return NV_OK;`：

```c
#define SEC2_POSTBL_TIMING_CMP_170HX_PCI_DEVICE_ID 0x20C2
NvU32 cfg1Value    = 0x02779000U;
NvU32 lmrValue     = 0x0000020BU;
NvU64 targetFbBytes = 0x0000001000000000ULL;  /* 64GB */
/* plus the devId == 0x20C2 guards */
```

43 行 `cmpunlocker` 独有的行，每一行都是 10 GB（`0x2082`）对应部分：拆分为 `SEC2_POSTBL_TIMING_CMP_170HX_8GB_PCI_DEVICE_ID 0x20C2` 与 `SEC2_POSTBL_TIMING_CMP_170HX_10GB_PCI_DEVICE_ID 0x2082`，`cfg1Value = 0x02669000U` / `lmrValue = 0x0000028AU` 分支，`targetFbBytes ... : 0x0000000A00000000ULL`，以及双设备 ID 守卫。其他没有任何不同。这些几何值就是 [显存几何](../unlock/memory-geometry.md) 记载的标准值。

---

## 两种解读，未定案

证据与自己真实地张力共存，源集合无法解决。两种解读在此完整陈述，因为读者有权自己掂量。

=== "A 面：按净室自己的规则是干净的"

    正式 ROP payload 中的每个常量都可以从早于 `patch.diff` 的、带日期的公开或净室衍生材料独立推导：gadget 地址来自 2026-07-01 发布的反汇编，gadget 语义来自 2026-07-10 发布的图谱，帧网格来自 2026-07-15T04:47Z 的公开 git 提交，缓冲、守卫、填充与大小常量来自 2026 年 6 月的论文。净室此前已在同一网格上独立构建了一条可用的 ROP 链，使用相同的缓冲、相同的守卫地址与相同的 `reg_write_indirect` BAR0 写原语（在 `0x10b9` 进入，正式链在 `0x10aa` 进入），并在四天前公开发货。按此解读，净室满足了自己的规则（"秘密知识只有在能证明同一信息可从公开来源推导时方可接受"），评估的否定结论正确，甚至说得保守了。

=== "B 面：不干净"

    正式代码不是对 `patch.diff` 的净室重实现。它**就是** `patch.diff`，在它出现 70 分钟后逐字采纳，移除一处恶意改动、添加一个设备 ID 分支。原则上可推导不是事实上已推导，而且按几位参与者对净室规则的理解（"it is dirty, 100%"），无论其中的信息是否可独立获得，该工件都被禁止使用。材料是被清除而非裁决的，而工具还是采纳了那段代码。

**什么能定案。** 现有材料里没有。两份私有驱动修改指南能确立 `patch.diff` 是否真的是私人小组的代码，采纳它的维护者的一纸声明能确立代码是被复制还是趋同写成。这两者在源集合中都不存在。

> [!NOTE]
> **未解决问题**
>
> 同一问题的一个更窄、可操作的版本：补丁 `0004`（BAR0 PRAMIN clamp）与 `0005`（CE scrub 变通方案）是 `cmpunlocker` 原创，还是也在私有指南中？它们的内容在 `patch.diff` 与 `cmpunlocker` 之间字节级相同，所以是一起进来的，但社区对 `patch.diff` 的总结只描述了签名劫持、PLM 打开、寄存器 poke、签名重建、`fb_length` 伪造与 late-PMA 扩展。它们没有提到 PRAMIN clamp 或 CE scrub 变通方案。只有指南能定案。

---

## 学术论文及其披露立场

预印本是这项工作唯一指定的干净输入及其方法论基础。其摘要声明 CMP 170HX"与旗舰 A100 同一颗 die，但在三个商业轴线上被熔丝阉割：SM 数学速率（节流到 1/32）、显存容量（10 GB 而非 80 GB）与 PCIe 链路（Gen1 而非 Gen4）"，"三项限制都是软的"，并给出"约 31-62 倍计算、8 倍容量、2 倍链路"的核心收益。它与 `arXiv:2505.03782` 是不同的论文，后者被引为参考文献 [13]。

作者刻意拒绝了发布前的 embargo。第 10 节记录：工作仅在单卡实验室进行，无转售、无持久硅片改动、未提取签名密钥、未伪造签名，测量后卡被恢复到原生配置。厂商的产品安全团队是**与发布同时**获知，而非提前。声明的理由在此如实记录为作者的主张，而非背书：

> 协调披露假定厂商的补救能保护用户，而在防御者是设备、攻击者是设备拥有者的倒置威胁模型中，这一假定不成立。私有 embargo 窗口会让厂商在已出货硬件上烧掉相关的防回滚熔丝，在这些用户有机会得知或行动之前，永久移除这项能力——而这项能力正是本工作所关心的。

论文还描述了作者基于 booter 指令流构建的静态检查器：它把 DMA 即复制的摘要提升到 IR，把 DMA 视为污点源，在 DMA 汇点应用有界写检查（`L <= S - o`），并在考虑 link-map 感知的布局相邻性时升级。作为差分门运行，它把开放内核时代 booter 的签名读取传输标记为其唯一无界汇点，并零误报通过较旧的 booter 家族。置信度：**中**。该检查器未包含在归档材料中，也无人独立复现。

给实现者的实际脚注：论文"3-4 处 BAR0 值改动"的表述误导了每一位独立复现者。那三四次写是琐碎的；全部难点在于先打开四个 PLM。参见 [权限级掩码](../unlock/privilege-level-masks.md)。

---

## 一个下游法律事件

**NVIDIA 于 2026-07-17 对至少一个 `cmpunlocker` fork 发出 DMCA 下架通知**，使该仓库离线。接收方称通知直接来自 NVIDIA，并停止了该项目上的公开工作。其他人猜测这是自动化过滤触发的执法，并指出已经存在许多 fork；流传的建议是改名并重写 fork。置信度：**中**。该报告是一手报告，仓库可观察地离线，但源集合中没有下架文件。此处内容均非法律建议，本 wiki 对是非曲直不持立场。

---

## 面向本 wiki 读者的出处卫生

直接源自上述记录的三条警示。

> [!CAUTION]
> **不要引用项目的 `docs` 分支**
>
> `docs/ARCHITECTURE.md` 声称 `cmpunlocker` 向 SS0 与 SS1 都写入 `0xffffffff`。正式补丁写的是 `0x0082381c = 0x88888888` 与 `0x00823820 = 0x00000008`。同一分支编造了在代码或记录中无处可寻的缩写展开（SS 为 "Suspension State"，PLM 为 "Program Logic Modules"，PMM 为 "Permute Mask Model"，LMR 为 "LM (Local Memory) Request register"，PMA 为 "Power Management Array"），断言一条不存在的 `SEC2_DEBUG: Executing unlock sequence...` 日志行，并指示用户运行 `sudo ./uninstall.sh --yes`——而正式脚本是 `remove.sh`。它有七个提交，不是权威。

> [!WARNING]
> **流传最广的架构笔记自评约 10% 已证实**
>
> 其作者发布时附带警示："I do hold some notes. I try to double-check each statement, but this work can not be given to LLMs, so it is goes REALLY slow. This is what I have now. I do not state that this information is accurate, I would say, just ~10% has reliable proofs/sources." 对任何试图做综合写稿的人也给出并行警告："most of things known about throttling mechanism are based on hypotheses and some experiments that do not contradict them... if you simply collect all points mentioned in chat you will likely get many wrong conclusions and it will get your llm insane." 被点名作为可靠启蒙材料的三个来源是 Zenodo 论文、公开 GA100 熔丝参考表与带注释的 `booter_load` 汇编。请把这条警示专门挂在架构笔记上，而不是寄存器 dump 或反汇编上——后两者的支撑明显更好。

项目各处使用的函数名是从行为推断的，不是从符号表读出的：该二进制没有符号。有一对名字在文档间不一致地命名。`0xd66` 与 `0xccb` 在 LLM 概览中是 `regtable_reverse_lookup` 与 `regtable_rw_indexed`，但在 ROP 文档中是 ACR mutex 的获取与释放。代码支持 mutex 解读，正式链把 `0x00000ccb` 放在 `D[0xFFF4]`、紧随其干净退出 `0x00007f2f` 之前，所以 mutex 解读才是正式代码所依赖的。

---

## 带日期工件索引

本页所有携带解码时间戳的内容，按顺序。

| 日期与时间（UTC） | 工件或事件 |
|---|---|
| 2026-05-05 / 2026-05-07 | 两张物理 CMP 170HX 10 GB 卡被探测（每张 120 个寄存器） |
| 2026-05-31 | Drive A100 32 GB（PG199）被探测；15 卡熔丝参考表完成 |
| 2026-06-26 | Canary 预印本在解锁器服务器传阅 |
| 2026-06-27 | 净室规则集作为频道政策声明 |
| 2026-06-30 | 公开 AES-128-ECB 测试密钥与 `rijndael-tool.zip` 在频道内发布 |
| 2026-07-01T12:40:37Z | 原始 debug booter 反汇编发布（545,149 B） |
| 2026-07-02 | debug 对生产等价性定案；寄存器出处标准被接受 |
| 2026-07-03T17:12:52Z | 带注释的反汇编发布 |
| 2026-07-09T03:03:21Z | 带注释的反汇编 v2 发布（607,702 B，11,875 行） |
| 2026-07-10T13:40:14Z | 寄存器 Gadget 图谱发布 |
| 2026-07-14T21:47:02-07:00 | `cmpunlocker` 初始提交 `9b9fb2f`，携带帧网格常量 |
| 2026-07-15T18:48:10Z | `ROP_CHAINS_1180f8` 文档，记录 `每次写 +0x18 DMEM` |
| 2026-07-16T06:07:12Z | 论文以 `main.pdf` 发布到净室服务器 |
| 2026-07-17 | 对至少一个 fork 的 DMCA 下架 |
| 2026-07-18T18:01:15Z | `patch.diff` 发布 |
| 2026-07-18T18:26:26Z | 正式补丁集文件 mtime |
| 2026-07-18T18:40:16Z | LAPSUS$ 出处评估发布 |
| 2026-07-18T19:11:01Z | `06fabf2 "WORKING MEMORY UNLOCK"` |
| 2026-07-18T21:46:49Z | `e4026e5 "Memory working!"` 合并到 `master` |

---

## 参见

- [项目时间线](timeline.md)，完整的带日期序列，含技术里程碑
- [工具谱系](tool-lineage.md)，哪些工具取代了哪些，哪些已死
- [死路](dead-ends.md)，被尝试并被驳倒的路线
- [ROP 链](../unlock/rop-chain.md)，本页所涉出处的 payload
- [六个驱动补丁](../unlock/driver-patches.md)
- [Falcon 与 Booter](../unlock/falcon-and-booter.md)
- [熔丝与 OTP](../hardware/fuses-and-otp.md)，120 寄存器差分语料
- [方法论](../appendix/methodology.md) 与 [外部来源](../appendix/external-sources.md)
