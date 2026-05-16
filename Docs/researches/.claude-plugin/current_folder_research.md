# DIR `.claude-plugin` 研究文档

## 场景与职责

`.claude-plugin` 在仓库根目录下的职责不是“单个插件 manifest”，而是“插件市场清单（marketplace catalog）”：

- 目录内仅有一个文件：`.claude-plugin/marketplace.json`。
- 该文件声明本仓库打包分发的插件集合，`plugins[].source` 指向 `./plugins/*`。
- 它与各插件自身的 `.claude-plugin/plugin.json` 分工不同：
  - 根目录 `.claude-plugin/marketplace.json`：市场索引层（列出有哪些插件可装）。
  - 各插件目录 `.claude-plugin/plugin.json`：插件定义层（插件自身元数据与自动发现入口）。

对应场景：

1. 插件发现与安装入口：通过 `/plugin install ...@<marketplace>` 使用市场索引安装插件。  
2. 插件发布/维护入口：新增插件时需在该索引里追加条目。  
3. 仓库资产组织入口：把 `plugins/` 下多个官方插件统一编排为一个市场。

## 功能点目的

### 1) 统一插件目录编排

`marketplace.json` 当前包含 13 个插件条目，按 `category` 分为：

- `development`: 6
- `productivity`: 4
- `learning`: 2
- `security`: 1

目的：让同一仓库内不同能力类型（命令、agent、skill、hook）的插件可被统一发现与分发。

### 2) 提供市场级元数据

顶层字段（`name/version/description/owner`）用于表达“市场”身份，而非某个插件身份。  
每个 `plugins[]` 条目提供：

- `name`：插件标识
- `description`：市场列表显示说明
- `source`：插件源目录（相对路径）
- `category`：分类
- 可选 `version`、`author`

目的：支持安装界面展示、分类筛选与可读性。

### 3) 解耦市场条目与插件内部实现

根市场文件不关心插件内部具体是 `commands/agents/skills/hooks` 哪一种；只负责把 source 指向插件根目录。  
插件内部由各自 `plugin.json` 和目录结构完成自动发现。

目的：市场层保持轻量，插件层自由演进。

### 4) 支撑 plugin-dev 工作流中的“发布步骤”

`plugins/plugin-dev/commands/create-plugin.md` 的 Phase 8 明确要求“若发布，则添加 marketplace entry”。  
目的：把“创建插件”与“加入市场”串成标准流程。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 数据结构：`marketplace.json`

根文件结构如下（缩略）：

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "claude-code-plugins",
  "version": "1.0.0",
  "owner": { "name": "...", "email": "..." },
  "plugins": [
    {
      "name": "feature-dev",
      "source": "./plugins/feature-dev",
      "category": "development",
      "version": "1.0.0",
      "author": { "name": "...", "email": "..." }
    }
  ]
}
```

关键约束（由仓库内约定和文档共同体现）：

- `source` 使用相对路径并落到仓库内插件目录。
- 实际可用插件还应在该目录下存在 `.claude-plugin/plugin.json`（插件结构规范中的硬性要求）。

### B. 运行时关键流程（结合文档与变更记录）

1. Claude Code 插件系统读取 marketplace（`/plugin marketplace ...`）。  
2. 用户通过 `/plugin install <plugin>@<marketplace>` 安装。  
3. 安装后插件加载其自身 `.claude-plugin/plugin.json`，并自动发现 `commands/agents/skills/hooks`。  

说明：该仓库中没有 CLI 内核代码；上述流程依据 `plugins/README.md`、`plugin-dev` 技能文档与 `CHANGELOG.md` 的插件系统演进记录归纳。

### C. 仓库内维护流程（作者侧）

1. 在 `plugins/<name>/` 实现插件内容。  
2. 添加或更新 `plugins/<name>/.claude-plugin/plugin.json`。  
3. 在根 `.claude-plugin/marketplace.json` 增加条目，设置 `source/category/description`。  
4. 通过 `claude plugin validate`（变更日志提及）与人工审查确保可用。  

### D. 关键命令与协议片段

- 安装：`/plugin install plugin-dev@claude-code-marketplace`（`plugins/plugin-dev/README.md`）。  
- 本地开发绕过市场：`cc --plugin-dir /path/to/plugin-dev`。  
- 市场维护相关命令在变更日志中可见：`/plugin marketplace add ...`、`/plugin marketplace update`。  
- 市场 schema：`$schema = https://anthropic.com/claude-code/marketplace.schema.json`。

### E. 实际一致性核对结果（基于仓库现状）

对 `marketplace.json` 的 13 个 `source` 做落盘核对后发现：

