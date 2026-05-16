# DIR `plugins/explanatory-output-style/hooks` 研究文档

## 场景与职责

`plugins/explanatory-output-style/hooks` 是该插件的 Hook 配置目录，当前仅包含 `hooks.json`。它本身不执行业务逻辑，职责是把 Claude Code 的 `SessionStart` 事件映射到真实可执行处理器（`hooks-handlers/session-start.sh`）。

在整体链路中的位置：
1. 调用方（上游）
- Claude Code 插件加载器在插件启用后读取 hook 配置（默认路径为 `./hooks/hooks.json`）。
- 会话启动阶段触发 `SessionStart` 事件并执行本目录声明的 command hook。

2. 被调用方（下游）
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh`：输出 `hookSpecificOutput.additionalContext`。
- 系统 `bash`：执行上述 shell 脚本。

3. 业务边界
- 本目录只做“事件 -> 命令”声明，不做 JSON 输入解析、不做权限判定、不读写代码文件。
- 与 `PreToolUse`/`Stop` 这类决策型 hook 不同，本目录对应的是“上下文注入型” SessionStart hook。

## 功能点目的

### 1) 把已废弃的 Explanatory output style 迁移到插件机制

- 插件 README 明确该插件用于复刻已废弃的 Explanatory 风格，并通过 SessionStart 注入说明：`plugins/explanatory-output-style/README.md:3-4,20-22,44-56`。
- `hooks.json` 是这个迁移能力的配置入口：没有这里的映射，handler 不会被事件触发。

### 2) 在会话开始时注入教育性上下文

- 当前仅注册 `SessionStart`，说明目标是会话级风格设定，而非每次工具调用动态拦截：`plugins/explanatory-output-style/hooks/hooks.json:4-13`。
- 注入内容由下游脚本输出，强调“任务完成 + 教学解释”并定义 Insight 格式：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-11`。

### 3) 保证插件可移植部署

- 命令路径使用 `${CLAUDE_PLUGIN_ROOT}`，避免硬编码绝对路径，支持插件被复制到不同目录：`plugins/explanatory-output-style/hooks/hooks.json:9`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 插件发现与入口
- marketplace 条目将插件源目录指向 `./plugins/explanatory-output-style`：`.claude-plugin/marketplace.json:51-59`。
- 插件总览中将该插件标记为 SessionStart hook：`plugins/README.md:19`。

2. Hook 配置装载
- 插件 manifest 位于 `.claude-plugin/plugin.json`，未显式覆写 `hooks` 字段时按默认读取 `./hooks/hooks.json`：
  - `plugins/explanatory-output-style/.claude-plugin/plugin.json:1-8`
  - `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`

3. 事件绑定
- `hooks.json` 采用插件格式包装：
  - 顶层 `description`（可选）
  - 顶层 `hooks`（必需）
  - `hooks.SessionStart[0].hooks[0]` 为 `type: command`
  - `command: ${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`
  证据：`plugins/explanatory-output-style/hooks/hooks.json:1-15`。

