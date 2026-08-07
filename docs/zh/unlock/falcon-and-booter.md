# SEC2 Falcon 与 Booter Load 微码

**本页涵盖：** 整个 CMP 170HX 解锁运行所在的安全协处理器：SEC2 Falcon 是什么、"Booter Load" 微码做什么、如何进入和离开重型安全（heavy-secure）模式、booter 镜像在两个地址空间中的内部布局、GSP 签名缓冲区在哪里，以及补丁驱动如何精确调用 booter。漏洞利用本身在 [ROP 链](rop-chain.md)；它打开的掩码在 [权限级掩码](privilege-level-masks.md)。

**两句话版关键结论。** NVIDIA 自签名、AES 加密的 `booter_load` 微码被正常加载和认证，然后在认证*已经成功之后*被一个由主机驱动控制的签名缓冲区破坏。没有伪造签名、没有提取密钥、也从不执行攻击者提供的指令：booter 成为解锁的执行引擎，同时逐字节仍是 NVIDIA 发布的微码。

---

## 1. 为什么 Booter 会存在

GA100 裸片上带着两个与此相关、截然不同的处理器。

| 处理器 | 位置 | 核心 | 密码学 | 作用 |
|---|---|---|---|---|
| SEC2 Falcon | BAR0 `0x00840000` | Falcon v4/v5，16 位哈佛 | AES + RSA + SCP 机密 | 安全协处理器。可以解密并验证自己的代码镜像。 |
| GSP | BAR0 `0x00110000` / `0x00111000` | NVIDIA RISC-V（NVRISCV） | 无 | 运行 GSP-RM，即资源管理器固件。 |

因为 GSP RISC-V 核心没有密码学功能，它无法验证自己的镜像。验证被委托给一个名为 *booter* 的 SEC2 Falcon 微码。存在两个 booter：`booter_load` 和 `booter_unload`；本页讲的是 `booter_load`。

Booter **不是** [VBIOS](../hardware/vbios.md) 的一部分。它随驱动包作为 `nvidia.ko` 中编译进的 BINDATA 数组发布，并且是版本相关的，就像它验证的加密 GSP 固件一样。在 610 驱动中，GA100 数组是：

```text
kgspBinArchiveBooterLoadUcode_GA100_BINDATA_LABEL_IMAGE_DBG_data[]
  in src/nvidia/generated/g_bindata_kgspGetBinArchiveBooterLoadUcode_GA100.c
  DATA SIZE (bytes): 60160
  COMPRESSED SIZE (bytes): 34145
```

因此运行漏洞利用不需要单独的 booter 文件。驱动已经带着它。

解锁攻击的完整启动链是：上电时从 SPI flash 加载 VBIOS，片上 Falcon 验证 VBIOS，芯片进入安全模式，驱动提供 `gsp_tu10x.bin`、由 Falcon 检查其签名，然后驱动使用 GSP 客户端读取内存容量并暴露设备。解锁攻击第四步，在 booter DMA 的 GSP 签名缓冲区中植入载荷。

---

## 2. Falcon 安全模型 {#2-the-falcon-security-model}

自 Maxwell 以来 Falcon 有三种执行模式。

| 模式 | 如何进入 | 能做什么 |
|---|---|---|
| 非安全（NS） | 加载任意代码，设置 BOOTVEC，STARTCPU | 唯一不需要 NVIDIA 签名微码就能到达的模式。被限制访问许多寄存器和 DMA。 |
| 轻安全（LS） | 只能从 Heavy-Secure 上下文进入（GM20x 起） | 介于 NS 和 HS 之间。 |
| 重型安全（HS） | PC 落在标记为安全的代码块上、MAC 比较成功之后由硬件授予 | Falcon 变成黑盒：内部状态无法从外部读写。运行在 LEVEL2/L3，可以重写权限级掩码并编程受保护区域。 |

本项目"L0 到 L3"的权限级词汇指同一个模型。这些级别如何按寄存器执行，见 [权限级掩码](privilege-level-masks.md)。

`booter_load` 绝不能在 NS 模式下执行。它的主体是 AES 加密的，只能在 HS 模式下的 Falcon 内部解密。Falcon 通过运行 0x100 字节明文 NS 前导、然后发出那条解密代码、验证代码并切换到 HS 的特殊指令来进入 HS。其后果是架构性的，也是整个解锁形态如此的原因：

> [!NOTE]
> **塑造一切的那条规则**
>
> 解锁中的每次 HS 特权寄存器写入都必须从被劫持的真实 booter 内部发出。自制微码做不到，因为自制微码无法被放进 HS 运行。

HS 入口例程（为 Tegra TSEC 逆向、GA100 SEC2 上结构相同）计算 `microcode_start = (*SEC & 0xFF) << 8` 和 `microcode_size = ((*SEC >> 24) & 0xFF) << 8`，把微码的 Davies-Meyer MAC 算进 `$c5`，然后运行：

```asm
csecret $c3, 0x1
ckeyreg $c3
cenc     $c3, $c7
ckeyreg  $c3
cenc     $c4, $c5
csigcmp  $c4, $c6
```

起始地址或大小为零会引发 `OP_SECURE_FAULT`。认证必须满足四个条件：微码页必须映射到预先选定的虚拟地址、标记为 secret、该信息载入 `SEC` 寄存器，且密码学寄存器 6 中存在有效 MAC。

> [!WARNING]
> **两套不同的验证方案，不要混为一谈**
>
> 不可变启动 ROM 对 HS booter 镜像的检查在语料中被描述为 RSA-3K 检查，且一个 384 字节签名 blob 确实随镜像发布在 `PATCH_LOC = 0x8900`。另外，booter *自己*对它所加载 GSP 镜像的验证被追踪为 `_acrVerifySignature_TU10X` 到 `_acrCalculateDmhash_TU10X` 到 `_acrDeriveLsVerifKeyAndEncryptDmHash_TU10X` 到 `_acrMemcmp`，这是 Davies-Meyer 哈希加由 `csecret` 派生密钥的 AES 密钥派生，该路径上没有 RSA。Booter 确实包含一个单独用于别处的 PKA/modexp 块（`rsa_pubkey_load 0x4768`、`pka_modexp_run 0x54ab`）。对两者协调的置信度为中等：存档中没有人用完全相同的措辞陈述过这一点。

两套方案都没有被解锁破坏。

---

## 3. 哈佛架构：绝不能混用的两个地址空间

> [!CAUTION]
> **IMEM 地址和 DMEM 地址看起来一样，但不可互换**
>
> SEC2 Falcon 是一个**哈佛架构**核心，有独立的 16 位指令存储器（IMEM）和 16 位数据存储器（DMEM）空间。`0x6340` 作为 IMEM 地址是无意义的代码；`0x6340` 作为 DMEM 地址是栈金丝雀防护全局变量。若干流传文档把两者混成一张"内存映射图"，那是错的。本 wiki 上每个地址都标注为 IMEM、DMEM、CSB（Falcon I/O）或 BAR0。

| 属性 | IMEM | DMEM |
|---|---|---|
| 大小 | `0x10000`（64 KB） | `0x10000`（64 KB） |
| Falcon 虚拟窗口 | `0x4000000`-`0x400FFFF` | `0x4010000`-`0x401FFFF` |
| 块对齐 | 256 字节（`FLCN_BLK_ALIGNMENT`） | 256 字节 |
| 漏洞利用触及的部分 | 无 | `0x0800`-`0xFFFF` |

因为 GSP 签名缓冲区位于 DMEM，签名 DMA 只能砸 DMEM，永远写不到 IMEM。这单一事实就是为什么漏洞利用是对已签名镜像的纯返回导向编程，而不是代码注入。这也意味着 DMEM 的 16 位空间本身无法寻址 32 位 BAR0 寄存器，所以载荷需要一个驱动 Falcon BAR0 主控的 gadget。

全文还出现另外两个空间：

- **CSB / Falcon I/O**，通过 `iord` / `iowrs I[...]` 寻址。示例：`I[0x1000]` 是 MAILBOX0，`I[0x9100]` 是 `FALCON_CSBERRSTAT`，`I[0x1c000]`/`I[0x1c100]`/`I[0x1c200]` 是 BAR0 主控。
- **BAR0 / PRI**，主机对特权寄存器接口的 32 位视图。主机、SEC2 和 GSP RISC-V 都通过 PRI 通信；PLM 门控其中的区域。

---

## 4. 从主机看 SEC2：BAR0 寄存器映射

SEC2 位于 `NV_PSEC_BASE = 0x00840000`，是 Falcon 核心，不是 RISC-V（`HWCFG2` 位 10 读出 0）。完整偏移，由 GPU `10de:20c2` 上可工作的无驱动 C 加载器验证：

