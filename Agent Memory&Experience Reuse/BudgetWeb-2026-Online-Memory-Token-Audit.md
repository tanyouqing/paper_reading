---
title: "BudgetWeb：在线记忆与技能是否值得它们消耗的 token？"
aliases:
  - BudgetWeb
  - Are Online Skill and Memory Modules Always Worth Their Tokens?
  - A Budget-Constrained Study of Web Agents
tags:
  - paper-reading
  - agent/memory
  - evaluation/budget
  - agent/web-navigation
year: 2026
venue: EMNLP
track: 主会
status: 精读与代码核对
cssclasses:
  - paper-note
updated: 2026-10-30
---

[[00-Agent-Memory-Experience-Reuse|← 论文索引]] · [[README|仓库主页]]

# BudgetWeb：在线记忆与技能是否值得它们消耗的 token？

> [!abstract] 作者做了什么？
> **把在线网页 agent 的记忆、技能生成、检索、验证和上下文 token 一起计入成本，再与“没有跨任务记忆、但能多操作几步”的普通 agent 比较。**
> 输入是连续的网页任务和相同底座模型；输出是任务成功率、完整模块 token 与重复运行统计。真正被改进的是评测和成本对照方式，没有提出新的记忆学习器或训练基础模型。

> [!info] 阅读范围与简称
> 已阅读 EMNLP 正式版 18 页正文、附录 A–G，并静态核对官方代码 `aa02981e37d222c8109eeed41e2c845d46f59615`；未重跑 API 或 GPU 实验。**BudgetWeb 是本笔记为方便索引使用的简称，不是作者命名的新方法。**

## 1. 论文信息与选择理由

