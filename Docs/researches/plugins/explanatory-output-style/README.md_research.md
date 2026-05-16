# FILE `plugins/explanatory-output-style/README.md` 研究文档

## 场景与职责

`plugins/explanatory-output-style/README.md` 是该插件的“产品契约文档”，职责不是实现逻辑，而是把插件的使用场景、行为边界和迁移路径讲清楚，给用户与维护者提供统一预期。

它所在的场景是：Claude Code 早期曾提供内建 output style（含 Explanatory），后续逐步弃用并迁移到插件/系统提示机制；该 README 对应的插件用于复刻被弃用的 Explanatory 风格。

- 历史背景：
  - `CHANGELOG.md:1837` 发布 Explanatory/Learning output styles。
  - `CHANGELOG.md:1528` 弃用 output styles，建议迁移到插件等机制。
  - `CHANGELOG.md:206` 再次明确 `/output-style` 弃用，且 style 固定在会话开始阶段（SessionStart）。
- 当前插件定位：
  - README 明确“recreates the deprecated Explanatory output style as a SessionStart hook”：`plugins/explanatory-output-style/README.md:3-4`。
  - 插件列表也把它定义为 SessionStart Hook：`plugins/README.md:19`。

从工程职责看，README 承担 4 个关键职责：

1. 定义行为目标（解释型输出、教育性 insight）。
2. 定义运行机制（SessionStart 注入 additional context）。
3. 定义迁移关系（旧 `"outputStyle": "Explanatory"` -> 安装插件）。
4. 定义边界与替代路径（token 成本警告；非软件开发任务更适合 subagents）。

## 功能点目的

README 中每个功能块对应明确目的：

1. `WARNING`（`plugins/explanatory-output-style/README.md:6-7`）
- 目的：提前暴露成本。该插件通过额外提示词驱动行为，必然增加上下文 token 与输出长度，避免用户“无感安装后成本上升”。

2. `What it does`（`.../README.md:9-17`）
- 目的：定义期望行为，不涉及实现细节。
- 三个目标：解释实现选择、解释代码库模式、平衡交付与教学。

3. `How it works`（`.../README.md:18-28`）
- 目的：把“风格”映射为可执行机制（SessionStart + additionalContext）。
- 规范化 insight 输出格式，降低回答风格漂移。

4. `Usage`（`.../README.md:30-40`）
- 目的：说明“零配置启用”与 insight 关注点（代码库特异性，而非泛化知识）。

5. `Migration from Output Styles`（`.../README.md:42-62`）
- 目的：提供向后兼容心智模型。
- 给出旧配置示例与替代方案，并说明 SessionStart hook 与 CLAUDE.md 的关系。
- 同时给出“何时不该用 SessionStart hook”的边界：跨任务类型更适合 subagents。

6. `Managing changes`（`.../README.md:64-72`）
- 目的：给运维动作（禁用/卸载/本地定制）而不是开发动作。

## 具体技术实现（关键流程/数据结构/协议/命令）

README 本身是文档层，但其声明可映射到稳定实现链路：

1. 插件被发现与注册
- marketplace 注册项：`.claude-plugin/marketplace.json:51-60`。
- 插件元数据：`plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9`。

2. Hook 配置加载
- 文件：`plugins/explanatory-output-style/hooks/hooks.json:1-15`。
- 结构：顶层 `description` + `hooks` 包装；事件为 `SessionStart`；hook 类型为 `command`。
- 命令：`${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`（可移植路径，不依赖绝对路径）。

3. SessionStart 触发并调用处理器
- 处理器：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`。
- 实现方式：`cat << 'EOF'` 输出静态 JSON，`exit 0` 成功返回。

4. 协议层输出
- 核心数据结构：
  - `hookSpecificOutput.hookEventName = "SessionStart"`
  - `hookSpecificOutput.additionalContext = <长文本指令>`
- 对应实现：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-11`。

5. 会话行为生效
- Claude 在会话启动时获得 additionalContext，进而在“写代码前后”给出 insight 块（README 定义格式与目标）。

### 数据与协议要点

1. Hook 输入协议
- Hook 系统会通过 stdin 传入标准 JSON（`session_id/cwd/hook_event_name` 等）。参考：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-312`。
- 本插件 handler 当前不消费 stdin（纯静态输出），因此与输入字段解耦。

2. Hook 输出协议
- command hook 使用 stdout 输出 JSON + 退出码驱动处理。
- 退出码语义参考：`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`。
- 本插件返回 0（成功）并输出 `hookSpecificOutput.additionalContext`。

3. 运行时环境变量
- `hooks.json` 使用 `${CLAUDE_PLUGIN_ROOT}` 解析脚本路径：`plugins/explanatory-output-style/hooks/hooks.json:9`。
- 该模式与 hook 开发规范一致：`plugins/plugin-dev/skills/hook-development/SKILL.md:331-338`。

### 已执行命令与观察（实测）

1. Handler 冒烟测试（通过）
- 命令：
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/explanatory-sessionstart-input.json`
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh /tmp/explanatory-sessionstart-input.json`
- 结果：退出码 0，输出 JSON 可被 `jq` 解析，`hookEventName=SessionStart`。

2. 指令体量测量
- 命令：`plugins/explanatory-output-style/hooks-handlers/session-start.sh | jq -r '.hookSpecificOutput.additionalContext'`
- 结果：约 `1193` 字符，`140` 词（仅该附加上下文，不含模型回复增量）。

3. Lint 检查
- 命令：`plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh`
- 结果：通过，但有 1 条 warning：缺少 `set -euo pipefail`（脚本极简，风险可控）。

4. Schema 校验脚本兼容性
- 命令：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/explanatory-output-style/hooks/hooks.json`
- 结果：脚本报 `jq` 索引错误并退出非 0。
- 直接原因：该校验脚本按“顶层事件键”迭代，而本插件 `hooks.json` 采用 `{"description","hooks":{...}}` 包装格式（与仓库内官方插件普遍格式一致）。