| 寄存器 | 偏移 | 绝对地址 | 备注 |
|---|---|---|---|
| `IRQSCLR` | `+0x004` | `0x00840004` | |
| `IRQSTAT` | `+0x008` | `0x00840008` | |
| `MAILBOX0` | `+0x040` | `0x00840040` | Falcon CSB `I[0x1000]` 的主机别名 |
| `MAILBOX1` | `+0x044` | `0x00840044` | |
| `SFTRESET` | `+0x07c` | `0x0084007c` | PL0 写入对 HSMODE 无效 |
| `FALCON_RM` | `+0x084` | `0x00840084` | |
| `EXCI` | `+0x0d0` | `0x008400d0` | 异常信息 |
| `HWCFG2` | `+0x0f4` | `0x008400f4` | 位 10 = RISCV |
| `CPUCTL` | `+0x100` | `0x00840100` | 位1 STARTCPU，位3 IREADY，位4 HALTED，位5 STOPPED，位6 ALIAS_EN |
| `BOOTVEC` | `+0x104` | `0x00840104` | |
| `DMACTL` | `+0x10c` | `0x0084010c` | 位1 DMEM_SCRUB_PENDING，位2 IMEM_SCRUB_PENDING |
| `DMATRFBASE` | `+0x110` | `0x00840110` | `(phys >> 8) & 0xFFFFFFFF` |
| `DMATRFMOFFS` | `+0x114` | `0x00840114` | |
| `DMATRFCMD` | `+0x118` | `0x00840118` | 位0 FULL，位1 IDLE，位2-3 SEC，位4 IMEM，位5 WRITE，位8-10 SIZE（`0x6` = 256 B），位12-14 CTXDMA，位16 SET_DMTAG |
| `DMATRFFBOFFS` | `+0x11c` | `0x0084011c` | |
| `DMATRFBASE1` | `+0x128` | `0x00840128` | `((phys >> 8) >> 32) & 0x1FF` |
| `CPUCTL_ALIAS` | `+0x130` | `0x00840130` | |
| `TRACEPC` | `+0x14c` | `0x0084014c` | 位 [23:0] = PC 快照 |
| `IMEMC0` | `+0x180` | `0x00840180` | 位24 AINCW，**位28 SECURE**，位23:8 BLK，位7:2 OFFS |
| `IMEMD0` | `+0x184` | `0x00840184` | |
| `IMEMT0` | `+0x188` | `0x00840188` | |
| `DMEMC0` | `+0x1c0` | `0x008401c0` | |
| `DMEMD0` | `+0x1c4` | `0x008401c4` | |
| `SCTL` | `+0x240` | `0x00840240` | 位0 LSMODE，位1 HSMODE（只读），位14 AUTH_EN |
| `IMEM_PRIV_LEVEL_MASK` | `+0x280` | `0x00840280` | |
| `DMEM_PRIV_LEVEL_MASK` | `+0x284` | `0x00840284` | 在 LS 模式下读出 `0xFF`，完全开放 |
| `FBIF_TRANSCFG(n)` | `+0x600 + 4n` | `0x00840600`+ | TARGET 位 [1:0]：0 LOCAL_FB，1 COHERENT_SYSMEM，2 NONCOHERENT_SYSMEM；MEM_TYPE 位 2 = PHYSICAL |
| `FBIF_CTL` | `+0x624` | `0x00840624` | 位 7 = ALLOW_PHYS_NO_CTX |
| `FALCON_ENGINE` | `+0x3c0` | `0x008403c0` | 位0：1 = 复位，0 = 运行 |
| `RESET_PRIV_LEVEL_MASK` | `+0x3c4` | `0x008403c4` | 复位 PLM。见第 9 节。 |
| `PRIVSTATE_PLM` | `+0x3d0` | `0x008403d0` | 仅具名，从未写测 |
| `SCP_CTL_P2PRX` | `+0x530` | `0x00840530` | 位3 SFK_LOADED |
| `KFUSE_LOAD_CTL` | `+0x11ec` | `0x008411ec` | 读取以触发 SFK 加载 |

GSP 一侧作对比：`NV_FALCON2_GSP_BASE = 0x00111000`、`RISCV_STATUS 0x00111240`、`RISCV_CPUCTL 0x00111268`、GSP `MAILBOX0 0x00110040`、GSP FBIF 基址 `0x00110600`。

`EXCI` 解码为 `expc = ((exci >> 28) << 20) | (exci & 0xFFFFF)`、`excause = (exci >> 20) & 0x1F`，原因包括 `0x08` ILL_INS、`0x09` INV_INS、`0x0a` MISS_INS、`0x0b` DHIT_INS（IMEM 块存在但未经 BROM 认证）、`0x0d` SP_OVERFLOW、`0x0f` BRKPT_INS、`0x10` DMEM_MISS、`0x11` DMEM_DHIT、`0x12` DMEM_PAFAULT、`0x13` DMEM_PERM、`0x15` BROM_CALL、`0x16` KMEM_VIOLATION、`0x17` BMEM_PERM。

---

## 5. Booter Load 镜像

### 5.1 文件布局

| 区域 | 偏移 | 内容 |
|---|---|---|
| NS 引导 | `0x0000`-`0x0100` | 明文。256 字节。 |
| HS 代码 | `0x0100`-`0x8600` 或 `0x0100`-`0x8700`；**各来源相差 256 字节** | AES-128-ECB 加密。 |
| 数据（`osData`） | 算术上自 `0x8700` 起；`0x8600` 也有人引用 | 第 11 节的 `hsSigDmemAddr = patchLoc - dataOffset` 推导需要 `0x8700`（`0x8900 - 0x8700` = DMEM `0x200`），34,304 字节的安全代码测量也需要它（`0x0100`-`0x8700` 是 34,304 字节；`0x0100`-`0x8600` 只有 34,048）。两条证据线指向同一方向，因此 `0x8600` 是较弱的读数。对齐解密偏移前请对照你自己的镜像验证。 |
| 签名 | `0x8900`（`PATCH_LOC`） | 384 字节 |

未压缩代码大小为 `0x8700` = 34,560 字节。对分析的调试签名构建实测的节大小为 34,304 字节安全代码、25,600 字节数据和 256 字节非安全代码。原始提取的 GA100 文件被零填充到 60,100 字节；编译进的 BINDATA blob 为 60,160 字节。

加密区域恰好从 `0x100` 开始，是 16 的倍数，那里的第一条解密指令是：

```text
89 fc ff 00    mov $r9 0xfffc
```

### 5.2 密码学突破

这个发现使无需任何泄露源码就能做明文反汇编成为可能：

> [!NOTE]
> **调试版和生产版 booter 镜像包含完全相同的明文代码**
>
> 只有 AES 密钥不同。调试镜像用非机密的编号测试密钥加密，因此通过解密调试镜像就能读取生产版 HS 代码。底层密码学发现于 2026 年 5 月；2026-07-01 应用于 GA100。

实用标记：

| 项目 | 值 |
|---|---|
| 调试密钥编号 | 37 |
| 该密钥下尾部零填充的 AES-128-ECB 密文 | `717D1494 EACA317F F1061952 58B38377` |
| 反汇编器 | `envytools` / `envydis` |

如果上面的 16 字节常量出现在文件自己的零区域之前，说明调试 blob 被正确提取且正确对齐；如果它能用测试密钥工具解密回零，说明解密正确。生产版 blob 显示不同的尾部模式。一个错过会浪费数小时的工作流细节：4 字节轮密钥必须按人类可读形式**相反**的顺序喂给 AES/Rijndael 工具，因为密钥编号在最后一轮密钥而不是第一轮（置信度：中等，单条可执行指令，但它描述的工作流确实产出了正确输出）。

### 5.3 提取

提取通过给 NVIDIA 自己的 `extract-firmware-nouveau.py` 打补丁完成，让它只输出 GA100 prod 和 debug booter，解析 `kgspBinArchiveBooter{Load,Unload}Ucode_GA100_BINDATA_LABEL_IMAGE_{PROD,DBG}_data` 及对应的 `..._SIG_{PROD,DBG}_data`。**GA100 每个签名的大小是 384 字节**，而 TU10x 用 16，因此调用是 `booter('ga100','load',384,'prod')` 及其三个同类。所有非 GA100 芯片（tu102、tu116、ga102、ad102、gh100、gb100、gb202）在 `main()` 中都被注释掉。

第二个变体 `extract-firmware-nouveau-ga100-raw.py` 剥离所有容器结构（带 `0x10de` 魔数和 6 个 dword 的 `nvfw_bin_hdr`、带 9 个 dword 的 `nvfw_hs_header_v2`、签名 blob、`patch_loc` / `patch_sig` / `fuse_ver` / `engine_id` / `ucode_id` / `num_sigs` 以及描述符），只把原始固件写到 `booter_{load,unload}_{prod,dbg}-<ver>_raw.bin`。该原始布局供 envydis 和 objdump 使用，是所有后续工具链的输入。

第二条提取路线从已加载的原厂驱动工作：通过 `IMEMC 0x840180`（设置位 25 自动递增）和 `IMEMD 0x840184`（偏移 `0`..`0x8700`）读取 SEC2 IMEM，通过 `DMEMC 0x8401c0` / `DMEMD 0x8401c4` 读取 DMEM，然后拼接 IMEM（NS+HS）+ DMEM。

Booter 几何也可以**不用反汇编器**恢复：扫描原始镜像的前 `0x100` 字节即可。`imm_before(ns, ff9f04)` 得出 NS 末尾，`imm_before(ns, fd9e04bb9002b69410)` 得出 DMEM 偏移，其中 `imm_before` 要求标记前四字节位置是 `0x89`（`mov $r9 imm24` 操作码），然后组装小端 24 位立即数。如果任一标记缺失，或 `dmem <= base`，解析器报"not an ACR booter image"。加载随后把 `img[0:ns]` 无安全位写到 IMEM 0，把 `img[ns:dmem]` 以 SECURE 位 `1 << 28` 写到 `IMEM[ns]`，把 `img[dmem:]` 写到 DMEM 0。

