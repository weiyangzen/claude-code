# DIR 研究：plugins/plugin-dev/skills/skill-development/references

## 场景与职责

`plugins/plugin-dev/skills/skill-development/references` 是 `skill-development` 技能的“方法论来源层”，当前目录仅包含一个参考文件：

- `plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md`

它在体系中的职责不是执行插件逻辑，而是沉淀“如何创建技能”的通用方法，并被 plugin-dev 的插件化技能流程按需引用。

1. 上游调用方（谁引用本目录）
- `skill-development/SKILL.md` 在 Additional Resources 中显式引用该文件作为完整原始方法论（`plugins/plugin-dev/skills/skill-development/SKILL.md:616-620`）。
- `plugin-dev/README.md` 将 Skill Development 描述为“基于 skill-creator 方法并做插件适配”，本目录即该方法来源（`plugins/plugin-dev/README.md:175-191`）。
- `/plugin-dev:create-plugin` 在 Phase 5 创建技能组件时强制加载 `skill-development`，从而间接消费本目录知识（`plugins/plugin-dev/commands/create-plugin.md:157-180,373-375`）。

2. 下游被调用方（谁落实本目录规则）
- `skill-reviewer` 把“遵循 skill-creator best practices”作为核心审查职责，属于本目录规则的执行检查者（`plugins/plugin-dev/agents/skill-reviewer.md:40-46,67-79,99-105`）。
- `plugin-validator` 在技能校验时检查 SKILL frontmatter、references/examples/scripts 结构与引用有效性，承接本目录提出的结构性约束（`plugins/plugin-dev/agents/plugin-validator.md:98-106`）。

3. 与仓库研究流程的关系
- `.ops/research_guard.sh` 会把目录研究任务模板化下发，要求产出文档、勾选 checklist、更新 todo；本次任务即该流程产物（`.ops/research_guard.sh:195-229`）。

## 功能点目的

基于 `skill-creator-original.md` 的内容，本目录核心功能目标可拆解为 6 个点：

1. 定义技能基本构成与元数据契约
- 明确 `SKILL.md` 必需、`name/description` 必需，以及 `scripts/references/assets` 可选资源层（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:25-45`）。

2. 定义渐进披露（Progressive Disclosure）
- 将技能信息分为 metadata、SKILL 主体、按需资源三层，目标是降低上下文常驻负担并保留深度知识（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:77-85`）。

3. 给出技能创建的端到端流程
- 提供 Step1~Step6：场景澄清、资源规划、初始化、编辑、打包、迭代（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:87-209`）。

4. 约束技能写作风格与触发语义
- 要求描述三人称、正文祈使/不定式，以提升技能触发与执行的一致性（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:44,167-174`）。

5. 给出“可复用资源优先”的落地策略
- 先实现 scripts/references/assets，再回填 SKILL.md，减少重复劳动并提升可维护性（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:159-174`）。

6. 提供可分发闭环（通用 skill 场景）
- 包含初始化脚本与打包脚本命令，目标是产出可共享 zip 包（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:138-199`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

1. 需求到技能流程
- Step 1 先收集具体用户触发样例，再进入资源规划，避免“先写文档后补逻辑”的逆序问题（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:91-130`）。

2. 资源先行流程
- Step 2/4 要求从样例反推 scripts/references/assets，并优先落地这些资源，再编写 SKILL 主文档（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:108-130,159-174`）。

3. 初始化与分发流程（通用模式）
- Step 3 使用 `scripts/init_skill.py` 生成模板结构。
- Step 5 使用 `scripts/package_skill.py` 先校验后打包。
- Step 6 基于真实使用反馈迭代。
- 证据：`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:132-209`。

4. 在 plugin-dev 中的适配流程
- `skill-development/SKILL.md` 显式声明插件技能不走通用 `init_skill.py`，而是直接在 `plugin-name/skills/` 手工建结构（`plugins/plugin-dev/skills/skill-development/SKILL.md:137-147`）。
- 同时声明插件技能通常随插件分发，不需要单独 zip 打包（`plugins/plugin-dev/skills/skill-development/SKILL.md:278-281`）。

### 2) 关键数据结构/协议

1. 技能目录结构协议
```text
skill-name/
├── SKILL.md
└── scripts/ references/ assets/ (optional)
```
来源：`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:27-40`。

2. frontmatter 元数据协议
- 最小必需字段：`name`、`description`。
- 目录内示例还包含 `license` 字段（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:1-5`）。

3. 描述触发协议
- `description` 不只是说明文案，而是“何时触发”的行为契约，要求使用具体场景与第三人称表达（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:44`）。

4. 文档分层协议
- 细节尽量下沉到 `references/`，避免 SKILL 和 references 双份重复（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:64-67`）。

### 3) 关键命令与脚本语义

1. 初始化命令
```bash
scripts/init_skill.py <skill-name> --path <output-directory>
```
语义：生成 SKILL 模板 + 示例资源目录（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:142-153`）。

