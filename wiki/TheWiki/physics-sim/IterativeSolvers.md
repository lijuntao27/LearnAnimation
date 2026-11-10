---
type: concept
title: Jacobi 与 Gauss-Seidel 迭代（PBD 视角的数值线性代数）
created: 2026-10-22
updated: 2026-10-26
tags: [physics-sim, numerical-linear-algebra, iterative-solvers, jacobi, gauss-seidel, convergence, spectral-radius, math-foundation, nonlinear-convergence, projection-methods]
summary: 定常迭代法专题：分裂法 A=Q−(Q−A) 统一框架下 Jacobi（Q=D，并行、慢）与 Gauss-Seidel（Q=D−L，串行、快）；收敛判据三条路线——严格对角占优的归纳收缩证明（含 β/γ/s 因子递推）、谱半径判据 ρ(M)<1（含几何图像/灯喻/FAQ 三偏差/Young 定理适用范围）、SPD 能量范数单调（Ostrowski–Reich，PBD 最相关）；三角方程的向前代入 O(n²) "求逆"真相；全部概念均落到 PBD/XPBD 对应点。§4 接续非线性侧：PBD 循环投影的收敛性理论——三层"收敛"区分（稳定/迭代收敛/离散化收敛）、POCS 凸集循环投影全局保证、KM 平均（k<1 刚度缩放的定理级解释）、非凸集横截角局部线性收敛、XPBD 把 GS 变成能量单调的坐标下降，附常用约束逐个过堂表与翻车模式对照。
---

# Jacobi 与 Gauss-Seidel 迭代

> 本页是读 [[RawSource/articles/Jakobsen-2001-AdvancedCharacterPhysics-中文翻译|Jakobsen 2001]] 与 [[RawSource/articles/Muller-2006-PositionBasedDynamics-中文翻译|Müller 2006]] 时对 "Gauss-Seidel 型松弛求解器" 的数学深究沉淀（2026-10-21/22 会话），覆盖：线性代数地基 → 分裂法 → 收敛性三路线 → **非线性侧：PBD 循环投影的收敛性（§4）** → PBD 落点。

## 0. 一图定位

```
A x = b（PBD：约束线性化后的稀疏系统，b = 约束违反量）
 │ 分裂 A = D − L − U（对角/下三角/上三角；本约定 L,U 存 −a_ij）
 │ 不动点改造：A = Q − (Q−A) ⇒ x⁽ᵏ⁺¹⁾ = Q⁻¹(Q−A)x⁽ᵏ⁾ + Q⁻¹b
 │
 ├─ Q = D     → Jacobi：整行读旧值 → 分量零依赖 → 可并行，慢
 ├─ Q = D−L   → Gauss-Seidel：下方读新值 → 嵌套依赖 → 须串行，快
 └─ Q = 松弛版 → SOR（GS + 外推 ω）
 
收敛 ⟺ ρ(M) < 1（谱半径判据；三条充分条件路线见 §3）
```

## 1. 分裂法与两种迭代

### 1.1 定义与迭代矩阵

**Jacobi**——第 $i$ 行方程右边全部用上一轮的值：

$$x_i^{(k+1)} = \frac{1}{a_{ii}}\Big(b_i - \sum_{j \neq i} a_{ij}x_j^{(k)}\Big), \qquad M_J = D^{-1}(L+U)$$

**Gauss-Seidel**——编号更小的分量用本轮刚算出的新值：

$$x_i^{(k+1)} = \frac{1}{a_{ii}}\Big(b_i - \sum_{j<i}a_{ij}x_j^{(k+1)} - \sum_{j>i}a_{ij}x_j^{(k)}\Big), \qquad M_{GS} = (D-L)^{-1}U$$

一句话对比：**Jacobi 是"全班同时交卷再对答案"，GS 是"做完一题立刻把答案抄给下一题的人"**。GS 的"求逆"是抽象缩写——算法上是**向前代入**（逐行剥皮，$O(n^2)$；带宽 $b$ 的稀疏阵 $O(nb^2)$），显式算逆（$O(n^3)$、稠密化、误差多一个 $\kappa$）永远是坏主意。

### 1.2 为什么 $Q$ 必须可逆、左乘 $Q^{-1}$ 为什么合法

