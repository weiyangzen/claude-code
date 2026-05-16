# plugins/learning-output-style/.claude-plugin 目录研究（DIR）

## 场景与职责

`plugins/learning-output-style/.claude-plugin` 是 `learning-output-style` 插件的 manifest 目录，当前仅包含 `plugin.json`（`plugins/learning-output-style/.claude-plugin/plugin.json:1-9`）。

该目录不直接执行 Hook、命令或脚本，它在插件体系中的职责是“发现入口 + 身份声明”：

1. 提供 Claude Code 发现插件所需的必需 manifest 位置。  
   规范要求 manifest 必须在 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`，`plugins/README.md:49-55`）。
2. 声明插件身份元数据（`name/version/description/author`），用于识别、展示与归属（`plugins/learning-output-style/.claude-plugin/plugin.json:2-8`）。
3. 作为下游组件发现的前置条件。发现生命周期是先读取 manifest，再发现/注册 hooks 等组件（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。

在仓库分发上下文里，该目录还与 marketplace 条目形成路由关系：`.claude-plugin/marketplace.json` 中 `source: "./plugins/learning-output-style"` 把插件根目录接入加载链路（`.claude-plugin/marketplace.json:95-103`）。

## 功能点目的

### 1) 插件唯一标识与可发现性
- `name: "learning-output-style"` 是插件主键（`plugins/learning-output-style/.claude-plugin/plugin.json:2`）。
- 按规范，`name` 需 kebab-case，且用于插件识别与冲突检测（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-31`）。

### 2) 元数据承载（版本、说明、作者）
该目录承载发布最小元数据：
- `version: "1.0.0"`（`plugins/learning-output-style/.claude-plugin/plugin.json:3`）
- `description`（`plugins/learning-output-style/.claude-plugin/plugin.json:4`）
- `author.name/email`（`plugins/learning-output-style/.claude-plugin/plugin.json:5-7`）

同时，marketplace 条目保留同构字段（`.claude-plugin/marketplace.json:95-103`），共同影响安装展示与检索。

### 3) 下游 Hook 组件启用前置
manifest 未声明自定义 `hooks` 路径，意味着依赖默认目录发现规则（`plugins/README.md:51-60`，`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:12-15`）：
- 配置入口：`plugins/learning-output-style/hooks/hooks.json:1-15`
- 执行脚本：`plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`

### 4) Output Style 迁移身份锚点
该插件定位为“复刻/迁移 Learning 风格并融合 explanatory 能力”（`plugins/learning-output-style/README.md:3-5,76-82`），而 `.claude-plugin/plugin.json` 是其可安装实体标识。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 目标目录 -> 被调用方）

