# plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md 研究

## 场景与职责
`manifest-reference.md` 是 `plugin-structure` 技能体系里关于 `.claude-plugin/plugin.json` 的字段级规范文档，承担“配置契约源（contract source）”职责。它与 `component-patterns.md` 的分工是：

- `manifest-reference.md`：定义字段类型、默认值、路径解析顺序、校验错误样式（`manifest-reference.md:11-552`）。
- `component-patterns.md`：定义目录组织与架构演进模式（`component-patterns.md:29-567`）。

在上下游中，本文件是多个流程的共同依赖：

1. 上游调用方
- `plugin-structure/README.md` 将其列为 references 第一项，用于 `plugin.json` 深度规则下钻（`plugins/plugin-dev/skills/plugin-structure/README.md:34-39`）。
- `plugin-structure/SKILL.md` 的 manifest 章节给出基础字段，细节由本文件补全（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:46-107`）。
- `/plugin-dev:create-plugin` Phase 2/4 依赖 manifest 规则完成结构规划与初始 manifest 写入（`plugins/plugin-dev/commands/create-plugin.md:48-52,116-149`）。

2. 下游消费方
- `plugin-validator` 的 manifest 校验清单（required/optional/path/version/unknown fields）本质是本文件规则的执行化版本（`plugins/plugin-dev/agents/plugin-validator.md:56-66`）。
- `hook-development`、`mcp-integration` 等技能消费本文件中 `hooks` / `mcpServers` 字段和默认路径规则（例如 `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-330` 在多个研究文档中被反复引用）。

3. 仓库层对齐关系
- 仓库级 `plugins/README.md` 给出插件目录总纲（manifest 位于 `.claude-plugin/plugin.json`），本文件提供字段和解析细则，构成“结构总纲 + 配置契约”的双层文档（`plugins/README.md:47-61`，`manifest-reference.md:7-10`）。

## 功能点目的
本文件围绕 `plugin.json` 的可读、可写、可验证三个目标展开。

1. 定义硬性入口：manifest 文件位置
- 目的：消除“manifest 放错目录导致插件不可识别”的基础错误。
- 规则：必须放在 `.claude-plugin/plugin.json`（`manifest-reference.md:7-10`）。

2. 定义核心字段语义
- `name`：唯一标识 + kebab-case 正则（`manifest-reference.md:15-40`）。
- `version`：语义化版本与发布节奏（`manifest-reference.md:42-63`）。
- `description`：面向发现与展示的描述质量（`manifest-reference.md:65-83`）。
目的：保证最小可运行性与版本治理一致性。

3. 定义分发元数据
- `author/homepage/repository/license/keywords` 的结构、用途与格式建议（`manifest-reference.md:85-208`）。
目的：支撑发布、归属、可发现性与合规。

4. 定义组件路径字段
- `commands`、`agents`：string 或 string[]（`manifest-reference.md:211-257`）。
- `hooks`、`mcpServers`：路径字符串或 inline object（`manifest-reference.md:259-330`）。
目的：将“组件布局决策”映射为可发现配置。

5. 定义路径解析协议
- 路径必须是 `./` 开头的相对路径，禁止绝对路径与 `../`（`manifest-reference.md:334-350`）。
- 解析顺序：先默认目录，再 manifest 自定义路径，最后合并注册并处理冲突（`manifest-reference.md:352-371`）。
目的：保证跨平台与可移植行为一致。

6. 定义校验与错误修复模式
- 给出语法校验、字段校验、组件校验，以及常见错误的“错误示例 -> 修复示例”（`manifest-reference.md:373-447`）。
目的：把“抽象规范”降维为可操作整改步骤。

7. 定义复杂度分档模板
- 最小、推荐、完整三档 manifest（`manifest-reference.md:449-519`）。
目的：让不同阶段插件可按复杂度逐步进化。

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 关键流程
1. 加载流程（运行期）
- Claude Code 读取 `.claude-plugin/plugin.json`；
- 扫描默认目录（commands/agents/skills/hooks/.mcp）；
- 扫描 manifest 自定义路径；
- 合并组件并注册，命名冲突报错（`manifest-reference.md:352-371`）。

2. 校验流程（开发期）
- 语法层：JSON 合法性（`manifest-reference.md:379-383`）。
- 字段层：`name/version/path/url` 形态合法（`manifest-reference.md:384-389`）。
- 组件层：路径存在、hook/mcp 配置有效、无循环依赖（`manifest-reference.md:390-394`）。
- 错误纠正：按常见错误样例进行定向修复（`manifest-reference.md:395-447`）。

3. 发布流程（分发期）
- 从 minimal 逐步补充 metadata、路径字段、hooks 与 mcp 配置，迁移到推荐/完整清单（`manifest-reference.md:449-519,521-552`）。

### 2) 关键数据结构
1. `name` 字段
- 类型：string
- 约束：`/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/`
- 语义：插件唯一标识，参与冲突检测（`manifest-reference.md:21-36`）。

2. `version` 字段
- 类型：string
- 约束：MAJOR.MINOR.PATCH（支持预发布标签）
- 语义：兼容性与变更策略标识（`manifest-reference.md:44-63`）。

3. metadata 字段簇
- `author`：对象或字符串；
- `homepage`/`repository`：URL 或对象；
- `license`：SPDX；
- `keywords`：string[]。
（`manifest-reference.md:87-208`）

4. 组件路径字段簇
- `commands`、`agents`：string | string[]；
- `hooks`、`mcpServers`：path string | inline object。
（`manifest-reference.md:211-330`）

### 3) 协议细节
1. 路径协议
- 相对路径 + `./` 前缀 + 禁止 `../` + 使用正斜杠（`manifest-reference.md:336-351`）。

2. 默认值协议
- `commands` 默认 `./commands`；`agents` 默认 `./agents`；`hooks` 默认 `./hooks/hooks.json`；`mcpServers` 默认 `./.mcp.json`（`manifest-reference.md:214-215,247-248,261-263,300-302`）。

3. 合并协议
- 自定义路径“补充而非替代”默认路径，所有发现组件参与注册，冲突时报错（`manifest-reference.md:368-371`）。

4. 可移植路径协议
- hook 命令与 mcp args 示例都采用 `${CLAUDE_PLUGIN_ROOT}`/环境变量展开（`manifest-reference.md:283-285,318-321`）。

### 4) 关键命令与配置片段
1. hooks 中的 command hook 示例
```json
{
  "type": "command",
  "command": "bash ${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh",
  "timeout": 30
}
```
（`manifest-reference.md:281-285`）

2. mcpServers inline 示例
```json
{
  "mcpServers": {
    "github": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/servers/github-mcp.js"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```
（`manifest-reference.md:313-325`）

### 5) 配置、测试、脚本、文档上下文
1. 配置
- 本文件是 `plugin.json` 配置契约，直接服务 manifest 编写与审核。

2. 测试
- 本文件无测试脚本；实际校验由 `plugin-validator` 或手工 `jq` + 规则检查执行（`plugins/plugin-dev/agents/plugin-validator.md:56-66`）。

3. 脚本
- 文中出现的命令仅为路径示例，不是本目录自带可执行脚本。

4. 文档
- 与 `SKILL.md`、`component-patterns.md`、`examples/` 构成完整学习闭环（`plugins/plugin-dev/skills/plugin-structure/README.md:17-49,50-93`）。

## 关键代码路径与文件引用
1. 被研究对象
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:1-552`

2. 上游调用方（规范入口）
- `plugins/plugin-dev/skills/plugin-structure/README.md:34-39,85-93`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:46-107,339-356,476`
- `plugins/plugin-dev/commands/create-plugin.md:48-52,116-149,233-260`
- `plugins/plugin-dev/README.md:95-113`

3. 下游消费方（规则执行/复用）
- `plugins/plugin-dev/agents/plugin-validator.md:56-66,107-123`
- `plugins/plugin-dev/skills/hook-development/SKILL.md`（hooks 规范联动）
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md`（mcpServers 规范联动）
- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:17-23`
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:41-55`
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:131-142`

4. 横向一致性文档
- `plugins/README.md:47-61`（插件目录总纲）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:5-27`（发现/激活阶段补充）

## 依赖与外部交互
1. 仓库内依赖
- 依赖 `plugin-structure` 技能触发机制让该文档在需要时被加载（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:2-4`）。
- 依赖 `create-plugin` 与 `plugin-validator` 将文档规则转译为“创建动作 + 校验动作”。

2. 运行时协议交互
- 与 Claude Code 插件发现机制交互：manifest 定义组件位置、默认路径和合并顺序（`manifest-reference.md:352-371`）。
- 与 Hook/MCP 协议交互：`hooks` 与 `mcpServers` 双形态配置（path 或 inline object，`manifest-reference.md:259-330`）。

3. 外部标准交互
- SemVer：版本约定（`manifest-reference.md:44-63`）。
- SPDX：许可证标识（`manifest-reference.md:166-180`）。
- URL 规范：homepage/repository 的可访问性要求（`manifest-reference.md:115-163`）。

4. 外部系统交互边界
- 本文件不直接连接网络、MCP 服务或命令行；
- 它仅定义配置格式，真正交互发生在插件加载和组件执行阶段。

## 风险、边界与改进建议
1. 风险：文本规则缺少机器可执行约束
- 当前规则主要通过文档描述和示例约束，缺乏官方 JSON Schema 链接或仓内自动校验脚本。
- 建议：新增 `plugin-json.schema.json` 并在 CI 加入 schema 验证步骤，减少“文档正确、配置错误”的漏检。

2. 风险：默认值与实现漂移
- 文档声明默认路径与解析顺序（`manifest-reference.md:356-371`），若运行时实现调整而文档未同步，会导致误导。
- 建议：增加与运行时实现绑定的回归检查（例如 smoke plugin 自动验证默认发现路径）。

3. 风险：启发式阈值可能被误解为硬规则
- “inline hooks < 50 lines”“single inline server < 20 lines”是经验建议（`manifest-reference.md:293-330`），容易被理解成强制限制。
- 建议：在这些段落显式标注 `heuristic guidance`，避免误判。

4. 边界：未定义冲突处置细节
- 文档仅说明冲突会报错（`manifest-reference.md:371`），未明确优先级策略或错误码表现。
- 建议：补充“同名 command/agent 冲突案例”与诊断建议。

5. 边界：未覆盖平台差异与转义细节
- 已要求使用正斜杠和相对路径（`manifest-reference.md:338-351`），但未展开 Windows shell 转义、路径空格等实践差异。
- 建议：新增一节“Cross-platform path pitfalls”，给出 Bash/PowerShell 对照样例。

6. 改进优先级建议
1. 先补 schema + CI 校验（最高收益）；
2. 再补冲突案例与平台坑位；
3. 最后补示例片段自动抽取验证（确保文档 JSON 代码块长期可解析）。
