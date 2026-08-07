# 中文翻译规范（所有翻译 agent 必须遵守）

## 总原则

把 `docs/` 下的英文文档逐篇翻译为简体中文，输出到 **`docs/zh/` 下的同路径同名文件**。
例如 `docs/start/what-is-this-card.md` → `docs/zh/start/what-is-this-card.md`。

- 目标：专业、准确、通顺的简体中文技术文档，面向中文社区（矿卡玩家、DIY 爱好者、开发者）。
- 语气：技术参考文档风格，简洁明确，不口语化、不夹带英文腔。
- 保留所有 Markdown 结构与格式：标题层级、表格、列表、引用块、代码块、告警块（`> [!NOTE]` / `> [!WARNING]` / `> [!DANGER]` / `> [!TIP]`）、脚注、内联代码。
- 保留所有链接目标**原样不动**（相对路径、锚点、外部 URL、GitHub 链接、图片路径），只翻译链接的显示文字（若为英文）。
- 代码块、命令行、寄存器地址、十六进制数值、设备 ID（如 `10de:20c2`）、BIOS 版本号、日期一律不翻译、不改写。

## 术语翻译对照表（必须统一使用）

| 英文 | 中文 |
|---|---|
| CMP (Cryptocurrency Mining Processor) | CMP（加密货币矿卡处理器），首次出现可写全称 |
| unlock / unlocked | 解锁 |
| fuse | 熔丝 |
| OTP (one-time-programmable) | 一次性可编程（OTP） |
| streaming multiprocessor (SM) | 流式多处理器（SM） |
| CUDA core | CUDA 核心 |
| GPC (Graphics Processing Cluster) | 图形处理集群（GPC） |
| TPC (Texture Processing Cluster) | 纹理处理集群（TPC） |
| memory controller | 内存控制器 |
| HBM / HBM2e / HBM2 | HBM / HBM2e / HBM2（不翻译） |
| VBIOS / BIOS / firmware | VBIOS / BIOS / 固件 |
| driver | 驱动 |
| GPU driver patch | 驱动补丁 |
| register | 寄存器 |
| clock / clock speed | 时钟 / 频率 |
| boost clock | 睿频 / 升频（首次可用"睿频（boost clock）"） |
| power limit / power cap | 功耗上限 / 功耗墙 |
| thermal throttling | 热降频 |
| PCIe Gen1/Gen2/Gen3/Gen4 | PCIe 第 1/2/3/4 代 |
| link speed | 链路速率 |
| link width | 链路宽度 |
| lane | 通道（PCIe lane） |
| x4 / x8 / x16 | x4 / x8 / x16（不翻译） |
| NVLink | NVLink（不翻译） |
| SXM2 / SXM4 | SXM2 / SXM4（不翻译） |
| ReBAR / Resizable BAR | ReBAR / 可调整大小 BAR |
| BAR (Base Address Register) | BAR（基地址寄存器） |
| P2P (peer-to-peer) | P2P（对等直连） |
| ECC (Error Correction Code) | ECC（纠错码） |
| FP16/FP32/FP64/BF16/TF32/INT8 | 不翻译 |
| TFLOPS / TOPS / GB/s | 不翻译 |
| throughput | 吞吐 / 吞吐量 |
| bandwidth | 带宽 |
| VRAM / memory capacity | 显存 / 显存容量 |
| mining / miner | 挖矿 / 矿工 |
| hash rate | 算力（挖矿） |
| board / PCB | 板卡 / PCB |
| interposer | 中介层（interposer） |
| waterblock | 水冷头 |
| shroud | 导流罩 / 风罩 |
| air cooling | 风冷 |
| passive cooling | 被动散热 |
| ROP chain | ROP 链（保留英文 ROP） |
| Booter / Falcon / GSP-RM / RM | 保留英文（专有组件名） |
| csecret | 保留英文 |
| status board | 状态板 |
| dead end | 死路 / 已失败路线 |
| clean room | 净室（clean-room，指独立逆向工程） |
| provenance | 出处 / 来源 |
| OpenCL / CUDA / vLLM / llama.cpp / MkDocs / GitHub | 保留英文 |

## 告警块

`> [!NOTE]` 等 GitHub 告警语法必须保留，只翻译块内正文。示例：

```markdown
> [!WARNING]
> **切勿对供电电路短路。**
```

## 标题与文档头

- 翻译标题文本，保留 `#` 层级。
- 文档开头的 `**What this page covers:**` 之类的说明句翻译为 `**本页涵盖：**` 或相近表达。

## 校对要求

1. 翻译完成后通读一遍，修正错别字、漏译、格式破坏。
2. 确保所有相对链接仍指向正确的目标（因为镜像结构相同，链接路径无需改动）。
3. 确保没有遗留未翻译的英文正文段落（代码块、专有名词、表格内的纯英文数值列除外）。

## 交付

每个 agent 完成后，汇报：翻译了哪些文件、每个文件大致字数、是否有存疑术语或无法确定的地方。
