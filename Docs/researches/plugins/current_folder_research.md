# plugins 目录研究（DIR）

## 场景与职责

### 目录定位
`plugins/` 是本仓库内置插件集合，承载 Claude Code 的可扩展能力样例与官方工作流模板。它不是核心运行时实现，而是“插件内容层”（命令/代理/技能/Hook）。

- 仓库入口对该目录的上游引用：`README.md:48-50`
- 插件目录总览与结构约定：`plugins/README.md:11-61`
- 市场清单对本目录的注册（调用方入口）：`.claude-plugin/marketplace.json:10-149`

### 规模概览（当前仓库状态）
- 插件目录数：13
- 命令文件：15（`commands/*.md`）
- Agent 文件：15（`agents/*.md`）
- Skill 主文件：10（`skills/*/SKILL.md`）
- Hook 配置：5（`hooks/hooks.json`）
- Hook/脚本实现：16（`hooks/*.py|*.sh`, `hooks-handlers/*.sh`, `scripts/*.sh`）
- 文件总数：138

### 角色分层
1. 业务工作流插件
   - `feature-dev`、`code-review`、`commit-commands`、`pr-review-toolkit`、`agent-sdk-dev`
2. 输出风格/行为控制插件
   - `explanatory-output-style`、`learning-output-style`、`hookify`、`security-guidance`、`ralph-wiggum`
3. 开发方法与元插件
   - `plugin-dev`（用于开发插件本身）
4. 专项能力插件
   - `frontend-design`、`claude-opus-4-5-migration`

## 功能点目的

### 插件级能力目标
- `agent-sdk-dev`：以 `/new-sdk-app` + 双 verifier agent 提供 Agent SDK 项目脚手架与验收。
- `claude-opus-4-5-migration`：将 Sonnet/Opus 旧模型串迁移到 Opus 4.5，并按需注入提示词修复片段。
- `code-review`：围绕 PR 的高置信度自动审查与（可选）Inline 评论。
- `commit-commands`：封装 `commit / push / pr` 与 `gone` 分支清理。
- `feature-dev`：7 阶段特性开发流程（探索-澄清-设计-实现-复核）。
- `pr-review-toolkit`：6 个专项审查 agent（评论、测试、静默失败、类型设计、通用审查、简化）。
- `frontend-design`：为前端任务提供高辨识度设计约束，抑制“模板化 AI 风格”。
- `explanatory-output-style` / `learning-output-style`：通过 `SessionStart` 注入额外系统上下文。
- `hookify`：把用户意图转为 `.claude/hookify.*.local.md` 规则，并在 Hook 事件中动态执行。
- `ralph-wiggum`：基于 Stop Hook 的会话内循环执行机制。
- `security-guidance`：在写文件前匹配安全敏感模式并阻断提醒。
- `plugin-dev`：插件开发全流程工具链（命令 + agents + 7 个 skills + 验证脚本）。

### 组件组织目的
- `commands/`：把可复用流程封装为 Slash Command。
- `agents/`：把专门分析/生成能力分离为可触发子代理。
- `skills/`：沉淀领域知识与操作规约。
- `hooks/`：在会话事件点进行自动拦截/注入。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 插件注册与发现链路
1. 市场清单声明插件源：`.claude-plugin/marketplace.json:10-149`
2. 每个插件最小元数据：`plugins/*/.claude-plugin/plugin.json`
3. 运行时依据标准目录发现命令/agents/skills/hooks（目录约定见 `plugins/README.md:49-61`）

要点：当前插件 `plugin.json` 基本只含 `name/version/description/author`，不依赖复杂 manifest 扩展字段。

### 2) Command 协议与执行模型
Command 文件普遍采用“Frontmatter + 指令正文”模型：
- frontmatter 常见字段：`description`, `argument-hint`, `allowed-tools`
- 正文是“写给 Claude 的执行指令”，不是面向终端用户的文档

代表性流程：
- PR 自动审查：`plugins/code-review/commands/code-review.md:1-109`
  - `gh` + MCP inline comment 工具
  - 多 agent 并发审查 + 二次验证 + 置信度过滤
- 一次性提交推送建 PR：`plugins/commit-commands/commands/commit-push-pr.md:1-20`
  - 强约束“单次消息内完成工具调用”
- 插件开发 8 阶段：`plugins/plugin-dev/commands/create-plugin.md:1-415`

### 3) Hook 协议（事件、输入、输出）
Hook 配置在 `hooks/hooks.json`，由事件映射到 command/prompt hook：
- 例：`hookify` 同时注册 `PreToolUse/PostToolUse/Stop/UserPromptSubmit`
  - `plugins/hookify/hooks/hooks.json:1-49`
- 例：`security-guidance` 仅拦截写文件工具
  - `plugins/security-guidance/hooks/hooks.json:1-16`

