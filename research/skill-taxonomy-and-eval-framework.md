# Skill 分类与测评体系设计

> 状态:草案 v1.2 · 2026-06-11
> 来源:gstack 仓库考古(skill 演化史、反馈回路、preamble 机制)+ 业界标杆调研
> 配套资料:NotebookLM notebook `b067d181-6f89-410c-b2be-7fb1c42bfcfd`(标杆来源)
>
> **v1.1 修订**,两个来源:
> ① 对照 Anthropic skill-creator 2.0 博客 — L1 补自动修复回路、A/B 拆双频、
> 测评触发器补「模型更新」类、L3 补运行时成本、新增 §5 退役协议;
> ② gstack 四类冠军实测(`audits/gstack-2026-06.md`)— L0 补注册表完备性检查、
> L0↔L1 张力规则、L2 补行为塑造型 hook 化优先、「分类空格有信息量」。
>
> **v1.2 修订**,两个来源:
> ③ anthropics/skills 官方库实测(`audits/anthropic-skills-2026-06.md`)—
> 审计协议补可见性限定与仓库性质判断、L0 收录 description 路由算法模式、
> L1 负例集强制含同族 sibling;
> ④ 微软 SkillOpt(arXiv 2605.23904)— 新增轴 5 来源(provenance)、
> 改进段从「仅 gstack」改为双闸门哲学并存、eval-as-损失函数强化退役协议。

## 0. 背景与动机

SKILL.md 已于 2025 年 12 月成为开放标准,30+ 平台采纳,市场上插件过万。
但生态里的 skill 质量参差,且缺乏统一的分类与测评语言:

- Anthropic skill-creator 2.0 覆盖了「开发 + 测试」段(先写 eval 再写文档、多 agent 并行评测)
- obra/superpowers 覆盖了「过程方法论」段(严格 TDD、spec 先行)
- OpenAI Codex `$skill-installer` 与各插件市场覆盖了「分发部署」段
- PromptOps 工具链(promptfoo / Braintrust / LangSmith)覆盖了「测试 + 监控」段
- **「改进段」(生产遥测 → 学习 → 复盘 → 蒸馏 → 人工裁决)在业界标杆中是空白**,
  目前只有 gstack 一家做了完整闭环

本文档提出一套配套设计的分类体系与测评体系。核心原则:
**分类轴的选取标准不是方便归档,而是每个轴必须改变测评方式。**
不影响怎么测的维度,没资格成为轴。

## 1. 分类体系:五个正交轴

### 轴 1:本体类型(决定失效方式与测评重心)

| 类型 | 例子 | 典型失效方式 |
|------|------|-------------|
| 知识注入型 | API 文档参考、品牌规范 | 内容过时、与上游事实冲突 |
| 工具包装型 | 下载器、格式转换器 | 命令错误、错误路径未覆盖 |
| 工作流型 | /ship、/office-hours | 跳步、漏掉决策门、顺序错乱 |
| 行为塑造型 | TDD 强制、Confusion Protocol、反谄媚规则 | 静默退化(流程还在跑但灵魂缺席) |
| 元技能型 | skill-creator、/retro、/plan-tune | 错误抽象被固化、自我强化偏差 |

**分类空格有信息量**(v1.1,来自 gstack 实测):某类型为空集不是分类失败,
可能揭示架构选择——gstack 知识注入型为零,因为知识全部走 preamble 编译期注入。
审计报告应保留空格而不是强行填类。

### 轴 2:判断密度(决定断言方式)

- **确定性端**(转码文件、跑命令):精确断言,gate 级测试,阻塞合并
- **判断密集端**(审计划、做诊断):统计容差带断言(如发现数 ∈ [N-1, N+2])
  或 LLM-as-judge,periodic 级,定期巡检
- 对应 gstack 的 gate / periodic 双轨制

### 轴 3:爆炸半径(决定人工闸门要求)

- **只读**:无闸门要求
- **可逆写入**:测评中检查是否声明了回滚路径
- **不可逆外部动作**(部署、发布、付款):测评中「未经确认即执行」为一票否决项;
  必须有 AskUserQuestion 式闸门,且闸门触发本身要纳入 L2 合规测试

### 轴 4:触发方式(决定 L1 是否必测)

- **显式触发**(斜杠命令):跳过 L1
- **描述匹配自动触发**:必测 L1 触发精度(见下)

### 轴 5:来源 provenance(v1.2,决定审计重心)

