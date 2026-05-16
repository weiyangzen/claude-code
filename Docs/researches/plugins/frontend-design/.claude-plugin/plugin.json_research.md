# plugins/frontend-design/.claude-plugin/plugin.json 研究

## 场景与职责

`plugins/frontend-design/.claude-plugin/plugin.json` 是 `frontend-design` 插件的 manifest（插件清单），其核心职责是让 Claude Code 在插件发现阶段识别该插件，并将其纳入自动发现链路。

- 插件结构规范要求 manifest 必须位于 `.claude-plugin/plugin.json`，否则插件不会被识别（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-49`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- 组件生命周期文档说明，发现阶段会先读取每个插件的 manifest，再进入 commands/agents/skills/hooks 的发现与注册（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-15`）。
- 仓库 marketplace 将该插件注册到 `source: "./plugins/frontend-design"`，为运行时定位 manifest 提供入口（`.claude-plugin/marketplace.json:73-81`）。

该文件本身不执行前端生成逻辑，但它是 `frontend-design` skill 生效的必要前置。

## 功能点目的

### 1. 定义插件身份元数据

当前 manifest 字段如下（`plugins/frontend-design/.claude-plugin/plugin.json:1-9`）：

- `name: "frontend-design"`：插件唯一标识。
- `version: "1.0.0"`：语义化版本。
- `description`：插件用途描述。
- `author`：作者信息（名称和邮箱）。

字段形态遵循“最小可用 metadata”模式，未配置自定义组件路径。

### 2. 启用默认自动发现（尤其是 skills）

该 manifest 未声明 `commands`、`agents`、`hooks`、`mcpServers` 自定义路径，因此依赖默认扫描策略（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:354-367`）：

- 扫描 `./skills/` 下包含 `SKILL.md` 的子目录（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:346`）。
- 对本插件而言，下游核心对象是 `plugins/frontend-design/skills/frontend-design/SKILL.md`（`plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`）。

### 3. 作为仓库内多处文档/分发信息的锚点

`frontend-design` 在以下位置同时出现：

- 插件 manifest（`plugins/frontend-design/.claude-plugin/plugin.json:2-8`）
- marketplace 注册（`.claude-plugin/marketplace.json:73-81`）
- 插件说明（`plugins/frontend-design/README.md:1-31`）
- 仓库插件总览（`plugins/README.md:21`）

这带来可见性，但也引入元数据同步成本。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 目标对象 -> 被调用方）

1. 调用方（上游）
- marketplace 条目声明 `source: ./plugins/frontend-design`，使插件目录可被定位（`.claude-plugin/marketplace.json:80`）。
- 插件发现流程读取 `.claude-plugin/plugin.json` 作为入口（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`）。

2. 目标对象（本文件）
- 解析 JSON manifest，建立插件身份与元数据（`plugins/frontend-design/.claude-plugin/plugin.json:2-8`）。

3. 被调用方（下游）
- 自动发现并加载 `skills/frontend-design/SKILL.md`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:346`，`plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`）。
- 在任务上下文命中前端需求时，skill description 触发加载并约束输出风格（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-3`，`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:25`）。

简化流程：

`marketplace(source)` -> `plugins/frontend-design/.claude-plugin/plugin.json` -> 发现 `skills/frontend-design/SKILL.md` -> 前端任务触发 skill -> 按设计规则生成前端代码。

### B. 数据结构与约束

manifest 数据结构：

```json
{
  "name": "frontend-design",
  "version": "1.0.0",
  "description": "Frontend design skill for UI/UX implementation",
  "author": {
    "name": "Prithvi Rajasekaran, Alexander Bricken",
    "email": "prithvi@anthropic.com, alexander@anthropic.com"
  }
}
```

关键约束（来自 manifest 参考规范）：

- `name` 为必填并使用 kebab-case（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-36`）。
- `version` 推荐 semver（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:42-47`）。
- manifest 在加载时经过 JSON、字段和路径校验（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:377-393`）。

### C. 协议与命令

- 插件协议：manifest 驱动自动发现，默认目录 + 可选自定义路径，且为“补充而非替换”（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:355`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:368-371`）。
- skill 协议：通过 `SKILL.md` frontmatter 的 `description` 与任务语义匹配触发（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-3`，`plugins/plugin-dev/skills/skill-development/SKILL.md:162-168`）。
- 命令层：该插件无 `commands/` 目录，不提供 slash command；属于“纯 skill 插件”（`find plugins/frontend-design -maxdepth 4 -type f` 结果仅 3 个文件）。

## 关键代码路径与文件引用

### 目标对象

- `plugins/frontend-design/.claude-plugin/plugin.json:1-9`

### 调用方（上游）

- `.claude-plugin/marketplace.json:73-81`（插件注册与 source 路径）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`（发现阶段先读 manifest）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`（自动发现顺序）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,377-393`（路径与校验规则）

