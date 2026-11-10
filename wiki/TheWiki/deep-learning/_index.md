---
type: index
title: deep-learning 领域索引
created: 2026-08-27
updated: 2026-09-07
---

# deep-learning 领域索引

> 深度学习基础概念知识域。为 [[MotionInbetweeningTwoStage|motion-inbetweening 领域]]等动画技术论文提供底层概念支撑。
> 这些概念是分析任何神经网络动画方法（[[TheWiki/motion-matching/_index|motion-matching]] / [[TheWiki/motion-inbetweening/_index|motion-inbetweening]]）的工具箱。

## 知识页

| 页面 | 类型 | 派生? | Summary |
|---|---|---|---|
| [[NeuralNetworkBasics]] | concept | 原生 | 深度网络基本单元神经元 y=φ(Wx+b)，非线性是深度产生的原因；MLP 是最简单的全连接前馈网络，逐帧独立且权重共享。并入 Nielsen Ch.1：感知机→sigmoid 平滑动机、NAND 逻辑通用性、梯度下降/SGD 更新式（万能逼近留待 Ch.4）。 |
| [[RecurrentNetworks]] | concept | 原生 | RNN 用隐藏状态在时间步间传记忆处理序列，展开即深层共享权重网络，BPTT 致梯度消失与长程遗忘，自回归逐帧生成慢且局部；Encoder-Decoder 是 seq2seq 范式，论文 Context Transformer 的 decoder 只是 MLP 非解码器，是术语陷阱。 |
| [[RNN-LSTM-GRU]] | concept | 原生 | 以机器翻译为背景讲 RNN/LSTM/GRU 单元原理：裸 RNN 全量改写记忆致梯度消失，LSTM 用细胞状态+三门读写，GRU 精简为两门；逐项拆公式并三方对比。 |
| [[Transformer]] | concept | 原生 | 用自注意力替换循环的并行序列网络；核心是"每帧从所有帧按相关性取加权平均，相关性从数据现算"；每层=多头自注意力(跨帧混合)+FFN(逐帧变换)+残差+LayerNorm，堆 L 层；输入输出同形，反复精炼。 |
| [[SelfAttention]] | concept | 原生 | 注意力核心运算 softmax(Q·K^T)·V；Q/K/V 为查询/招牌/内容，权重随输入变故非固定矩阵；多头=多套投影并行；掩码让占位帧只从已知帧取信息实现补间。含 3×2 手算全程与 W_Q 维度辨析。 |
| [[Backpropagation]] | concept | 原生 | 反向传播=链式法则+局部性，梯度信号从损失倒流；矩阵乘法反向核心规则 ∂L/∂W=X^T·∂L/∂Y；Nielsen 四公式 BP1-BP4（误差 δ=∂C/∂z 逐层回传）为通用框架；雅可比是向量函数的导数，softmax 满雅可比须单独处理。项目 W_Q 反传实例见 Backpropagation-Transformer。 |
| [[Backpropagation-Transformer]] | concept | 原生 | 项目视角的反向传播实例：梯度从 Loss 经 decoder/FFN/残差/W_O/A·V/softmax/Q·K^T 一路传回 W_Q，最终形式 ∂L/∂W_Q=z^T·g_Q 与 MLP 的 ∂L/∂W=x^T·g_y 完全同形；softmax 需满雅可比单独处理。通用机制见 Backpropagation。 |
| [[TrainingMechanics]] | concept | 原生 | 深网训练四基石：①输出层饱和解法——sigmoid+交叉熵 与 softmax+对数似然 殊途同归，输出层误差 δ^L=a^L-y；②梯度消失/爆炸=反向传播链式连乘，每层因子≈wσ'(z)，sigmoid 的 σ'≤1/4 故常规深网几乎总消失（前层学不动），解法=ReLU+好初始化+归一化把因子控在 1 附近；③LayerNorm 逐帧跨 512 特征减均值除标准差再 γ/β 缩放，压住深网幅度失控，pre-norm 残流直通故深网更稳需补最终 LN；④Adam=SGD+动量+自适应步长，偏差修正配 warmup，Transformer 训练标配。 |
| [[ConvolutionalNetworks]] | concept | 原生 | CNN=局部感受野+共享权重+池化三种思想，利用图像空间结构（平移不变性）大幅减少参数、加速训练；MNIST 从全连接 97.8% 一路提到 99.67%（ReLU/数据增广/dropout/集成），是图像识别标配。项目里 CNN 仅作 FCN(1D conv) baseline，2D CNN 未采用。 |
| [[PCA]] | concept | 原生 | PCA 用协方差矩阵的特征分解找数据"散得最开"的方向，把高维压到低维子空间：特征值=方向方差、特征向量=形变模式，SVD(UΣVᵀ)是稳定算法。解释了布料 55638 维→100 系数、[[QNN]] 只输出 PCA 系数的数学地基。 |
| [[FiLM]] | concept | 原生 | FiLM=逐通道仿射调制 输出=γ×特征图+β，γ/β 由条件输入经生成器现算（是激活不是参数）；每通道仅 2 参数、与分辨率无关，且证明不依赖归一化。是 HyperBones/PhySkin 身份适配的出处。 |

