# CMP 170HX 中的 GA100 晶片

**本页涵盖：** 物理硅片：制程、晶体管数、晶片尺寸与封装；GPC / TPC / SM 层级结构，以及 170HX 实际有多少 SM、A100 有多少；决定这个数字的筛选熔丝以及它们随晶片如何变化；内存分区与缓存层级；计算能力及其带来的指令集；以及哪些固定功能引擎存在、被熔断关闭、或物理上缺失。

一句话总结：**CMP 170HX 搭载与 NVIDIA A100 相同的 GA100 晶片**，TSMC 7 nm，542 亿晶体管，826 mm²。它是这颗晶片的筛选 bin，8 GB 卡上标记为 `GA100-105F-A1`，10 GB 卡上标记为 `GA100-105A-A1`，**8 个 GPC 中启用 5 个：35 个 TPC、70 个 SM、4480 条 FP32 通道，计算能力 8.0**。两个 SKU 都恰好枚举出 70 个 SM。CMP 的限制没有任何部分会减少 SM，[计算解锁](../unlock/compute-throttle.md) 也不会加回任何 SM：挖矿专属限制是存放在独立熔丝中的发射速率分频器，而你得到的 SM 数量就是这一 bin 的常规硅片熔丝下限。

每张 170HX 读取 `PMC_BOOT_0`（`0x00000000`）= `0x170000a1`，这是 GA100 的芯片 ID 签名。GA10x 消费级控制部件在同一偏移处读 `0xb74000a1`，所以这一个双字就足以作为"这真的是 GA100 吗"的检验。

---

## 晶片与封装

| 属性 | 值 | 备注 |
|---|---|---|
| 架构 | Ampere GA100 | 与 A100 同一颗晶片，不是 CMP 专用 tape-out |
| 制程 | TSMC 7 nm（N7） | |
| 晶体管数 | 542 亿 | 密度 65.6 M/mm² |
| 晶片面积 | 826 mm² | 处于或接近 7 nm 掩模版极限（约 830 mm²） |
| 封装 | BGA-2743，约 55 × 55 mm | 散热器螺栓孔位 57 × 68 mm（中心距） |
| ASIC 标记，8 GB | `GA100-105F-A1` | GPU 部件号 `20C2-105-A1` |
| ASIC 标记，10 GB | `GA100-105A-A1` | GPU 部件号 `2082-105-A1` |
| 零售 A100 对比标记 | `GA100-883AA-A1` | 与 170HX 晶片并排拍摄过 |
| `PMC_BOOT_0` `0x00000000` | `0x170000a1` | 在所有有效 GA100 上 |
| PCI 设备 ID | `10de:20c2`（8 GB）/ `10de:2082`（10 GB） | 见[板卡与型号变体](board-and-variants.md) |
| 计算能力 | 8.0（`sm_80`） | OpenCL 3.0，CUDA 11+ |
| 基础 / 睿频 SM 时钟 | 1140 MHz / 1410 MHz | `nvidia-smi -pl 300` 下实测 1470 MHz |
| TDP / 最大软件功耗上限 | 250 W / 原厂 VBIOS 下最大 250 W | `nvidia-smi -q` 报告最低 100 W、最高 250 W；只有搭载 NVIDIA OC 挖矿 VBIOS 的卡才是 300 W。DevCap 中插槽供电上限为 75 W |

PCB 是 A100 40 GB PCIe 参考设计并刻意删减了部分器件，所以板卡级缺失（VRM 相位、NVLink 接口芯片、PCIe 交流耦合电容）与晶片级缺失是两个独立话题。见[物理改造](../operations/physical-mods.md)和[供电](power-delivery.md)。

---

## GPC / TPC / SM 层级结构

GA100 的规模寄存器描述的是完整晶片，与熔断内容无关。它们在 170HX 与 A100 级参考部件上读值一致：

