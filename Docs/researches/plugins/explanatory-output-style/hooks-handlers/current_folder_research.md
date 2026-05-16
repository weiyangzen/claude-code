# DIR `plugins/explanatory-output-style/hooks-handlers` 研究文档

## 场景与职责

`plugins/explanatory-output-style/hooks-handlers` 是插件的 Hook 处理器目录，当前仅包含 1 个可执行脚本：`session-start.sh`。它的职责不是做业务逻辑计算，而是在会话启动（`SessionStart`）时向 Claude 注入额外上下文（`additionalContext`），从而把回答风格切换为“解释型输出”。

在插件链路中的位置：
1. 上游调用方
- `plugins/explanatory-output-style/hooks/hooks.json` 将 `SessionStart` 事件绑定到 `${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`。
- Claude Code Hook 运行时在会话开始阶段执行该 command hook。

2. 下游被调用方/影响面
- Claude 会将脚本输出 JSON 中的 `hookSpecificOutput.additionalContext` 合并进会话上下文，影响后续回答风格。
- 不涉及仓库源码写入、不涉及网络请求、不涉及外部服务 API。

3. 目录边界
- 目录只负责“输出风格注入文本”。
- 不负责插件安装与启停、不负责 Hook 事件路由、不负责权限拦截（如 `permissionDecision`）。

## 功能点目的

### 1) 复刻废弃的 Explanatory Output Style
- 插件 README 明确该插件用于复刻已废弃的 Explanatory output style（`plugins/explanatory-output-style/README.md:3-4,44-56`）。
- 处理器目录通过 SessionStart 注入上下文，实现“风格迁移到插件机制”的关键执行点。

### 2) 强制引导“任务 + 教学”双目标输出
- `session-start.sh` 在 `additionalContext` 内要求模型在写代码前后提供简短 insight 区块，强调“讲解实现选择、聚焦当前代码库”。
- 这使输出行为更偏向教学解释，而不是仅给结论。

### 3) 保持实现最小化、可预测
- 脚本是静态 JSON 回传（heredoc），没有条件分支、没有输入解析、没有动态依赖。
- 实现简单，故障面较小，行为稳定可复现。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 插件发现与启用
- marketplace 声明 `source: ./plugins/explanatory-output-style`（`.claude-plugin/marketplace.json:51-59`）。
- 插件元数据位于 `.claude-plugin/plugin.json`（`plugins/explanatory-output-style/.claude-plugin/plugin.json:1-8`）。

2. Hook 事件绑定
- `hooks/hooks.json` 采用插件包装格式：顶层 `description` + `hooks`。
- 在 `hooks.SessionStart[0].hooks[0]` 上声明 command：`${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`（`plugins/explanatory-output-style/hooks/hooks.json:1-15`）。

3. 处理器执行
- `session-start.sh` 被执行后，直接输出 JSON：
  - `hookSpecificOutput.hookEventName = "SessionStart"`
  - `hookSpecificOutput.additionalContext = <解释型长文本指令>`
- 末尾 `exit 0` 表示成功（`plugins/explanatory-output-style/hooks-handlers/session-start.sh:6-15`）。

4. 运行时效果
- Claude 将 `additionalContext` 注入会话上下文，持续影响当前会话内回答风格。
- 这是“上下文注入”机制，不是“工具调用决策”机制。

### B. 关键数据结构

处理器输出结构（核心）：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "..."
  }
}
```

字段说明：
- `hookSpecificOutput.hookEventName`：声明该输出对应的 hook 事件。
- `hookSpecificOutput.additionalContext`：注入给模型的附加系统上下文文本。

对应实现：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-11`。

### C. 协议与约定

1. 输入协议（来自 Claude Code）
- command hook 通过 stdin 接收 JSON，常见字段如 `session_id`、`cwd`、`hook_event_name`（参考 `plugins/plugin-dev/skills/hook-development/SKILL.md:300-320`）。
- 当前脚本未消费 stdin，因此输出与输入内容无关。

2. 输出与退出码约定
- 退出码 `0` 表示成功；`2` 常用于阻断型 hook；其他退出码视为异常（参考 `plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`，以及测试器 `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:205-252`）。
- 当前脚本固定 `exit 0`，属于纯成功注入路径。

3. 路径可移植协议
- 通过 `${CLAUDE_PLUGIN_ROOT}` 定位处理器，避免硬编码绝对路径（`plugins/explanatory-output-style/hooks/hooks.json:9`）。

### D. 命令级实测（本次研究执行）

1. 生成 SessionStart 样例输入

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/explanatory-sessionstart-input.json
```

结果：成功生成包含 `hook_event_name: "SessionStart"` 的测试输入（生成模板见 `test-hook.sh:83-92`）。

2. 测试目标处理器

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh \
  plugins/explanatory-output-style/hooks-handlers/session-start.sh \
  /tmp/explanatory-sessionstart-input.json
```