输出协议特征：
- PreToolUse 可通过 `permissionDecision: deny` 阻断
- Stop 可通过 `{decision: block, reason: ...}` 阻断退出
- Hook 执行器多数“失败兜底为放行”（`sys.exit(0)`）

### 4) Hookify 规则引擎（核心实现）
核心数据结构与流程：
- `Condition` / `Rule`：`plugins/hookify/core/config_loader.py:16-84`
- Frontmatter 解析：`extract_frontmatter`（自实现 YAML 子集解析）`...:87-196`
- 规则加载与事件过滤：`load_rules` / `load_rule_file` `...:198-278`
- 规则求值：`RuleEngine.evaluate_rules` `plugins/hookify/core/rule_engine.py:35-94`
- 字段抽取与操作符：`_extract_field` / `_check_condition` `...:144-255`

事件执行器：
- `pretooluse.py`, `posttooluse.py`, `stop.py`, `userpromptsubmit.py`
  - 从 stdin 读 hook input JSON
  - 映射事件（如 `Bash -> bash`, `Edit/Write/MultiEdit -> file`）
  - 调用 RuleEngine，输出 JSON

### 5) Ralph 循环机制（状态机实现）
- 初始化：`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh`
  - 解析 `--max-iterations`、`--completion-promise`
  - 写状态文件 `.claude/ralph-loop.local.md`（frontmatter + prompt）
- 循环控制：`plugins/ralph-wiggum/hooks/stop-hook.sh`
  - 读取状态、读取 transcript、提取最后 assistant 文本
  - 检测 `<promise>...</promise>` 与上限
  - 未完成则 `decision=block` 并把原 prompt 回注到 `reason`

关键命令/协议依赖：`jq`, `awk`, `sed`, `perl`, transcript JSONL。

### 6) Security Guidance 拦截链路
实现文件：`plugins/security-guidance/hooks/security_reminder_hook.py`
- 规则库：`SECURITY_PATTERNS`（9 类，含 GitHub Actions 注入、`exec/eval/new Function`、XSS、pickle、os.system）`...:31-126`
- 会话态去重：`~/.claude/security_warnings_state_<session>.json` `...:129-180`
- 匹配中后输出提醒并 `sys.exit(2)` 阻断写操作 `...:271-273`
- 通过 `ENABLE_SECURITY_REMINDER=0` 可关闭 `...:220-224`

### 7) plugin-dev 的“自举工具链”
plugin-dev 不只是文档，还内置校验/测试脚本：
- Agent 校验：`skills/agent-development/scripts/validate-agent.sh`
- Hook schema 校验：`skills/hook-development/scripts/validate-hook-schema.sh`
- Hook lint/test：`hook-linter.sh`, `test-hook.sh`
- 设置解析/校验：`plugin-settings/scripts/parse-frontmatter.sh`, `validate-settings.sh`

## 关键代码路径与文件引用

### 顶层与注册
- `README.md:48-50`
- `plugins/README.md:11-61`
- `.claude-plugin/marketplace.json:10-149`

### 高价值实现路径
- Hookify 引擎
  - `plugins/hookify/hooks/hooks.json:1-49`
  - `plugins/hookify/hooks/pretooluse.py:1-75`
  - `plugins/hookify/hooks/posttooluse.py:1-67`
  - `plugins/hookify/hooks/stop.py:1-59`
  - `plugins/hookify/core/config_loader.py:16-278`
  - `plugins/hookify/core/rule_engine.py:27-267`
- Ralph 循环
  - `plugins/ralph-wiggum/commands/ralph-loop.md:1-17`
  - `plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:1-205`
  - `plugins/ralph-wiggum/hooks/stop-hook.sh:1-179`
- 安全提醒
  - `plugins/security-guidance/hooks/hooks.json:1-16`
  - `plugins/security-guidance/hooks/security_reminder_hook.py:31-280`
- 会话注入型输出风格
  - `plugins/explanatory-output-style/hooks/hooks.json:1-15`
  - `plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-17`
  - `plugins/learning-output-style/hooks/hooks.json:1-15`
  - `plugins/learning-output-style/hooks-handlers/session-start.sh:1-17`

### 代表性命令路径
- `plugins/code-review/commands/code-review.md:1-109`
- `plugins/commit-commands/commands/commit.md:1-17`
- `plugins/commit-commands/commands/commit-push-pr.md:1-20`
- `plugins/commit-commands/commands/clean_gone.md:1-52`
- `plugins/feature-dev/commands/feature-dev.md:1-120`
- `plugins/plugin-dev/commands/create-plugin.md:1-415`
- `plugins/pr-review-toolkit/commands/review-pr.md:1-189`

### 参考资料体系（plugin-dev）
- 7 个 Skill 主入口：`plugins/plugin-dev/skills/*/SKILL.md`
- 21 个 references 文档 + 9 个 examples 文档 + 6 个脚本
  - 涵盖 command frontmatter、hook event 协议、MCP server 类型与认证、manifest 字段规范、skill progressive disclosure 等。

