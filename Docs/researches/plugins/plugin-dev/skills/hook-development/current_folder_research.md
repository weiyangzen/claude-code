# DIR 研究：plugins/plugin-dev/skills/hook-development

## 场景与职责

`plugins/plugin-dev/skills/hook-development` 是 `plugin-dev` 工具包中的 Hook 专项能力单元，面向 Claude Code 插件开发场景，负责把“事件驱动 Hook 自动化”从概念、配置、实现到验证串成完整工作流。

其在仓库中的职责边界：

- 对上游（调用方）提供：
  - 触发式知识加载入口（通过 SKILL frontmatter description 中的触发短语匹配）。
  - Hook 设计规范（事件、输入输出、退出码、性能/安全约束）。
  - 可直接执行的工具链（schema 校验 / 运行测试 / 脚本 lint）。
- 对下游（被调用方）要求：
  - 插件内 `hooks/hooks.json` 配置与 Hook 脚本实现。
  - 运行环境具备 `bash`、`jq`、`timeout` 等基础工具。
  - Claude Code Hook 运行时注入标准输入 JSON 与环境变量。

仓库内主要调用关系（文档/流程级，而非代码 import 级）：

- `plugin-dev` 总体 README 将 hook-development 作为 7 大 skill 之一，并在开发流程中明确“Add Automation / Test & Validate”阶段依赖该目录下工具。见：
  - `plugins/plugin-dev/README.md:56-75`
  - `plugins/plugin-dev/README.md:249-257`
  - `plugins/plugin-dev/README.md:273-284`
- `/plugin-dev:create-plugin` 命令在 Phase 5 和 Phase 6 明确要求加载 hook-development skill，并使用 `validate-hook-schema.sh`、`test-hook.sh`。见：
  - `plugins/plugin-dev/commands/create-plugin.md:201-209`
  - `plugins/plugin-dev/commands/create-plugin.md:257-260`
- `plugin-validator` agent 在 hooks 校验步骤直接引用该 skill 的 schema 校验工具。见：
  - `plugins/plugin-dev/agents/plugin-validator.md:107-115`

结论：该目录是 `plugin-dev` 内“Hook 领域知识 + 本地验证工具”二合一中枢。

## 功能点目的

### 1) 核心技能文档（`SKILL.md`）

目标是让模型在触发后快速掌握：

- Hook 类型选择：prompt vs command（推荐 prompt 做复杂策略判断，command 做确定性快速检查）。
- 事件面覆盖：`PreToolUse/PostToolUse/Stop/SubagentStop/SessionStart/SessionEnd/UserPromptSubmit/PreCompact/Notification`。
- 输入输出与协议：stdin JSON 输入、结构化 JSON 输出、退出码语义。
- 工程实践：`$CLAUDE_PLUGIN_ROOT` 可移植路径、并行执行约束、超时、安全校验、调试与测试流程。

关键位置：

- 类型与事件：`SKILL.md:20-59`、`121-277`
- 输入输出与退出码：`SKILL.md:278-320`
- 环境变量与可移植路径：`SKILL.md:322-339`
- 并行与性能约束：`SKILL.md:493-518`
- 生命周期（需重启加载）：`SKILL.md:572-599`
- 实施步骤：`SKILL.md:698-712`

### 2) 参考文档（`references/*.md`）

提供“从常规到高级再到迁移”的分层策略：

- `patterns.md`：10 个可复用模式模板（安全校验、测试门禁、上下文注入、配置驱动等）。
- `advanced.md`：多阶段校验、跨事件协同、外部系统集成、性能与安全模式。
- `migration.md`：从 command-hook 到 prompt-hook 的迁移路径、保留 command-hook 的判定标准、混合架构建议。

目的不是重复 SKILL 主文，而是补齐“可复制片段 + 适用边界 + 演进策略”。

### 3) 示例脚本（`examples/*.sh`）

用于给出最低可运行样板：

- `validate-write.sh`：PreToolUse 写入路径/敏感文件校验。
- `validate-bash.sh`：PreToolUse Bash 命令风险分级（allow/deny/ask）。
- `load-context.sh`：SessionStart 阶段项目类型探测并写入 `CLAUDE_ENV_FILE`。

### 4) 工具脚本（`scripts/*.sh`）

形成开发闭环：

- `validate-hook-schema.sh`：配置静态校验。
- `test-hook.sh`：本地仿真执行与结果分析。
- `hook-linter.sh`：脚本最佳实践/安全风险扫描。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. Hook 配置数据结构

`SKILL.md` 定义了两类配置外壳：

- 插件 hooks 文件（声明为 wrapper 结构）：
  - 顶层可选 `description`
  - 顶层必有 `hooks`，事件位于 `hooks` 内
  - 见 `SKILL.md:62-80`
- 用户 settings 直写结构：
  - 顶层直接是事件键
  - 见 `SKILL.md:102-119`

事件项内部通用结构：

- `matcher`: 工具匹配规则（精确、管道、`*`、regex）
- `hooks`: 钩子数组
  - `type: command` + `command` (+ `timeout`)
  - 或 `type: prompt` + `prompt` (+ `timeout`)

