# FILE `plugins/ralph-wiggum/commands/help.md` 研究文档

## 场景与职责

`/help`（此处是插件内 `commands/help.md`）承担 Ralph Wiggum 插件的“会话内解释层”职责：当用户不知道该插件方法论、命令参数或停止条件时，模型可直接复述该文档，避免跳出当前会话再查 README。

它在体系中的定位：

- 调用方：用户触发帮助语义（命令帮助或上下文引导）。
- 被调用方：无直接脚本调用；该文件本质是知识提示模板。
- 旁路依赖：其说明内容必须和真实执行链（`commands/ralph-loop.md` -> `scripts/setup-ralph-loop.sh` -> `hooks/stop-hook.sh`）一致，否则会产生“文档-实现偏差”。

该文件不是执行入口，但会直接影响用户对循环控制、退出机制、风险边界的认知，因此属于“高影响文档控制面”。

## 功能点目的

1. 解释 Ralph 技术核心：同一 prompt 多轮输入，模型通过文件/历史观察自身上轮产物而迭代优化（`plugins/ralph-wiggum/commands/help.md:20-30`）。
2. 给出命令速查：`/ralph-loop` 和 `/cancel-ralph` 的用途、参数、示例（`plugins/ralph-wiggum/commands/help.md:34-71`）。
3. 说明关键停止协议：completion promise 必须以 `<promise>...</promise>` 输出，且与命令参数精确匹配（`plugins/ralph-wiggum/commands/help.md:76-85`，对应实现 `plugins/ralph-wiggum/hooks/stop-hook.sh:114-127`）。
4. 设定使用边界：推荐场景与不推荐场景，降低误用概率（`plugins/ralph-wiggum/commands/help.md:109-121`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 文档命令协议

frontmatter 仅声明 `description`（`plugins/ralph-wiggum/commands/help.md:1-3`），没有 `allowed-tools`，表示该命令不依赖外部工具执行，属于纯文本提示。

这与插件开发文档的 frontmatter 语义一致：`description` 主要用于 `/help` 展示，`allowed-tools` 仅在需要显式限制工具时才声明（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:17-31,52-76`）。

### 2) 内容结构与实现链路映射

文档中的三段关键说明映射到真实实现：

1. “Stop hook intercepts and feeds the same prompt again”（`help.md:26-27`）
   - 对应 Hook 阻断输出 `decision: block` + `reason: <prompt>`（`plugins/ralph-wiggum/hooks/stop-hook.sh:165-174`）。

2. “completion promise 标签作为停止信号”（`help.md:78-85`）
   - 对应 Hook 用 Perl 提取 `<promise>`，并与 `completion_promise` 字段做字面值匹配（`plugins/ralph-wiggum/hooks/stop-hook.sh:116-127`）。

3. “max iterations 可自动停止”（`help.md:45,54`）
   - 对应 Hook 在 `iteration >= max_iterations` 时删除状态并放行退出（`plugins/ralph-wiggum/hooks/stop-hook.sh:50-55`）。

### 3) 与状态文件的数据契约

帮助文档提到状态文件，但路径写作 `.claude/.ralph-loop.local.md`（`help.md:49,69`）；真实代码统一使用 `.claude/ralph-loop.local.md`：

- 写入：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140`
- 读取/删除：`plugins/ralph-wiggum/hooks/stop-hook.sh:13`、`plugins/ralph-wiggum/commands/cancel-ralph.md:3,11,16-17`

这属于“文档实现不一致”的具体技术问题，会影响人工排障操作。

### 4) 命令示例的行为约束

`help.md` 示例强调 `--completion-promise` 与 `<promise>` 搭配（`help.md:98-100`），而 `setup-ralph-loop.sh` 在启动后也会反复输出“不得虚假承诺”约束（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:178-202`），两者共同构成行为治理链。

## 关键代码路径与文件引用

- 目标文件：`plugins/ralph-wiggum/commands/help.md:1-126`
- 命令入口说明（对照）：`plugins/ralph-wiggum/README.md:50-69`
- 启动脚本（参数与状态文件）：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:13-150`
- Stop Hook（迭代、停止、回灌）：`plugins/ralph-wiggum/hooks/stop-hook.sh:50-55,114-127,165-174`
- 取消命令（人工停止）：`plugins/ralph-wiggum/commands/cancel-ralph.md:11-18`
- Hook 注册配置：`plugins/ralph-wiggum/hooks/hooks.json:3-13`
- 插件元信息：`plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`

测试/脚本上下文：

- `commands/help.md` 自身无测试。
- 相关可执行链路脚本语法检查通过：
  - `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`
  - `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh`

## 依赖与外部交互

1. 文档依赖：依赖 `README.md` 与实际脚本逻辑的一致性。
2. 协议依赖：依赖 Stop Hook JSON 决策协议（`decision/reason/systemMessage`）。
3. 文件系统语义依赖：依赖状态文件路径、字段命名不变化。
4. 外部交互：该文件无网络调用，但会影响用户如何与脚本、hook、状态文件交互。

## 风险、边界与改进建议

1. 状态文件路径错误（高优先级）：文档写 `.claude/.ralph-loop.local.md`，实现是 `.claude/ralph-loop.local.md`。
建议：立即修正文档路径并补“路径以 setup/cancel/stop-hook 为准”的一致性注记。

2. 命令可发现性歧义：文件名为 `help.md`，README 同时提示“Run `/help`”。用户可能不确定调用的是系统帮助还是插件帮助。
建议：在文档开头增加一句“这是 ralph-wiggum 插件帮助内容”，并给出命令触发上下文示例。

3. 对“无限循环”风险提醒强度不足：虽提到“Without promise or max iterations runs infinitely”（`help.md:84`），但未强调 `--max-iterations` 的默认建议值策略。
建议：增加“默认建议始终设置 `--max-iterations`（如 10/20/50）”的操作性指导。

4. 文案与执行行为耦合较强：若后续修改 Hook 的 promise 解析规则（例如支持多个 tag），help 文档会立刻过时。
建议：在文档中引用“实现以 stop-hook.sh 为准”，并在 CI 加“路径/关键字段一致性检查”。
