# 外部来源

**本页涵盖：** 一份带注释的参考书目，列出本 wiki 倚重的每一个外部参考：`cmpunlocker` 仓库及其未发布分支、上游 NVIDIA 开放内核模块、170th Street 社区 wiki、独立拆解评测、TechPowerUp VBIOS 与规格数据库、学术论文、envytools 和 Falcon 工具链家族，以及社区 fork、gist 和 issue 线程。每个条目说明该来源是什么、**该信任到什么程度**，因为这里几个被广泛引用的来源在特定、可识别的方式上是错的。

商业列表、市场、厂商产品页和任何与采购相关的东西都在范围之外，刻意缺席。

## 如何读信任列

| 评级 | 含义 |
|---|---|
| **主来源（Primary）** | 你可以自己从工件重新推导该断言。源码、一份签名测量，或一个你可以做哈希的文件。 |
| **可靠（Reliable）** | 独立、第一手，并且至少被印证过一次。可自由引用，但要说清它是什么。 |
| **谨慎使用（Use with care）** | 确实有用，但带有已知的具体缺陷。不核对下面的缺陷清单就不要引用。 |
| **不可引用（Do not cite）** | 已知包含自信但错误的技术内容。只作为历史记录有用。 |

---

## 1. 解锁实现

### `github.com/amoghmunikote/cmpunlocker`（`master` 分支）

**信任：主来源。** 这是发布版工具，也是任何可用代码表达之事的权威。标语：「A tool to unlobotomize your NVIDIA card!」。2026-07-14 公开；首个提交 `9b9fb2f Initial commit`；归档的 `master` 顶端是 `cc872cb Moved PR template location`（2026-07-23）。

`master` 恰好包含八个顶层条目：`.github/pull_request_template.md`、`.gitignore`、`LICENSE`、`README.md`、`common/constants.yaml`、`driver/`、`install.sh`、`remove.sh`。**没有** `verify.sh`、**没有** `tools/` 目录、**没有** `probe.sh`、**没有** `requirements.txt`（2026-07-19 删除）也没有测试套件。卸载器是 `remove.sh --yes`；**`uninstall.sh` 在整棵树中任何地方都不存在**。

`common/constants.yaml` 是机器可读的基准真值，与补丁 `0001` 完全一致。注意一个读 README 不会知道的重要行为：在当前 `master` 上，`--profile` 不再选择几何。补丁 `0001` 在 GSP 启动时按 `pGpu->idInfo.PCIDeviceID >> 16` 分支，`build.sh` 的内联重写找到全部六个标记并无编辑地退出，`--profile` 只影响横幅、`EXPECTED_MIB` 和元数据文件。2026-07-18 之前的说明在这点上已经过时。见[驱动补丁](../unlock/driver-patches.md)和[安装](../procedures/install.md)。

> [!WARNING]
> **README 对设备门槛的说法很松**
>
> 它说解锁以 `0x20C2` 为门槛，而驱动内门槛 `_kgspSec2PostblTimingEnabled()` 接受 `0x20C2` **和** `0x2082`。Master 完全没有 `DEBUGGING.md`：那行「所有 PLM 必须显示 `0xffffffff`」活在 `docs` 分支上，而且它是错的，因为发布版表把 WPR_CFG `0x001fa7cc` 打开到 `0xfffff0ff`。

### 十二个未发布分支

**信任：作为代码是主来源，作为建议是实验性。** 真实代码、未合并、若干处内部自相矛盾。恰好有 **12** 个未发布分支快照（算上 `master` 共 13 棵树）：`80`、`Gen2`、`PG199`、`clanker/driver-port`、`debug-gen2`、`deced`、`docs`、`ecc`、`far`、`housekeeping`、`memory`、`multiple-cards`。任何声称十三或十四个*快照*的来源都在数错。注意该仓库在抓取时带有 **17 个分支引用**，所以有四个未发布引用从未被快照、本站任何地方也没有分析它们：`code-simplification`、`dual-geometry-fix`、`fix` 和 `v0.1`。

