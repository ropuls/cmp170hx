# 保存的工件

**本页涵盖：** CMP 170HX 解锁工作中幸存下来的技术工件目录：Falcon 固件反汇编、gadget 图谱、ROP 载荷生成器、只读探测脚本、寄存器转储、驱动补丁文件，以及长篇文字稿。对每个工件，本页记录它包含什么、多大、何时出现、为什么重要。它是原始证据的地图，不是教程。关于这些工件共同确立了什么，参见[解锁原理](../unlock/how-it-works.md)和[寄存器参考](../unlock/register-reference.md)。

两个数字框定整个收藏。从两个 Discord 服务器归档了 **131 个文本与代码文件**，另有**共 1,121 个附件**（其余绝大多数是截图和照片）。另外快照了 **13 棵出货解锁工具的 git 树**（`master` 加 12 个未发布分支）和少量外部仓库与 gist。下面的一切都出自这些材料。

> [!NOTE]
> **命名与署名**
>
> 工件只按文件名、大小和日期引用。本 wiki 的任何地方都不记录作者身份。有几个文件被发布过不止一次，有时在不同频道、有时经过轻微编辑；凡发生这种情况都会点名，因为把一次重发当成两个独立观测是真实发生过的错误，在原始材料里至少出现过一次。

---

## 1. Falcon 固件反汇编

SEC2 `booter_load` 微码是整个漏洞利用的对象。它是一个 AES 加密、RSA-3072-PSS 签名的 Heavy Secure（HS）Falcon 镜像，约 **60,160 字节**，384 字节的分离签名在 `PATCH_LOC = 0x8900` 处拼接。调试变体用 NVIDIA 的公开 AES-128-ECB 测试密钥解密，且调试镜像与生产镜像大小相同——这正是净室反汇编能够成立的原因。参见[Falcon 与 Booter](../unlock/falcon-and-booter.md)。

| 工件 | 大小 | 日期 | 内容 |
|---|---:|---|---|
| `booter_load_515_dbg_disasm.asm.txt` | 389,197 B | 2026-06-30 | **515 分支** booter 的反汇编，作为制作 580 对应物的示范模板发布。语料中第一份固件清单。 |
| `booter_load_ga100_dbg_seccode.fuc5.asm` | 545,149 B | 2026-07-01 | 解密后的 GA100 调试安全核心的原始 `envydis -m fuc5` 输出。这是基础工件：之后每个工具使用的每个 gadget 地址都是这个文件中的一条指令边界。发布于 2026-07-01T12:40:37Z。 |
| `booter_load_ga100_dbg_seccode.annotated.fuc5.asm` | 591,794 B | 2026-07-03 | 同一清单，附 LLM 生成的逐函数、逐块注释。 |
| `booter_load_ga100_dbg_seccode.annotated.fuc5_v2.asm` | 607,702 B | 2026-07-09 | 工作参考：11,875 行、逐函数横幅，每条 `lcall` 都带内联注释，以 `lcall 0x1234 // my_function($r10, $r11)` 的形式命名被调方。2026-07-18 特意重发了一份不可变备份副本，以免搜索会话意外编辑它。 |
| `booter_load_ga100_OVERVIEW.md` | 14,164 B | 2026-07-03 | 对 booter 的叙述性通读，通过把原始汇编喂给语言模型生成。 |

> [!WARNING]
> **概览文档按构造即未经验证**
>
> 该概览发布时附带的告诫是作者"不知道其中哪部分是幻觉"。它至少在一处自相矛盾：第 2 节正确地把 CSB `0x9100` 位 31 识别为 `FALCON_CSBERRSTAT.VALID`（一个故障标志），而它自己的关键常数表仍把它称为忙/轮询位。故障解读才是对的，因为代码在位置位时跳转到自循环，而不是在位置位期间循环。项目中所有函数名（`csb_write`、`memcpy`、`wpr_region_program` 等）都是**从行为推断的**；二进制没有符号表。请对照注释清单验证，后者逐字节保留原始指令行。