2. 打包命令
```bash
scripts/package_skill.py <path/to/skill-folder>
scripts/package_skill.py <path/to/skill-folder> ./dist
```
语义：校验通过后产出 zip（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:179-199`）。

3. 当前仓库实现边界
- 仓库根目录 `scripts/` 未发现 `init_skill.py`/`package_skill.py`；当前仅有 issue/workflow 相关脚本（`scripts/` 目录实际内容）。
- 因此本目录中的这两条命令在当前仓库是“通用方法引用”，不是可直接运行的本地脚本。

### 4) 配置、测试、脚本、文档上下文

1. 配置
- 本目录主要提供 SKILL frontmatter 与目录结构规范；不含插件运行时配置文件（如 `.claude-plugin/plugin.json`）。

2. 测试
- 目录内没有自动化测试脚本；质量保证主要依赖 `skill-reviewer`/`plugin-validator` 的文档驱动检查（`plugins/plugin-dev/agents/skill-reviewer.md:47-97`，`plugins/plugin-dev/agents/plugin-validator.md:98-106`）。

3. 脚本
- 目录内无可执行脚本；仅给出通用命令示例（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:142-187`）。

4. 文档
- 该目录是 `skill-development/SKILL.md` 的深层参考补充，不作为独立入口文档使用（`plugins/plugin-dev/skills/skill-development/SKILL.md:616-620`）。

## 关键代码路径与文件引用

### 目标目录（被研究对象）

1. `plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md`
- 目录唯一文件，承载通用 skill-creator 全流程。

### 上游调用链（调用方）

1. `plugins/plugin-dev/skills/skill-development/SKILL.md:616-620`
- 显式引用本目录文件。

2. `plugins/plugin-dev/README.md:175-191`
- 定义 Skill Development 能力并指向“skill-creator 方法来源”。

3. `plugins/plugin-dev/commands/create-plugin.md:157-180,247-250,373-375`
- 创建插件时，技能组件开发与评审流程会间接依赖本目录规则。

### 下游消费链（被调用方）

1. `plugins/plugin-dev/agents/skill-reviewer.md:40-46,67-84,99-105`
- 将本目录中的“触发词、分层、资源组织”转换成可执行审查标准。

2. `plugins/plugin-dev/agents/plugin-validator.md:98-106`
- 在插件级验证中检查技能结构与 references 依赖可达性。

3. `plugins/plugin-dev/skills/skill-development/SKILL.md:137-147,278-281`
- 对本目录的通用流程做插件化改写（手工建结构、随插件分发）。

### 研究流程相关文件（非业务调用，但与本次落地相关）

1. `.ops/research_guard.sh:195-229`
- 定义目录研究任务模板。

2. `Docs/researches/blueprint_checklist.md:96`
- 本次要勾选的目标条目。

3. `.ops/generate_daily_research_todo.sh:1-42`
- 根据 checklist 生成当日待办快照。

## 依赖与外部交互

1. 对 Claude Code 技能机制的依赖
- 依赖 SKILL frontmatter 与技能按需加载语义；本目录本身不执行代码，只提供方法规范。

2. 对插件开发流程的依赖
- 通过 `create-plugin` 工作流和 `skill-reviewer/plugin-validator` 形成“编写-审查-验证”闭环（`create-plugin.md:153-269`）。

3. 对脚本执行环境的潜在依赖（通用模式）
- 参考文件期望可执行 `scripts/init_skill.py` 与 `scripts/package_skill.py`；这要求本地存在对应脚本与 Python 运行环境。
- 当前仓库缺失这两个脚本，说明依赖在本仓库内未满足。

4. 对外部交互边界
- 本目录不直接访问网络、不调用 API；外部交互主要发生在插件分发或用户手工执行脚本时。

## 风险、边界与改进建议

1. 风险：通用方法与 plugin-dev 实现存在语义落差
- reference 要求 `init_skill.py/package_skill.py`，而 plugin-dev 实际采用“手工建结构 + 随插件分发”。
- 建议：在 `skill-creator-original.md` 文件头增加“通用版说明”警示，明确与 plugin-dev 的差异入口（可链接 `skill-development/SKILL.md:137-147,278-281`）。

2. 风险：参考命令在当前仓库不可直接执行
- 根目录 `scripts/` 无 `init_skill.py/package_skill.py`。
- 建议：在 `skill-development/SKILL.md` 的 references 段增加一句“本仓库不提供通用打包脚本，请按插件模式执行”。

3. 风险：license 字段指向 `LICENSE.txt`，仓库内未见该文件
- 可能导致使用者误判许可文件位置。
- 建议：补充注释说明该字段来自上游原文；或在 reference 旁补一个“license source note”。

4. 风险：目录缺少可验证资产
- 当前目录只有文档，没有 examples/scripts/test，审查全依赖人工或 agent 规则。
- 建议：增加最小示例（如 `references/plugin-adaptation-note.md`）和引用完整性校验脚本（可复用 `rg` 检查引用路径）。

5. 边界结论
- 该目录是“方法论知识库”，不是“执行代码模块”。
- 其主要价值在于给 `skill-development` 提供可追溯原始语义；其主要风险在于通用流程与插件化流程的差异未被充分显式化。
