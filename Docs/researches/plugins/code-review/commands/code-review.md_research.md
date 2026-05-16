# plugins/code-review/commands/code-review.md 研究

## 场景与职责

`plugins/code-review/commands/code-review.md` 是 `code-review` 插件的核心执行协议文件，负责把一个自然语言命令 `/code-review` 约束成可重复执行的 PR 审查流程（多 agent 并发、问题复核、结果输出、可选评论回写）。

它在插件体系中的位置与职责边界如下：

- 插件发现入口：marketplace 将 `code-review` 插件注册到 `./plugins/code-review`，使其可被加载（`.claude-plugin/marketplace.json:29-37`）。
- 插件元数据入口：`plugin.json` 提供插件身份与描述（`plugins/code-review/.claude-plugin/plugin.json:1-9`）。
- 命令协议入口：`code-review.md` 的 frontmatter 定义工具白名单和描述（`plugins/code-review/commands/code-review.md:1-4`）。
- 用户说明层：`plugins/code-review/README.md` 提供该命令的行为说明、使用方式和运维前提（`plugins/code-review/README.md:11-33,143-147`）。

从“调用方/被调用方”看：

- 调用方（上游）
  - Claude Code 插件加载与命令发现机制（扫描 `commands/*.md`），文件名到命令名遵循 `code-review.md -> /code-review` 约定（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-309,341-345`）。
  - 终端用户在 PR 上下文触发 `/code-review [--comment]`（`plugins/code-review/README.md:27-33`）。
- 被调用方（下游）
  - GitHub CLI：读 PR / issue、发摘要评论（`plugins/code-review/commands/code-review.md:2,18,65,90`）。
  - MCP 工具：`mcp__github_inline_comment__create_inline_comment` 发行级（inline）评论（`plugins/code-review/commands/code-review.md:2,71`）。
  - 子代理模型编排：haiku（预检/路径收集）、sonnet（摘要/规范审查/规范复核）、opus（bug 扫描/bug 复核）（`plugins/code-review/commands/code-review.md:14,24,28,32,35,55`）。

## 功能点目的

该命令把“自动化审查”拆成 9 个显式步骤，每步都在降低误报或降低操作成本：

1. 预检是否应跳过审查
- 目的：尽早终止无价值执行，避免浪费 agent 与 API 调用。
- 规则：closed / draft / trivial / 已有 Claude 评论则停止（`plugins/code-review/commands/code-review.md:14-22`）。

2. 收集相关 `CLAUDE.md` 路径
- 目的：限定规则作用域，避免拿不相关规范误判。
- 范围：根目录 `CLAUDE.md` + 改动目录对应 `CLAUDE.md`（`plugins/code-review/commands/code-review.md:24-27`）。

3. 先做 PR 变更摘要
- 目的：给后续并发审查提供统一上下文，降低理解偏差。
- 实现：sonnet 子代理先产出 summary（`plugins/code-review/commands/code-review.md:28`）。

4. 四路并发初审
- 目的：通过“规则合规 + 缺陷扫描”的并行视角提升召回率。
- 分工：2 路 sonnet 审规则，2 路 opus 审 bug/逻辑/安全（仅限改动代码）（`plugins/code-review/commands/code-review.md:30-40`）。

5. 对候选问题做二次验证
- 目的：把“看起来像问题”收敛为“高置信真实问题”。
- 策略：bug/逻辑问题用 opus 复核，规则问题用 sonnet 复核（`plugins/code-review/commands/code-review.md:55`）。

6. 过滤未验证通过的问题
- 目的：只保留高信号，压低 false positive。
- 执行：剔除 step 5 未通过项（`plugins/code-review/commands/code-review.md:57`）。

7. 输出总结并按参数分支
- 目的：兼顾“本地只看结果”和“写回 PR”两种工作模式。
- 分支：
  - 无 `--comment`：仅终端输出并结束（`plugins/code-review/commands/code-review.md:59-64`）。
  - 有 `--comment` 且无问题：发固定模板摘要评论（`plugins/code-review/commands/code-review.md:65,93-101`）。
  - 有 `--comment` 且有问题：进入 inline 评论流程（`plugins/code-review/commands/code-review.md:67`）。

8. 先形成“待发评论计划”
- 目的：发送前最终自检，降低不当评论风险（`plugins/code-review/commands/code-review.md:69`）。

9. 逐条发 inline 评论
- 目的：将问题绑定到具体代码位置并给可执行修复建议（适用于小修）。
- 关键约束：`confirmed: true`、每个唯一问题只发一次、建议块必须可一次性修复问题（`plugins/code-review/commands/code-review.md:71-77`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程（执行时序）

- 阶段 A：资格预检
  - 读取 PR 状态与评论历史（至少要求检查 `gh pr view <PR> --comments`）判断是否终止（`plugins/code-review/commands/code-review.md:14-22`）。
- 阶段 B：规则作用域构建
  - 只收集路径，不拉取全文，且限定“同路径或父路径规则可生效”（`plugins/code-review/commands/code-review.md:24-27,33`）。
- 阶段 C：并发初审 + 并发复核
  - 初审 4 路并行，复核按问题类型选模型并行执行（`plugins/code-review/commands/code-review.md:30-40,55-57`）。
- 阶段 D：输出/回写
  - 统一终端摘要，必要时调用 `gh pr comment` 或 MCP inline 评论（`plugins/code-review/commands/code-review.md:59-67,71`）。

### 2) 隐式数据结构（由协议定义，不是代码类）

- `PRMeta`
  - 字段语义：PR 状态、是否 draft、标题、描述、评论历史中是否存在 Claude 评论。
  - 来源：`gh pr view/list` 系列与 `--comments` 检查（`plugins/code-review/commands/code-review.md:2,18`）。

- `GuidelinePaths`
  - 字段语义：root + 改动目录可见 `CLAUDE.md` 路径集合。
  - 来源：step 2 的路径采集规则（`plugins/code-review/commands/code-review.md:24-27,33`）。

- `IssueCandidate`
  - 字段语义：问题描述 + 被标记原因（例如 `bug`、`CLAUDE.md adherence`）。
  - 来源：step 4 初审输出（`plugins/code-review/commands/code-review.md:30`）。

- `ValidatedIssue`
  - 字段语义：通过 step 5 验证的问题子集。
  - 来源：step 6 过滤后结果（`plugins/code-review/commands/code-review.md:55-57`）。

- `CommentPlan`
  - 字段语义：step 8 中“准备发送但尚未发送”的评论集合。
  - 来源：发送前人工自检步骤（`plugins/code-review/commands/code-review.md:69`）。

### 3) 关键协议约束

- 工具约束
  - frontmatter 通过 `allowed-tools` 白名单限制命令可调用工具面，包含 `gh` 指定子命令和 1 个 MCP 评论工具（`plugins/code-review/commands/code-review.md:2`）。
  - `description` 用于命令说明暴露（`plugins/code-review/commands/code-review.md:3`；字段语义见 `plugins/plugin-dev/skills/command-development/README.md:148-154`）。

- 高信号筛选协议
  - 只接受确定性错误/明确规则违规；拒绝风格、主观建议和依赖特定输入状态的问题（`plugins/code-review/commands/code-review.md:41-51`）。
  - 明确“不要报”的假阳性类别（预存问题、linter 可抓问题、被显式静默规则等）（`plugins/code-review/commands/code-review.md:79-86`）。

- 评论协议
  - `--comment` 控制是否写回 GitHub（`plugins/code-review/commands/code-review.md:63-67`）。
  - 无问题时固定模板文案，保证输出一致（`plugins/code-review/commands/code-review.md:93-101`）。
  - inline 链接必须全 SHA + `#Lx-Ly` 且带上下文，避免 GitHub 预览失效（`plugins/code-review/commands/code-review.md:103-109`）。

