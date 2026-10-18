---
title: "MemEvolve：让记忆内容与记忆系统代码共同进化"
aliases:
  - MemEvolve
  - "MemEvolve: Meta-Evolution of Agent Memory Systems"
tags:
  - paper-reading
  - agent/memory
  - agent/harness
  - agent/meta-evolution
year: 2026
venue: ICML
track: 主会
status: 精读整理
cssclasses:
  - paper-note
updated: 2026-10-18
---

[[00-Agent-Memory-Experience-Reuse|← 论文索引]] · [[Agent Workflow Memory|对照 AWM]] · [[AFlow-2025-Workflow-Optimization|对照 AFlow]] · [[GPTSwarm-2024-Optimizable-Agent-Graphs|对照 GPTSwarm]]

# MemEvolve：让记忆内容与记忆系统代码共同进化

> [!abstract] 作者到底做了什么？
> 让智能体在完成任务后积累经验；再由外层系统读取执行日志，诊断记忆模块的问题，生成并评测新的 Python 记忆模块。最终改变的不只是记忆库里的内容，还包括“如何提炼、存储、检索、维护记忆”的程序。
> 基础 LLM 权重保持不变；核心 agent 执行循环也不在代码生成器的修改范围。它不使用策略梯度训练连边。

## 1. 论文信息与选择理由

