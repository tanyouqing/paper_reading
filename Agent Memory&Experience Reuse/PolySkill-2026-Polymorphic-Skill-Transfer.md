---
title: "PolySkill：分离抽象目标与网站实现，让程序技能跨网站复用"
aliases:
  - PolySkill
  - PolySkill Learning Generalizable Skills Through Polymorphic Abstraction For Continual Learning
tags:
  - paper-reading
  - agent/skills
  - agent/memory
  - agent/continual-learning
  - evaluation/transfer
year: 2026
venue: ICLR
track: 主会
status: 精读与代码核对
cssclasses:
  - paper-note
updated: 2026-11-01
---

[[00-Agent-Memory-Experience-Reuse|← 论文索引]] · [[NSI-2026-Logic-Grounded-Skill-Induction|对照 NSI]] · [[BudgetWeb-2026-Online-Memory-Token-Audit|对照成本审计]]

# PolySkill：分离抽象目标与网站实现，让程序技能跨网站复用

> [!abstract] 作者做了什么？
> **从成功网页轨迹生成 Python 技能时，先把共同功能登记到领域抽象类，再在网站子类中实现；上层组合技能只调用共同接口，换网站后复用组合逻辑、适配底层实现。**
> 输入：任务、网页观察、成功轨迹、领域及已有技能。输出：抽象接口、网站实现、组合技能库和新任务的浏览器动作。改进对象是外部程序库与归纳提示；没有训练基础LLM参数。

> [!info] 阅读与核对范围
> 已阅读ICLR正式版29页正文、算法1/2、附录A–G及关键任务/提示词；核对官方代码版本 `fff8807d7501d93188f9f658f4d0af2f29f35c23`。查看了正文Table1图像和附录Table8原版，避免只依赖PDF文本解析。
> 代码是作者后来用公共依赖重写的clean-room re-release，不等于原实验实现。未重跑浏览器/付费模型实验；只做静态审计、算术复核及元数据序列化的小验证。

## 1. 论文信息与选择理由

