# plugins/frontend-design/skills 目录研究（DIR）

## 场景与职责

`plugins/frontend-design/skills` 是 `frontend-design` 插件的技能容器目录，承担“可被自动发现的 skill 组织层”职责，而不是业务执行脚本层。

从当前仓库结构看，该目录只有一个技能子目录与一个核心文件：
- `plugins/frontend-design/skills/frontend-design/SKILL.md`

对应插件整体仅含 3 个文件（manifest、README、SKILL），说明该插件是典型的“纯 Skill 插件”，未引入 commands/agents/hooks 执行面（`plugins/frontend-design/.claude-plugin/plugin.json:1-9`，`plugins/frontend-design/README.md:1-31`，`plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`）。

其在系统中的职责边界：
1. 作为 `skills/*/SKILL.md` 的标准承载目录，被 Claude Code 自动扫描与加载（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-169,343-347`）。
2. 将“前端美学与实现质量约束”交给 `SKILL.md` 文本协议表达，不直接实现命令或脚本逻辑（`plugins/frontend-design/skills/frontend-design/SKILL.md:7-42`）。
3. 为上游插件发现链路提供下游能力落点：`marketplace -> plugin root -> skills 目录 -> SKILL.md`（`.claude-plugin/marketplace.json:73-82`，`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15,25`）。

## 功能点目的

### 1) 提供技能自动发现入口（目录级）
- 目录采用标准 `skills/<skill-name>/SKILL.md` 结构（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-168`）。
- 目的是让技能在插件启用后被自动注册，无需额外路径配置（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-347,355-367`）。

### 2) 承载前端任务触发语义（frontmatter）
- `SKILL.md` frontmatter 声明 `name: frontend-design` 与触发描述（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-4`）。
- 目的是在“用户请求构建组件/页面/应用”时自动命中 skill，而非依赖用户手动指定（`plugins/frontend-design/skills/frontend-design/SKILL.md:3`，`plugins/README.md:21`）。

### 3) 将设计决策流程前置
- 强制先做 `Purpose / Tone / Constraints / Differentiation` 决策再编码（`plugins/frontend-design/skills/frontend-design/SKILL.md:13-17`）。
- 目的是防止直接进入模板化代码生成，提升输出辨识度与一致性（`plugins/frontend-design/skills/frontend-design/SKILL.md:19,21-25`）。

### 4) 提供可执行的美学实现约束
- 对字体、色彩、动效、构图与背景细节提出明确策略与反模式限制（`plugins/frontend-design/skills/frontend-design/SKILL.md:30-38`）。
- 目的是将“前端审美要求”转化为可落地实现规范，直接影响代码风格与实现复杂度（`plugins/frontend-design/skills/frontend-design/SKILL.md:40-42`）。

### 5) 保持技能层轻量形态
- 当前该目录未引入 `references/`、`scripts/`、`assets/`，仅保留核心 `SKILL.md`。
- 目的是最小化加载面与维护面；代价是缺少结构化补充资料与可执行校验资产（对比技能开发推荐结构：`plugins/plugin-dev/skills/skill-development/SKILL.md:27-39,77-85`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用链）

1. 插件被 marketplace 声明并路由到插件根目录
- `.claude-plugin/marketplace.json` 注册 `frontend-design`，`source` 指向 `./plugins/frontend-design`（`.claude-plugin/marketplace.json:73-82`）。

2. 插件启用时读取 manifest
- 自动发现机制首先读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`；`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-14`）。

3. 组件扫描阶段进入 `skills/`
- 系统扫描 `skills/` 下含 `SKILL.md` 的子目录并加载技能（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-169,346`）。

4. 运行阶段根据任务上下文激活 skill
- 当任务语义匹配 skill 描述时触发技能规则（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:25`）。
- 本目录对应技能在前端构建任务中生效，驱动“先设计方向、再实现代码”的执行路径（`plugins/frontend-design/skills/frontend-design/SKILL.md:13-25`）。

### B. 数据结构

1. 目录结构（目标目录）
- `skills/`
- `skills/frontend-design/`
- `skills/frontend-design/SKILL.md`

2. 技能协议结构（Markdown + YAML frontmatter）
- frontmatter：`name`、`description`、`license`（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-4`）。
- body 分层：
  - `Design Thinking`（上下文分析与方向决策）`plugins/frontend-design/skills/frontend-design/SKILL.md:11-26`
  - `Frontend Aesthetics Guidelines`（实现约束与禁用项）`plugins/frontend-design/skills/frontend-design/SKILL.md:27-42`

3. 渐进加载协议（progressive disclosure）
- metadata 常驻；skill body 在触发时加载；资源按需加载（`plugins/plugin-dev/skills/skill-development/SKILL.md:79-85,273-276`）。
- 本目录目前只有前两层（metadata + body），无第三层资源文件（`references/scripts/assets`）。