| 规模寄存器 | 地址 | 值 | 含义 |
|---|---|---|---|
| `PTOP_SCAL_NUM_GPCS` | `0x00022430` | `0x8` | 完整晶片 8 个 GPC |
| `PTOP_SCAL_NUM_TPC_GPC` | `0x00022434` | `0x8` | 每 GPC 8 个 TPC → 64 个 TPC → 完整晶片 128 个 SM |
| `PTOP_SCAL_NUM_FBPS` | `0x00022438` | `0xc` | 12 个 FBP |
| `PTOP_SCAL_NUM_FBPAS` | `0x0002243c` | `0x18` | 24 个 FBPA（GA102 RTX 3090 读 6） |
| `PTOP_SCAL_NUM_LTCS` | `0x00022454` | `24` | 24 个 L2 缓存切片 |
| `PTOP_SCAL_FBPA_PER_FBP` | `0x00022458` | `2` | 每 FBP 2 个 FBPA |
| `PTOP_SCAL_NUM_NVLINK` | `0x0002246c` | `12` | 晶片上 12 个 NVLink 单元 |
| `PTOP_FS_STATUS` | `0x00022470` | `0x0000003f` | 筛选状态 |

因此完整 GA100 应为 128 SM / 8192 条 FP32 通道。没有任何产品以该配置出货：每个 A100 SKU 出货 108 SM / 6912 个着色单元，CMP 170HX 出货 70。

| 部件 | 启用的 GPC | TPC | SM | FP32 通道 |
|---|---|---|---|---|
| 完整 GA100 晶片 | 8 | 64 | 128 | 8192 |
| A100 SXM4 / PCIe（所有 SKU） | 7（`FUSE_GPC_DISABLE` = `0x08` / `0x08` / `0x80`，探测的三个 SKU 各置一位） | 54 | 108 | 6912 |
| DRIVE A100（`0x20bb`，PG199） | 6（`FUSE_GPC_DISABLE` = `0x84`） | 48 | 96 | 6144 |
| **CMP 170HX，两个 SKU** | **5**（`RING_ENUM_GPC` = 5） | **35** | **70** | **4480** |

170HX 的数据是实测而非推断。一张解锁卡上的 PTX 特殊寄存器转储程序报告 `SMs=70 warpsize=32`、`nsmid=70 nwarpid=64`，`smid` 值覆盖 0..69 且无缺口。一个八卡混合机器在所有八个设备上都枚举出 `cu: 70`，而显存容量在 9990 MiB 与 7954 MiB 之间交替——这是"SM 数量不随 SKU 变化"最干净的一次单独演示。解锁卡上的 OpenCL-Benchmark 同样报告 70 个计算单元、1410 MHz（4480 核心，理论 FP32 12.634 TFLOP/s）。

注意 35 个 TPC 是奇数：五个启用的 GPC 并非都带 8 个 TPC，这对筛选部件来说是正常的。

---

## 筛选与逐晶片分 bin

筛选（floorsweeping）就是在测试时熔断 OTP 熔丝以禁用有缺陷或多余的单元。在 170HX 上它完全通过熔丝和 STATUS 路径工作：**每个 CTRL_OPT 覆盖寄存器都读零**，拓扑报告自己的汇总行是 `held back by CTRL_OPT: 0 TPC = 0 SM`。这意味着 70 这个 SM 数已经是这些晶片的熔丝下限，其上方不存在可以放松的软件覆盖层。

| 寄存器 | 地址 | 作用 | 170HX 读数 |
|---|---|---|---|
| `OPT_GPC_DISABLE` | `0x00820350` | GPC 禁用掩码（熔丝） | 逐晶片，置 3 位 |
| `STATUS_OPT_GPC` | `0x00820c1c` | 生效的 GPC 掩码 | 总是镜像熔丝 |
| `OPT_GPC_DEFECTIVE` | `0x008205c4` | 哪些 GPC 真正有缺陷 | 部分卡为 `0x00` |
| `RING_ENUM_GPC` | `0x00120078` | 环上枚举出的 GPC | `5` |
| `CTRL_OPT_GPC` | `0x0082081c` | 软件筛选覆盖 | `0x00000000` |
| `CTRL_OPT_FBIO` / `_FBPA` / `_FBP` | `0x00820814` / `0x00820818` / `0x00820938` | 同上，内存侧 | `0x00000000` |
| `CTRL_OPT_PERLINK` / `_PCIE_LANE` / `_NVLINK` | `0x00820820` / `0x0082082c` / `0x008209b8` | 同上 | `0x00000000` |
| `FUSE_EN_SW_OVERRIDE` | `0x00820040` | 是否启用 CTRL_OPT 层 | `0x00000000` |
| `gpcMask` | `0x00408970` | GR 侧 GPC 掩码 | `0xdc`，写入后会重新断言 |
| `OPT_PCIE_DEVIDA` | `0x008204d8` | SKU 身份熔丝 | 一张 8 GB 卡报告 `0x20c2`（见下方备注） |
| `OPT_SLT_REV` | `0x008204bc` | 插槽 / 测试修订号 | 逐晶片 |

