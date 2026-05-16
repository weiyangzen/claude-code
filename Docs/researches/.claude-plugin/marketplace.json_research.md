# .claude-plugin/marketplace.json 研究

## 场景与职责

`.claude-plugin/marketplace.json` 是本仓库“插件市场清单（registry manifest）”的单一入口文件：

- 作为仓库内置插件集合的索引，声明插件名、描述、来源目录与分类（`.claude-plugin/marketplace.json:1-150`）。
- 通过 `plugins[].source` 将市场条目映射到真实插件目录 `./plugins/*`，驱动后续插件发现与组件装载（`.claude-plugin/marketplace.json:10-149`）。
- 通过 `$schema` 指向外部 schema，提供结构约束与工具侧校验入口（`.claude-plugin/marketplace.json:2`）。

从仓库文档定位看，它属于“插件分发层配置”，而不是插件业务逻辑本体：

- 仓库根 README 只声明插件能力在 `plugins/` 目录（`README.md:48-50`）。
- `plugins/README.md` 定义了插件能力目录及安装入口（`plugins/README.md:11-27,29-45`）。

## 功能点目的

1. 统一市场元数据出口
- 顶层 `name/version/description/owner` 描述该 marketplace 本身（`.claude-plugin/marketplace.json:3-9`）。

2. 插件可发现性与可安装性
- 每个 `plugins[]` 条目提供 `name + source + category`，使 `/plugin` 体系能够发现并展示插件（`.claude-plugin/marketplace.json:10-149`，`CHANGELOG.md:1624-1626`）。

3. 展示与筛选
- `description` 与 `category` 支持 UI/命令侧检索与分类（`.claude-plugin/marketplace.json:13-147`）。

4. 可选覆盖作者与版本
- 条目级 `version/author` 可覆盖或补充顶层 `owner`（例如 `agent-sdk-dev` 未声明条目级 `version/author`，其余大多声明）（`.claude-plugin/marketplace.json:11-16,18-148`）。

5. 与组织策略联动
- 市场来源会受 `strictKnownMarketplaces` 策略约束（`examples/settings/settings-strict.json:14`，`CHANGELOG.md:282,345`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构

`marketplace.json` 结构可分为两层：

- 顶层市场元数据：`$schema`, `name`, `version`, `description`, `owner`, `plugins`（`.claude-plugin/marketplace.json:2-10`）。
- 插件条目：`name`, `description`, `source`, `category`，可选 `version`, `author`（`.claude-plugin/marketplace.json:11-148`）。

当前仓库状态（基于文件内容）：

- 插件条目总数：13（`.claude-plugin/marketplace.json:10-149`）。
- 分类分布：development 6、productivity 4、learning 2、security 1。
- 唯一缺省 `version/author` 的条目：`agent-sdk-dev`（`.claude-plugin/marketplace.json:11-16`）。

### 2) 关键流程（调用链）

流程 A：插件安装/发现（运行时，调用方在 Claude Code 主程序，不在本仓库）

1. 用户通过 `/plugin install` 或 `/plugin marketplace` 进入插件管理流程（`plugins/README.md:43`，`CHANGELOG.md:1625`）。
2. 运行时读取 marketplace 条目并解析 source（`CHANGELOG.md:1113,171-173,231-233`）。
3. 读取插件目录下 `.claude-plugin/plugin.json` 完成插件识别（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`）。
4. 发现并注册 commands/agents/skills/hooks/MCP（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:12-16,23-27`）。

流程 B：插件发布/维护（仓库内维护链路）

1. `plugin-dev` 工作流在发布阶段提示“添加 marketplace entry”（`plugins/plugin-dev/commands/create-plugin.md:319-323`）。
2. 维护者同步更新 `plugins/README` 中插件列表与说明（`plugins/README.md:13-27`）。

流程 C：策略约束

1. 管理员可通过 `strictKnownMarketplaces` 限制市场来源（`examples/settings/settings-strict.json:14`，`examples/settings/README.md:5,16`）。
2. 该约束在历史版本不断增强（支持 pathPattern、owner/repo@ref 解析修复等）（`CHANGELOG.md:282,345`）。

### 3) 协议与命令面

- 市场协议：JSON + schema（`.claude-plugin/marketplace.json:2`）。
- 插件安装命令面：`/plugin install ...`、`/plugin marketplace ...`（`plugins/README.md:43`，`CHANGELOG.md:1625,281-282`）。
- 本仓库研究流水线命令面（非运行时、仅工程治理）：
  - `bash .ops/generate_daily_research_todo.sh` 依据 checklist 生成每日待办（`.ops/generate_daily_research_todo.sh:15-39`）。

### 4) 与下游目录结构耦合

`plugins[].source` 当前全部指向 `./plugins/*`（`.claude-plugin/marketplace.json:14,25,36,47,58,69,80,91,102,113,124,135,146`）。

按规范，下游插件目录应包含 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-48`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。

## 关键代码路径与文件引用

核心对象与上游文档：

- `.claude-plugin/marketplace.json:1-150`：市场清单主文件。
- `README.md:48-50`：仓库对插件子系统的入口说明。
- `plugins/README.md:11-27,43-45,49-71`：插件列表、安装方式、目录规范、贡献要求。

规范与发布流程依赖：

- `plugins/plugin-dev/commands/create-plugin.md:319-323`：发布阶段要求补充 marketplace 条目。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-48`：manifest 位置与结构规则。
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`：缺少 manifest 将不被识别。
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16,23-27`：运行时发现与激活流程。