### 5.4 版本可移植性

| 驱动分支 | GA100 `booter_load` | gadget 地址有效 |
|---|---|---|
| 515 时代 | 不同构建；金丝雀全局在 `0x2B20` 或 `0x2D20` | 否 |
| 580 到 610 | **逐位相同**（在 580.173.02、580.159.04、580.159.03、610.43.02、595.84 上验证） | 是 |

因此本 wiki 上每个 ROP gadget 地址在整个 580-610 范围内都成立，无需重新推导。515 booter 是本项目之前被公开反汇编的那一个，它不带这个漏洞。

已发表论文的跨版本语料覆盖 450、460、470、510（两个补丁版）、515、525、535、560、570 和 580 分支。510 SEC2 booter 的签名路径只使用常量或钳制 DMA 长度，没有按元数据大小的复制；580 booter 表现出无界复制；525 镜像无法恢复，因为 booter 打包方式变了。

> [!NOTE]
> **开放问题：第一个受影响的驱动分支**
>
> 溢出在 **510 中不存在**、在 **580 中存在**。515 到 570 分支**不确定**。通过恢复并分析 515、535、560 和 570 的 GA100 booter 是否带按元数据大小的复制来定案。

### 5.5 谱系命名

GA100 使用 **Turing 代**固件：GSP blob 是 `gsp_tu10x.bin`，SEC2 booter 是 Turing 血统的 `booter_load`。这在 NVIDIA 自己的 `nouveau/extract-firmware-nouveau.txt` 中有陈述，并由失败路径上 TU102 后缀的 RM 符号（`kgspBootstrap_TU102`、`s_executeBooterUcode_TU102`）佐证。经频道内更正后的正确命名：

| 前缀 | 覆盖 |
|---|---|
| `tu10x` | 全部 Turing |
| `ga100` | 仅 A100 和 CMP 170HX |
| `ga10x` | 其他 Ampere（GA102、RTX 3090、CMP 90HX） |

因为 170HX 加载 Turing 的 `booter_load`，它继承了 Turing booter 的 DMA/签名溢出。加载 Ampere booter 的卡是否从那条路径继承它，有争议且未解决；见 [开放问题](../frontier/open-questions.md)。

---

## 6. Booter 内部：IMEM 函数映射

本表所有地址都是解密调试签名镜像 `booter_load_ga100_dbg_seccode.fuc5.asm` 中的 **IMEM**（代码）地址。带注释清单 `booter_load_ga100_dbg_seccode.annotated.fuc5_v2.asm` 包含 10,934 行未修改代码，带逐函数横幅。

| IMEM | 符号 | 作用 |
|---|---|---|
| `0x0100` | `_start` | 入口点。十阶段 HS 前导。 |
| `0x04a7` | （自循环） | `3e a7 04 00 B lbra 0x4a7`。载荷填充 dword 的自旋停靠点。 |
| `0x04d0` | `_start` 出口 | |
| `0x04d4` | `dma_copy_block` | 真正的 DMA 到 DMEM 循环（`xdld`）。**溢出的帧。** |
| `0x0602` | `dma_dispatch_descriptors` | 提交最多四个子描述符，标记 `r14 = 0xa0..0xa3` |
| `0x0c7c` | `regblock_read_guarded` | |
| `0x0cbd` | elevator | `0x0c7c` 内部的 `mov $r10 $r0` |
| `0x0ccb` | `regtable_rw_indexed` | 对寄存器描述符表的索引访问；以 `mpopaddret $r5 0x8` 结束 |
| `0x0d66` | ACR 互斥锁获取 | id 字节读出 0 或 `0xff` 时错误 `0x1a`，类型错误时 `0x1c` |
| `0x0e85` | `memcpy` | |
| `0x0aa1` | `tgt_falcon_bringup` | 启动目标 Falcon；错误 `0x1c`、`0x11` |
| `0x1034` | `watchdog_set` | 用 `0x1312d00`（20,000,000）播种 `I[0x1c300]` |
| `0x1064` | `mailbox_wait_ready` | 轮询 `I[0x1c000]` 位 [14:12]：0 完成，1 自旋，否则错误 `0x15` |
| `0x10aa` | `reg_write_indirect` / `_acrlibBar0RegWrite_TU10X` | **任意 BAR0 写入原语。** 约 70 个调用点。 |
| `0x10b9` | （中途入口） | 跳过 `r10`/`r11` 到 `r0`/`r1` 的搬运 |
| `0x10ff` | `mpopaddret $r3 0x4` | `0x10aa` 的尾声；使写入自我链接 |
| `0x1196` | `reg_read_indirect` | |
| `0x14cf` | `tlb_scan_invalidate` | 冲刷镜像自身的陈旧映射，范围 `[0, 0x8700)` |
| `0x154a` | `wpr_desc_validate` | +0 处魔数 `0x371a60b3`，+4 处 `0xdc3aae21`；错误 `0x89`-`0x90` |
| `0x19a2` | `va_to_pa_walk` | 三级软件页遍历；错误 `0x2` |
| `0x1b44` | `set_1180f8_bit24` | 把 `0x01000000` OR 进 `0x001180f8` |
| `0x1ba3` | `check_1180f8_2724` | 要求 `0x001180f8[27:24] == 0`，否则错误 `0x88` |
| `0x1c0e` | `set_1180f8_top_nibble`（finalize） | 清除 [31:28] 并 OR `(r0 << 28)`；尾声 `0x1c72` |
| `0x1c75` | `check_1180f8_nibbles` | 要求 [31:28] == 0 **且** [23:20] == 0，否则错误 `0x29` |
| `0x1d0f` | `report_status` | 把 `$r0` 写入 MAILBOX0 |
| `0x1d3b` | `f100_field_save_restore` | 经 DMEM `0x1900` 对寄存器 `0xf100` 位 [4:6] 做 RMW |
| `0x1e09` | `scp_key_derive` | 用硬件机密 `0x37` 或 `0x36` 做 `csecret $c7` |
| `0x1f92` | `read_820344_820348` | |
| `0x1fb9` / `0x1fbd` / `0x1fca` | elevators | 见 [ROP 链](rop-chain.md) |
| `0x21f4` | `image_dma_loader` | 调用点 `0x2725` |
| `0x2120` | `chunked_dma_copy` | 针对寄存器 `0x4b00` 的 `0x100` 字节块 |
| `0x22ba` | `booter_load_wpr_main` | 错误 `0x5`、`0x89`、`0x8a`、`0x96`、`0x98`、`0x9c`、`0xa4` |
| `0x27fa` | `0x22ba` 内的 rejoin 点 | 写 `D[0x6f8]`/`D[0x6fc]`/`D[0x648]`；**不碰**任何 WPR2 寄存器 |
| `0x28ac` | `wpr_region_check` | 错误 `0x5` |
| `0x291e` | `wpr_region_program` | 实际写 `0x001fa824`/`0x001fa828`；拒绝空区域 |
| `0x2e80` | `image_auth_decrypt` | 流式 `0x100` 字节块，密钥句柄 `0x17d78414` |
| `0x3747` | `image_copy_verify` | 正常返回 `0x2740` |
| `0x37b3` | 签名 DMA 调用点 | `lcall 0x4d4` |
| `0x37b7` | DMA 后结果检查 | `ld $r9 D[$r1+0x50]` |
| `0x399a` | `ls_sig_verify` | 要求 `r10 == 0x700`；错误 `0x98` |
| `0x3c8f` | `firmware_load_main` | DMEM `0x5f00` 处魔数 `'FREE'` / `'HEAP'` |
| `0x4768` | `rsa_pubkey_load` | 模数零填充到 `0x200`，`e = 0x10001` |
| `0x54ab` | `pka_modexp_run` | 错误 `0x63`-`0x6d`（`0x6c` 超时） |
| `0x59c4` | `antirollback_version` | 密钥句柄 `0x17d78400`；错误 `0x5c`、`0x1` |
| `0x683f` | `boot_mode_dispatch` | |
| `0x68ed` | `reg_init` | 写 `0x110624 = 0x90`、`0x110684 = 1`、`0x11126c = 1` |
| `0x6a71` | `chipid_gate` | 只接受芯片 ID `0x170` 和 `0x171`；错误 `0x4b` |
| `0x6abd` | `rsa4096_pubkey_load` | 四张 512 字节表 |
| `0x76ee` | `fb_size_compute` | 解码 LMR。见 [显存几何](memory-geometry.md)。 |
| `0x79cc` | `memcfg_program` | |
| `0x7a64` | `memcfg_apply_poll` | 超时 100000，错误 `0xa6` |
| `0x7c65` | `memcfg_timing_program` | 超时 125000，错误 `0xa7`，基常量 `0x32a` |
| `0x7dd9` | `__stack_chk_fail` / `panic()` | 把 `0x47` 写入 MAILBOX0，在 `0x7def` 自旋 |
| `0x7de9` | （panic 主体） | 打印 `$r15` 中的任何内容。每个调试 ROP 的基础。 |
| `0x7df3` | `memcmp_ct` | 恒定时间比较 |
| `0x7e76` | `secure_teardown` | 从不返回 |
| `0x7eef` | 密码学自零清扫 | `secure_teardown` 内部 |
| `0x7f2f` | `secure_teardown` 内的退出 gadget | 正式载荷的终结符 |
| `0x7f82` | **`main`** | |
| `0x8137` | `booter_load_wrap` | |
| `0x815a` | `booter_load_wrap` 的金丝雀检查尾 / 栈吞噬器 | 见下方说明 |
| `0x8224` | `csb_write` | 存储是 `0x8232` 处的 `iowrs I[$r10] $r11` |
| `0x8262` | 裸 `ret` | 有用的对齐 gadget |
| `0x8264` | `csb_read` | |
| `0x8307` | `fbif_set_bit800` | 在掩码 `0x0ffff8ff` 下设置 `0x001fa814`/`0x001fa818` 中的位 `0x800` |