- **相等的东西做同一操作仍相等**（代入性质，逻辑公理，无条件）；
- **$Q$ 可逆 ⟺ 操作不丢信息**：$Q\mathbf{u} = Q\mathbf{v} \Rightarrow \mathbf{u} = \mathbf{v}$（$\ker Q = \{\mathbf{0}\}$），新旧方程**等价**（解不多不少）。左乘奇异矩阵等式仍成立但会**增解**（丢信息），如 $2x=6$ 两边乘 $0$ 得 $0=0$；
- **必须左乘**：列向量右乘维度不匹配；矩阵不可交换 ⟹ 两边必须同侧；
- 落地靠**结合律**：$Q^{-1}(Q\mathbf{x}) = (Q^{-1}Q)\mathbf{x} = I\mathbf{x}$；
- $D$ 可逆 ⟺ 对角元全非零；$D - L$ 下三角可逆 ⟺ 同一条件（$\det = $ 对角元之积）——**"对角元非零"一条全包**。可逆 ⟺ $\det\neq 0$ ⟺ 满秩 ⟺ 零空间 $\{0\}$ ⟺ 无零特征值 ⟺ $Ax=b$ 恒有唯一解（可逆性定理，等价刻画一族）。

### 1.3 三角方程的剥皮（GS 的真实算法形态）

$$y_i = \frac{1}{t_{ii}}\Big(r_i - \sum_{j<i}t_{ij}y_j\Big), \quad i = 1\dots n$$

第 1 行只含 $y_1$ → 直接除；第 $i$ 行只用已解出的分量。**GS 逐分量公式 ≡ $(D-L)^{-1}U$ 矩阵公式**——两种记法、同一个计算；矩阵式用于谱半径分析，逐分量式用于写代码。3×3 数值例（$A = \mathrm{tridiag}(2,10,2)$ 型）：剥皮三步六次乘加解完，与 GS 一轮手算逐位吻合。

## 2. 误差动力学

固定点相减消去 $\mathbf{c}$：$\mathbf{e}^{(k+1)} = M\,\mathbf{e}^{(k)} = M^k\mathbf{e}^{(0)}$。可对角化时按特征基展开 $\mathbf{e}^{(0)} = \sum c_i\mathbf{v}_i$：

$$\mathbf{e}^{(k)} = \sum_i \lambda_i^k\,c_i\,\mathbf{v}_i \qquad \text{矩阵的幂 = 各特征方向上的幂}$$

## 3. 收敛性：三条证明路线

**GS 并非无条件收敛**（反例：$\begin{pmatrix}1&2a\\a&1\end{pmatrix}$，$a>1$ 时对称不定、$\rho_{GS} = 4a^2 > 1$ 发散）。收敛前提三选一：

### 路线一：严格对角占优 → 嵌套收缩（归纳法，无特征值）

前提：$|a_{ii}| > \sum_{j\neq i}|a_{ij}|$，即 $\beta_i + \gamma_i < 1$（按 GS 读数位置把该行非对角元劈成下方 $\beta$ / 上方 $\gamma$——**这就是对角占优本身**，不是另一个条件）。

GS 更新的误差传递是嵌套的（下方读新误差、上方读旧误差），归纳构造收缩因子：

$$s_1 = \gamma_1, \qquad s_i = \gamma_i + \beta_i\max_{j<i}s_j \;\xrightarrow{\text{两行归纳}}\; s_i < 1$$

（奠基：$s_1 \leq \beta_1+\gamma_1 < 1$；递推：$s_i < \gamma_i + \beta_i < 1$。）结论：

$$\|\mathbf{e}^{(k+1)}\|_\infty \leq \Big(\max_i s_i\Big)\|\mathbf{e}^{(k)}\|_\infty = s\,\|\mathbf{e}^{(k)}\|_\infty, \quad s < 1$$

**$s_i$ 递推里 $\beta_i$ 乘的是"前面的 $s$"而非 $1$**——正是这一替换让 GS 的界严格压过 Jacobi 的 $\max_i(\beta_i+\gamma_i)$。

**$s_j$ 的几何意义：扫描顺序造成的误差新鲜度分级**。一轮 GS 扫描中，已覆写区 $[1..i-1]$ 的误差打了 $s_j$ 折（新值）、未覆写区 $[i+1..n]$ 满血（旧值）；**每写一个分量就为后续行制造一个打折感染源**，误差穿越下三角一次被压一轮。3×3 数值例：$s = (0.2, 0.24, 0.048)$，实测一轮收缩 0.2 ✓；同一行读新值 vs 旧值误差差 6 倍——$s$ 的可见形状。

