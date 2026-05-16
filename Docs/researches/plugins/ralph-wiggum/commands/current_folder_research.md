# DIR `plugins/ralph-wiggum/commands` 研究文档

## 场景与职责

`plugins/ralph-wiggum/commands` 是 Ralph Wiggum 插件的“命令入口层”，负责把用户在会话里的意图（启动循环、取消循环、查询说明）映射为可执行动作或解释文本。该目录本身不实现循环控制算法，核心职责是：

- 暴露用户入口：`/ralph-loop`、`/cancel-ralph`、`/help`（`plugins/ralph-wiggum/commands/*.md`）。
- 约束命令执行权限：通过 frontmatter `allowed-tools` 限定可调用工具范围（例如 `/ralph-loop` 只允许调用插件内 setup 脚本）。
- 把“会话命令”衔接到“状态文件 + Stop Hook”运行时链路：
  - 初始化状态：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`
  - 运行时拦停与续跑：`plugins/ralph-wiggum/hooks/stop-hook.sh:13-174`
  - Hook 注册：`plugins/ralph-wiggum/hooks/hooks.json:2-13`

在插件系统中的上下游位置：

- 上游调用方：Claude Code 插件命令自动发现机制（`commands/*.md`）与用户 slash command 输入（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:17-31`）。
- 下游被调用方：`setup-ralph-loop.sh`、状态文件 `.claude/ralph-loop.local.md`、Stop hook 脚本、以及取消命令中的 `test/rm/read` 操作。

仓库级定位：`plugins/README.md` 把该插件描述为“通过 `/ralph-loop` 与 Stop Hook 实现迭代闭环”的代表实现（`plugins/README.md:26`）。

## 功能点目的

### 1) `/ralph-loop`：在当前 session 启动循环

目的：把任务 prompt 与停止条件（最大迭代数、完成承诺）写入本地状态文件，激活后续 Stop Hook 的循环逻辑。

关键目标：

- 仅允许执行插件脚本：`allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh:*)`（`plugins/ralph-wiggum/commands/ralph-loop.md:4`）。
- 将用户参数透传给 setup 脚本（`plugins/ralph-wiggum/commands/ralph-loop.md:12-14`）。
- 在命令层强化“completion promise 不得造假”的行为约束（`plugins/ralph-wiggum/commands/ralph-loop.md:18`）。

### 2) `/cancel-ralph`：人工终止循环

目的：允许操作者在会话内主动停止无限循环，避免只能等待自动停止条件。

关键目标：

- 检测状态文件是否存在（`plugins/ralph-wiggum/commands/cancel-ralph.md:11`）。
- 存在时读取当前迭代并删除状态文件（`plugins/ralph-wiggum/commands/cancel-ralph.md:16-18`）。
- 通过最小工具面完成操作（`test -f`、`Read`、`rm`，`plugins/ralph-wiggum/commands/cancel-ralph.md:3`）。

### 3) `/help`：解释方法论与使用边界

目的：把 Ralph 技术原理、命令入口、停止条件、适用场景集中在一个帮助命令中，降低误用概率。

关键目标：

- 解释“同 prompt 重复输入 + 文件状态累积”的自引用机制（`plugins/ralph-wiggum/commands/help.md:20-29`）。
- 解释 `--max-iterations` 与 `--completion-promise` 的使用方式（`plugins/ralph-wiggum/commands/help.md:44-55,76-85`）。
- 提供正反场景和示例 prompt，降低无限循环风险（`plugins/ralph-wiggum/commands/help.md:94-121`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 命令 frontmatter 协议与权限边界

命令文件遵循 markdown + YAML frontmatter 协议（`description`、`argument-hint`、`allowed-tools`），用于在 `/help` 可发现性与工具访问控制间取得平衡（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:24-31,60-67,196-203`）。

本目录实际权限模型：

- `/ralph-loop`：仅可执行 `setup-ralph-loop.sh`（最小权限，`plugins/ralph-wiggum/commands/ralph-loop.md:4`）。
- `/cancel-ralph`：只读写一个状态文件并执行删除（`plugins/ralph-wiggum/commands/cancel-ralph.md:3`）。
- `/help`：无工具调用，纯文本说明（`plugins/ralph-wiggum/commands/help.md:1-3`）。

### B. 启动链路：`/ralph-loop` -> setup 脚本 -> 状态文件

1. 用户调用 `/ralph-loop <PROMPT> [--max-iterations N] [--completion-promise TEXT]`。
2. 命令透传参数执行脚本（`plugins/ralph-wiggum/commands/ralph-loop.md:12-14`）。
3. 脚本解析参数并校验：
   - `--max-iterations` 必须是 `^[0-9]+$`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:61-85`）。
   - `--completion-promise` 必须有值（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:87-103`）。
   - prompt 不能为空（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:115-128`）。
4. 创建 `.claude/ralph-loop.local.md` 并写入 frontmatter + body（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:131-150`）。

状态文件结构（运行时核心数据结构）：

```markdown
---
active: true
iteration: 1
max_iterations: <number>
completion_promise: <string|null>
started_at: "<UTC ISO8601>"
---

<PROMPT 原文>
```

字段消费方：`stop-hook.sh` 会读取 `iteration/max_iterations/completion_promise` 决策是否 block 停止（`plugins/ralph-wiggum/hooks/stop-hook.sh:20-25,50-55,114-127`）。

### C. 运行链路：Stop Hook 阻断退出并回灌同一 prompt

Hook 注册格式采用 plugin wrapper（`{"hooks": {...}}`）：`plugins/ralph-wiggum/hooks/hooks.json:1-15`，与 hook-development skill 规范一致（`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`）。

`stop-hook.sh` 的关键流程：

1. 从 stdin 读取 hook input JSON（`plugins/ralph-wiggum/hooks/stop-hook.sh:9-10`）。
2. 若无状态文件则直接放行退出（`plugins/ralph-wiggum/hooks/stop-hook.sh:15-18`）。
3. 解析 frontmatter 并校验数字字段（`plugins/ralph-wiggum/hooks/stop-hook.sh:20-48`）。
4. 达到 `max_iterations` 时清理状态并结束（`plugins/ralph-wiggum/hooks/stop-hook.sh:50-55`）。
5. 从 hook input 取 `transcript_path`，读取最后一条 assistant 文本内容（`plugins/ralph-wiggum/hooks/stop-hook.sh:57-95`）。
6. 若设置 completion promise，则从 `<promise>...</promise>` 提取文本并做字面值精确匹配（`plugins/ralph-wiggum/hooks/stop-hook.sh:114-127`）。
7. 未完成则 `iteration+1` 并回写状态文件（临时文件 + mv，`plugins/ralph-wiggum/hooks/stop-hook.sh:131-156`）。
8. 输出 Stop hook 决策 JSON：`{"decision":"block","reason":<prompt>,"systemMessage":<msg>}`，阻断退出并把同 prompt 送入下一轮（`plugins/ralph-wiggum/hooks/stop-hook.sh:165-174`）。

该输出与 Stop hook 协议样式一致（`decision/reason/systemMessage`，`plugins/plugin-dev/skills/hook-development/SKILL.md:202-208`）。

### D. 取消链路：`/cancel-ralph`

命令文档定义了人工终止流程：检测文件 -> 读取 `iteration` -> 删除文件 -> 回报取消结果（`plugins/ralph-wiggum/commands/cancel-ralph.md:11-18`）。

这条链路与 setup 文本中的“无限循环”提示形成互补，提供运维级逃生阀。

### E. 文档链路：README 与命令 help 的角色分工

- `plugins/ralph-wiggum/README.md` 偏“完整方法论 + 提示词工程建议 + 适用性边界”（`plugins/ralph-wiggum/README.md:72-165`）。
- `commands/help.md` 偏“会话内即查说明”，降低用户在命令执行时的上下文切换成本（`plugins/ralph-wiggum/commands/help.md:32-71`）。

## 关键代码路径与文件引用

目标目录：

- `plugins/ralph-wiggum/commands/ralph-loop.md`
- `plugins/ralph-wiggum/commands/cancel-ralph.md`
- `plugins/ralph-wiggum/commands/help.md`

直接被调用实现：

- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:1-203`
- `plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`
- `plugins/ralph-wiggum/hooks/hooks.json:1-15`

关键状态与配置文件：

- `.claude/ralph-loop.local.md`（运行时状态；由 setup 创建，被 stop-hook 与 cancel 消费）
- `plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`（插件元数据）

上下文文档与规范参考：

- `plugins/ralph-wiggum/README.md:1-179`
- `plugins/README.md:26`
- `plugins/plugin-dev/README.md:116-125,137-145`
- `plugins/plugin-dev/skills/plugin-settings/references/real-world-examples.md:129-227`
- `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:17-31,75-85`
- `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,183-208`

测试/校验/脚本相关：

- 目标目录 `plugins/ralph-wiggum/commands` 无测试文件（仓库扫描未发现 `test/spec`）。
- 可执行校验：
  - `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh` 通过。
  - `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh` 通过。
- 通用脚本 `validate-hook-schema.sh` 与插件 wrapper 格式不兼容，直接校验 `plugins/ralph-wiggum/hooks/hooks.json` 会失败（`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-75` 与 `plugins/ralph-wiggum/hooks/hooks.json:1-4` 的结构不匹配）。

## 依赖与外部交互

### 1) 运行时依赖

- Shell 工具链：`bash/sed/awk/grep/jq/perl/date/mv/rm/test`（主要在 `setup-ralph-loop.sh` 与 `stop-hook.sh`）。
- Claude 插件环境变量：`${CLAUDE_PLUGIN_ROOT}` 用于命令与 hook 路径可移植引用（`plugins/ralph-wiggum/commands/ralph-loop.md:4,13`）。

### 2) 文件系统交互

- 写入：`.claude/ralph-loop.local.md`（setup）。
- 读取：状态文件 + transcript JSONL（stop hook）。
- 删除：达到停止条件或手工取消时删除状态文件。

### 3) 协议交互（Claude Hook API）

- 输入：Stop hook 从 stdin 接收事件 JSON，并解析 `transcript_path`。
- 输出：返回 `decision=block` 的 JSON 以拦截退出。
- 非网络交互：该目录逻辑不依赖外部 HTTP API。

### 4) 文档与用户交互

- `/help` 与 README 作为行为约束和失败预防层（如 completion promise 真实性约束、`--max-iterations` 安全网建议）。

## 风险、边界与改进建议

1. `commands/help.md` 的状态文件路径写错。
- 现状：帮助文档写 `.claude/.ralph-loop.local.md`（`plugins/ralph-wiggum/commands/help.md:49,69`）。
- 实际实现：统一使用 `.claude/ralph-loop.local.md`（`setup-ralph-loop.sh`、`stop-hook.sh`、`cancel-ralph.md`）。
- 风险：用户按帮助文档手工排障会找错文件。
- 建议：修正文档路径并补一条“与 `/cancel-ralph` 一致”的提示。

2. “无法手工停止”描述与实际能力矛盾。
- 现状：setup 输出提示“cannot be stopped manually”（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:166-167`）。
- 同时插件提供 `/cancel-ralph` 明确支持手工终止（`plugins/ralph-wiggum/commands/cancel-ralph.md:7-18`）。
- 风险：用户误判控制权，可能影响生产环境下的使用信心。
- 建议：改成“默认无限循环，可通过 `/cancel-ralph` 或停止条件结束”。

3. `--completion-promise` 的 YAML 转义不完整。
- 现状：仅做双引号包裹（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:134-136`），未处理内部引号、反斜杠、换行。
- 风险：特殊字符可能导致 frontmatter 非法或解析偏差。
- 建议：改为 `jq -R` 或专用 YAML 转义函数生成安全字符串。

4. 参数解析把未知选项当作 prompt 文本。
- 现状：`case *` 分支直接加入 `PROMPT_PARTS`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:104-108`）。
- 风险：拼写错误选项（如 `--max-iteratons`）不会报错，用户误以为参数生效。
- 建议：对 `--*` 未知选项显式报错，提升可预测性。

5. `validate-hook-schema.sh` 与插件 hooks wrapper 格式不兼容。
- 现状：脚本按“顶层事件键”遍历（`validate-hook-schema.sh:41-66`），而插件实际是 `{ "description": ..., "hooks": { ... } }`（`hooks.json:1-4`）。
- 风险：会给出误报/崩溃，降低工程信任。
- 建议：校验脚本先检测并下钻 `.hooks`，或文档明确“先抽取 hooks 节点再校验”。

6. 边界条件：对 transcript 格式强耦合。
- 现状：通过 `grep '"role":"assistant"' | tail -1` + `jq` 提取文本（`stop-hook.sh:81-95`）。
- 风险：若 transcript 结构调整或出现无 text 块消息，循环会提前终止并删除状态（`stop-hook.sh:107-111`）。
- 建议：增加兼容分支（例如支持更多 content type）并把“自动停止原因”写入更结构化日志。

7. 命令命名边界：`help.md` 与内置 `/help` 的可发现性关系不够直观。
- 现状：目录包含 `help.md`（`plugins/ralph-wiggum/commands/help.md:1`），但插件 README 又提示“Run `/help` for detailed reference”（`plugins/ralph-wiggum/README.md:179`）。
- 风险：用户可能不清楚调用的是内置帮助还是插件帮助内容。
- 建议：在 README 明确插件帮助命令的调用方式（例如命名/命名空间展示形式），减少歧义。
