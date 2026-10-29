---
title: "Agentic RL · 论文地图与阅读队列"
aliases:
  - Agentic RL 主页
  - 智能体强化学习论文索引
tags:
  - paper-reading/index
  - agentic-rl
cssclasses:
  - paper-note
  - paper-index
updated: 2026-10-29
---

# Agentic RL：论文地图与阅读队列

[[README|← 仓库主页]] · [[00-Agent-Memory-Experience-Reuse|对照记忆与经验复用]]

> [!abstract] 阅读目标
> 理解语言模型智能体如何规划、调用工具、处理长程反馈，以及如何通过强化学习改善策略。先分清推理时搜索、外部记忆更新、外部策略训练与 LLM 参数训练，再比较论文。
> 已完成 6/19 篇：RAP、LATS、VOYAGER、Planner-R1、ToolRL、Memory-R1。原队列恢复位置：ARTIST。临时优先队列：AgeMem（完成）→Memory-R1（完成）→MemSkill→MemRL→LightMem；下一篇MemSkill。仅在完成实质精读并同步后标记完成。

## 临时优先队列（2026-10-26插队）

用户要求先逐篇完成本组，再恢复原19篇；每次一篇。原队列6/19，恢复位置ARTIST。优先队列完成2/5：AgeMem、Memory-R1；原队列第10项同步标记完成。下一篇MemSkill。

| 顺序 | 论文/定位编号 | 状态 |
| --- | --- | --- |
| 1 | [[AgeMem-2026-Unified-LTM-STM-Policy-RL\|AgeMem]]，2601.01885 | 完成精读；ACL 2026主会长论文 |
| 2 | [[Memory-R1-2026-RL-Memory-Management-Utilization\|Memory-R1]]，2508.19828 | 完成精读；ACL 2026主会长论文；原队列第10项同步完成 |
| 3 | MemSkill，2602.02474 | 下一篇，题名/身份待核验 |
| 4 | MemRL，2601.03192 | 待读，题名/身份待核验 |
| 5 | LightMem，2510.18866 | 待读，题名/身份待核验 |

[AgeMem完整笔记](AgeMem-2026-Unified-LTM-STM-Policy-RL.md)。四篇新增项不计入原19篇分母；Memory-R1提前读完后不重复推荐。论文内容已完成与远程同步状态分开记录，不以本地完成冒充推送成功。

### AgeMem补充的理解

- **出发点**：先看信息、重置短期窗口、经历干扰、再揭示任务，使早期存储与后期使用接受同一目标监督。
- **多轮粒度**：与ToolRL单步历史条件化不同，AgeMem训练中真实执行记忆/上下文工具，下一轮输入随之改变。
- **信用分配**：完整轨迹末端优势广播，不是每步独立奖励或因果贡献；默认grpo分组与multi_step_grpo实现不同，详见笔记。
- **记忆与迁移**：公开默认库每rollout清空，跨阶段保留不等于跨无限任务积累；策略参数迁移与具体事实迁移要区分。
- **成本**：平均prompt缩短3.1%/5.1%不等于总token费用下降，工具调用和外部Qwen-Max成本另计。

### Memory-R1补充的理解

- **维护与利用分开**：管理者修改外部库，回答者从60条候选选证据；两个角色固定对方训练，不是联合MARL。
- **奖励与loss**：最终EM评分，不是相对操作前的增量；论文PPO有裁剪，GRPO式无裁剪。尚无专用源码，token mask、KL估计和长度归一化未知。
- **教学与评测**：Memory Distillation不是参数蒸馏；LongMemEval用oracle。GRPO优于同架构SFT，PPO不全面优于SFT；正式PDF和HTML列名、24/50轮与切分比例冲突保留。
- **与AgeMem**：本文分工学习写库与读库；AgeMem统一LTM/STM六工具，并制造上下文重置压力。两者不都等同跨任务事实永久积累。

## 已完成论文

