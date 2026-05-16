# FILE 研究：plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md

## 场景与职责

### 文件定位
- 目标文件：`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md`
- 类型：`skill-development` 技能的参考文档（reference），不是可执行脚本。
- 作用：提供“通用 skill-creator 方法论”的原始版本，用于补充 plugin-dev 中的插件化技能开发流程。

### 在 plugin-dev 体系中的职责
1. 作为上游方法论来源：
- `skill-development/SKILL.md` 在 Additional Resources 明确指向该文件（`plugins/plugin-dev/skills/skill-development/SKILL.md:618-620`）。
2. 作为能力说明依据：
- `plugin-dev/README.md` 将 Skill Development 描述为“基于 skill-creator 方法做适配”，其方法来源即该文档（`plugins/plugin-dev/README.md:175-191`）。
3. 作为创建流程的补充知识源：
- `/plugin-dev:create-plugin` 在 Phase 5 要求加载 `skill-development` 技能，间接让该 reference 被消费（`plugins/plugin-dev/commands/create-plugin.md:157-180`）。

### 调用方与被调用方
- 调用方（直接/间接引用它的对象）：
  - `plugins/plugin-dev/skills/skill-development/SKILL.md`
  - `plugins/plugin-dev/README.md`
  - `plugins/plugin-dev/commands/create-plugin.md`（通过 skill-development 间接引用）
- 被调用方（落地它定义规范的执行者）：
  - `plugins/plugin-dev/agents/skill-reviewer.md`：检查 skill 是否遵循 skill-creator best practices（`plugins/plugin-dev/agents/skill-reviewer.md:40-45`）。
  - `plugins/plugin-dev/agents/plugin-validator.md`：校验 skills 目录结构、frontmatter、资源引用（`plugins/plugin-dev/agents/plugin-validator.md:98-106`）。

## 功能点目的

目标文件的核心目标不是“教如何写某个具体技能”，而是定义一个可复用的技能工程方法。可拆解为以下功能点：

1. 定义技能最小契约
- 必需 `SKILL.md`，frontmatter 必需 `name`/`description`（`.../skill-creator-original.md:25-45`）。
- 目的：保证技能可被系统识别和触发。

2. 定义资源分层
- `scripts/`、`references/`、`assets/` 三种可选资源分工（`.../skill-creator-original.md:46-75`）。
- 目的：把“执行逻辑、知识文档、输出素材”解耦。

3. 定义渐进披露（Progressive Disclosure）
- metadata 常驻、SKILL 主体按触发加载、资源按需加载（`.../skill-creator-original.md:77-85`）。
- 目的：控制上下文开销并保留深度知识。

4. 定义创建流程（Step 1-6）
- 从“场景样例澄清”到“资源规划、初始化、编辑、打包、迭代”（`.../skill-creator-original.md:87-209`）。
- 目的：减少拍脑袋设计，提升可复用与可交付性。

5. 约束触发与写作风格
- frontmatter 描述强调第三人称和触发语义（`.../skill-creator-original.md:44`）。
- 正文强调祈使/不定式写法（`.../skill-creator-original.md:167-174`）。
- 目的：提高触发准确度与指令可执行性。

6. 定义“可分发”闭环
- 初始化脚本 `init_skill.py` 与打包脚本 `package_skill.py`（`.../skill-creator-original.md:138-199`）。
- 目的：让技能产物可标准化校验并打包分发。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

#### 流程 A：需求澄清到资源规划
1. 收集用户会怎么提问（触发样例）`Step 1`（`...:91-107`）。
2. 从样例反推哪些能力需要稳定复用（scripts/references/assets）`Step 2`（`...:108-130`）。
3. 先做复用资源，再写 SKILL 主体 `Step 4`（`...:159-174`）。

价值：把技能从“静态说明文”变成“可反复执行的能力包”。

#### 流程 B：初始化到交付
1. `scripts/init_skill.py` 生成模板目录与示例文件（`...:138-153`）。
2. 手工清理不需要的示例资源并写入真实内容（`...:153,163`）。
3. `scripts/package_skill.py` 执行验证并打包 zip（`...:175-199`）。
4. 实战使用后迭代 `Step 6`（`...:201-209`）。

#### 流程 C：在 plugin-dev 中的适配分叉
- `skill-development/SKILL.md` 显式说明 plugin 场景不走 `init_skill.py`，而是在插件 `skills/` 下手工建结构（`plugins/plugin-dev/skills/skill-development/SKILL.md:137-147`）。
- 同文件说明插件技能随插件分发，不需要独立 zip（`plugins/plugin-dev/skills/skill-development/SKILL.md:278-281`）。

结论：`skill-creator-original.md` 是“通用流程基线”，plugin-dev 使用的是“插件内适配版流程”。

### 2) 关键数据结构

#### 技能目录结构协议（文档级协议）
```text
skill-name/
├── SKILL.md
└── scripts/ | references/ | assets/   (optional)
```
来源：`.../skill-creator-original.md:27-40`

#### frontmatter 契约
- `name`：技能标识。
- `description`：触发判定核心字段（何时使用此技能）。
- 该文件还带 `license` 字段（`...:1-5`），但在 plugin-dev 内不是运行时必须字段。

#### 写作协议
- 描述第三人称，正文祈使/不定式，避免二人称（`...:44,167-174`）。
- 通过语言规范降低触发与执行歧义。

### 3) 关键命令

#### 初始化命令（通用 skill 生态）
```bash
scripts/init_skill.py <skill-name> --path <output-directory>
```
作用：生成目录、模板 frontmatter、示例资源。

