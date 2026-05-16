# plugins/plugin-dev/skills/skill-development 研究

## 场景与职责

`plugins/plugin-dev/skills/skill-development` 是 `plugin-dev` 插件中的“元技能（meta-skill）”：它不直接实现业务功能，而是定义“如何开发其他技能”。在 `plugin-dev` 的 7 个核心技能里，它被定位为第 7 个能力，负责把技能开发流程、触发词写法、内容分层和验收标准固化成可复用规范。

核心职责有三层：

1. 技能建模职责：定义 `SKILL.md + references/examples/scripts` 的结构和 frontmatter 触发规范（`plugins/plugin-dev/skills/skill-development/SKILL.md:25-40,42-67`）。
2. 流程职责：给出从“需求澄清→资源规划→结构创建→编写→验证→迭代”的完整步骤（`plugins/plugin-dev/skills/skill-development/SKILL.md:87-247`）。
3. 质量治理职责：通过写作风格约束、校验清单、反模式与 quick reference 保证技能质量一致性（`plugins/plugin-dev/skills/skill-development/SKILL.md:362-603`）。

它在插件体系中的上下文位置：

- 上游调用方（谁要求使用它）：
  - `plugin-dev` 总览文档将其列为核心技能并声明触发词与用途（`plugins/plugin-dev/README.md:175-193`）。
  - `/plugin-dev:create-plugin` 在 Phase 5 明确要求“实现 skill 组件前加载 skill-development”（`plugins/plugin-dev/commands/create-plugin.md:157-180,371-375`）。
- 下游被调用方（它依赖谁来完成落地）：
  - `skill-reviewer` agent 用于技能质量审查（`plugins/plugin-dev/skills/skill-development/SKILL.md:225-230`，`plugins/plugin-dev/agents/skill-reviewer.md:40-46,60-79`）。
  - `references/skill-creator-original.md` 作为原始方法论补充（`plugins/plugin-dev/skills/skill-development/SKILL.md:618-620`）。

## 功能点目的

1. 定义高精度触发机制
- 通过 frontmatter 的 `description` 写具体用户原话，提升技能路由命中率，避免“泛触发”。
- 位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:2-5,44,162-182`。

2. 建立 progressive disclosure 分层
- 明确“元数据常驻 + SKILL 正文按触发加载 + 资源按需加载”三层，平衡上下文开销与知识深度。
- 位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:77-85,318-360`。

3. 标准化技能创建流程
- 将技能开发拆成 6 步，强制先做场景样本与可复用资产规划，再写正文与验证，降低“先写文档后补逻辑”的返工。
- 位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:87-247`。

4. 统一写作协议
- 要求正文使用祈使/不定式，frontmatter 用第三人称，目标是让“另一实例 Claude”直接按步骤执行。
- 位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:160-170,362-414`。

5. 提供质量门禁
- 用 Validation Checklist + Common Mistakes 把结构、触发词、资源引用、测试覆盖前置成显式检查项。
- 位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:415-540`。

6. 适配插件场景（区别于通用 skill-creator）
- 明确“插件技能不走 ZIP 打包分发、直接放在 plugin 的 `skills/` 子目录并随插件交付”。
- 位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:146,249-292,278-281`。
- 对照原始参考：通用 `skill-creator` 里有 `init_skill.py` 与 `package_skill.py` 步骤（`plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:132-200`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键数据结构

A. 技能目录协议

```text
skill-name/
├── SKILL.md
└── (optional) references/ examples/ scripts/ assets/
```

- `SKILL.md` 必须包含 YAML frontmatter 的 `name` 与 `description`。
- 资源目录按需存在，不强制齐全。
- 位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:27-40,216-223,441-443`。

B. frontmatter 触发协议

```yaml
---
name: Skill Name
description: This skill should be used when the user asks to "..."
version: 0.1.0
---
```

- `description` 语义是“触发规则声明”，而非说明文案。
- 位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:162-175,384-395,453-468`。

