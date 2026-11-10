---
type: raw-book-manifest
title: Neural Networks and Deep Learning — 全文抓取说明
created: 2026-09-04
tags: [deep-learning, neural-networks, book, michael-nielsen, raw-archive]
---

# Neural Networks and Deep Learning（在线全文存档）

**Michael A. Nielsen** 的免费在线书 *Neural Networks and Deep Learning* 全文英文抓取存档。

| 项 | 内容 |
|---|---|
| 书名 | Neural Networks and Deep Learning |
| 作者 | Michael A. Nielsen |
| 在线地址 | http://neuralnetworksanddeeplearning.com/ |
| 目录年份 | 2015（第一版在线书年份；页面现行文本为持续修订版，代码为 Python 2.7，注意年代） |
| 抓取日期 | 2026-09-04 |
| 章节 | 6 章 + 首页/about |
| 图片 | 已镜像至 `images/`（97 张，约 2.5 MB），md 内为相对引用 |

## 文件与章节对照

| 文件 | 章节 | 一句话内容 |
|---|---|---|
| `index.md` | 首页 + About | 全书导言：目标读者、前置要求、阅读路线 |
| `ch01-handwritten-digits.md` | Ch.1 Using neural nets to recognize handwritten digits | 感知机、sigmoid 神经元、随机梯度下降；74 行代码实现 MNIST 识别（96%） |
| `ch02-backpropagation.md` | Ch.2 How the backpropagation algorithm works | 反向传播完整推导：代价的两种假设、Hadamard 积、四个基本方程、证明与直觉 |
| `ch03-improving-learning.md` | Ch.3 Improving the way neural networks learn | 交叉熵、正则化（L2/dropout/增广）、权重初始化、超参数启发式、手写数字识别最终技巧 |
| `ch04-universal-computation.md` | Ch.4 A visual proof that neural nets can compute any function | 两层网络可近似任意连续函数的可视化论证（直觉式，非严格证明） |
| `ch05-deep-networks-hard-to-train.md` | Ch.5 Why are deep neural networks hard to train? | 深层 vs 浅层：梯度消失/爆炸、不稳定梯度问题 |
| `ch06-deep-learning.md` | Ch.6 Deep learning | 卷积网络（CNN）在 MNIST 的完整实现与可视化；图像/语音/文本进展与展望 |

## 抓取与清洗说明

- 正文用 **Defuddle** 从各章节 HTML 抽取，结构与图片相对路径保留。
- **数学公式**（原文为 HTML 内嵌 LaTeX/MathJax 源码）已清洗为 Obsidian/MathJax 可直接渲染的形式：
  - 行内公式 `$...$`：撤销 Defuddle 的 markdown 转义（反斜杠恢复为真实 LaTeX），如 `$x\_1$` → `$x_1$`。
  - 独立公式 `\begin{eqnarray}...\end{eqnarray}`（共 116 个）：包裹为 `$$ ... $$` 块，命令与 `\\` 换行符还原。
  - 原文的公式编号 `\tag{n}` 保留，正文"Equation (n)"引用可据此溯源。
- **章内交叉链接**（`chapN.html`、`index.html`、锚点）已重写为指向本目录 md 的相对链接；站外链接（Wikipedia、论文、GitHub 等）保持原样。
- 每章带 YAML frontmatter：`title / book / author / source / fetched / tags / summary`，与 RawSource 其余条目风格一致。
- 站点为纯静态，图片 `images/*` 已批量镜像；未来若在线版更新，可对相应章节重跑抓取。

## 与知识库的关系

- 本目录属 **RawSource 原始资料层**（`wiki/RawSource/articles/`），保持英文原文，供 LLM/TheWiki 通读后沉淀。
- 如需中文学习译本，可仿照 `Attention-Is-All-You-Need-2017-中文翻译.md` 另行产出，并在此登记。
- 中文学习译本进度：
  - ✅ `ch01-handwritten-digits-中文翻译.md`（第 1 章，2026-09-07）——正文忠实全译，公式/图片/代码保留原文，注释中译
  - ✅ `ch02-backpropagation-中文翻译.md`（第 2 章，2026-09-07）——含四个基本方程 BP1–BP4 证明与全部编号公式，注释中译
  - ✅ `ch03-improving-learning-中文翻译.md`（第 3 章，2026-09-07）——含交叉熵推导、softmax、L2/L1/dropout/数据增广、权重初始化、network2.py 完整代码、超参启发式；交互演示以标注占位
  - ✅ `ch04-universal-computation-中文翻译.md`（第 4 章，2026-09-07）——通用性定理可视化论证；交互/视频以标注占位
  - ✅ `ch05-deep-networks-hard-to-train-中文翻译.md`（第 5 章，2026-09-07）——梯度消失/爆炸/不稳定梯度完整推导
  - ✅ `ch06-deep-learning-中文翻译.md`（第 6 章，2026-09-07）——CNN 原理+MNIST 实战、network3.py 完整代码、图像识别进展、RNN/LSTM/DBN 简述与展望
  - 🎉 全书 6 章中文译本已完成
- 与此书相关的 TheWiki 页面建议：`[[NeuralNetworkBasics]]`（wiki/TheWiki/deep-learning/ 下已有草稿）可按章补充反向传播、交叉熵、正则化、CNN 等条目。

## 原始 HTML 归档

若要对照最接近源站的 HTML（含公式原始 LaTeX 与 MathJax 配置），可自行抓取 http://neuralnetworksanddeeplearning.com/chap{N}.html。本存档未附带 HTML，只保留清洗后的 Markdown + 图片，以减小体积。
