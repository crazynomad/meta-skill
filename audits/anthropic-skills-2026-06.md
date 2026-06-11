# anthropics/skills 审计报告(2026-06)

> 框架版本:v1.1 · 第二次实测
> 对象:github.com/anthropics/skills @ `5754626`,17 个 skill + 1 模板 + 规范
> 方法:同 gstack 审计——轴 1 全量分类 → 每类挑修改最频繁者 → 五层评测
> 范围限制:静态审计,未实跑付费 eval

## 〇、先校准假设:这不是一个「生产系统」仓库

与 gstack 的根本差异在仓库性质:

- **无任何 CI**(没有 .github/workflows),无测试夹具,无遥测,无 touchfiles 类机制
- 迭代次数低一个数量级:冠军 claude-api 仅 17 commits(gstack 的 ship 是 55,
  且 gstack 总量 ~1950 PR)
- 有 `.claude-plugin/marketplace.json` —— 这是一个**分发制品库(showroom)**,
  不是生产农场。工程回路(内部 eval、使用遥测)几乎可以肯定存在于 Anthropic
  内部,只是不在这个公开仓库里

**审计协议修正(回灌框架)**:对分发型仓库,L2-L4 的「缺失」可能意味着
「在别处、不公开」,而非「不存在」。审计结论必须带可见性限定词。

## 一、分类(轴 1,全量 17 个)

| 类型 | 数量 | 成员(commits) | 冠军 |
|------|-----|----------------|------|
| 知识注入型 | 3 | claude-api(17)、brand-guidelines(2)、internal-comms(2) | **claude-api(17)** |
| 工具包装型 | 8 | docx(4)、pptx(4)、xlsx(3)、pdf(3)、slack-gif-creator(3)、webapp-testing(2)、theme-factory(2)、web-artifacts-builder(2) | **docx(4,与 pptx 并列)** |
| 工作流型 | 2 | mcp-builder(2)、doc-coauthoring(1) | **mcp-builder(2)** |
| 行为塑造型 | 3 | frontend-design(3)、canvas-design(2)、algorithmic-art(2) | **frontend-design(3)** |
| 元技能型 | 1 | skill-creator(5) | **skill-creator(5)** |

与 gstack 形成镜像:gstack 知识注入型为空(知识走 preamble 编译);
官方库五类齐全,且**总冠军是知识注入型**(claude-api)——分发库的
价值重心在「可移植的知识与工具」,生产库的重心在「工作流」。

## 二、冠军评测

### claude-api(知识注入型,17 commits)— 🟢 L1 工艺的业界天花板

| 层 | 结果 | 证据 |
|----|------|------|
| L0 | ✓ | 42KB 主文件 + 49 个文件渐进披露(shared/ + 按语言分文档) |
| L1 | ✦✦ **业界最强** | description(1140B)内嵌完整路由算法:TRIGGER 正则触发面(模型名全形态)、SKIP 负触发(其他厂商)、甚至指示**先跑一条 grep 再决定**。这不是触发短语,是嵌入式决策树 |
| L2/L3 | ✗(公开层面) | 仓库内无任何针对它的 eval |
| L4 | ✗(公开层面) | — |
| 保鲜度(特有指标) | ✓ 手动管理 | 17 个 commit 几乎逐条对应模型发布(Fable 5、Opus 4.8 迁移、scheduled deployments)——commit 历史就是保鲜记录。但无自动化陈旧检测(与上游 docs 的 diff 比对),仍是机会点 |

### docx(工具包装型,4 commits)— 🟡 描述教科书,脚本无测试

| 层 | 结果 | 证据 |
|----|------|------|
| L0 | ✓ | 20KB + 61 文件(脚本 + OOXML 参考) |
| L1 | ✓✓ | 教科书式 description:正触发 + 口语触发("the xlsx in my downloads"式)+ 显式负空间("Do NOT use for PDFs, spreadsheets, Google Docs...") |
| L2/L3 | ✗ | 按矩阵,工具包装型 L3 是 ✦✦——但 61 个文件里的 Python 脚本在仓库内**零测试**。错误路径覆盖率无从谈起 |
| L4 | ✗ | — |

### mcp-builder(工作流型,2 commits)— 🟡 教别人写 eval,自己没有

含 `scripts/evaluation.py` + `reference/evaluation.md`——工作流本身把
「为你构建的 MCP server 写 eval」作为步骤教给用户。但这个工作流 skill
自身在仓库内无任何测评。「布道者悖论」的温和版本。

### frontend-design(行为塑造型,3 commits)— 🟠 最需要 A/B 的类型,最没有 A/B

