# DIR `examples/settings` 研究文档

## 场景与职责

`examples/settings` 是 Claude Code Settings 的“策略示例目录”，定位是给组织管理员和项目维护者提供可直接拷贝/组合的 JSON 配置模板，而不是仓库运行时代码。

目录内仅 4 个文件：
- `examples/settings/README.md`
- `examples/settings/settings-lax.json`
- `examples/settings/settings-strict.json`
- `examples/settings/settings-bash-sandbox.json`

职责边界：
1. 提供三档策略样例（lax/strict/bash-sandbox）。
2. 用 README 的功能矩阵解释差异和适用范围。
3. 作为“配置输入样例”被外部 Claude Code 设置层级读取（`managed-settings.json` / `settings.json` / `settings.local.json`），而不是被本仓库脚本直接执行。

调用链（基于仓库内证据）：
1. 调用方
- 人类操作者（管理员/开发者）手工选择并复制该目录 JSON 到实际 settings 文件。
- 证据：`examples/settings/README.md` 明确写了应用目标文件和 settings hierarchy。
2. 被调用方
- Claude Code 运行时的 settings 解析与策略执行模块（仓库中未内置其实现代码）。
3. 仓库内耦合
- 对 `.github/`、`scripts/`、`.ops/` 做全文检索未发现直接消费 `examples/settings` 的自动化链路。

## 功能点目的

### 1. `settings-lax.json`：低侵入基础治理
目标：最小限度收紧高风险入口，减少对日常操作的干扰。

核心策略：
- `permissions.disableBypassPermissionsMode = "disable"`：禁用绕过权限模式。
- `strictKnownMarketplaces = []`：按 README 矩阵意图，阻断 plugin marketplace 来源。

### 2. `settings-strict.json`：组织级强约束
目标：在权限、hooks、联网工具、sandbox 上同步收紧，形成更强的企业策略基线。

核心策略：
- `permissions.ask = ["Bash"]`：Bash 默认走审批。
- `permissions.deny = ["WebSearch", "WebFetch"]`：直接禁用外网检索/抓取工具。
- `allowManagedPermissionRulesOnly = true`：只允许受管控的权限规则。
- `allowManagedHooksOnly = true`：只允许受管控 hooks。
- `strictKnownMarketplaces = []`：限制 marketplace。
- `sandbox.*`：对 Bash 的网络、本地绑定、unix socket、代理口、嵌套沙箱行为进行限制。

### 3. `settings-bash-sandbox.json`：Bash 必须入沙箱
目标：把重点放在 Bash 执行隔离，阻断“临时逃逸到非沙箱执行”的路径。

核心策略：
- `allowManagedPermissionRulesOnly = true`
- `sandbox.enabled = true`
- `sandbox.allowUnsandboxedCommands = false`
- `sandbox.autoAllowBashIfSandboxed = false`
- `sandbox.network.*` 为空白/关闭策略

### 4. `README.md`：策略选择与风险提示入口
目标：用表格快速比较三份 JSON，避免直接阅读原始键值导致误选。

关键提示：
- 示例为 community-maintained，可能不完全正确。
- `sandbox` 只作用于 Bash，不作用于 Read/Write/WebSearch/WebFetch/MCP/hook/内部命令。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

流程 1：模板选择
1. 读取 `examples/settings/README.md` 的功能矩阵。
2. 选择 lax / strict / bash-sandbox 或组合片段。

流程 2：配置落地
1. 将 JSON 应用到目标层级文件（`managed-settings.json`、`settings.json` 或 `settings.local.json`）。
2. Claude Code 启动或配置变更后由 settings 层级系统解析。

流程 3：策略生效
1. 权限层：Bash 的 ask/deny/bypass 受 `permissions.*` 控制。
2. hooks 层：`allowManagedHooksOnly` 决定是否允许用户/项目 hooks。
3. 工具层：`permissions.deny` 可阻断 WebSearch/WebFetch。
4. 执行层：`sandbox.*` 约束 Bash 的隔离与网络。

### B. 关键数据结构（按文件）

`settings-lax.json`
```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  },
  "strictKnownMarketplaces": []
}
```

`settings-strict.json`
```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "ask": ["Bash"],
    "deny": ["WebSearch", "WebFetch"]
  },
  "allowManagedPermissionRulesOnly": true,
  "allowManagedHooksOnly": true,
  "strictKnownMarketplaces": [],
  "sandbox": {
    "autoAllowBashIfSandboxed": false,
    "excludedCommands": [],
    "network": {
      "allowUnixSockets": [],
      "allowAllUnixSockets": false,
      "allowLocalBinding": false,
      "allowedDomains": [],
      "httpProxyPort": null,
      "socksProxyPort": null
    },
    "enableWeakerNestedSandbox": false
  }
}
```

`settings-bash-sandbox.json`
```json
{
  "allowManagedPermissionRulesOnly": true,
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": false,
    "allowUnsandboxedCommands": false,
    "excludedCommands": [],
    "network": {
      "allowUnixSockets": [],
      "allowAllUnixSockets": false,
      "allowLocalBinding": false,
      "allowedDomains": [],
      "httpProxyPort": null,
      "socksProxyPort": null
    },
    "enableWeakerNestedSandbox": false
  }
}
```