- **手写**:why 可追溯(commit message 记录触发失败),人工可读可审
- **优化器产出**(SkillOpt 式文本空间训练):每条编辑只有 score delta,
  没有叙事性 why;向验证分数优化出的稠密文本人工抓手少。审计重心变化:
  1. **L2 行为合规升为 ✦✦**(不论本体类型)——要测的恰恰是 benchmark
     之外的品质(语气、安全边界、罕见场景),它们可能在优化中被悄悄交易掉
  2. 检查训练用验证集与部署场景的分布差距(benchmark 即损失函数,
     损失函数偏了 skill 就偏了)
  3. why 缺失是该来源的固有属性,不作扣分项,但要求保留训练配置
     与 benchmark 版本号作为替代溯源

## 2. 测评体系:五层金字塔

```
L4  生产遥测        持续、被动      （改进段,业界空白）
L3  端到端结果      $$$,diff 触发
L2  行为合规        $,LLM judge
L1  触发精度        ¢,批量跑
L0  静态校验        免费,每次提交
```

### L0 静态校验(免费,<1s,每次提交)

- frontmatter schema 合法(name、description 必填且合规)
- token 预算:生成产物不超过上限(参考 gstack 100KB ≈ 25K tokens)
- description 质量:长度、触发短语覆盖。**最佳实践上限是「路由算法式
  description」**(v1.2,claude-api 为证):内嵌 TRIGGER 正触发面 +
  SKIP 负触发 + 预检指令(先跑 grep 再决定),代价 ~1KB catalog token
  ——是否值得由该 skill 的误触成本决定
- 无硬编码路径 / 分支名 / 框架假设(平台无关性检查)
- 无注入风险模式(写入路径过 injection lint)
- **测试注册表完备性**(v1.1):每个 skill 必须在测试选择系统(如 touchfiles)
  有条目,或显式豁免。来历:gstack 的 guard/careful 因未登记而成为永久测试盲区
  ——「注册表漏项」比单个测试缺失更隐蔽

**L0↔L1 张力规则**(v1.1):description 是 catalog token 成本的主体,
瘦身省 token 但伤触发查全(gstack Token-reduction 后 codex 描述只剩一句话)。
规则:**任何 description 瘦身必须重跑 L1**,两层指标会互相吃。

### L1 触发精度(便宜,只测路由不测执行)

Skill 区别于普通 prompt 的独有测评层。每个自动触发 skill 配:

- 5+ 条**应该触发**的用户语句(正例)
- 5+ 条**不应该触发**的相邻语句(负例)。**负例集必须包含同族 sibling
  场景**(v1.2):同族触发重叠是跨库通病——gstack 的 careful/guard/freeze、
  官方库的 frontend-design/canvas-design/theme-factory
- 指标:激活查准率 / 查全率
- 参考实现:`skill-creator/scripts/run_eval.py`(测量)+
  `improve_description.py`(修复)

生态两大常见病(该出现时缺席、不该出现时抢戏)都在此层暴露。

**自动修复回路**(v1.1,借鉴 skill-creator 2.0):触发失败的修复手段几乎
只有改 description,可自动化——测出 FP/FN → 自动生成 description 修改建议
→ 重测。L1 不止是测量,应闭环到修复。

### L2 行为合规(中等成本,LLM-as-judge,~$0.1-0.5/次)

不看结果看过程。两项来自 gstack 实践的关键技术:

1. **种缺陷夹具(finding-floor)**:给审查类 skill 一份故意埋了
   「不可能漏看的明显缺陷」的输入(如:无理由手写 UUID 生成器而不用内置),
   断言至少报告一个发现。这是对「判断力静默退化」的最小成本回归测试。
   来历:2026-05 真实事故——review skill 写完整份报告却全程未提问。
2. **容差带断言**:对判断密集型 skill,断言发现数落在 [N-1, N+2] 区间,
   承认 LLM 非确定性,把行为当统计量处理。

其他合规项:禁用短语检测(反谄媚)、必经决策门是否触发、输出格式合规。

**行为塑造型优先 hook 化**(v1.1,来自 gstack /guard 实测):能降级为
确定性代码的行为约束就别留在 prompt 里。gstack 的 guard 用 PreToolUse hook
跑拦截脚本(脚本有单元测试),把「模型会不会听话」转化为「代码逻辑对不对」
——L2 最难测的问题被架构消解。先问能否 hook 化,再设计 judge 测试。
注意:hook 化后仍需一个 E2E 验证**接线**(hook 真的装上了、真的拦住了),
脚本单元测试不能替代。

