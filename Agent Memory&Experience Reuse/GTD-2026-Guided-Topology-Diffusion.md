---
title: "GTD：用图扩散与代理奖励逐步生成多智能体通信拓扑"
aliases:
  - GTD
  - Guided Topology Diffusion
  - "Dynamic Generation of Multi LLM Agents Communication Topologies with Graph Diffusion Models"
tags:
  - paper-reading
  - agent/multi-agent
  - agent/graph-optimization
  - generative-model/diffusion
  - optimization/cost-aware
year: 2026
venue: ACL
track: 主会长论文（Volume 1: Long Papers）
status: 精读与代码核对
cssclasses:
  - paper-note
updated: 2026-11-07
---

[[00-Agent-Memory-Experience-Reuse|← 论文索引]] · 对照 [[GDesigner-2025-Task-Conditioned-Communication-Graphs|G-Designer]]、[[GPTSwarm-2024-Optimizable-Agent-Graphs|GPTSwarm]]、[[MaAS-2025-Cost-Aware-Agentic-Supernet|MaAS]]

# GTD：用图扩散与代理奖励逐步生成多智能体通信拓扑

> [!abstract] 一句话说明作者做了什么
> 作者训练一个条件图扩散生成器：给定当前任务和一组预置 agent，先从噪声图开始逐步去噪；每一步都采样若干候选通信图，再由轻量代理奖励模型预测准确率与成本，选出当前更合适的候选，最终得到一张任务专属通信图供多智能体执行。
>
> **输入**：任务描述、agent/角色描述及可用团队。**输出**：有向通信邻接矩阵，随后是按该图执行得到的答案。**被改进对象**：外部图生成器和 agent 间的信息流，不是基础 LLM 权重、长期记忆库或 agent 角色本身。

> [!info] 阅读与核对范围
> 阅读 ACL Anthology 正式版全部19页，包括正文、算法1及附录D–Q；核对官方作者主页、ACL 元数据和作者主页链接的代码仓库。代码静态核对固定快照 22738c3739fd14189193fc8f9e412b2bfe627137，覆盖数据生成、proxy、扩散模型、引导采样和 GSM8K 执行脚本。未运行付费 LLM API、未完整复现训练或论文分数。
>
> **作者名元数据差异**：ACL Anthology HTML 元数据列出的第二、三作者姓名与正式 PDF 首页不同。Anthology 说明 PDF 是权威版本；本文按 PDF 首页和作者项目页使用 Mengting Li、Guancheng Wan，并记录这一差异，不混用两套名字。

## 1. 论文信息与选择理由