代码镜像止于 `0x86ff`。载荷中任何高于该值的地址都是 DMEM 指针，不是代码。

---

## 2. 由反汇编派生的 gadget 目录

这些工件把一份 600 kB 的清单变成可以搭建 ROP 链的东西。它们的重要性远超其体量，因为正是它们确立了出货漏洞利用的常数可以从净室材料推导出来。参见[ROP 链](../unlock/rop-chain.md)和[净室与出处](../history/clean-room-and-provenance.md)。

| 工件 | 大小 | 日期 | 内容 |
|---|---:|---|---|
| `register_gadget_atlas.md` | 33,060 B | 2026-07-10 | 从原始反汇编机器生成。只有当过程间可达性分析证明目标寄存器在 `ret` 时仍持有被设置的值（包括在被调函数内部）时才列出该 gadget。每个寄存器带可控性摘要（可经 `mpopaddret` 弹出、mov 设置器、ld 设置器、清零器）和逐行告诫：`canary(r15==r9)` 表示该路径会执行栈 canary 比较，`via-call` 表示会执行真正的子函数，`data-branch` 表示条件依赖你必须设置的状态。发布于 2026-07-10T13:40:14Z。 |
| `Bar0RegWrite.txt` | 2,769 B | 2026-07-02 | 手工提取的 `0x10aa` BAR0 写例程清单，逐指令展示 `mov $r3 0x6340` / `ld b32 $r9 D[$r3]`（canary 加载）及其后的参数搬运。这是整个漏洞利用所依托的原语。 |
| `DIRECT_ENGINE_FINDINGS.md` | 4,503 B | 2026-07-15 | 对 `0x8224` 处直接写 gadget 的分析：`iowrs I[$r10] $r11` 后接 `0x9100` CSB 状态检查与 `lcall 0x1d0f` 报告路径。 |
| `The_missing_piece_per-FBPA_hal` | 2,353 B | 2026-07-12 | 一个逐 FBPA 半容量熔丝假说，点名 `FUSE_HALF_FBPA_EN 0x82049C`、`STATUS_HALF_FBPA 0x820C00`、`CTRL_OPT_FBPA 0x820818` 和 `STATUS_FBP 0x820D38`。作为**假说**而非结果保存：`STATUS_FBPA` 是否根本可写、还是纯粹是熔丝合并输出，这个问题被提出过但从未得到回答。 |

图谱中 `0x0cbd`（"`$r10 <- $r0`，canary(r15==r9)，via-call，`mpopaddret $r3 0x4`"）和 `0x1fbd`（"`$r11 <- $r10`，canary(r15==r9)，via-call，`mpopaddret $r2 0x4`"）的条目，精确描述了这些 gadget 在出货驱动补丁中扮演的角色——而那是在该补丁出现之前八天。

---

## 3. Booter 提取与固件打补丁工具

