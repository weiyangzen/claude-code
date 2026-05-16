# DIR `examples` 研究文档

## 场景与职责

`examples/` 是仓库中的“可复制示例资产”目录，当前包含两类内容：

1. `examples/hooks/`：Claude Code Hook 示例脚本（Python）。
2. `examples/settings/`：Claude Code Settings 示例配置（JSON + 说明文档）。

它在仓库中的角色不是“被 CI/脚本直接执行的运行时代码”，而是“供用户或组织管理员手工接入的参考模板”。

调用关系可归纳为：

1. 上游调用方（谁触发）
- Claude Code 用户/管理员手动复制示例到自身配置体系。
- 在 Hook 场景中，由 Claude Code 的 `PreToolUse` 事件触发命令脚本。

2. 下游被调用方（它影响谁）
- `examples/hooks/bash_command_validator_example.py` 被 Claude Code Hook 运行器以 stdin JSON 协议调用。
- `examples/settings/*.json` 被 Claude Code Settings 层级配置系统读取（enterprise/user/project 等层级，按 `examples/settings/README.md` 说明）。

3. 与仓库其他目录边界
- 根仓库内未发现 workflow、脚本或命令对 `examples/` 的直接调用；它不属于自动化主链。
- 其功能和 `plugins/*/examples` 类似，主要用于示例和学习，但该目录面向“仓库级通用示例”。

## 功能点目的

### 1. Bash Hook 校验示例
- 文件：`examples/hooks/bash_command_validator_example.py`
- 目的：在 Bash 工具执行前进行静态命令检查，优先引导使用 `rg` 替代低效 `grep/find -name` 用法。
- 设计取向：以“阻断 + 提示”方式把命令规范前置到执行前（`PreToolUse`）。

### 2. Settings Lax 示例
- 文件：`examples/settings/settings-lax.json`
- 目的：提供“低侵入治理”最小集配置。
- 覆盖点：
  - 禁用 `--dangerously-skip-permissions`。
  - 限制插件 marketplace（依据同目录 README 的配置矩阵定义）。

### 3. Settings Strict 示例
- 文件：`examples/settings/settings-strict.json`
- 目的：提供组织级更强约束样例。
- 覆盖点：
  - Bash 工具强制进入 `ask`。
  - 拒绝 `WebSearch`/`WebFetch`。
  - 限制用户/项目自定义权限规则与 hooks。
  - 配置 Bash sandbox 网络与本地绑定约束。

### 4. Bash Sandbox 专项示例
- 文件：`examples/settings/settings-bash-sandbox.json`
- 目的：专门演示“Bash 必须在 sandbox 中执行”的配置组合。
- 覆盖点：`sandbox.enabled=true` + `allowUnsandboxedCommands=false` + 受限网络配置。

### 5. 文档化入口
- 文件：`examples/settings/README.md`
- 目的：作为三套 JSON 示例的说明与选型对照表，强调“社区维护、需自行验证”。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Hook 示例实现：`bash_command_validator_example.py`

#### 关键流程

1. 读取并解析 stdin JSON
- `input_data = json.load(sys.stdin)`。
- JSON 解析失败：stderr 输出错误并 `sys.exit(1)`。

2. 筛选工具类型
- 仅处理 `tool_name == "Bash"`，其他工具直接 `exit 0`。

3. 提取命令字段
- 从 `tool_input.command` 读取待检查命令。
- 空命令直接放行。

4. 正则规则校验
- `_VALIDATION_RULES` 为 `(pattern, message)` 元组列表。
- 目前两条规则：
  - `^grep\b(?!.*\|)`：建议改用 `rg`。
  - `^find\s+\S+\s+-name\b`：建议改用 `rg --files` 组合。

5. 反馈与阻断
- 命中规则：逐条写 stderr（`• message`）并 `sys.exit(2)`。
- 未命中：`exit 0`。

#### 协议与命令

1. Hook 注册协议（脚本 docstring 内嵌示例）
- 事件：`PreToolUse`
- 匹配器：`matcher: "Bash"`
- 命令：`python3 /path/to/claude-code/examples/hooks/bash_command_validator_example.py`

2. Hook 输入协议（运行时）
- 来自 Claude Code 的 JSON，至少包含：
  - `tool_name`
  - `tool_input.command`

3. Exit code 约定
- `0`：放行。
- `1`：输入异常（对用户可见错误，不阻断为策略拒绝）。
- `2`：策略阻断（PreToolUse 拒绝执行）。
- 该行为与 `plugins/security-guidance/hooks/security_reminder_hook.py` 的 `sys.exit(2)` 阻断模式一致。

### 2) Settings 示例实现：`examples/settings/*.json`

#### 数据结构

1. `settings-lax.json`
- `permissions.disableBypassPermissionsMode = "disable"`
- `strictKnownMarketplaces = []`

2. `settings-strict.json`
- `permissions.ask = ["Bash"]`
- `permissions.deny = ["WebSearch", "WebFetch"]`
- `allowManagedPermissionRulesOnly = true`
- `allowManagedHooksOnly = true`
- `strictKnownMarketplaces = []`
- `sandbox` 子对象：
  - `autoAllowBashIfSandboxed=false`
  - `network.allowUnixSockets=[]`
  - `network.allowAllUnixSockets=false`
  - `network.allowLocalBinding=false`
  - `network.allowedDomains=[]`
  - `httpProxyPort/socksProxyPort=null`