### L3 端到端结果(贵,~$1-4/次)

真实任务跑通,断言产物存在且正确。成本控制三件套(照抄 gstack):

- **touchfiles**:每个测试声明文件依赖
- **diff-based 选择**:只跑受本次改动影响的测试
- **gate / periodic 双轨**:确定性安全测试进 CI 阻塞合并;
  质量基准 / 非确定性 / 依赖外部服务的进每周巡检

**触发器是三类,不是两类**(v1.1,借鉴 skill-creator 2.0):
skill diff(已有)、定期巡检(已有)、**模型版本变更 → 全量 benchmark**。
skill 没动但脚下的模型动了,行为照样漂移——"Running evals against a new
model gives you an early signal when something shifts."

**运行时成本纳入指标**(v1.1):benchmark 记录 pass rate + 耗时 +
token 用量。通过率不变但 token 翻倍 = 效率回归。静态体积预算(L0)
只管产物大小,这里管执行成本,两者都要。

### L4 生产遥测(持续被动收集,JSONL 起步)

| 指标 | 采集方式 | 回答的问题 |
|------|---------|-----------|
| 激活率 | skill 运行日志 | 这个 skill 还活着吗(18 天未触发 → 候选删除) |
| 完成率 / 放弃率 | `.pending` 开始标记 + 结束日志 | 用户中途放弃说明哪一步劝退 |
| 用户推翻率 | 问答日志(AI 推荐 A,用户选 B) | 推荐质量;高推翻率 = 抽象与用户意图错位 |
| 死段检测 | 段落级影响标记 | skill 里哪些段落从未影响行为 |
| 失败分类 | error-class / failed-step 字段 | 失败聚集在哪一步 |

采集纪律(来自 gstack):遥测脚本永不非零退出(no `set -e`)、
限速上报、隐私分级、本地 JSONL 先行可审计。

## 3. 类型 × 层级交叉矩阵

| | L1 触发 | L2 合规 | L3 端到端 | L4 遥测 | 特有指标 |
|---|---|---|---|---|---|
| 知识注入型 | ✦ | — | ✦ | ○ | **保鲜度**(vs 上游文档滞后) |
| 工具包装型 | ○ | ○ | ✦✦ | ○ | 错误路径覆盖率 |
| 工作流型 | ○ | ✦✦ | ✦ | ✦ | 步骤合规率、闸门触发率 |
| 行为塑造型 | — | ✦✦ | ○ | ✦✦ | **A/B 区分度**(见 §4) |
| 元技能型 | ○ | ✦ | ✦ | ✦ | 提案采纳率 |

✦✦ 重点投入 · ✦ 必测 · ○ 抽测 · — 不适用

## 4. 终极指标:A/B 区分度(v1.1 拆为双频)

核心方法不变:judge **盲评**两组输出(不知道哪组是哪个版本)。
v1.1 起拆为两个频率和用途不同的工具:

**(a) 版本对比 — 高频,廉价,每次实质性修改都跑**

- skill v1 vs v2,回答「这次改动真的更好吗」
- 借鉴 skill-creator 2.0 的 comparator agents:"two skill versions...
  They judge outputs without knowing which is which"
- 没有它,skill 的每次修改都是盲飞——这是 prompt 工程版的回归测试缺口
- 极端参考实现(v1.2):SkillOpt 的验证门控把它做成每次编辑的硬约束
  ——held-out 验证分不严格提升就拒绝该编辑

**(b) 存在性测试 — 模型更新时跑(不再是固定季度)**

- skill vs 无 skill,回答「这个 skill 还该不该存在」
- 差异不显著 → 死重,白白消耗每次会话 token
- 把「触发率为零就删」从触发层推广到效果层:不仅要被用到,还要用到了有用
- **运行时机改为每次模型更新**:区分度持续走低是健康的退役信号(见 §5)

## 5. 退役协议(v1.1 新增)

skill 的死法有两种,性质完全不同:

| 退役信号 | 来源 | 处置 |
|---------|------|------|
| **死重**:长期零触发 / A/B 无区分度 | gstack「18 天零触发即删」 | 删除 skill 与其测试 |
| **被模型吸收**:基座模型不带 skill 也能通过其 eval | skill-creator 2.0:"if the base model starts passing your evals without the skill loaded, that's a signal the skill's techniques may have been incorporated" | **归档 skill,保留 eval** |