- **题目**：Dynamic Generation of Multi LLM Agents Communication Topologies with Graph Diffusion Models。
- **方法名**：Guided Topology Diffusion（GTD，指导式拓扑扩散）。
- **作者（正式 PDF 首页）**：Eric Hanchen Jiang、Mengting Li、Guancheng Wan、Xiao Liang、Sophia Yin、Yuchen Wu、Xinfeng Li、Yizhou Sun、Wei Wang、Kai-Wei Chang、Ying Nian Wu。
- **机构（PDF 首页）**：University of California, Los Angeles（UCLA）、University of Washington、Nanyang Technological University（NTU）。PDF 首页将大部分作者标为 UCLA，Yuchen Wu 标为 University of Washington，Xinfeng Li 标为 NTU。
- **发表身份**：**ACL 2026 主会长论文**，收录于 Proceedings of the 64th Annual Meeting of the Association for Computational Linguistics, Volume 1: Long Papers，页码 38042–38060，2026年7月2–7日。
- **官方来源**：[ACL Anthology 正式页面](https://aclanthology.org/2026.acl-long.1764/) · [正式 PDF（19页）](https://aclanthology.org/2026.acl-long.1764.pdf) · [DOI](https://doi.org/10.18653/v1/2026.acl-long.1764)。
- **预印本**：[arXiv:2510.07799](https://arxiv.org/abs/2510.07799)，2025年10月首次提交；最终发表身份以 ACL 2026 正式版为准。
- **作者项目页**：[Kai-Wei Chang 课题组论文页](https://web.cs.ucla.edu/~kwchang/bibliography/jiang2026dynamic/)。
- **代码**：[作者主页指向的代码仓库](https://github.com/ericjiang18/diffusion_agent) · [本次核对的固定快照](https://github.com/ericjiang18/diffusion_agent/tree/22738c3739fd14189193fc8f9e412b2bfe627137)。仓库 README 将其描述为 GTD 官方实现。
- **为什么选择它**：此前已经精读 G-Designer（ICML 2025），它用任务条件化图网络生成通信图。GTD 是自然且实质的后续：把一次性图预测改为多步图扩散，并在每个去噪步骤中用代理奖励做候选选择。它直接研究“当前任务下谁应该读取谁的输出”，同时把成本和故障鲁棒性纳入评测。
- **去重范围**：检查仓库最新 README、主题索引、模板、G-Designer/GPTSwarm/MaAS/AFlow 相关笔记及当前可访问推送记录；此前没有 GTD 独立精读。不能保证覆盖不可访问的全部历史。

## 2. 出发点：为什么不能给所有任务用同一张图？

多智能体系统中，通信拓扑（communication topology）决定 agent 能看到谁的输出。它不是装饰性的流程图，而是实际的信息访问约束：

- 若 $A_{ij}=1$，表示 agent $i$ 的消息可以传给 agent $j$。
- 若 $A_{ij}=0$，表示这条信息通道不存在。
- 因而图会改变每个 agent 的上下文、可利用证据和可能的错误传播路径。

常见固定结构包括：

- **Chain（链）**：前一个 agent 的输出传给下一个。通信较少，但前面出错可能一路传递。
- **Star（星）**：中心节点与其他节点交换信息。协调简单，但中心可能成为瓶颈或单点故障。
- **Complete（全连接）**：每个 agent 都能读取其他 agent 的消息。信息充分，但 token 和通信开销很高。
- **Layered（分层）**：先并行处理，再由上层汇总或检查。

作者关注的不是“哪一种结构永远最好”，而是**同一结构在不同题目上的边际价值不同**。简单问答可能只需一条短链；软件开发、数学验证等任务可能需要设计、实现、检查和修复之间更丰富的信息流。固定全连接会给简单任务增加冗余；固定链又可能让复杂任务缺少必要的交叉检查。

### 2.1 为什么已有自适应方法还不够？

作者将已有拓扑生成概括为两类限制：

1. **一次性生成（single-shot generation）**：例如单步 VAE 或 GNN，直接从任务条件预测整张图。它能学到训练图分布，但难以在生成过程中反复纠正结构。
2. **生成完再评分（post-hoc reward）**：先得到整张图，再用任务成绩评估。完整运行多智能体很贵，且反馈来得太晚，不容易知道应该改哪部分。

作者因此把图生成视为一个逐步构造问题：先有一张带噪声的图，再逐渐恢复出结构；每一步都可以用便宜的代理模型评估候选图，从而把外部目标注入生成过程。

这不是说扩散模型天然知道“好图”。它需要训练数据学习什么图比较像高质量结构；代理模型也必须足够准确，否则它会把生成过程引向错误方向。

### 2.2 必要前置概念

- **图扩散（graph diffusion）**：训练时逐渐给真实图加噪声；生成时从噪声出发，学习反向去噪，恢复图结构。
- **条件生成（conditional generation）**：生成分布不是无条件的 $p(A)$，而是 $p(A\mid C)$；这里条件 $C$ 与任务和可用团队有关。
- **代理奖励模型（surrogate reward model）**：用较便宜的神经网络预测某张图真正运行后的表现，避免每个候选都调用整套 LLM 团队。
- **零阶优化（zeroth-order optimization, ZO）**：不需要对真实奖励求梯度；通过采样候选、比较分数，选择更好的方向。
- **标量化多目标（scalarization）**：把准确率、成本等多个目标用权重合成一个分数。它方便优化，但权重决定了系统愿意牺牲多少准确率来节省成本。

## 3. 输入、输出和被改进的对象

| 项目 | 具体内容 |
| --- | --- |
| 任务输入 | 当前题目的自然语言描述 $q$ |
| 团队输入 | 预先给定的 agent 集合、角色/能力描述和工具配置 |
| 节点特征 | 角色描述的文本向量；代码中使用 all-MiniLM-L6-v2，384维 |
| 图状态 | 当前扩散步骤中的带噪声邻接矩阵 $A_t$ |
| 生成器输入 | $A_t$、任务条件、节点特征、扩散时间步 $t$ |
| 代理模型输入 | 候选邻接矩阵、节点特征和任务条件 |
| 最终输出 | 二值邻接矩阵 $A_0\in\{0,1\}^{N\times N}$ |
| 任务执行 | 把最终图交给多智能体运行框架，按边决定信息传递，再汇总答案 |

**重要边界：GTD 主要决定通信边，不自动决定角色。** 角色描述在生成前已经给定；论文主实验按领域预设团队，例如数学任务使用四个 MathSolver agent、代码任务使用四个 CodeSolver agent、MMLU 使用三个 KnowledgeableAcademic agent。它不是一个同时搜索“要几个 agent、分别扮演什么角色、调用什么工具”的完整系统设计器。

另外，“dynamic”主要指**不同任务可以生成不同拓扑**。论文明确承认，当前生成的图在单个任务开始前确定，执行期间不会根据新观察动态重连。

## 4. 方法：从带噪声的图逐步走到可执行拓扑

先用一个贯穿例子理解整个过程。

> [!example] 教学例子（假设，不是论文实验）
> 有四个预置 agent：问题分解者、计算者、验证者、答案汇总者。任务是“商店有3箱商品，每箱12件，另外有5件散装，一共多少件？”
>
> - 固定链式结构会让信息依次传递。
> - 全连接结构让四个人都读取其他人的消息，可能产生重复讨论。
> - GTD 的目标是根据这道题和四个 agent 的能力描述，生成一张更合适的通信图。
>
> 下文中的候选分数和边数均为**假设数字**，只用于说明算法；论文没有报告这个具体例子。

### 4.1 第一阶段：先收集“图—任务—结果”数据

作者先选取少量带答案的训练任务，在每道题上实际运行若干基线拓扑。论文/代码主路径包括六种：

1. Fully-connected
2. Chain
3. Star
4. Random graph 1
5. Random graph 2
6. Random graph 3

对每个任务和每张图，真正运行多智能体团队，记录：

- 任务条件（题目向量）；
- agent 节点特征；
- 邻接矩阵；
- 任务效用（utility，例如答案是否正确）；
- 通信成本（cost）。

论文附录报告只用训练集中的50个任务样本构造初始数据；代码的 GSM8K 数据生成入口也将默认上限设为50。若每题测试六种拓扑，理论上最多产生约300条“任务—图”记录（假设每题六种都成功执行）。

这里的关键不是“收集很多聊天记录”，而是先得到一个小型监督数据集，让模型知道不同任务下不同图的大致表现。

### 4.2 第二阶段：训练代理奖励模型 $P_\phi$

完整运行一个多智能体系统很贵，因此作者训练一个图神经网络来近似预测：

$$
P_\phi(A,C)=[\hat u,\hat c]
$$

其中：

- $A$：候选通信图；
- $C$：任务与团队条件；
- $\hat u$：预测任务效用；
- $\hat c$：预测通信成本。

代理模型的结构是：

1. 用两层图注意力网络（GAT）处理节点及边；
2. 对节点表示做 mean pooling，得到整张图的表示；
3. 把图表示与投影后的任务条件拼接；
4. 通过 MLP 输出 utility 和 cost。

训练目标是均方误差：

$$
L_\phi=\frac{1}{M}\sum_{j=1}^{M}
\left\|P_\phi(A_j,C_j)-P_j\right\|_2^2
$$

$P_j$ 是真实多智能体运行得到的表现向量。代理模型不直接生成图，只回答“这张候选图大概有多好、代价多大”。

论文报告的配置：两层 GAT、隐藏维度32、Adam、学习率 $10^{-3}$、10 epochs、batch size 16。

### 4.3 第三阶段：训练条件图扩散生成器 $G_\theta$

生成器要学习：给定任务条件和团队特征，什么样的图结构属于高质量图。

#### 前向加噪

邻接矩阵的原始边取值为0或1。作者先把它缩放到 $[-1,1]$，再逐步加入高斯噪声：

$$
q(A_t\mid A_0)=
\mathcal N\left(
A_t;
\sqrt{\bar\alpha_t}A_0,
(1-\bar\alpha_t)I
\right)
$$

其中：

- $A_0$：原始干净图；
- $A_t$：第 $t$ 步带噪声的图；
- $\beta_t$：每一步的噪声强度；
- $\alpha_t=1-\beta_t$；
- $\bar\alpha_t=\prod_{s=1}^{t}\alpha_s$；
- $I$：单位协方差矩阵。

随着 $t$ 增大，图结构逐渐被噪声淹没。

#### 反向去噪

生成器 $G_\theta(A_t,C,t)$ 接收：

- 当前带噪图；
- 任务/团队条件；
- 节点特征；
- 当前扩散时间步。

它预测原始干净图 $A_0$。论文使用 Graph Transformer，而不是让每条边独立预测：作者希望边的生成能考虑整张图的全局结构依赖，例如层次关系、循环和远距离依赖。

训练时只用高质量图子集 $D_{hq}$，用二元交叉熵让预测邻接矩阵接近真实图：

$$
L_\theta=
\mathbb E_{t,A_0,C,\epsilon}
\left[
\mathrm{BCE}
\left(
G_\theta(A_t,C,t),A_0
\right)
\right]
$$

代码将 utility 大于0.5的样本作为高质量图；在二值正确/错误奖励下，这相当于主要保留成功图。

论文报告的生成器配置：两层 Graph Transformer、2个 attention heads、学习率 $10^{-4}$、50个扩散时间步。

### 4.4 第四阶段：每个去噪步骤都用代理奖励引导

这是 GTD 相比普通条件扩散最关键的部分。

对一个新任务，先采样初始噪声图 $A_T$。然后从 $T$ 到1逐步去噪。每一步：

1. 生成器先预测一张未引导的干净图估计 $\hat A_0^{(t)}$。
2. 根据这个估计采样 $K$ 张二值候选图。
3. 代理奖励模型分别预测每张候选图的 utility 和 cost。
4. 用加权分数选出当前最优候选。
5. 用选中的候选替代原始预测，计算反向扩散后验，再采样下一步 $A_{t-1}$。

论文的选择式为：

$$
A_{0,\mathrm{best}}^{(t)}
=
\arg\max_{A_{0,k}^{(t)}}
\left(
w_u\hat u_k-w_c\hat c_k
\right)
$$

其中 $w_u$、$w_c$ 是效用和成本的权重。注意这不是 PPO/GRPO，也不是对代理模型反向传播；它是在候选图之间做梯度无关的比较。

论文 Algorithm 1 使用 $K=5$；每步生成5张候选图，代理模型打分后选一个。50步扩散意味着每个新任务最多进行约250次廉价 proxy 评分，而不是250次完整 LLM 团队执行。

**假设例子**：若代码采用的权重为 $w_u=1,w_c=0.1$，成本先按边数计：

| 候选图 | 代理预测准确率 $\hat u$ | 成本 $\hat c$（假设边数） | 分数 $\hat u-0.1\hat c$ |
| --- | ---: | ---: | ---: |
| A | 0.96 | 8 | 0.16 |
| B | 0.92 | 4 | 0.52 |
| C | 0.90 | 3 | 0.60 |
| D | 0.86 | 2 | 0.66 |
| E | 0.99 | 9 | 0.09 |

在这组**教学假设**中，D 得分最高，所以当前步骤选择D作为去噪参考。注意这不表示真实准确率就是0.86，也不保证最终图全局最优；它只是代理模型对候选的预测。

### 4.5 为什么要在每一步引导，而不是最后挑一次？

若只在最终生成后评分，前面几十步的结构选择可能已经把采样带到不理想区域。逐步引导的设计意图是：在图逐渐成形的过程中持续把采样轨迹偏向高奖励候选。

这也解释了为什么作者选扩散模型：它提供多次可干预的去噪步骤；代理模型提供廉价的外部目标近似。二者组合后，生成器不只是复现训练集里的常见图形，还能在新任务条件下做局部候选选择。

### 4.6 最终任务执行：图固定后才真正调用团队

扩散完成后得到 $A_0$，阈值化为二值邻接矩阵。运行框架据此设置通信掩码：

- 有边：接收者可读取发送者消息；
- 无边：该消息不进入接收者的可见上下文；
- 所有 agent 按既定团队和原有提示执行；
- 最终汇总者输出答案。

**要区分两个阶段：**

- **拓扑合成**：生成器和代理模型在 GPU 上处理图，不运行完整 LLM 团队来评估每个候选。
- **任务执行**：只把最终图交给真实 agent 团队运行一次。

当前方法不是每轮对话后重建图，也不是执行过程中不断学习新边。论文在 Limitations 中明确将 time-varying topology evolution 留作未来工作。

## 5. 论文的目标函数与理论保证应怎样理解？

论文先把目标抽象为：

$$
\max_A R(A,C)
=
f(\mathrm{Utility}(A,C),\mathrm{Cost}(A,C),\mathrm{Sparsity}(A),\ldots)
$$

这个式子表达了多目标设计动机，不代表实际实现了完整 Pareto 优化。

实际采样时，Algorithm 1 使用：

$$
\hat R(A,C)=w_u\hat u-w_c\hat c
$$

它把多个目标压成一个标量。权重越大，越偏好对应目标。若成本量纲不合适或权重没校准，系统可能过度删边，牺牲准确率。

附录给出一个 proxy 误差上界：若代理模型对所有图都有统一误差 $\epsilon_{max}$，且理想优化器能在所有图中找到代理分数最大者，那么真实奖励差距不超过 $2\epsilon_{max}$。但这个结论依赖两个强假设：

1. proxy 对所有候选图都满足统一误差界；
2. 优化器真正找到全局 proxy 最大图。

GTD 实际使用每步 $K=5$ 个候选和贪心选择，且只沿扩散轨迹探索，并没有枚举全部图。因此这个理论界**不能直接保证有限候选、有限扩散步的实际最终图是全局最优**。

## 6. 实验：它究竟证明了什么？

### 6.1 Benchmark 与协议

| 基准 | 任务类型 | 测试规模（论文表4） | 指标 |
| --- | --- | ---: | --- |
| GSM8K | 数学文字题 | 1,319 | 数值答案准确率 |
| MATH-500 | 数学推理 | 500 | 答案准确率 |
| MultiArith | 多步算术 | 180 | 数值答案准确率 |
| SVAMP | 算术应用题 | 300 | 数值答案准确率 |
| HumanEval | Python 函数生成 | 164 | Pass@1 |
| MMLU | 多学科选择题 | 1,530 | Accuracy |

主实验统一使用 GPT-4o-mini 作为 agent backbone。团队按任务领域配置：数学4个 MathSolver、HumanEval 4个 CodeSolver、MMLU 3个 KnowledgeableAcademic。基线包括 Vanilla、CoT、ComplexCoT、Self-Consistency、LLM-Debate、DyLAN、AgentVerse、GPTSwarm、ADAS、AgentSquare、AFlow、G-Designer、MaAS 等。

### 6.2 主结果

论文表1的关键数字如下（百分比）：

| 方法 | GSM8K | MATH | MultiArith | HumanEval | MMLU | SVAMP | 平均 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| GPTSwarm | 89.14 | 47.88 | 96.79 | 89.32 | 83.98 | 88.67 | 82.96 |
| AFlow | 91.16 | 51.28 | 96.22 | 90.93 | 83.28 | 88.33 | 83.53 |
| G-Designer | 92.09 | 51.00 | 97.78 | 91.11 | 84.50 | 90.00 | 84.41 |
| MaAS | 92.30 | 51.82 | 98.80 | 90.56 | 83.78 | 89.67 | 84.49 |
| **GTD** | **94.14** | **54.07** | **98.88** | **91.46** | **84.58** | **91.33** | **85.74** |

解读时不要只看平均分：

- GSM8K：比 MaAS 高1.84个百分点。
- MATH：比 MaAS 高2.25个百分点，是表1里较明显的差距。
- MultiArith：只比 MaAS 高0.08个百分点，几乎持平。
- HumanEval：比 MaAS 高0.90个百分点。
- MMLU：比 G-Designer 高0.08个百分点，几乎持平。
- SVAMP：比 G-Designer 高1.33个百分点。

论文正文把 SVAMP 写作91.30，表1写91.33；本文保留表格数值并记录差异。整体证据支持“在这些固定团队与预算下，GTD 在多数任务上取得较高准确率”，但不能说每个基准都有显著优势。

### 6.3 成本和速度

论文图5报告 GSM8K 上 GTD 达到约94%准确率，token 消耗约 $4.8\times10^6$；G-Designer 使用约多15%的 token 且准确率更低，LLM-Debate 的 token 消耗超过 GTD 五倍。MultiArith 上 GTD 约98.88%准确率、约 $8.4\times10^4$ token；SVAMP 上超过91%准确率、约 $1.4\times10^5$ token。

作者还报告一次性训练数据生成约 $4.0\times10^5$ token，而在 GSM8K 完整测试集上的推理约 $4.4\times10^6$ token/次；他们据此主张训练成本可被部署摊销。需要注意这依赖部署规模，不能把一次训练免费化。

GSM8K 每题平均推理时间（附录表6）：

| 方法 | 秒/题 | 准确率 |
| --- | ---: | ---: |
| Vanilla CoT | 1.8 | 87.45 |
| LLM-Debate | 11.7 | 89.47 |
| DyLAN | 8.5 | 89.98 |
| G-Designer | 9.2 | 92.09 |
| MaAS | 8.8 | 92.30 |
| GTD | 7.9 | 94.14 |

GTD 包含拓扑生成开销，但仍比多数多 agent baseline 快；它没有快过单 agent CoT。

### 6.4 鲁棒性

作者在 GSM8K 上随机挑选一个非关键 agent，让它输出错误结果，模拟单 agent 故障。GTD 准确率从94.1%降至93.8%，下降0.3个百分点；DyLAN 下降近13个百分点，完全图下降超过2个百分点。

附录表9进一步报告：

| 故障情形 | GTD下降 | MaAS下降 | G-Designer下降 |
| --- | ---: | ---: | ---: |
| 单 agent 故障 | 0.3 | 1.8 | 2.1 |
| 两 agent 故障 | 2.1 | 5.6 | 6.8 |
| 一个 agent 50% 输出噪声 | 1.4 | 3.2 | 3.9 |

这是有价值的结构性证据：代理目标和训练图可能保留冗余路径。但它是受控故障注入，不是对真实 API 故障、恶意 agent、提示注入或任意 agent 失效的全面鲁棒性证明。

### 6.5 消融：代理引导到底重要吗？

表2的关键结果：

| 变体 | GSM8K | HumanEval |
| --- | ---: | ---: |
| GTD完整方法 | 94.14 | 91.43 |
| 去掉引导（w/o Guidance） | 88.42 | 87.19 |
| 随机引导（w/ Random） | 89.65 | 88.32 |

去掉代理引导后，GSM8K下降5.72个百分点；随机引导也明显低于真实 proxy 选择。这支持“代理模型提供的有方向候选选择是关键组件”，而不仅仅是扩散模型本身。

候选数消融（附录表12）：

| 每步候选数 K | GSM8K准确率 | 秒/题 |
| ---: | ---: | ---: |
| 1（无有效ZO比较） | 88.42 | 3.2 |
| 3 | 93.15 | 5.8 |
| 5 | 94.14 | 7.9 |
| 10 | 94.31 | 18.1 |
| 20 | 94.38 | 36.8 |

K=5 已接近性能饱和；K继续增加，时间上升明显。

### 6.6 代理模型自己有多准？

附录表8报告在 $D_{gen}$ 的20% held-out split 上：

| 指标 | Utility | Cost |
| --- | ---: | ---: |
| MSE | 0.0089 | 0.0124 |
| Pearson相关 | 0.91 | 0.94 |
| 5选1 Top-1排序准确率 | 78.4% | 85.2% |
| 5选2 Top-2排序准确率 | 93.6% | 96.1% |

更值得注意的是 OOD topology 测试：Top-1 排序准确率72.8%，Top-2 of 5 为89.3%。这意味着代理模型遇到生成器提出的训练分布外拓扑时，仍有相当比例会把候选排错。论文报告的总体准确率提升，不能消除这一风险。

### 6.7 开源模型泛化

Qwen-3-8B 上，GTD 平均72.8%，G-Designer 72.1%，MaAS 71.8%；GSM8K 91.7、MATH 61.5、MMLU 65.2。LiveCodeBench Pass@1 为30.8%，高于 MaAS 29.3 和基础模型25.4。该实验支持跨 backbone 的一定迁移，但增益仍较小，且没有覆盖长程工具环境。

## 7. 论文局限与证据审计

论文在 Limitations 中承认：

1. 需要预先收集 baseline topology 的 seed dataset，存在启动成本；
2. 任务开始前生成一次拓扑，执行中不支持随状态变化动态重连；
3. 当前推理基准的收益在约4个 agent 后出现递减。

结合实验还应注意：

- **训练图分布有限**：每个任务主要从全连接、链、星和3张随机图收集数据；代理模型未必可靠覆盖所有可能图。
- **任务覆盖有限**：主要是数学、短代码、选择题；没有长期网页操作、开放式报告或跨任务经验复用。
- **故障测试偏窄**：随机让非关键 agent 输出错误，不能代表复杂攻击、共同失效或错误消息传播。
- **成本代理与实际 token 必须区分**：论文叙述强调 token/通信成本，当前代码数据生成路径却用边数作为 proxy cost 标签，不能把二者混成一个量。
- **多目标不是完整 Pareto 搜索**：实际选择是固定权重的 utility-cost 标量化；鲁棒性主要通过独立故障实验展示，没有在代码 proxy 的两个输出中直接学习。
- **held-out 的切分粒度未充分说明**：论文报告20% $D_{gen}$ holdout，但没有清楚说明是否按 task 分组，还是随机按 graph-task pair 切分；这会影响代理泛化结论。

## 8. 公开代码核对：论文描述和实现的差别

核对仓库固定快照 22738c3739fd14189193fc8f9e412b2bfe627137：

- [README](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/README.md)
- [GSM8K 主实验](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/experiments/run_gsm8k.py)
- [训练/测试拆分脚本](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/run_gsm8k.sh)
- [扩散模型](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/GDesigner/gdt/diffusion_model.py)
- [Graph Transformer](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/GDesigner/gdt/graph_transformer.py)
- [代理奖励模型](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/GDesigner/gdt/proxy_reward_model.py)
- [逐步引导](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/GDesigner/gdt/guided_generation.py)
- [框架封装](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/GDesigner/gdt/gtd_framework.py)

静态核对得到几个值得记录的边界：

1. **全局注意力与局部掩码不同**：论文 §4.2 说 Graph Transformer 通过 global self-attention 建模所有节点之间的依赖；代码把当前带噪邻接矩阵阈值化为注意力 mask，只允许沿当前图边注意，并加 self-loop 防止节点没有可见位置。代码行为不是论文文字描述的完全全局注意力。
2. **代码中的 cost 标签是边数**：GSM8K 数据生成函数把 cost 设为邻接矩阵元素之和，即边数；proxy 只预测 utility 与 cost 两个量。它不是直接回归真实 token 消耗。
3. **proxy 没有 robustness 输出**：代码实例化两个 reward components，权重为 utility=1.0、cost=-0.1。鲁棒性在论文中做独立故障测试，不是当前 proxy 的第三个训练目标。
4. **训练图来自少量固定模式**：代码按每个训练任务跑全连接、chain、star 和3张随机图；utility 用答案是否正确的二值信号；扩散训练只保留 utility>0.5 的图。
5. **held-out 复现差异**：论文附录报告 proxy 在 $D_{gen}$ 20% holdout 的表现；当前主脚本的 proxy 训练直接把整个 proxy_data_list 放进 DataLoader，没有显式划出20%验证集。不能据此断言论文数字错误，但当前公开主路径不足以复现表8的切分流程。
6. **团队角色固定**：主脚本把单一 agent 名称重复指定4次（如4个 MathSolver），并未在训练中自动分配新角色。
7. **训练/测试隔离**：当前 GSM8K shell 脚本明确分开 training 和 test 文件，先用训练集生成数据、训练 proxy/diffusion，再用测试集推理；这是好做法。其他数据集仍需逐个检查其实际脚本和划分。

这些差异不自动推翻论文结果，但说明复现时应以“论文方法”和“公开代码当前行为”两条线分别记录。

## 9. 复现可行性

- **代码开放**：是；GTD 的生成器、proxy、引导和 GSM8K 实验脚本均可访问。
- **数据开放**：论文列出 GSM8K、MATH-500、MMLU、HumanEval、MultiArith、SVAMP 的公开数据地址；MultiArith 的表4 license 标为 unspecified。
- **LLM backbone**：主实验 GPT-4o-mini，另有 Qwen-3-8B；运行 LLM agent 需要 API 或本地模型服务。
- **GPU**：论文报告全部实验使用4张 NVIDIA A6000、每张48GB。拓扑生成器与 proxy 的显存表报告5个 agent 为2.8GB，50个3.4GB，100个3.9GB，1000个4.9GB；这是图模块显存，不是完整 LLM 服务显存。
- **成本**：GSM8K 训练数据生成约0.4M token，完整测试集推理约4.4M token/次；论文未提供完整美元账单或每阶段精确耗时。
- **复现难点**：复现图网络本身不重；较贵的是生成图—任务—表现数据，需要真实运行多 agent。更重要的是先解决 proxy 的真实成本标签、训练/验证切分和全局/局部注意力差异。
- **对资源受限的建议**：先固定4个 agent、只做 GSM8K 小子集，收集 chain/star/complete/random 的真实准确率与真实 token；用小 GAT proxy 学 utility 和 token cost，再对比单步 GNN、无引导扩散、proxy-guided diffusion。无需训练或微调 GPT-4o-mini 本身。

## 10. 与研究选题的关系（简要）

| 方向 | 本文关系 | 具体边界 |
| --- | --- | --- |
| Harness / 运行框架优化 | **直接相关** | 自动生成 agent 间通信边，但不修改完整工作流代码、工具或角色定义 |
| 经验记忆与检索 | **未覆盖** | 训练图—任务表现数据不是长期经验库；没有轨迹摘要、记忆检索或在线记忆更新 |
| 单 agent 执行 | 间接 | 主要评估团队拓扑，不优化单 agent 的推理策略 |
| 多 agent 运行图 | **直接相关** | 任务条件化生成有向通信拓扑；但一题一图，任务中固定 |
| RL / 参数蒸馏 | **未覆盖基础模型RL** | 训练 proxy 和扩散图生成器是监督学习；推理时做 ZO 候选选择，不是 GRPO/PPO，也不蒸馏进 LLM |

最值得借鉴的是：把昂贵的多智能体执行反馈压缩为一个可复用的图生成器，再在新任务上用便宜 proxy 做条件化搜索。对您的方向，下一步可加入历史轨迹/经验作为图条件，并让真实执行后的 token、成功、故障信息更新 proxy；但这属于可研究扩展，不是 GTD 已实现的内容。

**一个小规模可执行实验**：固定4个角色、两个任务族（数学与短代码），收集每题多张候选图的真实准确率、真实 token 和故障表现。对照固定链、固定全连接、G-Designer式单步条件图、GTD式扩散+proxy。按任务族划分 train/test，而不是随机拆 graph-task pair。指标包括准确率、实际 token、边数、调用数、proxy 排序相关性、故障后性能。它能验证“任务条件化是否带来真正迁移”，以及“边数代理是否能代表 token 成本”。

## 11. 三个值得思考的研究问题

1. **代理模型的分布外可靠性**：当新任务需要完全不同的协作模式时，如何检测 proxy 排序失准，并决定回退到保守图或重新执行少量真实评估？
2. **从静态拓扑到时变运行图**：任务执行到一半发现前提错误，是否应根据当前轨迹和反馈局部重连，而不是从头生成整张图？
3. **记忆如何参与图生成**：是否可以把历史成功/失败轨迹抽象成“任务条件—角色互补—有效边/消息”的经验子图，作为扩散条件或候选约束，并用跨任务执行反馈更新其效用？

## 12. 阅读路线

建议按下面顺序读，先理解作者为什么设计，再看公式：

1. **§1 Introduction + Figure 1**：静态拓扑、单步生成、post-hoc reward 的问题。
2. **Figure 2 + §3.1**：输入、输出、邻接矩阵语义。
3. **§3.2–3.3**：图扩散与黑盒目标的困难。
4. **§4.1**：proxy 是怎样从图和任务预测 utility/cost 的。
5. **§4.2**：扩散模型如何训练，为什么 Graph Transformer 负责建模边依赖。
6. **§4.3 + Algorithm 1**：逐步引导的完整采样过程。
7. **Table 1、Table 2、Figure 5、Figure 6**：性能、成本、引导消融。
8. **§6.2 + Appendix D**：Qwen-3-8B 泛化与实际配置。
9. **Appendix G、J、K、L、M、N**：理论假设、proxy 排序、故障、生成器对照、候选数与资源。
10. **代码**：先读 run_gsm8k.sh，再看 run_gsm8k.py、guided_generation.py、diffusion_model.py 和 graph_transformer.py，对照论文实现边界。

## 原始来源

- [ACL Anthology 正式页面](https://aclanthology.org/2026.acl-long.1764/) · [ACL 正式 PDF](https://aclanthology.org/2026.acl-long.1764.pdf) · [DOI](https://doi.org/10.18653/v1/2026.acl-long.1764)
- [arXiv:2510.07799](https://arxiv.org/abs/2510.07799)
- [作者项目页](https://web.cs.ucla.edu/~kwchang/bibliography/jiang2026dynamic/)
- [作者代码仓库](https://github.com/ericjiang18/diffusion_agent) · [固定核对快照](https://github.com/ericjiang18/diffusion_agent/tree/22738c3739fd14189193fc8f9e412b2bfe627137)
- 关键代码：[GSM8K 数据与训练/推理](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/experiments/run_gsm8k.py) · [Graph Transformer](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/GDesigner/gdt/graph_transformer.py) · [扩散模型](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/GDesigner/gdt/diffusion_model.py) · [代理模型](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/GDesigner/gdt/proxy_reward_model.py) · [引导采样](https://github.com/ericjiang18/diffusion_agent/blob/22738c3739fd14189193fc8f9e412b2bfe627137/GDesigner/gdt/guided_generation.py)
