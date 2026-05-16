# FILE `plugins/learning-output-style/README.md` 研究文档

## 场景与职责

`plugins/learning-output-style/README.md` 是该插件的产品行为说明文档，职责不是执行 Hook，而是定义“插件应该如何改变 Claude 的会话行为”。

它所在的业务场景是 output style 从内建配置迁移到插件机制后的替代方案：
- 历史背景：`CHANGELOG.md:1837`（发布 Learning/Explanatory output styles）、`CHANGELOG.md:1528`（弃用 output styles，建议迁移到插件/系统提示）、`CHANGELOG.md:206`（`/output-style` 弃用且 style 在 SessionStart 固定）。
- 插件定位：`plugins/README.md:23` 将其定义为“交互式学习模式 + 教育性 insight”的 SessionStart Hook 插件。

README 在该插件中的核心职责：
1. 定义目标行为：学习式协作（用户写关键 5-10 行）+ explanatory insight（实现前后教学解释）。
2. 定义运行时边界：通过 `SessionStart` 注入上下文，而非逐工具动态控制。
3. 定义启用与迁移路径：零配置启用、兼容旧 output style 心智模型。
4. 定义成本预期：明确 token 成本与交互成本会增加。

## 功能点目的

围绕 README 的段落结构，可以映射为以下功能目的：

1. 插件定位与差异声明（`plugins/learning-output-style/README.md:3-5`）
- 目的：明确它不是“原始 Learning 的 1:1 复制”，而是“Learning + Explanatory”融合版。
- 价值：避免用户误以为只会收到“请求写代码”，实际上还会附带教学洞察。

2. 成本预警（`plugins/learning-output-style/README.md:7`）
- 目的：把 token 与交互开销前置说明，降低误装后的体验落差。

3. 行为目标定义（`plugins/learning-output-style/README.md:11-23`）
- 目的：要求模型在“可决策点”引导用户参与，而不是全自动交付。
- 包含两层能力：
  - Learning Mode：请求用户贡献有意义代码。
  - Explanatory Mode：解释实现与代码库模式。

4. 请求贡献与直接实现的边界（`plugins/learning-output-style/README.md:28-44`）
- 目的：避免互动学习退化为“把简单体力活甩给用户”。
- 策略：仅在有 trade-off、多解、领域判断时请求用户贡献。

5. 交互模板与 insight 格式（`plugins/learning-output-style/README.md:46-70`）
- 目的：标准化对话风格，降低会话间漂移；强调 insight 应聚焦当前代码库而非泛化编程常识。

6. 运行机制与迁移说明（`plugins/learning-output-style/README.md:24-27,76-82`）
- 目的：把“风格”落地为可执行机制（SessionStart Hook），并给旧 output style 用户明确迁移路径。

7. 运维动作与插件生命周期（`plugins/learning-output-style/README.md:84-89`）
- 目的：区分 disable / uninstall / 本地定制三种操作，降低维护门槛。

## 具体技术实现（关键流程/数据结构/协议/命令）

README 本身是文档层，但它声明的行为对应到稳定实现链路如下。

### 1) 关键流程（调用方 -> 被调用方）