| 分支 | 顶端 | 是什么 | 信任说明 |
|---|---|---|---|
| `multiple-cards` | `b1cb6d8`，2026-07-18 | 按设备 ID 的 profile、一个 `mixed` profile、`gpu_inventory`，以及仅分支独有的 `verify.sh` | 自成一体，最可能被合并。其 `verify.sh` 的 lspci 回退会静默丢弃 `10de:20b0`。 |
| `debug-gen2` -> `Gen2` -> `far` -> `deced` | `746d9f7` -> `a4de322` -> `8854d3e` -> `2326599` | PCIe Gen2 谱系。四个分支上都有 `0007-pcie-gen2.patch`；`0008-pcie-gen2-probe-retrain.patch` 从 `Gen2` 起存在。`deced`（2026-07-27）最新。 | 见下面的危险说明。 |
| `clanker/driver-port` | `153cd6d`，2026-07-21 | 按分支的补丁目录 `{580,590,595,610}/`。`driver/VERSION` 列出**十二**个版本，但 `constants.yaml` 只列**五个**：一个公认的内部不一致。其 `install.sh` 与 master 的逐字节相同。 | `610` 目录是 master 的逐字节副本。**595、590 或 580 上从未报告过任何启动。** |
| `80` | `3c53aca`，2026-07-19 | 10 GB 卡的 80 GB 尝试 | 见下面的危险说明。 |
| `ecc` | `bb4d669`，2026-07-18 | 单个提交，「Fixed dual geometry support」 | **不包含任何 ECC 代码。** 名字误导。 |
| `housekeeping`、`memory` | 2026-07-18 | 中间开发状态 | `housekeeping` 的补丁本来打不上：加入 `0x2082` 分支时没有更新 `@@` hunk 计数。 |
| `PG199` | | Drive A100 快照 | 仅作参考。 |
| `docs` | `651b6d5`，2026-07-27 | 散文文档 | **不可引用。** 见下。 |

> [!CAUTION]
> **两个会让你付出代价的分支缺陷**
>
> **`Gen2` 安装了一个 Gen1 钳制。** `debug-gen2` 和 `Gen2` 把 `NVreg_RegistryDwords="RmForceEnableGen2=1;RMPcieLinkSpeed=0x1"` 写进 `/etc/modprobe.d/cmp-pcie-gen2.conf`，在试图启用 Gen2 的同时把链路钉在 Gen1。`far` 提交 `8854d3e "Remove clamp link to Gen1"` 把它改成 `0x2`。哪个值正确真正未定案：两者都发布了，且不存在 A/B 启动测试。
>
> **`80` 分支编程的不是它的元数据所说的。** `80/common/constants.yaml` 携带 `lmr: "0x0000028B"` 和 `81920`，但 `build.sh` 从不读取该文件。`80/driver/build.sh` 第 93 行设置 `LMR="0x0000028A"`，`install.sh` 第 138 行打印 `CFG1=0x02779000 LMR=0x0000028A`，补丁 `0001` 第 144 行固化 `lmrValue = 0x0000028AU`。提交 `3c53aca "Correct LMR for 80GB"` 只改了惰性元数据。每个运行过该分支的测试者编程的都是 CFG1 `0x02779000`、LMR `0x0000028A` 和 `fb_length 0x0000001400000000`，这个三方分歧是该分支在恰好 40 GiB 处折叠的最佳解释。该分支没有任何一次构建携带过自洽值，尽管一个净室脚本触发过它。见[80 GB 问题](../frontier/80gb.md)。

### `github.com/amoghmunikote/cmpunlocker` 的 `docs` 分支

**信任：不可引用。** 七个提交。它是项目自己的文档分支，也是记录在案的错误来源：`docs/ARCHITECTURE.md` 声称 `SS0 = 0xffffffff` 和 `SS1 = 0xffffffff`，而发布版补丁写的是 `0x88888888` 和 `0x00000008`；`DEBUGGING.md` 说所有 PLM 必须读 `0xffffffff`；`docs/INSTALLATION.md` 和分支 README 都指示运行 `sudo ./uninstall.sh --yes`，而那个文件不存在；它还发明了代码和聊天中都不存在的缩写全称（SS 为「Suspension State」、PLM 为「Program Logic Modules」、PMM 为「Permute Mask Model」、LMR 为「LM (Local Memory) Request register」、PMA 为「Power Management Array」）。它还断言驱动会发出 `SEC2_DEBUG: Executing unlock sequence...` 日志行，而驱动从不发出。

