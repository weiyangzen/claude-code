# plugins/code-review/commands 目录研究（DIR）

## 场景与职责

`plugins/code-review/commands` 是 `code-review` 插件的命令协议层目录，当前仅包含一个命令文件：
- `plugins/code-review/commands/code-review.md`

该目录在整体链路中的职责是：把“PR 审查”定义为可执行的 slash command 协议（而非源码实现），并将审查流程拆分为多阶段 agent 编排 + GitHub 交互动作。

上游装配关系（调用方/入口）：
1. marketplace 将 `code-review` 插件注册到可发现列表，并指向插件根目录 `./plugins/code-review`（`.claude-plugin/marketplace.json:29-37`）。
2. 插件 manifest 提供元数据，使插件可被识别（`plugins/code-review/.claude-plugin/plugin.json:1-9`）。
3. 命令目录采用默认自动发现机制，`commands/*.md` 自动成为 slash command（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:112-115,343-345`）。
4. 文件名映射命令名，`code-review.md -> /code-review`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-309`）。

下游执行关系（被调用方/外部动作）：
- 通过 `gh pr view/list/diff/comment` 和 `gh issue view/list/search` 读取 PR 与发布摘要评论（`plugins/code-review/commands/code-review.md:2,18,65,90`）。
- 通过 `mcp__github_inline_comment__create_inline_comment` 发布行内评论（`plugins/code-review/commands/code-review.md:2,71`）。
- 通过并行子 agent 执行审查、复核与过滤（`plugins/code-review/commands/code-review.md:14-57`）。

职责边界：
- 本目录不包含可执行程序（TS/Python/Shell），主要是 prompt contract（自然语言协议）层。
- 本目录不承载测试用例或脚手架脚本；质量依赖协议约束与运行时 agent 执行一致性。

## 功能点目的

### 1) 审查前门禁（是否应跳过）
目的：在审查前快速过滤明显不需要 review 的 PR，减少无效计算和噪音输出。
- 关闭/草稿/无需审查/已被 Claude 评论过 -> 直接停止（`code-review.md:14-22`）。
- 要求检查评论历史使用 `gh pr view <PR> --comments`（`code-review.md:18`）。

### 2) 规则上下文收集（CLAUDE.md 定位）
目的：把规则检查限定在相关路径，避免跨目录误判。
- 只返回相关 `CLAUDE.md` 路径（不拉全文），并覆盖 root 与改动目录（`code-review.md:24-27`）。
- 路径作用域规则：仅使用与目标文件同路径或父路径的 `CLAUDE.md`（`code-review.md:33`）。

### 3) 多代理并行审查
目的：并行拆分审查维度，提高覆盖率并降低单代理偏差。
- 2 个 sonnet 代理做规范合规审查（`code-review.md:32-33`）。
- 2 个 opus 代理做 bug/逻辑问题扫描（`code-review.md:35-39`）。
- 每个子代理必须拿到 PR 标题与描述，以便基于作者意图判断（`code-review.md:53`）。

### 4) 高信号过滤与二次验证
目的：控制误报，把“看起来像问题”收敛到“高置信真实问题”。
- 只接受高信号问题：编译/解析失败、确定性逻辑错误、可引用规则的明确违规（`code-review.md:41-45`）。
- 明确排除项：风格、主观建议、依赖输入状态的不确定问题（`code-review.md:46-50`）。
- 对 bug 与规则问题再起子代理复核，未通过复核则过滤（`code-review.md:55-57`）。

### 5) 结果分支与评论策略
目的：将“本地审查输出”与“PR 写回”分离，减少误操作。
- 无 `--comment`：仅终端输出（`code-review.md:63`）。
- 有 `--comment` 且无问题：发布固定模板摘要评论（`code-review.md:65,93-101`）。
- 有 `--comment` 且有问题：逐条发布 inline comments（`code-review.md:67-77`）。

### 6) 评论规范与可落地性约束
目的：确保评论可以在 GitHub 渲染并支持直接提交修复。
- inline comment 必须 `confirmed: true`（`code-review.md:71`）。
- 小而闭环的问题可附 committable suggestion；大改动禁止 suggestion block（`code-review.md:73-75`）。
- 每个唯一问题只能发一次评论，避免重复骚扰（`code-review.md:77`）。
- 链接必须使用 full SHA + `#Lx-Ly`，并含前后文（`code-review.md:103-109`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（9 步状态机）

1. 初始化与工具约束
- frontmatter 暴露 `description` 与 `allowed-tools`，限制命令执行面的工具集合（`code-review.md:1-4`）。
- `allowed-tools` 的一般语义是“命令可用工具白名单”（`plugins/plugin-dev/skills/command-development/SKILL.md:128-146`）。

