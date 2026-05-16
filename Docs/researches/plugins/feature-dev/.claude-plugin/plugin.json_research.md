# plugins/feature-dev/.claude-plugin/plugin.json 研究

## 场景与职责

`plugins/feature-dev/.claude-plugin/plugin.json` 是 `feature-dev` 插件的 manifest（插件清单），在插件生命周期里承担“被发现”和“被识别”的第一入口职责。

- 插件结构规范要求 manifest 必须位于 `.claude-plugin/plugin.json`，否则插件不会被识别（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-49`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- 组件生命周期中，Claude Code 在发现阶段先读取每个已启用插件的 manifest，再发现命令/agent/skill/hook（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`）。
- 仓库级 marketplace 将 `feature-dev` 的插件源注册为 `./plugins/feature-dev`，这使运行时可以定位到该 manifest（`.claude-plugin/marketplace.json:62-70`）。

因此，这个文件本身不执行业务逻辑，但它决定了该插件后续能力是否可见，包括 `/feature-dev` 命令和 3 个子代理是否能进入自动发现链路。

## 功能点目的

### 1. 定义插件身份与发布元数据

当前 `plugin.json` 内容为（`plugins/feature-dev/.claude-plugin/plugin.json:1-9`）：

- `name: "feature-dev"`：插件唯一标识，符合 kebab-case 规范（`manifest-reference.md:15-36`）。
- `version: "1.0.0"`：语义化版本（`manifest-reference.md:42-47`）。
- `description`：说明插件为“特性开发工作流 + 多代理支持”。
- `author`：作者信息与联系邮箱。

这些字段用于安装识别、冲突检测、展示信息和维护归属，不直接承载执行逻辑。

### 2. 启动默认自动发现行为

该 manifest 没有覆写 `commands`、`agents`、`hooks`、`mcpServers` 等路径字段，因此运行时会走默认扫描：

- 扫描 `commands/` 加载命令（`plugin-structure/SKILL.md:343-345`）。
- 扫描 `agents/` 加载子代理（`plugin-structure/SKILL.md:345-346`）。
- 其他目录按需扫描（`plugin-structure/SKILL.md:346-348`，`manifest-reference.md:356-367`）。

对 `feature-dev` 来说，这意味着：

- `commands/feature-dev.md` 成为 `/feature-dev` 的核心协议定义（`plugins/feature-dev/commands/feature-dev.md:1-125`）。
- `agents/code-explorer.md`、`agents/code-architect.md`、`agents/code-reviewer.md` 被自动注册，供 Phase 2/4/6 调度（`plugins/feature-dev/commands/feature-dev.md:41-44,78-79,106-107`）。

### 3. 与文档和 marketplace 形成双源元数据对齐

`feature-dev` 的 name/description/version/author 同时存在于：

- 插件 manifest（`plugins/feature-dev/.claude-plugin/plugin.json:2-8`）
- marketplace 条目（`.claude-plugin/marketplace.json:62-69`）
- 插件 README 的作者/版本段（`plugins/feature-dev/README.md:406-412`）

这带来可读性和分发便利，也引入了双源漂移风险（见“风险与改进建议”）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 目标对象 -> 被调用方）

1. 上游调用方：marketplace 与插件发现器  
- marketplace 注册 `source: "./plugins/feature-dev"`（`.claude-plugin/marketplace.json:69`）。  
- 发现阶段先读取 `.claude-plugin/plugin.json`（`component-patterns.md:11-15`）。

2. 目标对象：本文件 `plugin.json`  
- 提供基础元数据（`plugins/feature-dev/.claude-plugin/plugin.json:2-8`）。

3. 下游被调用方：命令与 agents  
- 命令协议：`plugins/feature-dev/commands/feature-dev.md`，定义 7 阶段流程。  
- 子代理：`code-explorer`、`code-architect`、`code-reviewer`。  
- 用户入口：`/feature-dev`（`plugins/feature-dev/README.md:19-33`）。

4. 运行时编排（简化）  
`marketplace(source)` -> `plugins/feature-dev/.claude-plugin/plugin.json` -> 自动扫描 `commands/` + `agents/` -> 用户执行 `/feature-dev` -> Phase 2/4/6 并行拉起对应 agents -> 汇总后继续主流程（`commands/feature-dev.md:36-54,73-82,101-109`）。

### B. 数据结构与约束

文件是标准 JSON manifest，字段规模小、配置最小化：

```json
{
  "name": "feature-dev",
  "version": "1.0.0",
  "description": "Comprehensive feature development workflow with specialized agents for codebase exploration, architecture design, and quality review",
  "author": {
    "name": "Sid Bidasaria",
    "email": "sbidasaria@anthropic.com"
  }
}
```

约束来源：

- `name` 必须为 kebab-case（`manifest-reference.md:15-36`）。
- `version` 为 semver（`manifest-reference.md:42-47`）。
- manifest 在插件加载时会做语法与字段校验（`manifest-reference.md:377-389`）。

本文件未声明路径字段，属于“最小但完整 metadata”风格，符合“manifest 保持精简”的建议（`plugin-structure/SKILL.md:365-368`）。

### C. 协议与命令接口

1. 插件协议  
- 由 manifest 声明插件身份，触发自动发现。  
- 自动发现顺序与合并规则由插件结构规范定义（`manifest-reference.md:352-371`）。

2. 命令协议（被此 manifest 间接激活）  
- `/feature-dev` 命令前置原则：先理解再动手、必须澄清、必须用户批准后实现（`commands/feature-dev.md:12-16,61-69,89-93`）。
- 阶段化调用子代理：Phase 2/4/6 并行多 agent（`commands/feature-dev.md:41,78,106`）。