第二种不是失败,是毕业。由此推出本框架最重要的资产判断:

> **skill 是对当前模型能力缺口的临时脚手架,会折旧;
> eval 套件才是耐久资产** — 它是期望行为的规格说明,
> skill 退役后 eval 继续活着,变成检验「模型是否还需要脚手架」的标尺。

实践含义:写 skill 时投入在 eval 上的精力,折旧速度远低于投入在
prompt 措辞上的精力。

v1.2 强化:SkillOpt 把这条论断变成机械事实——没有 benchmark 就**无法**
训练 skill,eval 字面上成了损失函数,skill 是文本空间梯度下降的输出。
「先写 eval」从最佳实践升级为工程必然。

## 6. 评分政策:防古德哈特

- 综合健康分 = f(L0-L3 通过率, L1 触发精度, L4 遥测, 保鲜度)
- **健康分只用于排序巡检优先级,不用于自动门禁**
- 门禁只看 L0-L2 的硬断言
- 理由:任何变成目标的指标都会失真(参考 gstack slop-scan 章节
  「别追数字,接受 sloppy 模式恰是正确工程选择的发现」)

## 7. 落地顺序(按依赖关系)

1. **L0 静态校验** — 一天可成,立刻防住最常见的手改漂移
2. **L1 触发集** — 每 skill 5 正 5 负例句,机械工作量
3. **L2 种缺陷夹具** — 只给判断密集型 skill 配
4. **L4 遥测** — JSONL + 几个 bash 脚本起步,够用很久
5. **L3 端到端** — 最后做,最贵,需要 diff-selection 基建先行
6. **A/B 双频** — 版本对比随修改跑;存在性测试随模型更新跑(§4-5)

宿主形态二选一:独立评测仓库,或元 skill(`/skill-audit`:
对照本矩阵给任意 skill 出健康报告)。实测见 `audits/`
(gstack 2026-06、anthropics/skills 2026-06)。

### 审计协议(v1.2)

1. **先判断仓库性质**:生产库(skill 在此开发、测试、迭代)还是
   分发库(制品 showroom,工程回路在别处)。anthropics/skills 为证:
   零 CI 零遥测不等于不工程化——回路在内部不可见
2. **可见性限定**:分发库的 L2-L4「缺失」结论必须标注「公开层面」,
   区分 absent(不存在)与 not public(不公开)
3. 优化器产出的 skill 按轴 5 调整审计重心

## 8. 业界标杆映射(详见 NotebookLM notebook)

| 生命周期段 | 标杆 | 本框架对应层 |
|-----------|------|------------|
| 开发 + 测试 | Anthropic skill-creator 2.0 | L1-L3 |
| 过程方法论 | obra/superpowers | (正交,可叠加) |
| 分发部署 | Codex $skill-installer、插件市场 | (不在本框架范围) |
| 测试 + 监控 | promptfoo / Braintrust / LangSmith | L2-L3 通用基建 |
| 编译优化 | DSPy(学术)→ **SkillOpt**(生产级,v1.2) | A/B 门控的极端实现 |
| **改进闭环** | **双闸门哲学并存**(v1.2):gstack(人工裁决)vs SkillOpt-Sleep(指标裁决) | L4 + 蒸馏 + 闸门 |

## 9. 设计原则备忘

1. **可证伪性是第一块基石** — 三个独立来源共同指向:
   官方「先写 eval 再写文档」、superpowers「先有失败测试再有代码」、
   Respan「没有 eval 的版本管理只是 diff」
2. **写入廉价自动,固化过人工闸门** — AI 可提议抽象,不可自行固化抽象
3. **认识论卫生** — 数据条目区分来源(observed / user-stated / inferred),
   只有用户明示的才标记 trusted
4. **prompt 是生产系统** — 有测试、有预算、有遥测、有事故复盘、有保护边界
5. **框架有时间轴**(v1.1)— 模型在变:触发器含模型更新、A/B 常态化、
   skill 有退役协议。skill 是脚手架会折旧,eval 是耐久资产
6. **谁写 benchmark,谁掌握 skill 的灵魂**(v1.2)— 当抽象的生成与验证
   都被自动化(SkillOpt),人的位置上移到定义验证集。held-out 验证防的是
   过拟合,防不了「benchmark 没测的品质被交易掉」——人工裁决的最后阵地
   是验证集的设计与 L2 对 benchmark 外品质的把守
