# fuzz概览

#### 总体思路：网络协议中存在安全漏洞——》灰盒模糊测试（Grey-box Fuzzing）是当前最广泛应用的漏洞发现方法——》传统灰盒模糊测试存在缺陷，难以有效测试网络协议——》目前已有专门针对网络协议的模糊测试工具——》但现有工具仍面临三个主要问题——》解决这三个问题需要应对三个技术挑战——》针对每个挑战提出相应的优化策略。





## 背景

#### **网络通信协议**

定义了互联网上的设备如何相互交互。然而，在软件中实现这些协议的缺陷可能会导致安全漏洞。

#### **灰盒模糊测试**

已成为最流行的漏洞发现方法之一，通过基于变异的技术高效生成测试用例，具有快速检测能力和低误报率的特点。

#### **传统灰盒模糊测试缺陷**

然而，传统的灰盒模糊工具在模糊协议实现程序时面临挑战，因为这些程序不能直接从文件中读取测试用例，并且依赖于网络接口来接收客户端请求。此外，协议实现程序是有状态的，需要在客户端交互期间不断更新内部程序状态。因此，**缺乏网络接口且仅依赖代码覆盖率指标的传统模糊测试工具不适用于测试协议实现程序**。

#### 专门为网络协议设计的灰盒模糊测试工具

近年来，已经出现了几项研究工作，开发**专门为协议实现设计的灰盒模糊测试工具**（AFLNET、SNPSFuzzer、NSFUZZ），它们主要集中在提高交互速度和扩大状态覆盖范围上。



## 问题：

#### **（1）状态价值评估单一**

模糊测试工具未能充分考虑各状态潜在价值的差异性，且状态评估维度过于单一，导致无法准确量化状态的重要性。  

针对aflnet的传统状态选择算法：**RANDOM、 ROUND-ROBIN 和 FAVOR 等算法**。虽然 AFLNET考虑到了状态对协议测试的影响，但是它所提供的几种状态选择算法要么同等对待每个状态，要么使用简单又直观的公式来进行状态选择。具体地，RANDOM算法指的是随机选择状态， 而ROUND-ROBIN算法指的是对所有状态进行轮询选择，因此这两种状态选择算法是同等对待每一个状态并没有考虑到每个状态的潜力是不一样的，而 FAVOR 算法采用简单又直观的公式去评估每一个状态，但是评估状态的维度信息过于单一，导致无法准确评估出每个状态的真实潜力。

另外**AFLNetLegion**是在 AFLNET 的基础上设计了一种新的状态选择算法，即基于蒙特卡罗树搜索 （MCTS）的变体来进行状态选择。具体的做法是将原有的状态机模型展开 为更细粒度的树形状态机模型，便于结合蒙特卡罗树搜索算法进行状态选择。 虽然引入了其他领域中优秀的启发式搜索算法来帮助模糊器选择更有潜力的状态，但是它仍然使用单一的维度信息去评估每个状态。

#### **（2）测试用例生成质量低**

最近的模糊测试工具通常采用消息变异技术来创建新的测试用例。然而，这种方法通常会产生**许多违反协议结构规范的实例**。被测试的服务器无法解析这些实例，导致模糊测试资源的严重浪费。

#### （3） 状态转换覆盖不足

大多数模糊测试工具旨在覆盖单个状态，而不是程序状态转换。因此，这些工具优先考虑简单的测试用例以加快交互速度，**忽略了运行时状态转换中嵌入的交互信息**。鉴于协议实现涉及许多**可能存在错误的状态转换，对这些转换进行全面的安全评估至关重要**。



## 挑战：

#### **挑战1（C1）：引入多维度评估的状态选择算法降低测试器执行效率**

多维度状态选择算法需要综合考虑多个维度的组合，这会导致状态组合数量呈指数级增长，从而大幅增加算法的计算复杂度和存储需求，进而影响测试效率的保障。

#### 挑战2（C2）：随机突变通常无法产生符合协议消息结构的测试用例

协议实现将无法解析变异生成命令，中断后续的协议状态转换，阻碍模糊过程，浪费大量模糊资源。

#### 挑战3（C3）：短视的模糊策略阻碍了对深度程序状态路径的探索。