### `github.com/NVIDIA/open-gpu-kernel-modules`

**信任：主来源。** 上游驱动源码，`build.sh` 在安装时抓取它（`archive/refs/tags/${VERSION}.tar.gz`），也是签名 booter blob 的来源。有三个文件反复重要：`src/nvidia/generated/g_bindata_kgspGetBinArchiveBooterLoadUcode_GA100.c`（持有 `IMAGE_{DBG,PROD}`、`HEADER_{DBG,PROD}`、`SIG_{DBG,PROD}` 和 `PATCH_LOC = 0x8900`）、`kernel_gsp_booter_tu102.c` 和 `nouveau/extract-firmware-nouveau.txt`。发布版 `master` 支持的版本恰好是 `610.43.03`（默认）和 `610.43.02`；其余任何版本构建硬失败。见[驱动版本](../procedures/driver-versions.md)。

> [!NOTE]
> **下载没有完整性检查**
>
> `build.sh` 用 `curl -L --fail` 抓取 tarball 并缓存，整棵树中没有任何校验和或签名验证。为每个版本记录一个期望 SHA-256 是一行改进，至今未做。

---

## 2. 社区文档

### 170th Street（`170th-street.gitbook.io/hx`）

**信任：谨慎使用。** 这张卡最大的社区 wiki，截至 2026-07-27 也是项目自己指定的文档站点。其结构覆盖硬件（完整规格、拆解指南、一份泄露的 NVIDIA A100 原理图页面）、改装（PCIe 电容改装、水冷）、解锁、AI 和 ML 工作负载、基准，以及一个 NVLink 布件研究页。它运行基于 issue 的贡献流程，其 issue tracker 线程 #1 持有一场可观的早期研究讨论。

电容改装页是该流程的社区参考级报告，并被独立测量印证，所以是安全的。问题在别处：

- **它在 SM 数量上自相矛盾。** `hardware/full-specifications.md` 给出一张计算表：70 SM、4,480 CUDA 核心、280 张量核心，而它自己的「规格差异说明」一节说「8 GB 变体：56 SM、4,096 位显存总线」和「10 GB 变体：70 SM、5,120 位显存总线」，`introduction/what-is-the-cmp-170hx.md` 重复了这种分裂。在一张在线 8 GB 卡上用 PTX 特殊寄存器转储测出的值是 **70 SM**。
- **其 PCIe 页过时了。** `hardware/full-specifications.md` 仍写「PCIe Gen 1 x4（固件锁定），约 1 GB/s」，它自己的时间线页仍把电容改装描述为未确认。两者都已被超越。
- **其 FP16 对比混用了标量与张量速率**，拿 170HX 的标量性能去比其他卡的张量核心性能，这一点在频道内被指出过。
- `cmpunlocker` 维护者被问到 LnkCap 和 LnkCap2 值是否被证明来自熔丝时，明确宣布它过时且不可信。

把它当作一份组织良好的二手摘要。任何寄存器、熔丝或规格断言都要对照[寄存器参考](../unlock/register-reference.md)或一次测量重新核实。

### 独立拆解评测（`niconiconi.neocities.org/tech-notes/nvidia-cmp-170hx-review/`）

**信任：可靠，也是本参考书目中最好的物理观察来源。** 发布于 2023-10-25，比解锁工作早两年半，因此完全不被它污染。其关键发现被频繁引用、从未被反驳：CMP 170HX 使用的电路板与 A100 40 GiB 几乎甚至完全相同，唯一区别是 ASIC 型号 `GA100-105F-A1`，且板上有很多未装元件，包括省略的 VRM 相（移除的 DrMOS 晶体管及其输出电感）和**缺失的 NVLink 相关 IC**。