- **题目**：MemEvolve: Meta-Evolution of Agent Memory Systems。
- **作者（正式会议版本）**：Guibin Zhang、Haotian Ren、Chong Zhan、Junhao Wang、He Zhu、Wangchunshu Zhou、Shuicheng Yan。
- **机构**：National University of Singapore、北京邮电大学、香港中文大学、OPPO、ByteDance。
- **发表**：ICML 2026 主会，PMLR 306:158215–158234；会议日期为 2026-07-06 至 07-11。
- **入口**：[官方会议页](https://proceedings.mlr.press/v306/zhang26fa.html) · [正式全文及附录](https://raw.githubusercontent.com/mlresearch/v306/main/assets/zhang26fa/zhang26fa.pdf) · [官方代码](https://github.com/bingreeky/MemEvolve)。
- **选择理由**：AWM、G-Memory 说明经验怎样存与取；AFlow、GPTSwarm 说明执行结构怎样优化。本篇把“记忆模块的实现”纳入优化对象，直接贴近外部 Harness 与 RSI，且是较新的正式顶会论文。
- **去重范围**：核对当前 README、记忆索引、模板、GPTSwarm 笔记及 Agentic RL 队列；未重复 AFlow、AWM、G-Memory、GPTSwarm 或 RAP。
- **核对范围**：正式全文正文、附录 A/B/C（图示部分以图注与代码补充），公开记忆接口、生成提示、创建器、进化选择器、GAIA 评测和 CerebraFusion 读写路径。未实际重跑。
- **版本**：代码核对日期 2026-10-18，版本 6035d5659d7a092dbfa6a87b1a32a3cee652ba54。早期 arXiv 搜索记录的作者名单与会议页有差别，本笔记采用正式会议版本。

## 2. 出发点：经验在进步，使用经验的方法却没变

普通记忆系统可以积累更多内容，但其程序常常固定：始终保存整条轨迹，始终按相似度取前 k 条，始终全部塞进上下文。

这隐含假设：人工设计的记忆机制适合后续所有任务。实际上，完整轨迹可能太长，摘要可能漏掉工具参数，语义最相似的案例也未必能帮助当前步骤。

| 层次 | 改变什么 | 例子 |
| --- | --- | --- |
| 经验进化 | 记忆内容 M | 新增“查询版本历史要处理分页”的经验 |
| 架构进化 | 管理记忆的程序 Ω | 从整轨迹检索改为分阶段提取工具技巧与检查清单 |

这里的架构是软件逻辑与数据组织，不是 Transformer 层数。

> [!example] 贯穿例子：原文案例与教学延伸分开
> Figure 5 要求定位某获奖游戏的 Wikipedia 页面，计算指定发行月份之前的修订次数。原文展示记忆建议：定位规范页面、使用 MediaWiki 修订接口、按日期截断。
> 以下“旧系统为何失败、外层怎样修改”的过程是依据方法构造的教学延伸，不声称是这道题真实发生过的进化记录，也不给出未经核验的修订次数。

若旧记忆只保存整篇检索轨迹，新任务可能因同样出现“游戏”就召回大量无关内容。真正可复用的是：定位唯一页面、调用历史接口、处理分页、统一时间边界、保留核验依据。

可以只写入这条经验；也可以修改编码器和检索器，让以后类似经验都更容易被提炼和召回。本篇进一步做后者。

## 3. 四个模块：给代码搜索划定边界

论文将记忆系统写成：

$$
\Omega=(E,U,R,G)
$$

| 组件 | 职责 | 贯穿例子 |
| --- | --- | --- |
| 编码 Encode，E | 将轨迹、工具输出与反馈变成经验单位 | 提炼日期截断与分页规则 |
| 存储 Store，U | 写入持久存储并组织索引 | JSON 条目、向量或图节点 |
| 检索 Retrieve，R | 根据任务和当前状态提供记忆 | 已定位页面时提供修订接口指导 |
| 管理 Manage，G | 合并、清理、遗忘和维护 | 合并重复规则，清理低效关联 |

这是**功能分解**。公开基类并没有要求四个同名方法，而是暴露 initialize()、provide_memory(request)、take_in_memory(trajectory_data)。编码、存储、管理可在这些方法内部完成。不要把功能框误当 API 签名。

作者将既有方法重新实现到统一接口下，称为 **EvolveLab**；它既是比较平台，也是生成模块的接入规范。论文称十二种代表性系统，当前 README 列十一种 baseline 与两种 evolved system；公开文件覆盖与论文计数不完全相同，不应假设每项一键可复现。

## 4. 内层：固定记忆程序，完成一串任务

1. **开始**：将问题、状态等装入 MemoryRequest。
2. **检索**：provide_memory 返回文本指导，或可选的工具记忆。
3. **执行**：原 agent 继续规划、搜索、读取文件；执行中也可按当前上下文请求记忆。
4. **评测**：判断最终答案，记录正确性、token 和耗时。
5. **吸收**：将问题、交互过程、结果及元数据传给 take_in_memory。
6. **复用**：本次内容进入持久记忆，供后续任务使用。

可简化表达为：

$$
M_{t+1}=U(M_t,E(\tau_t))
$$

M 是记忆内容，τ 是任务轨迹，E 提炼经验，U 写入；管理还可能周期性运行。**这一内层任务流中，Ω 固定，M 变化。**

论文在候选内层评估开始时从空记忆初始化，减少“某候选继承了更多历史”的混淆。空记忆不等于没有模型先验或人工初始程序，复现还需检查种子内容和存储清理开关。

## 5. 外层：怎样写出新的记忆系统？

### 5.1 用轨迹诊断，而不只看总分

外层读取候选成绩、成本、延迟，并回放具体执行轨迹，查找召回不相关、抽象漏信息、重复条目堆积等问题，生成结构化诊断。

总分告诉系统“好不好”；轨迹帮助判断“为什么不好、可能改哪里”。诊断由模型产生，不保证每项归因都正确，所以必须通过后续运行验证。

### 5.2 生成真实 Python 模块

设计阶段读取父模块源码和诊断，生成 provider、配置、注册信息。对贯穿例子的**假设改进**可能是：

- 编码时分别提取工具技巧、检查清单。
- 检索时判断当前是规划还是执行阶段。
- 召回后增加适用性过滤。
- 写入时去重，周期性合并相近经验。

这不是固定配方，候选可以采用不同表示和检索机制。

公开 generation_prompt.yaml 明确限制：**核心 agent loop 不可修改；记忆内部的表示、检索、更新和索引允许修改。** MemorySystemCreator 将生成代码写入文件并更新注册/配置。MemoryEvolver 协调 Analyze → Generate → Create → Validate。

因此这里的“改代码”确实是改编程语言源码。生成可导入模块只是基础验证，效果还需跑任务。

### 5.3 让候选实际运行，再选择

默认实验为三轮外层进化，每轮保留一个父系统并生成三个后代。主文称每候选内层评测 60 条轨迹，其中 40 道新题、20 道上轮复用题。

附录同时按“每轮 40 道、三轮 120 道”描述进化题规模。可能分别指新增题与含复用的总评测数，但第一轮复用来源及脚本对应仍需核对日志；不能把两个数字默默当成同一口径。

候选评价向量：

$$
F_j=(\mathrm{Perf}_j,-\mathrm{Cost}_j,-\mathrm{Delay}_j)
$$

准确率越高越好，成本和延迟越低越好。论文先按帕累托非支配层级排序，同层再按性能。

**帕累托直觉**：甲更准且成本、延迟都不更差时，乙没有优势；甲更准但更慢时，两者存在取舍。

这里是评测驱动的程序进化搜索，不是 REINFORCE/PPO；双层、一阶/二阶进化描述改进对象的层级，不表示计算二阶导数。

### 5.4 留下什么，哪里算递归？

保留好的记忆程序及其诊断、评测记录，再据此生成下一代；每个候选内部又积累自己的记忆内容。

它把“从经验学习的机制”也变成可改进对象，但没有证明诊断器、生成器和选择器自身在无限递归重写。也没有优化多 agent 的角色分配或通信拓扑。

## 6. 具体产物：CerebraFusion 的经验图如何工作？

> [!info] 代码事实与统一方法分开
> 本节核对 cerebra_fusion_memory_provider.py。它是一个公开产物的实际行为，不是所有 MemEvolve 候选必须遵守的算法，也不能直接当作每行论文主结果的精确配置。

### 6.1 写入与筛选

代码优先检查 is_correct 等成功标签。失败任务会更新已有记忆使用成败统计，但跳过新内容吸收；成功任务才进行摘要和可选工具提炼。

文本节点包括成功关键点 success、一般模式 pattern、操作指南 playbook、检查清单 checklist。单次约最多十个节点，内容签名去重。工具记忆开启时还可提炼参数化 Python 函数，保存与注册供以后调用。

### 6.2 图的节点、边与跨轨迹关系

节点是经验文本，不是运行中的 agent；元数据包含来源任务等。

实际构建的关系包括：

- SAME_TASK：同任务条目之间，以一个锚点连接其他条目。
- SIMILAR_CONCEPT：同类型经验达到语义相似度阈值后建立双向边，可以跨任务连接。

枚举还有依赖、共现类型，但定义类型不等于各流程确实建立并使用它们。

### 6.3 新任务的完整检索流程

文本路径为：

1. 分析任务与当前上下文，产生检索重点。
2. TF-IDF 与语义向量混合搜索。
3. 沿命中节点的 SIMILAR_CONCEPT 边扩展邻居。
4. 用当前阶段的阈值筛选，再取 top-k。
5. 把相关内容组织为阶段适配的指导。
6. 以文本记忆返回给 agent。

**明确边界**：所核对的图扩展只使用语义相似边，不沿 SAME_TASK 边扩展。因此保存同任务关系不能等同于检索使用该关系。

工具路径为：检索工具候选 → 路由选择 → 包装成可调用工具 → 返回 API 类型记忆。这与返回工具使用的文字建议不同。

### 6.4 更新与维护

记录被使用的节点与边，任务完成后更新成败统计；周期性合并相似节点、裁剪低效边、调整权重，随后重建索引并持久化。

这里的权重调整属于记忆维护规则，不自动等于策略梯度 RL。跨任务经验由持久库共享；长短期组织取决于候选，论文没有强制所有产物采用统一三层或四层记忆。

## 7. Benchmark、评测协议与反馈来源

| 基准 | 任务形式 | 本文规模与用途 | 关键边界 |
| --- | --- | --- | --- |
| TaskCraft | 合成工具使用与任务执行问题 | 300 题工作集，120 道新增题参与三轮进化 | 进化分数不是独立测试 |
| WebWalkerQA | 多步网页信息查找并回答 | 原集 680 题，抽 170 题；架构在 TaskCraft 上进化 | 清晰的跨数据集架构迁移证据 |
| xBench-DeepSearch | 规划、搜索、工具和推理问答 | 100 题；架构在 TaskCraft 上进化 | 区分模型与尝试预算 |
| GAIA | 搜索、文件、多模态与推理综合任务 | 165 题，53/86/26 道 Level 1/2/3 | 附录称 Level 1 与 67 道 TaskCraft 用于进化，不能把总体分数全视为未见测试 |

主要执行框架为 SmolAgent（文中轻量双 agent 配置）和 Flash-Searcher（单 agent 深度搜索）。还将记忆迁移到 CK-Pro、OWL。主干与进化模型为 GPT-5-mini，跨模型测 Kimi K2、DeepSeek V3.2，附录还测 Claude-3.7-Sonnet。

主表报告 pass@1–3。多次尝试成功口径不能当作一次成功率；本笔记主要比较同模型、同框架的 pass@1。

### 答案如何判正确？

EvolveLab 支持精确匹配与 LLM 裁判。公开 GAIA 脚本调用 judge_equivalence(问题, 标准答案, agent答案)，将判断结果写入 is_correct，再交给记忆吸收。

这有标准答案支持的执行后反馈，不是完全无监督。它不等于执行模型解题前看到答案，但反馈会影响后续记忆。固定迁移的是**架构**；在线评测时内容仍可能变化，不能说整个记忆库被冻结。

## 8. 结果、消融与局限

### 8.1 同框架同模型的单次表现

Table 2/3，Flash-Searcher + GPT-5-mini：

| 基准 | 无记忆 | MemEvolve | 差值 |
| --- | ---: | ---: | ---: |
| WebWalkerQA | 71.18% | 74.71% | +3.53 个百分点 |
| xBench-DS | 69.00% | 74.00% | +5.00 个百分点 |
| GAIA | 69.09% | 73.33% | +4.24 个百分点 |

WebWalkerQA/xBench 更适合支撑跨数据集架构迁移；GAIA 需附带进化题重叠限制。

摘要最高 17.06% 对应 Kimi K2 + Flash-Searcher 的 WebWalkerQA：52.35→69.41，差 **17.06 个百分点**，不是所有模型平均提升，也不是相对比例。

### 8.2 加记忆并非总有益

GAIA 无记忆 69.09，统一实现的 AWM 67.27、ExpeL 66.06，而 MemEvolve 73.33。这支持固定记忆机制可能不适合新框架/任务；不能据此否定 AWM 原论文在网页任务中的表现。

### 8.3 模块消融与搜索对照

Table 4 从 R0=61.67 出发，第三轮最优：

| 进化范围 | TaskCraft 得分 |
| --- | ---: |
| 仅编码 | 68.33 |
| 仅检索 | 71.67 |
| 仅管理 | 66.67 |
| 仅存储 | 65.00 |
| 四模块 | 78.33 |

检索单项收益最大，联合搜索最好，支持跨模块配合的价值。但这些是有限进化题上搜索后的最优结果，不是普适结构的因果证明。

Table 8 的 R3 OOD WebWalkerQA：随机搜索 65.88、贪心替换 70.59、MemEvolve 76.27。贪心每轮枚举八种单模块替换，MemEvolve 生成三个整体后代，搜索预算与空间不完全一致。

### 8.4 重复实验与口径冲突

附录 Table 5 三次运行均值±标准差为 GAIA 73.94±0.86、WebWalkerQA 76.27±1.47、xBench 73.67±0.47；与主表单组 73.33/74.71/74.00 应分别引用。

正文称 DILU 提高 xBench，但 Table 3 为 69→68；本笔记以表格为准。GAIA 的具体进化设置也与“所有系统仅在 TaskCraft 进化”的笼统描述有张力，采用附录具体协议，避免过度宣称泛化。

## 9. 成本与复现

### 9.1 费用相近不代表更快

Table 3 的历史每题费用与延迟：

| 基准 | 无记忆费用→MemEvolve | 无记忆延迟→MemEvolve |
| --- | --- | --- |
| GAIA | $0.086→$0.085 | 505.46→693.33 秒 |
| xBench | $0.141→$0.136 | 523.05→773.06 秒 |
| WebWalkerQA | $0.048→$0.040 | 251.57→332.49 秒 |

API 费用接近或降低，但延迟增加。这是推理阶段的表，不能代表架构搜索总账单。完整进化总成本、运行主机/GPU 配置在本次核对范围未找到明确报告，标为未知；以上也不是当前报价。

### 9.2 实现入口及论文差异

相对路径基于 [核对版本](https://github.com/bingreeky/MemEvolve/tree/6035d5659d7a092dbfa6a87b1a32a3cee652ba54/Flash-Searcher-main)：

| 路径 | 看什么 |
| --- | --- |
| EvolveLab/base_memory.py | 实际接入接口 |
| MemEvolve/core/memory_evolver.py | 分析、生成、创建、验证 |
| MemEvolve/prompts/generation_prompt.yaml | 允许修改的范围 |
| MemEvolve/phases/memory_creator.py | 写 Python 文件与注册 |
| MemEvolve/core/auto_evolver.py | 多轮评测和选择 |
| EvolveLab/providers/cerebra_fusion_memory_provider.py | 图式记忆与工具读写 |
| run_flash_searcher_mm_gaia.py | 裁判与轨迹吸收时机 |

论文说帕累托排序、同层按性能；代码 AutoEvolver 默认 use_pareto_selection=False，普通路径按准确率、再按 token 排序。开启帕累托时，同层按 0.6×准确率 + 0.25×归一化 token 得分 + 0.15×归一化时间得分排序。复现需要记录配置，不能假设与论文逐字一致。

生成提示要求候选复杂度不低于模板，且引入较大改进，说明搜索受人为提示与空间约束，并非完全自由寻找最简单算法。

代码公开，README 有 Python 3.10、依赖、API 和数据入口；GAIA、WebWalkerQA、xBench 数据来源明确。全部 TaskCraft/多框架实验与当前公开入口的对应不能仅据 README 一概确认。

不训练基础模型，主要花费是候选执行与模型/搜索 API；向量模型可本地运行。实现难点包括接口兼容、候选存储隔离、生成代码评测、任务顺序、模型版本和网页动态变化。本次未实际重跑。

## 10. 创新、设计逻辑与选题关系

有依据的设计逻辑重建：任务所需经验不同 → 固定记忆不总有效 → 拆成可替换功能 → 用执行轨迹诊断 → 生成源码候选 → 依据效果与代价选择。这是公开设计的重建，不是作者私人思维过程。

- 相比 AWM/G-Memory：进一步搜索记忆机制的实现。
- 相比 AFlow：搜索对象收窄到记忆 provider，核心执行循环固定。
- 相比 GPTSwarm：不训练连边概率；产物里的经验图不是运行通信图。

| 研究方向 | 对应 |
| --- | --- |
| 经验优化 Harness | 直接相关：优化记忆层代码 |
| 长短期记忆与检索 | 直接相关，但具体组织依候选变化 |
| 单 agent 执行 | 直接相关：Flash-Searcher |
| 多 agent 协作 | 可接入，未优化角色/运行拓扑 |
| RL/参数蒸馏 | 未覆盖基础模型训练；代码进化不是策略梯度 |
| RSI | 学习机制可改进，元优化器自身未被证明递归自改 |

**小实验提议，非论文结论**：固定模型与执行 agent，比较无记忆、固定记忆只增内容、仅优化检索、四模块进化四组。在独立题集冻结架构，分别允许/禁止在线内容更新；记录正确率、token、延迟与搜索总成本。验证收益来自更好的机制还是更多内容和调用。

## 11. 阅读路线与三个问题

先读 Figure 1；再读 §3.2 与附录 A.1，区分四功能和接口；接着读 §4 两层循环；随后读 Table 3/4、附录 B.1 的划分；最后看 CerebraFusion 的 provide_memory/take_in_memory 与外层生成提示。

1. 怎样让架构搜索利用跨任务经验，而不过拟合少量进化题？
2. 记忆图与多 agent 执行图应该分开优化还是联合优化？
3. 在相同总预算下，进化代码是否优于固定代码加更多记忆？哪些改进能蒸馏进小模型？

## 原始来源

1. [ICML 2026 正式记录](https://proceedings.mlr.press/v306/zhang26fa.html)。
2. [全文与附录](https://raw.githubusercontent.com/mlresearch/v306/main/assets/zhang26fa/zhang26fa.pdf)：§3–4 方法，§5 与 Tables 2–8 实验，附录 A/B 协议。
3. [官方代码](https://github.com/bingreeky/MemEvolve)。
4. [生成提示](https://github.com/bingreeky/MemEvolve/blob/6035d5659d7a092dbfa6a87b1a32a3cee652ba54/Flash-Searcher-main/MemEvolve/prompts/generation_prompt.yaml)。
5. [候选选择](https://github.com/bingreeky/MemEvolve/blob/6035d5659d7a092dbfa6a87b1a32a3cee652ba54/Flash-Searcher-main/MemEvolve/core/auto_evolver.py)。
6. [CerebraFusion](https://github.com/bingreeky/MemEvolve/blob/6035d5659d7a092dbfa6a87b1a32a3cee652ba54/Flash-Searcher-main/EvolveLab/providers/cerebra_fusion_memory_provider.py)。
7. [GAIA 脚本](https://github.com/bingreeky/MemEvolve/blob/6035d5659d7a092dbfa6a87b1a32a3cee652ba54/Flash-Searcher-main/run_flash_searcher_mm_gaia.py)。