| 工件 | 大小 | 日期 | 用途 |
|---|---:|---|---|
| `extract-firmware-nouveau.py` | 32,942 B | 2026-07-01 | Nouveau 固件提取器，为 GA100 打了补丁。原版脚本会失败，因为生成的 C 数组名形如 `kgspBinArchiveBooter{LOAD}Ucode_{GPU}_BINDATA_LABEL_IMAGE_{fuse.upper()}_data`，熔丝后缀在某些架构上存在、在其他架构上缺失。产出 `booter_load_dbg/prod`、`booter_unload_dbg/prod` 和 bootloader 二进制块。 |
| `extract-firmware-nouveau-ga100-raw.py` | 32,972 B | 2026-07-05 | 同一工具，进一步打补丁以输出剥离了头部和签名的原始 booter，头部留下 `0x100` 字节未加密内容。 |
| `fwsec_patch.py` | 1,655 B | 2026-06-30 | 第一代 FWSEC 打补丁器。 |
| `fwsec_overcopy_test.sh` | 2,543 B | 2026-06-30 | 证明驱动打补丁器有效的实验：覆盖一个节并观察效果。 |
| `load_custom_bin.py` | 16,937 B | 2026-07-01 | 独立 Falcon 加载器，argparse `[-h] [--pci] [--dmem-out ADDR NDWORDS] [--timeout] [--no-engine-reset] [--quiet]` 加一个位置二进制参数。 |
| `patcher.py` | 15,753 B | 2026-07-02 | GSP 固件打补丁器：签名绕过加一个"thermal trampoline"，针对驱动 580.159.03 编写。 |
| `patch_gsp.py` | 2,563 B | 2026-07-11 | 对 `gsp_tu10x.bin` 做 ELF64 手术：解析 `e_shoff 0x28`、`e_shentsize 0x3A`、`e_shnum 0x3C`、`e_shstrndx 0x3E`，定位 `.fwsignature_ga100`，原地覆盖，把 `sh_size` 改为 `0xF800`，把 `.shstrtab` 追加到文件末尾，重写 `e_shoff`。 |
| `scan_dmem.py` | 10,160 B | 2026-07-16 | 更安全的 ELF 变体，使用 pyelftools，并在 EOF 处追加替换节时遵循 `sh_addralign`。还驱动一次完整 DMEM 扫描：以 4 为步长遍历 `DMEM_ADDR`，对每个值构造转储载荷、打固件补丁、重载模块、可选 FLR。 |

有一处更正值得保存：**`gsp_tu10x.bin` 从来不是反汇编对象。** 它是 booter 验证的 GSP RISC-V ELF 载荷，而不是 booter。Ghidra 从它吐出了约 100 MB 的 C，`riscv64-unknown-elf-objdump` 吐出了约 1.5 GB 汇编。真正的目标约 25 kB，反汇编后约 390 kB。这个文件仍然是正确的*投递载体*，这正是混淆持续存在的原因。

---

## 4. ROP 载荷生成器与载荷清单

载荷清单是 DMEM 地址到值的纯文本表格，手工编写并经肉眼审阅。它们是整个收藏中最易读的工件，也是理解这条链的最佳入口。

| 工件 | 大小 | 日期 | 内容 |
|---|---:|---|---|
| `170HX_ROP_payload_v1.txt` | 2,530 B | 2026-07-05 | 第一条链。Canary 全局 `6340 = FACEB13D`；写值 `02779000` 位于 `FF3C`，送往 `$r0`；canary 地址在 `FF40`/`FF44`。发布时明确标注未测试。 |
| `170HX_ROP_payload_v2.txt` | 2,648 B / 2,918 B | 2026-07-08 | 同一天的两个修订。第二个修复了一个 bug，并把返回 `main()` 的地址移到 `0x8119`。 |
| `170HX_ROP_payload_v3.txt` | 3,034 B | 2026-07-09 | 四次 BAR0 写入，经 `booter_load_wpr_main()` 重回，用于可能的 WPR2 释放。头部注明 `FEAT_OVR_SM_SPD` 和 `FEAT_OVR_SM_SPD_1` 仍必须在 PLM 解锁后从主机设置。 |
| `stack_gen.py` | 4,482 B | 2026-07-04 | 帧生成器：一个初始 `mpopaddret $r6 0x4` 块，然后每帧三个 5 字 `mpopaddret $r2 0x4` 块，返回地址 `0x1fb9`、`0x1fbd`、`0x8224`，退出 `0x79e7`，`payload_size 0xF700`，`dma_target 0x0900`，`stack_start 0xf75c`。 |
| `builder.py` | 9,810 B | 2026-07-01 | 载荷构建器，模式 A（限速写入、停机退出）。其头部精确点名漏洞：IMEM `0x29C4` 处的 `booterVerifyLsSignatures_TU10X` 以 `$r10 = 0x0900`、`size = sizeOfSignature` 调用 `lcall 0x0601`（`booterIssueDma_HAL`），没有任何边界检查。 |
| `payloadn.py`、`payload-lnject.py`、`payload_v3.py` | 2,285 / 4,592 / 2,200 B | 2026-07-08 至 07-09 | 连续的 Python 载荷注入器。 |
| `unlc.py` | 1,919 B | 2026-07-12 | 最小主机侧演示：一旦 FEAT PLM 打开，SS0 和 SS1 就是普通 BAR0 写入。这个两步模型正是出货补丁实现的。 |

