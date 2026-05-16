# plugins/ralph-wiggum/.claude-plugin/plugin.json 研究

## 场景与职责
`plugins/ralph-wiggum/.claude-plugin/plugin.json` 是 Ralph Wiggum 插件的 manifest 入口文件（`plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`）。

它本身不执行业务逻辑，但承担“让插件被系统识别并装配”的前置职责：
- 插件结构规范要求 manifest 必须位于 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- 自动发现机制启用插件时先读取 manifest，再扫描命令、hooks 等组件（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`）。
- 仓库 marketplace 将 `ralph-wiggum` 路由到 `./plugins/ralph-wiggum`，运行时最终落到该 manifest（`.claude-plugin/marketplace.json:128-136`）。

因此，`plugin.json` 是 `/ralph-loop`、`/cancel-ralph`、Stop Hook 整条链路被加载的根入口。

## 功能点目的
该文件只声明 4 类核心元数据，但每个字段都有明确作用：
- `name`（`plugins/ralph-wiggum/.claude-plugin/plugin.json:2`）：插件唯一标识 `ralph-wiggum`，对应规范中的 kebab-case 唯一命名约束（`manifest-reference.md:15-40`）。
- `version`（`plugins/ralph-wiggum/.claude-plugin/plugin.json:3`）：语义化版本 `1.0.0`，用于版本演进与市场展示（`manifest-reference.md:42-64`）。
- `description`（`plugins/ralph-wiggum/.claude-plugin/plugin.json:4`）：解释插件能力边界（同 prompt 的持续迭代 loop）；长度 198，符合规范建议 50-200 字符（`manifest-reference.md:65-83`）。
- `author`（`plugins/ralph-wiggum/.claude-plugin/plugin.json:5-8`）：作者归属与联系方式，供展示与追溯（`manifest-reference.md:87-113`）。

该 manifest 未定义 `commands/agents/hooks/mcpServers` 自定义路径，意味着依赖默认目录约定加载组件（`manifest-reference.md:259-263,354-361`）。

## 具体技术实现（关键流程/数据结构/协议/命令）
### 关键流程
1. 插件发现
- Claude Code 读取 `.claude-plugin/plugin.json`（`plugin-structure/SKILL.md:343`）。
- 因 manifest 未覆写路径，按默认扫描 `commands/` 与 `hooks/hooks.json`（`manifest-reference.md:354-361`）。

2. 命令触发初始化
- `/ralph-loop` 的 frontmatter 仅允许调用 setup 脚本（`plugins/ralph-wiggum/commands/ralph-loop.md:2-5`）。
- 命令体执行 `${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh $ARGUMENTS`（`plugins/ralph-wiggum/commands/ralph-loop.md:12-14`）。

3. 状态文件建模
- setup 脚本创建 `.claude/ralph-loop.local.md`（`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:140-150`）。
- frontmatter 字段包括：`active`、`iteration`、`max_iterations`、`completion_promise`、`started_at`（`setup-ralph-loop.sh:142-146`）。
- markdown body 存原始 prompt（`setup-ralph-loop.sh:149`）。

4. Stop Hook 驱动循环
- `hooks/hooks.json` 将 Stop 事件绑定到 `hooks/stop-hook.sh`（`plugins/ralph-wiggum/hooks/hooks.json:3-10`）。
- stop-hook 从 stdin 读取 hook 输入 JSON，再从 `transcript_path` 定位 transcript（`plugins/ralph-wiggum/hooks/stop-hook.sh:9-10,57-58`）。
- 读取状态 frontmatter，做迭代上限与 completion promise 判断（`stop-hook.sh:21-25,50-55,114-127`）。
- 未完成时递增 `iteration`（`stop-hook.sh:131,152-156`），并输出：
  - `decision: block`
  - `reason: <原始prompt>`
  - `systemMessage: <当前迭代提示>`
  （`stop-hook.sh:167-174`）

5. 手动取消
- `/cancel-ralph` 允许 `test/read/rm` 状态文件，删除后终止循环（`plugins/ralph-wiggum/commands/cancel-ralph.md:3,11-18`）。

### 关键数据结构
- Manifest（插件注册层）：`name/version/description/author`（`plugin.json:2-8`）。
- 本地状态文件（会话状态层）：YAML frontmatter + markdown body（`setup-ralph-loop.sh:140-150`）。
- Stop Hook 输入协议：stdin JSON，至少消费 `transcript_path`（`stop-hook.sh:9-10,57-58`）。
- Stop Hook 输出协议：stdout JSON，关键决策字段 `decision/reason/systemMessage`（`stop-hook.sh:165-174`）。

### 关键命令与解析协议
- Shell 解析：`bash` + `set -euo pipefail`（`setup-ralph-loop.sh:6`，`stop-hook.sh:7`）。
- frontmatter 解析：`sed` + `grep` + `awk`（`stop-hook.sh:21-26,136`）。
- transcript 解析：`jq` 抽取最后 assistant 文本（`stop-hook.sh:81,90-95`）。
- promise 提取：`perl -0777` 抽取 `<promise>...</promise>`（`stop-hook.sh:116-123`）。

## 关键代码路径与文件引用
### 目标对象
- `plugins/ralph-wiggum/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 必需位置）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`（自动发现顺序）
- `.claude-plugin/marketplace.json:128-136`（仓库 marketplace 对 `ralph-wiggum` 的 source 路由）
- `plugins/README.md:26,49-60,69`（插件能力与结构约定）

### 被调用方（下游）
- `plugins/ralph-wiggum/commands/ralph-loop.md:1-18`
- `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:1-203`
- `plugins/ralph-wiggum/hooks/hooks.json:1-15`
- `plugins/ralph-wiggum/hooks/stop-hook.sh:1-177`
- `plugins/ralph-wiggum/commands/cancel-ralph.md:1-18`
- `plugins/ralph-wiggum/commands/help.md:1-126`
- `plugins/ralph-wiggum/README.md:1-179`

### 配置、测试、脚本、文档覆盖
- 配置：`plugin.json` + `hooks/hooks.json` + 命令 frontmatter + `.claude/ralph-loop.local.md`。
- 脚本：`scripts/setup-ralph-loop.sh`（初始化）与 `hooks/stop-hook.sh`（循环控制）。
- 测试：目录内无 `tests/`、`spec`、`__tests__`，未发现插件专属自动化测试文件。
- 文档：`README.md` 与 `commands/help.md` 解释使用方式和机制。

## 依赖与外部交互
- 运行时二进制/命令依赖：`bash`、`jq`、`perl`、`sed`、`grep`、`awk`、`tail`、`date`、`mv`、`rm`（见 `stop-hook.sh` 与 `setup-ralph-loop.sh`）。
- 文件系统交互：读写 `.claude/ralph-loop.local.md`；读取 hook 输入提供的 transcript 文件（`stop-hook.sh:57-67`）。
- 平台协议交互：Stop hook 通过 stdin 接收 JSON，通过 stdout 输出决策 JSON（`stop-hook.sh:9-10,165-174`）。
- 外部文档链接：README 引用 `https://ghuntley.com/ralph/` 与 `https://github.com/mikeyobrien/ralph-orchestrator`（`plugins/ralph-wiggum/README.md:174-175`）。

补充验证（本次实测）：
- `bash -n plugins/ralph-wiggum/hooks/stop-hook.sh`：通过。
- `bash -n plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`：通过。
- `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/ralph-wiggum/hooks/hooks.json`：失败，报 `Cannot index string with number`，说明通用校验脚本与该 hooks 包装结构存在兼容差异。

## 风险、边界与改进建议
### 风险
- `commands/help.md` 中状态文件路径写成 `.claude/.ralph-loop.local.md`（`help.md:49,69`），与真实实现 `.claude/ralph-loop.local.md` 不一致（`setup-ralph-loop.sh:140`、`stop-hook.sh:13`、`cancel-ralph.md:11`）。
- setup 脚本提示“cannot be stopped manually”（`setup-ralph-loop.sh:166-167`）与 `/cancel-ralph` 可手动停止能力存在认知冲突（`cancel-ralph.md:1-18`）。
- `completion_promise` 仅用双引号包裹，未显式转义内部引号/反斜杠（`setup-ralph-loop.sh:134-136`），在极端输入下可能破坏 frontmatter 可解析性。
- `stop-hook.sh` 在 `set -euo pipefail` 下依赖 `grep '^iteration:'`/`grep '^max_iterations:'`（`stop-hook.sh:7,22-23`），字段缺失时可能提前退出，不一定进入后续错误提示分支。

### 边界
- `plugin.json` 只负责声明与装配入口，不负责任务循环算法。
- 实际执行路径完全在命令、脚本和 hook；manifest 的影响主要在“能否被发现”和“发现后用默认路径加载什么”。

### 改进建议
- 统一 help/README/命令文档中的状态文件路径表达，避免用户排障误导。
- 调整 setup 的停止文案，明确 `/cancel-ralph` 是人工终止通道。
- 对 `completion_promise` 做更严格 YAML 安全编码（例如转义双引号与反斜杠）。
- 在 `stop-hook.sh` 增加 frontmatter 字段缺失显式检查，减少 `set -e` 引发的不可解释退出。
- 为插件 hooks 格式补充可用的 schema 校验或修正通用校验脚本兼容性。
