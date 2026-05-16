# FILE `examples/settings/settings-bash-sandbox.json` 研究文档

## 场景与职责

`settings-bash-sandbox.json` 是一个“执行隔离优先”的策略样例：核心职责是强制 Bash 在沙箱内执行，并关闭绕过路径，不追求像 `settings-strict.json` 那样覆盖更广的权限与工具管控。

定位：
- 调用方：管理员将其内容应用到 managed/user/project settings 文件。
- 被调用方：Claude Code 的 Bash 执行沙箱与权限策略引擎。
- 仓库内无脚本直接读取该文件，它是外部运行时消费的配置输入。

## 功能点目的

1. `allowManagedPermissionRulesOnly: true`：限制权限规则来源到受管控策略。
2. `sandbox.enabled: true`：显式启用 Bash 沙箱。
3. `sandbox.allowUnsandboxedCommands: false`：关闭非沙箱执行逃逸通道。
4. `sandbox.autoAllowBashIfSandboxed: false`：即使在沙箱内也不自动批准 Bash，保留审批链路。
5. `sandbox.network.*` 与 `enableWeakerNestedSandbox: false`：维持保守网络与嵌套隔离策略。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 将 JSON 放入目标 settings 层级文件。
2. Claude Code 在解析配置后，把 `sandbox` 子树绑定到 Bash 工具执行路径。
3. Bash 工具请求到达时：
- 先进入沙箱策略判断（是否允许非沙箱、是否自动放行、网络约束等）。
- 再由权限系统决定是否需要用户审批。

### 关键数据结构

顶层对象包含：
- `allowManagedPermissionRulesOnly: boolean`
- `sandbox: object`

`sandbox` 结构包含：
- `enabled: boolean`
- `autoAllowBashIfSandboxed: boolean`
- `allowUnsandboxedCommands: boolean`
- `excludedCommands: string[]`
- `network: { allowUnixSockets, allowAllUnixSockets, allowLocalBinding, allowedDomains, httpProxyPort, socksProxyPort }`
- `enableWeakerNestedSandbox: boolean`

### 关键协议与命令

- 协议：Claude Code settings JSON（外部运行时解析）。
- 重要语义演进（changelog 证据）：
  - `allowUnsandboxedCommands` 是后续新增的策略开关（`CHANGELOG.md:1523`）。
  - `autoAllowBashIfSandboxed` 与 `excludedCommands` 组合曾出现权限绕过修复（`CHANGELOG.md:756`、`CHANGELOG.md:251`）。
- 研究验证命令：
  - `jq -e examples/settings/settings-bash-sandbox.json`
  - `git log --oneline -- examples/settings/settings-bash-sandbox.json`

## 关键代码路径与文件引用

- `examples/settings/settings-bash-sandbox.json:2-17`：完整策略定义。
- `examples/settings/README.md:13-21`：该文件在能力矩阵中的定位（“Bash 必须在沙箱中执行”）。
- `examples/settings/README.md:27`：`sandbox` 仅作用于 Bash 的边界声明。
- `CHANGELOG.md:1523`：`allowUnsandboxedCommands` 引入背景。
- `CHANGELOG.md:756`：`autoAllowBashIfSandboxed` 相关绕过修复。
- `CHANGELOG.md:251`：`sandbox.excludedCommands` 规则匹配修复。
- `git show 43d0eac -- examples/settings/settings-bash-sandbox.json`：补充 `allowAllUnixSockets`、`allowedDomains` 的演进证据。

## 依赖与外部交互

1. 依赖项：Claude Code settings 解析器、Bash 沙箱执行模块、权限审批模块。
2. 外部交互：
- 网络行为由 `sandbox.network` 定义（默认本示例是最保守空白策略）。
- 本文件自身不发起网络请求。
3. 与文档/脚本关系：
- 由 `examples/settings/README.md` 描述如何落地。
- 无仓库内测试或自动脚本直接验证其语义，仅可做 JSON 语法验证。

## 风险、边界与改进建议

1. 风险：仅覆盖 Bash，不覆盖 Read/Write/WebSearch/WebFetch/MCP/hook 行为。
2. 风险：默认网络限制较严，可能导致依赖内网代理、Unix socket 或域名访问的命令失败。
3. 风险：`excludedCommands` 误配置可能形成策略缺口。
4. 边界：该样例不包含 `permissions.ask`/`permissions.deny`，因此其“强制性”主要在执行隔离层而非工具访问层。
5. 改进建议：
- 在示例旁提供“常见例外模板”（如仅允许特定域名/socket）。
- 增加与 `settings-strict.json` 的组合示例，说明何时要同时启用 deny Web 工具与 hooks 托管。
- 引入 CI 级 schema 与字段漂移检查，及时发现 changelog 更新后的样例落后问题。