### 2) 关键流程（运行时与开发时）

A. 开发流程（6 步）

1. 收集具体使用样例与触发语句（Step 1）
2. 从样例反推出可复用资源（Step 2）
3. 在插件内创建 skill 目录结构（Step 3）
4. 编写资源与精简 SKILL 主体（Step 4）
5. 做结构/触发/风格/引用/脚本验证（Step 5）
6. 基于真实使用迭代（Step 6）

位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:91-247,621-635`。

B. 插件加载流程（Auto-Discovery）

1. Claude Code 扫描插件 `skills/`。
2. 发现含 `SKILL.md` 的子目录。
3. 常驻加载 metadata（`name+description`）。
4. 命中触发词时加载 `SKILL.md` 正文。
5. 任务需要时再加载 `references/examples`。

位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:269-277`；补充机制说明见 `plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`。

C. 工作流编排流程（create-plugin 命令）

- Phase 5 对不同组件类型分别加载对应 skill；其中 skill 组件固定走 skill-development。
- 完成 skill 后，调用 `skill-reviewer` 做质量评审。

位置：`plugins/plugin-dev/commands/create-plugin.md:157-181,247-250,349-375`。

### 3) 关键命令与交互协议

1. 目录初始化命令（插件场景）

```bash
mkdir -p plugin-name/skills/skill-name/{references,examples,scripts}
touch plugin-name/skills/skill-name/SKILL.md
```

位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:141-144`。

2. 本地验证安装命令

```bash
cc --plugin-dir /path/to/plugin
```

位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:284-292`。

3. 质量评审触发语句（agent 协议）

```text
Review my skill and check if it follows best practices
```

位置：`plugins/plugin-dev/skills/skill-development/SKILL.md:225-230`。

### 4) 实现特征与当前目录现状

- 当前目标目录只包含：`SKILL.md` 与 `references/skill-creator-original.md`。
- 目录中未提供 `examples/`、`scripts/`、自动化测试文件；它承担的是规范定义与方法论沉淀，不承担执行工具实现。
- 证据：`plugins/plugin-dev/skills/skill-development` 目录树（2 个文件）。

## 关键代码路径与文件引用

### 目标目录（被研究对象）

- `plugins/plugin-dev/skills/skill-development/SKILL.md`
- `plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md`

### 调用方（上游）

1. `plugins/plugin-dev/README.md:175-193`
- 声明该技能的触发词、覆盖范围与资源类型。

2. `plugins/plugin-dev/commands/create-plugin.md:157-181`
- Phase 5 规定创建技能组件时必须加载 skill-development。

3. `plugins/plugin-dev/commands/create-plugin.md:247-250`
- 技能创建后调用 `skill-reviewer` 审查。

4. `plugins/README.md:24`
- 在插件总目录层面把 plugin-dev 标记为“含 7 个技能 + create-plugin 工作流”的开发工具箱。

### 被调用方（下游）

1. `plugins/plugin-dev/agents/skill-reviewer.md:40-46,60-79,99-106`
- 承接 Step 5 的技能评审，检查描述质量、分层与风格。

2. `plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-169,339-348`
- 为 skill 自动发现机制提供结构级背景约束。

3. `plugins/plugin-dev/skills/skill-development/references/skill-creator-original.md:132-200`
- 提供通用 `skill-creator` 原始流程（含初始化与打包步骤）。

4. `plugins/plugin-dev/skills/*/SKILL.md`
- 作为“学习样例”被直接引用（hook/agent/mcp/plugin-settings/command/plugin-structure），见 `plugins/plugin-dev/skills/skill-development/SKILL.md:608-615`。

### 配置、测试、脚本、文档关联