| 层 | 结果 | 证据 |
|----|------|------|
| L0 | ✓ | 8KB,2 文件,极简 |
| L1 | ○ | description 偏抽象("distinctive, intentional visual design"),无负空间;与 canvas-design / theme-factory / web-artifacts-builder 存在同族触发重叠(同 gstack 的 careful/guard/freeze 问题) |
| L2 | ✗ | 行为塑造型按矩阵 L2+A/B 是 ✦✦:「装了它美学真的变了吗」恰恰是盲评 A/B 的教科书场景,仓库内为零 |
| L4 | ✗ | — |

正文质量本身很高(人格化姿态指令:"design lead at a small studio...
take one real aesthetic risk you can justify")——但效果无公开证据。

### skill-creator(元技能型,5 commits)— 🟢 把我们的 L0-L1 实现成了产品

`scripts/` 目录是一套近乎完整的框架下半层实现:

| 脚本 | 对应框架层 |
|------|-----------|
| `quick_validate.py` | **L0** 静态校验 |
| `run_eval.py` | **L1** 触发精度("Tests whether a skill's description causes Claude to trigger... for a set of queries") |
| `improve_description.py` | **L1 自动修复回路**(v1.1 §L1 的参考实现,先于我们写出) |
| `aggregate_benchmark.py` + `run_loop.py` | benchmark + 方差分析 |
| `eval-viewer/` | 结果仪表盘 |

**但**:这套工具在仓库内没有被用于自家 17 个 skill——无 query 集、无 CI、
无结果留存。健身房悖论:卖器材的店里没人健身(公开可见层面)。

## 三、汇总

| | L0 | L1 | L2/L3(公开) | L4(公开) | 总评 |
|---|---|---|---|---|---|
| claude-api | ✓ | ✦✦ 天花板 | ✗ | ✗ | 🟢(L1+保鲜撑起) |
| docx | ✓ | ✓✓ | ✗ 脚本零测试 | ✗ | 🟡 |
| mcp-builder | ✓ | ✓ | ✗ | ✗ | 🟡 |
| frontend-design | ✓ | ○ 同族重叠 | ✗ | ✗ | 🟠 |
| skill-creator | ✓ | ✓ | (是工具本身) | ✗ | 🟢 |

## 四、与 gstack 审计的对照

| 维度 | gstack | anthropics/skills |
|------|--------|-------------------|
| 仓库性质 | 垂直整合生产农场 | 分发制品库(showroom) |
| CI/测试 | 三层 + diff-selection + 双轨 | **零** |
| L1 工艺 | 中等(描述被 token 预算挤压) | **业界最强**(claude-api 路由算法) |
| L4 遥测 | 完整闭环 | 零(公开层面) |
| 迭代强度 | ship 55 commits | claude-api 17 commits |
| 知识注入型 | 空集(走 preamble) | 总冠军所在类 |
| 渐进披露 | 单文件为主 | 49-83 文件/skill,架构性使用 |

**两库恰好互补**:官方库赢在单 skill 工艺(L1 description、渐进披露、
工具实现);gstack 赢在系统工程(测试、遥测、改进回路)。
「最成熟」取决于问的是制品还是过程。

## 五、回灌框架的候选修订(待并入 v1.2)

1. **审计协议补「可见性限定」**:分发型仓库的 L2-L4 缺失 ≠ 不存在,
   结论标注「公开层面」。新增仓库性质前置判断:生产库 / 分发库
2. **L0 description 质量检查升级**:claude-api 证明 description 可以承载
   显式 TRIGGER/SKIP/预检指令的路由算法,不只是触发短语堆叠。
   L1 最佳实践应记录此模式(代价:1140B 的 catalog token——又是 L0↔L1 张力)
3. **L1 自动修复回路有参考实现**:`skill-creator/scripts/improve_description.py`,
   落地时直接借鉴
4. **知识注入型保鲜度的自动化机会**:commit-cadence 手动保鲜是现状,
   「与上游文档自动 diff 检测陈旧」业界仍空白
5. **同族触发重叠是跨库通病**:gstack 的 careful/guard/freeze、官方的
   frontend-design/canvas-design/theme-factory——L1 负例集应强制
   包含同族 sibling 的场景

## 六、未尽事项

- 全部为静态审计;run_eval.py 实跑(给 17 个 skill 建 query 集)是最有
  价值的后续——官方自己的工具测官方自己的 skill
- 行为塑造型三件套(frontend-design / canvas-design / algorithmic-art)
  的同族触发混淆矩阵值得实测
- Anthropic 内部 eval/遥测的存在性无法从公开仓库验证,本报告全部
  「✗」均限定于公开可见层面