结果：
- Exit Code: `0`
- 输出 JSON 可被 `jq` 解析
- 包含 `hookSpecificOutput.hookEventName` 与 `hookSpecificOutput.additionalContext`

3. 对目标脚本运行 linter

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh \
  plugins/explanatory-output-style/hooks-handlers/session-start.sh
```

结果（关键点）：
- 1 条 warning：缺少 `set -euo pipefail`
- 其余检查通过

4. 校验 hooks 配置（上下文依赖验证）

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh \
  plugins/explanatory-output-style/hooks/hooks.json
```

结果：脚本对 `description/hooks` 顶层键报 Unknown，随后 `jq` 报错 `Cannot index string with number`。说明该校验器实现仍按“事件直写顶层”遍历，不兼容插件包装格式（`{"hooks": {...}}`）。

## 关键代码路径与文件引用

### 目标目录（被研究对象）
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`

### 调用方（上游）
- `plugins/explanatory-output-style/hooks/hooks.json:1-15`
- `plugins/explanatory-output-style/.claude-plugin/plugin.json:1-8`
- `.claude-plugin/marketplace.json:51-59`

### 被调用方/同链路依赖（下游或运行时）
- Claude Code Hook runtime（SessionStart 生命周期）
- Shell 执行环境：`#!/usr/bin/env bash`

### 同类实现（对照）
- `plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`
- `plugins/learning-output-style/hooks/hooks.json:1-15`

### 配置/测试/脚本/文档上下文
- 业务文档：`plugins/explanatory-output-style/README.md:1-72`
- 插件总览：`plugins/README.md:13-27`
- Hook 配置格式：`plugins/plugin-dev/skills/hook-development/SKILL.md:60-80`
- Hook 输入输出与退出码：`plugins/plugin-dev/skills/hook-development/SKILL.md:278-338`
- 测试器：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-252`
- 校验器：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-158`
- Linter：`plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`

## 依赖与外部交互

### 运行依赖
1. `bash` 解释器
- 处理器脚本依赖 `#!/usr/bin/env bash`。

2. Claude Code Hook 生命周期
- 依赖会话启动时触发 `SessionStart`。

3. 环境变量
- 间接依赖 `${CLAUDE_PLUGIN_ROOT}`（在 `hooks/hooks.json` 中用于定位脚本）。
- 当前脚本本身不读取 `CLAUDE_*` 变量，也不写 `CLAUDE_ENV_FILE`。

### 开发与验证依赖
- `jq`：用于测试器和校验器的 JSON 解析。
- `test-hook.sh`：用于本地模拟 Hook 调用。
- `hook-linter.sh`：用于静态质量检查。

### 外部交互
- 无 HTTP 请求、无第三方 SDK 调用、无数据库交互。
- 唯一显著副作用：每次会话增加额外 prompt token（README 已明确 warning，`plugins/explanatory-output-style/README.md:6-7`）。

## 风险、边界与改进建议

### 风险
1. token 成本与上下文拥挤
- `additionalContext` 长文本在每次会话固定注入，短任务场景可能出现“收益低于开销”。

2. 工具链误报风险
- 通用校验脚本 `validate-hook-schema.sh` 当前对插件包装格式兼容不足，可能造成“配置可运行但校验失败”的误判。

3. 文案演进漂移
- `learning-output-style` 也内嵌 explanatory 文案，两个插件并行维护可能出现行为不一致。

4. 脚本鲁棒性基线偏弱
- 目标脚本未设置 `set -euo pipefail`（虽当前逻辑简单，仍有规范一致性风险）。

### 边界
1. 不做权限控制
- 不输出 `permissionDecision`，不参与 PreToolUse 拦截。

2. 不做输入驱动逻辑
- 不读取 stdin JSON，不根据项目状态动态定制上下文。

3. 不做环境持久化
- 不写 `CLAUDE_ENV_FILE`，仅执行静态上下文注入。

### 改进建议
1. 增强脚本稳健性
- 为 `session-start.sh` 增加 `set -euo pipefail`，与仓库 Hook 实践保持一致。

2. 抽离可复用提示模板
- 将 explanatory insight 文案提取为共享模板，供 `explanatory-output-style` 与 `learning-output-style` 复用，降低漂移。

3. 为该目录补充最小 smoke test
- 断言脚本输出可解析 JSON、`hookEventName == SessionStart`、`additionalContext` 非空。

4. 修复/扩展校验脚本
- 在 `validate-hook-schema.sh` 增加对插件包装格式（`{"hooks": {...}}`）的自动下钻解析，避免误报。