3. `settings-bash-sandbox.json`
- `allowManagedPermissionRulesOnly = true`
- `sandbox.enabled = true`
- `sandbox.allowUnsandboxedCommands = false`
- 其余网络与 strict 类似。

#### 演进细节（提交历史）

1. `2026-01-30`：新增 settings 三套示例（`f93f614`）。
2. `2026-01-30`：strict 配置补充 sandbox（`90c07d1`）。
3. `2026-02-01`：strict 与 bash-sandbox 同步新增 `allowAllUnixSockets`、`allowedDomains` 字段（`4936302`、`43d0eac`）。

### 3) 文档与脚本上下文

1. 文档上下文
- `examples/settings/README.md` 提供功能矩阵、层级说明与风险声明。
- 仓库根 `README.md` 未把 `examples/` 作为主入口，说明其定位偏“辅助示例”。

2. 脚本上下文
- 仓库内无脚本直接执行 `examples/`。
- 研究流程脚本（`.ops/generate_research_blueprint_checklist.sh`、`.ops/generate_daily_research_todo.sh`）仅管理研究任务，不消费示例内容本身。

3. 测试上下文
- 未发现针对 `examples/` 的自动化测试或 CI 校验。
- 目前质量保证主要依赖人工复制、手工验证、提交演进。

## 关键代码路径与文件引用

### 目标目录主文件
- `examples/hooks/bash_command_validator_example.py`
- `examples/settings/README.md`
- `examples/settings/settings-lax.json`
- `examples/settings/settings-strict.json`
- `examples/settings/settings-bash-sandbox.json`

### 关键上下文（调用方/被调用方/协议对照）
- `plugins/security-guidance/hooks/hooks.json`（生产级 PreToolUse 注册样例）
- `plugins/security-guidance/hooks/security_reminder_hook.py`（`sys.exit(2)` 阻断对照实现）
- `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh`（Bash hook 校验示例的平行实现）

### 研究流程关联
- `.ops/generate_research_blueprint_checklist.sh`
- `.ops/generate_daily_research_todo.sh`
- `Docs/researches/blueprint_checklist.md`
- `Docs/researches/todos_20260319.md`

### 调用链（调用方 -> 被调用方）
1. 用户/管理员配置 hooks -> Claude Code `PreToolUse` -> `examples/hooks/bash_command_validator_example.py`
2. 用户/管理员应用 settings JSON -> Claude Code settings 层级解析 -> 权限/marketplace/sandbox 策略生效
3. 研究守护脚本 -> checklist/todo 更新（与 `examples` 内容无运行时耦合）

## 依赖与外部交互

### 本地依赖

1. Hook 示例依赖
- `python3` 解释器。
- Python 标准库：`json`、`re`、`sys`。

2. Settings 示例依赖
- Claude Code settings 解析能力（运行时由 Claude Code 提供）。

### 外部交互

1. Hook 示例
- 无主动网络请求。
- 仅通过 stdin/stderr 与 Claude Code Hook 运行器交互。

2. Settings 示例
- JSON 文件本身无外部调用；由 Claude Code 读取后才影响运行策略。
- 文档中引用外部说明：`https://code.claude.com/docs/en/settings`。

### 测试与验证现状

1. 仓库中未发现 `examples/` 的自动化测试。
2. 没有 schema 校验脚本自动验证 `examples/settings/*.json` 与最新字段一致性。
3. Hook 示例未集成自动化回归（例如基于固定输入样本断言 exit code）。

## 风险、边界与改进建议

### 风险

1. Hook 规则覆盖面有限
- 当前正则只匹配以 `grep`/`find ... -name` 开头命令。
- `sudo grep`、子 shell、管道组合等变体可能绕过提示。

2. Hook 输出形态较简
- 仅 stderr 文本提示，不输出结构化拒绝 JSON；不同客户端展示一致性依赖运行器实现。

3. 示例与产品能力存在漂移风险
- Settings 字段若随版本演进，示例可能过时。
- 当前缺少自动化 schema 对齐检查。

4. `strictKnownMarketplaces: []` 的策略可读性风险
- 从 README 矩阵看该配置表达“阻断 marketplace”；但仅看 JSON 本身不直观，易被误解。

5. 无测试保护
- 示例文件改动后，无法在 CI 中及时发现语法/语义回归。

### 边界

1. `examples/` 不承诺生产即插即用，只是参考模板。
2. 该目录不负责 hooks 的分发注册，不负责 settings 的层级合并逻辑。
3. 目录内文件不参与仓库主流程自动执行。

### 改进建议

1. 为 Hook 示例补最小回归测试
- 增加样本输入（合法命令、违规命令、坏 JSON）并断言 exit code + stderr。

2. 为 Settings 示例增加 schema/字段校验脚本
- 在 CI 中跑 `jq` + JSON schema 或官方验证命令，避免示例字段过时。

3. 提升 Hook 规则表达能力
- 扩展正则以覆盖 `sudo` 前缀、命令链和子命令场景；并避免误报。

4. 增加“示例接入步骤”文档
- 在 `examples/settings/README.md` 增加从拷贝到验证的最短路径（放置位置、验证命令、回滚方式）。

5. 在根文档增加可发现性入口
- 在 `README.md` 或 `plugins/README.md` 增加一行指向 `examples/`，降低“存在但不易发现”的问题。