**占优从哪来（不是被保证、是被检查的）**：物理来源 = 质量矩阵对角项、正则化（XPBD 的 $\tilde\alpha$）、稀疏局部支撑、阻尼项。**经典链问题 $\mathrm{tridiag}(-1,2,-1)$ 只弱占优**（$2 = 1+1$），靠"弱占优 + 不可约"定理兜底——路线一的前提在模型问题上都要打折，真实世界更常用路线三。

### 路线二：谱半径判据（最一般）

$$\text{对一切初值收敛} \iff \rho(M) = \max_i|\lambda_i(M)| < 1$$

证明（可对角化情形）：$\Leftarrow$ 特征基展开有限和逐项归零；$\Rightarrow$ 反证取 $\mathbf{e}^{(0)} = \mathbf{v}_j$ 沿坏方向永不衰减。不可对角化用 Jordan 形 + "指数碾死多项式"补全；Gelfand 公式 $\lim\|M^k\|^{1/k} = \rho(M)$ 统一一切。

收敛速率 $R = -\ln\rho$，压小 $\varepsilon$ 倍需 $k \approx \ln(1/\varepsilon)/R$ 步。**任一具体范数可能骗人**（$\|M\| > 1$ 不代表发散），只有谱半径说了算。

**适用范围警示（会话中更正过的错误）**：$\rho_{GS} = \rho_J^2$（"快两倍"）**只在 Young 1952 的相容排序矩阵类上成立**（三对角属于此类）；一般矩阵上 GS 快于 Jacobi 是典型经验**而非定理**。3×3 例：$\rho_J = 0.4$、$\rho_{GS} = 0.16 = 0.4^2$（理论适用 ✓）。

#### 谱半径的几何图像（2026-10-22 补充，会话沉淀）

**主轴图框**：把 $M$ 看作"单位球的变形器"——它把单位圆捏成椭圆，特征方向 = 椭圆主轴（不拐弯的方向），特征值 = 该主轴的缩放率。$M^k$ 的几何 = **每根主轴各自独立做 $k$ 次指数运动**（对称情形 $\mathbf{e}^{(0)}$ 按正交特征基分解 $\sum c_i\mathbf{v}_i$，各分量互不干扰地乘 $\lambda_i^k$）。

**灯喻**：把误差按特征方向点亮 $n$ 盏灯，每轮每盏亮度乘 $|\lambda_i|$——

- $\rho < 1$：所有灯都在变暗，$\rho$ 是**最暗得最慢的那盏**（误差的最后防线），渐近衰减率由它独家决定；快分量早死、慢分量苟到最后；
- $\rho \geq 1$：至少一盏恒亮或变亮，屋子永远暗不下来——**哪怕其他灯全灭了**。收敛是"全维度萎缩"的合取条件，**一票否决，所以判据取 max 而非平均/最小**。

**为什么范数骗人而谱半径不骗**：剪切 + 收缩的合成矩阵可以 $\|M\|_2 > 1$ 依然收敛（单步把斜向量的长度暂时抻大，但每次都过一遍 $|\lambda|<1$ 的特征缩放，指数规律长期碾压）。范数看到的是"单步最大拉伸"（上界估计器），谱半径看到的是"长期指数趋势"（渐近真值）；Gelfand 公式 $\lim\|M^k\|^{1/k} = \rho$ 正是说 **$k$ 步平均单步拉伸率收敛于 $\rho$**——单步欺瞒在长期平均下被洗掉。

**布料拉伸的谱半径画像**：链长 $n$ 时 $\rho_J = \cos\frac{\pi}{n+1}$，最顽固误差是"整链同向摆"的低频驻波模式，链越长该方向的椭圆主轴越接近"不收缩"——衰减到 1% 的轮数 $\ln 100/(-\ln\rho)$：$n{=}10$ 约 56 轮、$n{=}100$ 约 9200 轮、$n{=}1000$ 约 92 万轮（$\sim n^2$）。有限迭代 = 误差没传到远端 = 布料拉伸。

#### 谱半径判据 FAQ：三个最容易复发的理解偏差（2026-10-22 会话沉淀）

**Q1：「谱半径就是最大特征值」——差"模长"二字。**

$$\rho(M) = \max_i \mathbf{|}\lambda_i\mathbf{|} \qquad \text{先取绝对值再取最大}$$