#### 打包命令（通用 skill 生态）
```bash
scripts/package_skill.py <path/to/skill-folder>
scripts/package_skill.py <path/to/skill-folder> ./dist
```
作用：先验证，再产出 `<skill-name>.zip`。

#### 本仓库实况
- 在当前仓库中检索不到 `scripts/init_skill.py` / `scripts/package_skill.py`。
- 根目录 `scripts/` 仅包含 issue 生命周期相关脚本（如 `scripts/sweep.ts` 等），说明上述命令在此仓库不可直接执行。

### 4) 协议与执行链（调用关系）

1. `/plugin-dev:create-plugin` 在组件实现阶段要求“创建技能时加载 skill-development skill”（`plugins/plugin-dev/commands/create-plugin.md:157-180`）。
2. `skill-development/SKILL.md` 再把完整原始方法论导向 `references/skill-creator-original.md`（`plugins/plugin-dev/skills/skill-development/SKILL.md:618-620`）。
3. `skill-reviewer` 与 `plugin-validator` 把方法论变成质量检查清单（`plugins/plugin-dev/agents/skill-reviewer.md:47-105`，`plugins/plugin-dev/agents/plugin-validator.md:98-106`）。

## 关键代码路径与文件引用

### A. 目标对象
1. `plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md`
- 研究对象本体；定义通用 skill-creator 方法与命令。

### B. 直接上下文（调用方）
1. `plugins/plugin-dev/skills/skill-development/SKILL.md:146`
- 明确“与通用 skill-creator 的差异：不使用 `init_skill.py`”。
2. `plugins/plugin-dev/skills/skill-development/SKILL.md:618-620`
- 显式引用目标文件为“完整原始方法论”。
3. `plugins/plugin-dev/README.md:175-191`
- 对外说明 Skill Development 基于该方法论。
4. `plugins/plugin-dev/commands/create-plugin.md:157-180`
- 在自动化工作流里要求加载 skill-development，形成间接调用链。

### C. 下游消费者（被调用方）
1. `plugins/plugin-dev/agents/skill-reviewer.md:40-45,60-79`
- 将“触发词质量、写作风格、渐进披露”做成审查维度。
2. `plugins/plugin-dev/agents/plugin-validator.md:98-106`
- 将“SKILL frontmatter + 目录结构 + 引用有效性”做成验证动作。

### D. 参考上游（外部依赖上下文）
1. `/home/sansha/.codex/skills/.system/skill-creator/SKILL.md`
- 本机当前 `skill-creator` 正式技能实现，较目标文件多了 Core Principles、agents/openai.yaml 元数据建议、progressive disclosure 模式细化等内容。
- 说明目标文件更接近“早期/原始版本”，存在内容漂移。

## 依赖与外部交互

### 配置依赖
1. 依赖技能 frontmatter 契约（`name`、`description`）作为触发入口。
2. plugin-dev 的插件结构约束由 `create-plugin.md`、`plugin-validator.md`、`skill-development/SKILL.md` 共同补齐。

### 脚本依赖
1. 文档声称依赖 `init_skill.py`、`package_skill.py`。
2. 当前仓库不存在上述脚本，属于“文档中的外部/上游依赖”。

### 测试依赖
1. 目标文件本身没有测试脚本。
2. 在 plugin-dev 中，质量保障通过代理/清单驱动：
- `skill-reviewer`（技能质量审查）
- `plugin-validator`（插件结构校验）
3. 没有直接针对 `skill-creator-original.md` 的自动化回归测试。

### 文档依赖
1. `plugin-dev/README.md` 是对外入口文档。
2. `skill-development/SKILL.md` 是插件场景的主文档。
3. 目标文件是补充型 reference，承担“原始方法论追溯”职责。

### 外部交互边界
1. 目标文件本身不发起网络请求。
2. 若按文档执行打包，需要本地 Python 脚本与文件系统写权限。
3. 分发交互发生在“zip 打包后分享给用户”阶段，不在该文件内实现。

## 风险、边界与改进建议

### 风险 1：方法论漂移导致误导
- 现象：目标文件仍以 Claude 生态和旧流程表述（如 `license` 字段、通用打包流程），而 plugin-dev 采用插件内创建/分发模式。
- 影响：读者可能误以为在本仓库可直接运行 `init_skill.py`、`package_skill.py`。
- 建议：在目标文件头部增加“适配说明”段，明确其为原始参考并链接 `skill-development/SKILL.md:137-147,278-281`。

### 风险 2：命令不可执行
- 现象：仓库无 `scripts/init_skill.py`、`scripts/package_skill.py`。
- 影响：按文档操作会失败，降低可信度。
- 建议：
1. 在目标文件命令段旁加注“上游命令，不在本仓库提供”。
2. 或在 plugin-dev 中提供兼容 wrapper（即便只输出迁移提示）。

### 风险 3：许可声明路径不一致
- 现象：frontmatter 指向 `LICENSE.txt`，目标目录未提供对应文件。
- 影响：许可证信息可追溯性弱。
- 建议：补充说明 license 来源，或在引用处注明该字段保留自上游原文。

### 风险 4：缺少针对 reference 的自动化校验
- 现象：该类文档依赖人工审读，未有“引用可达性/命令可执行性”自动检查。
- 影响：文档长期漂移难以及时发现。
- 建议：新增轻量脚本（如 `rg` 规则）检查：
1. reference 中命令是否在仓库可定位。
2. reference 中路径是否存在。

### 边界结论
- 该文件是“知识基线文档”，不是插件运行时代码。
- 它的价值是提供可追溯方法来源；它的主要缺口是与 plugin-dev 实际流程存在认知落差。
- 在不改变原文历史价值前提下，推荐增加“本仓库适配注释”和“可执行性提示”。