最后一点比任何寄存器值都重要：缺失的 NVLink 接口 IC 是 NVLink 解锁的*物理*障碍，与固件里的任何东西无关。见[NVLink](../frontier/nvlink.md)。该评测也是同一作者 FMA 禁用工作的来源，而那是这张卡整个软件侧故事的起点。

评测的配套拆解照片托管在同一个域名，是语料中质量最高的 PCB 图像。

---

## 3. 规格与 VBIOS 数据库

### TechPowerUp VBIOS 收藏

**信任：ROM 文件是主来源，不要信任元数据列。** `.rom` 镜像是真的、可哈希的；TechPowerUp 自己对这些条目的「Memory Size」列无法追溯到文件内的任何字段，不可靠。

收藏中有四张 CMP 170HX 镜像：

| 条目 | 版本 | 构建 | 设备 / 子系统 | 标签 | 实际 |
|---|---|---|---|---|---|
| 257744 | `92.00.67.00.01` | 2021-05-14 | `10DE 20C2` / `10DE 1585` | 8 GB | 原厂量产 8 GB 镜像，显存字段 364 MHz，250 W |
| 239457 | `92.00.67.00.01` | 2021-05-14 | `10DE 20C2` / `10DE 1585` | 「16 GB」 | **除 `flash_status_ledger` 外与 8 GB 镜像逐位相同**——后者每次刷写都会变，包括在工厂。16 GB 标签是错的。 |
| 268495 | `92.00.6D.00.0A` | 2022-04-07 | `10DE 20C2` / `10DE 1585` | 「0 GB」 | **300 W** ROM：显存字段 432 MHz，板功耗目标 250.0 W、上限 300.0 W、调节范围 -60% / +20%，MD5 `a58aae86e72b13d50603c15653350664`。0 GB 标签是错的。 |
| 268984 | `92.00.66.00.02` | 2021-04-23 | `10DE 2082` / `10DE 1557` | 10 GB | 10 GB 镜像 |

> [!CAUTION]
> **「16 GB」和「0 GB」镜像都不会解锁显存**
>
> 它们只差在功耗和频率字段。把 239457 刷到 10 GB 卡上产生黄色感叹号且驱动不接受，因为设备 ID 不匹配。把 8 GB VBIOS 刷到 10 GB 卡上让卡无法启动。第三个修订 `92.00.6D.00.09`、日期 2021-11-01 在野外存在，但不在 TechPowerUp 收藏里：它已经带 300 W 上限但没有显存超频。**VBIOS 版本对解锁是否奏效没有影响**，这一点在两张宿主上运行 `92.00.67` 和 `92.00.6D.00.0A` 的四张卡上得到确认。见[VBIOS](../hardware/vbios.md)。

同一收藏中有用的对比条目：A100 PCIe 40 GB（277449）、A100（283106）、A30（262595，其 `92.00.66.00.0x` 与 10 GB 170HX 镜像几乎相同）和 Tesla V100 16 GB（199146）。

### TechPowerUp GPU 规格数据库

**信任：谨慎使用。** 这张卡的正确条目是 `gpu-specs/cmp-170hx-8-gb.c3830`。

> [!CAUTION]
> **`c3824` URL 是个陷阱**
>
> `gpu-specs/cmp-170hx.c3824` 返回 HTTP 200 并重定向到 `/gpu-specs/radeon-pro-w6800x-duo.c3824`，一个 AMD 产品页。它流传甚广，包括进入过一份 agent 简报。相邻 ID 供参照：`c3821` 是 A100 PCIe 80 GB，`c3822` 是 CMP 70HX，`c3823` 是 PG506-242。

TechPowerUp 在裸片尺寸（826 mm²）、着色单元（4,480 = 70 x 64）、TMU/ROP/张量核心（280/128/280）和 L1（每 SM 192 KB）上可靠。它在要紧的地方**错两次**：它列出 **8 MB L2**，而 deviceQuery 和一项独立延迟尖峰微基准都测到 **32 MB**；它把供电接口描述为「2x 8-pin」，而板上是**一个 EPS 8-pin**、承载两条逻辑 12 V 轨。其 PG199 6144 位总线条目也被一位第一手拥有者指出错误。其 CMP 部件的带宽数字在频道内被指出偶尔出错。

