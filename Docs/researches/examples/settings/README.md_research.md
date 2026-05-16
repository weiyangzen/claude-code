# FILE `examples/settings/README.md` 研究文档

## 场景与职责

`examples/settings/README.md` 是 `examples/settings/` 目录的策略说明入口，职责不是执行配置，而是帮助管理员在三份示例 JSON（`lax`/`strict`/`bash-sandbox`）中快速选型并避免误配。

它处于“文档控制面”：
- 上游调用方是人（平台管理员、项目维护者）。
- 下游被调用方是 Claude Code 的 settings 解析与策略执行运行时（仓库内未包含该运行时代码）。
- 仓库内没有脚本/测试直接读取本文件来驱动自动化行为。

## 功能点目的

1. 定义示例配置的适用范围与层级：说明可用于 settings hierarchy，同时提示某些字段仅 enterprise 生效（`strictKnownMarketplaces`、`allowManagedHooksOnly`、`allowManagedPermissionRulesOnly`）。
2. 用矩阵表达三份配置差异：禁用 bypass、marketplace 限制、权限规则托管、hooks 托管、Web 工具 deny、Bash 审批、Bash 沙箱强制。
3. 提供部署前安全提示：社区维护、需自行验证、先本地测试到 `managed-settings.json` / `settings.json` / `settings.local.json`。
4. 明确 `sandbox` 边界：仅作用于 `Bash`，不覆盖 Read/Write/WebSearch/WebFetch/MCP/hooks/内部命令。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 选型：操作者阅读矩阵行（能力维度）与列（lax/strict/bash-sandbox）。
2. 组合：按组织策略将示例 JSON 片段合并或直接采用。
3. 落地：写入目标层级文件（`managed-settings.json`、`settings.json` 或 `settings.local.json`）。
4. 生效：Claude Code 运行时加载有效 settings 并执行权限/marketplace/hooks/sandbox 策略。

### 关键数据结构

核心结构是 Markdown 表格（配置能力矩阵）：
- 行是安全与治理能力。
- 列是三个示例配置文件。
- 单元格 `✅` 表示该配置覆盖该能力。

该设计将“键级别细节”抽象为“策略能力级别”，减少只看 JSON 键名导致的误读。

### 关键协议与命令

- 协议：settings hierarchy（文档内链接到官方 settings docs）。
- 约束：部分键仅 enterprise 层可生效。
- 研究验证命令：
  - `rg -n "settings hierarchy|managed-settings.json|sandbox" examples/settings/README.md`
  - `rg -n "examples/settings/README.md" -S --glob '!Docs/researches/**'`

## 关键代码路径与文件引用

- `examples/settings/README.md:3-5`：目标受众、层级与 enterprise-only 提示。
- `examples/settings/README.md:13-21`：三份示例配置能力矩阵。
- `examples/settings/README.md:23-27`：部署前提示与 sandbox 边界。
- `examples/settings/settings-lax.json:1-6`：低侵入配置样例。
- `examples/settings/settings-strict.json:1-28`：强约束配置样例。
- `examples/settings/settings-bash-sandbox.json:1-18`：Bash 沙箱强制样例。
- `CHANGELOG.md:429`：`permissions.disableBypassPermissionsMode` 在 VSCode 侧的可见行为。
- `CHANGELOG.md:345`、`CHANGELOG.md:282`：`strictKnownMarketplaces` 语义演进与边界。

## 依赖与外部交互

1. 依赖的内部资产：同目录三份 JSON 示例文件。
2. 依赖的外部系统：Claude Code settings 解析与权限执行运行时。
3. 外部文档交互：
   - `https://code.claude.com/docs/en/settings#settings-files`
   - `https://code.claude.com/docs/en/settings`
4. 与测试/脚本关系：
   - 仓库内未发现 CI 或脚本直接消费 `examples/settings/README.md`。
   - `.ops/generate_daily_research_todo.sh` 只处理研究 checklist，不处理该 README 配置逻辑。

## 风险、边界与改进建议

1. 风险：文档与产品字段漂移。
- `strictKnownMarketplaces`、sandbox 网络字段在 changelog 中持续演进，矩阵可能落后于实际行为。

2. 风险：enterprise-only 字段被放在非 enterprise 层时“配置看似存在但不生效”。

3. 边界：该文件只负责“说明与选型”，不提供任何自动校验、合并或部署能力。

4. 改进建议：
- 在矩阵中新增“生效层级”列（enterprise/user/project）。
- 增加“最小验证步骤”小节（例如校验 `/permissions`、尝试 WebSearch/WebFetch、验证 Bash 沙箱行为）。
- 在 `examples/settings/` 增加轻量 schema 校验脚本，避免示例与字段演进脱节。
