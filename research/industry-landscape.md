# Skill 工程化业界图景(2026-06 调研)

> 研究笔记 · 2026-06 · 来源全文存于 NotebookLM notebook
> `b067d181-6f89-410c-b2be-7fb1c42bfcfd`
> 2026-06-12 更新:补充微软 SkillOpt,修正「改进段空白」判断

## 背景事实

- SKILL.md 起源于 Anthropic 内部格式,**2025-12 开放为标准**,2026 年初已被
  30+ 平台采纳(Codex CLI、Claude Code、Gemini CLI、Copilot、Cursor、VS Code、
  Roo Code、Goose 等)
- Claude Code 插件市场 2026 年初公测,2026-02 已有 9000+ 插件
- 社区维护的跨平台 skill 库已达 1200+ 条目

## 标杆映射:各家占住生命周期的哪一段

| 生命周期段 | 标杆 | 核心能力 |
|-----------|------|---------|
| 开发 + 测试 | **Anthropic skill-creator 2.0**(2026-03) | 内置 eval 编写;多 agent 并行评测(干净上下文、独立 token/耗时指标);benchmark 模式;eval 可接 CI。官方最佳实践:**先写 eval 再写文档**;Claude A 设计 / Claude B 实测双实例工作法 |
| 过程方法论 | **obra/superpowers**(Jesse Vincent,~93k star,生态最流行插件) | 把工程方法论写成可组合 skill:严格红绿 TDD(**代码写在失败测试之前就删掉**)、头脑风暴→spec→subagent 开发→内置 review、用 skill 写 skill |
| 分发部署 | **OpenAI Codex** 双系统 skill | `$skill-creator` 脚手架 + `$skill-installer` 管发现/安装/更新/卸载——平台内置的 skill 包管理器;openai/skills 官方目录 |
| 测试 + 监控(通用基建) | **PromptOps 工具链** | promptfoo(CLI、YAML 进仓库、CI 门禁、红队);Braintrust(生产流量持续评测 + trace);LangSmith;DeepEval/RAGAS。已有 "PromptOps" 范式名词:prompt 是运营资产,需要与代码同级的版本/测试/监控 |
| 编译优化 | **DSPy**(斯坦福,学术)→ **SkillOpt**(微软,2026-06,生产级) | SkillOpt:把 skill 文档当可训练参数——rollout 打分 → 优化器生成 add/delete/replace 编辑 → held-out 验证门控(不严格提升即拒绝)→ 文本空间学习率/epoch。6 benchmark × 7 模型 × 3 harness(直接对话/Codex/Claude Code)52 格全部最优或并列;GPT-5.5 上较无 skill 基线 +19~25 分;跨模型跨 harness 可迁移;产物为 300-2000 token 独立 skill.md,推理期零额外调用。5.7k stars,arXiv 2605.23904 |
| 思想源头 | **Voyager**(NVIDIA,2023) | skill library 概念的学术起点:Minecraft agent 自己写技能、自动验证、入库、检索。今天的 SKILL.md 生态是它的工程化后裔(自动验证 → human in the loop) |

## 关键空白(2026-06-12 修正)

原判断「改进段只有 gstack 一家」已被 **SkillOpt-Sleep** 打破:它从每日
会话 transcript 通过夜间离线 replay 固化 skill,同一套验证门控。
改进段现在是**双闸门哲学并存**:

| | gstack | SkillOpt-Sleep |
|---|---|---|
| 抽象生成 | AI 蒸馏提案 | 优化器模型生成编辑 |
| 固化闸门 | **人按 Y**("never autonomous") | **held-out 验证分**严格提升 |
| 防的是 | 错误抽象被固化 | 过拟合训练集 |
| 防不了 | 人的判断成为瓶颈 | benchmark 没测的品质被悄悄交易掉 |

仍然空白的:两种闸门的混合形态(指标初筛 + 人工终审),
以及优化器产出 skill 的可解释性工具(每条编辑只有 score delta,
没有叙事性 why)。

## 行业收敛信号

三个独立来源指向同一结论 — **可证伪性是 skill 工程化的第一块基石**:

1. Anthropic 官方:"先写 eval 再写文档"
2. superpowers:"先有失败测试再有代码"(没有就删码)
3. Respan:"没有 eval 的 prompt 版本管理只是 diff 记录"

其他一切(版本、市场、遥测)都建在它上面。

## 与 gstack 的关系

- gstack 的"种缺陷测试 + LLM judge"实践早于官方 skill-creator 2.0 同类功能
- superpowers 与 gstack 几乎正交:前者约束 agent 怎么干活(过程纪律),
  后者工程化 skill 资产本身(遥测、学习回路、体积预算)——理论上可叠加
- PromptOps 工具链 = gstack 用 bash + JSONL 手搓那套的商业化通用版

## 主要来源

- https://github.com/microsoft/SkillOpt
- https://arxiv.org/abs/2605.23904

- https://claude.com/blog/improving-skill-creator-test-measure-and-refine-agent-skills
- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- https://github.com/obra/superpowers
- https://developers.openai.com/codex/skills
- https://www.braintrust.dev/articles/best-prompt-evaluation-tools-2025
- https://dasroot.net/posts/2026/02/prompt-versioning-devops-ai-driven-operations/
- https://www.respan.ai/blog/prompt-versioning-iteration-loop
- https://www.datacamp.com/tutorial/promptfoo-tutorial
- 全部 12 篇见 NotebookLM notebook `b067d181-6f89-410c-b2be-7fb1c42bfcfd`