**掩码随每张卡而异，不随 SKU，也不随驱动版本。** 三次独立调查共记录了七个不同的 `OPT_GPC_DISABLE` 值，其中四个来自一个下午读出的四张卡，全部禁用 3 个 GPC、全部总计 70 个 SM：

| `OPT_GPC_DISABLE` | 禁用的 GPC | 卡 |
|---|---|---|
| `0x85` | 0、2、7 | 10 GB |
| `0x45` | 0、2、6 | 8 GB |
| `0x13` | 0、1、4 | 8 GB |
| `0xa8` | 3、5、7 | 10 GB |
| `0xd0` | 4、6、7 | 熔丝表卡 |
| `0x23` | 0、1、5 | 熔丝表卡 |
| `0x15` | 0、2、4 | 10 GB |

其中一些被禁用的 GPC 被标记为**物理完好**。用于高安全写入实验的 8 GB 卡上，`OPT_GPC_DEFECTIVE` 读 `0x00000000`，而 `OPT_GPC_DISABLE` 置了三个位——也就是说三个被禁用的 GPC 都是健康硅片，只是为了达到产品规格而被熔断关闭。在一张 10 GB 卡上，`OPT_GPC_DEFECTIVE` = `0x81`（GPC 0 和 7 真正有缺陷），而 GPC 2 是被禁用但没有缺陷。

### 限制 / 分 bin 的区分

对两张物理 170HX 卡做完整 120 寄存器 diff，发现 **107 个寄存器相同、13 个不同，而 13 个全部是分 bin 值**。这是这颗晶片上最有实操价值的工具结果：

- **产品线常量，每张 170HX 都相同，配方中硬编码是安全的：** 九个速度选择熔丝（`FUSE_SS_DP` = `0x1`，其余八个 = `0x5`）、`FUSE_PCIE_GEN23_DIS` = `0x1`、`FUSE_PCIE_GEN3_DIS` = `0x1`、`FUSE_NVLINK_DIS` = `0x7`、`FBPA_CFG1_BROADCAST` = `0x02449000`、`FUSE_PCIE_DEVIDB` = `0x20c2`、`FUSE_ECC_EN` = `0x0`、`FUSE_EN_SW_OVERRIDE` = `0x0`。（120 寄存器 diff 中的两张卡都是 10 GB 单元，所以任何按 SKU 而非按晶片变化的寄存器在该 diff 中看起来都像常量。确实有两个：`FUSE_SKU_ID`（`0x00821060`）在 10 GB 卡上读 `0x68`、在 8 GB 卡上读 `0x80`；`FUSE_PCIE_DEVIDA`（`0x008204d8`）在 10 GB 卡上读 `0x2082`、在 8 GB 卡上读 `0x20c2`，而 `FUSE_PCIE_DEVIDB` 两张卡都是 `0x20c2`。DEVIDA 和 SKU_ID 都不宜硬编码，`lspci -nn` 仍是最简单的 SKU 检验。）
- **绝不可硬编码的逐晶片值：** 所有筛选掩码及其 STATUS 镜像、`FEAT_OVR_SM_SPD`（`0x0082381c`）、`FEAT_OVR_SM_SPD_1`（`0x00823820`）、`FEAT_OVR_QUADRO`（`0x00823808`）、`I1500_DATA`、`I1500_SHADOW_WDR`，以及每个 FBPA 的读回值。

