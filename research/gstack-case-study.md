# gstack 案例研究:skill 工程化的全生命周期实践

> 研究笔记 · 2026-06 · 方法:对 gstack 仓库(github.com/garrytan/gstack)的 git 考古
> 仓库规模:~1950 个 PR,版本 0.x → 1.57.x,52 个 skill,基本由一人 + AI 构建

## 1. Skill 是磨出来的,不是固化出来的

`/office-hours` 从诞生到 v1.57 经历 **29 次模板修改**,`/ship` 是 **55 次**。
演化呈清晰的三阶段:

1. **初版抽象**(`6000af45`,v0.7.0)— 一次性写出基本流程,相当于"从成功流程提炼"
2. **被真实使用打脸后的修正** — 典型如 `dbd98aff`(v0.9.9.0),commit message 直接记录动机:
   *"Address user feedback that gstack compromises toward founder input"*
   → 反谄媚规则(禁用短语清单)+ 5 个 BAD/GOOD pushback 范例 + 带门槛的跳过出口
3. **抽象的深化与删除** — 深化:识别回头客(`dbd7aee5`)、提问前先读项目大脑(`070722ac`);
   删除:`5d4f7def` *"delete AskUserQuestion fallback (root cause of forever war)"*
   — 曾被认为合理的容错抽象被证明是反复出问题的根源,直接拔掉

**关键证据 — 抽象被当作假设,用使用数据证伪**(`81159512`):

> "Contributor mode never fired in 18 days of heavy use"

一个完整设计的功能因 18 天零触发被整体删除(连同 2 个 E2E 测试)。
这是"prompt 里的死代码检测":抽象合不合理不靠争论,靠触发率说话。

**方法论提炼**:commit message 风格本身就是方法——每次修改 skill 都记录
**触发它的具体失败**。抽象的 *why* 被持久化,这正是自动提炼 skill 时最容易丢失的东西。

## 2. 四层反馈回路(日志喂反思,人裁决抽象)

| 层 | 机制 | 关键设计 |
|----|------|---------|
| L1 无感日志 | `gstack-telemetry-log` → skill-usage.jsonl | `.pending` 开始标记捕捉崩溃运行;遥测脚本永不非零退出(no `set -e`);限速 5 分钟、隐私分级上报 |
| L2 AI 会话末自省 | `gstack-learnings-log` → learnings.jsonl | `source` 字段(observed/user-stated/inferred/cross-model);**只有 user-stated 才标 trusted**;写入前过注入检测;append-only + 读时"同 key 最新者胜" |
| L3 AI 批量蒸馏 | `gstack-distill-free-text`("dream cycle") | 攒用户自由文本批量发 Haiku 蒸馏成提案;**"Proposals require explicit user Y before applying — never autonomous"** |
| L4 人工复盘 | `/retro` skill | 开场读入 learnings + skill-usage + eureka.jsonl;人面对聚合数据做裁决,不是空对空回忆 |

另一条独立采集线:**AskUserQuestion 问答日志** — PostToolUse hook 确定性捕获
"AI 推荐了哪个、用户实际选了哪个"(分歧数据);`tune:` 偏好指令只认用户本人
当前聊天消息(防 profile-poisoning,工具输出里的 tune: 一律拒绝)。

**回路闭合**:learnings 注入 13 个 skill 的开场(显示 *"Prior learning applied: [key]
(confidence N/10)"* 让复利可见);问答偏好固化后 hook 直接 AUTO_DECIDE;
retro 产出变成模板修改,又被三层测试守住。

**一句话总结**:靠日志发现问题,靠 AI 把日志蒸馏成假设,靠人批准假设变成抽象。
人的角色从"记录流程"上移到"裁决抽象边界"。

## 3. Preamble:给没有运行时的系统造静态链接器

Claude 原生 skill 机制极薄:SKILL.md + frontmatter,触发时读入,没有共享 loader、
没有 middleware。gstack 发明了**编译期组合**:`.tmpl` 模板里放 `{{PREAMBLE}}` 宏,
`gen-skill-docs.ts` 构建时展开为共享协议(更新检查、遥测、learnings、Confusion
Protocol……),按 tier 分级注入。

- 类比:静态链接把 runtime 库打进每个二进制。代价是产物膨胀(→ 100KB 上限 +
  Token-reduction 重构波次),换来改一处 resolver、几十个 skill 同步更新
- 第二动机:**多宿主** — 同一模板为 Claude/Codex/Cursor 生成各自方言
- 与原生 hooks 的分工:跨 skill 一致行为要么编译期生成,要么 hooks 运行时拦截,
  gstack 两条都用

## 4. 五个标杆级实践

1. **给"审查者的判断力"写回归测试** — 真实事故(review skill 写完报告全程未提问)
   → 种缺陷夹具(`forcing-finding-seeds.ts`,埋一个不可能漏看的缺陷)→ 断言至少
   一个发现(gate 级);periodic 级用 [N-1, N+2] 容差带,把非确定性当统计量
2. **prompt 体积是性能指标** — `parity-baseline-*.json` 按版本记录每个 skill 的
   字节/行数/token 估算 + topHeaviest 排行(ship: 174KB/4.3 万 token)。
   bundle-size budget 思路的移植:上下文窗口就是带宽
3. **价值观被编译进每个 skill,且受保护** — ETHOS.md 注入所有工作流 skill;
   "Boil the Ocean" 反转行业格言,论据是成本结构变了;同时社区 PR 碰 ETHOS/
   语气/文风必须 AskUserQuestion,AI 再认同也禁止自决
4. **连顿悟都有日志** — Layer 3 第一性原理洞察写入 eureka.jsonl,/retro 聚合展示。
   失败、使用、分歧、灵感全部可观测
5. **决策记忆是事件溯源的** — `gstack-decision-log` 支持 supersede/redact/compact,
   高危密钥拒收,契约 NON-INTERACTIVE(调用者是 agent,交互提示会挂死它)。
   "决策会过时"被设计进 schema

## 5. 母题

把软件工程全套纪律(回归测试、性能预算、事件溯源、可观测性、迁移脚本、
事故复盘)**原封不动施加在 prompt 和 agent 行为上**。gstack 真正的论证不是
某个 skill 写得好,而是:**prompt 是值得被工程化的一等生产资产**。

每个流程环节都对应一个真实踩过的坑:没有行为回归测试 → review 静默退化;
没有体积预算 → ship 膨胀;没有遥测 → 死功能存活 18 天无人知。流程不是仪式。

## 6. 生命周期映射表(skill 版 SDLC)

| 传统环节 | gstack 等价物 |
|---|---|
| 源码 vs 构建产物 | .tmpl + resolvers → SKILL.md(禁止直接编辑产物) |
| 本地开发 | dev:skill watch 模式、dev symlink 即时生效 |
| 测试 | 三层:静态(免费)→ LLM-judge(~$0.15)→ E2E(~$3.85) |
| CI 选择 | touchfiles 依赖声明 + diff-based selection + gate/periodic 双轨 |
| 性能预算 | 100KB 上限 + parity baseline 追踪 |
| 发布 | 四段版本号 + 面向用户的 CHANGELOG 纪律 |
| 部署 | ./setup 多宿主编译安装 |
| 数据库迁移 | gstack-upgrade/migrations/(磁盘状态结构变更) |
| 监控 | telemetry JSONL + .pending 崩溃捕捉 |
| 事故→回归 | 种缺陷夹具永久 gate 测试 |
| 反馈回路 | question-log / learnings / retro / 蒸馏提案 |