策略与演进证据：

- `examples/settings/settings-strict.json:14`：`strictKnownMarketplaces` 配置样例。
- `examples/settings/README.md:5,16`：企业策略层面对 marketplace 的限制语义。
- `CHANGELOG.md:1113,151,171-173,231-233,281-282,308,345,520,668,956,966,1272,1547,1624-1626`：市场相关功能/修复的演进轨迹。

研究自动化链路（工程治理上下文）：

- `Docs/researches/blueprint_checklist.md:112`：当前文件研究项。
- `.ops/generate_daily_research_todo.sh:15-39`：按 checklist 生成今日 TODO。
- `.ops/generate_research_blueprint_checklist.sh:10-66`：重建 checklist 并保留已完成状态。

## 依赖与外部交互

1. 外部 schema 依赖
- `$schema` 指向 `https://anthropic.com/claude-code/marketplace.schema.json`，若离线或 URL 变更，编辑器/校验工具可能失去 schema 提示（`.claude-plugin/marketplace.json:2`）。

2. Claude Code 运行时依赖（仓库外）
- `/plugin` 子系统负责市场读取、下载/更新、路径解析、安装状态同步；相关行为通过 CHANGELOG 可见（`CHANGELOG.md:171-173,231-233,281-282,308,1113`）。

3. 仓库内下游文件依赖
- 每个 `source` 条目依赖插件目录结构完整（manifest、commands/agents/skills/hooks 等）。
- 清单内容与 `plugins/README` 人工同步，存在双写维护（`.claude-plugin/marketplace.json:10-149` vs `plugins/README.md:13-27`）。

4. 配置侧外部交互
- `strictKnownMarketplaces` 会对可接受 market source 形成硬约束（`examples/settings/settings-strict.json:14`，`CHANGELOG.md:345`）。

5. 测试/脚本现状
- 仓库内未发现针对 `marketplace.json` 的专门测试文件（`*test*`/`*spec*` 未命中）。
- 未发现解析该文件的仓库内业务脚本；更多依赖 Claude Code 外部运行时实现。

## 风险、边界与改进建议

### 已识别风险

1. 元数据漂移风险（marketplace 与 plugin manifest 不一致）
- `feature-dev` 作者名在 marketplace 与 plugin manifest 拼写不同：`Siddharth Bidasaria` vs `Sid Bidasaria`（`.claude-plugin/marketplace.json:66`，`plugins/feature-dev/.claude-plugin/plugin.json:6`）。
- `frontend-design` 作者与邮箱在两处表达方式不同（`.claude-plugin/marketplace.json:77-79`，`plugins/frontend-design/.claude-plugin/plugin.json:6-7`）。
- `pr-review-toolkit` 在 marketplace 作者为 `Anthropic`，manifest 为 `Daisy`（`.claude-plugin/marketplace.json:121-123`，`plugins/pr-review-toolkit/.claude-plugin/plugin.json:6-7`）。

2. 条目完整性风险
- `agent-sdk-dev` 条目缺省 `version` 与 `author`，依赖顶层 owner 或运行时容错（`.claude-plugin/marketplace.json:11-16`）。

3. 下游结构一致性风险
- `plugins/plugin-dev` 被 marketplace 注册（`.claude-plugin/marketplace.json:106-114`），但目录中缺少 `.claude-plugin/plugin.json`（基于目录核查）。
- `plugins/security-guidance` 缺少 README（与 `plugins/README.md:65-71` 贡献约定不一致，基于目录核查）。

4. 运行时不可见风险
- 本仓库缺少 marketplace 清单的自动验证脚本与 CI 断言，错误通常在 `/plugin` 运行时才暴露。

### 边界说明

- 本文件仅定义“市场清单”，不承载插件执行逻辑。真正的安装、同步、冲突处理、git source 解析在 Claude Code 主程序内部完成（从 CHANGELOG 可见，但代码不在本仓库）。
- `source` 当前均为仓库相对路径，适配“同仓库打包/开发”场景；对跨仓库分发、远端市场镜像的可移植性依赖外部工具链。

### 改进建议（按优先级）

1. 增加一个仓库内校验脚本（建议放 `.ops/validate_marketplace.sh`）并接入 CI，至少检查：
- JSON 可解析；
- `plugins[].name` 唯一；
- `source` 目录存在；
- `source/.claude-plugin/plugin.json` 存在；
- marketplace 与 plugin manifest 的 `name/version` 一致性；
- README 存在性（按团队规范可设为 warning/error）。

2. 建立“单一元数据来源”策略
- 方案 A：marketplace 仅保留 `source` 与市场特有字段，其他展示字段从下游 manifest 聚合生成。
- 方案 B：反向生成 marketplace 条目，避免人工双写。

3. 标准化作者与版本字段
- 明确 `owner` 与 `author` 的继承/覆盖规则，减少展示与溯源歧义。

4. 在 `plugin-dev` 发布流程中加入自动检查
- 在 `create-plugin` 的“Phase 8”后追加校验步骤，自动提示 manifest/README 缺失与元数据不一致（`plugins/plugin-dev/commands/create-plugin.md:319-323`）。
