# DIR 研究：plugins/plugin-dev/skills/hook-development/scripts

## 场景与职责

`plugins/plugin-dev/skills/hook-development/scripts` 是 `hook-development` 技能的工程化工具层，职责是把 Hook 开发从“写配置/写脚本”推进到“可校验、可本地仿真、可静态体检”。

目录内 3 个脚本形成最小闭环：

- `validate-hook-schema.sh`：校验 `hooks.json` 结构与字段合法性。
- `test-hook.sh`：离线模拟 Claude Hook 运行时输入、环境变量与退出码语义。
- `hook-linter.sh`：对 Hook 脚本做静态规则检查（安全与最佳实践）。

上下文依赖关系：

- 上游调用方（流程/文档）：
  - `plugins/plugin-dev/README.md:249-257,273-284` 将本目录脚本放在开发流程的 “Add Automation / Test & Validate” 阶段。
  - `plugins/plugin-dev/commands/create-plugin.md:201-209,257-260` 在 Phase 5/6 明确要求使用 `validate-hook-schema.sh` 与 `test-hook.sh`。
  - `plugins/plugin-dev/agents/plugin-validator.md:107-115` 指定 hooks 校验应使用该工具链。
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:685-689,707-709` 将本目录列为 utility scripts 并纳入推荐实现流程。
- 下游被调用方（运行对象）：
  - 插件中的 `hooks/hooks.json`。
  - 实际 Hook 执行脚本（如 `examples/*.sh` 或插件 `hooks-handlers/*.sh`）。
  - Claude Hook 运行时协议（stdin JSON + exit code + 环境变量）。

结论：该目录并不直接实现业务 Hook，而是作为 Hook 生态的验证与调试基础设施。

## 功能点目的

### 1) `validate-hook-schema.sh`

目的：在 Hook 配置进入运行时之前，提前发现结构/字段错误与高风险配置，降低会话启动失败和线上误拦截。

覆盖点（按脚本实际实现）：

- JSON 语法校验。
- 事件名合法性提醒。
- `matcher`、`hooks[]`、`type`、`command/prompt` 必填校验。
- `timeout` 数值和范围检查。
- 命令 Hook 绝对路径提示（建议 `${CLAUDE_PLUGIN_ROOT}`）。

### 2) `test-hook.sh`

目的：在不启动 Claude Code 的情况下复现实例化 Hook 执行，验证输入、输出、退出码、超时与环境文件行为。

核心价值：

- 支持 `--create-sample <event>` 生成测试输入。
- 支持超时控制与 verbose 调试输出。
- 自动注入 `CLAUDE_PROJECT_DIR` / `CLAUDE_PLUGIN_ROOT` / `CLAUDE_ENV_FILE`。
- 对 Hook 执行结果给出“approve/block/timeout/异常”语义解释。

### 3) `hook-linter.sh`

目的：在脚本级别做低成本静态体检，优先发现可移植性与安全性问题。

检查项包括：

- shebang、可执行位、`set -euo pipefail`。
- stdin 读取习惯、`jq` 使用、变量引用、错误输出到 stderr。
- 硬编码路径、退出码约定、潜在长耗时代码。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. `validate-hook-schema.sh` 关键流程

代码位置：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1-159`

流程：

1. 参数与文件存在性校验（`8-25`）。
2. `jq empty` 做 JSON 语法检查（`30-37`）。
3. 定义 `VALID_EVENTS`，遍历顶层 key 判断事件名（`41-56`）。
4. 按 `事件 -> 条目 -> hooks[]` 三层遍历校验字段与类型（`65-146`）。
5. 统计 `error_count/warning_count` 并输出总结（`148-159`）。

数据结构假设（脚本当前实现）：

```json
{
  "PreToolUse": [
    {
      "matcher": "Write|Edit",
      "hooks": [
        {"type": "command", "command": "...", "timeout": 10}
      ]
    }
  ]
}
```

实测要点：

- 对插件常见 wrapper 结构 `{"description":...,"hooks":{...}}` 不兼容，直接校验会在 `jq` 下钻时报错退出（exit 5）。
- 在出现 warning（如 timeout > 600）时，脚本当前会直接以非 0 退出，未进入汇总输出。

### B. `test-hook.sh` 关键流程

代码位置：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:1-252`

流程：

1. 参数解析：`-h/-v/-t/--create-sample`（`102-127`）。
2. `create_sample()` 生成事件样例 JSON（`25-100`）。
3. 校验 Hook 脚本与输入文件存在性（`138-152`），并用 `jq empty` 验证输入 JSON（`154-158`）。
4. 注入测试环境变量（`170-174`）。
5. 使用 `timeout` + `bash -c` 执行 Hook，并捕获 stdout/stderr 与退出码（`189-191`）。
6. 依据退出码映射行为语义并输出结果（`205-252`）。

协议与命令约定：

- 输入协议：stdin JSON。
- 退出码解释：`0` 通过、`2` 阻断、`124` 超时、其他视为异常。
- 输出处理：若 stdout 是 JSON，会二次 `jq` pretty print。

实测要点：

- `--create-sample PreToolUse` 生成结果符合 Hook 输入字段约定。
- `SessionEnd` 样例实际写死为 `hook_event_name: "SessionStart"`；`SubagentStop` 样例写死为 `"Stop"`（事件标签不一致）。
- 输入必须是常规文件路径（`-f` 校验），不支持 process substitution 这种非常规路径。

### C. `hook-linter.sh` 关键流程

代码位置：`plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:1-153`

流程：

1. 接收一个或多个脚本路径（`8-21`）。
2. 对每个脚本执行 13 类检查（`23-132`）。
3. 汇总“存在 error 的脚本数”，决定最终 exit code（`138-153`）。

技术特征：

- 规则主要基于 `grep`/regex 启发式匹配，成本低、覆盖广，但准确率受限。
- warning 不影响最终退出码；error 会导致整体失败。

### D. 与配置/文档/测试链路的耦合

- `scripts/README.md:5-164` 定义了三脚本的标准调用顺序（lint -> sample -> test -> schema validate -> claude --debug）。
- `SKILL.md` 提供 Hook 输入输出协议与环境变量背景（`278-338`），脚本按该协议实现本地仿真。
- `create-plugin.md` 和 `plugin-validator.md` 把这些脚本嵌入插件构建与验收流程，属于“治理链路”而非单点工具。

## 关键代码路径与文件引用

目标目录文件：

- `plugins/plugin-dev/skills/hook-development/scripts/README.md`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`

直接上游（调用/规范）：

- `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,278-338,685-709`
- `plugins/plugin-dev/README.md:249-257,273-284`
- `plugins/plugin-dev/commands/create-plugin.md:201-209,257-260`
- `plugins/plugin-dev/agents/plugin-validator.md:107-115`

直接下游（被校验/被测试对象）：

- `plugins/*/hooks/hooks.json`（如 `plugins/security-guidance/hooks/hooks.json`、`plugins/hookify/hooks/hooks.json`）
- `plugins/plugin-dev/skills/hook-development/examples/*.sh`
- 其它插件 Hook handler 脚本（例如 `plugins/explanatory-output-style/hooks-handlers/session-start.sh`）

## 依赖与外部交互

运行依赖：

- 必需：`bash`、`jq`。
- `test-hook.sh` 额外依赖：`timeout`、`date`。
- `hook-linter.sh` 依赖：`grep`、`head` 等标准 Unix 工具。

协议依赖：

- stdin JSON 字段：`session_id`、`cwd`、`hook_event_name`、`tool_name`、`tool_input`、`reason` 等。
- 环境变量：`CLAUDE_PROJECT_DIR`、`CLAUDE_PLUGIN_ROOT`、`CLAUDE_ENV_FILE`。
- 退出码语义：与 Hook 运行时约定绑定（0/2/timeout）。

外部交互方式：

- 文件系统：读取 `hooks.json`、读取测试输入、可读取/删除 `CLAUDE_ENV_FILE`。
- 子进程：通过 `bash -c` 执行 Hook 脚本，`timeout` 包裹执行。
- 无远程网络调用，但可间接触发被测 Hook 脚本中的外部行为。

测试现状：

- 目录自身无 Bats/pytest 等自动化测试套件。
- 以工具脚本自检 + 手工场景测试为主，适合开发期 smoke test，不等价于系统化回归。

## 风险、边界与改进建议

1. 配置格式兼容风险（高优先级）
- 现象：`SKILL.md` 声明插件推荐 wrapper 格式 `{"hooks": {...}}`，而 `validate-hook-schema.sh` 仅按“事件在顶层”遍历。
- 影响：对仓库内多个真实插件 `hooks/hooks.json` 直接误报并异常退出（实测 exit 5）。
- 建议：在校验脚本中自动识别两种根结构（wrapper/direct），统一下钻到事件对象再校验。

2. 错误计数实现导致提前退出（高优先级）
- 现象：`validate-hook-schema.sh` 在首次 error/warning 后可能直接退出，无法输出完整问题清单与汇总。
- 影响：诊断信息不完整，降低修复效率；warning 被误判成失败。
- 建议：计数改为 `((error_count+=1)) || true` / `error_count=$((error_count+1))`，并确保始终进入汇总分支。

3. 样例事件标签不一致（中优先级）
- 现象：`test-hook.sh --create-sample SessionEnd` 生成 `SessionStart`；`SubagentStop` 生成 `Stop`。
- 影响：测试样本与目标事件不一致，易造成误测。
- 建议：按请求事件名原样输出 `hook_event_name`，并增加针对每个事件的样例一致性自检。

4. 测试输入形态边界（中优先级）
- 现象：`test-hook.sh` 仅接受常规文件（`-f`），不支持 stdin pipe / process substitution。
- 影响：CI 或一次性命令组合时可用性受限。
- 建议：支持 `-` 表示 stdin，并在 README 明确输入约束。

5. linter 规则的误报/漏报边界（中优先级）
- 现象：`hook-linter.sh` 基于简化 regex（如变量引用检测、路径检测），复杂脚本下可能误报或漏报。
- 影响：warning 信噪比不稳定，团队可能忽视真正问题。
- 建议：
  - 引入 `shellcheck` 作为可选增强模式。
  - 将当前规则区分为 `hard error` 与 `advisory` 两级。

6. 文档与实现一致性治理建议（中优先级）
- 现象：README/SKILL/脚本行为在少数细节上存在偏差（配置根结构、样例事件名、失败条件）。
- 建议：
  - 在 `scripts/README.md` 增加“当前已知边界”。
  - 增加最小回归脚本，覆盖 wrapper/direct、warning-only、每个 `--create-sample` 事件。

边界说明：本目录定位是“开发期验证工具集”，不能替代运行时集成测试与真实 Claude 会话验证；最终上线前仍需 `claude --debug` 实测 Hook 注册与执行日志。
