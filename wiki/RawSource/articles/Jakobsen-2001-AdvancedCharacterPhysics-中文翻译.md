---
type: translation
title: Advanced Character Physics（Jakobsen 2001）中文译本
source: RawSource/articles/Jakobsen-2001-AdvancedCharacterPhysics.txt
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, pbd, verlet, hitman, game-physics, gdc-2001, translation]
summary: Jakobsen（IO Interactive）GDC 2001 经典中文学习译本：《杀手 47》角色物理——Verlet 积分隐式存速度、投影处理碰撞、松弛迭代解约束、粒子+约束表示刚体与关节体、棍图粒子骨架模拟人体。PBD 的直接祖先。原始英文见 [[Jakobsen-2001-AdvancedCharacterPhysics]]。
---

# Advanced Character Physics — 中文译本

> [!info] 译本说明
> 本文件是原文的中文学习译本，供个人研读使用。公式用 LaTeX（`$...$` / `$$...$$`），伪代码用代码块，正文忠实传达原文论述，关键术语保留英文对照。原始英文提取文本见 [[Jakobsen-2001-AdvancedCharacterPhysics|原索引卡]] 所指的 `Jakobsen-2001-AdvancedCharacterPhysics.txt`。
>
> 原文为 GDC 2001 会议文章 / Game Developer Magazine 刊载，官方链接与各大学镜像均已失效；本文取自 [gotoandplay.it 2005 年转载](http://www.gotoandplay.it/_articles/2005/08/advCharPhysics.php)（两页）。按学术学习使用处理。
>
> **关键术语对照**：Verlet integration=Verlet 积分；penalty-based scheme=惩罚法（罚函数法）；projection=投影；relaxation=松弛（迭代）；stick constraint=棍约束（距离约束）；unilateral/bilateral constraint=单向/双向约束（不等式/等式约束）；articulated body=关节体（铰接体）；penetration depth=穿透深度；restitution coefficient=恢复系数；resting contact=静接触。
>
> **译注（谱系定位）**：本文是 [[PositionBasedDynamics|PBD]]（Müller 2006）的直接前身——文中"棍约束松弛"的伪代码正是 PBD 论文式 (10)(11) 的原型；"看似天真的重复实际收敛"即 Gauss-Seidel 迭代。对照阅读：[[Muller-2006-PositionBasedDynamics-中文翻译|PBD 2006 中文译本]] §3.1 把同一循环写成显式速度版本。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | Advanced Character Physics |
| 作者 | Thomas Jakobsen（IO Interactive 研究主管；丹麦技术大学数学博士） |
| 发表 | GDC 2001 / Game Developer Magazine |
| 应用 | 《杀手：代号 47》(Hitman: Codename 47)，IO 自研引擎 Glacier |

## 摘要

本文讲解一套适合交互使用的基于物理建模方法的基本要素。它简单、快速且相当稳定；其基础版本不需要高级数学知识（尽管背后有坚实的数学基础）。它可以模拟布料、软体与刚体，甚至可以用正向与逆向运动学模拟关节体（铰接体）。

这些算法为 IO Interactive 的游戏《杀手：代号 47》而开发。在那里，物理系统负责布料与植物的运动、刚体，以及让死人尸体依被击中的部位以独特方式倒下、与环境完全交互（由此产生媒体上的矛盾修辞"栩栩如生的死亡动画"）。

本文还处理穿透测试优化与摩擦处理等细节。

## 1. 引言

用基于物理的建模产生好看的动画已被考虑多时，现有技术大多相当精巧。文献中提出了多种方法 [Baraff, Mirtich, Witkin 等]，大量精力投入在构造精确可靠的算法上。事实上，物理与动力学的精确仿真方法在工程界早已熟知。然而对游戏与交互应用而言，**精度并不是首要目标**（尽管有当然好）——这里的重要目标是**可信度**（只要玩家仍感沉浸，程序员想怎么作弊都行）与**执行速度**（每帧只有一定的时间分给物理引擎）。就物理仿真而言，"可信度"也涵盖稳定性：若物体似乎漂过障碍物、或该静止时却在振动、或布料粒子动不动"爆炸"，这方法就不行。

本文展示的方法就是为达到这些目标而生。算法由作者为 IO Interactive 的《杀手：代号 47》开发实现，全部集成进 IO 自研引擎 Glacier。这些方法实现起来相当简单（至少与其他方案相比），性能很高。

算法是迭代的，从某一点起可以随时停下。这给出非常有用的时间/精度权衡：若接受一点不精确，代码就能跑得更快；这个误差容限甚至可在运行时自适应调整。某些情形下，该方法比其他现有方法快一个数量级。它在同一框架内同时处理碰撞与静接触，并能从容应对箱体堆叠等考验物理引擎的场景。

总览来说，方法的成功来自以下数项技术的正确组合，它们互相成就：

- 一套所谓的 Verlet 积分方案；
- 用投影处理碰撞与穿透；
- 一个用松弛法的简单约束求解器；
- 一个带来实在加速的漂亮平方根近似；
- 把刚体建模为带约束的粒子；
- 一个能计算穿透深度的优化碰撞引擎。

上述每一项都会很快讲解。写作本文时，作者试图让它对最广的受众可读，同时不丢失实现所必需的关键信息。这意味着技术性的数学解释与概念被压到最少——除非对理解主题至关重要。目标是示范：不陷入大量数学繁难，也能实现相当先进且稳定的物理仿真。

内容组织如下：第 2 节描述粒子系统的"无速度"表示——它有若干优点，最显著的是稳定性、以及约束实现简单这一事实。第 3 节描述碰撞如何处理。第 4 节用约束扩展粒子系统，使我们能建模布料。第 5 节解释如何设置适当约束的粒子系统来模拟刚体。第 6 节演示如何进一步扩展系统以支持关节体（即带角度等约束的互连刚体系统）。第 7 节包含各种札记并分享实现摩擦等方面的经验。最后第 8 节简短结论。

下文中，粗体表示向量。向量分量用下标索引，即 $\mathbf{x} = (x_1, x_2, x_3)$。

## 2. Verlet 积分

仿真的心脏是一个粒子系统。通常在粒子系统的实现里，每个粒子有两个主要变量：位置 $\mathbf{x}$ 与速度 $\mathbf{v}$。然后在时间步循环中，按如下规则计算新位置 $\mathbf{x}'$ 与新速度 $\mathbf{v}'$：

$$\mathbf{x}' = \mathbf{x} + \mathbf{v}\,\Delta t$$
$$\mathbf{v}' = \mathbf{v} + \mathbf{a}\,\Delta t$$

其中 $\Delta t$ 是时间步，$\mathbf{a}$ 是按牛顿定律 $\mathbf{f} = m\mathbf{a}$ 算出的加速度（$\mathbf{f}$ 为作用在粒子上的累积力）。这是简单的 Euler 积分。

而这里我们选择**无速度表示**与另一种积分方案：不存储每个粒子的位置和速度，而是存储其当前位置 $\mathbf{x}$ 与前一位置 $\mathbf{x}^*$。保持时间步固定，更新规则（积分步）为：

$$\mathbf{x}' = 2\mathbf{x} - \mathbf{x}^* + \mathbf{a}\,\Delta t^2$$
$$\mathbf{x}^* = \mathbf{x}$$

这叫 **Verlet 积分**（见 [Verlet]），在分子动力学模拟中被大量使用。它相当稳定——因为速度是隐式给出的，速度与位置更难失同步。（顺带一提，知名的水面涟漪 demo 特效用的就是类似方法。）它成立的原因：$2\mathbf{x} - \mathbf{x}^* = \mathbf{x} + (\mathbf{x} - \mathbf{x}^*)$，而 $\mathbf{x} - \mathbf{x}^*$ 是当前速度的近似（实际是上一时间步走过的距离）。它并不总是很精确（能量可能离开系统，即耗散），但快且稳。把系数 2 降低到比如 1.99，还能给系统引入少量拖曳（drag）。

每步结束时，每个粒子的当前位置 $\mathbf{x}$ 被存入对应变量 $\mathbf{x}^*$。注意处理大量粒子时，通过简单交换数组指针即可实现一个有用的优化。

得到的代码大致如下（Vector3 类应包含恰当的成员函数与重载运算符）：

```cpp
// Sample code for physics simulation
class ParticleSystem {
    Vector3  m_x[NUM_PARTICLES];   // Current positions
    Vector3  m_oldx[NUM_PARTICLES]; // Previous positions
    Vector3  m_a[NUM_PARTICLES];   // Force accumulators
    Vector3  m_vGravity;           // Gravity
    float    m_fTimeStep;
public:
    void TimeStep();
private:
    void Verlet();
    void SatisfyConstraints();
    void AccumulateForces();
    //  (constructors, initialization etc. omitted)
};
// Verlet integration step
void ParticleSystem::Verlet() {
    for(int i=0; i<NUM_PARTICLES; i++) {
        Vector3& x = m_x[i];
        Vector3 temp = x;
        Vector3& oldx = m_oldx[i];
        Vector3& a = m_a[i];
        x += x-oldx+a*fTimeStep*fTimeStep;
        oldx = temp;
    }
}
// This function should accumulate forces for each particle
void ParticleSystem::AccumulateForces() {
    // All particles are influenced by gravity
    for(int i=0; i<NUM_PARTICLES; i++)  m_a[i] = m_vGravity;
}
// Here constraints should be satisfied
void ParticleSystem::SatisfyConstraints() {
    // Ignore this function for now
}
void ParticleSystem::TimeStep() {
    AccumulateForces();
    Verlet();
    SatisfyConstraints();
}
```

以上代码为清晰而写，非为速度。一个优化是用 float 数组而非 Vector3 表示状态，这可能也让系统更易在向量处理器上实现。

这听起来还没什么突破性。不过当我们开始使用约束并转向刚体后，优势很快就会显现。届时将演示上述积分方案如何带来更高的稳定性与更少的计算量。

试着设 $\mathbf{a} = (0,0,1)$，用初始条件 $\mathbf{x} = (1,0,0)$、$\mathbf{x}^* = (0,0,0)$，手工做几次迭代看看会发生什么。

## 3. 用投影处理碰撞与接触

所谓**惩罚法**（penalty-based scheme）通过在穿透点插入弹簧来处理接触。实现虽简单，却有若干严重缺点：例如很难选出合适的弹簧常数，使物体既不穿透太深、系统又不失稳。其他一些物理仿真方案里，碰撞通过**时间回退**处理（比如二分搜索回退到精确碰撞点）、从那里解析地处理碰撞、然后重启仿真——从实时角度看这不实用，因为碰撞很多时代码可能跑得极慢。

这里我们用另一种策略：**把越界点直接投影出障碍物**。所谓投影，粗略地说，就是以尽可能小的移动把点移到脱离障碍物为止——通常意味着沿垂直方向朝碰撞表面移动。

看一个例子。假设世界是立方体 $(0,0,0)$–$(1000,1000,1000)$ 的内部，并假设粒子的恢复系数为零（即粒子碰撞后不反弹）。要让所有位置保持在有效区间内，对应的投影代码是：

```cpp
// Implements particles in a box
void ParticleSystem::SatisfyConstraints() {
    for(int i=0; i<NUM_PARTICLES; i++) { // For all particles
        Vector3& x = m_x[i];
        x = vmin(vmax(x, Vector3(0,0,0)), Vector3(1000,1000,1000));
    }
}
```

（vmax 对向量取逐分量最大值，vmin 取逐分量最小值。）这使所有粒子位置保持在立方体内，同时处理了碰撞与静接触。**Verlet 积分方案的美妙之处在于：速度的相应变化会被自动处理**。在随后的 TimeStep() 调用中，速度会被自动调节为不含沿表面法向的分量（对应恢复系数为零）。见图 1。

> **图 1**：十个时间步与两个粒子。

试一试就知道——不需要手动消除法向速度。以上内容对粒子而言看似平凡，但 Verlet 积分的强项正开始显露；等一下引入约束与耦合刚体后会真正显现。

## 4. 用松弛法求解多个并发约束

布料的一个常见模型是由弹簧与粒子互连而成的简单系统。但求解相应的微分方程组并非总是易事。它有着与惩罚法同样的毛病：强弹簧导致刚性方程组，用简单积分技术会不稳定、或至少性能糟糕——带来痛苦；反过来，弱弹簧又导致看起来有弹性的布料。

然而有趣的事情发生了：**若让弹簧刚度趋于无穷，系统突然变得可以用非常简单快速的方法稳定求解**。不过在继续谈布料之前，先回到上个例子。上面那个立方体可视为一组单向（不等式）约束（立方体每面一条），须时刻满足：

$$x_i \geq 0 \text{ 且 } \leq 1000, \quad i = 1, 2, 3 \tag{C1}$$

上例中约束通过把越界位置投影到立方体表面来满足（即把粒子保持在立方体内）。满足 (C1) 的伪代码如下：

```cpp
// Pseudo-code to satisfy (C1)
for i=1,2,3
    set xi=min{max{xi, 0}, 1000}
```

可以把这个过程想象为在粒子与穿透表面之间插入无限刚硬的弹簧——刚度恰好那么强、阻尼恰好合适，以致瞬间恢复到零静止长度。

现在把实验扩展为模拟一根长 100 的棍子。做法是设两个独立粒子（位置 $\mathbf{x}_1$、$\mathbf{x}_2$），要求它们相距 100。数学表达为如下双向（等式）约束：

$$|\mathbf{x}_2 - \mathbf{x}_1| = 100 \tag{C2}$$

虽然粒子初始可能放置正确，但一个积分步后其间距可能已失效。为重新获得正确距离，我们把粒子投影到 (C2) 描述的解集上：直接把粒子相互推开或拉近（取决于错误距离是太短还是太长）。见图 2。

> **图 2**：通过移动粒子修正无效距离。

满足约束 (C2) 的伪代码是：

```cpp
// Pseudo-code to satisfy (C2)
delta = x2-x1;
deltalength = sqrt(delta*delta);
diff = (deltalength-restlength)/deltalength;
x1 -= delta*0.5*diff;
x2 += delta*0.5*diff;
```

注意 delta 是向量，所以 delta*delta 实为点积。restlength=100 时，上述伪代码会把粒子推开或拉近，使它们恢复到正确的距离 100。同样可以把这想象成粒子间被插入了一根静止长度 100 的极硬弹簧，使它们瞬间被摆到正确位置。

现在假设我们仍要粒子满足立方体约束。然而满足棍约束时，可能把某粒子推出了立方体、从而破坏了一条或多条立方体约束。补救办法是立即把越界粒子位置再次投影回立方体表面——但这样又破坏了棍约束。

**理想情况下我们应该同时求解所有约束 (C1) 与 (C2)——那要解一个方程组。但我们选择以局部迭代间接进行**：把两段伪代码简单地先后重复若干次，寄望结果有用。得到的代码如下：

```cpp
// Implements simulation of a stick in a box
void ParticleSystem::SatisfyConstraints() {
    for(int j=0; j<NUM_ITERATIONS; j++) {
        // First satisfy (C1)
        for(int i=0; i<NUM_PARTICLES; i++) { // For all particles
            Vector3& x = m_x[i];
            x = vmin(vmax(x, Vector3(0,0,0)), Vector3(1000,1000,1000));
        }
        // Then satisfy (C2)
        Vector3& x1 = m_x[0];
        Vector3& x2 = m_x[1];
        Vector3 delta = x2-x1;
        float deltalength = sqrt(delta*delta);
        float diff = (deltalength-restlength)/deltalength;
        x1 -= delta*0.5*diff;
        x2 += delta*0.5*diff;
    }
}
```

（两粒子的初始化已省略。）这种纯粹重复的做法看似有点天真，**结果证明它确实收敛到我们要的解**！该方法称为松弛（relaxation）（依具体做法不同也叫 Jacobi 或 Gauss-Seidel 迭代，见 [Press]）。它通过连续满足各个局部约束再反复迭代来工作；条件合适时，这会收敛到同时满足所有约束的全局构型。它同样适用于许多其他需要同时满足多个相互依赖约束的场合。

所需迭代数随被模拟的物理系统与运动量而变，可以通过测量与上次迭代的差值做成自适应的。**若提前停止迭代，结果可能不完全有效，但靠 Verlet 方案，下一帧会更好、再下一帧更好……也就是说提前停止不会毁掉一切**，尽管产生的动画可能显得稍显邋遢。

**布料模拟**

棍约束可视为一根很硬的弹簧，这应当显见了它对本节开头所构布料模拟的用处。例如，假设已构造好描述布料的六角三角形网格。每个顶点初始化一个粒子，每条边在两对应粒子间初始化一条棍约束（约束的"静止长度"就是两顶点间的初始距离）。

然后 HandleConstraints() 对所有约束做松弛。松弛循环可以迭代多次。然而要得到好看的动画，**实际上多数布料只需一次迭代**！这意味着布料模拟的耗时主要取决于 N 次平方根运算与 N 次除法（N 为布料网格的边数）。我们将看到一个聪明的技巧把它降到每帧更新 N 次除法——这真是快得可以，而且可以说大概没法更快了。

```cpp
// Implements cloth simulation
struct Constraint {
    int   particleA, particleB;
    float restlength;
};
// Assume that an array of constraints, m_constraints, exists
void ParticleSystem::SatisfyConstraints() {
    for(int j=0; j<NUM_ITERATIONS; j++) {
        for(int i=0; i<NUM_CONSTRAINTS; i++) {
            Constraint& c = m_constraints[i];
            Vector3& x1 = m_x[c.particleA];
            Vector3& x2 = m_x[c.particleB];
            Vector3 delta = x2-x1;
            float deltalength = sqrt(delta*delta);
            float diff=(deltalength-c.restlength)/deltalength;
            x1 -= delta*0.5*diff;
            x2 += delta*0.5*diff;
        }
        // Constrain one particle of the cloth to origo
        m_x[0] = Vector3(0,0,0);
    }
}
```

现在讨论如何去掉平方根运算。若约束都（至少几乎）被满足，我们其实已经知道该约束表达式里平方根的结果应当是什么——即对应棍子的静止长度 $r$。利用这一点可以近似平方根函数。数学上，我们用平方根函数在静止长度 $r$ 邻域的一阶 Taylor 展开来近似它（等价于以 $r$ 为初值的一次 Newton-Raphson 迭代）。稍加改写得到如下伪代码：

```cpp
// Pseudo-code for satisfying (C2) using sqrt approximation
delta = x2-x1;
delta*=restlength*restlength/(delta*delta+restlength*restlength)-0.5;
x1 -= delta;
x2 += delta;
```

注意若距离已正确（即 $|\boldsymbol{\delta}| = r$），则 delta=$(0,0,0)$，什么都不会变。

现在每条约束用**零次平方根、仅一次除法**，而平方值 restlength\*restlength 甚至可以预计算！耗时运算降到每帧 N 次除法（及相应内存访问）——没法比这更快了，而且效果还挺好看。事实上在《杀手》里，布料模拟的总体速度主要受限于渲染系统能吞吐多少三角形。

仅一次迭代后约束不保证被满足，但靠 Verlet 积分方案，系统会在若干帧内迅速收敛到正确状态。事实上，只用一次迭代加平方根近似，恰好消除了棍子完全刚性时本会出现的刚性（stiffness）问题。

通过在共享邻居的战略顶点对之间放置支撑棍，布料算法可以扩展为模拟植物。同样在《杀手》里，松弛循环只跑一遍就够了（事实上，低迭代数恰好给了植物恰到好处的弯曲行为）。

本节的代码与公式假设所有粒子质量相同。当然也可以建模不同质量的粒子，只是公式稍复杂。要在尊重粒子质量的前提下满足 (C2)，用如下代码：

```cpp
// Pseudo-code to satisfy (C2)
delta = x2-x1;
deltalength = sqrt(delta*delta);
diff = (deltalength-restlength)/(deltalength*(invmass1+invmass2));
x1 -= invmass1*delta*diff;
x2 += invmass2*delta*diff;
```

这里 invmass1、invmass2 是两质量的数值倒数。若想让某粒子不可移动，只需设其 invmass=0（对应无限质量）。当然上式中平方根同样可以近似以加速。

## 5. 刚体

刚体运动方程早在现代计算机发明之前就被发现了。要在那个年代说出有用的结论，数学家需要符号运算能力。刚体理论由此发展出惯性张量、角动量、力矩、表示朝向的四元数等有用的概念与工具。然而，以如今数值处理海量数据的能力，把仿真中的计算分解为更简单的单元已变得可行、某些情况下甚至更有利。就 3D 刚体而言，这意味着**用四个粒子加六条约束建模刚体**（给出正确的自由度数：$4 \times 3 - 6 = 6$）。这简化了大量方面，正是我们将要做的。

考虑一个四面体，四个顶点各放一个粒子；再对四面体的六条边各建一条像上节棍约束那样的距离约束。这就足以模拟刚体了。四面体可以放开在前面那个立方体世界里，Verlet 积分器会让它正确运动。SatisfyConstraints() 要负责两件事：1) 粒子保持在立方体内（如前）；2) 六条距离约束被满足。这同样可用松弛法完成；3~4 次迭代应该够了，还可以选用平方根近似。