## 关键代码路径与文件引用

与目标 README 强相关的主路径如下：

1. 目标文档与直接实现
- `plugins/explanatory-output-style/README.md`
- `plugins/explanatory-output-style/hooks/hooks.json`
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh`
- `plugins/explanatory-output-style/.claude-plugin/plugin.json`

2. 调用方 / 注册与发现链路
- `.claude-plugin/marketplace.json:51-60`（marketplace 注册）
- `plugins/README.md:19`（插件目录总览）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`（自动发现机制）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`（hooks 默认路径 `./hooks/hooks.json`）

3. 协议/开发规范与测试脚本
- `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`（plugin hooks.json 包装格式）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:278-312`（输出/输入协议）
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`

4. 旁路对照（被复用/并行能力）
- `plugins/learning-output-style/README.md:3-5,78-82`（声明包含 explanatory 功能）
- `plugins/learning-output-style/hooks-handlers/session-start.sh`（同构的 SessionStart 注入方式）

5. 历史语义来源
- `CHANGELOG.md:1837`（output styles 发布）
- `CHANGELOG.md:1528`（output styles 弃用）
- `CHANGELOG.md:206`（`/output-style` 弃用 + 固定 SessionStart）
- `CHANGELOG.md:192`（修复 resume 时 SessionStart 重复触发）
- `CHANGELOG.md:597`（SessionStart 执行延后优化启动性能）

## 依赖与外部交互

## 内部依赖

1. Claude Code Hook 运行时
- 依赖 `SessionStart` 事件调度能力与 command hook JSON 输出解析能力。

2. 插件发现机制
- 依赖 `.claude-plugin/plugin.json` 与默认 `hooks/hooks.json` 自动发现路径。

3. Bash 运行环境
- handler 依赖 `#!/usr/bin/env bash`。

4. 路径变量展开
- 依赖 `${CLAUDE_PLUGIN_ROOT}` 正确注入。

## 外部交互

1. 文档链接
- README 引用外部文档：
  - 插件文档 `https://docs.claude.com/en/docs/claude-code/plugins.md`
  - subagents 文档 `https://docs.claude.com/en/docs/claude-code/sub-agents`

2. 无网络/文件副作用
- `session-start.sh` 不访问网络、不读写工程文件、不写 `$CLAUDE_ENV_FILE`。
- 外部效应主要是“模型上下文与回答风格变化”。

3. 与用户配置共存
- hook 开发文档说明插件 hooks 会与用户 hooks 合并执行：`plugins/plugin-dev/skills/hook-development/SKILL.md:383`。
- 因此在多插件并存下，最终行为受组合效应影响。

## 风险、边界与改进建议

1. 风险：上下文成本持续增加
- 证据：README 明确 warning（`.../README.md:6-7`）；实测 additionalContext 约 1193 字符。
- 影响：每次新会话都注入，长期提升 token 成本。
- 建议：提供“精简版 explanatory”配置（例如短模板/长模板两档）。

2. 风险：文案硬编码 + 单语言
- 现状：核心指令在 shell 脚本中硬编码英文长字符串。
- 影响：难以版本化复用，国际化困难，和其他插件文案容易漂移。
- 建议：把 prompt 模板抽到独立文本资源（如 `prompts/explanatory.md`），脚本只负责包装输出。

3. 风险：与 `learning-output-style` 功能重复导致行为漂移
- 现状：learning 插件声明“包含 explanatory 全功能”。
- 影响：后续更新若只改一处，会出现两插件体验不一致。
- 建议：抽取共享 insight 模板，两个插件复用同一来源。

4. 风险：校验工具链与实际格式不一致
- 现状：`validate-hook-schema.sh` 对本插件 `hooks/hooks.json` 报错；其逻辑按顶层事件遍历，并强制 `matcher`，与当前官方插件格式（wrapper + SessionStart 常无 matcher）不一致。
- 影响：开发者可能误判“配置错误”，降低工具可信度。
- 建议：升级校验脚本，兼容 plugin wrapper 格式并按事件类型判断 matcher 是否必需。

5. 边界：该插件只影响“回答风格”，不提供任务能力
- 现状：无 commands/agents/MCP，无业务逻辑执行；只做上下文注入。
- 影响：不能替代真实工作流自动化插件。
- 建议：在 README 增加“适用/不适用场景”对照表（如教学引导 vs 批量自动化）。

6. 边界：SessionStart 时机固定，非按任务动态切换
- 现状：行为在会话开始注入，非逐工具调用控制。
- 影响：会话中途无法细粒度按任务切换风格。
- 建议：提供与 `/config` 或局部 prompt 叠加的推荐实践，指导用户在单会话中做风格切换。

7. 轻量工程改进
- 当前脚本虽简单，但 linter 仍提示可改进项（`set -euo pipefail`）。
- 建议：统一为所有 hook handler 加基础安全模板，保持仓库风格一致性。