> [!CAUTION]
> **`stack_gen.py` v1 不可能工作**
>
> 它的首个版本把每个 canary 槽位都清零。`D[0x6340]` 处的参考字必须复制进每一帧，否则 `0x7dd9` 处的 `__stack_chk_fail` 会触发。作者在发布时就指出了这一点，后来的载荷把标记写进每一帧。保存它是因为这个失败模式有教育意义，不是因为这个文件可用。

### 无驱动 refire 链

`refire_chain_v6.py`（**27,769 B**，2026-07-24）在**不加载 NVIDIA 驱动**的情况下从用户态执行整个解锁，只使用 Python 标准库。它把 BAR0 映射为 16 MiB，把 SEC2 当作基址 `0x00840000`，复位 Falcon，把 NS 代码作为不安全代码加载到 IMEM 0、HS 代码作为安全代码加载到 `IMEM[ns]`，加载 DMEM，把 MAILBOX0/1 设为 WprMeta 物理地址，启动 CPU，并反复溢出已签名 Booter 的签名读取 DMA。模式：`--compute`、`--memory 40`、`--memory 80`、`--pcie-gen2`、`--pcie-retrain`、`--all`。

> [!WARNING]
> **实验性**
>
> 这是一条并行的、非出货路径。它不是 `cmpunlocker` 的一部分。前提条件很严格：root、GPU 已从任何 nvidia 驱动解绑、一张已签名的 GA100 `booter_load` HS 镜像、`echo 16 | sudo tee /proc/sys/vm/nr_hugepages`，以及 `intel_iommu=off` 或 `iommu=pt`，使 DMA 物理地址即主机物理地址。它只带 **10 GB WprMeta 模板**，因此不能原样用于 `0x20C2` 卡。其 `--memory 80` 模式声称"80 GB LMR HW-verified"，最可能的意思是寄存器接受了写入，而非 80 GB 真的可用。参见[80 GB 之问](../frontier/80gb.md)。

---

## 5. 只读探测与表征仪器

这些从未过时。它们是测量仪器，不是解锁器，并且至今仍是验证本页任何声明的正确方式。

| 工件 | 大小 | 日期 | 作用 |
|---|---:|---|---|
| `probe.sh`（mmio-probe） | 19,061 B | 首次 2026-05-31，归档副本 2026-07-07 | 自包含 bash 加内联 Python。只读映射 `/sys/bus/pci/devices/<BDF>/resource0`，转储约 120 到 130 个命名寄存器，外加在 `FBPA_BASE 0x900000`、`FBPA_STRIDE 0x4000` 处的 24 次每 FBPA 读取。输出 `registers.json`、`lspci.txt`、`nvidia-smi.txt`、`gpu-summary.csv`、`probe.log`，打包到 `/tmp/mmio-probe-<host>-<stamp>.tar.gz`。可选编译 CUDA PTX 特殊寄存器转储器（`nvcc -arch=sm_70`），使 SM 数量是**测量**而非报告所得。**它从不写 BAR0。** |
| `ga100_topology_report.py` | 4,848 B 后为 8,128 B | 2026-07-24 | 只读 BAR0 mmap，只转储决定 GA100 板枚举出多少个 SM 的寄存器，用于跨卡比较。第二个修订增加了 inforom 抓取。 |
| `pcielink.sh` | 4,944 B | 2026-07-24 | 标准 PCIe 现场报告采集器。在端点和父桥上解码 `CAP_EXP+0c.l`（LnkCap）、`+2c.l`（LnkCap2）、`+10.w`（LnkCtl）、`+12.w`（LnkSta）、`+24.l`（DevCap2）、`+28.w`（DevCtl2）、`+30.w`（LnkCtl2）、`+32.w`（LnkSta2），以及 sysfs 速率/宽度、`nvidia-smi pcie.link.gen`、AER 计数器和 `SEC2_DEBUG` dmesg 行数。 |
| A100 探测套件：`probe.py` + `README.md` + `sweep.sh` | 9,132 / 3,022 / 3,007 B | 2026-07-20 | 在一块**捐献 A100** 上的三步可选写工作流：只读清点，然后 `sweep.sh` 强制 Gen1/2/3 并经 EXIT 陷阱自动恢复，然后 `probe.py write-test --confirm` 写入并立即恢复 `0x880a8`、`0x8c044` 和 `0x88088`，把每项分类为 `WROTE-OK` 或 `REJECTED(PLM?)`。掩码读哨兵值 `0xBADF5040`。同日稍早还有一份 11,472 B 的 `probe.py` 先行版本。 |
| `check_fold.py` | 未作为文件归档 | 2026-07-24 | 判断解锁显存是否真实的权威测试：分配全部空闲显存减去 2 GiB，用 PTX `sm_80` 内核把每个 64 KB 页写入自己的索引，再逐页读回。必须是稠密写入，因为折叠在通道交织偏移处混叠。输出 `REAL, NO FOLD` 或 `FOLD/mismatch @<pageindex>`。 |
| `cuda_dbg.py` | 未作为文件归档 | 2026-07-19 | 更轻量的混叠测试：`cuMemGetInfo_v2`，然后 `cuMemAlloc_v2` 依次尝试 64、60、56、52、48、44、42 GiB 直到成功；在偏移 0 写 `0xAAAA0000`、40 GiB 处写 `0xBBBB0000`，读偏移 0。读回 `0xBBBB0000` 表示空间发生混叠。 |