---

## 4. 学术与正式出版物

### 「A Canary in the Crypto Mine: Defeating Stack Protection in a GPU Secure Coprocessor」

**信任：主来源，也是整个净室努力指定的唯一干净输入。** 2026 年 6 月，16 页，Zenodo 记录 **20916112**，镜像为 ResearchGate 出版物 **408132536**。2026-06-26 在解锁器服务器流传，2026-07-16T06:07:12Z 贴进净室服务器。

其摘要称 CMP 170HX 是「与旗舰 A100 相同的裸片，但在三个商用轴上被熔丝残废：SM 数学速率（限到 1/32）、显存容量（10 GB 而非 80 GB）和 PCIe 链路（Gen1 而非 Gen4）」，称「三个上限都是软的」，并报告大约 31 到 62 倍计算、8 倍容量和 2 倍链路的头条提升。

它为什么承重：净室规则指定它为唯一可接纳的输入文档，理由是它发布在科学出版物网站上、且已发给厂商。其第 5.5 节模拟器迹线发布 `buffer = 0x800`、`SIGSZ = 0xf800`、均匀填充 `V = 0x4a7`、`guard@0x6340` 和防护桩值 `0xc0deca7e`，发布版 payload 大量常量正是来自这里。其第 8.5 节「Persistence across FLR」是关于覆盖值位于常开岛如何把一次性利用变成持久状态的论证。

两点告诫。其「3-4 处 BAR0 值变化」的框架误导了每一位独立实现者：难度全在先打开四个 PLM，之后的 BAR0 写入微不足道。而且它的 Falcon 模拟器**从未发布**，这关闭了复现其分析的最直接路径。二手报告说论文把卡稳定在大约 35% 吞吐惩罚上，这里以低置信度记录、未核实。

论文作者拒绝了发布前禁运，在第 10 节论证协调披露假定厂商的补救能保护用户，而当防御者是设备、攻击者是它的拥有者时，这个假定不成立。

### arXiv:2505.03782

**信任：可靠，且经常与上文混淆。** 「Exploration of Cryptocurrency Mining-Specific GPUs in AI Applications: A Case Study of CMP 170HX」，2025 年 4 月 30 日提交，分类 cs.AR 和 cs.DC。它报告通过关闭 CUDA 源码中的 FMA 收缩，在原厂固件上用 OpenCL 基准、mixbench 和一个 LLAMA 基准测量，FP32 超过原能力的 **15 倍**、特定精度下 LLM 推理超过 **3 倍**。它是 Canary 论文的参考文献 [13]。**它不是利用论文**，有一段时间社区把两者混为一谈。

围绕这张卡积累的其他 Zenodo 记录：18994970、19002983 和 18995979（一份 170HX 张量核心分析，据报因分类、风险和术语被 arXiv 拒绝）。

---

## 5. Falcon 逆向工程工具链

### envytools / envydis（`envytools.readthedocs.io`、`github.com/envytools/envytools`）

**信任：在其覆盖范围内可靠，对其不覆盖的保持沉默。** 带 **`fuc5`** 目标的 `envydis` 成功反汇编 GA100 booter，生成的清单被独立审查并在硅片上正确执行。这成立，尽管 envytools 表名义上把 `fuc6` 分配给 GP102 及以后的部件（`fuc0 [G98, MCP77, MCP79]`、`fuc3 [GT215+]`、`fuc4 [GF119+]`、`fuc5 [GK208+]`、`fuc6 [GP102+, 仅选定引擎]`）。170HX SEC2 正式算 fuc5 还是 fuc6 仍未定案；频道内记录的实际回答是「我挑了能用的那个」。

