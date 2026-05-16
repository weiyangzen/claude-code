# plugins/explanatory-output-style 目录研究（DIR）

## 场景与职责

`plugins/explanatory-output-style` 是一个以 `SessionStart` Hook 为核心的轻量插件，目标是复刻已废弃的 Explanatory output style，把“讲解型输出偏好”从旧的 output style 配置迁移到插件机制。

它在仓库中的职责边界非常明确：
- 作为学习类插件被 marketplace 注册与发现：`.claude-plugin/marketplace.json:51-60`
- 在插件总览中被定义为“注入教育性上下文”的 `SessionStart` Hook：`plugins/README.md:19`
- 在目录内只维护 3 类资产：元数据（`plugin.json`）、Hook 配置（`hooks.json`）、Hook 处理脚本（`session-start.sh`）

它不负责代码生成、文件改写或外部系统调用，唯一行为是在会话启动时追加模型上下文。

## 功能点目的

### 1) 复刻 Explanatory 风格并承接迁移

- README 明确声明“recreates the deprecated Explanatory output style”：`plugins/explanatory-output-style/README.md:3-4`
- 给出迁移路径（旧配置 `"outputStyle": "Explanatory"` -> 安装该插件）：`plugins/explanatory-output-style/README.md:42-53`
- 结合变更历史，output style 先发布（`1.0.81`）后被弃用并建议迁移到 plugins（`2.0.30`）：`CHANGELOG.md:1837,1528`

### 2) 在每个会话开始时注入“教学导向”上下文

- `hooks/hooks.json` 将 `SessionStart` 事件绑定到命令 Hook：`plugins/explanatory-output-style/hooks/hooks.json:3-13`
- 处理脚本输出 `additionalContext`，要求模型在写代码前后给出简短 insight：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-10`

### 3) 强化“讲清楚为什么”而不改变任务主线

注入文本强调两点：
- 保持任务完成导向（不是纯教学闲聊）
- insight 聚焦“本仓库/当前改动特异性”，避免泛化编程常识：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:10`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 插件发现
- marketplace 条目将源目录指向 `./plugins/explanatory-output-style`：`.claude-plugin/marketplace.json:58`

2. 插件装载
- 元数据由 `.claude-plugin/plugin.json` 提供（名称、版本、描述、作者）：`plugins/explanatory-output-style/.claude-plugin/plugin.json:2-8`
- `plugin.json` 未覆写 `hooks` 字段时，按插件规范默认读取 `./hooks/hooks.json`：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`

3. Hook 注册
- `hooks/hooks.json` 使用插件格式包装（`{"hooks": {...}}`），事件为 `SessionStart`：`plugins/explanatory-output-style/hooks/hooks.json:3-14`
- Hook 类型为 `command`，命令路径采用 `${CLAUDE_PLUGIN_ROOT}` 保证可移植：`plugins/explanatory-output-style/hooks/hooks.json:8-10`

4. Hook 执行
- 会话开始时执行 `hooks-handlers/session-start.sh`
- 脚本输出 JSON：
  - `hookSpecificOutput.hookEventName = "SessionStart"`
  - `hookSpecificOutput.additionalContext = <长文本指令>`
  证据：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:7-12`

5. 运行时效果
- Hook 返回的 `additionalContext` 进入模型上下文，改变回答风格而非工具权限。
- 变更日志说明 output style 已固定在会话开始（有利 prompt caching），与该插件的 SessionStart 注入时机一致：`CHANGELOG.md:206`

### B. 数据结构与协议

1. 插件清单（manifest）
- 文件：`plugins/explanatory-output-style/.claude-plugin/plugin.json`
- 核心字段：`name`、`version`、`description`、`author`：`plugins/explanatory-output-style/.claude-plugin/plugin.json:2-8`

2. Hook 配置结构
- 插件格式（而非 settings 直写格式）要求 `hooks` 包装层：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`
- 当前配置仅定义一个 `SessionStart` 规则，无 `matcher`、无 `timeout` 字段：`plugins/explanatory-output-style/hooks/hooks.json:4-12`

3. Hook 输出协议
- 当前脚本返回 `hookSpecificOutput.additionalContext`（SessionStart 场景常用）：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-10`
- Hook 通用输出与退出码规范见 Hook 开发文档：`plugins/plugin-dev/skills/hook-development/SKILL.md:278-299`

### C. 关键命令与实测

1. 命令入口
- 注册命令：`${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh`：`plugins/explanatory-output-style/hooks/hooks.json:9`

