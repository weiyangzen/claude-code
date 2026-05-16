# plugins/code-review/README.md 研究

## 场景与职责

`plugins/code-review/README.md` 是 `code-review` 插件的用户说明层，职责不是执行审查，而是把 `/code-review` 的能力、用法、参数、运行前提、故障排查和可配置点传达给使用者。

在插件体系中的定位：
- 仓库根 README 仅声明“有 plugins 目录可用”，不展开细节，细节下沉到插件文档：`README.md:48-50`
- 插件总览把 `code-review` 作为一个生产力插件暴露给用户：`plugins/README.md:17`
- marketplace 通过 `source: ./plugins/code-review` 暴露此插件目录：`.claude-plugin/marketplace.json:29-37`
- 插件元信息由 `plugin.json` 提供，README 承接“人类可读说明”：`plugins/code-review/.claude-plugin/plugin.json:1-9`

与同目录对象的职责边界：
- `README.md`：用户文档与操作约定（本研究对象）
- `commands/code-review.md`：真正执行协议（步骤、工具白名单、评论写入规则）
- `.claude-plugin/plugin.json`：名称、版本、作者、描述等元数据

## 功能点目的

### 1. 解释命令价值与输出模式
README 首段和 Overview 明确该插件目标是“多 agent 并发审查 + 降噪过滤”，并声明输出有两种模式：默认终端输出，`--comment` 写回 PR。证据：`plugins/code-review/README.md:3,7,25,29-33`。

### 2. 让用户理解“何时审查/何时跳过”
文档把“自动跳过”条件（closed/draft/trivial/already-reviewed）写成显式预期，减少误操作疑惑。证据：`plugins/code-review/README.md:16,56,113-117,175-179`。

### 3. 为团队规范联动提供入口
README 明确 `CLAUDE.md` 是合规检查输入，指导用户维护规则文档以提升审查质量。证据：`plugins/code-review/README.md:17,53,100,104,147,205,208`。

### 4. 提供运维化故障排查手册
README 按常见失败场景给出操作化排查项（耗时、无评论、链接格式、`gh` 认证失败），降低使用门槛。证据：`plugins/code-review/README.md:149-201`。

### 5. 给出“可调整点”
README 将“阈值调整”和“审查焦点扩展”指向 `commands/code-review.md`，把可定制能力暴露给维护者。证据：`plugins/code-review/README.md:214-229`。

## 具体技术实现（关键流程/数据结构/协议/命令）

README 描述的是“实现意图”；实际执行由 `commands/code-review.md` 驱动。两者合并后可还原端到端技术实现。

### A. 关键流程（执行协议）
1. 预检查阶段：子 agent 判断 PR 是否应跳过（closed/draft/trivial/已被 Claude 评论）。`plugins/code-review/commands/code-review.md:14-22`
2. 规范发现阶段：收集 root + 改动目录相关 `CLAUDE.md` 路径。`plugins/code-review/commands/code-review.md:24-27`
3. 变更摘要阶段：sonnet agent 先给出 PR changes summary。`plugins/code-review/commands/code-review.md:28`
4. 并发发现阶段：4 个 agent 并发找问题（2 个合规，2 个 bug/logic）。`plugins/code-review/commands/code-review.md:30-40`
5. 并发复核阶段：对问题二次验证，bug/logic 用 Opus，规则问题用 Sonnet。`plugins/code-review/commands/code-review.md:55`
6. 结果过滤阶段：仅保留通过验证的问题。`plugins/code-review/commands/code-review.md:57`
7. 输出阶段：终端总结；若带 `--comment`，按“无问题摘要评论/有问题 inline 评论”分支执行。`plugins/code-review/commands/code-review.md:59-67,71-77,93-101`

### B. 关键数据结构（隐式）
该插件没有独立代码对象定义，数据结构体现在提示词契约中：
- `IssueCandidate`：步骤 4 的问题条目，至少含“问题描述 + 标记原因（bug/CLAUDE.md adherence）”。`plugins/code-review/commands/code-review.md:30`
- `ValidatedIssue`：步骤 5 通过验证的问题子集。`plugins/code-review/commands/code-review.md:55-57`
- `CommentPlan`：步骤 8 内部确认后再发送的评论列表。`plugins/code-review/commands/code-review.md:69`
- `PRReviewOutput`：步骤 7 的终端摘要或“无问题固定文案”。`plugins/code-review/commands/code-review.md:59-61,97-99`

### C. 协议与命令约束
- 工具白名单协议：frontmatter 将工具限定在 `gh issue/pr/search` 子命令和 `mcp__github_inline_comment__create_inline_comment`。`plugins/code-review/commands/code-review.md:1-4`
- 调用纪律协议：禁止 exploratory tool calls，且仅在必要时调用工具。`plugins/code-review/commands/code-review.md:8-10`
- 评论格式协议：inline comment 要求 `confirmed: true`、禁止重复、建议块有边界条件。`plugins/code-review/commands/code-review.md:71-77`
- 链接协议：必须 full SHA + `#Lx-Ly`，并包含上下文行。`plugins/code-review/commands/code-review.md:103-109`

### D. README 描述与执行协议的一致性观察
- README 宣称“0-100 打分 + 80 阈值过滤”。`plugins/code-review/README.md:23-25,78-83,216-221,239-243`
- 命令协议明确的是“验证通过即保留”，未出现显式数值打分字段或阈值判断语句。`plugins/code-review/commands/code-review.md:55-57`
- README 宣称第 4 个 agent 做 git blame/history 分析。`plugins/code-review/README.md:22,55,236,249`
- 命令中第 4 个 agent被定义为“introduced code 问题扫描”，未要求 `git blame`。`plugins/code-review/commands/code-review.md:38-40`