> [!NOTE]
> **已解决：`0x815a` 在 `booter_load_wrap` 内部**
>
> 一份目录称它为"主金丝雀检查尾"；另一份把它标注为"`booter_load_wrap` 中检查金丝雀然后什么都不做的栈吞噬器"。带注释的 v2 清单解决了它。`main` 运行 `0x7f82` 到 `0x8134`，其自身金丝雀检查以 `mpopaddret $r0 0x10` 结束。`booter_load_wrap` 运行 `0x8137` 到 `0x8173`，以 `mpopaddret $r0 0x4` 结束，下一个函数横幅是 `0x8176` 的 `nibble_rmw`。因此 `0x815a` 位于 `booter_load_wrap` 内部，由 `0x8150` 处跳过 `boot_mode_dispatch (0x683f)` 调用的 `bra b32 $r10 0x0 e 0x815a` 到达。它是该包装器的金丝雀检查尾；`main` 自己的尾是 `0x811d` 处单独的块。第二份目录是对的。

---

## 7. 启动流程

### 7.1 `_start`（IMEM `0x0100`）：十阶段 HS 前导

1. 在 `0x0107` 清扫 SCP 状态（`csigclr` / `csecret` / `cxor`）。每条 `csecret $cN 0x0` 后面紧跟 `cxor $cN $cN`，因此寄存器组在供给的瞬间就被清零。
2. 在 `0x014b` 清除每个通用寄存器 `$r0`..`$r15`。
3. 在 `0x016b` 让 Falcon 静默，轮询 `I[0x9100]` 位 31。
4. 在 `0x02b3` 清除中断使能标志 `ie0` / `ie1` / `ie2` 和定时器/异常标志（`mov $r9 0x10` / `0x11` / `0x12` / `0x18`，每条后面跟 `bclr $flags $r9`）。
5. **阶段 5** 设置陷阱向量 `$tv = 0xeb` 并清除 `$cauth` 安全故障使能位：`mov $r9 0xeb; mov $tv $r9; mov $r9 $cauth; mov $r15 -0x80001; and $r9 $r15; mov $cauth $r9`。
6. **阶段 6** 依次触碰 CSB `0x4e00`（掩码 `0xff000000`，然后 OR `0x80003000`）、`0x10100`（OR `0x101`，自旋直到位 `0x100` 清除，上限 `0x400` 次迭代）、`0x14000`（设置 `0x7fff`）、`0x14100`（低 16 位保留，OR `0x03ff0000`）、`0x14b00`（OR `0xff00`），然后再次 `0x10100`（OR `0x1000`）。
7. 在 `0x0433` 通过 `crnd` 自供给 SCP。
8. 在 `0x0463` 验证 SCP 并扫描 DMEM `0x6330`..`0x6340`。
9. **阶段 9** 清除 `0x10100` 位 `0x1000`（与 `-0x1001` 相与）并在 DMEM `0x6340` 安装栈金丝雀，取自扫描 DMEM `0x6330`..`0x6340` 时找到的第一个非零字。
10. 在 `0x04cc` `lcall 0x7f82` 进入 `main()`；在 `0x04d0` 退出。

### 7.2 `main`（IMEM `0x7f82`）

`main` 按顺序编排：`f100_field_save_restore (0x1d3b)`；权限级掩码和孔径编程；`tgt_falcon_bringup (0xaa1)`；`chipid_gate (0x6a71)`；描述符验证；`regtable_reverse_lookup (0xd66)`；`tlb_scan_invalidate (0x14cf)`；调用 `booter_load_wpr_main (0x22ba)` 的 `booter_load_wrap (0x8137)`；finalize 提交 `(0x1c0e)`；`report_status (0x1d0f)`；成功时 `secure_teardown (0x7e76)`。

在 MAIN.2 中，紧接 `watchdog_set` 之后，Booter Load 在 **CSB** 空间写入四个固定权限级掩码和孔径值：

```asm
I[0x12000] = 0x11111101
I[0x12400] = 0x00000111
I[0x12600] = 0x11111111
I[0x12100] = 0x00011100
```

每个后面都跟着内联的失败关闭（fail-closed）断言，因此四个中的任何一个 CSB 错误都会使 Falcon 卡在无限自分支中。

`chipid_gate` 读取寄存器 `0xa00` 位 [28:20]，只接受芯片 ID `0x170` 和 `0x171`。Strap `0x170` 无条件通过；strap `0x171` 还要求寄存器 `0x10200` 的位 20 被设置，否则错误 `0x4b`。CMP 170HX 上 BAR0 `0x00000000` 的 `PMC_BOOT_0` 读出 `0x170000a1`，因此实现 ID `0x170` 通过。见 [GA100 硅片](../hardware/ga100-silicon.md)。

`main` 的尾部，精确如下：

```asm
0x80fe:  r9 = sp+8 = 0xFFEC          ; DMEM
0x8101:  r10 = D[0xFFEC]
0x8103:  lcall 0x1c0e                ; finalize / set_1180f8_top_nibble
0x8107:  if r0 != 0 skip
0x810b:  r0 = r10 = D[0xFFEC]
0x810d:  mov r10, r0
0x810f:  lcall 0x1d0f                ; report_status -> MAILBOX0 = r0
0x8113:  if r0 == 0 -> 0x8119
0x8117:  exit                        ; raw HS halt
0x8119:  lcall 0x7e76                ; secure_teardown
```

**DMEM `0xFFEC` 是喂给退出状态、决定是否运行 teardown 的槽。** 这是由一次硬件 A/B 定案的，它驳斥了竞争的 `0xFFE4` 假说，并与已发表 ROP v3 注释 "FFEC 00000000 <- Return value to main() to indicate success ($r0)" 一致。把 `D[0xFFEC]` 设为 `0xDEADBEEF` 使 `0x001180f8` 从 `0x11000000` 变为 `0xf1000000`，与 `(r0 << 28)` 模型的预测完全一致。

### 7.3 Booter 实际为 GPU 做什么

除 GSP 交接外，还有两项重要工作：

- **内存时钟和时序编程。** `memcfg_program (0x79cc)` 读取 BAR0 `0x20414`、`0x136658`、`0x136e58` 和 `0x136458`，打包提取的位域并写 `0x11824c` 和 `0x118250`。`memcfg_apply_poll (0x7a64)` 只在 `0x11824c` 位 0 被设置时运行，然后轮询 `0x136600`、`0x136e00` 和 `0x136400`，超时 100000（错误 `0xa6`）。`memcfg_timing_program (0x7c65)` 根据从 `0x137178` 和 `0x136604` 导出的基常量 `0x32a` 计算缩放的时序和带宽值，超时 125000（错误 `0xa7`）。`const_out_write (0x797a)` 提供固定常量 `0x68`、`0x555`、`0x5be`、`0x5a0`。
- **目标 Falcon 启动。** `tgt_falcon_init_reset (0x9da)` 写 `0x3f0c = 0xa0100`，用八个 `0xfeed0000` 与索引相或的字填充 `0x3f40` 处的寄存器组，然后以 `0x3f00 = 3` 和 `0x104 = 1` 收尾。`mailbox_write_d000 (0xb28)` 轮询 `0xd000` 的位 `0x1c0000`，把数据写到 `0xd200`、命令写到 `0xd100`。`tgt_falcon_handshake (0xbc9)` 验证 `0xbadf0000` 哨兵和 `0x3f20`..`0x3f40` 范围，错误 `0x38`。

`booter_load_wpr_main` 的 finalize 尾部为 GSP RISC-V 交接武装：`0x286a` 处 booter 通过 `0x1b44` 设置 `SECURE_SCRATCH_14`（`0x001180f8`）的位 24，然后 `0x2874` 调用 `reg_init (0x68ed)`，后者写 GSP `FBIF_CTL 0x00110624 = 0x90`（ALLOW_PHYS_NO_CTX 位 7 加位 4）、`0x00110684 = 1` 和 `0x0011126c = 1`。任何打算交接给 GSP-RM 的链条都必须让这段运行或复现这三个写入。`0x110684` 和 `0x11126c` 的字段名是推断的，未经头文件确认。

---

## 8. DMEM 映射

