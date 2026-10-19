---
title: "LATS：让语言智能体在环境反馈中搜索、回溯与反思"
aliases:
  - LATS
  - Language Agent Tree Search
tags:
  - paper-reading
  - agentic-rl/planning
  - inference-time-search
  - agent/reflection
year: 2024
venue: ICML
track: 主会
status: 精读整理
cssclasses:
  - paper-note
updated: 2026-10-19
---

[[00-Agentic-RL|← Agentic RL 索引]] · [[RAP-2023-Reasoning-via-Planning|对照 RAP]]

# LATS：让语言智能体在环境反馈中搜索、回溯与反思

> [!abstract] 作者到底做了什么？
> 把冻结语言模型提出的多个行动放进搜索树，实际调用环境获得观察，用模型评分选择值得继续的分支，并把失败轨迹写成反思，让同一题的下一次尝试避开已知错误。
> 输入是任务、可交互且可回退的环境、提示与搜索预算；输出是最终答案、程序或购物轨迹。改变的是当前题的树统计与上下文，不是 LLM 权重。

## 1. 论文信息与证据范围

- **完整题目**：Language Agent Tree Search Unifies Reasoning, Acting, and Planning in Language Models。
- **作者**：Andy Zhou、Kai Yan、Michal Shlapentokh-Rothman、Haohan Wang、Yu-Xiong Wang。
- **机构**：University of Illinois Urbana-Champaign；Andy Zhou 另署 Lapis Labs。
- **发表**：ICML 2024 主会，PMLR 235，62138–62160。早期 ICLR 投稿页不能代替正式发表记录。
- **来源**：[PMLR 正式页](https://proceedings.mlr.press/v235/zhou24r.html) · [会议最终 PDF 含附录](https://raw.githubusercontent.com/mlresearch/v235/main/assets/zhou24r/zhou24r.pdf) · [arXiv 版本](https://arxiv.org/abs/2310.04406) · [官方代码](https://github.com/andyz245/LanguageAgentTreeSearch)。
- **选择理由**：用户指定队列第 2 篇，接在 RAP 后，比较“用模型预测状态”和“让环境真正返回观察”。
- **核对日期/版本**：2026-10-19，官方 main 提交 [853d81614607dd27433faf17c7b0a7d660f95d22](https://github.com/andyz245/LanguageAgentTreeSearch/tree/853d81614607dd27433faf17c7b0a7d660f95d22)。
- **阅读范围**：会议最终版主文、算法1、关键附录的限制、配置、消融和提示；静态核对三类实现与执行器。未运行 API 或复现成绩。
- **去重**：已读最新 README、Agentic RL 索引、模板、RAP 笔记与该目录；仅 RAP 已完成，没有 LATS 文件。未声称检查不可访问历史。

> [!info] 证据标签
> 论文事实、代码核对、教学假设、研究逻辑重建和建议实验分开标识；不宣称知道作者未公开思维过程。

## 2. 出发点：能查资料，为什么还需要搜索？

ReAct 将“思考 → 动作 → 环境观察”串成单条轨迹。它能查工具，但早期选错实体、查询词或程序结构后，容易围着错误选择打转。增加整条重试，未必保留已正确的前缀；只让模型反思，也可能反复得到空泛建议。

LATS 要保存分叉，保留有用前缀，在观察了实际结果后重新分配尝试次数。

> [!example] 贯穿教学例子：全部虚构，不是论文样本
> 问：“小说《潮汐之城》的作者出生在哪座城市？”
> 假设资料库中小说作者林澈出生于海州市；另有同名电影，其导演出生于青山市。
> 单条 ReAct 可能搜到电影，沿导演页找出生地，最后答青山市。页面是真实的，但关系链错了。
> 下文全部作品、人物、地名、评分与数值例子仅用于教学。

问题可能不在知识，而在“当前题先查哪个实体”的决策。LATS 让“查电影”“查小说及作者”等候选并存，再根据观察和反馈选择下一步。

## 3. 架构：状态、动作、输入输出

| 对象 | 实际含义 | 教学例子 |
| --- | --- | --- |
| 输入 x | 自然语言任务 | 小说作者的出生城市 |
| 环境 | 工具/API、代码执行器、购物模拟器 | 资料搜索与页面查找 |
| 状态 s | 问题与动作/观察历史；环境还可能有内部状态 | 已打开电影页 |
| 动作 a | 思考、工具命令或一份完整程序，依任务而定 | Search[小说 潮汐之城] |
| 观察 o | 执行环境返回的文本 | 小说页写作者林澈 |
| 输出 y | 最终答案、程序或完成任务轨迹 | Finish[海州市] |
| 搜索记忆 | 本题树、访问统计、失败轨迹和反思 | 不要把导演误当作者 |

动作不统一等于单 token。HotpotQA 一步是工具调用及思考；编程一步可是一份完整代码。模型逐 token 生成文本，但搜索控制器按更大单位组织它。

动作生成、状态评价、反思是功能角色，可复用同一 LLM，不等于三个协作 agent。部署时仍需搜索器、环境接口与提示；没有新训练权重替代它们。

~~~mermaid
flowchart TD
    A[任务与已有树] --> B[选择待展开分支]
    B --> C[LLM 提出多个候选]
    C --> D[恢复分支并调用环境]
    D --> E[评价观察后的状态]
    E --> F[沿高分分支继续试行]
    F --> G{终局反馈}
    G -->|失败或部分成功| H[回传数值并生成反思]
    H --> B
    G -->|成功| I[返回结果]
~~~

## 4. 一轮怎样运行？为什么这么设计？

按论文 §4.2、图2解释通用方法，代码差异见第8节。

### 4.1 选择：从已有树找值得继续的前缀

最初只有问题。后续从根沿已有分支走到待展开节点，兼顾既往表现与探索不足。不是每轮忘掉过去独立生成整条答案。

电影分支多次失败后，小说分支若尝试较少且观察相关，就可能获得下轮机会。

### 4.2 展开：同一前缀提出多个下一步

采样 n 个候选，例如查电影、查小说或在当前页找作者。候选各自调用环境，实际观察构成不同子节点；不能让模型编造“搜索结果”当作事实。

对纯推理环境，论文也有 CoT 版本，不能把所有实验说成真实工具交互。

### 4.3 评价：先看观察，再估计前途

评价器读问题、部分思考、动作、观察，输出分析与标量分数。还没回答不等于低分：小说页确认作者，是有价值的前缀；电影页再可信，也可能不回答小说问题。

论文式(2)：

$$
V_{\mathrm{init}}(s)=\lambda\,\mathrm{LM}(s)+(1-\lambda)\,\mathrm{SC}(s).
$$

LM(s) 是模型评分；SC(s) 是同状态采样的候选重复/一致性信号；λ 调权重。论文 HotpotQA/Game of24 用0.5，Programming/WebShop 用0.8。init 是本笔记为区分初始启发式与后续树估计而加的下标，原文都记 V。

一致性不是正确概率：模型也会重复推荐错实体。论文未提供跨任务都可唯一还原的 SC 归一化细节，当前代码也不能视作该混合式的逐字实现。

### 4.4 模拟：沿高分路继续到终局

小说分支找到作者，再查出生地并交答案。每层继续提候选、调用环境和评价。

这里模拟通常是在可恢复环境中实际试行；与 RAP 让 LLM 充当世界模型预测状态有区别。编程中动作已是完整代码，论文跳过一般长轨迹模拟，直接运行合成测试，用通过比例反馈。

### 4.5 回传：终局结果影响早期选择

HotpotQA oracle 会告诉提交答案对错。电影路线失败，回报沿分支回传，影响后续探索。

论文式(1)的 UCT（树上的置信上界）：

$$
\operatorname{UCT}(s)=\overline V(s)+w\sqrt{\frac{\ln N(p)}{N(s)}}.
$$

p 是父节点，N 是访问次数，w 控制探索。第一项偏向好表现，第二项给少访问分支机会。零访问须按实现初始化，不能直接除零。

> [!example] 教学数值
> 父节点访问10次，A平均价值0.6、访问8次；B为0.4、访问2次；w=1。
> UCT(A)≈1.136，UCT(B)≈1.473，下一次可选B。UCT不是概率，大于1正常。

正文 §4.2 的回传式把路径节点下标与新旧更新混写。以下用“同一节点的新旧统计”表达均值回传，属于教学澄清：

$$
N_{\mathrm{new}}(s)=N_{\mathrm{old}}(s)+1,
$$

$$
\overline V_{\mathrm{new}}(s)=
\frac{N_{\mathrm{old}}(s)\overline V_{\mathrm{old}}(s)+r}
{N_{\mathrm{new}}(s)}.
$$

r 是本次试行回报。假设此前2次均值0.8，本次r=0，新均值1.6/3≈0.533。路径上节点逐个更新，下轮读取新值。

这把后续成败关联到早期选择，是搜索的信用分配，不是 LLM 反向传播，也不证明各动作对成败的因果贡献。

### 4.6 反思：分数之外保存为什么失败

反思模型读失败轨迹与回报，生成教学建议：“问题问小说作者，我却沿同名电影的导演查出生地；先核对作品类型和作者身份。”

数值回传改变“哪条路再试”，文字反思改变“再试时考虑什么”。论文设计把失败轨迹和反思同时送给后续动作生成器与价值评价器，影响候选与评分。

这是上下文学习。论文“语义梯度”是类比，没有可微文本 loss。成功或预算用尽结束搜索；找到环境验证成功与模型自信成功需区分。

## 5. 是否存在 RL 训练、优势函数或代码 loss？

| 问题 | 本文实际包含 |
| --- | --- |
| 参数训练数据 | 没有 LATS 训练集；有少样本提示与本题试行 |
| 策略/价值网络训练 | 没有；复用冻结 LLM |
| 奖励去向 | 树统计回传、终局判断、失败反思 |
| 优势、重要性比率、PPO/GRPO、KL | 没有这些参数优化组件 |
| 理论目标与代码 loss | 本方法没有策略训练 loss |
| token/sequence loss 归一化 | 不适用，不能因逐 token 生成就虚构 token-level RL |
| on/off-policy | 不应当作 PPO 式参数训练算法归类；它是推理时规划 |
| 蒸馏 | 未将成功轨迹蒸馏到权重 |
| 跨题长期经验库 | 主方法未建立自动跨题检索/训练循环 |

HotpotQA lats_search 开头清空 failed_trajectories 和 reflection_map。论文称树为外部长期记忆，但主要证据是同题内跨尝试保留，不能直接等同跨任务长期经验库。日志落盘也不自动构成学习机制。

## 6. Benchmark：题目、反馈和正确判据

### 6.1 HotpotQA：多跳查询，而且允许问答案是否正确

Search[entity] 返回页面开头或相似实体；Lookup[keyword] 查当前页含词句子；Finish[answer] 结束。思考与这些命令共同构成动作空间。

论文 §5.1/D.1：随机100题，每法3个示例，GPT-3.5，通常 n=5、k=50。显式 oracle：提交答案环境给正误，允许重试。指标 EM 是最终短答案匹配，不是推理链或引用正确率。

代码 hotpot/wrappers.py：去大小写、标点、冠词、整理空白后比较字符串。不是 LLM 裁判，也不保证任意语义等价表述都算对。

因此这是“有答案正误反馈时怎样利用重试”的评测，不是普通真实问答的天然条件。正误oracle不等于把答案文本直接给模型；info 有 gt_answer 也不能仅据字段断言所有提示使用它。

**配置差异**：附录C写主实验深度7，D.1写6，当前 expand_node 用7；lats.sh为30次，主表50；lats.py初始化 split="train"。当前脚本不能直接还原“随机100题”历史设置。

### 6.2 HumanEval/MBPP：程序必须执行，测试分两层

输入函数签名/描述，输出Python函数。候选运行LLM生成的assert；失败断言、异常等作为观察，再反思修订。动作单位是一份程序。

论文§5.2协议：搜索中用合成内部测试回报，结束后选一份代码，再用正式测试计算pass@1。
- HumanEval全部164题，部分基线161题；
- MBPP随机397题；
- n=5、k=8。最终评一份程序，不等于内部只生成一次。
- 主文说4个内部测试，附录D.2说GPT-3.5为6、GPT-4为4；当前GPT-4脚本为2。

> [!warning] 公开编程实现的测试隔离问题
> programming/mcts.py 在循环内，当候选通过内部测试或到最后一轮，会调用 exe.evaluate(..., item["test"])。
> 正式测试通过会设置is_solved、保存解并提前停止；失败仍可能继续搜索。初始根解通过内部测试也会立刻正式评测并结束该题。
> 正式测试成败参与搜索/停止，不能称为“隐藏测试只在最后评一次”。
> 这是固定公开版本的静态行为；不能据此断言全部历史实验均如此或推断造成多少提升。

### 6.3 WebShop：平均满足条件不等于全部满足

输入商品需求，输出搜索、点击、选选项和购买。模拟环境按商品属性/选项匹配算奖励，涉及词汇和语义相似度，不是任意答案交LLM裁决。

50条指令，GPT-3.5、n=5、k=30、最大深度15。
- Score=100×平均reward；
- SR=reward恰为1的任务比例。

虚构需求红色、M码、低于预算，满足颜色尺码但超价可能有部分分数，仍非完全成功。原文评分不能简化为所有属性等权计数。

实现复制/恢复sessions用于分叉；回退模拟器不等于撤销真实付款。只复制对话历史不能普遍恢复外部状态。

### 6.4 Game of24

4个数各用一次，以基本运算得到24；50题，GPT-3.5，n=5、k=30、深度5。衡量合法等式成功率，不是工具调用能力。

## 7. 关键结果、消融与成本

| 任务/表 | 对照 | LATS | 支持范围 |
| --- | --- | --- | --- |
| HotpotQA表3 | ReAct32%、best-of-k38%、Reflexion51%、RAP(ReAct)54% | ReAct版本63%；CoT+ReAct71% | oracle与重试协议中更好利用搜索/反馈 |
| HumanEval表4 | GPT-3.5 Reflexion68.1%；GPT-4 Reflexion91.0% | 83.8%；92.7% | 差15.7/1.7个百分点；测试隔离和分母需核验 |
| MBPP表5 | Reflexion70.0% | 81.1% | 同样受实现测试协议边界影响 |
| WebShop表6 | ReAct53.8/28%；Reflexion64.2/35% | Score75.9/SR38% | 比ReAct加22.1分、SR加10个百分点 |
| Game of24表7 | RAP40% | 44% | 50题只差2题，需重复验证 |

表6 fine-tuning对照Score67.5/SR45%。LATS平均得分高、完全成功率低；不能说所有指标都优于训练方法。底座/预算也不完全一致，不能推出搜索普遍优于RL。

### 7.1 消融

表8 HotpotQA：完整63%，去反思58%（−5个百分点），改DFS42%（−21），去LM启发式37%（−26）。
说明该配置中评分和搜索控制作用明显，反思有额外价值；不同移除方式不能直接变成普适重要性排序。

表13 Game of24：混合一致性44%，仅LM评分40%；小样本差异且缺少充分区间。

### 7.2 成本与冲突

- 表9成功搜索条件下LATS173,290 tokens、RAP176,500、ToT210,215。不是所有题含失败的无条件平均，也不是API调用数或美元。
- 表10 k=50 LATS为61%，主表/表9为63%，保留冲突。
- 同样轨迹数不等同token预算；展开、评分、反思都要调用。
- 项目页旧HumanEval文案94.4%，正式会议摘要/表4为92.7%，采用会议版。
- 未充分报告逐实验GPU型号/数量、时长、API总费用。致谢提Delta NVIDIA GPUs，不能填具体配置。
- 无参数训练成本，但搜索推理成本不低。旧模型快照、接口与依赖也增加复现难度。
- 本次没有预训练污染审计，不断言污染。正式测试进入搜索是另一个已观察的协议问题。

## 8. 代码核对：不是一个完全统一的MCTS库

所有链接固定到853d816，未运行实验。

### 8.1 HotpotQA

[hotpot/lats.py](https://github.com/andyz245/LanguageAgentTreeSearch/blob/853d81614607dd27433faf17c7b0a7d660f95d22/hotpot/lats.py)：
- lats_search清空本题反思/失败，选择、展开、评分、rollout、回传；reward=1可提前返回。
- generate_new_states解析thought/action，env.step返回观察奖励，以thought::action去重。
- get_unique_trajectories按最终答案去重；get_samples仅在独特失败数大于已有反思数且小于4时生成反思，不是无限累积。
- evaluate_node直接用模型评分，未见显式λ与SC混合式。
- backpropagate把value存均值，但uct对已访问节点又除visits；与论文平均价值形式有差异。
- evaluate_node过滤终局节点后得分，却按完整children写回，复现需检查索引对齐。

[hotpot/hotpotqa.py](https://github.com/andyz245/LanguageAgentTreeSearch/blob/853d81614607dd27433faf17c7b0a7d660f95d22/hotpot/hotpotqa.py)：
- cot_prompt_wrap把失败反思放入生成上下文；
- value_prompt_wrap先构造带反思版本，随后if ... and False对应else又覆盖成普通value_prompt_reasoning，因此该版本评分器不保证获得论文所述反思上下文；
- value_outputs_unwrap按数字字符串包含关系解析，解释文本数字也可能干扰，不是稳健结构化解析。

[wrappers.py](https://github.com/andyz245/LanguageAgentTreeSearch/blob/853d81614607dd27433faf17c7b0a7d660f95d22/hotpot/wrappers.py)核对EM与奖励。

### 8.2 Programming

[run_lats_gpt4.sh](https://github.com/andyz245/LanguageAgentTreeSearch/blob/853d81614607dd27433faf17c7b0a7d660f95d22/programming/run_lats_gpt4.sh) → main.py的mcts策略 → [mcts.py](https://github.com/andyz245/LanguageAgentTreeSearch/blob/853d81614607dd27433faf17c7b0a7d660f95d22/programming/mcts.py) → [py_executor.py](https://github.com/andyz245/LanguageAgentTreeSearch/blob/853d81614607dd27433faf17c7b0a7d660f95d22/programming/executors/py_executor.py)。

execute运行内部assert；evaluate运行item["test"]和check(entry_point)。mcts每轮重新生成内部测试，不同轮奖励未必来自固定测试；gather_context_from_tree汇集祖先反馈与反思。

此目录value是累计奖励，UCT用value/visits；与HotpotQA的存均值方式不同，不能将不同目录的统计语义混用。

### 8.3 WebShop与复现

[webshop/lats.py](https://github.com/andyz245/LanguageAgentTreeSearch/blob/853d81614607dd27433faf17c7b0a7d660f95d22/webshop/lats.py)先深拷贝父节点env_state，恢复sessions，再执行候选；评分做归一化和终局加分，非论文式逐字实现。

需安装/启动WebShop并改本地地址。README说明提供编程轨迹，但HotpotQA/WebShop完整日志未上传。历史模型快照、数据文件、依赖和接口需另核对；现有模型替换不能冒充历史成绩复现。资源未报告者未知，不估填。

## 9. 增量贡献与简短研究联系

| 对照 | 主要新增 | 边界 |
| --- | --- | --- |
| ReAct | 单轨迹改分支、回传、回溯 | 不同调用预算 |
| Reflexion | 反思之外复用可回溯前缀 | 两种机制收益重叠 |
| ToT | 交互观察与MCTS适配 | “首个统一框架”是作者主张 |
| RAP | 环境转移代替LLM世界模型预测，加入失败反思 | 需要可回退环境 |

**研究逻辑重建**：候选提供多样性；环境减少事实臆测；评分填补稀疏终局奖励；UCT分配探索；反思传具体错误原因。每项仍可能出错，组合效果取决于任务与预算。

与用户方向：同题轨迹复用、外部上下文更新直接相关；跨题长期记忆可迁移但未完成；未研究多智能体拓扑、LLM参数RL或蒸馏。状态搜索树不是多agent运行图。

**建议小实验，非论文结论**：20–30道可执行验证的小程序题，同模型、同总token/API预算，比较独立重试、无反思搜索、有反思搜索。内部测试与正式测试隔离，最终隐藏测试只评一次；记录成功率、分项token、环境调用、每成功成本及“内部通过/正式失败”比例。检验收益来自分支复用/反思还是更多尝试与oracle信息，小样本不声称复现表4。

## 10. 阅读路线与三个问题

先图2/§4.2 → §5.1的oracle → §5.2测试分离 → 表6 Score/SR → 附录A–D → programming/mcts.py的evaluate调用。

1. 不允许查询标准答案正误、也不允许搜索途中接触正式测试，增益还剩多少？
2. 固定token与环境调用预算，分支搜索在哪些错误类型上胜过独立重试？
3. 反思同时影响候选和评分，会纠错还是放大共同偏见？独立评分器能否改善？

## 原始来源

- [正式会议记录](https://proceedings.mlr.press/v235/zhou24r.html)
- [会议最终全文及附录](https://raw.githubusercontent.com/mlresearch/v235/main/assets/zhou24r/zhou24r.pdf)：§3–4方法，§5实验，表2–13、附录A–G。
- [arXiv版本页](https://arxiv.org/abs/2310.04406)
- [官方实现固定提交](https://github.com/andyz245/LanguageAgentTreeSearch/tree/853d81614607dd27433faf17c7b0a7d660f95d22)
- [官方README：运行与日志范围](https://github.com/andyz245/LanguageAgentTreeSearch/blob/853d81614607dd27433faf17c7b0a7d660f95d22/README.md)