2. Step 1：跳过判定（haiku）
- 在最小代价下终止无需审查的 PR（`code-review.md:14-22`）。

3. Step 2：规则文件路径收集（haiku）
- 只收集路径，避免过早引入超量上下文（`code-review.md:24-27`）。

4. Step 3：PR 变更摘要（sonnet）
- 单独提炼变更概况，为后续并行代理共享背景（`code-review.md:28`）。

5. Step 4：4 路并行初审
- 2 路规则合规 + 2 路 bug/逻辑扫描，形成 issue 列表（`code-review.md:30-40`）。

6. Step 5：逐问题复核
- 按问题类型选择模型复核（bug/逻辑用 Opus，规则问题用 sonnet），并带上 PR 标题/描述（`code-review.md:55`）。

7. Step 6：仅保留复核通过问题
- 形成“高信号最终集”（`code-review.md:57`）。

8. Step 7：终端输出与 `--comment` 分支
- 无 `--comment` 到此结束；有 `--comment` 决定摘要评论或进入 inline 评论阶段（`code-review.md:59-67`）。

9. Step 8/9：评论准备与落地
- 先本地整理“准备发送的评论列表”再真正发送（`code-review.md:69`）。
- 使用 MCP API 逐条发送确认后的 inline 评论（`code-review.md:71-77`）。

### B. 隐含数据结构（命令协议层）

虽然文件中没有显式 JSON Schema，但流程隐含以下数据对象：
1. `PRMeta`: PR 编号、标题、描述、状态、是否 draft、是否已有 Claude 评论。来源于 `gh pr view/list`（`code-review.md:2,18`）。
2. `GuidelinePaths`: root + 改动目录相关 `CLAUDE.md` 路径集合（`code-review.md:24-27,33`）。
3. `IssueCandidate`: 初审代理输出的问题项（描述 + 标记原因，如 bug / CLAUDE.md adherence）（`code-review.md:30`）。
4. `ValidatedIssue`: 经 Step 5 复核通过的问题项（`code-review.md:55-57`）。
5. `CommentPlan`: Step 8 本地拟发布评论集合（`code-review.md:69`）。

### C. 协议约束（Prompt Contract）

1. 工具调用策略
- 明确要求“工具都可用，不做探测式调用；工具调用必须有明确目的”（`code-review.md:8-10`）。

2. 高信号定义
- 只收录确定性错误与明确规则违规（`code-review.md:41-45`），并禁止风格型噪音（`code-review.md:46-51`）。

3. 假阳性基线清单
- 预置“不要报”的类型（预存问题、linter 可抓问题、被显式静默的问题等）（`code-review.md:79-86`）。

4. 输出协议
- 无问题固定输出：`No issues found. Checked for bugs and CLAUDE.md compliance.`（`code-review.md:61,99`）。
- 有 `--comment` 且无问题时，要求按模板发 PR 评论（`code-review.md:93-101`）。

5. 链接协议
- 评论链接必须 full SHA，禁止 `$(git rev-parse HEAD)` 这种运行时占位（`code-review.md:103-106`）。

### D. 关键命令与外部接口

1. GitHub CLI（`gh`）
- PR 读取：`gh pr view`, `gh pr list`, `gh pr diff`（白名单见 `code-review.md:2`，评论历史见 `code-review.md:18`）。
- Issue 查询：`gh issue view/list`, `gh search`（`code-review.md:2`）。
- 摘要评论发布：`gh pr comment`（`code-review.md:2,65`）。

2. MCP Inline Comment
- `mcp__github_inline_comment__create_inline_comment` 用于精确到行的评论写回（`code-review.md:2,71`）。

### E. 文档承诺与命令协议的一致性观察

`plugins/code-review/README.md` 将该命令描述为“置信度评分 + 阈值 80 过滤”的工作流（`plugins/code-review/README.md:23-25,52,78-83,165,216-221,239-243`），并声明第 4 个代理会做“历史上下文/git blame”分析（`plugins/code-review/README.md:22,55,236`）。

而实际命令协议 `code-review.md` 当前表现为：
- 以“是否通过 Step 5 复核”做最终过滤（`code-review.md:55-57`），未出现显式 0-100 打分字段与阈值判定语句。
- 第 4 个代理描述为“introduced code 中的问题扫描”（`code-review.md:38-39`），未显式要求 git blame/history。

这代表“文档解释层”与“命令执行协议层”存在语义漂移风险。

## 关键代码路径与文件引用

### 目标对象（DIR）
- `plugins/code-review/commands/code-review.md:1-109`

