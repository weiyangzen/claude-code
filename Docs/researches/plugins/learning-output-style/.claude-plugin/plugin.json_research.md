# FILE `plugins/learning-output-style/.claude-plugin/plugin.json` 研究文档

## 场景与职责

`plugins/learning-output-style/.claude-plugin/plugin.json` 是 `learning-output-style` 插件的 manifest 入口文件。

它在运行链路中的职责是“可发现性 + 身份声明”，不是行为执行：

1. 让插件被 Claude Code 识别。
- manifest 必须位于 `.claude-plugin/plugin.json`，否则插件不会被识别（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-48`）。
- 自动发现第一步就是读取这个文件（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`）。

2. 声明插件身份与元数据。
- 当前定义 `name/version/description/author`（`plugins/learning-output-style/.claude-plugin/plugin.json:2-8`）。
- 仓库总览与 marketplace 对其定位一致（`plugins/README.md:23`，`.claude-plugin/marketplace.json:95-103`）。

3. 作为下游 Hook 链路生效的前置条件。
- 本 manifest 没有显式 `hooks` 字段，依赖默认路径 `./hooks/hooks.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263,356-361`）。
- 下游 `hooks/hooks.json` 把 `SessionStart` 绑定到 `hooks-handlers/session-start.sh`（`plugins/learning-output-style/hooks/hooks.json:4-10`）。

## 功能点目的

### 1) 唯一标识插件并参与冲突检测
- `name: "learning-output-style"` 符合 kebab-case 规则（规则见 `manifest-reference.md:15-36`；实值见 `plugin.json:2`）。

### 2) 承载插件版本与用途语义
- `description` 明确该插件是“交互学习 + 未发布 Learning 风格迁移”能力（`plugin.json:4`，`plugins/learning-output-style/README.md:3-5,76-82`）。
- 该定位与 output-style 历史演进一致：
  - 早期发布 Learning/Explanatory output styles（`CHANGELOG.md:1837`）
  - 后续废弃 output styles 并建议用插件/系统提示（`CHANGELOG.md:1528`）
  - 近期明确 output style 在会话开始固定（`CHANGELOG.md:206`）

### 3) 通过“最小 manifest”启用默认自动发现
- 仅保留元数据字段，不配置 `commands/agents/skills/hooks/mcpServers`（`plugin.json:1-9`）。
- 依赖默认目录扫描与注册顺序（`manifest-reference.md:356-371`，`plugin-structure/SKILL.md:343-355`）。

### 4) 提供归属信息与可追溯性
- `author.name/email` 已填（`plugin.json:5-7`），字段语义见规范（`manifest-reference.md:87-114`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 本文件 -> 被调用方）

1. 调用方（上游）
- marketplace 条目指向插件根目录 `./plugins/learning-output-style`（`.claude-plugin/marketplace.json:95-103`）。
- Claude Code 启用插件时读取 `.claude-plugin/plugin.json`（`component-patterns.md:11-16`）。

2. 目标对象（本文件）
- 解析 manifest，注册插件身份与元数据（`plugin.json:2-8`）。

3. 被调用方（下游）
- 按默认规则加载 `hooks/hooks.json`（`manifest-reference.md:259-263`）。
- `SessionStart` 触发 command hook：`${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`（`hooks/hooks.json:4-10`）。
- `session-start.sh` 输出 `hookSpecificOutput.additionalContext` 注入会话（`hooks-handlers/session-start.sh:8-12`）。

### B. 数据结构

manifest 当前结构：

```json
{
  "name": "learning-output-style",
  "version": "1.0.0",
  "description": "Interactive learning mode that requests meaningful code contributions at decision points (mimics the unshipped Learning output style)",
  "author": {
    "name": "Boris Cherny",
    "email": "boris@anthropic.com"
  }
}
```

字段规范：
- `name`：必填、kebab-case（`manifest-reference.md:15-36`）
- `version`：semver（`manifest-reference.md:42-63`）
- `description`：用途说明（`manifest-reference.md:65-83`）
- `author`：作者信息对象（`manifest-reference.md:87-114`）

### C. 协议与命令

1. 发现/加载协议
- manifest 固定位置：`.claude-plugin/plugin.json`（`manifest-reference.md:7-10`）。
- 默认扫描顺序：先默认目录，再 manifest 自定义路径（`manifest-reference.md:356-371`）。

2. Hook 配置协议（下游）
- 插件 `hooks/hooks.json` 使用 wrapper 格式：`{"description":..., "hooks": {...}}`（`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`）。

3. Hook 输出协议（下游）
- command hook 通过 stdout 返回 JSON；`0` 表示成功（`hook-development/SKILL.md:294-299`）。
- `SessionStart` 场景输出 `hookSpecificOutput.hookEventName/additionalContext`（`hooks-handlers/session-start.sh:8-12`）。