- 子代理执行协议
  - 全体子代理共享“工具可用且禁止探测式调用”前提，降低无目的调用（`plugins/code-review/commands/code-review.md:8-10`）。
  - 每个审查/复核子代理都应接收 PR 标题与描述以理解作者意图（`plugins/code-review/commands/code-review.md:53,55`）。

### 4) 关键命令面（外部接口）

- `gh pr view`, `gh pr list`, `gh pr diff`, `gh pr comment`（`plugins/code-review/commands/code-review.md:2,18,65`）。
- `gh issue view`, `gh issue list`, `gh search`（`plugins/code-review/commands/code-review.md:2`）。
- `mcp__github_inline_comment__create_inline_comment`（`plugins/code-review/commands/code-review.md:2,71`）。

## 关键代码路径与文件引用

核心文件与关系如下：

- 命令协议主文件
  - `plugins/code-review/commands/code-review.md:1-109`
  - 定义 `/code-review` 全流程、模型分工、过滤规则、评论协议。

- 插件文档（用户预期与操作说明）
  - `plugins/code-review/README.md:11-33,50-57,143-147,216-250`
  - 说明命令用途、参数、运维前提、阈值/技术细节。

- 插件元数据
  - `plugins/code-review/.claude-plugin/plugin.json:1-9`
  - 定义插件基础身份信息。

- marketplace 注册（上游发现入口）
  - `.claude-plugin/marketplace.json:29-37`
  - 将 `code-review` 指向 `./plugins/code-review`。

- 插件目录总览文档（跨插件视角）
  - `plugins/README.md:13-27`
  - 列出 `/code-review` 在插件总表中的角色与简介。

- 命令发现机制与命名约定（平台规则文档）
  - `plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-309,341-345`
  - 说明 `code-review.md -> /code-review` 以及命令扫描机制。

测试与脚本现状（针对 `plugins/code-review` 目录）：