这里所有地址都是 **DMEM**。`0x100` 以下完全没有分配，这就是为什么"在低 DMEM 分阶段布置 mega-ROP"被排除。

| DMEM | 内容 | 备注 |
|---|---|---|
| `0x0000`-`0x00FF` | 未分配 | |
| `0x0200` | booter 自己的 HS 签名 | 加载前补进镜像（`hsSigDmemAddr = patchLoc - dataOffset`，`0x8900 - 0x8700`）。**不是**溢出的那个缓冲区。 |
| `0x0530` | DMA/引擎配置描述符结构 | |
| `0x0600`-`0x06FF` | `WprMeta`，256 字节结构 | +0 处魔数 `0x371a60b3`，+4 处 `0xdc3aae21`。在 DMA 目标之下，溢出从不触及它。 |
| `0x0700` | 镜像描述符 | `ls_sig_verify` 要求 `r10 == 0x700` |
| **`0x0800`** | **GSP-RM LS 签名缓冲区** | **DMA 目标。漏洞利用。** |
| `0x103c` 起 | 密码学会话描述符 | 字段 `0x1004`、`0x107c`、`0x1080`、`0x1100` |
| `0x1900` | `f100` 字段保存槽 | 正式载荷在此植入 `0x00000007` |
| `0x1904` / `0x190c` / `0x1914` | `va_to_pa_walk` 的 PTE 缓存 | |
| `0x1a00` / `0x2a00` / `0x3a00` | 页缓冲区 | |
| `0x2383` | 寄存器描述符表 | 被 `0xF800` 载荷砸掉；错误 `0x35` 的来源 |
| `0x5f00` | 固件请求头 | `'FREE'` / `'HEAP'` 魔数 |
| `0x6330`-`0x633F` | 阶段 9 扫描的 scratch | |
| **`0x6340`** | **栈金丝雀防护全局变量** | 十进制 25408 |
| `0x8700` | booter 代码/数据末尾 | |
| `0x8e08` | 寄存器描述符表 | 也被砸掉 |
| 约 `0xFF3C`-`0xFFFF` | 活动调用栈 | 从顶部向下增长 |

### 栈金丝雀

每次启动都生成新的随机值，保存在 DMEM `0x6340` 的全局变量中。每个受保护函数把它复制到栈帧边界，退出时重新读取并比较；不匹配就调用 `0x7dd9` 的 `panic()`。规范前导是 `mov $rX 0x6340; ld b32 $r9 D[$rX]`；尾声是 `cmp b32 $r15 $r9; bra e <ok>; lcall 0x7dd9`。

因为该值每次启动都由硬件 RNG 重新生成，无法离线猜测。也不需要。防护全局位于可写数据内存中，正处在它要检测的同一溢出可达范围内，因此载荷用同一个选定的值覆盖全局变量**和**每个重建的金丝雀槽，每次尾声比较都通过。这就是已发表论文的论题 1：Falcon 栈金丝雀在*引用字完整性*上失败，而不是熵。在这个镜像中，工具链把防护发射在只读数据节的尾部，而在扁平 MPU 映射镜像中它位于可写数据跨度内。没有 RELRO 等价物、没有防护页、也没有 MPU 只读映射。

---

## 9. BAR0 主控、CSB 纪律与邮箱

### 9.1 BAR0 主控

Falcon 只能通过 Falcon CSB 空间中一个带互斥锁的间接邮箱到达外部 BAR0 寄存器。没有内存映射的"直接"路径。

| CSB 端口 | 作用 |
|---|---|
| `I[0x1c100]` | 目标 PRI 地址（完整 32 位） |
| `I[0x1c200]` | 数据。对于读，结果回到这里。 |
| `I[0x1c000]` | 命令：**`0x800000f2` = 写，`0x800000f1` = 读** |
| `I[0x1c300]` | 看门狗，由 `watchdog_set (0x1034)` 以 `0x1312d00`（20,000,000）播种 |

Booter 用这条路径做自己的活：`0x29b8 -> 0x10aa` 写 WPR2 寄存器，finalize 例程对 `0x001180f8` 的写入字面是 `1c4b: r10=0x1180f8 ; lcall 0x10aa`。对应读出现在 `1c35: r10=0x1180f8 ; lcall 0x1196`。

### 9.2 失败关闭的 CSB 访问

**Booter 中每次 CSB 访问都是失败关闭的。** 每次访问后代码采样 CSB 错误状态 `I[0x9100]` 位 31，即 `FALCON_CSBERRSTAT.VALID`，一个**故障标志**，意思是上一次 CSB 访问出错，**不是**忙或完成轮询。原始内联前导在出错时分支到自己，永久卡死 Falcon；两个辅助函数则报告状态 `0x15` 并退出。

```asm
mov  rX 0x9100
iords
shr  0x1f
bra
self-lbra
```

这个惯用法内联出现约 25 次，加在两个辅助函数中。`csb_read (0x8264)` 另外把返回数据与 `0xffff0000` 相与，测试 PRI 毒值哨兵 `0xbadf0000`，带白名单（`0x208c` 处 `reg_whitelist_40f00`，覆盖 `[0x40f00, 0x41f00)` 步进 `0x100`），并为寄存器 `0x1c200`、`0xc00`、`0xb00` 和 `0xd500` 提供重试路径。

### 9.3 MAILBOX0 语义

Falcon I/O `0x1000` 是 Falcon 自己的 MAILBOX0，在 BAR0 `0x00840040` 主机可见，MAILBOX1 在 `0x00840044`（`CSB 0x1000 / 64 = falcon 0x40`）。主机在 PL0 直接读 BAR0 `0x1000` 返回 `0xbadf5040`，这解决了一个长期困惑。

MAILBOX0 是利用期间唯一可观察的通道，统一规则是：

> [!NOTE]
> **在穿过 `report_status` 的任何返回路径上，MAILBOX0 等于 `$r0`**
>
> MAILBOX0 读出 `0x31` 只意味着 `report_status` 从未执行。Booter 自己在 ucode 偏移 `0x7a` 处印上 `0x31`（`mov $r15 0x31 / mov $r9 0x1000 / iowrs I[$r9] $r15`）作为它第一个活性标记，覆盖驱动植入的 WprMeta 物理地址参数。

实测：返回地址 `0x8117`（裸退出，跳过 `report_status`）给出 MB0 `0x31`；`0x810d` 在 `r0 = 0` 时给出 MB0 `0x0`，植入 `0xcafe` 时给出 `0xcafe`；`0x8d4` 给出 `0x0b`。

实用分类：`0x47` = 栈金丝雀检查失败，Falcon 在 `panic()` 自循环中；`0x31` = `report_status` 从未运行；`0x96` = 金丝雀完好地正常启动。

### 9.4 状态码

| 代码 | 来源 | 含义 |
|---|---|---|
| `0x01` | `antirollback_version 0x59c4` | 存储的版本超过候选 |
| `0x05` | `wpr_region_check 0x28ac` / `wpr_region_program 0x291e` | WPR 上限 < 基址，或空区域 |
| `0x11` | `pka_ready_check 0x580f` / `pka_status_check 0x5473` | |
| `0x15` | `csb_read` / `csb_write` / `mailbox_wait_ready` / `reg_read_indirect` | CSB/PRI 访问故障 |
| `0x1c` | 通用 | 参数错误 |
| `0x23` / `0x4e` | `verify_reg_bitlen` | |
| `0x29` | `check_1180f8_nibbles 0x1c75`（由 `0x80a5` 调用） | `0x001180f8` [31:28] 或 [23:20] 非零 |
| `0x2d` | `firmware_load_main` | |
| `0x31` | PC `0x7a` | 入口活性标记；未到达 `report_status` |
| `0x32` | `check_reg_4f00` | |
| `0x35` | `regtable_rw_indexed` | `0x2383`/`0x8E08` 处的 DMEM 描述符表读出零 |
| `0x38` | `tgt_falcon_handshake 0xbc9` | |
| `0x47` | `__stack_chk_fail 0x7dd9` | 金丝雀不匹配，然后挂起 |
| `0x4b` | `chipid_gate 0x6a71` | strap `0x171` 无 `0x10200` 位 20 |
| `0x54` | 未知 | 只在 PG199 板上观察到。见下方。 |
| `0x59` | 驱动侧 | `dmem.bin` 缺失。良性。 |
| `0x5c` | `antirollback_version` | 孔径检查 |
| `0x62` | PKA 路径 | |
| `0x63`-`0x6d` | `pka_modexp_run 0x54ab` | `0x6c` = 超时 |
| `0x6e` | `check_10200_820434` | |
| `0x74` | `check_reg_118128` | |
| `0x88` | `check_1180f8_2724 0x1ba3` | `0x001180f8[27:24]` 非零 |
| `0x89`-`0x90` | `wpr_desc_validate 0x154a` | `0x8e`/`0x8f` 是 `0x1ffff` 对齐和 `0xfff` 字段检查 |
| `0x96` | 正常 | 金丝雀完好地启动 |
| `0x98` | `ls_sig_verify 0x399a` / `booter_load_wpr_main` | |
| `0x9c` / `0xa4` | `booter_load_wpr_main` | |
| `0x9e` | `range_validate_windows` | |
| `0x9f` | `hw_state_gate`、`dma_region_lock_setup` | |
| `0xa5` | `firmware_load_main` | |
| `0xa6` / `0xa7` | memcfg 路径 | |

