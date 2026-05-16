# plugins/frontend-design/skills/frontend-design 目录研究（DIR）

## 场景与职责

`plugins/frontend-design/skills/frontend-design` 是 `frontend-design` 插件中的“单技能目录”，其核心职责不是提供可执行脚本，而是通过 `SKILL.md` 向 Claude 注入前端生成任务的设计约束与实现原则。

该目录在插件体系中的定位：
- 上游入口由 marketplace 声明插件源路径 `./plugins/frontend-design`（`.claude-plugin/marketplace.json:73-81`）。
- 插件启用后，Claude Code 按默认发现机制扫描 `skills/` 下包含 `SKILL.md` 的子目录（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-347`）。
- 本目录作为命中的技能载体，指导“前端组件/页面/应用生成”场景的输出风格与质量要求（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-3,7-42`）。

目录事实：
- 目录下仅 1 个文件：`plugins/frontend-design/skills/frontend-design/SKILL.md`。
- 未发现 `scripts/`、`references/`、`assets/`、测试代码或运行脚本。

## 功能点目的

1. 技能触发语义定义
- 通过 frontmatter `name` 与 `description` 声明技能语义和触发条件，目标是在“用户请求构建前端界面”时自动匹配该 skill（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-4`）。

2. 设计先行约束
- 明确要求编码前先做 `Purpose / Tone / Constraints / Differentiation` 四维决策，避免直接进入模板化代码拼接（`plugins/frontend-design/skills/frontend-design/SKILL.md:11-19`）。

3. 产出质量基线
- 要求实现必须“可运行、生产可用、风格一致、细节精修”，覆盖 HTML/CSS/JS/React/Vue 等常见前端输出形态（`plugins/frontend-design/skills/frontend-design/SKILL.md:21-25`）。

4. 美学与反模式控制
- 对字体、色彩、动效、布局与背景细节给出正向指导，同时显式禁止常见“AI 套路审美”（`plugins/frontend-design/skills/frontend-design/SKILL.md:29-38`）。

5. 复杂度匹配机制
- 要求实现复杂度与设计方向一致：极繁风格需更重实现，极简风格需克制和精度，避免“视觉目标与代码实现不匹配”（`plugins/frontend-design/skills/frontend-design/SKILL.md:40`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程（调用链）

1. Marketplace 注册插件条目并给出 source
- `frontend-design` 条目指向 `./plugins/frontend-design`（`.claude-plugin/marketplace.json:73-81`）。

2. 插件启用时读取 manifest
- 自动发现链路先读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`；`plugins/frontend-design/.claude-plugin/plugin.json:1-9`）。

3. 扫描技能目录
- 扫描 `skills/` 中包含 `SKILL.md` 的子目录并注册技能（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-168,346`）。

4. 任务匹配触发 skill
- 当前端需求与 `description` 语义匹配时加载该 `SKILL.md` 正文约束（`plugins/frontend-design/skills/frontend-design/SKILL.md:2-3`，`plugins/plugin-dev/skills/skill-development/SKILL.md:79-84`）。

5. 指令执行到代码输出
- 执行顺序是“先设计决策，再落地前端实现”，并受美学禁用规则约束（`plugins/frontend-design/skills/frontend-design/SKILL.md:11-42`）。

### 2) 关键数据结构/协议

1. `SKILL.md` frontmatter 协议（YAML）
- 字段：`name`、`description`、`license`（`plugins/frontend-design/skills/frontend-design/SKILL.md:1-5`）。
- 在技能体系中，`name/description` 是触发匹配的元数据核心（`plugins/plugin-dev/skills/skill-development/SKILL.md:32-35,44`）。

2. 技能正文协议（Markdown 指令）
- 结构化章节：`Design Thinking` 与 `Frontend Aesthetics Guidelines`（`plugins/frontend-design/skills/frontend-design/SKILL.md:11,27`）。
- 语义类型：流程约束（先决策后编码）+ 非功能约束（审美/复杂度/一致性）。

3. Progressive Disclosure 约束（体系级）
- metadata 常驻、正文触发加载、资源按需加载（`plugins/plugin-dev/skills/skill-development/SKILL.md:79-84`）。
- 本技能当前仅有正文层，未定义第三层资源（`scripts/`、`references/`、`assets/`）。

### 3) 关键命令与可操作验证

仓库内可用于验证该目录状态的命令：
- `find plugins/frontend-design/skills/frontend-design -maxdepth 4 -type f`
- `nl -ba plugins/frontend-design/skills/frontend-design/SKILL.md`
- `find plugins/frontend-design -maxdepth 6 \( -name '*test*' -o -name '*.spec.*' -o -name '*.test.*' -o -name '*.sh' -o -name '*.py' -o -name '*.js' -o -name '*.ts' \)`

验证结论：
- 该目录是“文档驱动 skill”，没有执行代码路径、没有单测入口。

## 关键代码路径与文件引用

核心对象（被研究目录）
- `plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`

直接上游（调用方/发现方）
- `.claude-plugin/marketplace.json:73-81`（marketplace 注册 `frontend-design` 插件）
- `plugins/frontend-design/.claude-plugin/plugin.json:1-9`（插件 manifest）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-168,343-347`（技能目录规范与自动发现顺序）
- `plugins/plugin-dev/skills/skill-development/SKILL.md:27-40,79-84`（技能三层加载模型与资源组织规范）