### D. 实测命令与结果

1. 生成 SessionStart 样例输入
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/learning_plugin_sessionstart_input.json
jq -r '.hook_event_name' /tmp/learning_plugin_sessionstart_input.json
```
结果：`SessionStart`。

2. 执行下游 handler 冒烟测试
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh \
  plugins/learning-output-style/hooks-handlers/session-start.sh \
  /tmp/learning_plugin_sessionstart_input.json
```
结果：`Exit Code: 0`，输出 JSON 可解析，包含 `hookSpecificOutput.hookEventName=SessionStart` 和 `additionalContext`。

3. 校验 hooks schema（观察工具兼容性）
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh \
  plugins/learning-output-style/hooks/hooks.json
```
结果：出现 `Unknown event type: description/hooks`，随后 `jq: Cannot index string with number`，退出码 `5`。根因是该脚本按“事件在顶层”遍历（`validate-hook-schema.sh:41-67`），与插件 wrapper 格式不兼容。

4. 脚本 lint
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh \
  plugins/learning-output-style/hooks-handlers/session-start.sh
```
结果：通过，2 条 warning（缺少 `set -euo pipefail`、stderr 规范提醒）。

## 关键代码路径与文件引用

### 目标文件
- `plugins/learning-output-style/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:95-103`（source 路径）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`（发现阶段）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`（自动发现顺序）

### 被调用方（下游）
- `plugins/learning-output-style/hooks/hooks.json:1-15`
- `plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`
- `plugins/learning-output-style/README.md:24-27,72-82`

### 配置、测试、脚本、文档上下文
- 配置：
  - `plugins/learning-output-style/.claude-plugin/plugin.json`
  - `plugins/learning-output-style/hooks/hooks.json`
  - `.claude-plugin/marketplace.json`
- 测试：插件目录内无 `test/spec` 文件；当前依赖通用 hook 工具链做验证。
- 脚本：
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100,170-252`
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`
  - `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`
- 文档：
  - `plugins/learning-output-style/README.md`
  - `plugins/README.md`
  - `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md`
  - `plugins/plugin-dev/skills/hook-development/scripts/README.md:5-22,29-61,62-82`

## 依赖与外部交互

### 依赖
1. Claude Code 插件发现机制：依赖 manifest 路径与字段合法性。
2. 默认组件路径约定：未显式配置时默认加载 `hooks/hooks.json`。
3. Hook 运行时与 shell 环境（间接）：下游 command hook 依赖 `bash`、环境变量 `${CLAUDE_PLUGIN_ROOT}`。
4. 工具链依赖（开发验证）：`jq`、`timeout`（见 `test-hook.sh`）。

### 外部交互
- `plugin.json` 本身无网络访问、无文件写副作用、无子进程执行。
- 间接外部效果是会话开始注入额外上下文，增加 token 消耗（`plugins/learning-output-style/README.md:7`）。

### 稳定性上下文
- 历史上 `SessionStart` 在 resume 场景曾出现重复触发问题，后续已修复（`CHANGELOG.md:192`）。
- `SessionStart` 执行被延后以优化启动性能（`CHANGELOG.md:597`）。

## 风险、边界与改进建议

### 风险
1. 可发现性单点风险：`plugin.json` 损坏会导致插件整体不可用。
2. 元数据双源漂移：`plugin.json` 与 `marketplace.json` 同时维护元数据，长期有不一致风险。
3. 校验工具错配：`validate-hook-schema.sh` 当前不兼容插件 wrapper，可能误导质量判断。
4. 运行成本风险（间接）：每次 `SessionStart` 注入长 `additionalContext`，固定增加 token 成本。
5. 文案漂移风险（间接）：与 `explanatory-output-style` 维护相似提示词，后续可能分叉。

### 边界
1. 本文件只定义元数据，不定义 hook 文案或执行逻辑。
2. 本文件不控制 matcher/timeout/permissionDecision 等运行策略。
3. 本文件不承担安装实现，仅作为发现与注册入口。

### 改进建议
1. 增加 manifest CI 校验：对 `plugins/*/.claude-plugin/plugin.json` 做 JSON/schema/name 正则校验。
2. 增加 marketplace 一致性检查：自动比对 `name/version/description/author/source`。
3. 修复 `validate-hook-schema.sh`：支持先下钻 `.hooks` 再做事件校验，兼容插件 wrapper 与 settings 直写两种格式。
4. 增加插件级 smoke test：校验 `SessionStart` handler 输出协议字段（`hookEventName/additionalContext`）稳定存在。
5. 共享提示词片段：抽取 explanatory 公共段落，降低多插件文案漂移。