显然，一般刚体在碰撞上并不表现得像四面体（虽然运动学上可能像）。还有一个问题：目前刚体与世界外部的碰撞检测只在顶点层面——若发现顶点在世界外就投影回来。只要世界内部是凸的这没问题；若世界非凸，四面体与世界外部可能实际互相穿透而没有任一四面体顶点处于非法区域（见图 3，三角形代表四面体的 2D 类比）。这个问题接下来处理。

> **图 3**：穿透世界的四面体。

我们先考虑问题的简化版本。取之前的棍子例子，假设世界表面有个小凸起。棍子现在可以在两个粒子都没离开世界的情况下穿透世界外部（见图 4）。我们不去深究碰撞检测引擎的构建——那本身就是一门科学。我们假设有一个子系统可用，能检测碰撞；而且该子系统能告知穿透深度并标出两个碰撞物体各自的穿透点。（穿透点与穿透深度的一种定义：穿透距离 $d_p$ 是把其中一物体沿合适方向平移 $d_p$ 后恰好不再穿透的最短距离；穿透点是平移发生后两物体上恰好接触的点。）

> **图 4a / 4b**：碰撞中的棍子（两种情形）。

再看图 4：棍子在 Verlet 步之后穿过了凸起。碰撞引擎已标出两个穿透点 $\mathbf{p}$ 与 $\mathbf{q}$。图 4a 中 $\mathbf{p}$ 恰好就是粒子 1 的位置，即 $\mathbf{p} = \mathbf{x}_1$。图 4b 中 $\mathbf{p}$ 位于 $\mathbf{x}_1$ 与 $\mathbf{x}_2$ 之间、距 $\mathbf{x}_1$ 为棍长的四分之一处。两种情形下 $\mathbf{p}$ 都在棍上，因此可表为 $\mathbf{x}_1$、$\mathbf{x}_2$ 的线性组合 $\mathbf{p} = c_1\mathbf{x}_1 + c_2\mathbf{x}_2$ 且 $c_1 + c_2 = 1$。第一种情形 $c_1 = 1, c_2 = 0$；第二种 $c_1 = 0.75, c_2 = 0.25$。这些值告诉我们对应粒子该移动多少。