正是这个区分让[计算解锁](../unlock/compute-throttle.md)可以只发布两个固定魔数且对每张卡都正确，也正是为什么任何拿你的卡与"那个"原厂 SS0 值比较的工具都是在跟噪声比较。

### 检视你自己的卡

只读检视工具 `ga100_topology_report.py`（v1 为 4848 字节；v2 为 8128 字节，增加了 InfoROM 转储）读取 `PMC_BOOT_0`、`OPT_GPC_DISABLE`、`STATUS_OPT_GPC`、`OPT_GPC_DEFECTIVE`、`RING_ENUM_GPC`、`PTOP_SCAL_NUM_GPCS`、`PBUS_SW_SCRATCH(1)`（`0x00001404`）、`0x00118f78`、`OPT_PCIE_DEVIDA`、`OPT_SLT_REV`，以及每组 GPC 的 OPT_DISABLE / RECONFIG / CTRL_OPT / STATUS / RECONF_OVR。至少有四个人在两个 SKU 上跑过它，输出自洽。

> [!NOTE]
> **开放问题：缺失的 38 个 SM**
>
> 从 70 个 SM 到 A100 的 108 个、或晶片的 128 个，将是这张卡上最大的单项收益，而在 `OPT_GPC_DEFECTIVE` = 0 的卡上，被禁用的 GPC 是已知完好的硅片。目前找到的每条写入路径都被闩锁。`FUSE_CTRL_OPT_TPC_GPC` 只能删不能加（在活动 TPC 上做 OR 测试甚至没有降低数量）；在一次带两个阳性对照（证明写入原语是活的）的实验中，对 `OPT_GPC_DISABLE`、`STATUS_OPT_GPC`、`OPT_TPC_GPC2`（`0x00820768`）和 `DIS_SW_OVR`（`0x00820084`）的高安全写入全部原样读回；而用三种方式强推 `gpcMask`（RM 结构体、向 `0x00408970` 的主机 MMIO 写入、把 GSP 固件的 `andi` 改成 `li a4,255`）会让软件栈报告 8 GPC / 112 SM，但 `0x00408970` 每次都读回 `0xdc` 且 `cuInit` 段错误。尚未尝试的候选：经由静态筛选掩码查询的 GSP-RPC 路径（类 `0x2080122a` / `0x2080122b`）、GR 影子写入、或把写入原语移植到 PMU / GSP / FECS / GPCCS。见[死路](../history/dead-ends.md)。

> [!WARNING]
> **实验性质**
>
> 市面上一张卡被报告为 CTRL_OPT 扫到 **56 个 SM**（而不是熔丝下限 70），夺回 6 个 SM 后达到 62，剩余 TPC 在启用时确实失败。这被描述为见过的第一张计算侧被扫的卡，且没有发布前后的寄存器转储。其他每张被检视的卡都已经在熔丝下限，CTRL_OPT 对它们毫无损失。把低于 70 SM 的情况视为罕见而非正常。

---

## 内存分区与缓存层级

帧缓冲侧与图形侧独立筛选。GA100 有 12 个 FBP，每个带 2 个 FBPA，所以是 24 个 FBPA、每个 256 位。每个 1024 位 HBM 接口其实是四个 256 位通道，这正是为什么部分堆栈启用是真实的硬件状态，以及为什么 FBPA 掩码显示的是部分而非干净的整堆栈故障。

| 量 | 8 GB SKU（`0x20c2`） | 10 GB SKU（`0x2082`） |
|---|---|---|
| 启用的 FBPA | 16（共 24） | 20（共 24） |
| 启用的 FBP | 8（共 12） | 10（共 12） |
| 内存总线宽度 | 4096 位 | 5120 位 |
| 原厂容量 | 8192 MiB | 10240 MiB |
| 解锁后容量 | 65536 MiB | 40960 MiB |

