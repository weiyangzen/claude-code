# FILE `plugins/explanatory-output-style/.claude-plugin/plugin.json` 研究文档

## 场景与职责

`plugins/explanatory-output-style/.claude-plugin/plugin.json` 是 `explanatory-output-style` 插件的 manifest（插件清单）入口。

它在链路中的职责是“可发现性 + 身份声明”，不是运行逻辑实现：

1. 让插件被 Claude Code 识别
- 规范要求 manifest 必须位于 `.claude-plugin/plugin.json`，否则插件不会被识别（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- 自动发现流程第一步即读取此 manifest（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`，`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。

2. 提供插件身份与元数据
- 当前声明了 `name/version/description/author`（`plugins/explanatory-output-style/.claude-plugin/plugin.json:2-8`）。
- 插件总览对该插件的定位与清单描述一致（`plugins/README.md:19`）。

3. 作为下游 Hook 链路生效的前置条件
- manifest 本身不含 `hooks` 字段，但按默认规则会加载 `./hooks/hooks.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`）。
- 下游 `hooks/hooks.json` 再把 `SessionStart` 绑定到 `hooks-handlers/session-start.sh`（`plugins/explanatory-output-style/hooks/hooks.json:4-10`）。

## 功能点目的

### 1) 标识插件唯一身份并参与冲突检测
- `name: "explanatory-output-style"` 符合 kebab-case 规则（规则：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-36`；实值：`plugins/explanatory-output-style/.claude-plugin/plugin.json:2`）。
- 该名称同时用于 marketplace 条目匹配（`.claude-plugin/marketplace.json:51`）。

### 2) 声明版本与描述，承接“旧 Output Style -> 插件”迁移语义
- `description` 明确“mimics the deprecated Explanatory output style”（`plugins/explanatory-output-style/.claude-plugin/plugin.json:4`）。
- README 同步声明该插件复刻已废弃 Explanatory 风格（`plugins/explanatory-output-style/README.md:3-4,44-53`）。
- CHANGELOG 记录了 output styles 的发布与弃用背景（`CHANGELOG.md:1837,1528`）。

### 3) 维持“最小 manifest”并依赖默认自动发现
- 当前 manifest 未设置 `commands/agents/skills/hooks/mcpServers`（`plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9`）。
- 这与仓库中多数官方插件一致（多插件 manifest 仅保留元数据字段）。
- 依据规范，此做法可减少路径配置复杂度，默认加载标准目录（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:365-368`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:214-215,247-248,262,301`）。

### 4) 提供作者归属与可追溯信息
- `author.name/email` 已填（`plugins/explanatory-output-style/.claude-plugin/plugin.json:5-7`）。
- 字段语义在 manifest 参考中定义（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:87-114`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 本文件 -> 被调用方）

1. 调用方（发现入口）
- marketplace 声明插件源目录 `./plugins/explanatory-output-style`（`.claude-plugin/marketplace.json:58`）。
- 插件系统启动时扫描已启用插件并读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。

2. 本文件（manifest）
- 提供插件元信息，确认插件身份（`plugins/explanatory-output-style/.claude-plugin/plugin.json:2-8`）。

3. 被调用方（默认装配）
- 默认读取 `hooks/hooks.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`）。
- `hooks/hooks.json` 在 `SessionStart` 执行 command hook：`${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`（`plugins/explanatory-output-style/hooks/hooks.json:4-10`）。
- `session-start.sh` 输出 `hookSpecificOutput.additionalContext` 注入会话上下文（`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-11`）。

### B. 数据结构

当前 manifest 数据结构：

```json
{
  "name": "explanatory-output-style",
  "version": "1.0.0",
  "description": "Adds educational insights about implementation choices and codebase patterns (mimics the deprecated Explanatory output style)",
  "author": {
    "name": "Dickson Tsai",
    "email": "dickson@anthropic.com"
  }
}
```

字段对应规范：
- `name`：必填，kebab-case（`manifest-reference.md:15-36`）
- `version`：语义化版本（`manifest-reference.md:42-53`）
- `description`：用途摘要（`manifest-reference.md:65-77`）
- `author`：作者信息对象（`manifest-reference.md:87-99`）

### C. 协议与命令（围绕本文件的可验证链路）

1. Hook 配置协议
- 插件的 `hooks/hooks.json` 使用 wrapper 格式 `{"description":...,"hooks":{...}}`（`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`）。

