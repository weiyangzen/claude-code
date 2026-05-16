# documentation-patterns.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md` 是 command-development 中的“文档工程规范层”。它面向两个对象：

1. 命令使用者：需要快速理解用途、参数、错误与示例。
2. 命令维护者：需要版本历史、迁移说明、依赖与维护备注。

该文件把“命令可运行”扩展为“命令可持续维护”，并作为 `README/SKILL` 的深层参考文档被下钻使用。

## 功能点目的

1. 自文档化模板（`11-115`）
- 把 PURPOSE/USAGE/ARGUMENTS/EXAMPLES/REQUIREMENTS/TROUBLESHOOTING/CHANGELOG 结构内嵌到命令文件。

2. 行内解释与决策点注释（`116-225`）
- 降低复杂命令阅读成本，明确“为什么此处需要人工确认”。

3. 帮助系统模式（`227-303`）
- 为复杂命令提供内建 `help` 和上下文帮助。

4. 错误信息规范（`304-391`）
- 错误不仅报错，还提供可执行恢复路径。

5. 示例驱动文档（`393-509`）
- 先例后理，提升学习速度。

6. 维护文档与发布 checklist（`510-739`）
- 把版本迁移、折旧、README 质量要求显式化。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 命令文档结构协议

模板把命令分为两层：

1. frontmatter（运行元数据）。
2. HTML 注释区（维护元数据）+ 实际执行指令体。

维护元数据字段示例（`21-60,512-555,561-599`）：

1. `VERSION/LAST UPDATED/AUTHOR`
2. `CHANGELOG/MIGRATION NOTES/DEPRECATION WARNINGS/KNOWN ISSUES`
3. `DEPENDENCIES/TESTING/RELATED FILES/FUTURE IMPROVEMENTS`

### 2) 帮助与错误处理流程

1. 帮助分支：识别 `help/-h/--help` 后直接输出 usage/subcommands/examples（`241-267`）。
2. 参数缺失错误：输出错误 + usage + 示例 + 重试建议（`315-329`）。
3. 文件缺失错误：输出原因分类 + 快速排查命令（`331-347`）。
4. 失败恢复：给出分步恢复剧本（`378-384`）。

### 3) README 配套协议

文档要求命令具备 companion README（`601-687`），覆盖：

1. 安装方式（含 `/plugin install ...`）。
2. 参数说明。
3. 典型与高级示例。
4. 配置文件格式（`.claude/*.local.md`）。
5. 故障排查与支持渠道。

## 关键代码路径与文件引用

核心文件：

1. `plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:1-739`

关键调用方：

1. `plugins/plugin-dev/skills/command-development/README.md:105`
2. `Docs/researches/plugins/plugin-dev/skills/command-development/references/current_folder_research.md:67-71,111,168`
3. `Docs/researches/CHANGELOG.md_research.md:124-125`（引用其 changelog 模板章节）

关联实现路径：

1. `plugins/plugin-dev/commands/create-plugin.md:183-191`（创建命令时需要文档化 frontmatter 与说明）
2. `plugins/plugin-dev/skills/command-development/examples/*.md`（可作为该文模板落地对象）

## 依赖与外部交互

1. 依赖 Markdown/frontmatter 约定与 slash command 渲染行为。
2. 依赖外部支持渠道占位（Issue URL、Docs URL、Support 邮箱）。
3. 与 `frontmatter-reference.md` 形成强依赖：
- 文档化字段必须与 frontmatter 真实行为一致。
4. 与 `marketplace-considerations.md` 联动：
- 文档质量直接影响分发与用户留存。

## 风险、边界与改进建议

### 风险

1. 模板中固定日期与版本（如 `2025-01-15`）容易被直接复制，导致发布信息失真。
2. `MAINTENANCE NOTES` 中列出的测试路径/脚本常是示例占位，不一定真实存在。
3. 注释文档与实际命令逻辑容易漂移，时间久后形成“文档债务”。

### 边界

1. 本文件定义文档模式，不执行自动校验。
2. 不保证 README/注释与代码同步，只提供治理建议和 checklist。

### 改进建议

1. 增加文档 lint 规则：检查 `VERSION/LAST UPDATED` 与 changelog 是否同步。
2. 把模板中的固定时间/版本改成占位符变量（`${DATE}`、`${VERSION}`）。
3. 为 README checklist 提供可执行脚本（例如扫描缺失章节和失效链接）。
4. 在 `create-plugin` 工作流中加入“文档一致性检查”步骤，避免仅创建而不维护。
