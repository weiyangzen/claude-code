# FILE `CHANGELOG.md` 研究文档

## 场景与职责

`CHANGELOG.md` 是仓库根目录的发布变更总账，承担“版本演进事实源（human-readable source of truth）”职责：

1. 对外职责：
- 面向用户/维护者披露每个版本的新增、修复、性能优化与兼容性变化。
- 覆盖 CLI、Hooks、MCP、插件系统、VSCode 集成、安全修复等多维能力。

2. 对内职责：
- 作为跨模块演进的历史索引，支持排障、回归分析、迁移决策。
- 为研究文档与插件开发指南提供“变更依据”引用（本仓已有多份研究文档直接引用其行号）。

3. 在当前仓库中的调用关系（研究语义下）：
- 上游（生产者）：发布维护流程中的人工编辑/版本发布步骤。
- 下游（消费者）：人工阅读、研究流程、插件开发文档规范。
- 边界：未发现仓内脚本直接解析 `CHANGELOG.md` 生成产物（即当前不是机器消费主链路）。

## 功能点目的

1. 版本可追溯
- 文件从 `2.1.79` 追溯到 `0.2.21`，形成连续历史窗口（按“新到旧”排序）。

2. 兼容性与迁移指引
- 明确记录 `Deprecated` / `Removed` / `Breaking change`，为升级决策提供依据。
- 典型示例：output style 的发布、弃用、回调、再次弃用路径清晰可查。

3. 运维与安全信息公告
- 记录安全修复、权限模型调整、沙箱行为修复、环境变量开关等高影响变更。

4. 多终端/多入口能力对齐
- 通过 `[VSCode]`、`[SDK]`、Windows/Bedrock/Vertex 等标签化叙述，帮助不同使用入口快速筛选相关变更。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 文档结构协议（Markdown 语法约定）

`CHANGELOG.md` 遵循稳定的层级结构：

1. 一级标题：`# Changelog`。
2. 二级标题：`## <semver>`（例如 `## 2.1.79`）。
3. 条目：`- ` 开头的 bullet list，单行或长句描述变更。
4. 条目内约定前缀：`Added` / `Fixed` / `Improved` / `Changed` / `Deprecated` / `Removed` / `Breaking change` / `**Security:**`。
5. 可选范围标签：`[VSCode]`、`[SDK]` 等。

这是一种“面向阅读”的轻约束协议，不是机器强校验 schema。

### 2) 信息模型（本次实测统计）

通过仓内命令统计得到：

```bash
# 版本段总数
rg -n '^## ' CHANGELOG.md | wc -l
# => 243

# 条目前缀统计（仅统计以 "- " 开头行）
# total=1662, Added=261, Fixed=675, Improved=168, Changed=31,
# Deprecated=6, Removed=10, Security=1, BreakingChange=3,
# VSCode=38, SDK=4
```

结论：
- `Fixed` 占比最高，说明该文件在“回归修复可追溯”上的价值大于“营销式发布说明”。
- 结构上无重复版本标题（`uniq -cd` 为空）。

### 3) 关键流程（从写入到消费）

1. 发布记录写入：维护者在对应版本段下追加条目（当前仓内未见自动生成器）。
2. 人工消费：开发者/用户按版本段阅读，筛选与自身场景相关条目。
3. 二次引用：研究文档或插件开发材料引用具体行号作为证据。
4. 历史溯源：通过关键词（如 `SessionStart`、`/output-style`、`worktree`、`MCP`）定位能力演进路径。

### 4) 典型演进链（协议与命令层）

1. Output style 生命周期：
- 发布：`1.0.81` 发布 output styles。
- 弃用：`2.0.30` 标记弃用，建议迁移到 system prompt / CLAUDE.md / plugins。
- 回调：`2.0.32` 因反馈取消弃用。
- 再策略收敛：`2.1.72` 再次废弃 `/output-style` 并固定在 SessionStart。

2. 插件系统里程碑：
- `2.0.12` 明确“Plugin System Released”，并同步 `/plugin ...` 管理命令。

3. 发布说明入口能力：
- `0.2.37` 引入 `/release-notes` 命令，说明“变更说明展示”在产品交互层也有映射。

### 5) 配置/命令信息承载方式

该文件大量承载以下“文本级协议元素”：

1. CLI 命令：如 `/config`、`/plugin`、`/sandbox`、`/mcp`、`/effort`。
2. 环境变量：如 `CLAUDE_CODE_*`、`ANTHROPIC_*` 系列。
3. 协议名词：MCP、Hooks 事件、Worktree、OAuth、LSP 等。

注意：`CHANGELOG.md` 只“描述”这些配置项，并非其真值定义源；权威定义仍在代码与官方文档。

## 关键代码路径与文件引用