- **原题**：Are Online Skill and Memory Modules Always Worth Their Tokens? A Budget-Constrained Study of Web Agents。
- **作者**：Sina Hajimiri、Masih Aminbeidokhti、Jose Dolz、Ismail Ben Ayed、Issam H. Laradji、Spandana Gella、Nicolas Gontier。
- **机构**：ServiceNow AI Research、ÉTS Montréal、University of British Columbia、McGill University；最后两位共同指导。
- **身份**：**EMNLP 2026 主会**，37676–37693 页，正式版标注会议日期 2026-10-24 至 2026-10-29。不是 Findings、Workshop 或仅预印本。
- **来源**：[ACL Anthology](https://aclanthology.org/2026.emnlp-main.1657/) · [正式全文](https://aclanthology.org/2026.emnlp-main.1657.pdf) · [作者项目页](https://sinahmr.github.io/budget-constrained-web-agents/) · [官方代码](https://github.com/ServiceNow/budget-constrained-web-agents)。
- **为什么选它**：我们已读多篇轨迹→经验→检索方法，以及 [[MaAS-2025-Cost-Aware-Agentic-Supernet|MaAS]] 的成本路由。本文直接检验这些模块是否带来超出额外计算的收益；对“技能迁移是否可靠、验证成本是否值得”的研究问题具有直接实证价值。仓库最新索引与可访问会话中均未精读过本文。

## 2. 出发点：先问对照是否公平

普通 agent 也可以读取网页、推理、点击、观察结果、修正错误。增强 agent 在这个执行器之外，再生成工作流、验证程序或维护策略记忆。

如果两者都限制十次决策，增强方法的成功率更高，容易得到“记忆有效”的结论。但它调用了更多模型，也在每一步携带更多文本。**比较的可能同时是“有没有记忆”和“有没有多花推理预算”。**

作者把问题改成：这些额外 token 用于记忆，是否比用于普通 actor 的额外环境交互更有效？这是预算的机会成本。普通 actor 指每一步真正决定浏览器动作的模型。

### 贯穿例子：历史购买流程遇到新的约束

以下是**教学例子，不是论文的原始轨迹**：新任务要买一盏符合预算、评分最高的灯。旧工作流写着“搜索灯→按评分排序→购买第一项”。

问题不只在有没有检索到相似任务。旧流程可能遗漏预算筛选、评分并列和当前排序方向。如果新 agent 没有检查这些差别，就可能很快买错；如果认真检查，判断旧流程是否适用也要花预算。

普通 actor 则可以把预算用来多看一次筛选结果、检查价格并纠正选择。本文要实测哪种花法更好，而不是先假设抽象经验必然胜过直接操作。

### 设计逻辑重建

> [!note] 公开动机支持的逻辑，不是作者私人思维过程
> 增强系统成功率上升 → 发现额外模块成本未被充分对齐 → 把所有模型调用一起计数 → 给普通 actor 增加交互机会 → 分开检验步数与页面压缩 → 重复运行检查是否只是偶然胜出 → 审计记忆污染和程序验证，以解释何时复用会失败。

研究对象是**在线增强**：执行较早任务后更新记忆，供之后任务使用。离线花大量计算构建技能库、再部署很多次的场景，成本摊销方式不同，不能直接套用本文结论。

## 3. 被比较的系统：谁保存什么、谁使用它

| 系统 | 跨任务保存内容 | 在当前任务怎样使用 | 回写依据 |
| --- | --- | --- | --- |
| AWM | 抽象工作流文本 | 作为操作指导进入 actor 上下文 | 模型判断成功后归纳、去重或替换 |
| ASI | Python 技能函数 | 加入可调用动作库，一次调用可含多个原子动作 | 成功轨迹归纳，再运行验证回合和检查 |
| ReasoningBank / RBank | 从成败经历提炼的策略记忆 | 召回语义相关条目并注入上下文 | 模型判断任务结果并提炼经验 |
| Vanilla-IB | 无跨任务技能或记忆库 | 当前目标、页面和本题动作历史指导下一步 | 不回写跨任务经验 |

**短期历史与长期记忆分开**：Vanilla-IB 并非每一步失忆，仍保留本题动作历史和错误反馈；它没有跨任务共享的长期经验库。

**执行图与页面树分开**：论文处理的可访问性树（accessibility tree / AXTree）是网页元素的树形描述，含角色、名称和元素 ID。对它剪枝是减少页面输入冗余，不是学习多 agent 通信图。

## 4. 方法：同样的额外预算可以怎么花

### 4.1 增强系统的一轮

1. 读入新任务、当前网页，以及已有工作流/技能/策略。
2. actor 根据观察产生动作，浏览器执行后返回新状态。
3. 持续到提交答案、任务结束或达到最多十次 actor 决策。
4. 用该方法的模型评价流程判断经历是否可用于归纳；这不是最终 benchmark 的真实成功评分。
5. AWM 归纳工作流；ASI 归纳代码并验证；RBank 从成败中总结策略。
6. 保存更新后的库，下一个任务使用它。它们没有在这里对基础模型做 RL 或蒸馏训练。

反馈改变的是**外部库内容和之后的模型上下文/可用动作**，不是模型参数。本文也没有从零提出上述三个更新机制，它们是被审计对象。

### 4.2 Vanilla-IB 的一轮

同样读目标与页面，也根据动作后果继续决策，但取消跨任务增强模块；最多十五次 actor 决策。用确定规则压缩页面树，尽量把增加五步的成本补回来。

所谓十五步是决策回合上限，不保证每题都走完；公开动作集允许同一决策含多个原子操作。因此不能把它说成“恰好点击十五次”。

**它不是新的流程搜索算法。**没有外层搜索树、MCTS、策略梯度训练或新提示词优化器。作者明确将它作为预算对照组。

### 4.3 页面树怎样压缩

假设某个按钮节点已经写着“提交订单”，其 StaticText 子节点又重复“提交订单”，就删除重复子节点。空文本也可删除。对段落、标题等正文容器设置例外，避免把正文掏空。

这是确定性字符串/树规则，不额外调用 LLM。它压缩输入，不负责判断商品是否符合条件。正式方法见附录 B；实现是 `constbudg/utils/prune_axtree.py`。

### 4.4 “预算匹配”到底多严格？

> [!warning] 步数近似，不是逐题 token 硬约束
> 三种增强方法的实际开销各不相同，任务会提前结束。本文采用十步对十五步作为主要预算控制，同时报告实测 token；它没有强制每个方法每道题正好使用相同 token。论文 §3.2 明确承认这一点。

因此，最准确的结论是“普通 actor 在近似可比的成本区间具有竞争力；部分设置还同时更准、更省”。不能写成“在每题严格等 token、严格等美元、严格等时间下均胜出”。

### 4.5 成本为何不止归纳器调用

下面是**教学记账式，不是论文提出的训练公式**：

$$
T_{\mathrm{total}}=T_{\mathrm{actor}}+T_{\mathrm{aux}}.
$$

$T_{\mathrm{actor}}$ 包含每一步的页面、任务、历史、注入的工作流/技能描述，以及模型输出；$T_{\mathrm{aux}}$ 包含归纳、筛选、清理、验证等辅助调用。

**重复注入成本已经在 actor token 里，不要再加一次导致双重计数。**把生成记忆的模型换成便宜小模型，只能降低一部分开销，不能自动消除 actor 反复阅读长记忆的成本。

回到灯具例子：如果操作指导没有解决预算条件，actor 每一步仍然要读它、判断是否偏离它。即使归纳过程很便宜，这份指导也未必划算。

## 5. 为什么技能“通过验证”仍可能不好用

附录 F 是与迁移可靠性最贴近的部分。

### 5.1 页面 ID 不是稳定的技能参数

ASI 技能是实际 Python 函数，封装点击和填写等动作。BrowserGym 元素标识 BID 可能在页面更新后改变。第一个动作成功打开页面，不代表后续动作里引用的 ID 仍然有效。

下面是**简化教学代码，不是原仓库中的真实技能**：

```python
def choose_product(sort_bid, first_item_bid):
    click(sort_bid)
    click(first_item_bid)  # 排序后页面更新，旧标识可能失效
```

这揭示的约束是“后一步参数依赖前一步的新观察”。并不意味着所有 Python 技能都脆弱：带新状态查询、稳定语义定位或稳定 API 的函数是不同设计，本文没有全面否定它们。

### 5.2 任务成功不等于技能自身成功

验证时先调用新函数；函数出错后，普通 actor 可以继续观察、用原子动作补救，最后完成整个任务。若整体成功被当作技能可靠的证据，信用就归错了对象。

Table 8 的 Gemini Shopping 平均每次完整运行：58.0 次归纳验证尝试，其中19.3次在第一个高层函数调用就失败；这些失败里69.0%随后恢复并仍被判正确。数字是多次运行平均，所以可以为小数。

这是有价值的失败统计，但不是某技能在未见任务变体上的成功概率，更不是所有入库技能的错误率。

> [!warning] 公开代码有额外检查，不能省略
> `induce_actions.py::write_tests` 不只是问模型“整任务成功吗”，还调用 `calc_valid_steps`。该检查要求至少一次诱导函数调用没有被判为错误并造成状态变化；它并不要求每个内部动作都完成其语义目标。
> 因此更准确的评价是：**现有检查仍不等价于完整技能契约验证**。没有运行日志、固定版本及入库事件的对应复算，不能声称当前代码“任何函数失败后只要任务成功就一定入库”，也不能将论文统计视为已独立复现。

### 5.3 错误经验可能在入库前就产生

AWM 用模型评价筛选成功经历；模型评价和环境真实评测可能不一致。Table 9：Shopping 中，Gemini 设置下49.5%的工作流归纳事件源自真实评测失败的任务；GPT设置为52.3%。

这里的分母是**归纳事件**，不是最终全部工作流，也不是每条指导都一定错。论文展示的是来源风险；要证明污染造成多少成绩下降，还需要替换评价器等干预消融。

RBank 也有标签污染。Table 10 中Gemini Shopping 的成功标签条目来源有52.9%对应真实失败轨迹；表中计数涉及任务与提炼记忆两个层次，不把它误写成“52.9%的每条记忆内容已被逐句证明错误”。

## 6. Benchmark 与评测协议

| 基准 | 任务形式 | 本文范围 | 评测含义 |
| --- | --- | --- | --- |
| WebArena Shopping | 搜商品、比较、购物等 | 187题 | 浏览器交互后按基准判整任务是否完成 |
| WebArena Reddit | 查找、导航、发帖等 | 106题 | 整任务成功率 |
| WebArena Admin | 商品管理、订单、账号管理等 | 182题 | 整任务成功率 |
| WebArena GitLab | 仓库、issue、项目管理等 | 180题 | 整任务成功率 |
| WorkArena-L1 | ServiceNow表单、知识搜索、列表排序/过滤、服务请求等 | 33种任务类型，每类3个种子 | 企业软件任务完成率 |

WebArena 共655题，排除 Map，论文说明网站可靠性问题；不是完整812题。任务按在线顺序运行，记忆只应从过去任务积累。这里没有通常的“训练模型/固定测试集”划分，而是评测流中的外部状态更新。

- WebArena：Gemini 3 Flash、GPT-5.4-mini、Qwen 3.6-27B，各域各方法三次独立运行，报告均值与标准差；总平均按域任务数加权。
- WorkArena-L1：只报告Qwen，不把一个模型的结果推广为三个模型均已验证。
- Vanilla-IB/AWM/ASI actor temperature=0；RBank保留原配置0.7，辅助模块保留各自默认配置。
- Vanilla-IB/AWM/ASI沿用ASI统一代码框架；RBank沿用自身实现，但论文说最终统一使用未修改BrowserGym评测器。
- 模型评价器用于归纳筛选，基准评分用于报告最终成绩。默认公开入口评价模型为GPT-4o，归纳默认跟随actor。**“比较底座相同”不等于系统内所有模型调用均同型号。**

## 7. 少量关键结果怎样支持研究主张

### 7.1 主要对照：不仅要看是否答对

正式版Table 1，Gemini3Flash，WebArena四域加权：

| 方法 | 成功率 | 每题总token |
| --- | ---: | ---: |
| Vanilla-IB | **44.78%** | **73.6K** |
| AWM | 39.34% | 99.3K |
| ASI | 41.02% | 107.3K |
| RBank | 39.33% | 82.6K |

相对ASI，Vanilla-IB高3.76个百分点，token少约31.4%。这支持这个设置下不需要在线技能模块也能得到更好的成功—成本组合。

其他底座也有相似的成功率排序：GPT普通actor32.67%，增强方法最高ASI29.00%；Qwen普通actor42.14%，ASI40.15%。但GPT设置下普通actor90.2K高于AWM88.5K与RBank81.0K，**不是所有模型下同时最省**。

WorkArena-L1（Table 2）：Vanilla-IB55.56±2.9%、109.4±14.8K；RBank55.56±2.8%、120.3±5.7K；AWM53.53±3.8%、102.8±2.6K。因此普通actor与RBank同分且更省，但AWM费用更低。不能说WorkArena上普通actor支配全部增强方法。

### 7.2 成本拆分：多数成本仍在actor

Table 3，Gemini Shopping：

| 方法 | actor输入 | actor输出 | 辅助输入 | 辅助输出 | 合计 |
| --- | ---: | ---: | ---: | ---: | ---: |
| Vanilla-IB | 45.2K | 0.5K | 0 | 0 | 45.7K |
| AWM | 64.1K | 0.4K | 6.1K | 0.2K | 70.8K |
| ASI | 61.8K | 0.5K | 20.0K | 0.5K | 82.8K |

AWM的辅助调用只有6.3K，但总成本差25.1K，提示只统计“归纳器额外调用”会漏掉重要部分。**差值还受步数、页面路径和剪枝影响，不能全归因于记忆文本长度。**这张表是成本分解，不是对记忆上下文的独立因果消融。

### 7.3 是剪枝赢了，还是多走几步赢了？

Table5的Gemini Shopping：

| 普通actor | 成功率 | token |
| --- | ---: | ---: |
| 10步，不剪枝 | 45.81% | 48.0K |
| 10步，剪枝 | 46.35% | 41.7K |
| 15步，不剪枝 | 47.24% | 54.0K |
| 15步，剪枝 | 47.77% | 45.7K |

增加步数主要改善成功率，剪枝主要减少输入开销。其他域中剪枝不总提高准确率，例如Admin十五步从57.14降到55.68。两个改变不能混成一个必然增益。

附录C也给增强方法剪枝：Qwen Shopping的ASI达到45.81%，接近Vanilla-IB45.99%，但使用76.7K对58.8K。这个小精度差距不能据此宣布显著胜出；成本差异仍值得关注。

附录D提高步数：增强15步、普通20步，Qwen三域平均普通48.6%/106.2K，ASI47.4%/145.6K；Admin单域ASI51.83略高于普通51.65。实验支持较高预算下总体趋势，不支持所有域逐项都胜出。

### 7.4 多次运行解决了什么，又没解决什么？

Any-of-3表示某题三次至少成功一次；All-of-3表示三次全成功。Gemini Shopping普通actor：平均47.77%，Any54.01%，All42.25%（Table4）。

这说明平均相近仍可能换了一批成功题。**Any不是可部署单次成功率；All不是理论成功概率下界或置信区间。**这是重复相同题目的稳定性诊断，不是新任务分布上的迁移可靠性保证。三次运行也不等于已经做了充分显著性分析。

## 8. 创新贡献怎样评价

本文不以新架构为贡献，作者明说Vanilla-IB只是控制组。比较独特的价值在于：

1. 将**普通actor多交互的机会成本**加入在线经验复用的对照，挑战“同决策步数就是公平”的评测习惯。
2. 将**显式辅助调用与隐式上下文膨胀**放入同一成本账本，而非只统计记忆生成器。
3. 联合报告三底座、多个网页域、另一个企业任务集及重复运行，并通过剪枝/步数拆解和失败审计解释结果。

与AWM/ASI/RBank相比，本文是在检验其价值边界；与MaAS相比，它不学习成本路由，而是问增强模块相对更简单替代方案是否值得；与NSI相比，它关注执行技能的脆弱性，但没有提出NSI式条件控制流或新验证器。

**实验确实支持**：在被测在线设置与成本区间，增强模块没有稳定压过较强普通actor。

**尚未证明**：现代底座已经普遍内化全部规划能力；记忆污染是全部性能差异的主因；任何程序技能都必须每子步调用LLM；离线库和长部署期也没有价值。后几项需要新的对照或更宽范围实验。

## 9. 代码核对、复现与证据边界

固定版本：[aa02981](https://github.com/ServiceNow/budget-constrained-web-agents/tree/aa02981e37d222c8109eeed41e2c845d46f59615)。本次核对主调度、actor上下文、工作流更新、技能测试、剪枝、模型用量记录和日志统计。

### 已接通的可见路径

- `constbudg/run_online.py`提供vanilla/awm/asi三入口，任务串行处理，`--max_steps`控制决策上限；`--run_suffix`隔离各次结果、工作流和函数库。
- AWM：执行→`evaluate_trajectory`→清理有效轨迹→`induce_memory`；工作流归纳读取已有库，按模型重叠判断与质量比较保留/替换，再重写文本库。
- `agent.py`初始化时读取工作流文件；每次决策将`self.memory`注入系统上下文。**这条公开路径注入整份文件，没有实现正文概述的“检索最近一条”或top-k；不能按论文叙述补出不存在的检索模块。**可能版本不一致，需要作者/日志进一步对齐。
- ASI：先生成动作函数与验证轨迹，再运行原题/子任务验证，结合模型回合评价与有效诱导动作检查决定回滚；不是形式化验证，也没有未见变体的独立保证。
- `llm_client.py`记录各提供方prompt/completion/total usage；`scripts/analyze_log_tokens.py`做模块统计。

### 必须说明的复现缺口

1. 当前`run_online.py`没有RBank入口。论文称采用其单独官方实现；当前仓库不能一条命令复现所有四方法。
2. 发布入口及README主要覆盖WebArena，未见与Table2完全对应的WorkArena-L1整套命令、33类型/种子清单与配置。
3. token分析器默认排除被重试替代的非最后一次执行；超时被杀进程还可能没有打印最终actor累计。`--include-non-final-retry-attempts`可计入有日志的旧尝试，但不保证补回没有输出的调用。
4. 因此应区分“过滤基础设施故障后的benchmark推理量”与“实际支付的完整运行账单”。没有原始日志不能独立确认发表表格用了哪一套过滤口径，也不能断言所有表格一定漏计多少。
5. 验证入库有实际额外检查，因此附录F的第一调用失败、之后补救、最终入库链条必须按相同版本逐事件重放，不能只用最终分数验证叙事。
6. 模型输出、页面故障与任务顺序会改变在线记忆内容；必须固定初始库、任务顺序、模型快照、温度、重试政策和回写边界。可访问代码没有给本次复算所需的全部发表运行日志。

### 硬件与成本

论文附录A报告Qwen3.6-27B用**4张NVIDIA H100、vLLM**部署。Gemini/GPT使用模型服务。没有基础模型训练。

官方开放代码和配置生成入口；WebArena/WorkArena依赖外部基准和网站服务，并非仓库内完整打包。环境为Python3.12、BrowserGym/Playwright/Chromium，README指定`libwebarena==0.0.5`。网站服务、访问凭据和模型服务仍需配置。

**未知**：完整GPU小时、全部实验美元账单、完整端到端延迟、页面服务服务器资源。token数不直接等于费用：输入/输出价格、缓存、模型、重试不同。未给这些资源编造估算。

## 10. 与研究选题的关系和一个小实验

| 方向 | 关系 | 本文真正提供的东西 |
| --- | --- | --- |
| 经验优化Harness | 直接相关 | 检验额外模块与普通执行器的成本收益 |
| 长短期记忆、检索 | 直接相关 | 区分本题历史和跨任务记忆，审计污染与重复上下文成本 |
| 单agent执行 | 直接相关 | 普通actor的额外交互是重要对照 |
| 多agent运行图协作 | 可迁移评测原则 | 没有优化角色、通信、拓扑或跨agent经验传播 |
| RL/参数蒸馏 | 未覆盖 | 不训练LLM参数，不提出RL算法 |

它为“预算约束的技能迁移可靠性验证”补上一个理由：**验证资源值得花，但必须证明比直接继续执行更值得；同时要区分技能贡献与actor补救。**本文没有完成可靠性概率校准、新变体验证或接受/修复/拒绝决策框架。

**建议的小规模实验，不是论文结论**：固定一个网页域、模型、初始技能和总实测token预算，比较普通actor追加操作、在线技能+现有回合验证、在线技能+独立技能后置条件验证。把适配用任务和未见确认变体分开，记录最终SR、技能独立成功率、补救率、错误接受率、完整token（含验证生成和故障重试）。检验“更严格验证是否减少错误复用且仍有净收益”。

## 11. 三个值得思考的问题与阅读路线

1. 同一个技能在目标、页面和前置条件变化到什么程度后，验证经验应该失效？怎样以有限测试确定边界？
2. 在相同实际token预算下，怎样选择“验证技能一次”还是“让actor直接多做几步”？
3. 技能调用失败后任务被补救完成，怎样分别估计技能收益、补救成本和整系统成功率？

**先读**Figure1和§1，理解预算机会成本；再读§3.2，避免把近似预算当严格匹配；随后Table1、3、5和AppendixC/D；接着AppendixF.1与Table8，区分技能验证和回合成功；最后§8限制、AppendixG和公开计费/入库路径。

## 原始来源

- [官方会议记录](https://aclanthology.org/2026.emnlp-main.1657/) · [正式论文18页](https://aclanthology.org/2026.emnlp-main.1657.pdf)
- [作者项目页](https://sinahmr.github.io/budget-constrained-web-agents/) · [机构页面](https://www.servicenow.com/research/publication/sina-hajimiri-are-emnlp2026.html)
- [固定代码版本](https://github.com/ServiceNow/budget-constrained-web-agents/tree/aa02981e37d222c8109eeed41e2c845d46f59615)
- [在线执行](https://github.com/ServiceNow/budget-constrained-web-agents/blob/aa02981e37d222c8109eeed41e2c845d46f59615/constbudg/run_online.py) · [actor上下文](https://github.com/ServiceNow/budget-constrained-web-agents/blob/aa02981e37d222c8109eeed41e2c845d46f59615/constbudg/agent.py)
- [技能测试](https://github.com/ServiceNow/budget-constrained-web-agents/blob/aa02981e37d222c8109eeed41e2c845d46f59615/constbudg/induce/induce_actions.py) · [有效动作检查](https://github.com/ServiceNow/budget-constrained-web-agents/blob/aa02981e37d222c8109eeed41e2c845d46f59615/constbudg/utils/calc_valid_steps.py)
- [工作流归纳与维护](https://github.com/ServiceNow/budget-constrained-web-agents/blob/aa02981e37d222c8109eeed41e2c845d46f59615/constbudg/induce/induce_memory.py) · [页面树剪枝](https://github.com/ServiceNow/budget-constrained-web-agents/blob/aa02981e37d222c8109eeed41e2c845d46f59615/constbudg/utils/prune_axtree.py)
- [模型usage记录](https://github.com/ServiceNow/budget-constrained-web-agents/blob/aa02981e37d222c8109eeed41e2c845d46f59615/constbudg/llm_client.py) · [日志token统计](https://github.com/ServiceNow/budget-constrained-web-agents/blob/aa02981e37d222c8109eeed41e2c845d46f59615/scripts/analyze_log_tokens.py)
