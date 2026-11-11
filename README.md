# hybrid-retrieval-scifact
基于 Haystack 与 BEIR SciFact 的 BM25、稠密检索、RRF 融合和 Cross-Encoder 重排序复现与评测
本项目使用预训练模型，未进行模型训练或微调。

## 项目内容

- 运行 PubMed 小规模检索示例，理解 Haystack 索引与检索流程。
- 使用独立文档库建立 SciFact 检索实验，保留原始文档 ID。
- 比较 BM25、Embedding、Hybrid + RRF 和 Hybrid + Cross-Encoder。
- 实现 Recall@10、MRR@10 评测及召回互补性分析。
- 检查相关文档的排名变化，分析检索遗漏与重排序失败案例。

## 实验配置

| 项目 | 配置 |
|---|---|
| 数据集 | BEIR SciFact |
| 文档数量 | 5,183 |
| 测试查询数量 | 300 |
| 文档输入 | 标题 + 正文 |
| Embedding 模型 | BAAI/bge-small-en-v1.5 |
| Cross-Encoder 模型 | BAAI/bge-reranker-base |
| 单路候选数量 | BM25、Embedding 各 Top 10 |
| 最终返回数量 | Top 10 |
| RRF 参数 | k = 60 |
| 运行平台 | Google Colab |
| Haystack 版本 | 3.5.0 |

PubMed 教程示例与 SciFact 实验使用独立文档库。

两种混合方案分别对两路候选排序：RRF 按倒数排名分数融合；Cross-Encoder 使用查询与候选文档的相关性分数重排序。本项目没有实现 RRF 后再接 Cross-Encoder 的串联方案。

## 实验结果

| 方法 | Recall@10 | MRR@10 |
|---|---:|---:|
| BM25 | 0.7851 | 0.6225 |
| Embedding | 0.8452 | 0.6845 |
| Hybrid + RRF | 0.8606 | 0.6875 |
| Hybrid + Cross-Encoder | **0.8782** | **0.6990** |

Recall@10 为逐查询召回率的平均值；MRR@10 为 Top 10 中首个相关文档倒数排名的平均值，未命中记为 0。

在本次实验配置下，Hybrid + Cross-Encoder 的两项指标最高。相比 Embedding，其 Recall@10 提高约 3.30 个百分点，MRR@10 提高约 1.46 个百分点。

以上结果来自 Notebook 中自行实现的评测代码，尚未与 BEIR 官方评测工具交叉验证。

## 召回互补性

以 Top 10 中是否至少包含一篇标注相关文档为命中标准：

| 查询分类 | 数量 |
|---|---:|
| 两种方法均命中 | 225 |
| 仅 BM25 命中 | 17 |
| 仅 Embedding 命中 | 32 |
| 两种方法均未命中 | 26 |

结果说明，两种检索方法在当前设置下存在召回互补性。这一分类统计使用查询命中口径，与 Recall@10 不同。

## 案例分析

### Query 1100：关键词检索更有利

查询：`Statins increase blood cholesterol.`

标注相关文档在 BM25 中排名第 2，在 Embedding 中排名第 15，因此仅 BM25 在 Top 10 内命中。

### Query 1049：向量检索命中

查询：`Ribosomopathies have a low degree of cell and tissue specific pathology.`

标注相关文档未进入 BM25 Top 100，在 Embedding 中排名第 6。查询与文档存在表达差异，但尚未通过控制实验确认其具体影响。

### Query 1：重排序后跌出 Top 10

查询：`0-dimensional biomaterials show inductive properties.`

标注相关文档在 Embedding 中排名第 6，在 RRF 中排名第 12，在 Cross-Encoder 中排名第 15。该案例说明，混合排序可能使单路已命中的相关文档跌出最终 Top 10。

## 运行方式

1. 下载本仓库中的 `33_Hybrid_Retrieval_fixed.ipynb`。
2. 在 Google Colab 中上传并打开。
3. 在「代码执行程序 → 更改运行时类型」中选择 GPU。
4. 从上到下运行全部单元格。

Notebook 会下载数据集与模型，完成索引、检索和评测，并生成：

- `metrics.json`：本次运行的实验配置与汇总指标。
- `requirements-lock.txt`：本次运行环境的依赖快照。

代码会根据 CUDA 是否可用选择 GPU 或 CPU。完整实验建议使用 Colab GPU；未验证 Windows CPU 下的完整运行表现。

仓库中的 `requirements-lock.txt` 记录了成功运行时的 Colab 环境，其中可能包含平台相关依赖，不保证能直接用于其他系统。Notebook 安装单元仅固定了 Haystack 主版本，其余依赖变化可能影响后续复现。

## 文件说明

| 文件 | 内容 |
|---|---|
| `33_Hybrid_Retrieval_fixed.ipynb` | 教程示例、SciFact 实验和运行输出 |
| `metrics.json` | 实验汇总指标 |
| `requirements-lock.txt` | 成功运行环境的依赖快照 |
| `README.md` | 项目说明与运行方法 |

## 局限

- 仅在 SciFact 数据集和当前参数设置下进行比较。
- 未开展模型训练、微调或新算法设计。
- 未评估延迟、显存占用、统计显著性及参数敏感性。
- 未系统分析长文本截断对检索与重排序的影响。
- 未分析预训练模型训练数据与评测集可能存在的重叠。
- 案例中的原因解释属于分析假设，未经消融实验验证。

## 参考与来源

本项目基于 Haystack 官方教程进行复现与扩展，基础 Pipeline 与组件使用方式来自官方示例；SciFact 评测、算法对比和案例分析在此基础上整理实现。

- [Haystack 官方 Hybrid Retrieval 教程](https://haystack.deepset.ai/tutorials/33_hybrid_retrieval)
- [BEIR 官方仓库](https://github.com/beir-cellar/beir)
- [BGE-small-en-v1.5](https://huggingface.co/BAAI/bge-small-en-v1.5)
- [BGE-reranker-base](https://huggingface.co/BAAI/bge-reranker-base)