### C. 协议与语义对照

1. settings hierarchy 协议
- README 指向官方 settings hierarchy，说明这些 JSON 可位于多个层级。
- 但 README 同时强调部分字段仅在 enterprise settings 有效（如 `allowManagedHooksOnly`）。

2. 权限与沙箱语义
- `permissions.disableBypassPermissionsMode` 的效果在 CHANGELOG 中可见：VSCode 权限模式选择器会受该字段约束。
- `allowUnsandboxedCommands`、`autoAllowBashIfSandboxed` 在 CHANGELOG 中有修复与行为说明，表明这两项属于高风险策略开关。

3. marketplace 约束语义
- `strictKnownMarketplaces` 在 CHANGELOG 多次出现修复/增强（如 `pathPattern`），说明该策略会直接影响 marketplace source 解析与限制。

### D. 本次研究执行过的关键命令

1. 结构与引用追踪
- `rg --files examples/settings`
- `rg -n "examples/settings|settings-lax.json|settings-strict.json|settings-bash-sandbox.json" -S --hidden --glob '!.git'`

2. 配置有效性
- `for f in examples/settings/*.json; do jq -e . "$f" >/dev/null; done`
- 结果：三份 JSON 均为 valid JSON。

3. 演进历史
- `git log --oneline -- examples/settings`
- `git show f93f614/90c07d1/43d0eac/4936302 -- examples/settings/...`

## 关键代码路径与文件引用

目标目录：
- `examples/settings/README.md`
- `examples/settings/settings-lax.json`
- `examples/settings/settings-strict.json`
- `examples/settings/settings-bash-sandbox.json`

直接上下文（文档/变更）：
- `CHANGELOG.md`（关键字段语义与修复记录，如 `strictKnownMarketplaces`、`allowUnsandboxedCommands`、`autoAllowBashIfSandboxed`、`disableBypassPermissionsMode`）
- `Docs/researches/blueprint_checklist.md`（研究任务清单，当前条目标注来源）
- `.ops/generate_daily_research_todo.sh`（将 checklist 生成当日待办）

调用方/被调用方关系：
1. 调用方
- 人类操作者把示例 JSON 放入实际配置文件。
2. 被调用方
- Claude Code settings 解析器与运行时权限/沙箱/marketplace/hook 策略模块。
3. 仓库内非调用关系（重要）
- `.github/workflows/*`、`scripts/*`、`.ops/*` 中未发现直接读取 `examples/settings/*` 的逻辑，说明它是“示例资产”而非“自动执行资产”。

测试与脚本关系：
1. `examples/settings` 目录内无测试脚本。
2. 仓库也没有针对该目录的 CI 校验（未检索到 workflow 或 script 对该目录的消费）。
3. 当前可执行的最小验证只有 JSON 语法检查（如 `jq -e`）与人工策略验收。

## 依赖与外部交互

本地依赖：
1. 文件格式依赖：JSON。
2. 验证工具依赖（可选）：`jq`。
3. 研究流程依赖：`rg`、`git`、`bash`。

运行时依赖：
1. Claude Code settings hierarchy 机制（外部运行时）。
2. Claude Code 对权限、hooks、sandbox、marketplace 的策略解释器（外部运行时）。

外部交互：
1. README 指向官方文档 `https://code.claude.com/docs/en/settings`。
2. 示例文件本身不发起网络请求；真正网络边界由被应用后的 `permissions.deny` 与 `sandbox.network` 决定。

## 风险、边界与改进建议

风险：
1. 示例与产品字段演进不同步风险。
- 证据：该目录在 2026-01-30 创建后，2026-02-01 立即补过 network 字段（`allowAllUnixSockets`、`allowedDomains`），说明 schema 变化节奏较快。
2. `strictKnownMarketplaces: []` 可读性风险。
- 仅看 JSON 不直观，实际语义依赖 README 表格和产品定义，容易被误配。
3. 缺乏自动化回归。
- 没有 CI 对该目录做 schema/语义校验，改动后可能静默失效。
4. 误用层级风险。
- README 已提示某些字段仅 enterprise 生效；若放在 project/user 层，可能“看起来配置了但不生效”。

边界：
1. 该目录不实现 settings 解析逻辑，只提供示例输入。
2. 该目录不负责 hooks 脚本实现；只通过配置字段约束 hooks 是否允许。
3. 该目录不参与仓库自动化主链，不提供可执行服务或 CLI。

改进建议：
1. 增加示例配置自动校验脚本。
- 在 `.ops` 或 `scripts` 增加对 `examples/settings/*.json` 的结构检查（至少 `jq -e` + 必填字段断言）。
2. 在 README 增加“字段生效层级”对照列。
- 明确哪些键只在 enterprise 生效，减少误配。
3. 增加“最小验证流程”章节。
- 示例可给出：落地文件位置 -> 启动验证步骤 -> 观察点（permissions/hook/sandbox 是否按预期生效）。
4. 增加与 CHANGELOG 的同步机制。
- 当 `sandbox.*`/`strictKnownMarketplaces` 等字段发生变更时，触发对示例 JSON 的审阅任务。