2. Hook 输出协议
- `SessionStart` handler 输出 `hookSpecificOutput.additionalContext`（`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-11`）。
- 通用退出码约定：`0` 成功，`2` 阻断（`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`）。

3. 本次命令实测
- 生成 SessionStart 样例：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/explanatory_sessionstart_input.json`
  - 结果：`hook_event_name = SessionStart`。
- 运行下游 handler：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh /tmp/explanatory_sessionstart_input.json`
  - 结果：exit code `0`，输出 JSON 可解析，包含 `hookSpecificOutput.hookEventName` 与 `additionalContext`。
- 运行 schema 校验器：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/explanatory-output-style/hooks/hooks.json`
  - 结果：报 `Unknown event type: description/hooks`，随后 `jq` 报 `Cannot index string with number`（exit `5`），暴露“校验脚本按顶层事件键遍历，不兼容插件 wrapper 格式”的问题（脚本逻辑见 `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`）。
- 运行 hook linter：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh`
  - 结果：通过，1 条 warning（缺少 `set -euo pipefail`）。

## 关键代码路径与文件引用

### 目标文件
- `plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9`

### 上游调用方与发现配置
- `.claude-plugin/marketplace.json:51-60`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`

### 下游被调用方
- `plugins/explanatory-output-style/hooks/hooks.json:1-15`
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`
- `plugins/explanatory-output-style/README.md:18-33`

### 对照与关联文件
- `plugins/README.md:19`（插件目录总览说明）
- `plugins/learning-output-style/README.md:5,80`（声明包含 explanatory 功能）
- `plugins/learning-output-style/.claude-plugin/plugin.json:1-9`（同类最小 manifest）

### 测试/脚本/文档
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100,170-252`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`
- `CHANGELOG.md:1528,1837,1908`

## 依赖与外部交互

### 依赖

1. Claude Code 插件发现机制
- 负责识别 `.claude-plugin/plugin.json` 并触发组件发现。

2. 默认组件路径约定
- 本文件未显式配置路径，依赖默认 `hooks/hooks.json` 加载规则（`manifest-reference.md:262`）。

3. Shell 执行环境（间接）
- 运行时会通过 hooks 执行 `session-start.sh`，因此间接依赖 `bash`。

4. 环境变量约定（间接）
- `hooks/hooks.json` 使用 `${CLAUDE_PLUGIN_ROOT}` 定位脚本（`plugins/explanatory-output-style/hooks/hooks.json:9`，规范见 `plugins/plugin-dev/skills/hook-development/SKILL.md:327-337`）。

### 外部交互

- `plugin.json` 本身无网络、无子进程、无文件读写副作用。
- 间接效果是启用后在每个 SessionStart 注入额外上下文，增加 token 消耗（`plugins/explanatory-output-style/README.md:6-7`）。

## 风险、边界与改进建议

### 风险

1. 单点可发现性风险
- `plugin.json` 是插件发现入口；JSON 结构损坏会导致整插件不可用。

2. 元数据双源漂移风险
- `plugin.json` 与 `.claude-plugin/marketplace.json` 都维护 `name/version/description/author`，存在同步偏差风险。

3. 校验工具错配风险
- `validate-hook-schema.sh` 与插件 wrapper 格式兼容不足，可能给出误导性失败结果。

4. 运行成本风险（间接）
- manifest 触发的默认 hooks 链路会在 SessionStart 注入长上下文，持续增加 token。

### 边界

1. 本文件只定义元数据，不定义行为逻辑
- 不直接声明 hook 文本、不直接执行命令、不处理 stdin。

2. 本文件不控制动态运行策略
- 不涉及 matcher/timeout/permissionDecision 等 Hook 行为参数。

3. 本文件不承担安装与分发实现
- 分发入口在 marketplace 清单，运行行为在 hooks/handler。

### 改进建议

1. 增加 manifest 自动校验
- 在 CI 中对 `plugins/*/.claude-plugin/plugin.json` 做 JSON schema + name 正则校验。

2. 增加 marketplace/manifest 一致性检查
- 自动比较 `name/version/description/author`，防止双源漂移。

3. 修复 Hook schema 校验脚本
- `validate-hook-schema.sh` 应先探测并下钻 `.hooks`，同时兼容“插件 wrapper 格式”和“settings 直写格式”。

4. 补充最小回归测试
- 针对该插件增加 smoke test：manifest 可解析、hooks 路径可加载、SessionStart handler 输出协议字段存在。