主机侧作对比：`NV_ERR_TIMEOUT = 0x00000065`、`NV_ERR_MEMORY_ERROR = 0x72`、`NV_ERR_GENERIC = 0xffff`。观察到的复合 `RmInitAdapter` 失败包括 `0x62:0x40:2028`、`0x62:0x55` 和 `0x62:0x65:2674`。

> [!NOTE]
> **开放问题：Booter 状态 `0x54`**
>
> 把修改过的 cmpunlocker 应用到 PG199 板失败，报 `s_executeBooterUcode_TU102: Booter failed 0x54`，尽管 CFG1 和 LMR 写入落盘、PLM 已打开。其他每个状态码都是通过在反汇编中定位其写入点钉死的；`0x54` 应该用同样的方法，反汇编已在手。

---

## 10. 离开重型安全模式与复位 PLM {#10-leaving-heavy-secure-mode-and-the-reset-plm}

IMEM `0x7e76` 的 `secure_teardown` 是设计的出口。它重新启用 `0x10100` DMA 孔径（OR `0x101`，自旋直到位 `0x100` 清除，上限 `0x400` 次迭代），设置 `$cauth |= 0x80000`（位 19，在停机前抑制中断和异常），然后对 `$c0`..`$c7` 中的每一个发出 `csecret $cN 0x0` 后跟 `cxor $cN $cN`，从 0 到 `0x10000` 循环 `st b32 D[$r9] $r14; add $r9 0x4`（`r14 = 0`），清除 `r0`..`r15`，并执行裸 `exit` 操作码（`f8 02`）。它从不返回。正是那个 `exit` 让 Falcon 掉出 HS 模式，允许加载新代码。

从 `main` 到达它需要 `0x8113` 处 `r0 == 0` 分支到 `0x8119`。如果 `r0` 非零，走 `0x8117 exit`，完全没有 teardown。

**错误路径总是按顺序先调用 `report_status (0x1d0f)` 再调用 `secure_teardown (0x7e76)`**：该对出现在 `0x873`/`0x877`、`0x88a`/`0x88e` 和 `0x8a7`/`0x8ab`。成功路径两者都不调用，为 GSP 交接保留密码学和环境。因此流传的"邮箱 XOR teardown"框架是错的：错误路径两者都做。

### 复位 PLM `0x008403C4`

SEC2 复位源 PLM 门控 `+0x3c0` 处的 SEC2 `FALCON_ENGINE` 复位控制。它触发后的值决定 SEC2 能否再次复位。

| 值 | 含义 |
|---|---|
| `0xff` | 完全开放。干净空闲、SBR 后、SEC2 未使用。主机 PL0 `kflcnReset` 会生效。 |
| `0xdf` | 原厂驱动在其 GSP 启动 teardown 后留下的正常工作状态。仍允许复位。 |
| `0xcf` | 在驱动的 GSP-prime 重新锁定 PLM 后观察到（位 4 清除）。 |
| `0x8f` | HS 退出污染。低半字节 `0xf` = 所有级别可读；高半字节 `0x8` = 写锁定到安全源，因此 PL0 复位写入被弹回。 |

规则：`reset_allowed = resetPLM in {0xff, 0xdf}`。`0xdf = 0x8f | 0x50`。

关键的是，`0x8f` 在 **HS 到 NS 退出转换时由硬件闩锁**，不是任何 booter 指令写入的：静态分析在 booter 中找不到对 `0x8403C4` 的零条指令引用。离开 HS 会让硬件把每个 HS 门控的 PLM 重新保护为安全默认值。实测：在 `0x8117` 走裸退出留下 resetPLM `0xff`；让 `secure_teardown` 运行会把它重新闩锁为 `0x8f`。

> [!NOTE]
> **开放问题：`resetPLM = 0x8f` 会阻止加载新 SEC2 ucode 吗？**
>
> 一份报告说 SEC2 在 `0x8f` 下可重载（Hello World 触发了，MAILBOX0 从 `0x0` 变为 `0x31`）；另一份说加载新 ucode 需要 SFTRESET，而 SFTRESET 由复位 PLM 门控，报告 `NS load mismatch (HS-locked, needs --flr)`。可能的调和——NS 重载可行而 HS 签名重载不可行——被提出但从未定案。一个受控实验：在已知 `0x8f` 下背靠背加载一个 NS ucode 和一个 HS 签名 ucode，记录每次的 `CPUCTL` 和加载器错误字符串，就能回答。

正式驱动完全绕开这套纪律：它**从不读写 `0x008403C4`**。grep 正式仓库找不到对 `0x008403c4`、`0x001180f8`、`0x001fa81c` 或 `0x001fa820` 的任何引用。驱动内路径改经驱动自己的 `kflcnReset`/FWSEC 序列重新触发 Booter Load，补丁 0002 通过记录 `SEC2_DEBUG: kflcnReset for FWSEC: 0x%x` 和 `SEC2_DEBUG: kflcnResetIntoRiscv: 0x%x` 确认这一点。

---

## 11. 签名缓冲区

这是整个解锁围绕的对象。

| 属性 | 原厂 | 解锁下 |
|---|---|---|
| 分配 | `NV_ALIGN_UP(pGspFw->signatureSize, 256)`；观察到 4,096 字节 | `SEC2_POSTBL_TIMING_SIGNATURE_SIZE = 0x0000f800ULL` = 63,488 字节 |
| 对齐 | 256 字节，`ADDR_SYSMEM` | 256 字节，`ADDR_SYSMEM` |
| DMEM 目标 | `0x0800` | `0x0800` |
| DMEM 覆盖 | `0x17FF` | `0xFFFF`，恰好 DMEM 顶部 |
| 长度来源 | `WprMeta.sizeOfSignature` | `WprMeta.sizeOfSignature` |

Booter 从 `WprMeta.sizeOfSignature` 逐字取复制长度，没有任何形式的边界检查，而驱动同时控制缓冲区内容和该字段。扩大因子是 15.5。原厂签名的 DMA 只到达 DMEM `0x17FF`，这就是正常启动保持 DMEM `0x2383` 和 `0x8E08` 处寄存器描述符表完好的原因。

DMA 目标 `0x800` 由 IMEM `0x37ad` 处的 `mov $r10 0x800` 设置，后面是 `0x37b3` 处的 `lcall 0x4d4`。接下来发生什么见 [ROP 链](rop-chain.md)。

> [!NOTE]
> **不冲突**
>
> `kernel_gsp_booter.c:329` 计算 `pUcode->hsSigDmemAddr = patchLoc - pUcode->dataOffset`，在 `patchLoc = 0x8900`、`dataOffset = 0x8700` 时把签名放在 DMEM `0x200`。那是 **booter 自己的 HS 签名**，加载前补进 booter 镜像。DMEM `0x800` 是 booter 从 sysmem DMA 的 **GSP-RM LS 签名**落点，那才是溢出的。两个不同的缓冲区。置信度：中等，此调和与每个观察一致，但没有人明确陈述过。

还有一个属性对解锁的持久性叙述很重要：**原厂 AES-MAC 签名在几何改变后仍然有效**，因为它覆盖静态 GSP 固件镜像（静止时），不覆盖运行时 WPR 元数据或硬件几何。WPR 元数据由驱动在运行时计算。早先相反的声明被其作者明确收回。

---

## 12. 正式驱动如何调用 Booter Load

本节的一切都直接读自正式 `master` 的 `driver/patches/0001-sec2-postbl-plm-ss-cfg.patch` 和 `0002-booter-verify.patch`。完整补丁集见 [驱动补丁](driver-patches.md)。

### 12.1 门控

```c
#define SEC2_POSTBL_TIMING_CMP_170HX_8GB_PCI_DEVICE_ID   0x20C2
#define SEC2_POSTBL_TIMING_CMP_170HX_10GB_PCI_DEVICE_ID  0x2082
```

`_kgspSec2PostblTimingEnabled()` 用 `pGpu->idInfo.PCIDeviceID >> 16` 对照恰好这两个值测试。`10de:20b0` 卡（`install.sh` 也会 grep 到）安装但不解锁。目标驱动版本是 `610.43.03`（默认）和 `610.43.02`；其他任何版本构建硬性失败。

### 12.2 顺序序列