| 笔记 | 发表 | 类别 | 核心机制 | 改变什么 |
| --- | --- | --- | --- | --- |
| [[RAP-2023-Reasoning-via-Planning\|RAP]] | EMNLP 2023 主会 | 规划、世界模型、推理时搜索 | 冻结 LLM 提出动作并预测状态，MCTS 利用过程奖励选择分支 | 当前题搜索树的价值与访问记录；LLM 权重不变 |
| [[LATS-2024-Language-Agent-Tree-Search\|LATS]] | ICML 2024 主会 | 规划、环境交互、搜索、反思 | 冻结 LLM 提出候选、评价观察、总结失败；MCTS 复用前缀 | 本题搜索统计与反思上下文；非 LLM 参数训练 |
| [[VOYAGER-2024-Executable-Skill-Lifelong-Agent\|VOYAGER]] | TMLR 2024 期刊 | 开放探索、程序性记忆、可执行技能 | 自动课程、程序执行修订与成功技能检索组合 | 外部代码/描述索引与任务记录；无模型参数训练 |
| [[Planner-R1-2025-Reward-Shaping-Agentic-RL\|Planner-R1]] | arXiv 2025 预印本 | LLM参数RL、规划、工具、奖励塑形 | 终局Schema门控、约束部分得分、GRPO | LLM策略权重；无长期技能库，配置/正文聚合差异保留 |
| [[ToolRL-2025-Structured-Tool-Reward-GRPO\|ToolRL]] | NeurIPS 2025 主会 | 工具调用、结构奖励、LLM参数RL | 工具名/参数key/值分解奖励与GRPO，单步历史条件化训练 | LLM权重；保留默认源码奖励KL/熵、匹配及评测差异 |
| [[Memory-R1-2026-RL-Memory-Management-Utilization\|Memory-R1]] | ACL 2026主会长论文 | 外部记忆维护、证据筛选、PPO/GRPO | 冻结对方分别训练两角色，以最终答案EM监督；Memory Distillation是生成时选证据 | 两种LLM策略参数与部署外部库；无专用源码，保留数据/目标/结果冲突 |

[在 GitHub 阅读 RAP](RAP-2023-Reasoning-via-Planning.md) · [LATS 精读](LATS-2024-Language-Agent-Tree-Search.md) · [VOYAGER 精读](VOYAGER-2024-Executable-Skill-Lifelong-Agent.md) · [Planner-R1 精读](Planner-R1-2025-Reward-Shaping-Agentic-RL.md) · [ToolRL 精读](ToolRL-2025-Structured-Tool-Reward-GRPO.md)

## 队列外补充阅读

本栏记录临时指定论文，不计入19篇队列，不改变定时任务或下一篇。固定队列现为6/19，优先队列先行；原队列恢复位置ARTIST。

| 笔记 | 发表状态 | 主题与核心机制 | 核验状态 |
| --- | --- | --- | --- |
| [[CB-PbRL-2026-Constrained-Bilevel-Preference-RL\|CB-PbRL]] | NeurIPS 2026主会已录用，官方Poster条目 | 上层多维偏好与违规分类学习奖励/成本；下层熵正则RL对偶、最优λ响应与Clarke近似 | 已完成上传稿精读；2026-10-22按用户提供的教学重写版替换正文，检查Markdown、公式与流程图；录用终稿未逐页比对、源码未取得 |

[在GitHub阅读CB-PbRL笔记](CB-PbRL-2026-Constrained-Bilevel-Preference-RL.md)

### CB-PbRL / CB-RLHF 补充的学习维度

- 该文进行奖励/成本模型与连续控制策略学习，和RAP/LATS的推理时搜索不同；没有LLM多工具智能体实证。
- 上层F明确依赖最优λ和策略诱导轨迹分布，不能将普通交替拟合等同完整超梯度。
- 原文表1的累计回报为真实奖励减真实成本；四环境体现风险/表现折中，Walker末期违规仍达87%，不能说保证实际零违规。
- 版本分开：NeurIPS 2026录用条目称CB-PbRL；本次用户上传稿为ICLR匿名投稿版CB-RLHF。精读依据写明，不冒充终稿已比对。
- 固定队列现为6/19、恢复位置ARTIST；队列外补充现有1篇已完成上传稿全文精读。