1. 调用方（上游）
- Marketplace 注册插件并指向 source：`.claude-plugin/marketplace.json:95-103`。
- Claude Code 初始化时先读每个插件的 `.claude-plugin/plugin.json`，再发现组件并注册 Hook：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`。

2. 目标目录（本研究对象）
- 读取 `plugins/learning-output-style/.claude-plugin/plugin.json:1-9`，完成插件身份识别。

3. 被调用方（下游）
- 发现 `hooks/hooks.json` 并注册 `SessionStart` 事件：`plugins/learning-output-style/hooks/hooks.json:3-14`。
- 事件触发后执行 `${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`：`plugins/learning-output-style/hooks/hooks.json:8-10`。
- 脚本输出 `hookSpecificOutput.additionalContext` 注入学习/讲解模式：`plugins/learning-output-style/hooks-handlers/session-start.sh:8-12`。

简化链路：
`marketplace source` -> `plugin root` -> `.claude-plugin/plugin.json` -> `hooks/hooks.json` -> `SessionStart command hook` -> `additionalContext` 注入。

### B. 数据结构（manifest）

本目录核心数据结构为最小 manifest：

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

字段语义与规范来源：
- `name`（必需、kebab-case）：`manifest-reference.md:15-40`
- `version`（语义化版本）：`manifest-reference.md:42-63`
- `description`（用途描述）：`manifest-reference.md:65-83`
- `author`（归属与联络）：`manifest-reference.md:87-114`

### C. 协议与命令（与本目录相关）

本目录不直接执行命令，但决定下游 Hook 协议是否可被加载：

1. Hook 配置协议  
- 插件 hooks 使用 wrapper 格式 `{"description": "...", "hooks": {...}}`（`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`）。  
- `learning-output-style` 的 `hooks/hooks.json` 符合该格式（`plugins/learning-output-style/hooks/hooks.json:1-15`）。

2. SessionStart 协议  
- SessionStart 事件用于会话起始注入上下文（`plugins/plugin-dev/skills/hook-development/SKILL.md:238-257`）。  
- 命令路径使用 `${CLAUDE_PLUGIN_ROOT}`，确保插件可移植（`plugins/learning-output-style/hooks/hooks.json:9`，`plugins/plugin-dev/skills/hook-development/SKILL.md:326-338`）。

3. Hook 输出约定  
- Hook 以 JSON 输出，`exit 0` 视为成功（`plugins/plugin-dev/skills/hook-development/SKILL.md:278-299`）。  
- 本插件脚本固定输出 `hookSpecificOutput.hookEventName + additionalContext`，并 `exit 0`（`plugins/learning-output-style/hooks-handlers/session-start.sh:8-15`）。

### D. 脚本/测试验证（实测）

1. `test-hook.sh` 验证通过  
命令（实测）：  
`bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/learning-output-style/hooks-handlers/session-start.sh <SessionStart样例JSON>`  
结果：`exit code 0`，输出可解析 JSON，且包含 `hookSpecificOutput.additionalContext`。

2. `hook-linter.sh` 通过但有 warning  
命令（实测）：  
`bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh plugins/learning-output-style/hooks-handlers/session-start.sh`  
结果：脚本整体通过；提示缺少 `set -euo pipefail`、stderr 规范建议等（linter 规则见 `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:49-53,106-111`）。

3. `validate-hook-schema.sh` 对当前插件 hooks 格式不兼容  
命令（实测）：  
`bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/learning-output-style/hooks/hooks.json`  
结果：`jq: Cannot index string with number`（退出码 5）。  
原因：该脚本按顶层事件键遍历（`validate-hook-schema.sh:43,65-67`），未先进入插件 wrapper 的 `.hooks` 节点；而插件格式正是 wrapper（`hook-development/SKILL.md:62-80`）。

## 关键代码路径与文件引用

### 目标目录（直接对象）
- `plugins/learning-output-style/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:95-103`（marketplace 注册与 source）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 必需路径）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`（发现/注册顺序）
- `plugins/README.md:49-60`（插件标准结构）

### 被调用方（下游）
- `plugins/learning-output-style/hooks/hooks.json:1-15`（SessionStart Hook 配置）
- `plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`（Hook 处理器）
- `plugins/learning-output-style/README.md:24-27,72-82`（工作机制与迁移语义）

### 配置、测试、脚本、文档上下文
- 配置：manifest + marketplace + hooks.json（`plugin.json`、`marketplace.json`、`hooks/hooks.json`）。
- 测试：插件目录无专属 `test/spec` 文件；采用 `plugin-dev` 的通用 hook 测试脚本做行为验证（`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:7-252`）。
- 脚本：插件自有执行脚本仅 `hooks-handlers/session-start.sh`；辅助校验来自 `hook-linter.sh` 与 `validate-hook-schema.sh`。
- 文档：`plugins/learning-output-style/README.md` + `plugins/README.md` + `plugin-dev` 参考文档。

## 依赖与外部交互

### 1) 仓库内依赖
- 依赖 manifest 路径正确，否则插件不可发现（`manifest-reference.md:7-10`）。
- 依赖 marketplace `source` 路由到正确插件目录（`.claude-plugin/marketplace.json:102`）。
- 依赖默认 hooks 自动发现机制与 SessionStart 事件调度（`component-patterns.md:11-16,26`）。
- 依赖 `${CLAUDE_PLUGIN_ROOT}` 进行跨环境可移植路径解析（`hooks/hooks.json:9`）。

### 2) 外部交互
- 本目录自身无网络调用、无文件写入、无命令执行副作用。
- 间接交互来自下游 SessionStart 注入：增加系统上下文文本，影响 token 成本与对话行为（README 警告见 `plugins/learning-output-style/README.md:7`）。

### 3) 历史演进依赖
- Learning/Explanatory output style 最初作为内建 output styles 发布（`CHANGELOG.md:1837`）。
- 后续 output style 被弃用并建议迁移到插件/系统提示（`CHANGELOG.md:1528`）。
- 近期明确 output style 固定在会话开始阶段（`CHANGELOG.md:206`），与本插件 SessionStart 注入模型一致。

## 风险、边界与改进建议

### 风险

1. 单点失效风险  
本目录只有 `plugin.json`，一旦 JSON 损坏或字段非法会导致插件无法被识别。

2. 元数据双源漂移  
`plugin.json` 与 `marketplace.json` 同时维护 `name/version/description/author`，存在同步漂移风险（当前该插件两处内容一致：`plugin.json:2-8` vs `marketplace.json:95-101`）。

3. 校验工具与真实格式不一致  
`validate-hook-schema.sh` 目前不兼容插件 wrapper 格式，可能误报/中断，影响 hooks 质量门禁稳定性（`validate-hook-schema.sh:43,65-67` 对比 `hook-development/SKILL.md:62-80`）。

4. 指令负载与行为偏移风险  
SessionStart 注入长 `additionalContext`（`session-start.sh:10`）会增加 token 成本，且可能改变模型交互风格；README 已显式警告（`README.md:7`）。

### 边界

1. `.claude-plugin` 目录只负责 manifest 声明，不承载业务逻辑或权限决策。  
2. 它无法约束下游 `additionalContext` 文案质量，只决定插件“是否可加载”。  
3. 目录内无专属测试或脚本体系，运行质量主要依赖外部通用校验工具和运行时行为。

### 改进建议

1. 增加 manifest CI 校验  
对 `plugins/*/.claude-plugin/plugin.json` 执行 schema + `name/version` 规则校验，尽早阻断不可发现故障。

2. 增加 marketplace 一致性校验  
自动比对 `source` 指向目录下 manifest 与 marketplace 的关键字段（`name/version/author/description`）。

3. 修复 `validate-hook-schema.sh` 对插件 wrapper 的支持  
优先解析 `.hooks` 节点再校验事件，避免当前 `jq` 索引错误。

4. 为学习风格插件补充轻量 smoke test  
在 CI 中加入：`test-hook.sh + jq parse`，确认 `SessionStart` 输出 JSON 结构稳定，降低发布回归风险。