1. **`_kgspCreateSignatureMemdesc`** 以 `0x0000f800` 而不是 `NV_ALIGN_UP(pGspFw->signatureSize, 256)` 分配签名 memdesc。在改作它用之前，原厂签名字节被复制到 `pKernelGsp->pStockSignatureData` / `stockSignatureSize`，这是添加到 `g_kernel_gsp_nvoc.h` 中 `KernelGsp` 的两个新字段。记录 `SEC2_DEBUG: saved stock signature (%llu bytes)`，卡上报告 4096。
2. **可选外部载荷。** `os_open_and_read_file()` 尝试把 `SEC2_POSTBL_TIMING_DMEM_PATH = "/lib/firmware/nvidia/ga100/gsp/dmem.bin"` 读进新缓冲区。成功记录 `SEC2_DEBUG: loaded %llu bytes from %s`；缺失记录 `SEC2_DEBUG: %s not found (0x%x), using built-in payload`，带 `0x59`，并回退到预填 `writeAddr = 0x009a0148`、`writeValue = 0xffffffff` 的内置载荷。无论哪种方式都调用 `memdescFlushCpuCaches()`。
3. **保存 WPR2。** 读取一次 `0x001fa824`（低）和 `0x001fa828`（高），记录 `SEC2_DEBUG: saved WPR2 lo=0x%08x hi=0x%08x`。
4. **PLM 循环。** 对四个表条目中的每一个，最多尝试两次：

    ```c
    for (plmIdx = 0; plmIdx < 4; plmIdx++)
        for (attempt = 0; attempt < 2 && !opened; attempt++) {
            GPU_REG_WR32(pGpu, 0x001fa824, savedWpr2Lo);
            GPU_REG_WR32(pGpu, 0x001fa828, savedWpr2Hi);
            kgspSec2PostblTimingRefillPayload(pGpu, pKernelGsp,
                                              plmTable[plmIdx].addr,
                                              plmTable[plmIdx].value);
            kgspExecuteBooterLoad_HAL(pGpu, pKernelGsp,
                memdescGetPhysAddr(pKernelGsp->pWprMetaDescriptor, AT_GPU, 0));
            regVal = GPU_REG_RD32(pGpu, plmTable[plmIdx].addr);
            opened = (regVal == plmTable[plmIdx].value);
        }
    ```

    失败记录 `SEC2_DEBUG: FAILED to open <name> after 2 attempts`。表及其精确值在 [权限级掩码](privilege-level-masks.md)。

5. **循环后再次恢复 WPR2。**
6. **PL0 下四个普通主机写入**，不再需要漏洞利用：`0x0082381c = 0x88888888`（SS0）、`0x00823820 = 0x00000008`（SS1）、`0x009a0204 = cfg1Value`、`0x00100ce0 = lmrValue`。然后 `SEC2_DEBUG: POST-WRITE SS0=… SS1=… CFG1=… LMR=…`。见 [计算节流](compute-throttle.md) 和 [显存几何](memory-geometry.md)。
7. **`kgspSec2PostblTimingRebuildStockSignature()`** 释放并销毁 `0xf800` memdesc，以 `MEMDESC_FLAGS_ALLOC_IN_UNPROTECTED_MEMORY` 分配一个 `NV_ALIGN_UP(stockSignatureSize, 256)` 的替代品，把 `pStockSignatureData` 复制回去，并重新指向 `pWprMeta->sysmemAddrOfSignature` / `sizeOfSignature`。失败以 `SEC2_DEBUG: rebuild stock signature failed: 0x%x` 中止启动。
8. **`kgspPopulateWprMeta_HAL` 第二次运行**，使 WPR 元数据反映扩大的 FB。卡上 dmesg 显示 `WPR meta updated fbSize=0x0000001000000000 …`，紧接着是 `normal BooterLoad status=0x0`。

这就是 GSP-RM 在*同一次*驱动加载中正常启动的原因：漏洞利用和真正启动在同一次加载内顺序发生，不需要冷启动等价交接。

### 12.3 重填辅助函数

`kgspSec2PostblTimingRefillPayload(pGpu, pKernelGsp, writeAddr, writeValue)` 用 `memdescMapInternal(..., TRANSFER_FLAGS_NONE)` 映射 memdesc，为那一对 `(writeAddr, writeValue)` 重写整个 `0xf800` 字节载荷，取消映射，在签名 memdesc 上调用 `memdescFlushCpuCaches()`，重新发布 `pWprMeta->sysmemAddrOfSignature = memdescGetPhysAddr(...)` 和 `pWprMeta->sizeOfSignature = memdescGetSize(...)`，然后冲刷 `pWprMetaDescriptor` 上的 CPU 缓存。memdesc 为 NULL 时返回 `NV_ERR_INVALID_STATE`，映射失败时返回 `NV_ERR_INSUFFICIENT_RESOURCES`。把 `0xf800` 的签名长度交给 booter **就是**溢出。

缓存冲刷不是可选的。签名 DMA 是非一致的，没有显式冲刷，Falcon 会读到过期的 RAM。

### 12.4 补丁做出的两项让步

- **WPR2 已 up。** 致命路径

    ```c
    NV_PRINTF(LEVEL_ERROR, "unexpected WPR2 already up, cannot proceed with booting GSP\n");
    return NV_ERR_INVALID_STATE;
    ```

    变成 `NV_PRINTF(LEVEL_WARNING, "WPR2 already up before GSP boot; continuing for recovery\n")`。重复触发 booter 会让 WPR2 保持 up，每次触发都重新划分它。空/INIT 状态是 LO `0x0fffffff`、HI `0`，**HI = 0 使 `kgspIsWpr2Up()` 返回 false**。正式驱动恢复*保存的*对，而不是写空区域。

- **`0002-booter-verify.patch`** 把 `kgspBootstrap_TU102` 中的若干 `NV_ASSERT_OK_OR_RETURN` 点转换为记录日志的状态检查，并为设备 ID `0x20C2` / `0x2082` 增加五个寄存器的 BooterLoad 后回读：

    ```c
    #define SEC2_DEBUG_PRI_FEATURE_OVERRIDE_PLM        0x00823804
    #define SEC2_DEBUG_PRI_FEATURE_OVERRIDE_SM_SPEED   0x0082381c
    #define SEC2_DEBUG_PRI_FEATURE_OVERRIDE_SM_SPEED_1 0x00823820
    #define SEC2_DEBUG_PRI_FBPA_CFG1                   0x009a0204
    #define SEC2_DEBUG_PRI_MMU_LMR                     0x00100ce0
    ```

### 12.5 Booter 运行多少次 {#125-how-many-times-the-booter-runs}

| 情况 | Booter Load 触发次数 |
|---|---|
| 每个 PLM 首次尝试即打开 | 4 次漏洞利用触发 + 1 次正常启动 = **5** |
| 每个 PLM 都需要两次尝试 | 8 次漏洞利用触发 + 1 次正常启动 = **9** |

每次触发恰好执行**一次**任意 BAR0 写入。

### 12.6 读日志

> [!WARNING]
> **寄存器回读是唯一有效的成功标准**
>
> 每次载荷执行都记录 `s_executeBooterUcode_TU102: Booter failed with non-zero error code: 0x31` 和 `kgspExecuteBooterLoad_TU102: failed to execute Booter Load: 0xffff`，**而寄存器写入仍然落盘**。在 `s_executeBooterUcode_TU102` 中，seccode 错误在每次运行后位于 MAILBOX0，`mailbox0 != 0` 返回 `NV_ERR_GENERIC`（`0xffff`）。正式循环的成功测试是精确回读相等，这是正确的测试。项目 README 说同样的话：早期 PLM 轮次中 `0x31` / `0xffff` 等 Booter 状态码如果最终启动成功，往往无害。

见 [验证](../procedures/verify.md) 和 [故障排查](../procedures/troubleshooting.md)。

## 13. 无驱动调用（对比）

一族独立的 Python 和 C 工具（`refire_chain_v2.py` 到 `v9.py`、`load_gsp_sec2_falcon.c`、`load_custom_bin.py`）在没有 NVIDIA 驱动的情况下触发 booter。它们**不在**正式仓库中，这个领域里大多数表面矛盾只要把两套代码分开就化解了。这些工具重要，因为大部分寄存器纪律是在这里学会的。见 [工具谱系](../history/tool-lineage.md)。

在 SEC2 上运行非安全代码很简单，除了公开文档外不需要任何东西：把二进制 DMA 进 IMEM，设置 `BOOTVEC`，发出 `STARTCPU`，然后如果代码用 `exit` 就轮询 `CPUCTL` 位 4（HALTED）。完整无驱动启动序列是：引擎复位、等待 DMA scrub、`ALLOW_PHYS_NO_CTX`、物理 DMA 孔径、DMA IMEM + DMEM、设置 `BOOTVEC`、设置邮箱、`STARTCPU`、轮询 HALTED。

已实现并工作的复位序列：

```text
if SCTL (SEC2+0x240) has HSMODE (bit 1) set:
    write SFTRESET (SEC2+0x07c) = 1 and read back
pulse ENGINE (SEC2+0x3c0): 1 then 0
poll DMACTL (SEC2+0x10c) until scrub bits 0x6 clear, ignoring 0xffffffff reads
poll SCP_P2PRX (SEC2+0x530) bit 3, with KFUSE_CTL (SEC2+0x11ec) bit 0 set and bit 1 clear
OR AUTH_EN (1 << 14) into SCTL
```

默认超时 10.0 s；失败路径报告 "scrub timeout"。Booter 加载随后使用带自动递增的 IMEMC/IMEMD 和每 256 字节的 IMEM tag，HS 区域设置 SECURE 位 `1 << 28`。孔径通过写 `0x00840600`/`0x604`/`0x608` 处的 `FBIF_TRANSCFG[0..2] = 4, 5, 6` 然后写 `0x00840624` 处的 `FBIF_CTL |= 0x80` 强制为物理模式，之后 `start_wait`，MAILBOX0/1 设置为 WprMeta 物理地址的低和高半。