> [!NOTE]
> **`probe.sh` 中两个已记载的缺陷**
>
> 其头部第 9 行承诺的 `/dev/mem` 回退并不存在；解析块以 `ERROR: cannot find resource0` 和 `exit 2` 结束。而且该目录先于解锁存在，因此没有任何 `0x001fa7c4`、`0x001fa7cc`、`0x001fa824`/`0x001fa828`、`0x009a0148` 或 `0x00100ce0` 的条目——出货解锁实际操作的正是这五个寄存器。把它们加进去是任何人都能做的最有价值改动，而且不需要硬件。

读取 BAR0 需要 root **外加** `CAP_SYS_RAWIO`，容器化的 GPU 主机通常会丢弃该能力；此时探测会报 `cannot open .../resource0 (EPERM) even as root`。

---

## 6. 寄存器转储与抓取日志

原始抓取是寄存器参考大部分内容的承载证据。参见[寄存器索引](register-index.md)。

| 工件 | 大小 | 日期 | 内容 |
|---|---:|---|---|
| `regs_01.txt` | 16,103 B | 2026-07-12 | 一块原厂卡上的定向带注释寄存器读取：`SM_ISSUE_RATE_MODIFIER 0x00504204 = 0xbadf1201`、`FECS_FEAT_OVERRIDE 0x00409664 = 0xbadf5040`、`FEAT_OVR_ECC_PLM 0x00823800 = 0xffffff8f`、`FEAT_OVR_PLM 0x00823804 = 0xffffff8f`、`FEAT_OVR_QUADRO 0x00823808 = 0x00000081`（消息来源把 `0x00000081` 归给*解锁后*的探测而非原厂卡，所以不要把它当作任何一者）、`FEAT_OVR_ECC 0x0082380c = 0x00888888`，等等，每条都带一行用途说明。 |
| PLM 范围扫描（`save.sh`） | 13,544 B 原始 / 12,265 B 清洗后 | 2026-07-16 / 07-18 | 一份 `script(1)` 打字记录，覆盖 `0x823800` 到 `0x823FFC`，**510 个地址/值对**，28 秒墙钟时间（`Script started 22:51:04+07:00`、`Script done 22:51:32+07:00`），还完整包含操作员先敲错一次命令行。`0x823800` 到 `0x82382C` 块内**十一**个寄存器是活的，外加 `0x823B00`，共**十二**个活 dword。`0x823828` 和 `0x823850` 两个地址完全不在转储中，这正是扫描有 510 对、而不是完整步进 4 扫描该窗口会产生的 512 对的原因。其余全部读 `0xBADF5040`。**两天后在不同频道发布的清洗副本，510 对全部逐字节等价。它是一次观测，不是两次。** |
| `a100.json` | 84,011 B | 2026-07-20 | 一块真实 A100 的完整清单，是解释 170HX 熔丝读数的差分参考。 |
| `a100_native_unbound.json`、`gen_native.json`、`gen1.json`、`gen2.json`、`gen3.json` | 33,259 / 33,259 / 33,273 / 33,271 / 33,271 B | 2026-07-20 | 来自捐献 A100 的强制代际扫描组。那块卡上的写测试没有成功，因此只有读取数据。 |
| `a100-80g.json` | 1,367 B | 2026-07-20 | 来自租用 A100 80 GB 的短转储。 |
| `registers.json` | 23,254 B | 2026-07-25 | 来自一块 **CMP 90HX** 的探测转储，用于跨家族比较。 |
| `ga100_topology_output.txt` | 3,345 B | 2026-07-24 | 一块活卡的拓扑报告输出。 |
| `reg-ref-a100-vs-170hx.csv` | 1,857 B | 2026-07-27 | 两天活体 A100 对原生 8 GB 170HX 的 strap 与熔丝比较，寻找 PCIe Gen3 strap。**阴性结果就是发现**；该方法被宣布为死路，且 170HX 侧缺少完整的逐 FBPA 抓取。 |
| `00_33_31_scanning_lspci.txt` | 16,427 B | 2026-06-25 | 早期完整 `lspci` 扫描。 |
| `dmesg_large.txt` | 7,405 B | 2026-07-09 | 实验的内核日志，测试签名之后 DMEM 中是否有任何重要内容。 |
| `bendy2pcielink.txt` | 2,579 B | 2026-07-24 | 一份归档的 `pcielink.sh` 现场报告。 |

