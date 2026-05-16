# FILE 研究：plugins/plugin-dev/skills/hook-development/examples/validate-write.sh

## 场景与职责
`validate-write.sh` 是 `PreToolUse` 事件下针对 `Write/Edit` 场景的文件写入校验示例。其职责是在工具真正写盘前，对目标路径做快速安全分级，输出 `allow/deny/ask` 决策，防止明显危险写入。

该脚本属于“命令型 Hook 的最小安全模板”：强调确定性与可解释性，而非完整数据安全审计。

## 功能点目的
1. 从 Hook 输入提取 `tool_input.file_path`。
2. 路径为空时快速放行，避免中断非文件写入调用。
3. 阻断路径穿越（`..`）模式。
4. 阻断系统目录写入（`/etc`、`/sys`、`/usr`）。
5. 对敏感文件名（`.env`、`secret`、`credentials`）改为 `ask`，要求额外确认。

## 具体技术实现（关键流程/数据结构/协议/命令）
脚本实现位于 `plugins/plugin-dev/skills/hook-development/examples/validate-write.sh:1-38`。

1. 输入读取
- `input=$(cat)` 读取 stdin。
- `file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')` 提取文件路径。

2. 校验与决策
- 空路径：输出 `{"continue": true}`，`exit 0`。
- 路径穿越：若包含 `..`，stderr 输出 deny JSON，`exit 2`。
- 系统目录：若前缀命中 `/etc/*`、`/sys/*`、`/usr/*`，stderr 输出 deny JSON，`exit 2`。
- 敏感命名：若命中 `*.env`、`*secret*`、`*credentials*`，stderr 输出 ask JSON，`exit 2`。
- 其余：`exit 0` 放行。

3. Hook 协议实现
- 使用 `hookSpecificOutput.permissionDecision` 表达权限决策。
- 使用 `systemMessage` 传递可读原因。
- 通过 `exit 2` 触发 Claude 对 stderr JSON 的阻断/确认处理。

实测（本次研究执行）：
- `src/main.ts` -> `exit 0`（放行）。
- `../etc/passwd` -> `exit 2` + deny JSON。
- `.env` -> `exit 2` + ask JSON。

## 关键代码路径与文件引用
- 主脚本：`plugins/plugin-dev/skills/hook-development/examples/validate-write.sh:1-38`
- Path Safety 指南（同类逻辑示意）：`plugins/plugin-dev/skills/hook-development/SKILL.md:447-465`
- PreToolUse 输出结构：`plugins/plugin-dev/skills/hook-development/SKILL.md:144-153`
- 输入字段协议：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-317`
- 迁移文档中的“基础写入校验”示例：`plugins/plugin-dev/skills/hook-development/references/migration.md:83-128`
- 测试辅助脚本（PreToolUse 样例生成与结果判定）：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:30-45`、`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:205-218`
- Lint 约束（输入读取、变量引用、退出码等）：`plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:49-90`

## 依赖与外部交互
1. 运行依赖
- `/bin/bash`
- `jq`（必须）

2. 输入/输出
- 输入：stdin JSON，期望包含 `tool_input.file_path`。
- 输出：
  - 放行：`exit 0`（一般无输出）。
  - 拒绝/确认：stderr JSON + `exit 2`。

3. 上下游交互
- 上游：`hooks.json` 中 `PreToolUse` 规则调用该脚本（`Write|Edit` 常见）。
- 下游：Claude Hook 引擎读取退出码与 JSON 决策，决定继续、拒绝或询问。
- 脚本本身无文件写入、无网络调用。

## 风险、边界与改进建议
1. 路径穿越判断过于粗粒度
- 风险：仅按 `..` 子串判断，无法覆盖符号链接、规范化路径绕过、编码路径。
- 建议：对目标路径做 `realpath` 规范化并校验是否仍在允许根目录下。

2. 系统目录覆盖范围有限
- 风险：仅限制 `/etc|/sys|/usr`，未覆盖 `/bin`、`/sbin`、`/root`、`/boot`、`/var/lib` 等高风险目录。
- 建议：维护可配置 denylist，并允许项目级覆盖。

3. 敏感文件匹配存在误报/漏报
- 风险：`*secret*` 可能误报普通文件名，且未覆盖 `id_rsa`、`.pem`、`token` 等。
- 建议：使用更精细规则（文件名词典 + 扩展名 + 路径上下文），并在 `ask` 场景附带更具体提示。

4. 未结合内容安全
- 风险：只检查路径，不检查写入内容是否包含凭据/密钥。
- 建议：结合 `tool_input.content` 做轻量 secret pattern 扫描（见 `references/advanced.md` 的 secret detection 模式）。

5. 事件绑定未自校验
- 风险：误绑定到其他工具事件时，可能出现无意义放行。
- 建议：增加 `.hook_event_name` 与 `.tool_name` 断言，异常时返回结构化错误。