一个独立 C 程序在硬件上执行了完整九步无驱动启动，从 `FALCON_MAILBOX0` 读出停机返回值 `0xb`：即使 Falcon 以非零状态停机，加载路径在没有 NVIDIA 驱动的情况下也能工作。

无驱动路径还发现两个进一步要求，正式驱动都以其他方式满足：

- **缓存冲刷。** 一个 17 字节 JIT 汇编的 x86-64 桩（`0F AE 3F 48 83 C7 40 48 83 EE 40 7F F3 0F AE F0 C3`，即 `clflush [rdi]` / `add rdi,64` / `sub rsi,64` / `jg` / `mfence` / `ret`）被映射为 `PROT_EXEC` 并在载荷、radix3 和 WprMeta 缓冲区上运行，向上取整到 64 字节缓存行。`refire_chain_v6.py` 用 `MAP_HUGETLB`（`0x40000`）分配 2 MiB 大页、mlock 它们，并通过检查 present 位后 `(entry & ((1<<55)-1)) * 4096` 从 `/proc/self/pagemap` 解析物理地址。
- **一张最小 radix3 页表。** `stage_radix3()` 分配 `0x6000` 字节并写三个 64 位描述符（PDE2 在 `0x0000` 到 `phys+0x1000`，PDE1 在 `0x1000` 到 `phys+0x2000`，PDE0 在 `0x2000` 到 `phys+0x3000`），数据页和 bootloader 主体留零，然后冲刷。没有它，booter 的签名前 DMA 会以原因 `0x9` 失败。WprMeta 模板是从一次真实 10 GB 启动捕获的 256 字节，只覆盖 radix3 指针（`+0x10`）、radix3 大小（`+0x18`）、bootloader 指针（`+0x20`）、bootloader 大小（`+0x28`）、签名指针（`+0x48`）和签名大小（`+0x50`，设为 `0xF800`）。

注意两条路径的邮箱语义也不同：独立加载器以 5 s 超时轮询 HALTED，然后读 `0x840040`，期望 `0x31` / `0x96` / `0x47`；驱动内路径无论结果如何都报告 `0xffff`。

---

## 14. 本页的开放问题

> [!NOTE]
> **无驱动触发能交接给原厂驱动吗？**
>
> 原厂驱动的 booter 通过经典两加载"互斥锁犄角"拒绝触发后的 SEC2 状态：`0x31`（互斥锁被持有）、`0x62`（WPR2 up）和 `0x29`（`0x001180f8` 错误，因为 `mutexfree` 终结符留下 `0xf0000000`，`0xf` 高半字节触发检查）。这在几何一致、甚至 10 GB 下也失败，证明被触发扰动的是 SEC2 / `0x001180f8` 交接状态，不是几何也不是写入次数。提出了两个修复：让终结符把 `0x001180f8` 高半字节留零，或从补丁驱动内部分阶段布置几何。**正式解锁器选了第二个。**

> [!NOTE]
> **不 FLR 地跨越 RmInitDone 之墙**
>
> `whole_stack_rejoin` 终结符在漏洞利用后不 FLR 地重启 SEC2，能让 booter 完成、GSP-RM RISC-V 核心启动，但 init 从不完成。这是同一个 `0x65` 启动之墙。`0x001180f8` 是 `NV_PGC6_BSI_SECURE_SCRATCH_14`，位 26 是 `BOOT_STAGE_3_HANDOFF`（INIT = 0，DONE = 1），只有 HS 下的 SEC2 设置它。预写 DONE 没有帮助：读路径被 PLM 毒化，`0x001180f8` 读回 `0xdead5ec1`，在毒化读上位 26 已读作 1，产生假 DONE，反而稍后杀死 GSP-RM。两个候选根修复是保留 booter 成功路径让 SEC2 启动其 RTOS 并自己设置 DONE，或恢复 AON `SECURE_SCRATCH` PLM/priv 状态——后者今天只有电源域复位能做到。

> [!NOTE]
> **移植到其他 CMP 卡**
>
> CMP 50HX 是 TU102，使用完全不同的内存访问控制寄存器组。CMP 90HX 是带 10 GB GDDR6X、没有额外物理内存的 GA102，因此只有计算解锁有意义。既定规则是，同一 Turing booter、脚本和漏洞利用适用于任何 SEC2 接受 Turing 代 AES 和 RSA 密钥的卡。一位测试者报告 TU10x `booter_load` 在 GA102 CMP 90HX 上加载、SS0/SS1 PLM 写入成功，同时自我限定说写入的值"不对"，并警告一个阳性测试不够。另一份 GA102 booter 的静态分析得出结论没有溢出点，因为大小被严格验证。没人运行决定性测试：在 GA102 上加载 TU10x booter，尝试一次已知良好的单 PLM 写入并回读。

> [!NOTE]
> **Windows 和非 Linux**
>
> 漏洞在 GPU 固件中，与操作系统无关。当前实现是 Linux，Unix 主机和开源驱动都不是硬性要求，但 Windows 移植被描述为远超几行代码的工作。

> [!NOTE]
> **恢复 `csecret`**
>
> 三个索引映射到三个能力：`secret(6)` 解密 ECB 固件 blob（将产出 121.7 KB 明文固件加 Booter 代码）；`secret(2)` 伪造内容 MAC（经该路线做 CFG1 显存解锁和 PCIe 速率解锁的前提）；`secret(0)` 是启用带 `SKIP_VBIOS_SIG` 的 HULK 证书的调试旁路。**没有恢复任何 csecret。** 三个仍然是需要电压毛刺硬件的差分故障分析目标。没有 **Booter 解密密钥**，加密 booter 无法重建（只能经调试密钥路线读取）；没有 **VBIOS 调试密钥**，VBIOS 无法重新签名或调试模式运行。当前解锁通过复用原厂签名 booter 作为执行引擎绕开两者。

> [!NOTE]
> **同一 bug 类的第二个实例**
>
> 论文（第 5.5 节）指出 GSP-RM 自己的常驻 blob 携带同一 bug 类的第二个实例，那里的防护全局是**公开硬编码常量**而不是 RNG 播种。没有公布地址、没有基于它构建漏洞利用，存档中也无人验证。

### 已记录的负面结果

- **`envytools` 无法佐证以上任何一点。** 它的 Falcon 密码学页有 Introduction、IO registers、Interrupts、"Submitting crypto commands: ccmd"、"Code authentication control" 和 "Crypto xfer control" 的小节标题，**每一节都标着"Todo: write me"**。没有关于 AES 引擎、密钥处理、签名代码认证、安全模式进入或退出、代码页签名检查或任何 CMAC/CBC-MAC 方案的文档。记录于此以免有人重新搜索。envytools 也只把 Falcon 硬件记录到 v5（GK208+），没有 Ampere 或 GA100 覆盖；它的寄存器映射（`UC_CTRL 0x100`、`UC_ENTRY 0x104`、`UC_CAPS 0x108`、`UC_STATUS 0x128`、`CODE_INDEX 0x180`、`CODE 0x184`、`DATA_INDEX[0-7] 0x1c0`、`SCRATCH0 0x040`）只是结构背景，**不得**用来验证 GA100 寄存器地址。
- **通过返回到 IMEM `0x100` 的 `_start` 连续两个签名地重新进入 booter** 已测试，以邮箱中同样的 `0x31` 失败。Falcon 进入 HS 模式时 `0x00` 处的轻安全引导被抹掉，因此没有可返回的东西。
- **跳过 `secure_teardown` 收割活 SCP 机密**在提出当天就被两份对抗性逐字节静态追踪驳斥：`0x107`-`0x147` 的前导立即把每个机密自 XOR 为零，真正的密钥使用是 `0x1e20`-`0x1e70` 的 AES 验证，`0x1e74`-`0x206e` 的清扫扫描连续三轮自零传递，最后一个密码学操作是 `0x206e cxor $c0, $c0`。从 `0x2070` 到 `0x7eef` **零**密码学操作，劫持点（`0x37b3` 的 `lcall 0x4d4`）正好坐在那段密码学静默缺口内。跳过省不了什么，因为大约再往前 0x1500 字节代码处寄存器组就已经空了。
- **逆向 booter 获得 HS 签名权限**被其提出者自己放弃。即使从硅片提取 AES 密钥，RSA 私钥仍然缺失：裸片上只存公钥。剩余理论路线是启用调试模式并用调试 RSA 私钥，但生产卡上有物理熔丝禁用调试模式，只有工程样品启用。
- **主机侧把 PCI 设备 ID 伪造成 A100 ID（`0x20b0`）** 行不通：VBIOS/devinit 在驱动或 GSP 有机会之前就按卡级设备 ID 取键，而且所有 GA100 卡甚至 Turing 卡都是同一个 booter，下游没有任何东西按主机 ID 分支。

---

## 相关页面

- [解锁的工作原理（端到端）](how-it-works.md)
- [ROP 链](rop-chain.md)
- [权限级掩码](privilege-level-masks.md)
- [驱动补丁](driver-patches.md)
- [寄存器参考](register-reference.md) 和 [寄存器索引](../appendix/register-index.md)
- [词汇表](../start/glossary.md)
- [死路](../history/dead-ends.md)