仅旨在覆盖状态的模糊策略是次优的，因为它们丢弃了复杂的实例，最终使探索深层路径和发现漏洞变得困难。



## 解决思路：

#### C1：通过引入粒子群优化算法的变体来帮助选择更有潜力的状态

通过引入**粒子群优化算法的变体**来帮助协议模糊器选择更有潜力的状态作为测试的目标状态，同时设计了**多种不同的算法触发机制**能够让引入的粒子群优化算法自适应地选择合适的时机进行调用，还设计一种新的消息序列分析器来帮助模糊器去探索更深层次的协议状态和逻辑代码，以此来解决网络协议模糊器在进行状态选择时存在的问题。

#### C2：通过内容感知模块筛选突变样本

参考协议的RFC文档，我们构建了协议状态转换和交互命令的映射，将这些映射用作模糊测试工具的先验知识输入。使用这些输入，我们开发了一个**内容感知模块**。该模块采用**过滤方法筛选突变样本**。在较高水平上，这种方法安全地丢弃了大量无效样本，而不会影响最优解。具体来说，**模糊器通过提供协议命令（事件）来检测潜在的漏洞，从而通过状态转换来驱动协议实现**。

#### C3：通过状态引导变异模块扩展模糊迭代范围

基于先前的模糊器知识设计了一个状态转换字典d。此字典存储将协议服务器从当前状态A转换到目标状态B的消息序列。此外，当前程序将S及其在状态链中的深度D作为模糊器的参数。

然后，我们根据**两种情况调整模糊策略**：（1）如果循环中所选模糊状态的深度小于最小初始化状态长度**，以及**（2）如果状态在状态链中重复出现而没有导致崩溃（例如，相同的状态信息出现在状态链的不同深度）。当模糊迭代遇到这些情况时，SGMFuzz会调用我们的状**态引导变异模块**。该模块根据当前状态S从d读取消息序列，这将目标服务器转换到新的状态S'，形成新的消息序列M'.                                       

**通过触发目标服务器中的状态变化，我们扩展了模糊迭代的探索范围，从而增强了工具测试更深层次程序状态路径的能力。**





## 大纲

在进行模糊测试时，主要面临的问题包括**测试过程效率低和代码覆盖率不足**。优化的目标是**提升模糊测试的执行速度，并提高代码覆盖率**。具体而言，优化过程可以分为两个目标，其中目标之一侧重于**提升测试的效率（执行速度）**，另一个目标则致力于通过最优化方法**提高测试的全面性和深度（代码覆盖率）**。为实现这些优化目标，碰到一些**挑战（例如大量无效测试用例等）**，我们可以通过**设计合理的优化算法**（包括整体算法和小算法相结合）来有效地提升模糊测试的性能和测试覆盖率。









## 有限状态机建模fuzzing过程

#### **Mealy FSM：**

$$
A = (Q, q_0, \Sigma, \Lambda, \delta, \lambda)
$$

|     Parameter      |   Sign   |       Content        |
| :----------------: | :------: | :------------------: |
|     有限状态集     | ***Q***  | *q1, q2, . . . , qn* |
|    系统初始状态    | ***q0*** |          /           |
|    输入（种子）    | ***Σ***  | *σ1, σ2, . . . , σm* |
|  输出（事件结果）  | ***Λ***  | *o1, o2, . . . , on* |
|    状态转换函数    | ***δ***  | ***δ(q, σ) = q ′***  |
|  状态转换的输出集  | ***λ***  |     ***Q×Σ→Λ***      |
| 生成实例（序列集） | ***C***  | *c1, c2, . . . , cn* |





## 优化目标建模

### 优化目标1：最大化代码覆盖路径

$$
\begin{equation}
\arg \max_{M_1, \dots, M_N} \delta(q_0, M_n) \to O_n \quad \forall M_n \in \{M_1, \dots, M_N\}
\end{equation}
$$

