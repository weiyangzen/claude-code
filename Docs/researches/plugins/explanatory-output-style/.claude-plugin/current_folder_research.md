# plugins/explanatory-output-style/.claude-plugin 目录研究（DIR）

## 场景与职责

`plugins/explanatory-output-style/.claude-plugin` 是 `explanatory-output-style` 插件的 manifest 目录，当前仅包含一个文件：`plugin.json`（`plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9`）。

该目录不执行 Hook、不处理会话输入，也不直接产出提示词；它的核心职责是作为插件生命周期中的“发现入口 + 身份声明层”：

1. 让 Claude Code 在插件发现阶段识别该插件。  
   依据：manifest 必须位于 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`；`plugins/README.md:49-55`）。
2. 提供插件的基础元数据（名称、版本、描述、作者），用于安装展示、归属和冲突识别。  
   依据：`plugins/explanatory-output-style/.claude-plugin/plugin.json:2-8`。
3. 作为下游组件自动发现（本插件的 `hooks/hooks.json` 与 `hooks-handlers/session-start.sh`）的前置条件。  
   依据：发现流程会先读 manifest，再注册组件（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`）。

在仓库上下文中，它还是 marketplace 路由链路的一环：`.claude-plugin/marketplace.json` 通过 `source: "./plugins/explanatory-output-style"` 把插件根目录接入发现流程（`.claude-plugin/marketplace.json:51-60`）。

## 功能点目的

### 1) 插件身份与唯一标识
`plugin.json` 的 `name` 字段为 `explanatory-output-style`（`plugins/explanatory-output-style/.claude-plugin/plugin.json:2`），这是插件唯一标识，影响识别与冲突检测。  
`name` 格式规范为 kebab-case（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-36`）。

### 2) 元数据承载（发布、归属、说明）
本目录承载如下关键元数据：
- `version: "1.0.0"`（`plugins/explanatory-output-style/.claude-plugin/plugin.json:3`）
- `description`（`plugins/explanatory-output-style/.claude-plugin/plugin.json:4`）
- `author.name/email`（`plugins/explanatory-output-style/.claude-plugin/plugin.json:5-7`）

这些字段服务于插件展示、可追溯性和版本管理。

### 3) 默认组件发现策略确认
manifest 未声明 `commands/agents/skills/hooks/mcpServers` 的自定义路径（`plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9`），因此依赖默认目录自动发现：
- `hooks/hooks.json` 被作为 Hook 配置入口；
- `hooks-handlers/session-start.sh` 被 `hooks.json` 的 command Hook 调用。

对应机制说明见 `plugins/plugin-dev/skills/plugin-structure/SKILL.md:86-107,200-232`。

### 4) 与旧 Output Style 的迁移锚点
README 将该插件定位为“已废弃 Explanatory output style 的替代”（`plugins/explanatory-output-style/README.md:3-4,44-53`）；`.claude-plugin/plugin.json` 提供这个替代方案的可安装身份。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 目标目录 -> 被调用方）

1. 调用方（上游）
- marketplace 注册插件：`.claude-plugin/marketplace.json:51-60`。
- Claude Code 启动时读取每个插件的 `.claude-plugin/plugin.json` 并注册组件：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`。

2. 目标目录（本研究对象）
- `plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9` 提供元数据入口。

3. 被调用方（下游）
- Hook 配置：`plugins/explanatory-output-style/hooks/hooks.json:1-15`。
- Hook 处理器：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`。
- SessionStart 时输出 `hookSpecificOutput.additionalContext` 注入讲解型上下文：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-10`。

简化链路：
`marketplace source` -> `plugin root` -> `.claude-plugin/plugin.json` -> `hooks/hooks.json` -> `SessionStart command hook` -> `additionalContext` 注入。

### B. 数据结构

本目录核心数据结构是最小 manifest：

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

字段语义：
- `name`：唯一标识（`manifest-reference.md:15-31`）
- `version`：语义化版本（`manifest-reference.md:42-53`）
- `description`：用途摘要（`manifest-reference.md:65-77`）
- `author`：归属与联系方式（`manifest-reference.md:87-114`）

### C. 协议与命令（与该目录强相关）

虽然本目录不直接执行命令，但它是 Hook 协议生效前提：
- 插件 Hook 配置采用 wrapper 格式：`{"description": ..., "hooks": {...}}`（`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`；`plugins/explanatory-output-style/hooks/hooks.json:1-15`）。
- `SessionStart` 事件的 command Hook 通过 `${CLAUDE_PLUGIN_ROOT}` 引用脚本，保证可移植（`plugins/explanatory-output-style/hooks/hooks.json:8-10`；`plugins/plugin-dev/skills/hook-development/SKILL.md:331-338`）。
- Hook 脚本输出 JSON 到 stdout、返回码 0 表示成功（`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`；`plugins/explanatory-output-style/hooks-handlers/session-start.sh:6-15`）。