- 目录中仅有 3 个文件：`README.md`、`commands/code-review.md`、`.claude-plugin/plugin.json`，未包含本插件专属 `tests/`、`scripts/`、`hooks/` 或 `.mcp.json`。
- 这意味着该命令当前主要依赖“提示词协议正确性 + 运行时工具可用性”，而不是仓库内自动化测试保障。

## 依赖与外部交互

### 内部依赖

- 插件发现与加载：依赖 marketplace 注册和标准目录扫描（`.claude-plugin/marketplace.json:29-37`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-345`）。
- 插件文档一致性：命令行为需与 `plugins/code-review/README.md` 和 `plugins/README.md` 描述一致（`plugins/code-review/README.md:15-25`，`plugins/README.md:17`）。
- 规则输入：依赖仓库中 `CLAUDE.md` 分布和质量；命令只定义了采集与作用域规则（`plugins/code-review/commands/code-review.md:24-27,33`）。

### 外部依赖

- GitHub CLI 环境
  - 需要 `gh` 已安装且认证可用（`plugins/code-review/README.md:145-147,194-201`）。
  - 命令协议要求通过 `gh` 读取与写评论（`plugins/code-review/commands/code-review.md:2,65,90`）。

- MCP GitHub inline 评论能力
  - 需要 `mcp__github_inline_comment__create_inline_comment` 能调用（`plugins/code-review/commands/code-review.md:2,71`）。

- GitHub 仓库上下文
  - 评论链接要求“目标仓库名 + full SHA + 行号范围”，否则 Markdown 预览不正确（`plugins/code-review/commands/code-review.md:103-109`）。

### 运行时输入/输出边界

- 输入
  - PR 上下文信息（状态、标题、描述、diff、评论历史）。
  - 可选参数 `--comment`（控制是否写回 PR）（`plugins/code-review/commands/code-review.md:63-67`）。

- 输出
  - 终端摘要（必有，至少“无问题固定文案”）。
  - 可选 GitHub 评论（摘要评论或 inline 评论）（`plugins/code-review/commands/code-review.md:65,71,93-101`）。

## 风险、边界与改进建议

### 风险与边界

1. 文档与协议存在漂移风险
- `README.md` 将流程描述为“0-100 打分 + 80 阈值过滤 + history analyzer”（`plugins/code-review/README.md:22-25,78-83,233-243`），但命令协议本体是“复核通过/不通过”过滤，且 Agent #4 未强制 `git blame/history`（`plugins/code-review/commands/code-review.md:38-40,55-57`）。
- `plugins/README.md` 写“5 parallel Sonnet agents”（`plugins/README.md:17`），与命令中“4 个并行审查 agent，含 Opus”不一致（`plugins/code-review/commands/code-review.md:30-39`）。

2. 错误处理边界较弱
- 协议前提写明“工具都可用，不做探测式调用”（`plugins/code-review/commands/code-review.md:8-10`），在真实环境里一旦 `gh` 或 MCP 权限异常，缺少降级与恢复策略。

3. 高信号策略可能带来漏报
- 明确禁止依赖上下文的潜在问题（`plugins/code-review/commands/code-review.md:48-49`），可显著降噪，但会漏掉“仅在上下文下可证实”的真实缺陷。

4. 无仓库内自动化回归测试
- 当前插件目录无命令行为测试，主要靠提示词维护；当流程文本被修改时，缺少自动校验来阻断回归。

5. 规则依赖外部文件布局
- 合规检查质量直接取决于 `CLAUDE.md` 可用性与粒度（`plugins/code-review/commands/code-review.md:24-27,33`；`plugins/code-review/README.md:100,147`）。仓库无 `CLAUDE.md` 时，规则类问题检出能力会弱化。

### 改进建议

1. 先做文档对齐
- 统一 `plugins/code-review/README.md`、`plugins/README.md` 与命令协议的“agent 数量/模型角色/是否评分阈值”描述，避免用户预期偏差。

2. 把“高信号过滤”从文字协议升级为结构化输出约束
- 要求初审与复核子代理输出固定字段（`type`、`evidence`、`confidence`、`validated`、`scope_path`），并在 step 6 用显式条件过滤，减少口径漂移。

3. 增加失败降级分支
- 在协议中补充：`gh`/MCP 调用失败时至少输出“终端可读总结 + 未回写原因”，避免静默失败。

4. 为评论去重补充可执行判重键
- 当前只写“唯一问题只评论一次”（`plugins/code-review/commands/code-review.md:77`），建议明确判重键（文件路径 + 行区间 + 规则/错误类型 + 归一化描述）。

5. 为命令协议建立最小回归测试工件
- 新增轻量脚本或用例清单（例如样例 PR 元数据 + 预期输出分支），至少覆盖：
  - skip 分支
  - 无问题且 `--comment`
  - 有问题 inline 评论
  - 链接格式校验（full SHA + `#Lx-Ly`）