同插件文档与功能说明
- `plugins/frontend-design/README.md:1-31`（用途、样例、外部 cookbook 链接）
- `plugins/README.md:21`（插件总览中声明该 skill 自动用于 frontend work）

研究流水线上下文（本任务执行相关）
- `.ops/research_guard.sh:195-229`（DIR 研究任务模板、报告命名规则、checklist 行号传递）
- `.ops/generate_daily_research_todo.sh:5-41`（从 checklist 生成当天 todo）
- `Docs/researches/blueprint_checklist.md:49-52`（该目录在蓝图中的状态行）

## 依赖与外部交互

内部依赖
- 依赖插件标准目录协议：`skills/<skill-name>/SKILL.md`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-168`）。
- 依赖自动发现机制：插件启用后扫描技能目录（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-347`）。
- 依赖 frontmatter 质量：描述文本决定触发覆盖面与准确性（`plugins/plugin-dev/skills/skill-development/SKILL.md:44`）。

外部交互
- 运行时不直接调用外部 API、命令或服务；本目录仅提供提示协议。
- 通过 README 间接引用外部知识链接（Frontend Aesthetics Cookbook），属于文档学习入口，不是运行时依赖（`plugins/frontend-design/README.md:24-27`）。

配置、测试、脚本、文档现状
- 配置：由 `SKILL.md` frontmatter 与插件 `plugin.json` 元数据组成。
- 测试：目录内未发现测试文件。
- 脚本：目录内未发现可执行脚本。
- 文档：`SKILL.md` 为主文档，`README.md` 为外围说明。

## 风险、边界与改进建议

风险 1：许可声明文件不一致
- `SKILL.md` 声明 `license: Complete terms in LICENSE.txt`（`plugins/frontend-design/skills/frontend-design/SKILL.md:4`），但插件目录未包含 `LICENSE.txt`。
- 影响：合规信息可追溯性不足，自动化检查可能误报。
- 建议：补充 `plugins/frontend-design/LICENSE.txt` 或将声明改为仓库现有许可文件路径（如根目录 `LICENSE.md`）。

风险 2：规则全在单文件，扩展性受限
- 当前无 `references/` 分层，若继续加入复杂规则（a11y、性能预算、响应式策略）会导致 `SKILL.md` 膨胀。
- 影响：可维护性下降，规则冲突更难治理。
- 建议：保持 `SKILL.md` 作为核心流程，细则拆分到 `references/`，符合技能分层最佳实践（`plugins/plugin-dev/skills/skill-development/SKILL.md:64-66,79-84`）。

风险 3：审美约束强，但工程约束粒度不足
- 文本强调“大胆审美”和“高质量视觉”，但缺少明确可执行阈值（例如动画性能、降级策略、键盘可达性最低标准）。
- 影响：不同会话输出一致性和工程可验收性可能波动。
- 建议：补充可量化 guardrails（如首屏动画时长上限、`prefers-reduced-motion` 必须支持、最小对比度规则）。

边界说明
- 本目录是指导性协议层，不承担构建、测试、发布等执行责任。
- 不定义 hooks/commands/agents/mcp，不直接产生运行时副作用。

改进优先级建议
1. 先修复 license 路径一致性问题（低成本、高确定性）。
2. 再引入 `references/`，拆出可访问性与性能约束清单。
3. 最后补充示例资产或模板，提升输出稳定性与可复用性。