- 配置：技能本体依赖 `SKILL.md` frontmatter 协议；插件级依赖 `.claude-plugin/plugin.json` 与 `skills/` 自动发现约定（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-49,166-169`）。
- 测试：本目录无自动化测试；建议通过 `cc --plugin-dir` + 触发词对话人工验证（`plugins/plugin-dev/skills/skill-development/SKILL.md:284-292,446-450`）。
- 脚本：本目录无执行脚本；质量验证主要委托给 `skill-reviewer` agent 与跨目录工具（`plugins/plugin-dev/skills/skill-development/SKILL.md:223-230`）。
- 文档：`plugin-dev/README.md` 是入口说明，`create-plugin.md` 是编排器，`skill-creator-original.md` 是方法来源补充。

## 依赖与外部交互

### 运行时依赖

1. Claude Code 技能路由机制
- 依赖 `description` 触发匹配与技能自动发现。
- 交互方式：隐式（运行时由 Claude Code 完成加载）。

2. Claude Code Agent/Task 协作机制
- 通过 `skill-reviewer` 承接质量评审。
- 交互方式：显式（由主流程提示触发 agent）。

3. 命令行环境
- 文档推荐命令依赖 `bash` 与 Claude Code CLI（`cc --plugin-dir`）。
- 目标目录本身无 shell 脚本，因此不引入额外系统命令依赖。

### 外部文档与平台交互

- `plugins/README.md` 指向官方 plugin 文档（外部 URL），但本目录执行层面不直接发起网络调用。
- 插件分发方式在该技能中定义为“随插件安装”，不走 skill ZIP 发布协议（`plugins/plugin-dev/skills/skill-development/SKILL.md:278-281`）。

## 风险、边界与改进建议

### 风险与边界

1. 文档体量偏离自身准则
- `skill-development/SKILL.md` 约 3196 词，而文内推荐“理想 1500-2000，<3000 更佳”（`plugins/plugin-dev/skills/skill-development/SKILL.md:190,329,587-588`）。
- 风险：触发后上下文占用偏大，削弱 progressive disclosure 的收益。

2. 与 README 统计信息不一致
- README 仍写“Core SKILL.md (1,232 words)”（`plugins/plugin-dev/README.md:189-190`），与现状（3196 词）存在漂移。
- 风险：读者对加载成本和文档规模预期失真。

3. 参考文档包含仓库内不存在的通用脚本路径
- `references/skill-creator-original.md` 依赖 `scripts/init_skill.py` 与 `scripts/package_skill.py`（`.../skill-creator-original.md:138-187`），当前仓库未找到这些脚本。
- 风险：若执行者直接照搬 reference，可能走进不存在命令路径。

4. 本目录无 examples/scripts 实物
- 技能文本强调 examples/scripts 的价值，但当前目录仅有主文档与 reference。
- 风险：学习者缺少该技能自身的“可执行示例”，主要依赖跨目录样例。

5. 仓库内 plugin-dev 缺失 `.claude-plugin/plugin.json` 实体
- `plugin-structure` 要求 manifest 必须存在（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-49`），但当前 `plugins/plugin-dev` 目录未见该文件。
- 风险：该目录更像“开发教材集合”而非可直接安装插件，若用户按“可立即安装”理解会产生落差。

### 改进建议

1. 拆分并瘦身 `skill-development/SKILL.md`
- 把“长清单/错误示例/quick reference”进一步下沉到 `references/`，主文档保留流程主干与关键规则。

2. 同步 README 的资源统计
- 更新 `plugin-dev/README.md` 中 skill-development 的字数与资源描述，避免误导。

3. 给 `skill-creator-original.md` 增加“插件场景差异”警示块
- 在引用处或 reference 文件头部注明：`init_skill.py/package_skill.py` 为通用 skill 生态路径，plugin-dev 目录下不直接提供。

4. 为本目录补一个最小可执行样例
- 增加 `examples/minimal-skill-template/`（含触发词、lean SKILL、references 链接），降低新手迁移成本。

5. 加一个轻量一致性校验脚本
- 校验项建议：frontmatter 第三人称、触发词数量、字数上限、引用文件存在性，并在 `create-plugin` Phase 6 中可选调用。
