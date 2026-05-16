# plugins/plugin-dev/skills/command-development/README.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/README.md` 是 `command-development` 技能目录的导航入口文档，承担“技能说明书 + 快速操作手册 + 资源索引”三重职责。

在 `plugin-dev` 体系中的定位：

1. 作为 `Command Development` 能力域的总览说明，对外解释该技能覆盖哪些命令开发问题（`plugins/plugin-dev/README.md:135-152`）。
2. 作为目录内文档路由层，把使用者从高层概览引导到核心规范 `SKILL.md`、深度 references 和 examples（`plugins/plugin-dev/skills/command-development/README.md:19-111`）。
3. 作为快速抄写模板，为用户/维护者提供 frontmatter、参数、文件引用、bash 注入的速查片段（`plugins/plugin-dev/skills/command-development/README.md:113-202`）。

它不直接被运行时执行，不参与命令调用链，但直接影响技能可发现性、学习成本和规范一致性。

## 功能点目的

### 1. 统一技能认知边界

README 在开头明确该技能关注“slash command 结构、frontmatter、参数、@ 文件引用、!\`bash\`、命名空间、插件特性与校验”（`plugins/plugin-dev/skills/command-development/README.md:7-17`）。

目的：防止用户把命令开发问题误投到其他技能（如 `hook-development` 或 `agent-development`）。

### 2. 提供渐进披露入口

README 将知识分层为：

1. `SKILL.md`（核心原则与常用模式）
2. `references/`（细则与扩展策略）
3. `examples/`（可复用模板）

对应位置：`plugins/plugin-dev/skills/command-development/README.md:94-111`。

目的：减少一次性加载体积，同时保留深度。

### 3. 提供“可立即复制”的快速参考

README 给出可直接迁移的命令骨架（frontmatter、参数、文件引用、bash）与开发流程步骤（设计→创建→补 frontmatter→测试→迭代），位置见：

- 快速参考：`plugins/plugin-dev/skills/command-development/README.md:113-202`
- 开发流程：`plugins/plugin-dev/skills/command-development/README.md:204-232`
- 最佳实践摘要：`plugins/plugin-dev/skills/command-development/README.md:233-241`

目的：降低首次创建命令的路径复杂度。

### 4. 暴露维护契约

README 末尾提供维护约束（保持 `SKILL.md` 精简、细节下沉到 references、示例可运行）和版本历史（`plugins/plugin-dev/skills/command-development/README.md:256-272`）。

目的：让后续维护者知道“应该改哪里、如何演进”。

## 具体技术实现（关键流程/数据结构/协议/命令）

README 虽是文档，但内部仍定义了关键协议和工作流。

### A. 文档驱动流程（使用时序）

1. 识别触发问题
- 由“create slash command / command frontmatter / define arguments”等触发词进入本技能（`plugins/plugin-dev/skills/command-development/README.md:82-93`）。

2. 快速建模
- 通过“File Format + Frontmatter 表 + Common Patterns”先搭建命令草案（`plugins/plugin-dev/skills/command-development/README.md:113-202`）。

3. 进入精细阶段
- 根据问题类型下钻 `SKILL.md` 与 `references`（`plugins/plugin-dev/skills/command-development/README.md:94-111`）。

4. 落地与回归
- 按 README 的 5 步开发流程执行，并依据“Best Practices Summary”校验（`plugins/plugin-dev/skills/command-development/README.md:204-241`）。

### B. 关键协议（README 中暴露的最小契约）

1. 命令文件协议（Markdown + 可选 YAML frontmatter）
- 示例见 `plugins/plugin-dev/skills/command-development/README.md:117-128`。

2. frontmatter 字段协议
- 字段集合：`description / allowed-tools / model / argument-hint / disable-model-invocation`（`plugins/plugin-dev/skills/command-development/README.md:150-157`）。

3. 参数替换协议
- `$ARGUMENTS`（整体）与 `$1/$2/$3`（位置参数）（`plugins/plugin-dev/skills/command-development/README.md:138-140`）。

4. 文件与命令注入协议
- 文件：`@path`
- bash：`!\`command\``（README 速查说明位于 `plugins/plugin-dev/skills/command-development/README.md:142-147`）。

### C. 关键目录映射（位置语义）

README 统一了三类命令位置：

1. 项目级：`.claude/commands/`
2. 用户级：`~/.claude/commands/`
3. 插件级：`plugin-name/commands/`

见 `plugins/plugin-dev/skills/command-development/README.md:130-135`。

## 关键代码路径与文件引用

### 目标文件

1. `plugins/plugin-dev/skills/command-development/README.md`
- 技能入口与速查手册。

### 直接依赖（被调用方）