要修正棍的无效构型，得把它向上移动。目标是让 $\mathbf{p}$ 移到与 $\mathbf{q}$ 相同的位置以避免穿透。做法：沿 $\boldsymbol{\Delta} = \mathbf{q} - \mathbf{p}$ 的方向调整两粒子 $\mathbf{x}_1$、$\mathbf{x}_2$ 的位置。

第一种情形直接把 $\mathbf{x}_1$ 投影出非法区域（朝 $\mathbf{q}$ 方向）即可，$\mathbf{x}_2$ 不动。第二种情形 $\mathbf{p}$ 离 $\mathbf{x}_1$ 更近，可以论证 $\mathbf{x}_1$ 该比 $\mathbf{x}_2$ 移动得更多。实际上既然 $\mathbf{p} = 0.75\mathbf{x}_1 + 0.25\mathbf{x}_2$，我们选择：$\mathbf{x}_2$ 每移动 0.25，$\mathbf{x}_1$ 就移动 0.75。换言之，新粒子位置由下式给出：

$$\mathbf{x}_1' = \mathbf{x}_1 + 0.75\,\lambda\,\boldsymbol{\Delta}, \qquad \mathbf{x}_2' = \mathbf{x}_2 + 0.25\,\lambda\,\boldsymbol{\Delta} \tag{$*$}$$