> [!NOTE]
> **开放问题**
>
> envytools 大约八年没有更新，而且**完全无法印证安全启动材料**：其 Falcon 加密页有标题无内容，只记录到 v5 的 Falcon 硬件版本，并且没有这项工作依赖的若干寄存器条目。`gitlab.freedesktop.org/nouveau/envyhooks` 上的 `envyhooks` 被建议为继任者，但被发现缺少等价功能。定案 fuc5 对 fuc6 需要对同一镜像的两种解码做 diff，寻找只有一种目标能自洽解析的指令。

同一家族里还有：

- **`github.com/vbe0201/faucon`**：一个 Falcon 模拟器，明确仅 fuc5。其 `faucon-emu/src/cpu/instructions/data.rs` 被用作指令语义参考。
- **`github.com/CAmadeus/falcon-tools`**（`requiem` 子树）：Falcon 安全启动工具、keygen、payload 和逆向工程材料。需要 Python 3.6+、PyCryptodome、envytools、make 和 m4，且不直接针对相关世代的任何 NVIDIA GPU。
- **`github.com/karolherbst/nouveau_tools`**（`dbg_falcon.sh`）：一个 Falcon 调试辅助。
- **`hexkyz.blogspot.com`**（「Je ne sais quoi: Falcons over the Horizon」，2021 年 11 月）和 **switchbrew TSEC 页**：Falcon 安全模式行为的标准外部参考，包括 `$sr10` 语义和停机前抑制中断与异常的位。
- **`github.com/ttabi/extract-firmware-nova`** 和 **`github.com/NVIDIA/nova`**（`drivers/gpu/nova-core/devinit.rs`、`vbios.rs`）：NVIDIA 内核驱动的 Rust 重写，有用是因为它在普通源码中点名寄存器，而 C 驱动把它们藏在宏后面。

---

## 6. 社区 gist 与参考表

**信任：作为测量记录是主来源。** 两个重要 gist 发布后都被删除、又被他人重新 fork，所以引用内容，不要引用某个特定 fork。

| Gist ID | 内容 | 为什么重要 |
|---|---|---|
| `0480d2b2b35ad594e57b6543952be307` | **GA100 熔丝与寄存器参考表**（约 50 kB）加 `probe.sh`（约 19 kB） | 净室的差分语料：15 张 Ampere 卡上读取的 120 个寄存器（2 张物理 170HX 10 GB、11 张云端租用、2 张物理 Drive A100 32 GB）。确立恰好有**五**组寄存器把 170HX 与同一硅片的 A100 区分开：SM 速率选择、PCIe 启动代际、NVLink 禁用、ECC 启用和 FBPA CFG1 几何。还确立两块物理 170HX 在 **120 个寄存器中的 107 个**上一致，全部 13 处差异都是逐裸片分档工件，这正是解锁配方能在卡间移植的原因。 |
| `84cd3921788d2ffbc1e9bf8b6f2c9396` | **GA100 VBIOS 对比表**（约 27 kB）加 `z1_dump_and_parse_vbios.sh` 和 `z2_parse_vbios_table.py` | 静态解析七份 ROM，CFG1 strap 表由启发式定位，显存训练条目被解码。转储脚本对 flash 是只读的：不存在写路径。 |
| `da...`（A100 对比）、`dafea7b6663c13edc28b33872f6e51be` | 补充 VBIOS 对比材料 | 次要。 |

> [!WARNING]
> **VBIOS 解析器带有过时标签**
>
> `z2_parse_vbios_table.py` 的 docstring 与它自己的输出矛盾。它声称 A100 PCIe strap 表位于约 `0x3FB18`，而对比表把它放在 `0x4285A`。它把 RFRD 标为「power table」，而 RFRD 是镜像布局描述符，其 `field_0C` 是 MAC 验证的范围大小，不是功耗上限。其 FBPA tier 提取器搜索 CFG1 表周围的窗口，若没有别的合格就会匹配到 CFG1 表本身。任何人逐字使用其输出标签都会传播这一切。

---

## 7. Fork、重实现与相邻工具

发布后数日内至少有六个公开仓库 fork 或重实现了解锁。没有一个对 `master` 有权威性。

