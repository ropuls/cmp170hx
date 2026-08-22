# CMP 170HX 维基

面向 NVIDIA CMP 170HX（GA100）的全面技术参考：芯片、固件、社区解锁、操作流程与开放前沿。

55 页，约 27.8 万词。截至 **2026-07-31**。

**English:** [README.md](README.md) · [English wiki](https://github.com/Consensus-Protocol/cmp170hx/wiki)

## 两种阅读方式

- **[Wiki 标签页](https://github.com/Consensus-Protocol/cmp170hx/wiki)** 用于浏览，带侧边栏。那里的页面由 `docs/` 生成，除链接样式外完全一致。
- **本仓库中的 `docs/`** 是权威源。可审查、可接受 pull request，并通过 MkDocs 构建带搜索的完整主题站点。

两者必须保持一致。`to_github_wiki.py` 把 `docs/` 发布到 wiki；如果直接在 wiki 上编辑，先同步回来再继续，确保两棵树永不分叉。

## 中文版说明

中文翻译位于 **`docs/zh/`**，与英文 `docs/` 结构一一对应，链接相对路径保持一致。用独立配置构建中文站：

```bash
pip install mkdocs mkdocs-material
mkdocs build -f mkdocs.zh.yml    # 静态站点输出到 ../site-zh
```

术语对照与翻译约定见 `docs/zh/translation-guide.md`（供后续贡献者保持一致）。

## 阅读

页面是纯 Markdown，在 GitHub 或任何编辑器里都能直接阅读。提示框使用 GitHub alert 块引用（`> [!NOTE]`），GitHub 原生渲染；MkDocs 未启用对应扩展时按普通引用块渲染。要获得搜索和导航：

```bash
pip install mkdocs mkdocs-material
mkdocs serve          # http://127.0.0.1:8000
mkdocs build          # 静态站点输出到 ../site
```

## 内容涵盖

| 章节 | 内容 |
|---|---|
| `start/` | 入门引导、卡型识别、快速入门、风险、术语表 |
| `hardware/` | GA100 芯片、板卡变体、内存子系统、熔丝与 OTP、PCIe、NVLink、供电、散热、VBIOS |
| `unlock/` | 端到端机制：Falcon 与 Booter、ROP 链、特权级掩码、内存几何、计算限速、驱动补丁、PCIe Gen2、完整寄存器参考 |
| `procedures/` | 安装、验证、故障排查、恢复、多 GPU、驱动版本、卸载 |
| `operations/` | 散热、供电与电源、物理改造、性能、LLM 推理、调校 |
| `frontier/` | 状态板与未解决问题：PCIe Gen3/Gen4、NVLink、ECC、80 GB、P2P |
| `history/` | 时间线、净室与出处问题、死路、工具谱系 |
| `appendix/` | 寄存器索引、保留的工件、外部来源、方法论 |

## 本维基坚持的两件事

**容量按型号区分，不可互换。** 8 GB 卡（`10de:20c2`）解锁为 **64 GB**。10 GB 卡（`10de:2082`）解锁为 **40 GB**。针对 10 GB 卡的 80 GB 配置曾被构建、测试并因不稳定而被否决。

**PCIe 链路速率和链路宽度是两个独立问题。** Gen1 到 Gen2 是软件解锁，自 2026-07-29 起随 cmpunlocker `master` 发布，因此任何安装了解锁的卡都运行在 Gen2。超过 x4 宽度需要手工焊接 24 个交流耦合电容。两者互不相干。

## 约定

普通散文是已确认的事实。实验性、危险和未解决的内容用告警标记。仅依赖单一观察的声明会在句子中说明。当证据确实冲突且无法定论时，维基会直说，而不是悄悄选边。

全站不点任何个人名字。发现按日期和频道归因，而不是按人。底层声明如何收集、裁决和验证，以及局限性的诚实说明，见 `docs/appendix/methodology.md`。
