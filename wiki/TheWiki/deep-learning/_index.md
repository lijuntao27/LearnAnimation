---
type: index
title: deep-learning 领域索引
created: 2026-08-27
updated: 2026-08-27
---

# deep-learning 领域索引

> 深度学习基础概念知识域。为 [[MotionInbetweeningTwoStage|motion-inbetweening 领域]]等动画技术论文提供底层概念支撑。
> 这些概念是分析任何神经网络动画方法（[[TheWiki/motion-matching/_index|motion-matching]] / [[TheWiki/motion-inbetweening/_index|motion-inbetweening]]）的工具箱。

## 知识页

| 页面 | 类型 | 派生? | Summary |
|---|---|---|---|
| [[NeuralNetworkBasics]] | concept | 原生 | 深度网络基本单元神经元 y=φ(Wx+b)，非线性是深度产生的原因；MLP 是最简单的全连接前馈网络，逐帧独立且权重共享，是 Transformer 中 encoder/decoder/FFN 的基础砖块。 |
| [[RecurrentNetworks]] | concept | 原生 | RNN 用隐藏状态在时间步间传记忆处理序列，展开即深层共享权重网络，BPTT 致梯度消失与长程遗忘，自回归逐帧生成慢且局部；Encoder-Decoder 是 seq2seq 范式，论文 Context Transformer 的 decoder 只是 MLP 非解码器，是术语陷阱。 |
| [[Transformer]] | concept | 原生 | 用自注意力替换循环的并行序列网络；核心是"每帧从所有帧按相关性取加权平均，相关性从数据现算"；每层=多头自注意力(跨帧混合)+FFN(逐帧变换)+残差+LayerNorm，堆 L 层；输入输出同形，反复精炼。 |
| [[SelfAttention]] | concept | 原生 | 注意力核心运算 softmax(Q·K^T)·V；Q/K/V 为查询/招牌/内容，权重随输入变故非固定矩阵；多头=多套投影并行；掩码让占位帧只从已知帧取信息实现补间。含 3×2 手算全程与 W_Q 维度辨析。 |
| [[Backpropagation]] | concept | 原生 | 反向传播=链式法则+局部性，梯度信号从损失倒流；矩阵乘法反向核心规则 ∂L/∂W=X^T·∂L/∂Y；从 Loss 经 W_O/A·V/softmax/Q·K^T 传回 W_Q，最终形式 z^T·g_Q 与 MLP 同形；雅可比是向量函数的导数，softmax 满雅可比须单独处理。 |
| [[TrainingMechanics]] | concept | 原生 | LayerNorm 逐帧跨 512 特征减均值除标准差再 γ/β 缩放，压住深网幅度失控稳定训练，沿特征维不依赖 batch，pre-norm 残流直通故深网更稳需补最终 LN；Adam=SGD+动量(一阶矩)+自适应(二阶矩逐参数缩放步长)，偏差修正配 warmup，Transformer 训练标配。 |
| [[PCA]] | concept | 原生 | PCA 用协方差矩阵的特征分解找数据"散得最开"的方向，把高维压到低维子空间：特征值=方向方差、特征向量=形变模式，SVD(UΣVᵀ)是稳定算法。解释了布料 55638 维→100 系数、[[QNN]] 只输出 PCA 系数的数学地基。 |

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
                   └── TrainingMechanics (LayerNorm/Adam/Noam)
```

## 阅读路径建议

1. [[NeuralNetworkBasics]] —— 神经元、MLP、为什么非线性关键
2. [[RecurrentNetworks]] —— RNN 的递归与局限、Encoder-Decoder 范式
3. [[Transformer]] —— 核心一句话、层结构、完整数据流
4. [[SelfAttention]] —— Q/K/V 手算全程、掩码做补间、W_Q 维度
5. [[Backpropagation]] —— 从 Loss 传回 W_Q、雅可比
6. [[TrainingMechanics]] —— LayerNorm、Adam vs SGD、Noam warmup
7. [[PCA]] —— 协方差/特征值/SVD：布料 55638 维压到 100 系数（读 [[QNN]] 前补的数学）