- **正式题目**：PolySkill: Learning Generalizable Skills Through Polymorphic Abstraction For Continual Learning。
- **作者**：Simon Yu、Gang Li、Weiyan Shi、Peng Qi；后两位共同指导。
- **机构**：Northeastern University、Uniphore。
- **发表**：**ICLR 2026 主会**，正式会议页面与PDF均确认；不是Workshop。预印本于2025年发布，正式年份记2026。
- **原文**：[官方会议记录](https://proceedings.iclr.cc/paper_files/paper/2026/hash/e350897ed832f9cad6f2e223e7acad6d-Abstract-Conference.html) · [正式29页全文](https://proceedings.iclr.cc/paper_files/paper/2026/file/e350897ed832f9cad6f2e223e7acad6d-Paper-Conference.pdf)。
- **官方实现/说明**：[simonucl/PolySkill](https://github.com/simonucl/PolySkill)。

**为什么此时读**：已读工作往往讨论轨迹到经验，当前问题进一步收敛为“同类经验在新目标/前提下怎样可靠迁移，验证是否值得”。本文直接尝试通过技能表示改善跨网站迁移，适合作为方法近邻；它没有完成预算约束的迁移可靠性验证。已核对最新仓库README、两个主题的已读导航、本主题索引和可访问历史，未发现本文已有精读。

## 2. 出发点：变量化代码仍然可能只适用于旧网站

AWM的主方法是操作指导；ASI把成功子流程封装为Python函数；SkillWeaver从探索中合成网站工具。这些方法可以减少重复推理，但函数内部仍可能绑定某个网站的页面路径和元素定位规则。

例如同样叫“搜索商品”，网站A用顶部输入框和回车，网站B先打开浮层再填写。只替换商品名，不足以把A的操作代码搬到B。

**作者发现的问题**：熟悉网站上的成功和技能数量，不说明所学结构能在同类新网站上发挥作用。论文Figure2用ASI/SkillWeaver的学习曲线及低复用率诊断过度专门化；这是特定比较设置，不能推广成所有程序技能都没有泛化。

### 贯穿教学例子（并非论文真实评测记录）

目标是在A、B两个购物网站搜索咖啡机并加入购物车。

- 两站共同的目标结构：搜索商品 → 选择目标商品 → 加入购物车。
- 两站不同的部分：搜索入口、结果页表示、商品标识、购物车按钮和中间页面。

若只保存A的完整程序，B需要重写整段。PolySkill尝试只重写B的底层操作，上层组合保持同一份。

### 为什么走向“多态”

多态（polymorphism）在这里指：**同一个接口，在不同网站对象上执行不同实现。**它是成熟的软件工程概念，作者将其用于LLM从轨迹归纳和组织技能，不是发明Python继承。

> [!note] 有依据的设计逻辑重建
> 单网站成功代码在新网站失效 → 区分共同目标与页面实现 → 先建立共同方法签名 → 各网站维护自己的实现 → 上层组合只依赖共同签名 → 用这些签名指导新网站探索。
> 这是根据引言、§3.2–3.3和方法重建的研究逻辑，不冒充作者未公开心理过程。

## 3. 技能长什么样：不是只保存一段文字

下面是**为讲清调用关系改写的教学Python结构，不是论文原始代码或可直接运行程序**。`Product`是抽象的返回对象占位，论文示意不完整定义这个类型。

```python
class ShoppingSite:
    def search_product(self, query):
        raise NotImplementedError

    def add_to_cart(self, product):
        raise NotImplementedError

    def search_and_add(self, query):
        product = self.search_product(query)
        return self.add_to_cart(product)

class SiteA(ShoppingSite):
    def search_product(self, query):
        # A站：定位顶部搜索框、输入、读取当前结果
        ...

    def add_to_cart(self, product):
        # A站：定位当前商品的购物车操作
        ...

class SiteB(ShoppingSite):
    def search_product(self, query):
        # B站：打开搜索浮层，按B站结构返回结果
        ...

    def add_to_cart(self, product):
        # B站：执行B站的购物车操作
        ...
```

调用 `SiteB().search_and_add(query)` 时：父类组合方法 → B的search实现 → B的add实现。无须重新写一份B专用的组合方法。

论文Table1采用`AbstractShoppingSite`，定义`search_product`、`add_to_cart`、`checkout`等，再用`find_and_add_to_cart`、`purchase_item`组合它们。右侧网站实现包含click/fill/keyboard等操作。它是示意结构，未完整定义元素绑定、返回对象和所有依赖，不直接当成可执行复现代码。

**真正被复用的是接口和组合结构，不是已经证明跨站通用的底层页面程序。**新网站仍需要具备有效实现。方法同名也不证明语义一致：例如search是返回列表、打开第一项还是停留结果页，都会影响后续组合。

## 4. 任务执行与系统学习：分成两个闭环

### 4.1 执行当前任务

论文§3.1的工作记忆M包含目标和本题观察/动作历史；长期技能库K扩展行动空间：

$$
A_t=A_p\cup K_t.
$$

$A_p$为点击、填写、滚动、导航等原子动作；$K_t$为当前可用技能。模型结合目标、观察和历史，选择原子动作或调用技能。技能内部也能调用其他已学技能。

这不是任意多agent拓扑搜索。父子类关系是技能组织结构，方法之间是执行依赖；论文没有训练通信连边或角色分配策略。

### 4.2 成功经历怎样变成技能

按§3.3的描述：

1. 先用原子动作和已有技能完成任务，保存观察—动作轨迹。
2. LLM评价器判断这次经历是否成功。
3. 第一次进入购物类网站时，归纳领域抽象类；以后扩充其方法签名。
4. 从成功子流程中提出可复用操作，先登记抽象方法，再生成网站子类中的具体实现。
5. 用新生成技能重新尝试同一个任务；论文继承ASI的回合验证流程，只有成功才应保留。
6. 新任务读取更新后的技能库。

归纳提示要求参数化实例字段、控制技能复杂度，并重写轨迹为技能调用。附录E.5的基础提示既有“至少三动作”等表述，也有“2–5步”的要求，约束不完全一致；不能说它给出了严格统一的程序搜索空间。

**反馈更新的是外部Python程序、接口及库内容。**归纳器、评价器和执行模型不因这一步更新权重。Algorithm1把新技能验证的细节藏在InduceSkill里，伪代码没有列出完整重执行和回滚，讲解时需结合文字描述。

### 4.3 新网站到底学了什么

在A站已学出购物接口，来到B站：检索/读取领域蓝图，知道值得探索搜索、购物车等功能；通过实际完成任务获得B站成功经历，再生成B的实现。

因此存在两种不同的迁移：

- 复用“应该有哪些功能”和上层组合逻辑。
- 通过新经历适配这些功能在B上的实现。

**带在线更新的成绩不是完全零适配迁移。**成功适配过B的早期任务后，后续B任务可以利用这些经历。复现必须记录任务顺序、初始库和何时回写。

### 4.4 自提出任务的探索

Algorithm2将预设任务换成模型提出的目标。根据当前观察和技能库选择网站/目标→执行→评价→归纳→更新→继续。

购物实验比较：单站50轮；预设顺序总150轮；三站自引导探索150轮。开发平台实验总100轮。接口是探索线索，让系统补齐功能，而不是只随机浏览。

这是外部经验学习和课程构建，没有RL参数训练；也未证明归纳器/评价器本身会递归改进。

### 4.5 成本公式不是实际RL损失

论文写出目标：

$$
\max_{\pi_L,K}\;\mathbb E_{q\sim Q}[g(\tau,q)-\gamma|\tau|].
$$

$q$是任务，$\tau$是执行轨迹，$g$表示是否完成目标，$\gamma$惩罚轨迹长度。

作者明确说使用该效率原则指导提示，**没有把它实现为模型参数训练的loss**。高层技能调用算一步，包含多个原子动作，故$|\tau|$下降不代表总token、原子操作或延迟必然下降。

## 5. 记忆、检索和验证的边界

- **短期**：当前任务、观察与动作历史。
- **长期**：领域接口、网站实现、组合技能；论文不是G-Memory式轨迹图/抽象经验图数据库。
- **生成与筛选**：成功轨迹驱动代码归纳，模型判断与回合重试验证。
- **使用**：模型选择可用技能，程序执行实际动作；不是仅返回“应该怎么做”的文本经验。
- **更新**：新增接口/网站实现及组合；没有可靠性置信度校准或独立新变体的验收协议。
- **共享**：多网站复用同一领域接口。多agent技能共享是未来工作，不是已经验证的跨agent经验传播。

论文没有给出足以完整复算的语义检索算法、全部接口版本合并和失效撤回策略，不自行补出向量检索、前置条件证明或严格行为子类型检查。

## 6. Benchmark：输入输出与成功判据

| 基准 | 任务 | 论文采用的评测 |
| --- | --- | --- |
| Mind2Web | 订票、购物、过滤等自然语言网页任务；来自137网站、31领域 | 使用任务指令在网页执行，GPT-4.1 WebJudge判整任务成功；非原始离线下一步匹配指标 |
| WebArena | 本地部署购物、后台、论坛、GitLab、地图和跨站任务 | 环境功能正确性评分；模型评价器另外用于技能归纳筛选 |
| 扩展真实网站任务 | Amazon50、Target50；GitHub章节标题称75 | 研究跨站适配、保持旧站能力及自探索；完整任务/次数/划分复现材料不足 |

Mind2Web表6/7列：轨迹归纳阶段1009任务；测试cross-task252、cross-website177、cross-domain912。这里“training”是建外部技能库，非LLM微调。

Mind2Web论文采用模型评价并引用原WebJudge约85%人类一致率；**不是本文对自身所有模型和网站重新获得85%准确率**，也不等同于环境真值。归纳与评测同用模型评价会带来相关偏差，需要独立人工/环境审计。

WebArena表8任务数：187+182+106+180+109+48=812；附录E.3写811，存在文字冲突，以具体表为记录。Mind2Web和WebArena成绩不能与AWM早期不同模型/评测协议的分数直接纵向比较。

模型：GPT-4.1、Claude-3.7-Sonnet、Qwen3-Coder-480B-A35B、GLM-4.5。基线：无技能、ASI、SkillWeaver；SkillWeaver主要在WebArena/探索比较，没有完整Mind2Web对照。图3称三次运行标准误；其他表中±的口径未全部明示，不将其统一当成95%置信区间。

## 7. 指标：复用频率不是迁移可靠性

附录E.1定义：

1. **SR**：测试任务完成比例。
2. **Steps**：只对成功轨迹平均，每次技能调用算一步；有成功样本筛选和动作打包效应。
3. **Skill Reusability**：最终库中至少在测试轨迹里被调用过一次的技能数/库技能总数。
4. **Task Coverage / Skill Adoption Rate**：调用过至少一个技能的测试轨迹数/全部测试轨迹数。
5. **Compositionality**：每个新技能平均引用多少先前已学技能。

> [!warning] 这些不是同一件事
> 技能被调用、技能自身成功、整任务成功、跨站新变体可靠，是不同指标。Reusability只统计调用，并未要求该调用造成任务成功；接口共享也没有直接证明新实现满足原语义契约。

教学示例：库中10个技能，3个在测试中用过，Reusability=30%；100个任务中20个调用过技能，Coverage=20%。不能从这两个数推出“迁移成功率30%”或“成功概率20%”。

论文关于复用比例的叙述有明显冲突：引言称31%对低于18%；§3.2又举低于9%/3%；结论写73%对低于31%。没有同口径汇总表统一解释，因此本笔记**不采用一个笼统的73%迁移率**，也不把摘要1.7倍解释成成功率1.7倍。

## 8. 关键结果：哪些证据最有解释力

### 8.1 Mind2Web：先看静态，再看在线

正式表2/表6，GPT-4.1：

| 方法 | Cross-task | Cross-website | Cross-domain |
| --- | ---: | ---: | ---: |
| 无技能 | 53.8 | 56.2 | 62.3 |
| ASI静态，全部技能 | 52.3 | 54.9 | 57.3 |
| PolySkill静态，全部技能 | 55.4 | 57.6 | 60.1 |
| ASI+Update | 59.4 | 58.7 | 62.1 |
| PolySkill+Update | 63.2 | 61.3 | 63.4 |

**支持的结论**：PolySkill静态优于ASI静态；在线适配进一步提高表现。

**重要反例**：静态PolySkill在跨领域仍低于无技能62.3（60.1，-2.2个百分点）；在线后63.4只高1.1个百分点。不能说抽象接口让跨域问题自动解决。

摘要的“+9.4%”对应cross-task从53.8到63.2，实为9.4个百分点，且是对无技能，不是对ASI+Update的9.4个百分点；匹配在线设置的提升是3.8个百分点。

表6印刷的若干括号增量错误：GPT跨域PolySkill60.1标-0.1，实际相对62.3是-2.2；ASI+Update62.1标+1.9，实际-0.2；PolySkill+Update63.4标+3.2，实际+1.1。记录原始值并重算，不沿用错误标注。

技能库规模：GPT cross-task静态PolySkill43对ASI50；在线47对66（表2）。但同一抽象技能的各站实现不分别计入作者的技能数，不能据此直接宣布总代码量/存储一定更少。

开源Qwen表7也有静态迁移反例：cross-domain无技能37.5，PolySkill35.9，在线后39.9。在线方差较大，不能把每个小差距宣布显著。

### 8.2 WebArena：可用局部值，谨慎使用总平均

表8，GPT-4.1 Shopping：无技能37.4、ASI46.3、PolySkill51.4；GitLab分别38.9、46.8、54.2。体现部分任务的技能表示收益。

但不是每域提高：GPT Reddit73.2低于ASI73.7，Map18.9低于ASI21.5。作者称总体更好，不等于每种任务适合抽象技能。

**平均值校验问题**：根据表8各域任务数复算，GPT PolySkill49.34约等于表列49.3，符合任务加权；但Claude PolySkill复算57.20，表列59.5；Qwen基线复算28.26，表列34.4；Qwen PolySkill复算42.49，表列45.2。普通域均值也不统一解释。缺少原始结果和汇总规则，**不使用这些冲突平均值构造总体优势结论**。已查看PDF原表，非仅文本解析问题。

### 8.3 自探索和顺序效应

表3购物域：自引导探索150轮后，WA Shopping43.1、Amazon66.7、Target75.2；选定的SkillWeaver顺序基线为39.8、64.4、74.2。

不同固定顺序各有长处：WA→Target→Amazon在Amazon70.5/Target76.8高于自引导；因此自引导不是所有站点逐项最好。单站50轮与总150轮预算不同，不把它们的差异全归因于结构。

图5重复三次跟踪WA→Amazon→Target，作者报告适配新站后旧站表现较稳、末尾对ASI约4.9个百分点优势。它支持此课程下减轻干扰，不证明任意领域都无遗忘。

表9开发平台自探索100轮，GitLab66.2、GitHub84.0；固定课程最优单项为GitLab62.1、GitHub80.1。多站探索与抽象层、任务选择、预算分配有耦合；缺少充分正交消融来识别各因素的独立贡献。

## 9. 成本：执行阶段省，当前总量仍增加

正式表10，GPT-4.1、Mind2Web cross-task，每任务平均输入+输出token：

| 方法 | 动作生成 | 验证 | 技能归纳 | 合计 |
| --- | ---: | ---: | ---: | ---: |
| 无技能 | 55,367.4 | — | — | 55,367.4 |
| ASI | 53,840.5 | 4,673.8 | 5,773.2 | 64,287.5 |
| PolySkill | 49,710.3 | 4,963.2 | 8,476.4 | 63,149.9 |

PolySkill动作生成省约10.2%，但合计比无技能**多约14.1%**，比ASI总量略低约1.8%。它没有给普通方法更多交互以匹配总token；也没有提供所有方法统一预算的SR/token曲线。

作者称验证/归纳费用可随部署摊销。这需要环境稳定、实现可复用、后续调用节省持续存在，不能把首轮总量增加写成已经实证长期省钱。

> [!example] 只解释摊销逻辑的假设，不是论文测得回本轮数
> 若一次技能构建成本为C，每次部署节省Δ，至少需NΔ>C才回本。实际还应计入检索/上下文、适配新站、独立验证、错误复用和修复；同接口不代表这些成本为零。

没有报告完整美元账单、GPU小时或各阶段端到端延迟。技能步数下降也不直接代表这些资源下降。

## 10. 公开代码：方法可读，完整复现仍有缺口

固定版本：[fff8807](https://github.com/simonucl/PolySkill/tree/fff8807d7501d93188f9f658f4d0af2f29f35c23)。README明确说明原实现依赖内部基础设施，2026-07重写为BrowserGym+litellm版本。因此应区分原实验行为与当前开源路径。

### 10.1 已核对的实现

- `core/inducers/polymorphic_inducer.py`构造抽象优先提示，AST解析无具体抽象父类的类和其网站子类，生成CODE类型Skill。
- `prompts/induction/polymorphic.py`预置领域命名、抽象接口/组合示例；购物/开发类别映射有人提供，不是自动发现任意领域边界。
- `core/online_hook.py`先评价原轨迹，再调用归纳器；`use_polymorphism:true`选多态归纳。
- `core/skill_storage.py`提供JSON索引、代码文件、版本备份、过滤/检索；任务相似过滤是至少两个词重叠，非向量图检索。
- 规划器和执行器构成既定层次结构；这不是自动优化多agent协作图。

### 10.2 足以影响主张对齐的缺口（不能忽略）

1. **原任务重执行验证未接通到已核对多态hook**：该路径评价旧轨迹→归纳→返回Skill，没有调用浏览器重置后执行新技能的验证回合。另有validator模块只做语法/动作交集类检查，也不能当作真实重执行。
2. **归纳返回与持久化/使用断开**：`induce()`返回Skill，不自行store；公开`_run_and_induce`及调用者统计新技能、重新读取库，未看到对该多态Skill的store调用。`get_current_skills`默认取得SimpleSkillInducer，会替换缓存类型；已有接口签名也没有由该hook传入abstract_methods。
3. **技能上下文/运行接线不足**：runner把current_skills放进配置，但`make_agent`未传该字段，构建的是默认启发式SimpleSkillInducer包装器；默认action_file_path为空。不能仅凭“Available skills injected”的日志文字证明下一任务真正执行了新多态技能。
4. **抽象源码序列化丢失**：多态归纳把abstract_source等挂到`metadata.extra`，但`SkillMetadata.to_dict()`不导出extra。本次直接加载该标准库数据类做小检查，确认extra在序列化时被省略；保存/重载不能据此保留完整抽象层。
5. **文件名与网站隔离**：代码内容文件按方法name而非含网站的id命名，同名跨站实现可能指向同一个`code/<name>.py`；导出直接拼接具体类源码，抽象父类依赖也需处理。动作加载器筛选顶层函数，不能直接注册类中的方法。
6. **自探索实现不同**：当前`explore_loop.py`使用SimpleSkillInducer，不是完整多态接口驱动的同一归纳路径。不能把自探索入口存在等同于论文机制已完整复现。

这些是固定公开版本的静态证据和一项小检查，**不证明原实验根本没有执行多态技能**；说明当前发布版本需要补齐验证、持久化、接口管理和动作注册，不能声称一键复现主表。

### 10.3 数据与资源

- 代码、示例、提示和部分任务元数据开放；WebArena需要另外部署网站服务，模型API需自备。
- 当前Mind2Web公开包为177/142/694题，来自Multimodal-Mind2Web子集；论文252/177/912。起始URL是按网站名称尽力推断，实时网页还会变动，不可直接对齐正式表分数。
- 论文报告开源模型用**8张H100集群**，闭源模型官方API。没有模型训练；自托管480B级MoE推理仍重，可用API学习机制，但成本未知。
- 未报告足以复算的总GPU小时、全部美元成本、逐模型温度/上下文/并发配置及原始全量轨迹。仓库默认30步等是新实现配置，不当作原实验已确认一致。

## 11. 创新与已读工作的区别

| 工作 | 与PolySkill的区别 |
| --- | --- |
| [[AWM-2025-Workflow-Memory|AWM]] | 主方法为上下文指南；PolySkill用接口/子类组织可执行代码 |
| [[NSI-2026-Logic-Grounded-Skill-Induction|NSI]] | NSI强调技能内部状态分支和恢复嫁接；PolySkill强调跨网站共享接口和上层组合 |
| [[G-Memory-2025-Hierarchical-Memory|G-Memory]] | 记忆图组织案例；这里的层次是程序接口和实现依赖 |
| [[AFlow-2025-Workflow-Optimization|AFlow]] / [[ADAS-2025-Automated-Agentic-System-Design|ADAS]] | 搜索完整任务系统；PolySkill主要积累可复用子能力 |
| [[MaAS-2025-Cost-Aware-Agentic-Supernet|MaAS]] | 训练按题路由器；PolySkill没有控制器参数RL |
| [[BudgetWeb-2026-Online-Memory-Token-Audit|BudgetWeb]] | 提供普通执行器预算对照；PolySkill成本表没有完成这类公平预算比较 |

最有辨识度的贡献：**把网站变化隔离到实现层，用抽象接口组织经验归纳、组合和探索，并增加调用/覆盖/组合指标来检查技能如何被用。**它不是多态本身的新发明，不是形式验证，也不是新的参数RL。

方法有清楚的可理解机制，但当前证据存在数值和实现缺口。最值得借鉴的是问题分解，不是直接接受其所有“泛化已解决”的表述。

## 12. 与研究方向的关系（简短）

| 关注点 | 对应 |
| --- | --- |
| 经验优化Harness | 直接：轨迹驱动接口/程序库更新 |
| 长短期记忆与检索 | 部分直接：本题历史、长期技能库；不含完整事实/策略双源知识图 |
| 单agent执行 | 直接：层次规划/执行与技能调用 |
| 多agent运行图 | 未做拓扑、通信、角色优化；共享接口可迁移为团队模块设计 |
| RL/参数蒸馏 | 未覆盖；论文列为未来方向 |
| 迁移验证与成本 | 直接的近邻问题，但未估计新变体可靠性或学习何时验证/拒绝复用 |

**一个小实验建议，非论文结论**：在两个可控网页环境提供同一购物接口，固定模型/原子动作/经验和总token，比较独立脚本、共享接口、共享接口+独立语义契约验证。先只改变页面布局，再改变“加入购物车”等接口的前置条件/返回语义。用未参与适配的变体测试SR、技能独立成功率、错误接受率、回归率、总token和净节省。要回答：共享接口的优势究竟来自减少重写，还是它真能识别不安全迁移？

## 13. 三个研究问题与阅读路线

1. 相同方法名/参数下，怎样检查各网站实现的前后条件足够一致，允许上层组合复用？
2. 什么时候只适配底层实现，什么时候必须修改抽象接口或原组合流程？
3. 固定完整预算后，接口复用、增加直接操作和独立迁移验证，哪一项更值得花token？

先读Table1与§3.3，搞清谁继承谁、新站还要学什么；再读Algorithm1/2区分执行和归纳；然后看Table6原始分数和Table10总成本；接着读E.1指标定义；最后核对代码重写说明、持久化/验证路径和Table8汇总问题。不要先记摘要的“73%”“1.7倍”或总平均。

## 原始来源与代码入口

1. [ICLR官方记录](https://proceedings.iclr.cc/paper_files/paper/2026/hash/e350897ed832f9cad6f2e223e7acad6d-Abstract-Conference.html)。
2. [正式论文29页](https://proceedings.iclr.cc/paper_files/paper/2026/file/e350897ed832f9cad6f2e223e7acad6d-Paper-Conference.pdf)：§3、Table1/2/3、附录E/F、Table6/7/8/10、Algorithm2。
3. [官方仓库固定版本与重写声明](https://github.com/simonucl/PolySkill/tree/fff8807d7501d93188f9f658f4d0af2f29f35c23)。
4. [多态归纳器](https://github.com/simonucl/PolySkill/blob/fff8807d7501d93188f9f658f4d0af2f29f35c23/polyskill/core/inducers/polymorphic_inducer.py) · [归纳提示](https://github.com/simonucl/PolySkill/blob/fff8807d7501d93188f9f658f4d0af2f29f35c23/polyskill/prompts/induction/polymorphic.py)。
5. [在线hook](https://github.com/simonucl/PolySkill/blob/fff8807d7501d93188f9f658f4d0af2f29f35c23/polyskill/core/online_hook.py) · [调度与回写调用者](https://github.com/simonucl/PolySkill/blob/fff8807d7501d93188f9f658f4d0af2f29f35c23/polyskill/experiments/run_eval_with_skill_induction.py)。
6. [存储](https://github.com/simonucl/PolySkill/blob/fff8807d7501d93188f9f658f4d0af2f29f35c23/polyskill/core/skill_storage.py) · [元数据序列化](https://github.com/simonucl/PolySkill/blob/fff8807d7501d93188f9f658f4d0af2f29f35c23/polyskill/core/base/skill_types.py)。
7. [动作加载](https://github.com/simonucl/PolySkill/blob/fff8807d7501d93188f9f658f4d0af2f29f35c23/polyskill/agents/agent/asi_utils/utils.py) · [agent工厂](https://github.com/simonucl/PolySkill/blob/fff8807d7501d93188f9f658f4d0af2f29f35c23/polyskill/agents/__init__.py)。
8. [Mind2Web说明与数据差异](https://github.com/simonucl/PolySkill/blob/fff8807d7501d93188f9f658f4d0af2f29f35c23/docs/MIND2WEB.md)。