其中 $\lambda$ 是待定值。移动两粒子后 $\mathbf{p}$ 的新位置为 $\mathbf{p}' = c_1\mathbf{x}_1' + c_2\mathbf{x}_2'$。

我们要 $\mathbf{p}' = \mathbf{q}$，即 $\lambda$ 应选得使 $\mathbf{p}'$ 恰好与 $\mathbf{q}$ 重合。由于粒子只在 $\boldsymbol{\Delta}$ 方向上移动，$\mathbf{p}$ 也只沿 $\boldsymbol{\Delta}$ 方向移动，于是解方程

$$\mathbf{p}' \cdot \boldsymbol{\Delta} = \mathbf{q} \cdot \boldsymbol{\Delta} \tag{$**$}$$

即可求 $\lambda$。展开左边：

$$\mathbf{p}' \cdot \boldsymbol{\Delta} = (0.75\,\mathbf{x}_1' + 0.25\,\mathbf{x}_2') \cdot \boldsymbol{\Delta} = \mathbf{p} \cdot \boldsymbol{\Delta} + \lambda\,(0.75^2 + 0.25^2)\,\boldsymbol{\Delta}^2$$

与 $(**)$ 右边联立得：

$$\lambda = \frac{(\mathbf{q} - \mathbf{p}) \cdot \boldsymbol{\Delta}}{(0.75^2 + 0.25^2)\,\boldsymbol{\Delta}^2}$$

