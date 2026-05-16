# plugins/frontend-design 目录研究（DIR）

## 场景与职责

`plugins/frontend-design` 是一个“纯 Skill 型”的前端设计质量增强插件。它不提供 slash command、agent、hook 或可执行脚本，而是通过 `SKILL.md` 在前端生成任务中注入明确的美学约束与实现标准，目标是避免“模板化 AI 页面风格”。

该目录在仓库中的定位与上游入口如下：
- 根 README 将 `plugins/` 作为插件能力总入口：`README.md:48-50`
- 插件总览将本插件声明为 `frontend-design` skill（自动用于前端任务）：`plugins/README.md:21`
- marketplace 注册将本插件暴露为可安装项：`.claude-plugin/marketplace.json:73-82`
- 插件自身最小元数据位于：`plugins/frontend-design/.claude-plugin/plugin.json:1-9`

目录构成非常精简（仅 3 个文件）：
- `plugins/frontend-design/.claude-plugin/plugin.json`
- `plugins/frontend-design/README.md`
- `plugins/frontend-design/skills/frontend-design/SKILL.md`

职责边界：
1. 规定“先确立设计方向，再编码实现”的前端生产流程。
2. 在字体、配色、动效、空间构图、背景细节上提供硬约束与反模式清单。
3. 明确输出必须是“可运行生产代码”（HTML/CSS/JS/React/Vue 等），而不是设计描述文本。

## 功能点目的

### 1) 插件可发现与可安装
- marketplace 条目声明 `name/version/description/source/category`，使其成为 `/plugin` 体系下的可发现组件：`.claude-plugin/marketplace.json:73-82`
- `plugin.json` 补充本地插件标识与作者信息：`plugins/frontend-design/.claude-plugin/plugin.json:2-8`

目的：让该能力不仅存在于文档，还具备可注册、可加载、可分发的插件形态。

### 2) Skill 自动触发（前端场景）
- `SKILL.md` frontmatter 明确触发语义：当用户请求构建 web component/page/application 时启用：`plugins/frontend-design/skills/frontend-design/SKILL.md:2-4`
- 插件总览也描述了“Auto-invoked for frontend work”：`plugins/README.md:21`

目的：减少用户显式指定技能的心智负担，把设计质量约束默认嵌入前端任务。

### 3) 设计决策先行
- 要求编码前先明确 `Purpose/Tone/Constraints/Differentiation` 四要素：`plugins/frontend-design/skills/frontend-design/SKILL.md:13-17`
- 强调“清晰概念方向 + 精准执行”，避免平均化风格：`plugins/frontend-design/skills/frontend-design/SKILL.md:19`

目的：把“可用 UI 生成”提升到“有明确视觉立场的产品级界面实现”。

### 4) 前端美学与实现质量约束
- 字体：避免 Arial/Inter 等泛化字体，要求有辨识度组合：`plugins/frontend-design/skills/frontend-design/SKILL.md:30`
- 色彩：要求主题一致、使用 CSS 变量、强调主次对比：`plugins/frontend-design/skills/frontend-design/SKILL.md:31`
- 动效：强调高影响力编排（如 staggered reveal），优先 CSS，React 可用 Motion：`plugins/frontend-design/skills/frontend-design/SKILL.md:32`
- 构图与背景：鼓励非模板化布局、纹理/渐变/层次细节：`plugins/frontend-design/skills/frontend-design/SKILL.md:33-35`
- 明确禁用“AI 套路审美”：`plugins/frontend-design/skills/frontend-design/SKILL.md:36-38`

目的：用可执行设计约束降低“看起来都一样”的前端输出风险。

### 5) 文档化使用入口
- README 提供自然语言示例请求（dashboard/landing/settings panel）：`plugins/frontend-design/README.md:16-20`
- README 外链到 Frontend Aesthetics Cookbook：`plugins/frontend-design/README.md:24-27`

目的：为用户和维护者提供快速理解与可复用触发语句。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用链）

1. 仓库入口发现插件能力
- 用户从仓库 README 进入插件体系：`README.md:48-50`

2. marketplace 注册到具体目录
- `source: "./plugins/frontend-design"` 将条目映射到本目录：`.claude-plugin/marketplace.json:80`

3. 插件元数据识别
- 读取 `.claude-plugin/plugin.json` 获取插件基本身份信息：`plugins/frontend-design/.claude-plugin/plugin.json:2-8`