### 上游调用与装配路径
- `.claude-plugin/marketplace.json:29-37`：注册插件与 source 路径。
- `plugins/code-review/.claude-plugin/plugin.json:1-9`：插件元数据。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:112-115,307-309,343-345`：`commands/*.md` 自动发现与文件名到 slash command 映射。
- `plugins/README.md:17`：仓库级目录说明该插件提供 `/code-review`。

### 下游被调用路径
- GitHub CLI 调用集合：`plugins/code-review/commands/code-review.md:2,18,65,90`。
- MCP inline 评论：`plugins/code-review/commands/code-review.md:2,71`。
- 评论链接格式契约：`plugins/code-review/commands/code-review.md:103-109`。

### 相关文档路径
- 插件用户文档：`plugins/code-review/README.md:11-258`。
- 插件总览索引：`plugins/README.md:1-63`。

### 测试与脚本路径现状
- `plugins/code-review/commands/` 当前仅 `code-review.md`，无测试、无脚本。
- `find plugins/code-review -maxdepth 4 -type f \( -iname '*test*' -o -iname '*spec*' -o -path '*/scripts/*' \)` 返回空。

## 依赖与外部交互

### 内部依赖
1. 插件发现链路依赖：marketplace + plugin manifest + commands auto-discovery。
2. 规则依赖：仓库内 `CLAUDE.md` 布局质量直接影响审查信号（`code-review.md:24-27,33`；`plugins/code-review/README.md:100,147`）。
3. 命令参数分支依赖：`--comment` 决定是否写回 GitHub（`code-review.md:63-67`）。

### 外部依赖
1. GitHub CLI 认证与可用性（`plugins/code-review/README.md:145-147,194-201`）。
2. GitHub API/MCP 服务可用性（由 `gh` 和 `mcp__github_inline_comment__create_inline_comment` 执行）。
3. 仓库远程与 PR 上下文存在性（若不在有效 PR 语境，命令无法完成目标）。

### 交互协议与输出介质
1. 本地终端输出：审查摘要（`code-review.md:59-63`）。
2. PR 摘要评论：`gh pr comment`（`code-review.md:65,93-101`）。
3. PR 行内评论：MCP inline comment（`code-review.md:71-77`）。

### 配置项与可调节点
1. 命令 frontmatter：`allowed-tools`、`description`（`code-review.md:1-4`）。
2. README 指导可修改阈值/审查焦点，但阈值语句在命令正文未显式呈现（`plugins/code-review/README.md:216-229`）。

## 风险、边界与改进建议

### 风险

1. 文档-协议漂移风险（高）
- README 叙述有“0-100 评分 + 80 阈值 + history analyzer”（`README.md:23-25,55,236,239-243`），命令正文主要是“复核通过即保留”（`code-review.md:55-57`）且未显式 history 任务。
- 影响：用户预期与实际执行可能不一致，出现“为何未按 80 阈值过滤/为何无历史分析”的认知偏差。

2. 工具可用性假设过强（中）
- 命令显式要求“不要探测工具，假设工具可用”（`code-review.md:8-10`）。
- 影响：在 `gh` 未登录、MCP 不可用或权限缺失时，缺少降级策略会导致流程中断。

3. 流程高度依赖子代理质量（中）
- 关键判断（初审/复核）都在子代理，且无结构化 schema 约束输出（`code-review.md:30,55`）。
- 影响：边界案例可能出现漏报/误报波动。

4. 评论写回风险（中）
- Step 9 直接写 GitHub inline comments（`code-review.md:71-77`）；如果问题去重或定位不稳定，会影响 PR 讨论质量。

### 边界

1. 本目录是“命令协议层”，不是“可执行代码层”；不存在编译、单测覆盖率等传统工程指标。
2. 它不负责插件安装/启用实现，仅消费插件系统自动发现机制。
3. 它不负责规则内容本身，`CLAUDE.md` 的质量与层级策略属于仓库治理范畴。

### 改进建议

1. 统一 README 与命令协议
- 二选一：
  - 在 `code-review.md` 增加显式评分字段与阈值判定步骤；或
  - 更新 README，改为“复核通过过滤”叙述。
- 同时明确第 4 代理是否包含 `git blame/history`。

2. 增加失败降级策略
- 在命令中补充：`gh`/MCP 失败时如何退化为“仅终端输出 + 不写回 PR”，并显式给出错误处理模板。

3. 引入结构化 issue schema
- 约束子代理输出固定字段（`type`, `evidence`, `location`, `confidence/validated`），提升 Step 5 复核与 Step 9 评论的一致性。

4. 增加最小回归样例
- 为 `code-review` 插件补充静态回归文档或脚本（例如输入固定 PR 元信息与 diff 片段，校验输出分支），降低 prompt 变更回归风险。

5. 增强去重策略
- 在 Step 8 显式定义“唯一问题判定键”（文件+行+规则ID/错误类型），避免多代理并行导致重复评论。
