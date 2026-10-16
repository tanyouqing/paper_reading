---
title: 论文阅读与研究资料
tags:
  - paper-reading/index
cssclasses:
  - paper-note
  - paper-index
---

# 论文阅读与研究资料

> [!abstract] 从论文到可复用的研究理解
> 关注智能体记忆、经验复用、多智能体协作与工作流优化。论文精读和导入观点分别保存，通过索引相互关联。

## 从这里开始

- [[00-Agent-Memory-Experience-Reuse|智能体记忆与经验复用 · 论文索引]]：论文分类、核心总结与比较。
- [[AFlow-2025-Workflow-Optimization|AFlow]]：从验证反馈出发，修改代码形式的智能体工作流。
- [[G-Memory-2025-Hierarchical-Memory|G-Memory]]：组织历史任务与抽象经验，为新任务提供记忆。
- [[Agent Workflow Memory|AWM]]：ICML 2025，从历史轨迹提炼工作流记忆；含在线/离线机制、两类网页评测与代码核对。
- [[GPTSwarm-2024-Optimizable-Agent-Graphs|GPTSwarm]]：ICML 2024，用任务奖励学习多智能体连边概率；区分外部策略 RL、节点示例和模型训练。
- [[XHS Notes/知识/Agent Memory 些许感悟|Agent Memory 些许感悟]]：已有导入随笔与配图。

## GitHub 浏览入口

GitHub 不原生解析 Obsidian 双向链接，可从下列标准链接进入：

- [论文索引](Agent%20Memory%26Experience%20Reuse/00-Agent-Memory-Experience-Reuse.md)
- [AFlow 精读](Agent%20Memory%26Experience%20Reuse/AFlow-2025-Workflow-Optimization.md)
- [G-Memory 精读](Agent%20Memory%26Experience%20Reuse/G-Memory-2025-Hierarchical-Memory.md)
- [AWM 精读](Agent%20Memory%26Experience%20Reuse/Agent%20Workflow%20Memory.md)
- [GPTSwarm 精读](Agent%20Memory%26Experience%20Reuse/GPTSwarm-2024-Optimizable-Agent-Graphs.md)
- [Agent Memory 导入随笔](XHS%20Notes/知识/Agent%20Memory%20些许感悟.md)

## 笔记组织与增量维护

论文笔记位于 `Agent Memory&Experience Reuse/`；导入资料位于 `XHS Notes/`，媒体仍使用原有相对路径。

每篇精读重点说明出发点、创新、架构、方法、输入输出、benchmark、评测与局限。论文事实、代码行为、教学例子和研究逻辑重建分开标识；不把推测写成作者结论。

新增论文时先检查仓库当前版本与索引，再新增独立笔记并更新主题索引。修改已有笔记保留用户内容与原始来源，注明重要更正。四篇论文均已完成精读整理；原文通过各笔记的官方链接访问。

## 内容维护规范

- 每篇论文保留独立 Markdown；文件名稳定，改标题时优先使用 aliases，避免破坏链接。
- YAML 统一记录 title、tags、year、venue、track、status、cssclasses 和 updated。
- 完整笔记包括动机、输入输出、架构、方法、评测、局限、复现入口与原始来源。
- 研究逻辑重建与作者明示动机分开；代码行为注明核对范围，数字冲突如实记录。
- 新增或修订论文同步更新主题索引；PDF 默认使用官方在线链接。
- 模板位于 [[Templates/Paper-Reading-Template|论文精读模板]]；主题索引仅统计实际完成的论文。
- 不提交 API 密钥、运行日志、临时下载或个人认证信息；不批量覆盖原有随笔与插件配置。

## Obsidian 使用

仓库根目录已有 `.obsidian`，可将本仓库作为 vault 打开。主页为本文件，主题入口为论文索引。无需 Dataview 等插件。

可选样式已放在 `.obsidian/snippets/agent-paper-notes.css`：设置 → 外观 → CSS 代码片段，刷新后启用 `agent-paper-notes`。样式只作用于带 `cssclasses: paper-note` 的笔记，未自动修改现有外观设置。

## 同步约定

在本地仓库执行 `git pull --ff-only` 获取远程更新；编辑后提交并推送自己的修改。若本地有未提交变更或双方修改冲突，先处理本地改动，再同步，不强制覆盖。

已移除初始欢迎页和 AWM 的本地 PDF，使用官方论文链接；已有随笔、图片和插件保留。此仓库当前为公开仓库。