### B. 输入输出协议

命令 Hook 协议（由 Claude Code 运行时驱动）：

- 输入：stdin JSON
  - 公共字段：`session_id/transcript_path/cwd/permission_mode/hook_event_name`
  - 事件特有字段：如 `tool_name/tool_input/tool_result/user_prompt/reason`
  - 见 `SKILL.md:300-320`
- 输出：JSON（可包含 `continue/suppressOutput/systemMessage` 或事件特定字段）
  - 见 `SKILL.md:280-293`
- 退出码约定：
  - `0` 通过
  - `2` 阻断并将 stderr 反馈模型
  - 其他为非阻断异常
  - 见 `SKILL.md:294-299`

### C. `validate-hook-schema.sh` 流程

入口参数：`<path/to/hooks.json>`。

主要步骤：

1. 参数和文件存在性校验（`validate-hook-schema.sh:8-25`）
2. `jq empty` 做 JSON 语法校验（`30-37`）
3. 顶层 key 与合法事件集合比对（`41-57`）
4. 双层循环逐条校验：
   - `matcher` 必填（`70-75`）
   - `hooks` 数组必填（`78-83`）
   - `type` 必须 `command|prompt`（`89-101`）
   - command/prompt 特定字段校验（`104-127`）
   - `timeout` 数值及范围告警（`131-142`）
5. 聚合 `error_count` 与 `warning_count` 决定退出码（`148-159`）

协议实现特点：

- 以“错误阻断 + 告警放行”为主。
- 对 prompt event 兼容性给告警而非硬失败。

### D. `test-hook.sh` 流程

功能：构造/消费测试输入，模拟 Hook 在本地运行。

关键流程：

1. 参数解析，支持 `--create-sample` 生成不同事件样例（`25-100`、`106-127`）
2. 校验 Hook 脚本、输入 JSON（`138-158`）
3. 注入运行环境变量：
   - `CLAUDE_PROJECT_DIR`
   - `CLAUDE_PLUGIN_ROOT`
   - `CLAUDE_ENV_FILE`
   - 见 `170-174`
4. 使用 `timeout ... bash -c "cat input | hook"` 执行（`189-191`）
5. 解释退出码语义并尝试 `jq` 解析输出（`205-233`）
6. 如产生 env 文件，打印并清理（`236-241`）

该脚本使 Hook 开发从“只能在 Claude Code 内调试”变为“可离线回归”。

### E. `hook-linter.sh` 流程

静态规则扫描（13 项）覆盖：

- 可执行权限、shebang、`set -euo pipefail`
- stdin 读取、`jq` 使用倾向
- 变量未引用风险、硬编码路径
- 退出码惯例、潜在长耗时逻辑、stderr 使用
- 输入校验建议

实现方式以 grep/regex 为主（`36-117`），最终按 errors/warnings 聚合退出（`122-153`）。

### F. 示例脚本实现要点

- `validate-write.sh`：
  - 从 `.tool_input.file_path` 读取路径（`11`）
  - path traversal、系统目录、敏感文件分类处理（`19-35`）
  - 使用 `permissionDecision` + `systemMessage` 返回（`21`、`27`、`33`）
- `validate-bash.sh`：
  - 从 `.tool_input.command` 解析命令（`11`）
  - 先快路径放行安全命令（`19-22`）
  - 再拦截破坏性命令与提权命令（`25-40`）
- `load-context.sh`：
  - 进入 `CLAUDE_PROJECT_DIR` 并探测项目类型（`8-47`）
  - 通过写 `CLAUDE_ENV_FILE` 持久化会话环境（`15`、`19`、`24` 等）

### G. 典型命令清单

- 配置校验：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh hooks/hooks.json`
- 生成样例输入：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample PreToolUse`
- 执行 Hook 单测：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh my-hook.sh test-input.json`
- 脚本 lint：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh my-hook.sh`

## 关键代码路径与文件引用

目标目录核心文件：

- `plugins/plugin-dev/skills/hook-development/SKILL.md`
- `plugins/plugin-dev/skills/hook-development/references/patterns.md`
- `plugins/plugin-dev/skills/hook-development/references/advanced.md`
- `plugins/plugin-dev/skills/hook-development/references/migration.md`
- `plugins/plugin-dev/skills/hook-development/examples/validate-write.sh`
- `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh`
- `plugins/plugin-dev/skills/hook-development/examples/load-context.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/README.md`

上游调用链（流程入口）：

- `plugins/plugin-dev/README.md:56-75,249-257,273-284`
- `plugins/plugin-dev/commands/create-plugin.md:201-209,257-260`
- `plugins/plugin-dev/agents/plugin-validator.md:107-115`
- `plugins/plugin-dev/skills/plugin-structure/README.md:95-100`（作为关联 skill）
- `plugins/plugin-dev/skills/skill-development/SKILL.md:298-304`（作为“优秀 skill 范例”被反向引用）

下游被作用对象（配置消费者）：