1. `plugins/plugin-dev/skills/command-development/SKILL.md`
- README 指定的核心正文（`plugins/plugin-dev/skills/command-development/README.md:21-42`）。

2. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md`
3. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md`
- README 在“References”显式列出（`plugins/plugin-dev/skills/command-development/README.md:43-59`）。

4. `plugins/plugin-dev/skills/command-development/references/interactive-commands.md`
5. `plugins/plugin-dev/skills/command-development/references/advanced-workflows.md`
6. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md`
7. `plugins/plugin-dev/skills/command-development/references/documentation-patterns.md`
8. `plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md`
- 在“Progressive Disclosure”列为深度参考（`plugins/plugin-dev/skills/command-development/README.md:99-106`）。

9. `plugins/plugin-dev/skills/command-development/examples/simple-commands.md`
10. `plugins/plugin-dev/skills/command-development/examples/plugin-commands.md`
- 在“Examples”与“Progressive Disclosure”中引用（`plugins/plugin-dev/skills/command-development/README.md:60-80,107-109`）。

### 调用方（谁在引用该技能能力）

1. `plugins/plugin-dev/README.md`
- 将 Command Development 作为 toolkit 第 5 项技能能力公开（`plugins/plugin-dev/README.md:13,135-152`）。

2. `plugins/plugin-dev/commands/create-plugin.md`
- 在 Phase 5 明确要求“Commands: Load command-development skill”（`plugins/plugin-dev/commands/create-plugin.md:157-164,183-191`）。

3. `plugins/plugin-dev/skills/skill-development/SKILL.md`
- 将 `../command-development/` 作为“示例技能”推荐学习对象（`plugins/plugin-dev/skills/skill-development/SKILL.md:608-614`）。

## 依赖与外部交互

### 依赖类型

1. 文档依赖
- README 本身不执行逻辑，但通过链接依赖 `SKILL.md + references + examples`。

2. Claude Code 语义依赖
- README 的速查内容隐含依赖 Claude Code 命令系统语法：frontmatter 字段、`$ARGUMENTS/$1`、`@file`、`!\`bash\``。

3. 插件体系依赖
- 依赖 `plugin-dev` 的编排命令与技能触发机制，尤其是 `/plugin-dev:create-plugin` 的“按组件加载技能”流程（`plugins/plugin-dev/commands/create-plugin.md:157-164,183-191`）。

### 外部交互面

README 无直接 I/O、无脚本执行、无测试脚本调用。它通过“规范传播”间接影响：

1. 命令作者编写的 frontmatter 与权限边界。
2. 命令运行时是否按预期读取文件、执行 bash、处理参数。
3. 插件分发场景下命令文档与可维护性质量。

## 风险、边界与改进建议

### 风险 1（中）：References 显式列表与“渐进披露列表”不一致

现状：
- “References”小节仅详细展开 2 个文件（frontmatter、plugin-features）（`plugins/plugin-dev/skills/command-development/README.md:43-59`）。
- 但“Progressive Disclosure”声明还有 5 个深度参考（interactive/advanced/testing/documentation/marketplace）（`plugins/plugin-dev/skills/command-development/README.md:99-106`）。

影响：入口读者易误判参考文档全貌。

建议：在 “References” 主体中补齐全部 7 个参考文档的职责摘要。

### 风险 2（中）：状态段落与当前目录成熟度存在漂移

现状：
- README 标注 advanced/testing/documentation/marketplace “in progress”（`plugins/plugin-dev/skills/command-development/README.md:250-254`）。
- 实际对应文件已存在且篇幅较完整（`plugins/plugin-dev/skills/command-development/references/*.md`）。

影响：对外信号偏保守，降低文档可信度。

建议：把状态改成“已提供 + 后续增强项”。

### 风险 3（低）：快速参考中的 bash 语法展示可读性较弱

现状：
- `- !\`command\`` 的展示在 markdown 阅读器中不够直观（`plugins/plugin-dev/skills/command-development/README.md:146`）。

影响：新手容易误抄。

建议：在示例块内统一展示完整格式，并附一条可执行示例（如 `!\`git status\``）。

### 风险 4（低）：字数统计与实际内容可能失配

现状：
- README 标注 `SKILL.md (~2,470 words)`、references 总量等（`plugins/plugin-dev/skills/command-development/README.md:21,98-107`），但该数据随版本演进容易过期。

建议：改为“约数区间 + 自动统计脚本来源”或移除具体字数。

### 边界说明

1. README 只提供“规范与导航”，不提供可执行验证逻辑。
2. 实际正确性由命令运行时行为、validator 规则、以及 references/examples 的一致性共同决定。
