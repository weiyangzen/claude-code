# FILE `examples/settings/settings-strict.json` 研究文档

## 场景与职责

`settings-strict.json` 是 `examples/settings` 中覆盖最全面的“强治理模板”，目标是为组织级默认策略提供可直接落地的高约束基线。

职责覆盖四层：
1. 权限层：禁用 bypass、Bash 走审批、deny 外网搜索抓取工具。
2. 管控层：仅允许受管控权限规则与受管控 hooks。
3. 生态层：限制 marketplace 来源。
4. 执行层：收紧 Bash sandbox 的网络与嵌套行为。

## 功能点目的

1. `permissions.disableBypassPermissionsMode: "disable"`
- 阻止切换到绕过权限模式。

2. `permissions.ask: ["Bash"]`
- 所有 Bash 请求进入审批流程。

3. `permissions.deny: ["WebSearch", "WebFetch"]`
- 阻断默认联网搜索/抓取工具。

4. `allowManagedPermissionRulesOnly: true`
- 禁止用户/项目层随意放宽权限规则。

5. `allowManagedHooksOnly: true`
- 禁止用户/项目层自定义 hooks 生效。

6. `strictKnownMarketplaces: []`
- 对 plugin marketplace 来源施加严格限制。

7. `sandbox.*`
- 关闭 `autoAllowBashIfSandboxed`，保留 Bash 审批。
- 默认不放行 Unix socket、本地绑定、允许域名、代理端口。
- 禁用弱化嵌套沙箱。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 管理员将该 JSON 放入生效层级。
2. Claude Code 启动时合并 settings 并计算有效策略。
3. 工具调用时按链路执行：
- 权限规则匹配（ask/deny/bypass）。
- hooks 来源合法性校验（managed-only）。
- marketplace 来源限制。
- Bash 执行进入 sandbox 约束。

### 关键数据结构

该文件是“组合策略对象”：
- `permissions` 对象（`disableBypassPermissionsMode` + `ask` + `deny`）
- 三个治理开关（`allowManagedPermissionRulesOnly`、`allowManagedHooksOnly`、`strictKnownMarketplaces`）
- `sandbox` 对象（执行隔离与网络边界）

与 `lax` 相比，`strict` 是跨权限/插件生态/执行环境三域联动的配置，而不是单一键位收紧。

### 关键协议与命令

- 协议：Claude Code settings hierarchy（enterprise/user/project）。
- changelog 关键语义：
  - `disableBypassPermissionsMode` 对 VSCode 权限 UI 的直接影响（`CHANGELOG.md:429`）。
  - `strictKnownMarketplaces` 新增 `pathPattern` 与解析修复（`CHANGELOG.md:345`、`CHANGELOG.md:282`）。
  - `sandbox.excludedCommands`/`autoAllowBashIfSandboxed` 历史绕过修复（`CHANGELOG.md:251`、`CHANGELOG.md:756`）。
- 研究验证命令：
  - `jq -e examples/settings/settings-strict.json`
  - `git log --oneline -- examples/settings/settings-strict.json`
  - `git show 90c07d1 -- examples/settings/settings-strict.json`
  - `git show 4936302 -- examples/settings/settings-strict.json`

## 关键代码路径与文件引用

- `examples/settings/settings-strict.json:2-27`：完整策略定义。
- `examples/settings/README.md:15-20`：该文件在能力矩阵中的覆盖声明。
- `examples/settings/README.md:5`：enterprise-only 字段提示。
- `CHANGELOG.md:429`：bypass 模式 UI 行为。
- `CHANGELOG.md:345`、`CHANGELOG.md:282`：marketplace 规则演进。
- `CHANGELOG.md:251`、`CHANGELOG.md:756`：sandbox 相关安全修复。
- `git show 90c07d1 -- examples/settings/settings-strict.json`：首次引入 sandbox 子树。
- `git show 4936302 -- examples/settings/settings-strict.json`：补充 `allowAllUnixSockets` 与 `allowedDomains`。

## 依赖与外部交互

1. 依赖：权限引擎、hooks 管理策略、marketplace 来源校验、Bash sandbox 模块。
2. 外部交互：
- 通过 deny Web 工具影响外网访问能力。
- 通过 sandbox 网络字段限制 Bash 网络行为。
3. 文档依赖：
- 由 `examples/settings/README.md` 提供选型与边界说明。
4. 测试与脚本：
- 仓库无该文件的专门自动化回归；当前验证主要是 JSON 语法和人工策略验收。

## 风险、边界与改进建议

1. 风险：策略过严可能影响研发效率。
- 典型表现是 Bash 频繁审批、Web 工具完全不可用、某些联网命令被沙箱网络限制阻断。

2. 风险：enterprise-only 字段在非 enterprise 层无效，易形成“误以为已加固”。

3. 风险：`excludedCommands` 如果后续被填充不当，可能重新打开绕过路径。

4. 边界：即便 strict 已较全面，README 已声明 sandbox 仍只作用于 Bash，不自动覆盖其他工具。

5. 改进建议：
- 提供 strict 的“例外配置手册”（按团队角色放开最小权限）。
- 增加策略生效自检脚本（校验 ask/deny/managed-only/sandbox 关键键存在且类型正确）。
- 把 changelog 中涉及上述关键键的更新纳入例行审阅流程。