把 $\lambda$ 代回 $(*)$ 即得使 $\mathbf{p}'$ 与 $\mathbf{q}$ 重合的新粒子位置。

> **图 5a / 5b**：碰撞 I / II 已解决。

图 5 展示移动粒子后的情形：物体不再穿透，但棍长约束又被破坏了。修正办法是再做一次（或几次）松弛循环迭代，然后就完成了。

上述策略对四面体完全类似地适用。先找到穿透点 $\mathbf{p}$ 与 $\mathbf{q}$（它们也可能在三角形内部），把 $\mathbf{p}$ 表为四个粒子的线性组合 $\mathbf{p} = c_1\mathbf{x}_1 + c_2\mathbf{x}_2 + c_3\mathbf{x}_3 + c_4\mathbf{x}_4$ 且 $c_1 + c_2 + c_3 + c_4 = 1$（这需要解一个小线性方程组）。求出 $\boldsymbol{\Delta} = \mathbf{q} - \mathbf{p}$ 后计算：

$$\lambda = \frac{(\mathbf{q} - \mathbf{p}) \cdot \boldsymbol{\Delta}}{(c_1^2 + c_2^2 + c_3^2 + c_4^2)\,\boldsymbol{\Delta}^2}$$

新位置由下式给出：

$$\mathbf{x}_1' = \mathbf{x}_1 + c_1\,\lambda\,\boldsymbol{\Delta}, \quad \mathbf{x}_2' = \mathbf{x}_2 + c_2\,\lambda\,\boldsymbol{\Delta}, \quad \mathbf{x}_3' = \mathbf{x}_3 + c_3\,\lambda\,\boldsymbol{\Delta}, \quad \mathbf{x}_4' = \mathbf{x}_4 + c_4\,\lambda\,\boldsymbol{\Delta}$$