| 仓库 | 是什么 | 信任 |
|---|---|---|
| `arabel1a/cmpunlocker`（2026-07-15） | 早期 fork | 历史 |
| 另外六个个人 fork 和重打包 | Fork 和重打包 | 历史。其中一个带 `combined-multiple-cards-gen2` 分支，是 Gen2 工作与多卡支持的一次可观的社区合并。按本 wiki 的匿名化政策省略所有者姓名。 |
| `asm64-hooligan/cmpunlocker` 的 `mem_overclock` 分支 | 显存超频实验，倍率从 72 降到 70 | 实验性，单一作者，已在频道内请求测试 |
| `theneocorp/cmppatcher` | 一种**不同方法**：直接给 NVIDIA 驱动**二进制**打补丁，使改动扛过驱动更新。报告了 3D 加速和 FP32 FMA 绕过。 | 独立，本站未核实 |
| `abobasixseven/unlock-cmp-170hx` | **不是报告。** 只含 `README.md` 和 `cmp90_compute_unlock_prompt.md`，都以 AI 代理执行指令结尾，如「EXECUTE STEP BY STEP: 5 -> 6 -> 6.5 -> 7」，并在整个备份和克隆命令中硬编码某位用户的 home 目录。 | 谨慎使用。其寄存器表与发布版补丁一致；其散文和 PCIe 章节是二手摘要，不是测量。 |
| `eastmoe/CMPGPU-patch-script`（`optimize-cmp-cuda.py`） | 交互式 llama.cpp 源码打补丁器，五个独立优化组，每组默认否：`fp32_fma_flag`（加 `-fmad=false`）、`fp32_fma_split`（把 `quantize.cu` 中的 `fmaf(...)` 重写为 `__fadd_rn(__fmul_rn(...))`）、`math_intrinsics`、`dp2a`、`fp16_bf16_cuda_core`。七个文件里十一条 PatchSpec 条目、`.cmp-bak` 备份、`--dry-run`/`--no-backup`/`--restore`。 | 可靠，且它自己的 README 警告在非 170HX 的 CC 8.x 设备上性能可能**下降**。 |
| `cachenetics/170tune` | 调优与合格化平台，安装为 `/usr/local/bin/170hx-oc`；测量、把关并恢复频率与电压设置，把「一次完成的基准」当作「什么也不算的证据」 | 方法上可靠。它是否跨重启持久化设置是其作者自己标记的开放问题。见[调优](../operations/tuning.md)。 |
| `Kepling5001/Miners`（`CMP170HX_Compute_Unlock_v8_3.sh`） | 一个计算解锁 shell 脚本，被公开泄露、又被迅速删除。其作者称它「只是计算逻辑……做了一些小改动想跑在多个 GPU 而不是 1 个上。没有新东西」 | 仅历史。不包含任何显存解锁内容。 |
| `arabel1a/ml-on-cmp`、`arabel1a/gpu-micro-bench` | 微基准仓库 | 对其发布的测量可靠 |
| `Highwayaiexpose/CMP-170hx-64gb-LLM-benchmarks` | 已解锁 64 GB 卡上的社区 LLM 基准收集 | 谨慎使用：按平台、单一来源 |
| `InnovativeOSS117/Gaming-on-A100` | GA100 上的图形工作 | 相邻；与显示输出和 3D 问题相关 |

### 第三方验证与测量工具

反复使用、值得认识：`ComputationalRadiationPhysics/cuda_memtest`（v1.2.3，维护者推荐的 VRAM 验证器，遇首个错误即退出，**在 80 GB 配置下无限挂起，除非上限 39 GB**）、`GpuZelenograd/memtest_vulkan`、`wilicc/gpu-burn`、`ProjectPhysX/OpenCL-Benchmark`、`ReinForce-II/mmapeak`（张量吞吐）、`zzc0721/torch-performance-test-data`（GEMM）、`sasha0552/nvidia-pstated`（空闲功耗管理；见其 issue #6），以及没有可调整大小 BAR 固件支持的宿主上的 `xCuri0/ReBarUEFI`。

---

## 8. Issue 线程与讨论足迹