$$
\begin{equation}
\text{s.t.} \quad \delta(q_0, M_n) \to O_n \neq \delta(q_0, M'_n) \to O'_n \quad \forall M_n \in C
\end{equation}
$$

​	如公式中所示，在每一轮模糊过程中，**M中的每个元素都代表了一个独特的探索路径**，该路径导致协议实现产生以前未观察到的结果。

​	因此，我们的目标1是**最大化集合M（代码覆盖路径）**。此外，M中的元素是由模糊工具F基于运行时收集的信息而生成的。

### 优化目标2：最小化执行时间

$$
\arg\min T 
\\
\text{s.t.} 
\quad C(M) \geq C_{target}
$$

​	如公式所示，其中**T是生成并执行所有测试用例的总时间**（包括增加M和未增加M的测试用例），**条件是达到或超过规定的代码覆盖率**。

​	因此，我们的目标2是**最小化执行时间**。

### 多目标优化模型

$$
\arg \max {F = \frac{\text{Coverage}} {\text{TestTime}+ \epsilon}}
\\
\begin{aligned} 
s.t.\quad
& \delta(q_0, M_n) \to O_n \neq \delta(q_0, M_{n'}) \to O_{n'} \quad \forall M_n, M_{n'} \in M, n \neq n' \\ 
\quad
& T \leq T_{\text{max}} \quad \text{（总时间约束）}\\ 
& M_n \in \Sigma^* \quad \text{（协议规范约束）}\\ 
& |M_n| \leq L_{\text{max}} \quad \text{（消息序列长度约束）}\\ 
& C(M) \geq C_{target} \quad \text{（代码覆盖率约束）} 
\end{aligned}
$$

![image](https://github.com/ZZT711/fuzz/blob/main/%E5%9B%BE%E7%89%87/1.jpg)



### 综合算法：优化框架

![image](https://github.com/ZZT711/fuzz/blob/main/%E5%9B%BE%E7%89%87/2.jpg)

```python
1: Initialize FSM \(S\), Seed Queue \(Q \leftarrow q_0\), Time \(T \leftarrow 0\), DP Table \(dp\)   
3: while \(T < T_{\text{max}}\) and not abort:  
4:   s ← ChooseState(S)  # 算法2：动态规划选择有价值状态  
5:   M, score ← ChooseQuece(Q, s, dp)   
6:   (M₁, M₂, M₃) ← Split(M)  
7:   energy ← AssignEnergy(M)  
8:   for i in 1 to energy:  
9:     D ← CalculateDepth(S, s)  
10:    if IsStateGuided(D, β):  
11:      M₂' ← StateGuidedMutate(M₂, d, s)  #算法3：状态引导变异算法
12:    else:  
13:      M₂' ← RandMutate(M₂)  
14:    M' ← (M₁, M₂', M₃)  
15:    if ContextAware(M', d, L_max):  # 算法4：内容感知算法  
16:      R' ← SendToServer(P_t, M')  
17:      T += TimeCost(M')  # 更新时间  
18:      if IsCrash(R'):  
19:        Q_c.add(M')  
20:      elif IsInteresting(R', S):  
21:        Q.add(M')  
22:        S ← UpdateFSM(S, R')  
23:      UpdateDPTable(dp, M', score, T)  # 更新动态规划表  
24:    else:  
25:      Q_c.add(M')  
26: return Q, Q_c, S  
```



### 算法2：状态选择函数（ChooseState）

### PSO算法解决   通过两个指标选择最好的测试状态

```python
def ChooseState(S):  
//pso算法

return best_M, max_score  
```

$$
% 评估函数定义
F(S_j) = \frac{\text{Paths}(S_j)}{\text{TotalPaths}} \cdot \frac{1}{\text{AvgTime}(S_j) + \epsilon}
\\
% 粒子适应度计算（公式3.4修改）
\text{FitnessValue}(i) = \sum_{j=0}^{n-1} \left( x(i,S_j) \cdot F(S_j) \right)
$$

### 算法3：状态引导变异函数

![image](https://github.com/ZZT711/fuzz/blob/main/%E5%9B%BE%E7%89%87/6.jpg)

### 算法4：内容感知模块（动态规划约束检查）

```python
1: def ContextAware(M', d, L_max):  
2:   if |M'| > L_max:  
3:     return False  
4:   for cmd in M':  
5:     if cmd not in Σ (RFC规范):  
6:       return False  
7:   # 检查协议语义约束（如FTP命令顺序）  
8:   if not CheckSemantics(M', d):  
9:     return False  
10:  return True  
```

### 

