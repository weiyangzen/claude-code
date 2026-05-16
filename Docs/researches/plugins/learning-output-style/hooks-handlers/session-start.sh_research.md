# FILE `plugins/learning-output-style/hooks-handlers/session-start.sh` 研究文档

## 场景与职责

`plugins/learning-output-style/hooks-handlers/session-start.sh` 是 `learning-output-style` 插件在运行时的核心执行脚本，属于 **SessionStart 阶段的上下文注入器**，不是业务逻辑处理器。

它所处的执行场景：
- 插件目标是复刻未发布的 Learning output style，并叠加 explanatory 能力：`plugins/learning-output-style/README.md:3-5`
- 插件被注册为 SessionStart Hook 类型：`plugins/README.md:23`
- Hook 路由配置在 `hooks/hooks.json`，把 `SessionStart` 事件绑定到本脚本：`plugins/learning-output-style/hooks/hooks.json:4-10`

它的直接职责：
- 输出 Hook 协议 JSON 到 stdout，携带：
  - `hookSpecificOutput.hookEventName = "SessionStart"`
  - `hookSpecificOutput.additionalContext = <学习+讲解指令文本>`
  见：`plugins/learning-output-style/hooks-handlers/session-start.sh:8-11`
- 以 `exit 0` 返回成功，不阻断会话启动：`plugins/learning-output-style/hooks-handlers/session-start.sh:15`

## 功能点目的

该脚本承载 6 个核心目的：

1. 迁移旧 output style 能力到插件机制
- `CHANGELOG.md:206` 指出 `/output-style` 弃用且输出风格固定在会话开始阶段。
- 本插件用 SessionStart 注入实现“风格固定于会话开始”的替代路径：`plugins/learning-output-style/README.md:76-83`

2. 在会话开始一次性设定“学习式协作”行为
- 指令要求模型主动识别可由用户补充的关键实现（5-10 行），而非全自动实现：`plugins/learning-output-style/hooks-handlers/session-start.sh:10`

3. 明确“请求用户贡献”的触发边界
- 请求贡献：业务逻辑、错误处理、算法/数据结构、架构/设计权衡。
- 不请求贡献：样板代码、显而易见实现、配置、简单 CRUD。
- 这些策略均内嵌于 `additionalContext`：`plugins/learning-output-style/hooks-handlers/session-start.sh:10`

4. 约束请求方式，保证用户贡献是“高价值输入”
- 先准备文件与函数签名、标出 TODO，再提出聚焦请求并解释 trade-off：`plugins/learning-output-style/hooks-handlers/session-start.sh:10`