1. 插件被发现
- 入口元数据：`plugins/learning-output-style/.claude-plugin/plugin.json:1-9`。
- 自动发现机制依据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`。

2. Hook 配置被加载
- 默认 hooks 路径：`./hooks/hooks.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`）。
- 本插件配置：`plugins/learning-output-style/hooks/hooks.json:1-15`。

3. SessionStart 事件触发
- 事件绑定：`SessionStart -> type=command -> ${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`（`plugins/learning-output-style/hooks/hooks.json:4-10`）。

4. Hook 处理器输出上下文
- 处理器：`plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`。
- 输出协议：`hookSpecificOutput.hookEventName="SessionStart"` + `hookSpecificOutput.additionalContext=<长文本指令>`（`plugins/learning-output-style/hooks-handlers/session-start.sh:8-10`）。

5. 会话行为生效
- Claude 在会话起始获得 `additionalContext`，后续执行 README 描述的“互动学习 + 教学 insight”行为。

### 2) 数据结构与协议

1. Hook 配置（插件包装格式）
- 本插件使用 wrapper：
  - 顶层 `description`
  - 顶层 `hooks`
  - 事件键 `SessionStart`
- 见：`plugins/learning-output-style/hooks/hooks.json:1-15`。
- 该格式与 hook-development 规范一致：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`。

2. Hook 输入协议
- Hook 运行时通过 stdin 注入 JSON（包含 `session_id/cwd/hook_event_name` 等通用字段）。
- 参考：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-312`。
- 现状：`session-start.sh` 未读取 stdin，是“静态输出型 SessionStart hook”。

3. Hook 输出协议
- 输出 JSON 中使用 `hookSpecificOutput.additionalContext` 注入上下文。
- 退出码语义：`0` 成功，`2` 阻断型错误，其他为非阻断异常（`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`）。
- 本插件固定 `exit 0`。

4. 运行时路径协议
- 通过 `${CLAUDE_PLUGIN_ROOT}` 引用脚本，避免安装路径耦合：`plugins/learning-output-style/hooks/hooks.json:9`。
- 该实践与规范建议一致：`plugins/plugin-dev/skills/hook-development/SKILL.md:331-338`。

### 3) 关键命令与实测结果

1. SessionStart 样例输入构造
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/learning-readme-sessionstart-input.json
jq -r '.hook_event_name' /tmp/learning-readme-sessionstart-input.json
```
结果：`SessionStart`。

2. Hook 执行验证
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/learning-output-style/hooks-handlers/session-start.sh /tmp/learning-readme-sessionstart-input.json
```
结果：`Exit Code: 0`，并输出可解析 JSON，包含 `hookSpecificOutput.additionalContext`。

3. 上下文体量测量
```bash
plugins/learning-output-style/hooks-handlers/session-start.sh | jq -r '.hookSpecificOutput.additionalContext' | wc -m
```
结果：`3035` 字符（仅 `additionalContext` 文本，不含模型回答输出）。

4. schema 校验脚本兼容性检查
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/learning-output-style/hooks/hooks.json
```
结果：报 `Unknown event type: description/hooks`，并在逐项校验阶段 `jq` 异常 `Cannot index string with number`，最终退出码 `5`。

5. hook-linter 检查
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh plugins/learning-output-style/hooks-handlers/session-start.sh
```
结果：通过（exit 0），但给出 warning：缺少 `set -euo pipefail`、stderr 输出建议等。

## 关键代码路径与文件引用

与目标 README 强相关的代码路径（含调用方、被调用方、配置、测试脚本、文档）如下。

1. 目标对象
- `plugins/learning-output-style/README.md`

2. 直接实现与配置（被调用方）
- `plugins/learning-output-style/.claude-plugin/plugin.json`
- `plugins/learning-output-style/hooks/hooks.json`
- `plugins/learning-output-style/hooks-handlers/session-start.sh`

3. 调用方与发现链路
- `.claude-plugin/marketplace.json:95-103`（learning-output-style 注册与 source 路径）
- `plugins/README.md:23,47-61`（插件目录职责与标准结构）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`（自动发现顺序）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263,354-361`（hooks 默认路径与解析顺序）

4. 协议与开发文档
- `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`（plugin hooks wrapper 格式）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:238-264`（SessionStart 能力）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:278-312,322-329`（输出/输入协议、环境变量）

5. 测试与校验脚本
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/README.md`

6. 同类插件对照（README 声明“包含 explanatory 功能”）
- `plugins/explanatory-output-style/README.md`
- `plugins/explanatory-output-style/hooks/hooks.json`
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh`

## 依赖与外部交互

### 内部依赖

1. Claude Code 插件运行时
- 依赖 manifest 识别（`.claude-plugin/plugin.json`）与 hooks 自动装配机制。

2. Hook 事件系统
- 依赖 `SessionStart` 事件触发与 command hook 执行。

3. Shell 运行环境
- `session-start.sh` 使用 `#!/usr/bin/env bash`，无 Python/Node 依赖。

4. 路径变量展开
- 依赖 `${CLAUDE_PLUGIN_ROOT}` 注入正确，否则 command hook 无法定位脚本。

### 外部交互

1. 运行时外部副作用
- 无网络调用、无仓库业务文件写入、无 `$CLAUDE_ENV_FILE` 写入。
- 主要外部效应是“注入会话上下文，改变模型交互模式”。

2. 文档外链
- README 提供官方插件文档链接（`https://docs.claude.com/en/docs/claude-code/plugins.md`），属于文档指引而非运行依赖。

3. 与其他插件/用户配置的组合关系
- Hook 文档说明插件 hooks 与用户 hooks 可合并执行：`plugins/plugin-dev/skills/hook-development/SKILL.md:383`。
- 因此最终行为可能受多插件叠加影响。

4. 测试覆盖现状
- 在仓库中未发现 `learning-output-style` 专属自动化测试文件（`*test*` 命名路径下无命中），目前主要依赖 plugin-dev 通用脚本进行手工验证。

## 风险、边界与改进建议

1. 风险：会话固定增加上下文成本
- 证据：README 明确 warning（`plugins/learning-output-style/README.md:7`）；实测 `additionalContext` 为 3035 字符。
- 影响：每次新会话都会注入，长期提高 token 成本。
- 建议：提供“短版/完整版”两档 prompt 模板，给用户按成本选择。

2. 风险：文档与脚本双写，易漂移
- 现状：README 行为描述与 `session-start.sh` 长文本都在独立维护。
- 影响：功能演进时容易出现“文档说 A、运行时是 B”。
- 建议：抽取共享 prompt 片段（如 `prompts/learning-output-style.md`），脚本只做 JSON 包装。

3. 风险：与 explanatory 插件功能重叠导致不一致
- 现状：README 声明已包含 explanatory 全能力（`plugins/learning-output-style/README.md:5,78-80`），但两插件各维护独立 `additionalContext` 文案。
- 影响：后续只更新一侧时，用户体验会分叉。
- 建议：抽取 insight 区块为共享模板，learning 插件在其上追加“用户贡献引导”部分。

4. 风险：校验工具与真实配置格式不一致
- 现状：`validate-hook-schema.sh` 对 wrapper 格式 `hooks/hooks.json` 不兼容（本插件实测退出码 5）。
- 影响：开发者可能误判官方插件配置“非法”。
- 建议：校验脚本先识别并下钻 `.hooks`，同时按事件类型放宽 `matcher` 强制规则。

5. 边界：仅影响交互风格，不直接提供执行能力
- 现状：插件无 commands/agents/MCP，仅通过 SessionStart 附加上下文改变回答策略。
- 影响：不适用于需要确定性自动化执行的场景。
- 建议：README 增加“适用/不适用场景”对照表，降低误用。

6. 边界：SessionStart 决定注入时机，粒度较粗
- 背景：`CHANGELOG.md:206` 指出 output style 固定在 session start。
- 影响：同一会话中难以细粒度切换学习风格强度。
- 建议：补充“在单会话临时降级互动强度”的操作建议（如本地插件副本或会话级附加指令）。