## 按图片顺序的 19 篇队列

顺序为从左到右、每列从上到下。图片标题写 17 篇，实际方框共 19 个，以全部条目为准。简称需在各次阅读时通过原文唯一定位；待阅读项的年份、会议与身份尚未在本索引核验。图片箭头不作为论文继承关系证据。

| 顺序 | 图片分组 | 论文简称 | 状态 |
| --- | --- | --- | --- |
| 1 | Planning | [[RAP-2023-Reasoning-via-Planning\|RAP]] | 完成精读 |
| 2 | Planning | [[LATS-2024-Language-Agent-Tree-Search\|LATS]] | 完成精读 |
| 3 | Planning | [[VOYAGER-2024-Executable-Skill-Lifelong-Agent\|VOYAGER]] | 完成精读 |
| 4 | Planning | [[Planner-R1-2025-Reward-Shaping-Agentic-RL\|Planner-R1]] | 完成精读 |
| 5 | Tool Use | [[ToolRL-2025-Structured-Tool-Reward-GRPO\|ToolRL]] | 完成精读 |
| 6 | Tool Use | ARTIST | 待读，优先队列结束后恢复 |
| 7 | Tool Use | ToRL | 待读 |
| 8 | Tool Use | GiGPO | 待读 |
| 9 | Tool Use | ARPO | 待读 |
| 10 | Memory | [[Memory-R1-2026-RL-Memory-Management-Utilization\|Memory-R1]] | 完成精读（临时优先队列提前完成，不重复推送） |
| 11 | Memory | MemAgent | 待读 |
| 12 | Memory | ReSum（图中标注 Context Folding） | 待读 |
| 13 | Self-Improvement | Reflexion | 待读 |
| 14 | Self-Improvement | Absolute Zero | 待读 |
| 15 | Self-Improvement | SiriuS | 待读 |
| 16 | Reasoning | DeepSeek-R1 | 待读 |
| 17 | Reasoning | DAPO | 待读 |
| 18 | Reasoning | GSPO | 待读 |
| 19 | Reasoning | Satori | 待读 |

先读完队列，再主动调研新论文。队列包含搜索、反思、记忆与一般推理 RL；保留用户选项，每次解释实际学习对象，不能一概称为参数强化学习。

## 先分清五种改变

| 机制 | 本仓库例子 | 反馈进入哪里 | 跨任务留下什么 |
| --- | --- | --- | --- |
| 本题规划搜索 | [[RAP-2023-Reasoning-via-Planning\|RAP]] | 搜索树分数与访问记录 | 主方法不建立自动长期经验库 |
| 工作流程序搜索 | [[AFlow-2025-Workflow-Optimization\|AFlow]] | 候选程序与验证表现 | 可复用的工作流 |
| 外部连边策略 RL | [[GPTSwarm-2024-Optimizable-Agent-Graphs\|GPTSwarm]] | 连边分布的可训练参数 | 外部图分布及节点示例 |
| 跨任务经验记忆 | [[VOYAGER-2024-Executable-Skill-Lifelong-Agent\|VOYAGER]]（可执行程序）、[[G-Memory-2025-Hierarchical-Memory\|G-Memory]]、[[AWM-2025-Workflow-Memory\|AWM]] | 经验内容、组织或示范 | 后续任务可用记忆 |
| LLM 参数 RL / 蒸馏 | [[Planner-R1-2025-Reward-Shaping-Agentic-RL\|Planner-R1]]（GRPO，无蒸馏）、[[ToolRL-2025-Structured-Tool-Reward-GRPO\|ToolRL]] | 规则回报→组内优势→token概率更新 | 模型权重，需单独验证泛化 |

RAP 中树节点表示任务状态，AFlow 的搜索节点表示工作流候选，GPTSwarm 操作节点执行 LLM/工具调用；这些图的层级与语义不同。

## RAP → LATS：从预测状态到观察环境

