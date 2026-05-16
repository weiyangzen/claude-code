# FILE 研究：plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh

## 场景与职责
`validate-bash.sh` 是 `PreToolUse` 事件下针对 `Bash` 工具的命令校验示例。它在工具执行前读取 Hook 输入 JSON，按命令模式执行“快速放行 / 拒绝 / 要求确认（ask）”三类决策，并通过约定退出码和 JSON 返回给 Claude。

该脚本定位为“确定性、低成本”的命令安全闸门示例，不追求完整语义安全分析，而是演示 command hook 的基础协议与控制流。

## 功能点目的
1. 读取并解析 `tool_input.command`，获取待执行 Bash 命令。
2. 对空命令场景快速返回，避免误阻断非 Bash 上下文输入。
3. 对明显安全命令（`ls/pwd/echo/date/whoami`）快速放行，降低交互噪声。
4. 对高危破坏命令（`rm -rf`、`dd if=`、`mkfs`、重定向到 `/dev/*`）直接 deny。
5. 对提权命令（`sudo`/`su`）返回 ask，让上层触发确认流程。

## 具体技术实现（关键流程/数据结构/协议/命令）
脚本实现位于 `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh:1-43`。

1. 输入读取与 JSON 提取
- `input=$(cat)` 读取 stdin。
- `command=$(echo "$input" | jq -r '.tool_input.command // empty')` 提取命令字符串。
- 依赖 `jq`，且在 `set -euo pipefail` 下，解析失败会直接退出。

2. 分支决策顺序
- 空命令：输出 `{"continue": true}`，`exit 0`。
- 安全白名单：正则 `^(ls|pwd|echo|date|whoami)(\s|$)` 命中即 `exit 0`。
- 高危拒绝 1：包含 `rm -rf` 或 `rm -fr`，stderr 输出 `permissionDecision=deny` JSON，`exit 2`。
- 高危拒绝 2：包含 `dd if=`/`mkfs`/> `/dev/`，同样 deny + `exit 2`。
- 提权确认：命令前缀匹配 `sudo*` 或 `su*`，stderr 输出 `permissionDecision=ask`，`exit 2`。
- 其余默认放行：`exit 0`。

3. 协议与退出码
- `PreToolUse` 决策信息使用 `hookSpecificOutput.permissionDecision` + `systemMessage`。
- 退出码约定：`0` 表示成功放行，`2` 表示阻断/需确认路径（stderr 回传给 Claude）。

实测（本次研究执行）：
- `{"tool_input":{}}` -> `exit 0`，输出 `{"continue": true}`。
- `{"tool_input":{"command":"ls -la"}}` -> `exit 0`。
- `{"tool_input":{"command":"rm -rf /tmp/demo"}}` -> `exit 2` + deny JSON。
- `{"tool_input":{"command":"sudo systemctl restart nginx"}}` -> `exit 2` + ask JSON。

## 关键代码路径与文件引用
- 主脚本：`plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh:1-43`
- PreToolUse 输出结构定义：`plugins/plugin-dev/skills/hook-development/SKILL.md:144-153`
- Hook 输入公共字段：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-317`
- 退出码语义：`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`
- 迁移文档（该脚本作为“基础命令 Hook”示例）：`plugins/plugin-dev/skills/hook-development/references/migration.md:14-53`
- 进阶文档中的测试示例：`plugins/plugin-dev/skills/hook-development/references/advanced.md:381-402`
- 测试工具使用入口：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:17-18`、`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:205-218`

## 依赖与外部交互
1. 运行依赖
- `/bin/bash`
- `jq`（JSON 解析必需）

2. 输入输出接口
- 输入：stdin JSON（Claude Hook payload）。
- 输出：
  - 放行：通常无输出，`exit 0`。
  - 拒绝/确认：stderr JSON（含 `permissionDecision` 与 `systemMessage`），`exit 2`。

3. 上游配置
- 通常在 `hooks/hooks.json` 中以 `PreToolUse + matcher=Bash` 方式注册（`SKILL.md` 与 `migration.md` 有对应配置片段）。

4. 外部副作用
- 无文件写入、无网络调用、无子进程执行待检命令；仅做字符串匹配决策。

## 风险、边界与改进建议
1. 模式匹配可绕过
- 风险：仅匹配少量字面模式，`rm -r -f`、分号拼接、变量展开、编码绕过等可能漏检。
- 建议：先做 token 化/规范化，再基于规则集匹配；补充 `curl|bash`、`wget|sh`、`chmod 777 -R /` 等高危模式。

2. 前缀判断有误报
- 风险：`[[ "$command" == su* ]]` 会把 `sum ...` 误判为提权命令。
- 建议：改为 `^sudo([[:space:]]|$)` 与 `^su([[:space:]]|$)`。

3. 缺少事件/工具名守卫
- 风险：若被错误绑定到非 Bash 场景，仍按 `command` 字段处理。
- 建议：校验 `.hook_event_name == "PreToolUse"` 且 `.tool_name == "Bash"`。

4. 依赖失败即硬退出
- 风险：`jq` 不存在或输入非 JSON 时，脚本因 `set -e` 直接失败，行为不透明。
- 建议：启动时检查 `command -v jq`，并在解析失败时返回结构化错误 JSON。

5. 安全策略粒度有限
- 风险：当前仅路径级 deny/ask，无法结合项目上下文（分支、目录、用户意图）。
- 建议：将该脚本定位为“第一层快速闸门”，复杂判断迁移到 prompt hook（见 `references/migration.md` 的推荐路径）。