2. 脚本行为
- 纯 `cat << 'EOF'` 输出静态 JSON，不读取 stdin，不访问网络，不读写工程文件：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:6-15`

3. 实测验证（本次）
- 使用通用测试脚本生成 SessionStart 输入并执行：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart`
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh /tmp/explanatory-sessionstart-sample.json`
- 结果：退出码 `0`，输出可被 `jq` 解析，包含 `hookSpecificOutput.additionalContext`。

## 关键代码路径与文件引用

### 目录内核心路径

- 元数据：`plugins/explanatory-output-style/.claude-plugin/plugin.json`
- 使用说明：`plugins/explanatory-output-style/README.md`
- Hook 配置：`plugins/explanatory-output-style/hooks/hooks.json`
- Hook 处理器：`plugins/explanatory-output-style/hooks-handlers/session-start.sh`

### 上游调用方（谁发现/触发它）

- 仓库插件市场清单：`.claude-plugin/marketplace.json:51-60`
- 插件总览入口：`plugins/README.md:19,43-45`
- 根 README 的插件入口索引：`README.md:48-50`

### 下游被调用方（它调用谁）

- Claude Code Hook 执行器（SessionStart 事件）
- 系统 `bash`（执行 `session-start.sh`）
- 环境变量展开机制（`${CLAUDE_PLUGIN_ROOT}`）

### 相关上下文文件（同类/耦合）

- `learning-output-style` 复用 explanatory 能力并扩展互动学习模式：
  - 文档声明：`plugins/learning-output-style/README.md:3-6,78-82`
  - 同构 Hook 配置：`plugins/learning-output-style/hooks/hooks.json:3-13`
  - 同构 SessionStart 脚本输出模式：`plugins/learning-output-style/hooks-handlers/session-start.sh:8-11`

## 依赖与外部交互

### 依赖

1. Claude Code 插件系统
- 负责插件发现、Hook 生命周期触发、输出上下文合并。

2. Bash 运行环境
- Hook 为 shell 脚本实现，依赖 `#!/usr/bin/env bash`：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:1`

3. 插件路径环境变量
- 命令通过 `${CLAUDE_PLUGIN_ROOT}` 定位处理脚本：`plugins/explanatory-output-style/hooks/hooks.json:9`
- Hook 文档建议所有插件命令都使用该变量以保持可移植：`plugins/plugin-dev/skills/hook-development/SKILL.md:327-337`

### 外部交互

- 无网络请求、无第三方 API 调用。
- 无仓库文件写操作。
- 主要“外部效应”是增加 prompt token 占用（README 已显式警告）：`plugins/explanatory-output-style/README.md:6-7`

### 配置 / 测试 / 脚本 / 文档覆盖

1. 配置
- `.claude-plugin/plugin.json`（插件元数据）
- `hooks/hooks.json`（事件到命令映射）

2. 测试
- 目录内没有专属 `test`/`spec` 自动化用例。
- 可借助通用 Hook 测试脚本做离线验证：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:1-252`

3. 脚本
- 目录内仅 `hooks-handlers/session-start.sh` 一个执行脚本。

4. 文档
- 插件文档：`plugins/explanatory-output-style/README.md`
- 上层目录文档：`plugins/README.md`
- 变更历史：`CHANGELOG.md`（output style 发布/弃用/迁移路径）

## 风险、边界与改进建议

### 风险与边界

1. token 成本持续增加
- 每次会话启动都注入较长 `additionalContext`，README 已提示会增加 token 成本：`plugins/explanatory-output-style/README.md:6-7`

2. 风格约束可能与任务场景冲突
- 指令要求“写代码前后都输出 insight”，在紧急修复或极短答复场景可能导致额外噪音：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:10`

3. 仅会话启动生效，无法中途细粒度切换
- 2.1.72 变更注明 output style 在 SessionStart 固定（为缓存优化），中途切换能力受限：`CHANGELOG.md:206`

4. 脚本验证工具与插件 hooks 包装格式存在错配
- `validate-hook-schema.sh` 当前按顶层事件键校验，直接校验插件 `hooks/hooks.json` 会误报并在后续 `jq` 处报错（本次实测 exit code=5）。
- 脚本实现：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`
- 插件 hooks 包装格式规范：`plugins/plugin-dev/skills/hook-development/SKILL.md:64-80`

5. 维护重复风险
- `learning-output-style` 内嵌 explanatory 能力，若后续解释模板演进，两个插件的提示词可能漂移：
  - `plugins/learning-output-style/README.md:5,80`
  - `plugins/learning-output-style/hooks-handlers/session-start.sh:10`

### 改进建议

1. 为 explanatory/learning 提示词抽取共享模板
- 把 insight 片段抽到可复用文本文件或生成脚本，减少双份维护漂移。

2. 增加最小回归测试脚本
- 在插件目录新增 smoke test，校验 `session-start.sh` 输出 JSON 可解析，且包含 `hookEventName` 与 `additionalContext`。

3. 为 Hook 配置补充显式 `matcher` 与 `timeout`
- 与其他插件风格对齐，提升配置可读性和运行时边界清晰度。

4. 修正 `validate-hook-schema.sh` 对插件格式的兼容
- 先检测并下钻 `.{hooks}`，再遍历事件键；避免对插件格式产生系统性误报。

5. 文档补充“适用边界”
- 在 README 增加建议：高吞吐/低延迟任务不建议启用；需要交互教学时启用该插件。