---

## 7. 驱动补丁文件与安装器时代 shell 脚本

| 工件 | 大小 | 日期 | 内容 |
|---|---:|---|---|
| `patch.diff` | 35,867 B | 2026-07-18 | 横跨 11 个文件的 887 行：把泄露包自带的 open-modules 树与上游标签 `610.43.03` 做 diff 得到。发布于 18:01:15Z。历史性决定作用：参见[净室与出处](../history/clean-room-and-provenance.md)。 |
| `cmpunlocker` 出货补丁集 | 37,415 B | 2026-07-18 | 六个补丁、890 行、10 个目标文件：`0001-sec2-postbl-plm-ss-cfg.patch`（19,741 B）、`0002-booter-verify.patch`（3,988 B）、`0003-late-pma.patch`（10,580 B）、`0004-bar0-pramin-clamp.patch`（861 B）、`0005-ce-scrub-workarounds.patch`（1,642 B）、`0006-persistent-sw-state.patch`（603 B）。 |
| `0007-pcie-gen2.patch`、`0008-pcie-gen2-probe-retrain.patch` | **2026-07-29 合并入 `master`**（提交 `2e0a2c02`） | 2026-07-23 起 | Gen2 工作，在分支上开发了一周后合并。`0007` 经 Booter 载荷原语推入一份 23 项 `xp3gTable`；`0008` 在 `kernel-open/nvidia/nv.c` 中加入 `nv_cmp170hx_retrain_gen2()`。参见[PCIe Gen2](../unlock/pcie-gen2.md)。 |
| `mod.txt` | 1,725 B | 2026-07-12 | 一段手写内核补丁，定义 `CMP170HX_WPR2_SAFE_LIMIT 0x0A00000000ULL`（40 GB），并带警告把 `pWprMeta->fbSize` 钳制到其上。对 WPR2 尺寸问题的早期独立表述。 |
| `Guide_SM.sh` | 9,127 B | 2026-07-12 | 全流水线驱动：阶段 1、FLR、卸载、解锁。 |
| `nuke.sh` | 7,359 B | 2026-07-16 | PLM 批量测试 v3：打补丁的 GSP、ROP、FLR、kill、FLR、在无驱动加载下读取 PLM。以 9 轮 3 循环跑了一次 27 地址持久性扫描，每循环两次 FLR。在 `CANARY_ADDR 0x6340` 使用 canary `0xFACEB13D`，`DMA_TARGET 0x0800`。 |
| `test_580.sh`（三个修订）、`test_580v6.sh`、`scaffold_580.sh`、`driver.sh`、`a.sh`、`b.sh` | 3.8 至 7.7 kB | 2026-07-08 至 07-12 | 580 分支实验夹具，包括一次 STRAP 写入修复和一次 `0x65` 状态排查。 |
| `cmp170hx-gen2-setup.sh` | 12,389 B | 2026-07-26 | 独立 Gen2 启用器，与驱动内分支方案不同。其自身头部精确说明范围：Gen1 约 0.85 GB/s 到 Gen2 约 1.71 GB/s，恰好 2 倍，64 GB 解锁、计算与 HBM 带宽不受影响，不刷 VBIOS，不改内核命令行。 |
| `build-llama-170hx.sh` | 3,802 B | 2026-07-27 | 该卡的可复现 llama.cpp 容器构建。 |