5. 叠加 explanatory 模式，提升可学习性
- 要求代码前后提供 `★ Insight` 结构化洞察块：`plugins/learning-output-style/hooks-handlers/session-start.sh:10`
- 该能力与 explanatory 插件同构：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:10`

6. 保持实现最小化、跨项目可移植
- 单文件 Bash heredoc 实现，无语言运行时依赖，无外部 API：`plugins/learning-output-style/hooks-handlers/session-start.sh:1-13`

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程（调用方 -> 目标脚本 -> 运行时消费）

1. 插件元数据被加载  
- `learning-output-style` 清单定义在 `.claude-plugin/plugin.json`：`plugins/learning-output-style/.claude-plugin/plugin.json:1-9`

2. Hook 配置被自动发现并注册  
- 插件结构规范说明：默认从 `hooks/hooks.json` 加载 Hook：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`  
- manifest 参考也定义默认 hooks 路径为 `./hooks/hooks.json`：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`

3. SessionStart 触发并执行 command hook  
- 路由配置：`${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`：`plugins/learning-output-style/hooks/hooks.json:9`

4. 本脚本输出 JSON 协议载荷  
- 通过 `cat << 'EOF'` 原样输出：`plugins/learning-output-style/hooks-handlers/session-start.sh:6-13`

5. 运行时消费 `additionalContext` 并并入会话上下文  
- SessionStart 用于“会话启动时加载上下文/设置环境”：`plugins/plugin-dev/skills/hook-development/SKILL.md:238-264`

### 2) 关键数据结构

脚本输出（简化）：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "..."
  }
}
```

字段语义：
- `hookSpecificOutput`：Hook 特定输出容器。
- `hookEventName`：声明输出所对应的事件类型。
- `additionalContext`：长文本行为约束，决定后续会话中的协作/讲解风格。

### 3) 协议与退出码

- 命令 Hook 输入通常为 stdin JSON（包含 `session_id/cwd/hook_event_name` 等）：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:83-92`
- 本脚本不消费 stdin，属于静态注入型 handler。
- 退出码约定：`0` 成功，`2` 常用于阻断（针对决策型 Hook）：`plugins/plugin-dev/skills/hook-development/SKILL.md:176-180`
- 本脚本固定 `exit 0`：`plugins/learning-output-style/hooks-handlers/session-start.sh:15`

### 4) 关键命令与本次实测结果

1. 生成 SessionStart 样例输入：

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/learning_sessionstart_input.json
jq -r '.hook_event_name' /tmp/learning_sessionstart_input.json
```

结果：`SessionStart`。

2. 使用官方测试脚本执行目标文件：

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh \
  plugins/learning-output-style/hooks-handlers/session-start.sh \
  /tmp/learning_sessionstart_input.json
```

结果：Exit Code `0`，输出 JSON 可解析，包含 `hookSpecificOutput.additionalContext`。

3. 直接验证协议字段：

```bash
plugins/learning-output-style/hooks-handlers/session-start.sh | jq -r '.hookSpecificOutput.hookEventName'
plugins/learning-output-style/hooks-handlers/session-start.sh | jq -r '.hookSpecificOutput.additionalContext | length'
```

结果：`hookEventName = SessionStart`；`additionalContext` 长度为 `3034`（字符）。

4. Lint 目标脚本：

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh \
  plugins/learning-output-style/hooks-handlers/session-start.sh
```

结果：通过但有 warning（缺少 `set -euo pipefail`、stderr 错误输出规范提示）。

5. 校验 hooks 配置：

```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh \
  plugins/learning-output-style/hooks/hooks.json
```

结果：`validate-hook-schema.sh` 会把 wrapper 顶层键 `description/hooks` 当事件遍历，随后触发 jq 索引错误并退出（工具与当前插件格式存在错配）。

## 关键代码路径与文件引用

主链路（最关键）：
- `plugins/learning-output-style/hooks/hooks.json:4-10`（SessionStart 路由到脚本）
- `plugins/learning-output-style/hooks-handlers/session-start.sh:6-15`（输出与退出）

插件与分发上下文：
- `plugins/learning-output-style/.claude-plugin/plugin.json:2-4`
- `.claude-plugin/marketplace.json:95-103`
- `plugins/README.md:23`

语义说明文档：
- `plugins/learning-output-style/README.md:3-5`
- `plugins/learning-output-style/README.md:24-27`
- `plugins/learning-output-style/README.md:72-83`

协议/工具/测试上下文：
- `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`（plugin hooks wrapper 格式）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:238-264`（SessionStart 语义）
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100,170-203`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:49-59`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-73`

历史行为变更（运行时边界）：
- `CHANGELOG.md:1908`（1.0.62 引入 SessionStart Hook）
- `CHANGELOG.md:597`（2.1.47 延后 SessionStart 执行以提速）
- `CHANGELOG.md:192`（2.1.73 修复 resume 时 SessionStart 重复触发）
- `CHANGELOG.md:1079`（2.1.2 SessionStart 输入增加 `agent_type`）

## 依赖与外部交互

### 运行时依赖
- `bash` 解释器：脚本 shebang 为 `#!/usr/bin/env bash`。
- Claude Code Hook runtime：负责触发 SessionStart、执行 command hook、消费 JSON 输出。
- `${CLAUDE_PLUGIN_ROOT}`：由运行时提供，用于可移植命令路径展开（定义在 `hooks.json`）。

### 配置依赖
- `.claude-plugin/plugin.json`：插件身份与元信息。
- `hooks/hooks.json`：事件到脚本的路由。
- `README.md`：行为约束与使用边界的“人类可读合同”。

### 开发/验证依赖
- `jq`：测试脚本与人工验证解析 JSON。
- `timeout`：`test-hook.sh` 中做超时保护。
- `hook-linter.sh`、`validate-hook-schema.sh`：静态校验与规范检查工具。

### 外部交互特征
- 不访问网络。
- 不读写项目业务文件。
- 不写入 `$CLAUDE_ENV_FILE`（与 `load-context.sh` 这类 SessionStart 脚本不同）。
- 唯一输出通道是 stdout JSON；唯一控制信号是退出码。

## 风险、边界与改进建议

### 风险

1. Token 成本固定增加  
- 每次 SessionStart 注入 3k+ 字符指令文本；README 已明确 token 成本警告：`plugins/learning-output-style/README.md:7`

2. 行为“全局且静态”  
- 不读取 hook 输入字段，无法根据 `agent_type/cwd/任务类型` 动态分流；启用即全会话生效。

3. 文案硬编码维护成本高  
- `additionalContext` 是单一超长字符串，review 与 diff 可读性较差；与 explanatory 插件存在共性段落，长期易漂移。

4. 质量门禁脚本错配  
- `validate-hook-schema.sh` 当前与 plugin wrapper 格式不完全兼容，容易产生误报/假失败。

### 边界

1. 该脚本仅负责“上下文注入”，不做权限决策（无 `permissionDecision`）。
2. 不处理异常输入，也不做项目探测和状态持久化。
3. 测试保障主要依赖通用工具脚本，当前插件目录无独立自动化测试文件。

### 改进建议

1. 新增文件级 smoke test（优先）
- 断言输出 JSON 可解析、`hookEventName == SessionStart`、`additionalContext` 非空且长度在预期范围。

2. 对 `additionalContext` 做模块化维护
- 将“Learning 段”和“Explanatory 段”拆成模板片段再拼装，降低文案漂移与维护成本。

3. 增加轻量模式（可选）
- 提供精简版指令文本，允许用户在“教学强度”与“token 成本”间选择。

4. 加强脚本健壮性一致性
- 补 `set -euo pipefail`，并按统一规范处理潜在异常输出（即使当前逻辑较简单）。

5. 修复 schema 校验工具对 wrapper 格式支持
- 在 `validate-hook-schema.sh` 中先下钻 `.hooks` 再做事件遍历，兼容官方插件常见结构。