以上是单个刚体与不可移世界的碰撞。该方法可推广到多个刚体间的碰撞：逐对处理碰撞，此时不是只动 $\mathbf{p}$，而是 $\mathbf{p}$ 与 $\mathbf{q}$ 相向移动。调整粒子位置满足非穿透约束后，再处理组成刚体的六条距离约束，依此类推。用这个方法，四面体甚至可以嵌入另一个物体、用后者代替四面体本身处理碰撞。图 6 中四面体嵌入了一个立方体。

> **图 6**：把四面体嵌入另一个物体。

首先立方体得"固定"到四面体上。一种做法是取系统质心 $0.25(\mathbf{x}_1 + \mathbf{x}_2 + \mathbf{x}_3 + \mathbf{x}_4)$ 作为立方体位置，再考察粒子当前位置导出朝向矩阵。发现碰撞/穿透时，碰撞点 $\mathbf{p}$（此时位于立方体上）按上面完全相同的方式处理、相应更新粒子位置。作为优化，立方体所有顶点的 $c_1$–$c_4$ 值可以预计算：若穿透点是顶点则直接查表；否则 $\mathbf{p}$ 在面三角形内部或边上，$c_1$–$c_4$ 可由对应三角形顶点的预计算值插值得到。

通常 3~4 次松弛迭代就够。因为松弛迭代提前终止，物体不会表现得完全刚性。**这多半其实是个优点——世上本无完美刚体，人体尤其不是**。它还让系统更稳定。

通过重排组成四面体的粒子位置，可以相应改变物理属性（数学上，惯性张量随粒子位置与质量改变而改变）。四面体之外的粒子-约束排布也可行，比如把粒子放在坐标系基的模式上，即 $(0,0,0)$、$(1,0,0)$、$(0,1,0)$、$(0,0,1)$。令 $\mathbf{a}$、$\mathbf{b}$、$\mathbf{c}$ 为从粒子 1 分别指向粒子 2、3、4 的向量。约束要求：$\mathbf{a}$、$\mathbf{b}$、$\mathbf{c}$ 长度为 1，且三对向量两两成 90 度角（相应点积为零）。（注意这同样是四粒子六约束。）

