# FILE `examples/settings/settings-lax.json` 研究文档

## 场景与职责

`settings-lax.json` 是三档策略中最轻量的一档，职责是提供“低侵入但有底线”的默认治理起点：限制高风险开关，不显著影响日常开发流程。

调用关系：
- 调用方：组织管理员/项目维护者手工应用此 JSON。
- 被调用方：Claude Code 权限模式与 marketplace 限制逻辑。
- 仓库内未发现自动调用它的脚本/测试。

## 功能点目的

1. `permissions.disableBypassPermissionsMode = "disable"`
- 禁用 `--dangerously-skip-permissions` 相关绕过模式，防止会话被切换到“完全跳过权限门控”。

2. `strictKnownMarketplaces = []`
- 按目录 README 的矩阵定义，用于阻断或严格约束 plugin marketplace 来源。

该文件明确不做的事情：
- 不要求 Bash `ask` 审批。
- 不 deny `WebSearch` / `WebFetch`。
- 不启用 sandbox 配置。
- 不限制 hooks 来源。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 配置写入 settings hierarchy 目标文件。
2. 运行时读取 `permissions.disableBypassPermissionsMode`，影响权限模式可选项。
3. 运行时读取 `strictKnownMarketplaces`，影响 marketplace 来源解析与准入。

### 关键数据结构

这是一个两字段最小对象：
- `permissions`（子字段仅 `disableBypassPermissionsMode`）
- `strictKnownMarketplaces`（数组）

它刻意保持体积小（6 行），便于作为“组织策略最小基线”向更严格配置演进。

### 关键协议与命令

- 协议：managed settings JSON。
- changelog 行为证据：
  - `CHANGELOG.md:429`：当该字段为 `disable` 时，VSCode 权限模式选择器隐藏 bypass。
  - `CHANGELOG.md:345`、`CHANGELOG.md:282`：`strictKnownMarketplaces` 解析规则和边界持续演进。
- 研究验证命令：
  - `jq -e examples/settings/settings-lax.json`
  - `rg -n "disableBypassPermissionsMode|strictKnownMarketplaces" examples/settings/settings-lax.json CHANGELOG.md`

## 关键代码路径与文件引用

- `examples/settings/settings-lax.json:2-5`：两项策略键定义。
- `examples/settings/README.md:15-16`：矩阵声明其覆盖“禁用 bypass、限制 marketplace”。
- `examples/settings/README.md:5`：enterprise-only 字段提示（该文件涉及 `strictKnownMarketplaces`）。
- `CHANGELOG.md:429`：`disableBypassPermissionsMode` 的可观测效果。
- `CHANGELOG.md:345`、`CHANGELOG.md:282`：`strictKnownMarketplaces` 的语义变化背景。
- `git show f93f614 -- examples/settings/settings-lax.json`：该文件初始引入记录。

## 依赖与外部交互

1. 依赖：Claude Code 权限系统与 marketplace 策略解析器。
2. 外部交互：
- 主要影响插件市场操作路径，不直接触发网络请求。
3. 文档交互：
- 由 `examples/settings/README.md` 提供部署指引。
4. 测试交互：
- 仓库未提供专门测试，当前仅可做 JSON 语法检查与人工验证。

## 风险、边界与改进建议

1. 风险：安全覆盖面有限。
- 未限制 Web 工具、hooks、自定义权限规则与 Bash 沙箱。

2. 风险：`strictKnownMarketplaces: []` 的语义不直观，若脱离 README 易被误读。

3. 边界：此文件是“最小基线”，不适合直接作为高合规企业策略终态。

4. 改进建议：
- 在文件旁增加注释型 companion 文档，解释 `[]` 的策略含义与常见误配。
- 给出从 `lax -> strict` 的渐进升级步骤，避免一次性切换造成业务中断。
- 结合 changelog 变更自动提醒审阅 `strictKnownMarketplaces` 相关样例。