特征值可以是负数甚至复数，而收敛看的是**长度**不是符号。$\lambda_1 = -0.9$：误差沿该方向每轮翻符号振荡、但长度 ×0.9——**振荡着萎缩也是萎缩**，照样收敛、速率按 0.9 算；$\lambda = 0.8i$：转着圈萎缩，速率按 0.8 算。"模长"把振荡/旋转/单调三种衰减统一成一个数。

**Q2：「只要 < 1 就收敛」——对，但要拧三个螺丝。**

1. 宾语是**误差**不是解：$\mathbf{x}^{(k)} \to \mathbf{x}^*$ ⟺ $\mathbf{e}^{(k)} \to \mathbf{0}$，且**前提是不动点确为解**（分裂格式构造保证）；
2. 这是**充要**（$\iff$），不是充分条件：$\rho \geq 1$ 时必然存在不收敛的初值（放在坏特征方向上即可）。对比：对角占优/SPD 是**门票**（充分），谱半径是**终点线本身**（充要）；
3. 量词"**对一切初值**"不可省：$\rho = 2$ 的矩阵对初值 $(0,1)$（坏方向分量恰为零）照样收敛，但工程上没有初值保证。

另注意适用面：判据只对**定常线性迭代**成立——非线性迭代（PBD 每轮重新线性化）没有单个 $M$，判据只能用在局部线性化上（非线性侧的完整理论见 §4）。

**Q3：「< 1 就够了」——渐近赢 ≠ 预算内赢。**

| $\rho$ | 压到 1% 所需轮数 |
|---|---|
| $0.5$ | 7 |
| $0.9$ | 44 |
| $0.99$ | 459 |
| $0.999$ | 4,603 |
| $\to 1^-$ | $\to \infty$ |

$\rho = 0.999$ 数学收敛、工程上近乎不收敛。**判据回答"会不会"，$\rho$ 的数值回答"几轮才够"**——长链布料（$\rho$ 贴 1）正是"数学收敛、预算内不收敛"的典型，实践永远看 $\rho$ 离 1 多远。

### 路线三：SPD → 能量范数单调（Ostrowski–Reich，PBD 最相关）

**$A$ 对称正定 ⟹ GS 必收敛**（Jacobi 不必）。证明框架：能量内积/范数 $\|\mathbf{u}\|_A^2 = \mathbf{u}^TA\mathbf{u}$（正定性使之为真内积），误差一步 $\mathbf{e}^{(k+1)} = \mathbf{e}^{(k)} - \mathbf{d}^{(k)}$（$\mathbf{d} = (D-L)^{-1}A\mathbf{e}$），平方展开：

$$\|\mathbf{e}^{(k+1)}\|_A^2 = \|\mathbf{e}^{(k)}\|_A^2 - 2\langle\mathbf{e},\mathbf{d}\rangle_A + \|\mathbf{d}\|_A^2$$

核心引理为 $2\langle\mathbf{e},\mathbf{d}\rangle_A \geq \|\mathbf{d}\|_A^2$（"$D{-}L$ 可逆且 $(D{-}L)^{-1}A$ 对称部分正定"，完整证明见 Saad《Iterative Methods》§4 / Axelsson–Barker），由此得**单调下降**：

$$\|\mathbf{e}^{(k+1)}\|_A^2 \leq \|\mathbf{e}^{(k)}\|_A^2 - \|\mathbf{d}^{(k)}\|_A^2$$

**PBD 相关性**：XPBD 乘子系统 $\nabla C M^{-1}\nabla C^T + \tilde\alpha$ 是 SPD（Gram 矩阵 + 正则化）——路线三的前提是 PBD 物理设定的天然产物，**这也是 XPBD 分母永不归零、数值稳定的结构性原因**。能量范数 = 势能差，单调下降 = 数值耗散 = "PBD 越迭代越僵越闷"（XPBD §6.4、Small Steps 图 6）的**第一性解释**。

## 4. 非线性收敛：PBD 的投影几何视角（2026-10-26 会话沉淀）