### 被调用方（下游）

- `plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`（前端设计约束主体）
- `plugins/frontend-design/README.md:5-27`（用户层用途与触发示例）
- `plugins/README.md:21,49-60`（仓库总览与标准插件结构）

### 配置 / 测试 / 脚本 / 文档上下文

- 配置：
  - `plugins/frontend-design/.claude-plugin/plugin.json`（插件元数据）
  - `.claude-plugin/marketplace.json`（插件分发注册）
  - `plugins/frontend-design/skills/frontend-design/SKILL.md` frontmatter（skill 触发语义）
- 测试：
  - `plugins/frontend-design` 目录未发现 `*.test.*`、`*.spec.*` 等测试文件（目录文件清单仅 3 个）。
- 脚本：
  - 插件目录未发现 `*.sh`、`*.py`、`*.js`、`*.ts` 执行脚本；运行行为由技能文本约束驱动。
- 文档：
  - `plugins/frontend-design/README.md`（用户文档）
  - `plugins/README.md`（仓库级插件总览）
  - `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md`（manifest 协议参考）

## 依赖与外部交互

### 仓库内依赖

1. 依赖 marketplace source 的正确性
- 若 `.claude-plugin/marketplace.json` 的 `source` 偏离 `./plugins/frontend-design`，manifest 可能无法被找到（`.claude-plugin/marketplace.json:80`）。

2. 依赖标准目录自动发现
- 由于 manifest 未配置自定义路径，skills 发现依赖默认 `./skills/` 约定（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:356-361`）。

3. 依赖 skill frontmatter 触发质量
- `frontend-design` 的实际行为由 `SKILL.md` 描述触发并约束；manifest 只负责“可被发现”（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-4`）。

### 外部交互

- `plugin.json` 本身无网络/命令执行逻辑。
- 外部交互主要来自文档引用：README 链接到外部 cookbook（`plugins/frontend-design/README.md:26`）。
- 该插件未定义 hooks 或 MCP，因此不直接引入额外运行时外部系统依赖（manifest 中无 `hooks`/`mcpServers` 字段）。

### 一致性观察

存在 manifest 与 marketplace 的轻微元数据不一致：

- `description`：manifest 为简短版（`plugin.json:4`），marketplace 为更长版（`.claude-plugin/marketplace.json:74`）。
- `author`：manifest 使用双人逗号拼接（`plugin.json:6-7`），marketplace 为 `A & B` 且只保留一个邮箱（`.claude-plugin/marketplace.json:77-79`）。

这不一定导致运行故障，但会增加维护与展示一致性风险。

## 风险、边界与改进建议

### 风险

1. manifest 单点失效风险
- JSON 语法或字段格式错误会在加载阶段阻断插件可用性（`manifest-reference.md:377-393`）。

2. 元数据多源漂移风险
- `plugin.json`、`marketplace.json`、README 均含插件描述与作者信息，可能长期不一致。

3. 作者字段解析兼容风险
- 当前 `author.email` 为逗号分隔双邮箱字符串（`plugin.json:7`）；若某些消费方默认单邮箱语义，可能出现展示或解析歧义。

4. 行为验证缺口风险
- 插件为纯技能文本驱动，缺少本地自动化测试覆盖；skill 触发是否稳定依赖运行时语义匹配。

### 边界

1. 本文件仅提供插件身份与发现入口，不直接执行 UI 生成代码。
2. 本文件不定义命令、agent、hook、MCP 的具体实现。
3. 插件效果质量主要由下游 `SKILL.md` 内容和模型执行策略决定。

### 改进建议

1. 增加元数据一致性校验
- 在 CI/脚本中对比 `plugin.json` 与 `marketplace.json` 的 `name/version/description/author`，减少双源漂移。

2. 规范 author 结构
- 将多作者、多邮箱格式显式约定（例如 `author` 使用对象数组，或在文档中定义分隔语义），避免下游解析分歧。

3. 为纯 skill 插件增加 smoke check
- 最小化验证：manifest 可解析、`skills/frontend-design/SKILL.md` 可发现、skill frontmatter 合法。

4. 补充许可一致性
- `SKILL.md` 中声明 `license: Complete terms in LICENSE.txt`（`plugins/frontend-design/skills/frontend-design/SKILL.md:4`），但插件目录未见 `LICENSE.txt`；建议补齐文件或改为现有许可路径，降低合规歧义。
