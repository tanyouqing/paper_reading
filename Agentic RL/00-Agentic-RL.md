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
updated: 2026-10-17
---

# Agentic RL：论文地图与阅读队列

[[README|← 仓库主页]] · [[00-Agent-Memory-Experience-Reuse|对照记忆与经验复用]]

> [!abstract] 阅读目标
> 理解语言模型智能体如何规划、调用工具、处理长程反馈，以及如何通过强化学习改善策略。先分清推理时搜索、外部记忆更新、外部策略训练与 LLM 参数训练，再比较论文。
> 已完成 1/19 篇：RAP。下一篇：LATS。仅在完成实质精读后标记完成。

## 已完成论文

| 笔记 | 发表 | 类别 | 核心机制 | 改变什么 |
| --- | --- | --- | --- | --- |
| [[RAP-2023-Reasoning-via-Planning\|RAP]] | EMNLP 2023 主会 | 规划、世界模型、推理时搜索 | 冻结 LLM 提出动作并预测状态，MCTS 利用过程奖励选择分支 | 当前题搜索树的价值与访问记录；LLM 权重不变 |

[在 GitHub 阅读 RAP](RAP-2023-Reasoning-via-Planning.md)

## 按图片顺序的 19 篇队列

顺序为从左到右、每列从上到下。图片标题写 17 篇，实际方框共 19 个，以全部条目为准。简称需在各次阅读时通过原文唯一定位；待阅读项的年份、会议与身份尚未在本索引核验。图片箭头不作为论文继承关系证据。

| 顺序 | 图片分组 | 论文简称 | 状态 |
| --- | --- | --- | --- |
| 1 | Planning | [[RAP-2023-Reasoning-via-Planning\|RAP]] | 完成精读 |
| 2 | Planning | LATS | 待读，下一篇 |
| 3 | Planning | VOYAGER | 待读 |
| 4 | Planning | Planner-R1 | 待读 |
| 5 | Tool Use | ToolRL | 待读 |
| 6 | Tool Use | ARTIST | 待读 |
| 7 | Tool Use | ToRL | 待读 |
| 8 | Tool Use | GiGPO | 待读 |
| 9 | Tool Use | ARPO | 待读 |
| 10 | Memory | Memory-R1 | 待读 |
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
| 跨任务经验记忆 | [[G-Memory-2025-Hierarchical-Memory\|G-Memory]]、[[Agent Workflow Memory\|AWM]] | 经验内容、组织或示范 | 后续任务可用记忆 |
| LLM 参数 RL / 蒸馏 | 后续队列继续补充 | 模型权重 | 参数化能力，需单独验证泛化 |

RAP 中树节点表示任务状态，AFlow 的搜索节点表示工作流候选，GPTSwarm 操作节点执行 LLM/工具调用；这些图的层级与语义不同。

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