4. Skill 组件发现与触发
- 插件结构规范约定 `skills/*/SKILL.md` 自动发现：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:166-169`
- skill 触发时加载 `SKILL.md` 正文指导执行：`plugins/plugin-dev/skills/skill-development/SKILL.md:79-84`

5. 运行期行为
- Claude 基于 skill 指令先做设计方向决策，再生成可运行前端代码（HTML/CSS/JS/React/Vue）：`plugins/frontend-design/skills/frontend-design/SKILL.md:21`

### B. 数据结构与协议

1. 插件元数据协议（JSON）
- 文件：`plugins/frontend-design/.claude-plugin/plugin.json`
- 字段：`name/version/description/author`
- 作用：插件标识、版本、展示描述、作者信息。

2. Skill 协议（Markdown + YAML frontmatter）
- 文件：`plugins/frontend-design/skills/frontend-design/SKILL.md`
- frontmatter 字段：`name/description/license`：`plugins/frontend-design/skills/frontend-design/SKILL.md:2-4`
- body：分为 Design Thinking 与 Frontend Aesthetics Guidelines 两层规则：`plugins/frontend-design/skills/frontend-design/SKILL.md:11-42`

3. progressive disclosure 机制（上下文加载协议）
- metadata 常驻、正文按触发加载、资源按需加载：`plugins/plugin-dev/skills/skill-development/SKILL.md:79-84`
- 当前插件仅有 `SKILL.md`，未定义 `references/`、`scripts/`、`assets/`（即只有前两层，没有第三层资源）。

### C. 命令与执行特征

- 本插件无自定义 slash command（无 `commands/`）
- 无 agent（无 `agents/`）
- 无 hook（无 `hooks/`）
- 无目录内自动化脚本（无 `scripts/`）

因此它的“执行协议”本质是：`frontmatter 触发语义 + SKILL 正文约束`，由 Claude 在用户任务上下文中解释执行。

## 关键代码路径与文件引用

### 目录内核心文件
- 插件元数据：`plugins/frontend-design/.claude-plugin/plugin.json:1-9`
- 用户文档与示例：`plugins/frontend-design/README.md:1-31`
- 规则主体：`plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`

### 上游调用方（谁让它生效）
- 仓库插件入口：`README.md:48-50`
- 插件目录总览（前端 skill 声明）：`plugins/README.md:21`
- marketplace 注册项：`.claude-plugin/marketplace.json:73-82`
- 插件结构与自动发现规范：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:22-33,166-169`

### 下游被调用方（它影响谁）
- Claude 的前端代码生成决策过程（视觉方向、布局、动效、字体等）
- 用户代码仓中的前端实现文件（HTML/CSS/JS/框架组件），由任务执行时实际写入
- 外部参考资料：Frontend Aesthetics Cookbook（README 外链）：`plugins/frontend-design/README.md:26`

### 配置、测试、脚本、文档现状
- 配置：`plugin.json` + `SKILL.md` frontmatter
- 测试：目录内无 `test/spec` 文件
- 脚本：目录内无可执行脚本
- 文档：`README.md`（用途与示例）+ `SKILL.md`（执行规则）

## 依赖与外部交互

### 1) 运行时依赖
- Claude Code 插件加载与 skill 调度机制（目录发现 + frontmatter 匹配）
- 用户任务上下文（前端需求描述）作为触发条件输入

### 2) 协议依赖
- 依赖插件标准目录与 manifest 位置约定（`.claude-plugin/plugin.json`、`skills/*/SKILL.md`）：
  - `plugins/README.md:49-60`
  - `plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-43,166-169`

### 3) 外部交互
- README 指向外部 cookbook 文档（GitHub 链接），为提示工程提供扩展材料：`plugins/frontend-design/README.md:24-27`
- 插件本身不直接调用网络 API；外部交互主要是“文档参考级”，不是“运行时请求级”。

### 4) 与其他插件/规则的关系
- 该 skill 与仓库系统级“前端设计避免 AI 套路”导向一致（都强调字体/配色/动画/背景的差异化策略），但本目录内没有与其他插件的硬连接配置。
- 从维护角度看，它属于“知识约束层”，可与命令型/Hook 型插件叠加使用，但当前不存在显式冲突解决机制。

## 风险、边界与改进建议

### 风险与边界

1. 规则强、可验证性弱
- 当前规则大量是审美与策略约束（如“避免通用字体/套路布局”），但缺少自动化验证脚本，难以客观判定执行达标。

2. 元数据与市场信息可能漂移
- marketplace 中作者邮箱仅保留单邮箱：`.claude-plugin/marketplace.json:76-79`
- 插件 `plugin.json` 中作者邮箱为逗号分隔双邮箱：`plugins/frontend-design/.claude-plugin/plugin.json:6-7`
- 两处维护可能长期不一致。

3. `license` 字段引用未落地文件
- `SKILL.md` frontmatter 声明 `license: Complete terms in LICENSE.txt`：`plugins/frontend-design/skills/frontend-design/SKILL.md:4`
- 但当前目录无 `LICENSE.txt`（目录仅 3 文件），存在文档指向失效风险。

4. 无 references/scripts 导致扩展性受限
- 目前没有 `references/`（例如可访问性清单、动效性能红线）和 `scripts/`（例如质量检查脚本），后续规则增量只能继续堆在 `SKILL.md`，可维护性会下降。

5. 与其他前端提示规范存在语义重叠风险
- 仓库内其他迁移/提示资料也包含“前端质量”段落，长期并行维护可能出现“同主题不同表述”的策略漂移。

### 改进建议

1. 增加可执行质量护栏
- 在 skill 目录补充 `references/`（如 a11y 清单、性能预算、动画降级策略）
- 增加轻量 `scripts/`（如对字体 fallback、`prefers-reduced-motion`、对比度与语义标签进行静态检查）

2. 统一元数据单一来源
- 约定以 `plugin.json` 为源，marketplace 由脚本生成或校验，减少作者/描述漂移。

3. 修复 license 引用
- 若需要声明条款，补充 `LICENSE.txt`；若无单独条款，删除或改写 frontmatter 的 `license` 字段，避免无效引用。

4. 建立最小回归样例
- 增加 `examples/`（不同风格输入与预期输出要点），用于人工/半自动回归，降低规则调整后风格退化风险。

5. 为“创造性”增加边界条件
- 在 `SKILL.md` 增补可访问性、响应式、性能预算的硬约束优先级，确保“视觉大胆”不压过“可用性与可维护性”。