- **RAP**：LLM 充当世界模型预测后续状态，以过程奖励搜索当前题路径。
- **LATS**：在可回退环境中执行候选动作，观察实际结果，并把失败反思放回本题后续尝试；纯推理任务也保留 CoT 版本。
- 两者均不训练基础模型权重，搜索树也不是多智能体通信图。LATS 的“记忆”主要在同题跨尝试复用，不能自动视为跨任务长期经验库。
- **评测边界**：LATS 的 HotpotQA 采用答案正误 oracle；公开编程实现搜索途中调用正式测试，和论文“最后才评正式测试”的描述有差异。WebShop Score 75.9 / SR 38%，不能只凭平均分断言完全成功率优于所有微调方法。
- **复现边界**：论文与当前代码在测试数、深度、迭代数、评分及树统计上存在差异，详见独立笔记。

## LATS → VOYAGER：从本题搜索到跨任务程序积累

- **LATS**：保存当前任务的候选分支、访问统计和失败反思；主要改进本题的搜索过程。
- **VOYAGER**：为开放探索提出任务，修订实际执行的程序，将成功程序与描述存入可检索技能库，供后续目标和新世界复用；无MCTS，也没有LLM权重更新。
- **实现区别**：生成提示只放top-5技能，运行时载入整个库；成功入库没有自动验证新初态重放，不能把一次任务通过等同普遍鲁棒的技能。
- **实验边界**：160轮程序生成预算、结构化状态与高层API、死亡保留物品；表1钻石级仅1/3成功。表2新世界四目标各3/3，支持有限的同域组合迁移，不能宣称任意领域泛化。
- **版本边界**：正式身份为TMLR 2024；本次实读arXiv v2（42页）及官方代码55e45a8，期刊终稿受访问验证限制，未逐页比对。

## VOYAGER → Planner-R1：从外部程序积累到LLM参数RL

- **VOYAGER**：冻结LLM，执行反馈修改程序，成功代码进入跨任务技能库。
- **Planner-R1**：本文准确定位为旅行规划奖励塑形论文（2509.25779），不是近名Embodied Planner-R1或Plan-R1；用GRPO训练Qwen3权重，不建立长期技能库。
- **反馈粒度**：所谓密集奖励仍是计划结束后给约束部分分；整轨迹优势广播给生成token，没有逐调用的过程奖励模型。
- **实现边界**：正文按轨迹长度平均，附录配置token-mean；专用训练源码/权重未定位到，不能冒充已核验作者loss和mask。基准评测代码已另行固定版本核对。
- **实验边界**：180个不同训练查询不等于180条rollout；主训练16×H200，3.5倍效率只统计更新FLOPs、不含rollout。8B 3000步有两项域外退化，不能说完全没有遗忘。

## Planner-R1 → ToolRL：同为部分奖励，监督对象不同

- **Planner-R1**：模型在训练采样中多轮查旅行工具，最终按整份计划的全局约束打分；格式是奖励门控。
- **ToolRL**：主训练将多步历史拆成单步样本，比较下一轮工具名、参数key和值；格式分与正确性相加。部署与Bamboogle评测才包含实际循环搜索。
- **出发点**：工具结构可以提供比全对/全错更细的排序信号；消融同时检验长度、权重与平滑课程，贡献重在可验证设计和系统实验，不是新GRPO。
- **loss边界**：论文附录写响应概率比率，代码为token裁剪；默认源码仍扣0.001奖励KL并加0.001熵项，与论文主设置去KL有差异。没有历史日志不替作者消除冲突。
- **评测边界**：BFCL V3多轮7B仍只有18.12%；作者API-Bank适配597项是调用字典存在匹配，Bamboogle125题使用GPT-4o语义裁判。总体准确率不能代替真实完整任务成功率。

## 阅读与维护规范

- 先讲作者做了什么、输入输出、一个贯穿例子，再解释模块和公式。
- 主要篇幅用于论文本身；与研究设想关系放最后。
- 分开写论文事实、固定版本代码行为、教学假设和待验证设想。
- benchmark 必须说明具体任务、反馈来源、预算、分母与成功判据。
- 对 RL 明确策略、动作粒度、奖励、信用分配、数据流与更新对象；未涉及的部分写清边界。
- 官方来源链接优先，PDF 仅在线引用；不上传原文。
- 新增笔记同时更新索引、队列和必要主页导航，沿用 paper-note 样式。