## 关键代码路径与文件引用

核心路径（从发现到执行再到文档）：
1. marketplace 注册入口：`.claude-plugin/marketplace.json:29-37`
2. 插件元数据：`plugins/code-review/.claude-plugin/plugin.json:1-9`
3. 仓库插件总览：`plugins/README.md:17`
4. 目标文档（用户说明层）：`plugins/code-review/README.md:1-258`
5. 命令执行协议（实现层）：`plugins/code-review/commands/code-review.md:1-109`
6. 命令命名约定（`code-review.md -> /code-review`）：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-309`

调用关系（README 相关）：
- 调用方（上游）：用户通过 `/code-review` 触发命令；仓库文档与插件总览将用户导向该插件。`README.md:48-50`，`plugins/README.md:17`
- 被调用方（下游）：`gh` CLI、MCP inline comment 工具、并行子 agent（haiku/sonnet/opus）执行实际审查与评论发布。`plugins/code-review/commands/code-review.md:2,14-40,55,65,71`

配置/测试/脚本/文档上下文：
- 配置：`plugin.json` 与 command frontmatter (`allowed-tools`, `description`)。`plugins/code-review/.claude-plugin/plugin.json:1-9`，`plugins/code-review/commands/code-review.md:1-4`
- 测试：`plugins/code-review` 目录下未发现 test/spec 文件（目录仅 3 文件：README/command/plugin.json）。
- 脚本：`plugins/code-review` 目录下未发现 `scripts/`。
- 文档：目标 README + 仓库根 README + `plugins/README` 形成三级文档链路。`README.md:48-50`，`plugins/README.md:17,49-61`，`plugins/code-review/README.md:1-258`

## 依赖与外部交互

### 内部依赖
- 插件发现与加载：依赖 marketplace 配置中的 source 路径。`.claude-plugin/marketplace.json:29-37`
- 命令执行：依赖 `plugins/code-review/commands/code-review.md` 的步骤协议。`plugins/code-review/commands/code-review.md:12-77`
- 规则输入：依赖仓库中可能存在的 `CLAUDE.md` 文件布局与质量。`plugins/code-review/commands/code-review.md:24-27,33`

### 外部交互
- GitHub PR/Issue 读取与评论：通过 `gh`（如 `pr view/diff/comment` 等）。`plugins/code-review/commands/code-review.md:2,18,65,90`
- 行内评论写回：通过 `mcp__github_inline_comment__create_inline_comment`。`plugins/code-review/commands/code-review.md:2,71`
- 运行前提：`gh` 已安装且鉴权、仓库具备 GitHub remote。`plugins/code-review/README.md:145-147,194-201`

### 交互结果面
- 不带 `--comment`：结果只输出到终端。`plugins/code-review/commands/code-review.md:63`
- 带 `--comment` 且无问题：写一条固定模板摘要评论。`plugins/code-review/commands/code-review.md:65,93-101`
- 带 `--comment` 且有问题：逐条 inline comments（去重）。`plugins/code-review/commands/code-review.md:67,71-77`

## 风险、边界与改进建议

### 风险与边界
1. 文档承诺与执行协议存在偏差
- README 的“显式打分阈值模型”与命令的“验证通过过滤模型”不完全一致，容易造成用户对可配置阈值的预期偏差。证据见前述一致性观察。

2. 历史上下文能力表述可能过度
- README 多处强调 git blame/history，但命令步骤没有对应强制动作，可能导致“宣传能力 > 实际协议”问题。`plugins/code-review/README.md:22,55,236,249` vs `plugins/code-review/commands/code-review.md:38-40`

3. 外部依赖单点故障
- `gh` 鉴权、权限、网络或 MCP 能力失败会影响完整闭环，尤其 `--comment` 模式。`plugins/code-review/README.md:194-201`，`plugins/code-review/commands/code-review.md:71`

4. 无自动化回归
- 该插件是提示词协议实现，缺少自动化测试/回归脚本，后续 prompt 调整存在行为漂移风险。

5. 高信号策略的覆盖边界
- 明确排除 style/主观建议等噪音（优点），但也可能漏掉“重要但需上下文验证”的问题。`plugins/code-review/commands/code-review.md:41-51`

### 改进建议
1. 统一 README 与命令协议
- 方案 A：在命令中补上结构化打分字段与阈值分支。
- 方案 B：在 README 中将“评分过滤”改写为“并发复核后的高信号过滤”。

2. 显式化历史分析能力
- 若要保留 history/blame 能力，应在命令中增加对应步骤、触发条件和输出格式。

3. 增加最小回归检查
- 新增轻量校验脚本：检查命令步骤关键节点存在、`allowed-tools` 与说明一致、无问题评论模板与链接格式规则存在。

4. 提升可观测性
- 终端摘要增加结构化统计（发现数/复核通过数/跳过原因），便于 CI 或日志系统消费。

5. 收敛文档重复与冲突
- 对 `README.md`、`plugins/README.md` 的 agent 数量/模型角色描述做定期一致性核对，避免跨文档漂移（例如 `plugins/README.md` 当前写“5 parallel Sonnet agents”，而命令步骤是 4 个并行 agent 且包含 Opus）。`plugins/README.md:17`，`plugins/code-review/commands/code-review.md:30-39`