> §3 的三条路线全部针对**定常线性迭代**（存在固定 $A$、固定迭代矩阵 $M$）。PBD 的 GS 每轮每约束在新位置**重新线性化**、系统非对称、混合等式/不等式、$M \neq 3N$——线性定理只能做不动点附近的**局部线性化分析**（谱半径画像、"smoother" 论断的落点）。本节回答两个问题：PBD 有没有自己的收敛定理？约束要满足什么条件才能收敛？文献映射（Bregman / Gubin–Polyak–Raik / KM / Lewis–Luke–Malick / BCD）由数值分析与凸分析侧接入，**PBD 原论文与 [[RawSource/articles/Bender-2017-PBDTutorial-CourseNotes-中文翻译|EG 2017 讲义]]均未引用**——这是给框架"正名"，不是原文内容。

### 4.0 先拆开三种"收敛"——PBD 各占哪层

| 层次 | 含义 | PBD 的状况 |
|---|---|---|
| ① 稳定（不发散） | 位置/速度有界，永不爆炸 | ✅ Müller 2006 的 "unconditionally stable" 指的就是这层 |
| ② 迭代收敛 | 固定时间步内 $n_s \to \infty$，位置序列趋于不动点 | ⚠️ **部分有**——取决于约束几何（本节主题） |
| ③ 离散化收敛 | 网格细化时解趋近连续物理 | ❌ 讲义 §8 原话 "not convergent… with mesh refinement" |

**①不蕴含②**：有界序列可以振荡、循环、停在不满足约束处。①的机制：投影修正量以当前违反量为上界（$\|\Delta x\| \lesssim |C|$，非扩张性），速度由位置差导出——位置有界 → 速度有界。**PBD 论文从未声称②。**

### 4.1 PBD 一次扫掠 = 逆质量度量下的循环投影

把每个约束的解写成投影算子（在以 $M$ 加权内积的 Hilbert 空间里）：

$$P_j(\mathbf{x}) = \arg\min_{\mathbf{y}}\ \tfrac12(\mathbf{y}-\mathbf{x})^T M (\mathbf{y}-\mathbf{x}) \quad \text{s.t.}\ C_j(\mathbf{y}) \succ 0$$

一次 GS 扫掠 = **循环投影** $\mathbf{x}^{(k+1)} = P_M \circ \cdots \circ P_1(\mathbf{x}^{(k)})$。关键观察：PBD 的标准修正 $\Delta\mathbf{x} = \lambda M^{-1}\nabla C^T$（讲义式 (6)）**正是 $M^{-1}$-度量投影的方向**——"修正沿梯度 + 逆质量加权"不是工程巧合，是换内积的正交投影。因此 1960 年代的凸分析定理直接可用：

### 4.2 三条定理（按约束几何强度递减）

**A1 · 凸集循环投影（Bregman 1965；Gubin–Polyak–Raik 1967）**：每个 $\{C_j \succ 0\}$ 是**闭凸集** ⟹ 循环投影必收敛——交非空时收敛到交内一点；两组不一致凸集收敛到最小距离对（最小化违反的折中点，= Goldenthal 伪逆解的几何版本）。

**A2 · Krasnoselskii–Mann 平均（k<1 的定理级解释）**：非扩张映射 $T$ 的 averaged 迭代

$$\mathbf{x} \leftarrow (1-k)\,\mathbf{x} + k\,T(\mathbf{x}), \qquad k \in (0,1)$$

在不动点集非空时必收敛。**PBD 的刚度缩放恰是此形式**（$\mathbf{x} \leftarrow \mathbf{x} + k(P\mathbf{x}-\mathbf{x})$）——$k=1$ 的纯投影可以循环不收敛，averaged 版本被 KM 兜住。"$k$ 小更稳"由此有了定理背书。Jacobi 并行版的约束平均 $\Delta\tilde{x}_i = \frac{1}{n_i}\Delta x_i$（讲义式 (48)）= 欠松弛 = A2 的 Jacobi 版，讲义 §6.1.2 承认无平均时 "may not converge at all"。

**A3 · 非凸集局部结果（Lewis–Luke–Malick 2009 等，prox-正则集理论）**：球面这类非凸但 prox-正则的集合：两约束流形**横截相交**（交角 $\theta > 0$）⟹ 交替投影**局部线性收敛**，收缩率量级 $\sim \cos^2\theta$；$\theta \to 0$（相切）时退化。**这就是 §3 谱半径画像的非线性对应物**：链被拉直 = 距离约束流形近相切 = $\rho_J = \cos\frac{\pi}{n+1} \to 1$——两个视角讲的是同一件事。

### 4.3 能量路线：XPBD 把 GS 变成坐标下降（路线三的非线性版）