- 仓库内插件 hooks 配置文件：
  - `plugins/explanatory-output-style/hooks/hooks.json`
  - `plugins/hookify/hooks/hooks.json`
  - `plugins/learning-output-style/hooks/hooks.json`
  - `plugins/ralph-wiggum/hooks/hooks.json`
  - `plugins/security-guidance/hooks/hooks.json`

## 依赖与外部交互

### 运行时依赖

- Shell 环境：`bash`
- JSON 工具：`jq`（schema 校验、测试输入校验、示例脚本字段读取）
- 超时控制：`timeout`（`test-hook.sh`）
- 常见基础命令：`cat/head/grep/stat/date/nc/curl/psql`（后者多见于 references 高级示例）

### Claude Code 环境变量与协议依赖

- `CLAUDE_PROJECT_DIR`
- `CLAUDE_PLUGIN_ROOT`
- `CLAUDE_ENV_FILE`
- `CLAUDE_CODE_REMOTE`

见：`SKILL.md:324-330`。

### 外部系统交互（文档级示例）

`references/advanced.md` 提供可选外部集成样式：

- Slack Webhook（`curl`）
- 数据库日志（`psql`）
- 指标系统（`nc` -> statsd）

说明该 skill 支持从本地校验扩展到组织级审计/观测平台，但仓库当前仅提供示例，不绑定具体生产端点。

### 测试与验证现状

- 该目录内没有自动化测试框架（如 Bats/shunit2/pytest）集成。
- 当前质量保障主要依赖手工执行脚本：
  - `validate-hook-schema.sh`
  - `test-hook.sh`
  - `hook-linter.sh`
- `plugin-dev` 文档流程要求在交付前执行这些脚本，但并未看到 CI 强制门禁。

## 风险、边界与改进建议

### 1) 配置格式约定存在潜在不一致

现象：

- `SKILL.md` 声明插件 `hooks/hooks.json` 推荐 wrapper 格式 `{"description":...,"hooks":{...}}`（`SKILL.md:62-80`）。
- `validate-hook-schema.sh` 直接把顶层 key 当事件名遍历（`validate-hook-schema.sh:41-43`），即偏向“直写事件顶层”结构。

影响：

- 对 wrapper 格式配置可能出现误报（如把 `description/hooks` 识别为未知 event）。
- 工具与文档认知分裂，降低可用性。

建议：

- 在 schema 校验器中同时兼容两种格式：若存在顶层 `hooks` 则转入 `jq '.hooks'` 校验。
- 在脚本输出中明确提示当前解析模式（wrapper/direct）。

### 2) `matcher` 必填假设与仓库示例实践有偏差

现象：

- 校验器强制每个事件项存在 `matcher`（`70-75`）。
- 仓库内多个实际插件 `hooks/hooks.json` 事件项未显式给 `matcher`（例如 explanatory-output-style、hookify、learning-output-style、ralph-wiggum）。

影响：

- 使用该校验器检查现有插件配置时可能出现系统性错误。

建议：

- 统一规范：若 matcher 可缺省，应在校验器中设默认 `*`；若必须填写，应批量修复示例与插件配置。

### 3) `load-context.sh` 的 CI 检测条件可能误判

现象：

- 脚本使用 `[ -f ".github/workflows" ]`（`examples/load-context.sh:50`），但该路径常为目录。

影响：

- 常见 GitHub Actions 项目会误判 `HAS_CI=false`。

建议：

- 改为目录判断 `-d .github/workflows`，并保留对 `.gitlab-ci.yml/.circleci/config.yml` 的文件判断。

### 4) lint 规则存在误报/漏报边界

现象：

- `hook-linter.sh` 的“未引用变量检测”使用简化 regex（`68-72`），在复杂脚本中可能误报。
- “是否读取 stdin”仅通过 `cat|read` 文本匹配（`56-59`），可能漏掉函数封装读取场景。

建议：

- 将 lint 定位为“启发式检查”，在 README 中明确 non-authoritative。
- 若后续要提高精度，可引入 `shellcheck` 作为补充并分级输出。

### 5) 缺少 CI 自动执行

现象：

- 当前主要依赖手工命令执行，未见 CI 对 hooks schema/lint/test 的统一门禁。

建议：

- 增加 `scripts/check-hooks.sh` 聚合脚本，供 CI 一键执行。
- 在 `plugin-dev` 或各插件仓库中接入最小 CI step，保证回归一致性。

### 6) 并行执行模型下的状态共享风险

`SKILL.md` 与 `advanced.md` 已强调 hooks 并行执行、不应依赖顺序；但高级示例中“临时文件串联状态”只在跨事件串行情境安全。若开发者误用于同事件并行 hooks，可能出现竞态。

建议：

- 在 `advanced.md` 的状态共享段落增加“同事件并行禁用”红色警示示例。
- 为跨事件状态命名引入 session 粒度键，减少冲突。

---

综合判断：`hook-development` 目录设计完整、教学路径清晰、工具链实用，是 `plugin-dev` 中工程化程度较高的 skill；当前主要问题集中在“文档格式规范与校验器行为的一致性”以及“示例对规范收敛不足”。优先修复 schema 兼容和 matcher 规则，可显著降低使用摩擦。