## 本次核对范围

2026-10-17：读取 main 的 README、模板、记忆主题索引、文件树及已有 GPTSwarm 笔记；结合当前聊天的图片队列去重。四篇既有记忆主题笔记保留，RAP 为新主题第一篇。RAP 有正文/表格数字冲突及现代代码与历史配置差异，详见精读。


2026-10-19：完成 LATS 会议最终版与关键附录精读，核对官方代码 853d816；新增独立笔记并保留 oracle、正式测试调用、配置/数字冲突。未实际运行复现。队列下一篇 VOYAGER。

2026-10-20：按用户一次性请求新增CB-PbRL资料笔记，确认NeurIPS 2026录用；全文受访问验证限制，已明确证据范围和未核验项。单列队列外补充，不计入2/19；下一篇及定时任务保持不变。

2026-10-20（全文重写）：取得用户上传的28页ICLR匿名稿CB-RLHF，重写原资料笔记；核对正文、附录、表1和曲线，补齐标签/loss、对偶与超梯度、理论边界、环境与SAC配置。上传稿精读完成；源码与NeurIPS终稿版本对照未完成。固定队列及定时任务不变。


2026-10-21：完成VOYAGER arXiv v2正文与附录精读，渲染核对表1/2和图9，静态核对官方代码55e45a8；新增独立笔记，区分TMLR发表身份、实际阅读版本、外部技能更新与参数RL，保留预算/判定/环境辅助和实现差异。未运行游戏复现。队列3/19，下一篇Planner-R1；队列外CB-PbRL及记忆主题内容保留。

2026-10-22：以用户提供的CB-PbRL教学重写版替换原笔记；保留其14节正文与讲解顺序，检查公式、表格、Mermaid与内部链接，更新过时的队列提示。文件名和索引入口保持稳定；固定队列仍为3/19，下一篇Planner-R1。

2026-10-23：完成Planner-R1（Reward Shaping，2509.25779v2）正文、附录A/B与C关键案例精读；核对奖励、GRPO公式、公开配置、表1/2与FLOPs范围，静态检查TravelPlanner评测e52c87f。未定位到专用训练代码/模型，未运行复现；仅标预印本。队列4/19，下一篇ToolRL；CB-PbRL重写与七篇记忆主题内容保留。


2026-10-25：完成ToolRL NeurIPS 2025主会正式31页稿及关键附录，核对官方代码8cee13e的数据、奖励、组优势、token loss、默认KL/熵与三套评测；运行纯Python奖励样例，未训练GPU或复现成绩。保留贪心/最优匹配、论文/代码公式、Equal Max数字和项目页冲突。队列5/19，下一篇ARTIST；队列外CB-PbRL和八篇记忆主题内容保留。

2026-10-27：完成AgeMem正式ACL 2026主会27页稿、附录A–D与作者代码98f563f精读；新增完整笔记及临时优先队列，核对实际多轮执行、优势广播、默认grpo/step-wise分组差异、奖励/KL配置与成本边界。只运行本地纯Python代理检查，未训练复现。GitHub连接工具未提供，命令行推送预检缺少认证，远程main未更新；保留本地提交/补丁与可恢复笔记，优先恢复同步，不将新增论文计入原19篇。


2026-10-29：恢复GitHub连接，补交AgeMem完整笔记、优先队列和主页导航；优先队列1/5，下一篇Memory-R1。原19篇5/19、恢复位置ARTIST不变。


2026-10-29（Memory-R1）：完成ACL 2026主会21页正式稿与A–G附录精读，核对作者仓库9c413a2尚无训练实现；渲染检查式(2)/(3)、表1及算法3–5。新增单篇笔记，同步标记优先队列第2项与原队列第10项；优先2/5、原队列6/19，下一篇MemSkill，恢复原队列位置ARTIST。不修改定时日程，不声称已GPU复现。