### D. 可验证命令与现状

围绕该目录下游组件执行了仓库内置脚本验证：

1. Hook 单元测试通过  
命令：
`bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh <SessionStart样例>`  
结果：exit code 0，输出可解析 JSON，含 `hookSpecificOutput.additionalContext`。

2. Hook linter 通过但有安全建议  
命令：
`bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh`  
结果：通过，提示缺少 `set -euo pipefail`（warning）。

3. Hook schema 校验脚本与插件格式存在兼容缺口  
命令：
`bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/explanatory-output-style/hooks/hooks.json`  
结果：`jq` 报错并退出（`Cannot index string with number`，exit code 5）。

原因：该脚本按“顶层事件键”遍历（`validate-hook-schema.sh:43,65-67`），而插件 hooks 使用 `hooks` 包装层；脚本未先进入 `.hooks` 节点，导致把 `description` 当成事件并在字符串上索引。

## 关键代码路径与文件引用

### 目标目录（直接对象）
- `plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:51-60`（插件注册与 source 路径）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`（启动发现流程）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 必须路径）
- `plugins/README.md:47-61`（标准目录结构）

### 被调用方（下游）
- `plugins/explanatory-output-style/hooks/hooks.json:1-15`（SessionStart Hook 配置）
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`（Hook 处理器实现）
- `plugins/explanatory-output-style/README.md:20-33`（工作原理与激活方式）

### 同类/演化关联
- `plugins/learning-output-style/README.md:3-6,80`（声明吸收 explanatory 功能）
- `plugins/learning-output-style/hooks/hooks.json:1-15`（同样的 Hook wrapper 结构）

### 校验与测试脚本
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100,170-252`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`

## 依赖与外部交互

### 1) 仓库内依赖
- 依赖 marketplace 条目把插件目录纳入安装/发现候选（`.claude-plugin/marketplace.json:51-60`）。
- 依赖插件发现器读取 `.claude-plugin/plugin.json`（`component-patterns.md:9-16`）。
- 依赖默认 hooks 目录自动发现（`plugin-structure/SKILL.md:200-232`）。
- 依赖 Hook 配置中的 `${CLAUDE_PLUGIN_ROOT}` 环境变量定位脚本（`hooks/hooks.json:9`）。

### 2) 外部交互
- 本目录本身无网络、无 shell 执行、无文件 IO 副作用；纯元数据声明。
- 间接外部效应来自下游 SessionStart 注入：增加模型上下文与 token 消耗（README 已警告，`plugins/explanatory-output-style/README.md:6-7`）。

### 3) 配置/文档耦合
- manifest 与 marketplace 在 `name/version/description/author` 形成双源元数据（`plugin.json:2-8` vs `marketplace.json:51-57`）。
- README 承诺“复刻 Explanatory 风格”（`README.md:3-4`）与 Hook 文本实际内容（`session-start.sh:10`）需要长期一致。

## 风险、边界与改进建议

### 风险
1. 单点失效风险：本目录仅一个 `plugin.json`，JSON 损坏会导致整插件无法发现。  
2. 双源漂移风险：`plugin.json` 与 `marketplace.json` 重复维护同类字段，易出现版本/描述不一致。  
3. 校验盲区：现有 `validate-hook-schema.sh` 对插件 wrapper 格式兼容不足，可能让“脚本失败”掩盖真实配置质量。  
4. 可观测性不足：目录无专门自动化测试覆盖，更多依赖人工回归或运行时暴露。

### 边界
1. `.claude-plugin` 目录只定义元数据，不承载业务逻辑或安全策略。  
2. 它不能控制 Hook 的具体文本质量、长度或 token 成本，只决定插件是否可加载。  
3. 目录内无 `hooks`/`commands` 配置字段，路径策略完全走默认发现约定。

### 改进建议
1. 为 `plugins/*/.claude-plugin/plugin.json` 增加 CI schema/lint（至少校验 `name`、`version`、`author`、kebab-case 规则）。
2. 增加 marketplace 与 manifest 一致性校验（`name/version/author/description` 自动比对）。
3. 修复 `validate-hook-schema.sh`：兼容插件 `{"hooks": {...}}` 格式，先解析根节点再校验事件。  
4. 在 `.claude-plugin` 目录研究流程中增加最小回归脚本：读取 manifest + 校验 `source` 可达 + 检查同目录插件根是否有 README/hook 文件，降低发布回归概率。
