# FILE `plugins/ralph-wiggum/commands/cancel-ralph.md` 研究文档

## 场景与职责

`/cancel-ralph` 是 Ralph Wiggum 插件的人工终止入口，作用是在会话级循环已经被激活后，提供一个可控“刹车”动作。它本身不执行循环控制算法，而是通过受限工具集合直接操作状态文件 `.claude/ralph-loop.local.md`，从而影响 Stop Hook 的后续行为。

在调用链上的位置：

- 调用方：用户在 Claude Code 中执行 `/cancel-ralph`（命令文件由插件命令扫描机制加载，见 `plugins/ralph-wiggum/commands/cancel-ralph.md:1-5`）。
- 被调用方：状态文件读写动作（`Read(.claude/ralph-loop.local.md)`、`Bash(test -f ...)`、`Bash(rm ...)`），以及间接影响 `plugins/ralph-wiggum/hooks/stop-hook.sh:13-18` 的“是否存在 active 状态文件”判断。

该命令承担的职责边界非常明确：

1. 判定是否有激活中的 Ralph 状态。
2. 有状态时读取当前迭代号并删除状态文件。
3. 无状态时返回“未发现活动循环”。

## 功能点目的

1. 提供人工中断能力，避免仅依赖 `--max-iterations` 或 completion promise 才能退出（与 `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:166-167` 的“默认无限循环风险”形成补位）。
2. 让中断结果可观测：删除前读取 `iteration`，向用户回报“停止时已运行到第 N 轮”（`plugins/ralph-wiggum/commands/cancel-ralph.md:16-18`）。
3. 通过最小权限执行中断，降低误操作面：前置检查、精确路径、单文件删除。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 命令 frontmatter 与权限模型

frontmatter 中定义：

- `description`：帮助系统中的说明文案（`plugins/ralph-wiggum/commands/cancel-ralph.md:2`）。
- `allowed-tools`：仅允许三类动作：
  - `Bash(test -f .claude/ralph-loop.local.md:*)`
  - `Bash(rm .claude/ralph-loop.local.md)`
  - `Read(.claude/ralph-loop.local.md)`
  （见 `plugins/ralph-wiggum/commands/cancel-ralph.md:3`）
- `hide-from-slash-command-tool: "true"`：命令对自动 slash 工具隐藏（`plugins/ralph-wiggum/commands/cancel-ralph.md:4`）。

这个权限边界体现“最小化工具面”：该命令无法直接执行任意 shell，只能围绕单一状态文件完成存在性检查、读取和删除。

### 2) 核心流程

命令正文定义的执行流程（`plugins/ralph-wiggum/commands/cancel-ralph.md:9-18`）：

1. `test -f` 检测状态文件是否存在。
2. 若不存在，返回 “No active Ralph loop found.”。
3. 若存在：
   - 读取状态文件 frontmatter 的 `iteration:` 字段。
   - `rm .claude/ralph-loop.local.md` 删除状态文件。
   - 报告 “Cancelled Ralph loop (was at iteration N)”。

### 3) 关键数据结构

该命令依赖 `setup-ralph-loop.sh` 写入的状态文件结构（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-149`）：

```markdown
---
active: true
iteration: 1
max_iterations: <number>
completion_promise: <string|null>
started_at: "<UTC timestamp>"
---

<prompt body>
```

`/cancel-ralph` 只消费其中 `iteration`，并通过删除整文件的方式终止循环。

### 4) 与 Stop Hook 的协议耦合点

`plugins/ralph-wiggum/hooks/stop-hook.sh:15-18` 明确规定：状态文件不存在时直接 `exit 0` 放行退出。因此 `/cancel-ralph` 的删除动作是“协议级中断信号”，不需要修改 Hook 逻辑本身。

## 关键代码路径与文件引用

- 命令定义：`plugins/ralph-wiggum/commands/cancel-ralph.md:1-18`
- 状态文件生产方：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`
- 状态文件消费方（退出拦截）：`plugins/ralph-wiggum/hooks/stop-hook.sh:13-18`
- Hook 注册：`plugins/ralph-wiggum/hooks/hooks.json:3-13`
- 插件总览文档（命令入口说明）：`plugins/ralph-wiggum/README.md:63-69`
- 插件元数据：`plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`

测试/脚本上下文：

- 该命令目录下未发现独立测试文件（仓库中未检索到 `ralph-wiggum` 对应 `test/spec`）。
- 相关脚本语法检查可通过：
  - `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`
  - `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh`

## 依赖与外部交互

1. 文件系统依赖：`.claude/ralph-loop.local.md`（存在性、读取、删除）。
2. Claude 命令框架依赖：frontmatter 的 `allowed-tools`/`description`/隐藏字段机制。
3. 无网络依赖：该命令不调用 HTTP API，不依赖外部服务。
4. 与插件其他组件的交互：通过“状态文件存在与否”影响 Stop Hook 是否继续阻断退出。

## 风险、边界与改进建议

1. 并发竞争窗口：`test -f` 与 `rm` 之间可能发生状态变化（例如 Stop Hook 同时删除文件），会造成“读取到迭代但删除失败”或“删除时文件已不存在”。
建议：删除时容错 `rm -f` + 明确分支消息（例如“state already cleared”）。

2. `iteration` 解析健壮性依赖命令执行者实现：命令文本要求“读取 iteration 字段”，但未给出严格解析规则。
建议：在命令正文中补一条“若字段缺失，使用 unknown”以减少格式漂移导致的报错。

3. 人工中断提示与 setup 文案存在认知冲突：setup 脚本提示“cannot be stopped manually”（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:166-167`），而本命令实际提供了手工停止。
建议：统一为“默认无限；可通过 `/cancel-ralph` 手动停止”。

4. 权限声明中的 `Bash(test -f ...:*)` 过滤式样与实际调用写法存在实现依赖。
建议：在插件开发文档中补一个该写法示例，避免后续维护者改坏权限匹配模式。