XPBD 给出能量泛函后，逐约束 GS = **循环块坐标下降（cyclic BCD）**：

$$\min_{\mathbf{x}}\ F(\mathbf{x}) = \tfrac12(\mathbf{x}-\tilde{\mathbf{x}})^T M (\mathbf{x}-\tilde{\mathbf{x}}) + \sum_j \frac{C_j(\mathbf{x})^2}{2\alpha_j}$$

每个 XPBD 约束步是该块的（近似）精确最小化 ⟹ $F$ **单调不增** ⟹ 紧性下收敛到驻点；凸能量到全局最小（Luo–Tseng 型正则条件给线性速率）。且 XPBD 分母 $\nabla C M^{-1}\nabla C^T + \tilde\alpha$ 是 **Gram 矩阵 + 正则化 = SPD**——§3 路线三（Ostrowski–Reich 能量单调）的前提在 XPBD 里是**结构性质而非假设**。相关文献坐标：**Liu 2013**（讲义 [LBOK13]，§4.2.4 变分联系出处）证明 PBD 迭代极限 = 隐式 Euler 约束流形投影解；**Goldenthal 2007** 给 Newton + 投影收敛论证；**Projective Dynamics [BML∗14]** 是本族证明最干净者（局部-全局交替 = MM 算法，目标单调必收敛），代价是约束类型受限（需投影代理）。

### 4.4 约束应满足的条件 + 常用约束逐个过堂

条件按强度递减：**可行性**（可行集交非空，矛盾静止长度 → 无解 → 收敛到极限环/折中点，这是最小二乘行为不是 bug）→ **凸性**（全局保证）→ **平滑 + 横截角 > 0**（非凸局部线性保证）→ **单约束闭式可解**（沿梯度线性的约束精确一步；二次以上约束只要求每步下降）。

| 约束（讲义 §5） | 可行集形状 | 凸？ | 理论待遇 |
|---|---|---|---|
| 单边距离 $\|x_{12}\| \le d$（LRA、应变限幅） | 球 | ✅ | **POCS 全局收敛（A1）**——LRA 稳健性的深层原因 |
| 双边距离 $= d$ | 球面 | ❌（prox-正则） | 横截时局部线性（A3）；拉直 = 近相切 = 退化（$\rho\to1$） |
| 接触半空间 $n^Tx \ge d$（式 25） | 半空间 | ✅ | 全局（A1） |
| 四面体体积 $= V_0$ | 多仿射水平集 | ❌ | 近解良好；翻转态无保证 |
| 二面角弯曲 $= \phi_0$ | 非凸流形 | ❌ | **最易卡局部极小**（布料粘在折叠态的名场面） |
| 密度约束（PBF，式 41） | 非凸 | ❌ | 仅经验稳定 |

### 4.5 翻车模式 ↔ 工程对策（PBD 文献的"病理-药方"对照）

| 病理 | 定理解释 | 讲义中的药方 |
|---|---|---|
| Jacobi 并行发散 | 非凸/不定系统无 SPD 兜底 | 约束平均（式 48，= A2 平均化）、SOR $\omega \le 2$（§6.1.2） |
| 低频误差收敛慢 | smoother 本性；$\rho\to1$ | HPBD 多网格（§4.2.3）、LRA 换凸约束（§5.6） |
| 卡在非凸局部极小 | 无 A1/A3 保证 | 空气网格的"记忆"（§5.5.4） |
| 提前终止 = 人工柔度 | 未达能量最小点 | XPBD 收敛点良定义；[[Macklin-2019-SmallSteps\|Small Steps]] 子步化把线性化误差 $\propto\Delta t^2$ 压死 |

**一句话**：PBD 的"无条件稳定"只是①有界性；迭代收敛性是"按约束几何定价的合同"——凸集全局保（POCS），非凸但横截局部保（$\cos^2\theta$），相切/矛盾/强非凸各有各的翻车法；XPBD 的贡献是让 GS 成为能量单调下降的 BCD，把路线三恢复为结构性质。

## 5. 与 PBD 的对应总表