## 关系速览

```
                    NeuralNetworkBasics (DNN/MLP 砖块)
                          │
              ┌───────────┴───────────┐
              ▼                         ▼
        RecurrentNetworks          Transformer ──含── SelfAttention (核心运算)
        (RNN/Enc-Dec 范式)           │                    │
              │                     ├── FFN(=MLP)        ├── 多头
              │ 对比基线              ├── PositionalEnc   ├── 掩码
              │ (ERD-QV)             └── 堆叠+残差         │
              │                                           │
              └──► 被 motion-inbetweening 取代 ◄───────────┘
                         │
        训练侧 ──── Backpropagation (传回 W_Q)
                   └── JacobianMatrix (softmax 满雅可比)
                   └── TrainingMechanics (交叉熵/梯度消失/LayerNorm/Adam/Noam)

        调制侧 ──── FiLM (γ⊙h+β 条件化；不混合跨位置信息)
                     └── 落地于 cloth-animation 2026 两篇的身份/形状适配
```

> [[RNN-LSTM-GRU]] 是 [[RecurrentNetworks]] 的单元级下钻页：裸 RNN/LSTM/GRU 的记忆机制与门控公式（以机器翻译为例）。

> 项目主线是**序列**建模（RNN → Transformer）；**ConvolutionalNetworks**（图像领域标配）在项目里只作 FCN(1D conv) baseline 对比出现，未进入主线。

## 阅读路径建议

1. [[NeuralNetworkBasics]] —— 神经元、MLP、为什么非线性关键
2. [[RecurrentNetworks]] —— RNN 的递归与局限、Encoder-Decoder 范式（单元门控下钻见 [[RNN-LSTM-GRU]]）
3. [[Transformer]] —— 核心一句话、层结构、完整数据流
4. [[SelfAttention]] —— Q/K/V 手算全程、掩码做补间、W_Q 维度
5. [[Backpropagation]] —— 通用机制：链式法则/局部性、Nielsen 四公式、雅可比；[[Backpropagation-Transformer]] —— 项目实例：从 Loss 传回 W_Q
6. [[TrainingMechanics]] —— LayerNorm、Adam vs SGD、Noam warmup
7. [[ConvolutionalNetworks]] —— 图像领域标配：局部感受野/共享权重/池化（项目未采用，仅 FCN baseline）
8. [[PCA]] —— 协方差/特征值/SVD：布料 55638 维压到 100 系数（读 [[QNN]] 前补的数学）
9. [[FiLM]] —— 逐通道仿射调制：条件化如何"改网络状态"而非"塞进数据流"（读 [[HyperBones2026]] / [[PhySkin2026]] 的身份适配前需补）