> [!CAUTION]
> **`gpuValidateRegOps` 绕过只存在于 `patch.diff` 中**
>
> `patch.diff` 在 `gpuValidateRegOps` 的首条语句处无条件插入 `return NV_OK;`，对所有 GPU 生效，彻底禁用寄存器读写验证。该改动在出货 `cmpunlocker` 树和全部十二个未发布分支中**都不存在**。任何把该改动归给出货工具的文字都是错的。

---

## 8. 长篇文字稿与现场手册

| 工件 | 大小 | 日期 | 它论证什么 |
|---|---:|---|---|
| `ROP_CHAINS_1180f8_nibble_writeup_20260715.md` | 9,628 B | 2026-07-15 | 交接状态问题，以需求表形式陈述：无驱动开火后，`resetPLM 0x8403C4` 必须读 `0xff`（因为 `0x8f` 会阻挡 SEC2 SFTRESET），WPR2 `0x1FA824/28` 必须清除，`1180f8` 的位 [31:28] 必须为 `0x1`。前两项已解决；半字节没有。还记录了每写一次 `+0x18 DMEM` 的帧步长，并把 `D[0xFF50]` 到 `D[0xFF84]` 制表。发布于 2026-07-15T18:48:10Z。 |
| `WRITEUP.md` + `WRITEUP_EXPLAINED.md` | 12,950 + 12,335 B | 2026-07-23 | 在 `0x2082` 卡上的 FFMA 限速调查：实测 0.315 TFLOPS 对理论 25.27 TFLOPS，**对 GSP 固件做 14 次二进制补丁加多次内核模块修改，无一移动限速**。其结论——该限制由熔丝强制、固件无法触及——是宝贵的阴性结果，只被完全不同的 SEC2 路线推翻。 |
| `CMP_170HX_40GB_UNLOCK_GUIDE.md` | 11,994 B | 2026-07-22 | 单次加载 40 GB 驱动补丁的端到端讲解：一次 `modprobe`，无 FLR，无固件替换，`modprobe` 时刻序列逐步写出。 |
| `PCIE_GEN1_LOCK.md` | 15,467 B | 2026-07-24 | PCIe 速率上限现场手册。开篇区分**速率**与**宽度**，明确把宽度（24 电容 C1100 到 C1350 焊接改装）排除在范围外。状态行："软件与无密钥固件面已穷尽；剩余路径是物理的。" |
| `ga100_fbpa_hbm_timing_registers.md` | 23,682 B | 2026-07-25 | `0x9A0200` 到 `0x9A0300` 范围内、位于广播 FBPA 区间 `0x009A0000` 到 `0x009A3FFF` 的 HBM 时序寄存器定义，关键观测是 `CONFIG0.USE_TIMING_REGS`（`0x9A0290` 位 31）读 **0**，因此生效的参数是内部生成的 `TIMING*_GEN` 影子，而不是原始 `TIMING*`/`CONFIG*` 值。 |
| `untitled.md` | 5,814 B | 2026-07-17 | 个人架构笔记，发布时附作者自评：大约 **10%** 有可靠证明或来源，并警告把所有聊天要点拼成文档会得出许多错误结论。 |
| `README.txt` | 4,637 B | 2026-07-18 | 泄露包的随附说明。 |