3. agent 协议（被此 manifest 间接激活）  
- `code-explorer`：追踪调用链与关键文件清单（`agents/code-explorer.md:41-51`）。  
- `code-architect`：输出可执行架构蓝图（`agents/code-architect.md:24-34`）。  
- `code-reviewer`：默认基于 `git diff` 审查，并只报告 `>=80` 置信度问题（`agents/code-reviewer.md:13,33`）。

## 关键代码路径与文件引用

### 目标对象

- `plugins/feature-dev/.claude-plugin/plugin.json:1-9`

### 调用方（上游）

- `.claude-plugin/marketplace.json:62-70`（feature-dev 注册与 source 路径）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`（发现阶段先读 manifest）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-49,343-348`（manifest 位置与自动发现顺序）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,377-393`（manifest 位置与加载校验）

### 被调用方（下游）

- `plugins/feature-dev/commands/feature-dev.md:1-125`（7 阶段主协议）
- `plugins/feature-dev/agents/code-explorer.md:1-51`
- `plugins/feature-dev/agents/code-architect.md:1-34`
- `plugins/feature-dev/agents/code-reviewer.md:1-46`
- `plugins/feature-dev/README.md:19-33,56-66,113-126,176-191,249-314`（用户侧行为说明）

### 配置 / 测试 / 脚本 / 文档上下文

- 配置：
  - `plugins/feature-dev/.claude-plugin/plugin.json`（插件元数据）
  - `.claude-plugin/marketplace.json`（插件分发注册）
  - `commands/feature-dev.md` 与 `agents/*.md` frontmatter（运行协议与工具权限）
- 测试：
  - `plugins/feature-dev` 目录下仅 6 个文件（manifest/README/1 命令/3 agent），无专用自动化测试目录或测试脚本（`find plugins/feature-dev -maxdepth 4 -type f` 结果）。
- 脚本：
  - `plugins/feature-dev` 无本地 shell/python/js 执行脚本；属于“提示词编排型插件”。
- 文档：
  - 用户文档主入口为 `plugins/feature-dev/README.md`。
  - 仓库索引文档引用该插件能力为 `/feature-dev + 3 agents`（`plugins/README.md:20`）。

## 依赖与外部交互

### 仓库内依赖

1. 依赖 marketplace 正确映射  
- `source` 必须指向插件根（当前为 `./plugins/feature-dev`），否则无法定位 manifest（`.claude-plugin/marketplace.json:69`）。

2. 依赖默认目录结构  
- 由于 manifest 未配置自定义路径，命令和 agent 必须在标准目录下，才能被自动发现（`plugin-structure/SKILL.md:343-346`）。

3. 依赖命令与 agent 名称协同  
- 命令文本里硬编码了 `code-explorer`/`code-architect`/`code-reviewer` 调度语义（`commands/feature-dev.md:41,78,106`），必须与 agents 实际定义名保持一致（`agents/*.md:2`）。

### 外部交互（间接）

`plugin.json` 本身不直接发起网络/系统调用；外部交互来自其激活后的下游 agents：

- 三个 agent 均声明 `WebFetch`、`WebSearch`、`BashOutput`（`agents/code-explorer.md:4`，`agents/code-architect.md:4`，`agents/code-reviewer.md:4`）。
- `code-reviewer` 默认以 `git diff` 为审查对象，对本地 Git 工作区有运行时依赖（`agents/code-reviewer.md:13`，`README.md:365-367`）。

### 一致性依赖

当前作者名存在轻微不一致：

- manifest：`Sid Bidasaria`（`plugin.json:6`）
- marketplace：`Siddharth Bidasaria`（`.claude-plugin/marketplace.json:66`）

邮箱一致，但展示名不一致，说明当前元数据同步依赖人工维护。

## 风险、边界与改进建议

### 风险

1. 单点失效风险  
- manifest 损坏（JSON 语法错误、关键字段错误、路径不合法）会在插件加载阶段直接阻断整个插件可用性（`manifest-reference.md:377-393`）。

2. 双源元数据漂移风险  
- `plugin.json`、`marketplace.json`、README 都含版本/作者/描述信息，长期迭代易产生不一致。

3. 运行语义不一致风险  
- `code-reviewer` 明确“只报 >=80”（`agents/code-reviewer.md:33`），但 README 输出分档示例写到 50-74（`README.md:309-312`），会影响用户预期。

4. 校验自动化不足风险  
- 当前 `feature-dev` 目录缺少针对 manifest 与命令/agent 协同关系的自动化验证脚本与测试。

### 边界

1. 本文件只负责插件身份与发现入口，不负责业务执行与工具调用。
2. 本文件不定义任何 hooks/MCP 运行配置，也不含命令实现代码。
3. 功能行为的真实执行路径由 `commands/feature-dev.md` 与 `agents/*.md` 决定。

### 改进建议

1. 增加 manifest 与 marketplace 一致性校验  
- 在 CI 增加脚本，逐项对齐 `name/version/description/author`，防止双源漂移。

2. 增加插件装配 smoke test  
- 至少检查：manifest 可解析、`/feature-dev` 命令可发现、3 个 agent 名称都可解析。

3. 统一评审阈值文档语义  
- 将 README 的置信度分档与 `code-reviewer` 的 `>=80` 规则对齐，避免用户误解。

4. 维持 manifest 精简原则但补充校验门  
- 保持当前最小 manifest 设计不变（简单、清晰），同时在流程中加自动校验，降低人工维护风险。