4. 运行时执行
- SessionStart 触发后，Claude Code 执行上述 command hook，脚本输出 JSON 到 stdout：
  - `hookSpecificOutput.hookEventName = "SessionStart"`
  - `hookSpecificOutput.additionalContext = <长文本说明>`
  证据：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:6-13`。

### B. 数据结构与协议

1. 配置结构（目录内核心）

```json
{
  "description": "Explanatory mode hook that adds educational insights instructions",
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

- 这是“插件 hooks 包装格式”，不是用户 settings 的“事件直写格式”；对照规范：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,102-119`。

2. 输入/输出协议
- Hook 输入：Claude Code 通过 stdin 提供 JSON（包括 `session_id`、`cwd`、`hook_event_name` 等）；参考：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-320`。
- 当前 handler 不消费 stdin，仅静态输出 JSON。
- 标准退出码：`0` 成功、`2` 阻断、其他为错误；参考：`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`。

### C. 本次命令级实测

1. 生成 SessionStart 测试输入
- 命令：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/explanatory-sessionstart-input.json`
- 结果：生成了包含 `hook_event_name: "SessionStart"` 的样例 JSON。

2. 执行目标 handler
- 命令：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh /tmp/explanatory-sessionstart-input.json`
- 结果：`Exit Code: 0`，输出 JSON 可解析，含 `hookSpecificOutput.additionalContext`。

3. 校验 hooks 配置脚本兼容性
- 命令：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/explanatory-output-style/hooks/hooks.json`
- 结果：先提示根键 `description/hooks` 为未知，再在 `jq` 阶段报错 `Cannot index string with number`，最终退出 `5`。
- 结论：`validate-hook-schema.sh` 当前实现按“事件在顶层”遍历，与插件包装格式不兼容（至少在该脚本版本下）。

## 关键代码路径与文件引用

### 目标目录（本次研究对象）
- `plugins/explanatory-output-style/hooks/hooks.json`

### 直接上下游（调用方/被调用方）
- 上游插件入口与发现：
  - `.claude-plugin/marketplace.json:51-59`
  - `plugins/README.md:19,47-60`
- 插件 manifest：
  - `plugins/explanatory-output-style/.claude-plugin/plugin.json:1-8`
- 下游执行脚本：
  - `plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`

### 配置/测试/脚本/文档上下文
- 配置规范：
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:60-80`
  - `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-271`
- 测试脚本：
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100,170-252`
- 校验脚本：
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`
  - `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:49-59,81-90,114-117`
- 业务文档：
  - `plugins/explanatory-output-style/README.md:9-40`
  - `CHANGELOG.md:192,206,597,1908`

### 对照实现（同类 SessionStart 设计）
- `plugins/learning-output-style/hooks/hooks.json:1-15`
- `plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`

## 依赖与外部交互

### 依赖
1. 运行时
- Claude Code Hook 生命周期（SessionStart 触发）。
- `bash` 解释器（执行 command hook）。

2. 环境变量
- `${CLAUDE_PLUGIN_ROOT}`：用于命令路径可移植定位。

3. 工具链（验证阶段）
- `jq`：JSON 语法/结构校验。
- `test-hook.sh` / `validate-hook-schema.sh` / `hook-linter.sh`：来自 `plugin-dev` 的辅助脚本。

### 外部交互
- 目录自身无网络请求、无第三方 API 交互。
- 目录自身不直接读写仓库业务文件。
- 主要外部效应是增加会话提示词长度和 token 成本（README 已给出 warning）：`plugins/explanatory-output-style/README.md:6-7`。

### 测试现状
- 本目录没有专属单元测试或 CI 用例。
- 当前可用的是通用脚本级 smoke test（本次已实测通过 handler 执行）。

## 风险、边界与改进建议

### 风险
1. 校验脚本错配风险
- `validate-hook-schema.sh` 与插件包装格式不兼容，容易造成“配置本身可用但校验失败”的误导。

2. 指令注入成本风险
- 每次 `SessionStart` 注入长文本，固定提高 token 开销，并可能在短任务中造成额外冗长度。

3. 配置可读性风险
- 当前 `SessionStart` 条目无显式 `matcher`（虽然该事件通常全局触发），对新维护者理解成本略高。

4. 文案漂移风险
- `learning-output-style` 含 explanatory 能力的并行文案，后续演进时两边易不同步。

### 边界
1. 本目录不处理工具权限决策
- 无 `permissionDecision`、无 `updatedInput` 修改能力的使用。

2. 本目录不负责动态逻辑
- 无 stdin 字段解析、无条件分支、无环境写入（`$CLAUDE_ENV_FILE` 未使用）。

3. 本目录不负责插件启停/安装
- 安装、启用、市场分发由插件系统与 marketplace 清单负责。

### 改进建议
1. 修复校验脚本兼容性
- 在 `validate-hook-schema.sh` 中增加对 `{"hooks": {...}}` 结构自动下钻，或显式支持两种输入格式。

2. 增加目录级 smoke test
- 新增最小测试脚本：校验 `hooks.json` JSON 语法、handler 输出可解析、`hookEventName` 等于 `SessionStart`。

3. 提升配置显式性
- 评估在 `SessionStart` 条目增加 `matcher: "*"`（若运行时支持该字段），统一与其他 hooks 风格。

4. 抽取共享提示模板
- 将 explanatory insight 文案抽为可复用模板，供 `explanatory` 与 `learning` 两个插件共用，降低漂移。
