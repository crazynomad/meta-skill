# meta-skill

对 AI agent skill 本身做工程化的研究与工具:分类体系、测评框架、健康审计。

Skill 已经是开放标准(SKILL.md,30+ 平台采纳),但生态缺一套统一的分类与测评语言。
这个仓库从「prompt 是生产系统」的判断出发,探索 skill 的全生命周期工程化——
特别是业界标杆尚未覆盖的「改进段」:遥测 → 学习 → 复盘 → 蒸馏 → 人工裁决。

## 目录

```
meta-skill/
├── research/    # 研究与构思:设计文档、调研笔记、框架草案
├── audits/      # 框架实测:对真实 skill 库的审计报告
└── README.md
```

## 当前内容

- [`research/skill-taxonomy-and-eval-framework.md`](research/skill-taxonomy-and-eval-framework.md)
  — Skill 分类与测评体系设计 v1.2:五轴分类(本体类型 / 判断密度 / 爆炸半径 /
  触发方式 / 来源 provenance)+ 五层测评金字塔(静态校验 → 触发精度 → 行为合规
  → 端到端 → 生产遥测)+ A/B 双频 + 退役协议 + 审计协议。每个版本由实测
  与新标杆对照驱动(v1.1: skill-creator 2.0 + gstack 实测;v1.2: 官方库实测 + SkillOpt)。
- [`research/gstack-case-study.md`](research/gstack-case-study.md)
  — gstack 案例研究(git 考古):skill 演化三阶段、四层反馈回路、preamble 编译期
  组合机制、五个标杆级实践、skill 版 SDLC 映射表。框架的主要经验来源。
- [`research/industry-landscape.md`](research/industry-landscape.md)
  — 业界图景(2026-06):Anthropic skill-creator 2.0 / superpowers / Codex /
  PromptOps / DSPy / Voyager 各占生命周期哪一段;"改进段"空白;
  可证伪性收敛信号。
- [`audits/gstack-2026-06.md`](audits/gstack-2026-06.md)
  — 框架首次实测:gstack 55 个 skill 全量分类,四类冠军(ship / codex /
  guard / retro)五层评测。实测发现回灌为框架 v1.1 的四条修订。
- [`audits/anthropic-skills-2026-06.md`](audits/anthropic-skills-2026-06.md)
  — 第二次实测:Anthropic 官方 skills 库。结论:分发制品库而非生产系统,
  单 skill 工艺天花板(claude-api 路由算法式 description)与系统工程缺位
  (零 CI / 零遥测)并存,与 gstack 恰好互补。产出 5 条 v1.2 候选修订。

## 可能的方向

- `/skill-audit` 元 skill:对照分类矩阵给任意 skill 出健康报告
- L0 静态校验器 + L1 触发精度测试集的参考实现
- A/B 区分度测评:回答「这个 skill 该不该存在」