1. 核心对象
- `CHANGELOG.md:1`：文档根标题。
- `CHANGELOG.md:3`：最新版本段 `2.1.79`。
- `CHANGELOG.md:2390`：最旧可见版本段 `0.2.21`。

2. 关键里程碑证据
- `CHANGELOG.md:1622-1629`：`2.0.12` 插件系统发布与 `/plugin` 命令族。
- `CHANGELOG.md:1835-1837`：`1.0.81` output styles 发布。
- `CHANGELOG.md:1520-1528`：`2.0.30` output styles 弃用。
- `CHANGELOG.md:1505-1507`：`2.0.32` output styles 取消弃用。
- `CHANGELOG.md:206`：`/output-style` 再次弃用并固定 SessionStart。
- `CHANGELOG.md:1905-1908`：`1.0.62` 引入 `SessionStart` hook。
- `CHANGELOG.md:2354-2356`：`0.2.37` 引入 `/release-notes`。

3. 研究流程与任务驱动（本仓上下文依赖）
- `.ops/research_guard.sh:286-310`：对 FILE 研究任务模板，明确要求输出 `原文件名_research.md`、勾选 checklist、更新 todo、提交。
- `.ops/generate_research_blueprint_checklist.sh:10-24,44-66`：保留已勾选状态并重建研究蓝图清单。
- `.ops/generate_daily_research_todo.sh:15-18,33-39`：从 checklist 统计 pending/done 并生成每日 todo。
- `Docs/researches/blueprint_checklist.md:143`：当前目标项 `[FILE] CHANGELOG.md`。
- `Docs/researches/todos_20260320.md:14`：当天待办中包含 `[FILE] CHANGELOG.md`。

4. 外围文档中对 changelog 的规范性引用
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:543`：维护建议明确要求“Maintain changelog”。
- `plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:512-554`：给出版本与 changelog 维护模板。
- `plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:733`：发布前清单包含 “Changelog maintained”。
- `plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:754,836`：发布/补丁策略要求更新 changelog。
- `plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:382,391-401`：发布工作流示例中存在 release notes 生成链路。

## 依赖与外部交互

1. 仓内依赖
- 无脚本直接解析 `CHANGELOG.md` 作为输入（基于仓内全文检索结果）。
- 其主要依赖为“人工维护纪律”和“文档使用习惯”。

2. 与配置/测试/脚本的关系
- 配置：文件中频繁提及设置项和环境变量，承担“变更公告”角色而非配置落地角色。
- 测试：未发现针对 changelog 结构或内容一致性的自动化测试/CI 校验。
- 脚本：`.ops` 系列研究脚本会把 `CHANGELOG.md` 纳入研究对象清单，但不消费其业务内容。

3. 外部交互
- 条目中包含外链（Anthropic 博客与 docs），对外部页面可达性与内容稳定性有隐含依赖。
- 历史上存在 `/release-notes` 命令，表明发布说明可能在产品中被展示，但仓内未包含该命令实现代码。

## 风险、边界与改进建议

1. 风险：缺少结构化 schema，难以自动消费
- 当前纯 Markdown 自由文本不利于自动生成迁移报告、按组件筛选、变更告警。
- 建议：在保留 Markdown 的同时，引入机器可读伴生文件（如 `changelog.json`）。

2. 风险：版本段缺少日期字段
- 虽有语义版本号，但缺少统一发布日期，不利于审计与跨系统对账。
- 建议：采用 `## x.y.z - YYYY-MM-DD` 或在段首统一增加日期行。

3. 风险：标签规范未强约束
- 存在 `VSCode` / `[VSCode]`、`SDK` / `[SDK]` 等表述并存，自动筛选成本高。
- 建议：定义并 lint 前缀规范（例如固定 `[Scope] Type:`）。

4. 风险：行号引用脆弱
- 研究文档大量以 `CHANGELOG.md:line` 引用，一旦前文插入，行号会整体漂移。
- 建议：研究文档增加“版本号+关键词”双锚点，减少单纯行号耦合。

5. 风险：重大变更识别依赖人工
- `Breaking change`、`Security` 条目虽然存在，但未见 CI 校验“每次重大变更必须打标签”。
- 建议：引入简单校验脚本（pre-commit/CI）检查关键标签与最小信息集。

6. 边界说明
- `CHANGELOG.md` 不是运行时代码，不直接影响执行路径；其价值在于“沟通与治理”。
- 因此核心质量指标是可读性、一致性、可检索性，而不是运行性能。

7. 可执行改进路线（低风险）
- 第一步：补齐每个版本发布日期。
- 第二步：固定条目前缀与 scope 标签词表。
- 第三步：增加一个只读校验脚本，检查标题格式、重复版本、关键标签完整性。
- 第四步：输出 machine-readable 伴生索引，服务自动化 release note 摘要与迁移提醒。