---

## 9. 基准、调优与工作负载工件

| 工件 | 大小 | 日期 | 内容 |
|---|---:|---|---|
| `170hx-tuning-guide.md` | 26,987 B | 2026-07-27 | 随 `170tune` 夹具发布。针对一张具名参考卡（GA100、70 SM、64 GB 已解锁、驱动 610.43.03、300 W VBIOS `92.00.6D.00.0A`、PCIe Gen2 x4）编写，并明确说明逐卡硅片会有差异，附"鉴定一张新卡"流程，任何偏移被信任用于另一块卡之前必须先做鉴定。也是已封闭死路的记录，避免任何一条被走两遍。 |
| `170HX-benchmark-results.md` | 5,290 B | 2026-07-27 | 八张已解锁 64 GB 卡，驱动 610.43.02，`sm_80`，PCIe **Gen1 x4**，无 cap 改装。逐卡 torch GEMM：FP16 162.7 TFLOPS tensor，BF16 171.4 TFLOPS tensor。 |
| `GLM-5.2-benchmark-report.md` | 7,999 B | 2026-07-24 | 一份值得保留的阴性结果报告：467 GB 4 位量化无法加载，因为主机的 88 GB 系统内存远低于 llama.cpp 加载期计算图遍历所需，RSS 钉在约 87.6 GB 且毫无进展。这是主机侧失败，不是卡的限制。 |
| `cublas_benchmark1` | 826,072 B | 2026-07-16 | 一个 x86-64 ELF 可执行文件，动态链接 `ld-linux-x86-64.so.2`。一份**分发给别人运行的编译二进制**，不是结果日志。 |

这些数字在上下文中的意义参见[性能](../operations/performance.md)和[LLM 推理](../operations/llm-inference.md)。

---

## 10. 未保存的内容

明确列出缺口也是目录的一部分。

- **没有 booter 二进制。** 已签名 HS 镜像及其分离签名被工具点名（`booter_load_580_image.bin`、`booter_load_580_sig.bin`，预期位于 `cmp170hx_boot_bins/verified_hs/` 下），但不在档案中。
- **没有 8 GB WprMeta 抓取。** 无驱动链的 256 字节模板来自一次真实的 10 GB 引导。抓取 8 GB 对应物是一个小而未被阻塞的任务。
- **没有对调试与生产 booter 镜像做哈希比较。** bindata 档案使其轻而易举，而且会定案两者是逐字节相同还是仅仅大小相同。
- **没有模拟器。** 学术论文描述的 Falcon 模拟器从未发布。
- **没有驱动修改指南。** 能定案几个出处问题的两份私人文档不在本来源集中。
- **`master` 上没有 `verify.sh`**，其任何分支副本中也没有 Gen2 检查。
- **`master` 没有 `tools/` 目录。** `probe.sh`、`pcielink.sh`、`check_fold.py`、`cuda_dbg.py`、A100 探测套件和 refire 链全部带外分发。克隆仓库得不到其中任何一个。

---

## 相关页面

- [解锁原理](../unlock/how-it-works.md)
- [Falcon 与 Booter](../unlock/falcon-and-booter.md)
- [ROP 链](../unlock/rop-chain.md)
- [驱动补丁](../unlock/driver-patches.md)
- [寄存器参考](../unlock/register-reference.md) 与[寄存器索引](register-index.md)
- [工具谱系](../history/tool-lineage.md)
- [外部来源](external-sources.md)
- [方法论](methodology.md)
