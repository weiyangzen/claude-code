# plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh 研究

## 场景与职责

`plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh` 是 `plugin-settings` 技能中的可执行 Hook 示例，演示如何读取 `.claude/my-plugin.local.md` 并据此在 PreToolUse 阶段做放行/阻断决策。

它在调用链中的位置：

1. 上游文档由 `plugin-settings/SKILL.md` 在“From Hooks”与“Example Files”章节直接引用（`plugins/plugin-dev/skills/plugin-settings/SKILL.md:62-97`, `517-523`）。
2. 在 `plugin-dev/README.md` 中归类为 Plugin Settings 的核心 examples 之一（`plugins/plugin-dev/README.md:127-133`, `286-293`）。
3. 协议侧依赖 `hook-development` 对 PreToolUse 输入/输出、退出码语义的定义（`plugins/plugin-dev/skills/hook-development/SKILL.md:278-317`）。

## 功能点目的

1. 快速退出，降低未配置成本  
   若 settings 文件不存在直接 `exit 0`，避免无配置场景报错（`read-settings-hook.sh:10-14`）。

2. 读取 frontmatter 配置  
   使用 `sed` 提取 `---` 间内容，再用 `grep + sed` 取字段（`read-settings-hook.sh:16-22`）。

3. 通过 `enabled` 做总开关  
   `enabled != true` 时直接退出，形成“文件存在但功能停用”的轻量控制（`read-settings-hook.sh:24-27`）。

4. 按 `strict_mode` 切换校验策略  
   严格模式检查路径穿越与敏感文件；标准模式仅阻断系统目录（`read-settings-hook.sh:34-51`）。

5. 按 `max_file_size` 做内容长度限制  
   当 `max_file_size` 为数字时，对 `tool_input.content` 进行字节级长度比较（`read-settings-hook.sh:53-61`）。

6. 使用 Hook 阻断协议返回决策  
   违规时输出 `hookSpecificOutput.permissionDecision=deny` + `systemMessage` 到 stderr，并 `exit 2`（`read-settings-hook.sh:37-60`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 输入/输出协议

1. 输入协议  
   通过 stdin 接收 JSON，并使用 `jq -r '.tool_input.file_path // empty'` 与 `'.tool_input.content // empty'` 读取字段（`read-settings-hook.sh:30-31`, `55`）。  
   此结构与 hook-development 的 PreToolUse 样例一致（`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:30-44`）。

2. 输出协议  
   阻断时输出：

```json
{"hookSpecificOutput":{"permissionDecision":"deny"},"systemMessage":"..."}
```

并返回 `exit 2`，符合 Hook 退出码规范（`hook-development/SKILL.md:294-299`）。

### 2) 关键流程

1. `set -euo pipefail` 开启严格错误处理（`read-settings-hook.sh:5`）。
2. 文件存在性 quick-exit（`10-14`）。
3. frontmatter 抽取（`16-17`）。
4. 字段读取 `enabled/strict_mode/max_file_size`（`20-22`）。
5. enabled 总开关 quick-exit（`24-27`）。
6. 读取 Hook 输入 JSON（`30-31`）。
7. 模式分支校验并返回 deny JSON + `exit 2`（`34-50`）。
8. 可选内容大小校验（`53-61`）。
9. 全部通过 `exit 0`（`64-65`）。

### 3) 实测验证（本次研究）

在临时目录复制脚本并构造 `.claude/my-plugin.local.md` 后执行，得到：

1. 无 settings 文件：退出码 `0`（符合 quick-exit）。
2. `strict_mode: true` 且路径 `../secret.env`：退出码 `2`，stderr 输出 deny JSON。
3. 缺少 `strict_mode` 字段：退出码 `1`（`set -euo pipefail` 下字段提取链提前失败）。
4. `max_file_size: 5` 且内容超长：退出码 `2`，stderr 输出“超过大小限制”。

该结果说明脚本对“必填字段缺失”没有降级路径。

### 4) 命令与工具

核心命令栈：`sed`, `grep`, `jq`, bash 条件判断。  
同技能目录可复用的工具脚本：

```bash
bash plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh .claude/my-plugin.local.md enabled
bash plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh .claude/my-plugin.local.md
```

## 关键代码路径与文件引用

1. 目标文件  
   `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh`

2. 上游规范  
   `plugins/plugin-dev/skills/plugin-settings/SKILL.md:60-97`, `136-171`, `517-523`  
   `plugins/plugin-dev/README.md:114-133`, `286-293`

3. 协议与测试工具  
   `plugins/plugin-dev/skills/hook-development/SKILL.md:278-317`  
   `plugins/plugin-dev/skills/hook-development/examples/validate-write.sh:19-35`  
   `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:30-44`  
   `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:86-97`

4. 同目录与下游关系  
   `plugins/plugin-dev/skills/plugin-settings/examples/example-settings.md`（配置样例）  
   `plugins/plugin-dev/skills/plugin-settings/examples/create-settings-command.md`（配置创建模板）  
   `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`  
   `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`

## 依赖与外部交互

1. 系统依赖  
   必需：`bash`, `sed`, `grep`, `jq`。  
   缺失 `jq` 时脚本会在解析输入时失败。

2. Claude Hook 运行时交互  
   通过 stdin 读事件 JSON；通过 stderr 输出阻断原因；通过退出码区分 allow/deny。

3. 配置文件交互  
   读取相对路径 `.claude/my-plugin.local.md`，依赖执行时 `cwd` 为项目根。

4. 生态工具交互  
   可被 `hook-development` 的 `test-hook.sh` 进行离线测试，也可被 `hook-linter.sh` 做规则检查。

## 风险、边界与改进建议

1. 字段缺失会被 `set -euo pipefail` 放大为硬失败  
   `grep '^strict_mode:'` 未命中时返回 1，导致脚本整体退出 1（实测复现）。  
   建议：字段提取加 `|| true`，并在缺省时回落到默认值（如 `strict_mode=false`）。

2. 配置健壮性不足  
   `enabled`、`max_file_size` 非法值没有统一告警和降级策略。  
   建议：先调用 `validate-settings.sh` 或在脚本内增加显式类型校验与默认值回退。

3. 路径策略较粗粒度  
   当前匹配基于字符串包含（如 `*".env"*`、`*"secret"*`），可能误伤正常文件名。  
   建议：改为更精确规则（后缀、目录白名单、正则边界）。

4. 输出 JSON 构造可维护性一般  
   现在依赖字符串拼接，后续新增字段易转义出错。  
   建议：改为 `jq -n --arg ...` 构造 JSON，避免手工转义。

5. 可移植性与路径上下文约束  
   依赖相对路径与当前目录，若 Hook 运行目录变化会读不到配置。  
   建议：优先使用 `$CLAUDE_PROJECT_DIR/.claude/...` 或在脚本开头显式归一化路径。