## 6. 关节体

多个刚体可以用铰链、销关节等连接。**让两个刚体共享一个粒子，它们就被销关节连接；共享两个粒子，就被铰链连接**。见图 7。

> **图 7a**：销关节。/ **图 7b**：铰链。

也可以用棍约束或任何其他约束连接两个刚体——只需把相应的"修正"代码加进松弛循环即可。

这个方法可以构建完整的人体关节模型。为增加真实感，还需实现各种角度约束。实现方式不一。一种简单方式是使用只在两粒子距离低于阈值时才强制执行的棍约束（数学上是单向距离约束 $|\mathbf{x}_2 - \mathbf{x}_1| > 100$），直接效果是两粒子永不过分靠近。见图 8。

> **图 8**：两条棍约束加一条不等式约束（虚线）。

约束角度的另一种方法是满足点积约束：

$$(\mathbf{x}_2 - \mathbf{x}_0) \cdot (\mathbf{x}_1 - \mathbf{x}_0) < \alpha$$

粒子也可以被限制在例如某些平面内运动。同样地，不满足上述约束的粒子应被移动——具体怎么移比棍约束稍复杂。

实际上，《杀手》里的尸体并不是用四面体刚体组成的，还更简单——它们是由棍约束连接的粒子构成的**棍图（stick figure）**。见图 9。然后每个肢体的位置与朝向（一个向量加一个矩阵）在渲染时由粒子位置用叉积与向量归一导出（确保膝盖与手肘自然弯曲）。

> **图 9**：《杀手》里表示人体解剖结构的粒子/棍构型。

换言之，单独看每个肢体并不是通常 6 自由度的刚体。这意味着**肢体绕自身长轴的旋转在物理上没有被模拟**。取而代之，用于设置角色多边形网格的骨骼动画系统被强制给腿定向、使膝盖看起来自然弯曲。由于腿臂绕长轴的旋转并不构成倒下人体的本质运动，这样做没问题，而且实际上大幅优化了速度。

角度约束用于强制人体解剖结构的限制。简单自碰撞由策略性地引入上述不等式距离约束处理——例如两膝之间——确保双腿永不交叉。

与环境（由三角形组成）的碰撞：每根棍建模为加盖圆柱体（capped cylinder）。碰撞系统的某处有一个子程序处理加盖圆柱与三角形的碰撞。发现碰撞时提取穿透深度与穿透点，然后对肇事棍完全按第 5 节开头描述的方式处理。

自然，为把结果调到恰好，还需要大量额外的微调。

## 7. 札记

本节包含没处安放的各种备注。

**运动控制**

要影响模拟对象的运动，直接相应移动粒子即可。若某人肩部中弹，把肩部粒子沿与打击力度成比例的距离向后移，Verlet 积分器会自动让肩膀进入运动状态。

这也让模拟很容易从底层传统动画系统"继承"速度：只需记录粒子在两帧中的位置，交给 Verlet 积分器，它便自动延续运动。炸弹可以实现为把系统里每个粒子推离爆炸点、距离与粒子到爆心的距离平方成反比。

可以把特定肢体（比如手）约束到空间固定位置。这样就能实现**逆向运动学（IK）**：在松弛循环内部，不断把特定粒子（或多个粒子）的位置设为想要的位置。给粒子无限质量（invmass=0）有助于让它对物理系统不可移动。《杀手》里拖拽尸体时就用这策略：尸体的手（或脖子或脚）被约束跟随玩家的手。

**摩擦处理**

摩擦还没处理。这意味着不做点什么的话，粒子会像在冰面上一样沿地面滑动。按 Coulomb 摩擦模型，摩擦力取决于接触物体间法向力的大小。为实现它，我们在发生穿透时测量穿透深度 $d_p$（在把穿透点投影出障碍物之前）。把粒子投影到表面后，切向速度 $\mathbf{v}_t$ 按与 $d_p$ 成正比的量削减（比例系数即摩擦常数）。这是通过适当修改 $\mathbf{x}^*$ 完成的。见图 10。要注意切向速度不能反向——那种情况应直接置零，因为这表明穿透点已停止切向移动。可以也应当实现比这更好的摩擦模型。

> **图 10**：带摩擦的碰撞处理（投影与切向速度修正）。

**碰撞检测**

如上呈现的物理仿真里，瓶颈之一在碰撞检测——它在松弛循环内可能被执行多次。不过对各种约束迭代的次数可以不同，仍能得到好结果。

《杀手》里碰撞系统的工作方式：先把被模拟对象包围盒内的所有三角形剔除（cull，用八叉树）。对每个（静态背景）三角形，构造并缓存一个用于对加盖圆柱做快速碰撞查询的结构。这个策略带来可观提速。

为防止高速物体（因时间步过大）穿过障碍，做一个简单测试：设想一条线（或半径恰当的加盖圆柱）从上一帧对象中点的位置延伸到当前帧的位置。若这条线撞到任何东西，就把对象位置设到碰撞点。理论上可能出问题，但实践中运行良好。