### C. 命令/脚本执行特征

- 本目录不提供 slash command（无 `commands/`）。
- 不提供 agent（无 `agents/`）。
- 不提供 hooks 与 hook 脚本（无 `hooks/`）。
- 不提供 skill 内脚本工具（无 `scripts/`）。

因此该目录的“技术执行协议”是纯文本技能协议：通过 frontmatter 匹配 + body 规则约束来影响 Claude 的前端代码生成行为，而不是通过本地命令执行实现。

## 关键代码路径与文件引用

### 目标对象（DIR）
- `plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`

### 调用方（上游）
- marketplace 路由：`.claude-plugin/marketplace.json:73-82`
- 插件总览声明（Auto-invoked skill）：`plugins/README.md:21`
- 插件标准结构与 skills 目录约定：`plugins/README.md:49-60`
- 自动发现顺序：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-347`
- 生命周期与激活阶段：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:7-16,19-27`

### 被调用方（下游）
- 前端实现生成流程（HTML/CSS/JS/React/Vue）约束：`plugins/frontend-design/skills/frontend-design/SKILL.md:21`
- 前端审美规则执行体：`plugins/frontend-design/skills/frontend-design/SKILL.md:30-42`
- 插件 README 的用法样例与外部学习材料：`plugins/frontend-design/README.md:16-27`

### 配置 / 测试 / 脚本 / 文档
- 配置：技能触发配置由 `SKILL.md` frontmatter 提供（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-4`）。
- 测试：`plugins/frontend-design/skills` 目录下未发现测试文件。
- 脚本：`plugins/frontend-design/skills` 目录下未发现执行脚本。
- 文档：技能正文即主要规范文档，外部补充在插件 README（`plugins/frontend-design/README.md:24-27`）。

## 依赖与外部交互

### 1) 仓库内部依赖
- 依赖 marketplace 注册与 source 路径正确（`.claude-plugin/marketplace.json:73-82`）。
- 依赖 plugin manifest 能被正确读取（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`）。
- 依赖 `skills/*/SKILL.md` 自动发现机制（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-169,346`）。

### 2) 协议依赖
- 依赖 YAML frontmatter 的触发描述质量，决定是否命中 skill（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-4`；`plugins/plugin-dev/skills/skill-development/SKILL.md:44,162-168`）。
- 依赖技能正文保持“核心流程可读 + 规则可执行”，否则会降低运行期可控性（`plugins/plugin-dev/skills/skill-development/SKILL.md:190-196,587-588`）。

### 3) 外部交互
- 目录自身无 API/网络调用与系统命令执行。
- 间接外部交互为 README 外链文档（Frontend Aesthetics Cookbook），属于知识参考而非运行时依赖（`plugins/frontend-design/README.md:24-27`）。

## 风险、边界与改进建议

### 风险

1. 审美规则强，但缺少自动化验证
- 目前是纯文本约束，缺少脚本或测试去检查字体选择、动效降级、对比度等执行结果，回归质量依赖人工抽检。

2. `license` 字段引用可能失效
- `SKILL.md` 声明 `license: Complete terms in LICENSE.txt`，但插件目录内未见 `LICENSE.txt`（`plugins/frontend-design/skills/frontend-design/SKILL.md:4`）。

3. 规则演进易产生漂移
- 该目录没有 `references/` 分层，若继续扩展会把细节堆在单一 `SKILL.md`，后续维护和一致性校验成本升高（对比推荐：`plugins/plugin-dev/skills/skill-development/SKILL.md:64-66,190-194`）。

4. 与其他前端指导来源可能重叠
- 仓库存在系统级前端审美约束与其他文档指引，若无统一来源策略，可能出现同主题不同口径。

### 边界

1. 本目录只负责 skill 组织与规则承载，不负责命令执行编排。
2. 本目录不承担 hooks、agent、MCP、脚本工具链责任。
3. 本目录不提供测试资产与运行时健康检查逻辑。

### 改进建议

1. 增加 `references/` 拆分详细规范
- 将可访问性、性能预算、响应式断点策略、动效降级策略下沉到引用文档，保持 `SKILL.md` 核心且稳定。

2. 增加轻量 `scripts/` 质量检查工具
- 例如检测 `prefers-reduced-motion`、语义标签、基础对比度与字体 fallback，形成可执行护栏。

3. 修正 license 指向
- 若确有条款，补充 `LICENSE.txt`；否则更新 frontmatter 描述避免悬挂引用。

4. 建立最小样例回归
- 增加 `examples/`（如 dashboard / landing / settings）并定义期望要点，用于规则变更后的人机回归校验。