筛选掩码同样逐晶片而异。两张 10 GB 卡读取 `FUSE_FBPA_DISABLE`（`0x00820368`）、`FUSE_FBIO_DISABLE`（`0x0082036c`）、`STATUS_FBPA`（`0x00820c18`）和 `STATUS_OPT_FBIO`（`0x00820c14`）：一张全部为 `0x0003c000`（FBPA 14 到 17 关闭），另一张全部为 `0x000000c3`（FBPA 0、1、6、7 关闭），两者都留下 20 个启用。被禁用的 FBPA 会从自己的 CSTATUS 寄存器返回 `0xbadf20xx` 哨兵值而非数值，哨兵随关闭的 FBPA 变化：禁用 FBPA 0、1、6、7 的卡返回 `0xbadf2010` 和 `0xbadf2013`，禁用 FBPA 14 到 17 的卡返回 `0xbadf2017` 和 `0xbadf2018`。一张 8 GB 卡的转储逐个枚举了它的 12 个 FBP 半堆栈区域：FBP 1 和 4 禁用、FBP 6 和 11 有缺陷、其余八个启用，即 8 × 8 GB = 64 GB——正好是解锁能达到的容量。完整细节见[内存子系统](memory-subsystem.md)和[内存几何](../unlock/memory-geometry.md)。

| 缓存层级 | 大小 | 依据 |
|---|---|---|
| 每 SM 的 L1 / 共享内存 | 192 KB | GA100 架构数据 |
| L2 | 32 MB（`32768 KB`） | CUDA `deviceQuery` 加一份独立的延迟尖峰微基准 |
| L2，完整 A100 | 40 MB | 供对比 |
| OpenCL 全局缓存 / 本地内存 | 1960 KB / 48 KB | 解锁卡上的 OpenCL-Benchmark |

> [!NOTE]
> **有争议**
>
> TechPowerUp 把 170HX 的 L2 列为 8 MB。运行时的 `deviceQuery` 数据 32768 KB 和一次独立的指针追逐延迟测量都说是 32 MB，本 wiki 采用 32 MB。语料库中没有来源调和两者；公布一条延迟/带宽曲线、展示工作集在哪掉下悬崖，就能了结此事。另请注意，正确的 TechPowerUp 条目是 `gpu-specs/cmp-170hx-8-gb.c3830`；旧的 `c3824` URL 现在会重定向到一张 AMD 卡。

HBM 带宽在这颗部件上**不受限制**，解锁也不改变它（同卡 A/B 在 256 MB 工作集下实测原厂 1592 GB/s、改装后 1599 GB/s，比值 1.0 倍，而同一张表里 FP32 移动了 30.7 倍）。理论峰值来自 5120 位上的 1215 MHz DDR，为 1555.2 GB/s（= 1448.4 GiB/s）。实测数字跨度 **1305.86 到 1600 GB/s**，取决于工具与访问模式，不存在单一权威数字；见[性能](../operations/performance.md)。

---

## 计算能力与指令集

计算能力 8.0（`sm_80`）决定了这颗晶片能做什么、不能做什么，与任何熔丝无关：

- **每 SM 64 条 FP32 通道**，每 SM 64 个 warp，warp 大小 32。FP64 为 GA100 的 1:2 比例，非张量 FP16 相对 FP32 为 4:1（架构上不寻常，也是锁定卡本来就能用于大模型 token 生成的原因）。
- **280 个第三代张量核心**（每 SM 4 个）、280 个 TMU、128 个 ROP。张量核心存在且可用，没有被熔断关闭：解锁卡实测 FP16 张量 158.7 到 190 TFLOPS、BF16 张量 164.4 到 192.7 TFLOPS。
- **没有 FP8 和 NVFP4 硬件路径。** 它们需要 `sm_89`+ / `sm_120`。在这张卡上枚举支持 MMA 形状的工具把 `mma_mxf8mxf8f32_16_8_32` 和 `mma_f8f8f16/f32_16_8_32` 列为不支持，这对 Ampere 是预期。INT8、INT4 和 INT1 是硬件原生支持，不过 INT1 与 INT8 共用 XNOR-popcount 路径，没有专用单元。
- 对推理的实际影响：`sm_80` 不支持 FP8 KV cache，所以 KV 必须是 BF16。

---

## 存在、被熔断关闭与缺失的引擎