对尸体还用了另一个碰撞"作弊"：万一发生罕见情况——快速移动的肢体被放置成加盖圆柱两端各在一堵墙的两侧——就把圆柱投影到墙的与躯干相连的那一侧。

**杂项**

《杀手》里松弛迭代数在 1 到 10 之间，视模拟对象类型而定。虽然这不足以精确求解全局约束系统，但足以让运动显得自然。这个方案的妙处是**误差不会累积或在系统里视觉上持续存在导致物体漂移之类**——某种意义上，投影与 Verlet 方案的组合把复杂计算分摊到了多帧上（其他方案必须借助进一步的稳定化技术，比如 Baumgarte 稳定化）。幸运的是，误差在运动小时最小甚至不存在，运动剧烈时最大——这很好，因为快速或复杂的运动多少会让小误差瞒过人眼。

一种软体也可以实现：用"软"约束——每帧只修复偏差的某个百分比的约束（即若两粒子间棍的静止长度为 100 而实际距离是 60，松弛代码可以先设距离为 80、下帧 90、95、97.5 等等）。

如前所述，我们刻意避免使用重型的数学记号以触达更广背景的读者。这意味着尽管这些方法数学上根基牢固，其来源可能显得有点含糊甚至神奇。不过对数学倾向的读者：我们实际做的事是一种求解**微分包含**（differential inclusions，微分方程的变体）的时间步进方法，用的是一种简单的**内点算法**（见 [Stewart]，其中讨论了类似方法）。满足约束时，我们实际是在把系统状态投影到约束描述的流形上；而这又是通过求解线性方程组完成的。求解约束的线性方程或代码可由约束函数的 Jacobian 导出。本文把松弛讨论为解该系统的隐式方式。虽然没有涉及，有时改变松弛系数、甚至使用超松弛（over-relaxation）是有用的（见 [Press]）。由于松弛求解器有时收敛慢，也可以显式构造方程组用其他方法解（例如稀疏矩阵预条件共轭梯度求解器、利用上一帧结果做预条件——从而利用时间相干性）。

注意 Verlet 积分器有多种变体，如 Leapfrog 积分器与 velocity Verlet 积分器，用它们可能提高精度。

奇异性（除零，通常由粒子重合引起）可以这样处理：把粒子随机微移一下。

作为优化，物体静止下来后应当超时休眠。

想玩玩《杀手：代号 47》里死亡角色的动画系统：打开 Hitman.ini 文件，在底部加两行 "enableconsole 1" 与 "consolecmd ip_debug 1"。把光标指向敌人按 shift+F9 会在其附近引爆一颗小炸弹把他炸飞。按 K 切换自由相机模式（相机用方向键、shift 与 ctrl 控制）。

注意由于所有操作基本都发生在粒子层面，这些算法应当非常适合向量处理（例如 Playstation 2）。

## 8. 结论

本文描述了《杀手》里物理系统的实现。"迭代方法 + 稳定积分器"的组合哲学被证明是成功的、且适合在计算机游戏里实现。最值得注意的是：统一处理碰撞与接触的粒子基框架、以及在不累积视觉可见误差的前提下权衡速度与精度的能力——这些都是强大的特性。当然还有许多细节可以改进，特别是刚体的四面体模型还需要打磨——相关工作正在进行。

在 IO Interactive，我们最近在用完整 Navier-Stokes 方程做交互式水与气体模拟的实验。我们正在研究把与本文类似的技术应用上去，希望产出更快更稳定的水模拟。

## 9. 致谢

作者感谢 Jeroen Wagenaar 的有益讨论，以及 IO Interactive 全体同仁的合作与出色的工作环境。

欢迎反馈与评论。

## 参考文献

- [Baraff] Baraff, David, *Dynamic Simulation of Non-Penetrating Rigid Bodies*, Ph.D. thesis, Dept. of Computer Science, Cornell University, 1992.
- [Mirtich] Mirtich, Brian V., *Impulse-based Dynamic Simulation of Rigid Body Systems*, Ph.D. thesis, University of California at Berkeley, 1996.
- [Press] Press, William H. et al, *Numerical Recipes*, Cambridge University Press, 1993.
- [Stewart] Stewart, D. E., and J. C. Trinkle, "An Implicit Time-Stepping Scheme for Rigid Body Dynamics with Inelastic Collisions and Coulomb Friction", International Journal of Numerical Methods in Engineering.
- [Verlet] Verlet, L., "Computer experiments on classical fluids. I. Thermodynamical properties of Lennard-Jones molecules", Phys. Rev., 159, 98-103 (1967).
- [Witkin] Witkin, Andrew and David Baraff, *Physically Based Modeling: Principles and Practice*, SIGGRAPH '97 course notes, 1997.

---

> [!tip] 译读导航（非原文内容）
> 按导读计划，读完本篇的下一步：[[Muller-2006-PositionBasedDynamics-中文翻译|Müller 2006 PBD 中文译本]]——重点对照其 §3.1 的 17 行仿真循环（本文 TimeStep 的"显式速度版"）与 §3.3 约束投影推导（本文棍约束伪代码的一般化，式 (9)–(11)）。完整路线见 [[PBDReadingGuide]]。
