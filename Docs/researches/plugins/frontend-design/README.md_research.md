# plugins/frontend-design/README.md 研究

## 场景与职责

`plugins/frontend-design/README.md` 是 `frontend-design` 插件的人类可读入口文档，职责是用最短路径说明该插件“做什么、怎么触发、输入示例、扩展学习入口、作者信息”。

- 面向对象：插件使用者与维护者，而非运行时执行器。
- 在插件链路中的位置：
  - marketplace 将 `frontend-design` 映射到插件目录（`.claude-plugin/marketplace.json:73-81`）。
  - 插件 manifest 声明基础元数据（`plugins/frontend-design/.claude-plugin/plugin.json:1-9`）。
  - 真正运行时行为由 skill 规则驱动（`plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`）。
  - README 承担“文档入口与示例提示”，不是执行逻辑载体（`plugins/frontend-design/README.md:1-31`）。

## 功能点目的

README 的功能点可拆为 5 个目的明确的文档段落：

1. 插件定位声明（`README.md:1-4`）
   - 给出“distinctive, production-grade frontend interfaces”定位，强调避免通用 AI 审美。
2. 能力说明（`README.md:5-13`）
   - 指定自动用于 frontend 工作，并列举输出目标：大胆审美、字体配色、动效细节、上下文感知实现。
3. 输入示例（`README.md:14-22`）
   - 通过 3 条自然语言请求提示用户“如何触发并使用该 skill”。
4. 外部学习入口（`README.md:24-27`）
   - 引导到 cookbook，补充更完整的提示工程范式。
5. 作者归属（`README.md:28-31`）
   - 提供维护者身份与联系方式。

## 具体技术实现（关键流程/数据结构/协议/命令）

虽然目标文件是文档，但它参与了插件能力的“声明-发现-激活”协作链路，关键实现如下。

### 1) 发现与激活流程（README 所在上下文）

1. 插件被注册：marketplace 条目将名称 `frontend-design` 指向 `./plugins/frontend-design`（`.claude-plugin/marketplace.json:73-81`）。
2. 插件被识别：Claude Code 按规范读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`；`plugins/frontend-design/.claude-plugin/plugin.json:1-9`）。
3. 组件被发现：扫描 `skills/` 下包含 `SKILL.md` 的子目录（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:346`）。
4. skill 被激活：任务上下文匹配 `SKILL.md` 的 frontmatter `description` 时加载正文规则（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:25`；`plugins/frontend-design/skills/frontend-design/SKILL.md:2-4`）。
5. README 参与方式：为用户提供示例输入与预期输出风格，间接影响触发质量（`plugins/frontend-design/README.md:16-22`）。

### 2) 关键数据结构

- 插件 manifest（JSON）：
  - `name/version/description/author`（`plugins/frontend-design/.claude-plugin/plugin.json:2-8`）。
  - 规范要求路径必须是 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- skill 协议（YAML frontmatter + Markdown body）：
  - `name`、`description` 决定触发时机（`plugins/plugin-dev/skills/skill-development/SKILL.md:44`）。
  - 正文定义前端设计决策与实现约束（`plugins/frontend-design/skills/frontend-design/SKILL.md:11-42`）。
- README 输入协议（自然语言示例）：
  - 三条引导式 prompt（`plugins/frontend-design/README.md:16-20`）作为“任务描述模板”。

### 3) 命令与脚本视角

- `plugins/frontend-design` 目录仅含 `plugin.json`、`README.md`、`SKILL.md`，无命令、无 agent、无 hooks、无可执行脚本。
- 目录内也未发现测试文件（`find plugins/frontend-design ...` 结果为空）。
- 因此该插件是“纯 Skill 文档驱动插件”，运行行为全部依赖 Claude Code 对 manifest 与 skill frontmatter 的自动发现机制。

## 关键代码路径与文件引用

核心对象与上下游关系如下：

- 目标文件：`plugins/frontend-design/README.md:1-31`
- 直接被调用方（运行规则主体）：`plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`
- 配置声明：`plugins/frontend-design/.claude-plugin/plugin.json:1-9`
- 注册入口：`.claude-plugin/marketplace.json:73-81`
- 插件目录索引：`plugins/README.md:21`
- 自动发现规范：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`
- 生命周期与 skill 激活条件：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-27`
- skill 触发元数据规范：`plugins/plugin-dev/skills/skill-development/SKILL.md:44,79-84`
- manifest 路径与字段约束：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,15-31`
- 外部文档依赖（README 外链）：`plugins/frontend-design/README.md:26`

## 依赖与外部交互

### 内部依赖

- 对 `SKILL.md` 的依赖：README 的“自动使用”叙述成立，依赖 `skills/frontend-design/SKILL.md` 的触发描述质量与正文约束。
- 对 plugin manifest 的依赖：若 `.claude-plugin/plugin.json` 缺失或不合法，插件无法被识别。
- 对 marketplace 条目的依赖：目录映射错误会导致安装/发现链断裂。

### 外部交互

- README 仅有一个外部交互点：指向 GitHub 上的 Frontend Aesthetics Cookbook 链接（知识参考，不是运行时 API 调用）。
- 无网络 API、无本地命令调用、无 MCP server、无 hooks 事件交互。

### 测试与脚本状态

- 插件目录内未提供测试或验证脚本；质量保障主要依赖文档规范一致性与人工评审。

## 风险、边界与改进建议

1. 文档承诺与运行行为存在漂移风险
   - README 声明“Claude automatically uses this skill for frontend work”（`README.md:7`），但真实命中取决于 `SKILL.md` 描述匹配质量与任务上下文，非强制调用。
   - 建议：在 README 增加“auto-invoked when context matches frontend tasks”的条件性表述。

2. 元数据口径不一致风险
   - 作者信息在三个位置存在细微差异：
     - README 双作者双邮箱（`README.md:30-31`）
     - plugin.json 用逗号拼接邮箱（`plugin.json:6-7`）
     - marketplace 仅保留一个邮箱（`marketplace.json:77-79`）
   - 建议：统一作者与联系方式格式，降低维护歧义。

3. 许可证引用边界不清晰
   - skill frontmatter 使用 `license: Complete terms in LICENSE.txt`（`SKILL.md:4`），但插件目录未见对应文件。
   - 建议：补齐 `plugins/frontend-design/LICENSE.txt` 或引用仓库现有 `LICENSE.md`。

4. 测试缺位导致回归不可见
   - 无自动化测试校验 README 示例是否仍能稳定触发目标 skill。
   - 建议：在仓库文档质量流程中加入“README 示例 prompt 与 skill 描述一致性”检查。

5. 外链可用性与版本漂移风险
   - README 依赖外部 cookbook URL，链接失效或内容变更会影响使用者学习路径。
   - 建议：补充仓库内简版摘要或镜像要点，降低外链单点风险。

6. 与其他前端规则源可能发生重复维护
   - 仓库内其他迁移/提示资料也包含前端审美建议，长期并行可能漂移。
   - 建议：抽取共享片段或建立“规范主源”引用关系，减少分叉。