## 依赖与外部交互

### 运行时依赖（命令/脚本）
- Shell 工具：`bash`, `sed`, `awk`, `grep`, `find`, `timeout`, `date`
- JSON 工具：`jq`（Ralph Hook、plugin-dev hook test/validate）
- Python：`python3`（hookify/security-guidance）
- Git/GitHub：`git`, `gh`（commit-commands、code-review、pr-review-toolkit）
- Perl：Ralph stop hook 用于多行 `<promise>` 提取

### 运行时文件交互
- 项目态配置：`.claude/hookify.*.local.md`, `.claude/ralph-loop.local.md`
- 会话状态：`~/.claude/security_warnings_state_<session>.json`
- transcript 读取：Stop hook 从 hook input 的 `transcript_path` 读取 JSONL

### 外部网络/文档依赖
- Agent SDK 命令与 verifier 依赖 docs.claude.com、npm、PyPI 链接
- `code-review` / `commit-push-pr` 依赖 GitHub API（通过 `gh` 或 MCP inline comment）
- `mcp-integration` skill 文档覆盖 SSE/HTTP/WS/stdio 远程服务配置

### 调用方 / 被调用方关系
- 调用方（上游）：Claude Code 插件加载器、`/plugin` 管理体系（变更记录见 `CHANGELOG.md` 多处条目）
- 被调用方（下游）：
  - 本地工具链（git/gh/jq/python）
  - 远端文档与服务（GitHub、Agent SDK 文档、MCP server endpoint）
  - 项目工作区与用户 home 下状态文件

## 风险、边界与改进建议

### 已验证风险（实测/静态阅读）
1. `validate-hook-schema.sh` 与插件 hooks 包装格式不兼容
   - 脚本按顶层事件键校验，但插件 `hooks/hooks.json` 实际是 `{"hooks": {...}}` 包装。
   - 实测：`bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/hookify/hooks/hooks.json`
     - 出现 `Unknown event type: description/hooks`，随后 `jq: Cannot index string with number`。

2. `validate-agent.sh` 在 warning 分支可能提前退出
   - 脚本启用 `set -euo pipefail`，并使用 `((warning_count++))`；当表达式返回 0 时会触发非零退出状态，导致流程中断。
   - 现象：多个 agent 在第一次 warning 后即提前停止输出完整报告。

3. Ralph 帮助文档中的状态文件路径有不一致
   - `commands/help.md` 写 `.claude/.ralph-loop.local.md`
   - 真实脚本与其他命令均用 `.claude/ralph-loop.local.md`

4. `pr-review-toolkit/agents/type-design-analyzer.md` 使用 `color: pink`
   - 与 `validate-agent.sh` 允许集（blue/cyan/green/yellow/magenta/red）不一致，导致工具侧 warning。

5. `hookify` frontmatter 解析器为自实现 YAML 子集
   - 优点：无第三方依赖。
   - 风险：复杂 YAML 结构/转义语义兼容性有限，规则文件一旦复杂化会出现解析边界。

6. `security-guidance` 缺少独立 README
   - 与其余插件不一致，降低可发现性和维护透明度。

7. 自动化测试薄弱
   - `plugins/` 下无系统化单元/集成测试套件，主要依赖“脚本验证 + 人工触发”。

### 设计边界
- 目录内大多为“提示工程与工作流定义”，其正确性强依赖 Claude 运行时行为与外部工具可用性。
- 多数插件并不“执行业务代码”，而是编排其他工具，因此失败模式常来自环境（权限、CLI 版本、登录态、网络）。

### 改进建议（优先级）
1. 修复 plugin-dev 校验脚本与真实格式对齐
   - `validate-hook-schema.sh` 同时支持 `plugin hooks wrapper` 与 `settings direct` 两种根结构。
   - `validate-agent.sh` 规避 `set -e` 与 `((x++))` 的退出码陷阱（例如改为 `x=$((x+1))`）。

2. 建立最小 CI 回归
   - 对每个插件做基础校验：JSON 语法、frontmatter 解析、hooks schema smoke test、关键脚本 dry-run。

3. 统一 Ralph 文档路径
   - 修正 `.claude/.ralph-loop.local.md` -> `.claude/ralph-loop.local.md`。

4. 补齐 `security-guidance/README.md`
   - 明确触发条件、环境变量、阻断策略、状态文件生命周期。

5. 为 Hookify 增加解析鲁棒性
   - 至少增加 frontmatter 解析失败时的错误定位与示例修复提示；中长期可考虑标准 YAML 解析方案。

6. 引入“插件能力矩阵”自动生成
   - 从 `plugins/*` 自动汇总 command/agent/skill/hook 清单，降低文档漂移。