1. `plugins/plugin-dev` 被市场收录，但目录内缺少 `.claude-plugin/plugin.json`。  
2. `plugins/agent-sdk-dev` 在市场条目未写 `version`，但其 `plugin.json` 有 `1.0.0`。  
3. 其余已核对条目 `name/version` 基本与各插件 `plugin.json` 一致。

## 关键代码路径与文件引用

### 目标对象

- `.claude-plugin/marketplace.json`

### 上下文调用方（消费或约束该对象）

- `plugins/README.md`（说明插件来自市场安装）
- `plugins/plugin-dev/README.md`（给出 `/plugin install ...@claude-code-marketplace`）
- `CHANGELOG.md`（`/plugin`、marketplace、validate、路径解析等演进记录）
- `plugins/plugin-dev/commands/create-plugin.md`（发布阶段要求新增 marketplace entry）

### 被调用方（由该对象指向的插件实体）

- `plugins/agent-sdk-dev`
- `plugins/claude-opus-4-5-migration`
- `plugins/code-review`
- `plugins/commit-commands`
- `plugins/explanatory-output-style`
- `plugins/feature-dev`
- `plugins/frontend-design`
- `plugins/hookify`
- `plugins/learning-output-style`
- `plugins/plugin-dev`
- `plugins/pr-review-toolkit`
- `plugins/ralph-wiggum`
- `plugins/security-guidance`

以及其插件 manifest（预期路径）：

- `plugins/*/.claude-plugin/plugin.json`

### 相关脚本/流程文件

- `.ops/generate_research_blueprint_checklist.sh`（研究蓝图生成，含 `.claude-plugin` 条目）
- `.ops/generate_daily_research_todo.sh`（按 checklist 生成当日 todo）
- `.ops/research_guard.sh`（自动选取 pending 项并下发研究任务）

### 测试与验证现状

- 仓库中未发现针对根 `.claude-plugin/marketplace.json` 的专门自动化测试或 lint 脚本。
- `CHANGELOG.md` 显示存在 `claude plugin validate` 能力，但仓库未提供针对该文件的 CI 规则。

## 依赖与外部交互

### 直接依赖

- JSON schema 外链：`https://anthropic.com/claude-code/marketplace.schema.json`
- 本地目录依赖：`plugins/*`（由 `source` 字段引用）

### 工具链与命令依赖（运行/维护视角）

- Claude Code `/plugin` 命令族（安装、市场更新、验证）
- 本地开发命令：`cc --plugin-dir`
- 建议校验工具：`jq`（仓库维护中可用于字段一致性检查）

### 外部交互

- marketplace 更新/安装涉及 git 远程交互（变更日志多次提及 marketplace clone/update/path 处理问题）。
- 插件系统由 Claude Code 客户端实现，本仓库主要提供市场数据与插件资产。

## 风险、边界与改进建议

### 风险

1. **市场条目与插件 manifest 不一致风险（高）**  
- 证据：`plugins/plugin-dev` 已在市场中登记，但缺少 `.claude-plugin/plugin.json`。  
- 影响：按插件结构规范，该插件可能无法被标准识别链路正确加载。

2. **元数据漂移风险（中）**  
- 证据：`agent-sdk-dev` 市场条目无 `version`，但插件 manifest 有版本。  
- 影响：市场展示与版本管理不一致，发布追踪困难。

3. **缺少自动化校验风险（中）**  
- 现状：无 CI 自动验证 `source` 存在性、`plugin.json` 存在性、字段一致性。  
- 影响：回归只能在安装或运行期暴露。

4. **远程 schema 变更风险（低-中）**  
- 现状：`$schema` 指向远程 URL。  
- 影响：若 schema 演进而仓库条目未同步，可能出现兼容性问题。

### 边界

- 根 `.claude-plugin` 只负责市场索引，不承担插件内部逻辑实现。
- 插件实际行为由 `plugins/*` 内容决定（commands/agents/skills/hooks）。
- 本仓库不包含 Claude Code 插件管理内核，因此“安装/更新/解析”的代码路径主要在外部产品中。

### 改进建议

1. 增加 CI 校验（建议新建 `.ops/validate_marketplace.sh` + workflow）  
- 校验点：`source` 目录存在、`source/.claude-plugin/plugin.json` 存在、`name/version` 一致性、category 白名单。

2. 统一版本策略  
- 要求市场条目显式填写 `version`，避免展示与追踪歧义。

3. 降低重复元数据漂移  
- 可考虑从 `plugins/*/.claude-plugin/plugin.json` 生成市场条目的 name/version/author 基础字段，仅手工维护 category 与市场描述。

4. 把 `claude plugin validate` 纳入发布前检查  
- 在 PR 模板或发布脚本中强制执行一次，减少上线后发现问题。