| 引擎 / 功能 | 在 170HX 上的状态 | 证据 |
|---|---|---|
| CUDA 核心、张量核心 | 存在、可用，原厂下发射速率受限 | 见[计算限速](../unlock/compute-throttle.md) |
| FP64 单元 | 存在，由解锁恢复 | 实测为 1:2 比例 |
| **NVENC** | **缺失。** GA100 晶片一般不搭载视频编码器 | 频道内报告，与晶片功能集一致 |
| **NVDEC** | 晶片级：按规格数据库为 5 个第四代 NVDEC 实例。驱动级：此 SKU 未暴露。探测 NVDEC falcon 邮箱 `0x00830040` 返回 `0xbadf1100`，即被阻塞或只读 | 规格数据库，中置信度；falcon 探测 |
| **显示引擎 / 输出** | **缺失。** 没有任何显示输出，`Slot Width: IGP`，不暴露 DirectX / Vulkan / OpenGL | 规格数据库及每一次 `lspci` 抓取，设备被命名为 `3D controller` |
| **NVLink** | **熔断关闭。** `FUSE_NVLINK_DIS` / `STATUS_OPT_NVLINK`（`0x00820db8`）= `0x7`；板卡还缺少 NVLink 接口芯片。`0x00823800`–`0x0082382c` 块中不出现任何 NVLink 寄存器，也没有分支包含 NVLink 代码 | 熔丝读取加拆解；见[NVLink 硬件](nvlink-hardware.md) |
| **ECC** | **熔断关闭。** `FUSE_ECC_EN` = `0x0`，无遥测，无已知杠杆。`FEAT_READOUT_0` 的高位 ECC 状态半字节读零，与熔丝一致 | 见[ECC 前沿](../frontier/ecc.md) |
| **P2P** | 缺失。正式发布或分支树中都没有 P2P 代码；唯一的 MIG profile 报告 `P2P: No` | 见[P2P 前沿](../frontier/p2p.md) |
| **MIG** | 硬件支持（`0x00820840` 第 0 位启用），但只暴露一个 profile `1g.64gb`，所以 GPU 实际无法分区 | 社区发现，不在正式发布代码中 |
| **可调整大小 BAR** | 存在但限制为 64 MiB | `lspci` 能力 `[bb0]` |
| **FLR** | 存在（DevCap 中 `FLReset+`），且每个解锁 harness 都依赖它 | `lspci -vvv` |

> [!NOTE]
> **开放问题：NVENC 是熔断关闭还是根本没造？**
>
> 频道内的说法是 "nvenc is disabled ... idr if it's fused off or if it's fuse gated but it's not available by default if it has the hardware"（NVENC 被禁用了……不知道它是被熔断关闭还是被熔丝门控，但即便有硬件，默认也不可用）。没有人报告过可用的 NVENC 会话，也没有人提出解锁路径。显然的下一步是破解计算限速时用过的同一种差分方法：在 170HX 上读取 `0x00820xxx` 块中与 NVENC 相关的 `OPT_*_DISABLE` 熔丝，再在 A100 或 DRIVE A100 对照部件上读取，做 diff。在那之前，请假定这张卡没有硬件编码器。

---

## 这颗晶片为什么对解锁很重要

正是这颗特定硅片的三个属性让整个项目成为可能：

1. **总开关熔丝未熔断。** `OPT_FEATURE_FUSES_OVERRIDE_DISABLE`（`0x008203f0`）读 `0x00000000`。如果它被熔断，每个功能覆盖都会被永久锁定，软件路径将不存在。
2. **限制熔丝是产品线常量，不是逐晶片分 bin。** 一份配方适用于每张卡。
3. **DRIVE A100（`0x20bb`，PG199，`GA100-550F-A1`）是干净的阴性对照。** 两块物理板卡共享 170HX 的 NVLink 击杀和 `EN_SW_OVERRIDE` = 0 / `DIS_SW_OVR` = 1 状态，但九个速度选择熔丝全部读 `0x00000000`，`FEATURE_READOUT_1` = `0x00000000`，并且以 96 个 SM 全速计算。正是这个单变量隔离证明了 `OPT_SM_SPEED_SELECT` 熔丝块——而不是其他任何东西——才是计算限制。

完整的熔丝图见[熔丝与 OTP](fuses-and-otp.md)，覆盖寄存器如何被触达见[计算限速](../unlock/compute-throttle.md)。