**`github.com/dartraiden/NVIDIA-patcher` issue #73。** *信任：作为记录可靠，作为分析不可靠。* 这就是整个努力开始的地方，2026 年 3 月，随后 4 月迁往 Discord。它还携带显存 strap 解释（「每个 HBM2 栈上可寻址 RAM 的数量由一个 DMEM 区域特定位置的 32 位字定义」）、strap 电阻讨论（R999/R1000 处的 Strap4、PCIE_CFG），以及第一条公开「40 GB 确认可用」报告。注意 NVIDIA-patcher 项目**本身无法驱动 170HX**：它面向图形，产出的是被归类为 GeForce 的 GPU，而不是计算解锁。把它用到 170HX 上不影响 FP32 限速。

2026 年 4 月从该线程链出的一份报告，经过约 18 小时自动化分析后得出结论：「FP 限速由硬件强制、无法覆盖」。它自己的页脚说明它在驱动 **535.288.01** 上执行，早于发布版解锁器瞄准的 GSP 布局，其结论被发布版计算解锁驳斥。它是一个精心记录的错误答案的好例子。

**`github.com/ggml-org/llama.cpp`**：issue #24616（CMP 专用补丁组，在 90HX 上、PCIe 1.1 x4 下达到 240 t/s pp512）、issue #24730（无 DSA 注意力支持，这正是 GLM 级模型回退到密集注意力、在这里变得不可用的原因）、PR #19378（经 `--split-mode tensor` 的后端无关张量并行），以及 discussion #15013。见[LLM 推理](../operations/llm-inference.md)。

**`github.com/JustVugg/colibri`**、**`github.com/LaurieWired/tailslayer`**（刷新时序调优）、**`github.com/microsoft/Tutel`**、**`github.com/sgl-project/sglang`**、**`ikawrakow/ik_llama.cpp`**：为工作负载工作流传的相邻工具，在源材料中没有一个在这张卡上验证过。

**FluidX3D issue #8（2023-10-27）。** FMA 禁用发现的原始功劳，比它 2023-12-06 到达 NVIDIA-patcher issue #73 早两个月。对 `src/lbm.cpp` 中 `LBM_Domain::device_defines()` 的两行补丁（索引 `d99202f..28aeb25`，约第 286 行）在不触碰内核源码的情况下把 `#pragma OPENCL FP_CONTRACT OFF` 加宏遮蔽应用到每个生成的 OpenCL 程序，并在 1175 GB/s 下、去掉 FMA 后测到 **7,681 MLUPs/s**：比原厂 2,276 MLUPs/s 提升 **3.4 倍**。另外，NVIDIA-patcher 线程上报告的同一技术把 170HX FP32 从 **0.395 → 6.285 TFLOPS，15.9 倍**。两者都是 2023 年锁定卡结果。（6.25 TFLOPS 是另一个不同的锁定模式自定义 GEMM 读数，在语料中记录为失败解锁特征：不要把它挂到这个结果上。）

---

## 9. 刻意排除的来源

- **各类商业与市场链接。** 采购在本 wiki 范围之外。
- **分销商料号。** 电容改装件流传着两个不同分销商 SKU，没有来源能定案哪个正确。本 wiki 只引用厂商件号 Taiyo Yuden `MAASJ105SB7224KFCA01`（220 nF、6.3 V、X7R、0402）。见[物理改装](../operations/physical-mods.md)。
- **泄露材料。** 2022 年 2 月至 3 月的 NVIDIA 泄露缓存只在记录中作为一个出处问题被提及。其内容在这里不被使用、引用或链接。见[净室与出处](../history/clean-room-and-provenance.md)。
- **作为证据分享的 AI 聊天记录。** 语料中出现过几十份共享的助手对话。它们被记录为线索和记录在案的幻觉，从不作为来源。
- **视频教程。** 存在若干份、若干语言。没有一份能对照寄存器或日志核实，所以没有一份被引用。

---

## 相关页面

- [已保存的工件](artifacts.md)
- [方法论](methodology.md)
- [净室与出处](../history/clean-room-and-provenance.md)
- [工具谱系](../history/tool-lineage.md)
- [死路](../history/dead-ends.md)
