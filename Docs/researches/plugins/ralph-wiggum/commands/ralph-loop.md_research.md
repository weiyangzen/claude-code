# FILE `plugins/ralph-wiggum/commands/ralph-loop.md` 研究文档

## 场景与职责

`/ralph-loop` 是 Ralph Wiggum 插件的主入口命令，负责把用户提供的任务 prompt 和停止参数转换为“可被 Stop Hook 消费的本地状态”。它是从自然语言意图到循环运行时的桥梁层。

链路角色如下：

- 调用方：用户在当前 Claude 会话执行 `/ralph-loop ...`。
- 该命令职责：限制工具权限、透传参数到 setup 脚本、在命令层注入行为约束（不可输出虚假 promise）。
- 被调用方：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`（直接调用），以及后续由 Hook 系统触发的 `plugins/ralph-wiggum/hooks/stop-hook.sh`（间接调用）。

这个命令文件本身很短，但位于最关键控制点：若入口参数传递或权限声明错误，整个循环都无法正确启动。

## 功能点目的

1. 提供统一启动接口：`PROMPT [--max-iterations N] [--completion-promise TEXT]`（`plugins/ralph-wiggum/commands/ralph-loop.md:3`）。
2. 实施最小权限执行：只允许运行插件目录内 setup 脚本（`plugins/ralph-wiggum/commands/ralph-loop.md:4`）。
3. 让循环在“当前会话内”持续，而不是依赖外部 while-true 脚本（由 setup 写状态 + Stop Hook 拦截退出共同实现）。
4. 增加强约束文案：completion promise 只能在“完全为真”时输出，防止通过虚假承诺逃逸循环（`plugins/ralph-wiggum/commands/ralph-loop.md:18`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) frontmatter 协议与执行约束

该命令 frontmatter：

- `description`：说明命令职责（`plugins/ralph-wiggum/commands/ralph-loop.md:2`）。
- `argument-hint`：参数提示字符串（`plugins/ralph-wiggum/commands/ralph-loop.md:3`）。
- `allowed-tools`：仅允许 `Bash(${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh:*)`（`plugins/ralph-wiggum/commands/ralph-loop.md:4`）。
- `hide-from-slash-command-tool: "true"`：对 slash command 工具隐藏（`plugins/ralph-wiggum/commands/ralph-loop.md:5`）。

这符合插件命令 frontmatter 模型中“通过 `allowed-tools` 收紧能力边界”的实践（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:52-97`）。

### 2) 启动流程

命令正文只有一条执行语句（`plugins/ralph-wiggum/commands/ralph-loop.md:12-14`）：

```bash
"${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh" $ARGUMENTS
```

关键机制：

1. 所有用户参数原样交给 setup 脚本解析（`setup-ralph-loop.sh:13-110`）。
2. setup 校验参数与 prompt 后，创建 `.claude/ralph-loop.local.md`（`setup-ralph-loop.sh:130-150`）。
3. Stop Hook 在会话尝试退出时读取该状态并决定是否 `block`（`stop-hook.sh:15-18,165-174`）。

因此 `ralph-loop.md` 虽然不直接处理迭代，但决定“状态初始化是否正确进入 Hook 回路”。

### 3) 关键数据结构与协议

`/ralph-loop` 间接依赖两类协议：

1. 状态文件协议（`.claude/ralph-loop.local.md`）：
   - frontmatter 字段 `iteration/max_iterations/completion_promise` 被 Hook 消费。
2. Stop Hook 决策协议：
   - Hook 输出 `{ "decision": "block", "reason": <prompt>, "systemMessage": <msg> }`（`stop-hook.sh:170-174`），使 Claude 继续下一轮。

### 4) 参数行为的实际语义（由 setup 脚本定义）

- `--max-iterations` 仅接受非负整数（`setup-ralph-loop.sh:61-85`）。
- `--completion-promise` 必须带参数（`setup-ralph-loop.sh:87-103`）。
- 未提供 prompt 会报错并退出（`setup-ralph-loop.sh:115-128`）。
- completion promise 匹配是字面值精确比较（Hook `=` 比较，`stop-hook.sh:121-127`）。

## 关键代码路径与文件引用

- 目标命令：`plugins/ralph-wiggum/commands/ralph-loop.md:1-18`
- 启动脚本：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:13-150`
- 停止拦截与续跑：`plugins/ralph-wiggum/hooks/stop-hook.sh:50-55,114-127,165-174`
- Hook 注册：`plugins/ralph-wiggum/hooks/hooks.json:3-13`
- 取消入口（人工终止）：`plugins/ralph-wiggum/commands/cancel-ralph.md:11-18`
- 用户说明文档：`plugins/ralph-wiggum/README.md:50-69`
- 插件元数据：`plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`

测试/脚本上下文：

- 当前目录无独立测试用例文件。
- 关联脚本语法校验通过：
  - `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`
  - `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh`

## 依赖与外部交互

1. 运行环境依赖：`${CLAUDE_PLUGIN_ROOT}` 路径变量（命令调用 setup 与 hooks 配置都依赖它）。
2. Shell 依赖：setup/hook 脚本使用 `bash/sed/awk/grep/jq/perl/date/mv/rm` 等本地命令。
3. 文件系统依赖：`.claude/ralph-loop.local.md` 作为会话循环状态单一事实来源。
4. 外部交互边界：无 HTTP 网络请求，主要是本地文件与 Claude Hook API 输入输出交互。

## 风险、边界与改进建议

1. 参数透传的可诊断性有限：命令层不做参数检查，全部下沉到脚本。若用户输入拼写错误参数，会在脚本层才暴露，定位成本更高。
建议：在命令文案中补“参数校验由 setup 脚本执行，失败会返回具体错误”。

2. `allowed-tools` 依赖路径模式匹配正确性：若后续改脚本路径或重命名文件但未同步 frontmatter，命令将直接失效。
建议：新增一个轻量校验脚本，对 `commands/ralph-loop.md` 中声明路径与实际文件存在性做 CI 检查。

3. 伦理约束仅靠提示词文本：`CRITICAL RULE` 是行为声明，不是硬技术防护。最终是否停循环仍由 Hook 的 promise 检测决定。
建议：把“检测到明显虚假 promise 时继续阻断”作为 Hook 层可选增强策略。

4. 与帮助文档的一致性风险：`help.md` 目前存在状态文件路径误写（`.claude/.ralph-loop.local.md`），会反向影响 `/ralph-loop` 的可用性认知。
建议：将命令、help、README 的关键常量（状态文件路径/参数名）抽成统一文档片段或做一致性 lint。