| 线性世界 | PBD 世界 |
|---|---|
| 解一行方程 | 解一个约束（单约束 Newton 步 $\Delta\mathbf{x} = -\frac{C_j}{\sum w\lvert\nabla C_j\rvert^2}w\nabla C_j$） |
| $A$ 稀疏（链=三对角） | 约束只牵涉 2~4 粒子 |
| Jacobi 并行 | GPU 求解器所有约束同时读数 + **约束平均**（防共享粒子过冲，Macklin 2014） |
| GS 串行 | 经典 PBD 的 Projected Gauss-Seidel（PGS） |
| 对角元 / 预条件 | 逆质量 $w_i$（对角预条件；$w{=}0$ 钉死=消元） |
| $\rho\to 1$ 的长链 | 布料拉伸（$\rho_J = \cos\frac{\pi}{n+1} \to 1$，收敛 $O(n^2)$ 轮）——HPBD/[[Kim-2012-LongRangeAttachments\|LRA]]/[[Macklin-2019-SmallSteps\|Small Steps]] 的共同靶子 |
| SOR 松弛 $\omega$ | Jakobsen §7 提示的 over-relaxation；$\omega_{opt}$ 模型问题上 $O(n^2)\to O(n^{3/2})$ |
| 预分解 + 回代（全局法） | Projective Dynamics/PCG——撕裂要重分解，XPBD 局部迭代对此免疫（XPBD §2 的对比论点） |
| 条件数 $\kappa$ 大 → 残差小 ≠ 误差小 | "看起来静止了但布料还拉伸着" |
| 循环投影 POCS（凸集） | PBD 扫掠本身 = 逆质量度量循环投影；凸约束（单边距离/接触半空间）全局收敛（§4.2 A1） |
| 横截角 $\theta \to 0$（流形相切） | 非凸流形近相切 = 长链拉直 = $\rho\to1$ 的几何版本（§4.2 A3） |

## 6. 手算基准例（3×3，全页通用）

$$A = \begin{pmatrix}10&2&0\\2&10&2\\0&2&10\end{pmatrix}, \quad \mathbf{b} = (12,14,12)^T, \quad \mathbf{x}^* = (1,1,1)^T, \quad \mathbf{x}^{(0)} = \mathbf{0}$$

| 量 | Jacobi | Gauss-Seidel |
|---|---|---|
| 2 轮后最大误差 | $0.4 \to 0.08$ | $0.2 \to 0.032$ |
| 收缩界 | $\max(\beta_i{+}\gamma_i) = 0.4$ | $\max s_i = 0.24$ |
| 谱半径 | $0.4$ | $0.16 = 0.4^2$ |

GS 两轮精度 ≈ Jacobi 四轮；剥皮解 $(D{-}L)\mathbf{y} = \mathbf{b}$ 得 $(1.2, 1.16, 0.968)$ 与 GS 一轮逐位吻合。

## 7. 关联

- **上游阅读**：[[RawSource/articles/Jakobsen-2001-AdvancedCharacterPhysics-中文翻译|Jakobsen 译本]] §4（松弛=GS 的工程版）、[[RawSource/articles/Muller-2006-PositionBasedDynamics-中文翻译|Müller 2006 译本]] §3.2（"修正立即可见加速收敛"的谱半径解释）
- **直接下游**：[[PositionBasedDynamics]] / [[XPBD]]（求解器底座）、[[Macklin-2019-SmallSteps|Small Steps]]（把胜负从迭代法内战转移到时间离散）、[[Muller-2008-HierarchicalPBD|HPBD]] 与 [[Kim-2012-LongRangeAttachments|LRA]]（治 $\rho\to1$ 长链病的两条路）
- **配套概念**：Verlet 积分推导（Taylor 展开相加消速度项、辛结构、$O(\Delta t^2)$ 全局误差）——同为本次会话数学地基，暂未独立成页，需要时补 `VerletIntegration.md`
- **延伸阅读**：Saad《Iterative Methods for Sparse Linear Systems》§4（Ostrowski 定理完整证明）、Young 1952（相容排序）、Axelsson–Barker
- **§4 文献锚点**（原文未引、外侧接入）：Bregman 1965 / Gubin–Polyak–Raik 1967（凸集循环投影）；Krasnoselskii–Mann（平均迭代）；Lewis–Luke–Malick 2009（非凸 prox-正则集横截角局部线性收敛）；Bauschke–Borwein 1996 综述（POCS 全景）；Luo–Tseng 1993（BCD 线性速率）；Liu et al. 2013（讲义 [LBOK13]，PBD=隐式 Euler 投影）；Goldenthal et al. 2007（快速投影）；Bouaziz et al. 2014（Projective Dynamics，MM 单调收敛）
