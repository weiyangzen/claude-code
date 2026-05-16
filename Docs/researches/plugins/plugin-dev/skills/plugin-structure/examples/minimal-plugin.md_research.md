# plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md 研究

## 场景与职责
`minimal-plugin.md` 是 `plugin-structure` 技能中的“最小可运行模板”，职责不是提供复杂能力，而是证明 Claude Code 插件的最小闭环：

- 仅需 `.claude-plugin/plugin.json` 的 `name` 必填字段，即可被识别为插件（`minimal-plugin.md:17-23`，`plugin-structure/SKILL.md:50-56`，`references/manifest-reference.md:15-36`）。
- 在默认 `commands/` 目录放置一个带 frontmatter 的 Markdown 命令文件，即可通过自动发现机制注册为 slash command（`minimal-plugin.md:25-46`，`plugin-structure/SKILL.md:112-135,339-349`）。
- 通过“无 hooks、无 agents、无 skills、无 MCP”强调基础目录约定优先于显式配置（`minimal-plugin.md:64-67`）。

从调用关系看，它主要被以下文档消费：

- `plugins/plugin-dev/skills/plugin-structure/README.md` 把它定义为三档示例中的入门档（`README.md:52-58`）。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md` 的 Minimal Plugin 模式与其结构一致（`SKILL.md:406-415`）。

## 功能点目的
该文件的功能点很少，但目的清晰：

1. 定义最小目录结构
- 展示 `hello-world/.claude-plugin/plugin.json + commands/hello.md`（`minimal-plugin.md:7-13`）。

2. 演示最小 manifest
- 只保留 `{"name":"hello-world"}`，映射到 manifest 参考中的最小案例（`minimal-plugin.md:19-23`，`manifest-reference.md:451-462`）。

3. 演示命令 frontmatter 协议
- `name: hello`、`description: ...` 的 YAML frontmatter 作为命令注册元数据（`minimal-plugin.md:28-31`，`plugin-structure/SKILL.md:124-133`）。

4. 演示自动发现后可调用
- 用 `/hello` 的交互片段说明“无需额外配置即可执行”（`minimal-plugin.md:48-60`）。

5. 给出扩展路线
- 后续按目录增量扩展 commands/agents/hooks（`minimal-plugin.md:76-83`）。

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 发现与加载关键流程
- 插件启用时，Claude Code 先读取 `.claude-plugin/plugin.json`（`SKILL.md:343`）。
- 在默认路径扫描 `commands/` 目录下 `.md` 文件（`SKILL.md:344`）。
- 解析 frontmatter，注册 `/hello` 命令（`SKILL.md:124-135`）。
- 会话中用户输入 `/hello` 后触发该命令的指令文本（`minimal-plugin.md:52-55`）。

### 2) 核心数据结构
- `plugin.json`（最小形态）
```json
{
  "name": "hello-world"
}
```
来源：`minimal-plugin.md:19-23`。

- `commands/hello.md` frontmatter + body
```yaml
name: hello
description: Prints a friendly greeting message
```
来源：`minimal-plugin.md:28-31`。

### 3) 协议与约束
- 命令文件遵循“Markdown + YAML frontmatter”协议（`SKILL.md:113-114,124-133`）。
- `name` 字段应满足 kebab-case 规范（此示例的插件名 `hello-world` 合规）（`manifest-reference.md:26-36`）。
- 没有 hooks/MCP，因此不涉及额外 JSON 协议或外部工具进程。

## 关键代码路径与文件引用
核心路径：

- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:7-13`：最小目录结构。
- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:19-23`：最小 manifest。
- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:28-46`：命令定义与执行指令。
- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:64-67`：最小模式设计要点。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-349`：自动发现机制总流程。
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:451-462`：最小 manifest 对照样例。
- `plugins/plugin-dev/skills/plugin-structure/README.md:52-58`：该示例在技能文档中的定位。

## 依赖与外部交互
### 仓库内依赖
- 依赖 `plugin-structure` 技能定义的目录约定与自动发现规则（`SKILL.md:20-45,339-356`）。
- 被 README 与其他研究文档作为“基线模板”引用（`README.md:52-58`）。

### 外部交互
- 本示例不包含 shell 脚本、hook 命令或 MCP server。
- 运行时仅与 Claude Code 命令分发机制交互（`/hello`）。

## 风险、边界与改进建议
### 风险与边界
1. 可执行逻辑完全由自然语言描述驱动
- `hello.md` 只描述“输出消息+时间戳”，没有脚本与结构化约束，行为依赖模型解释，稳定性低于脚本化命令。

2. 缺少质量与安全护栏
- 无 `hooks/hooks.json`，无法在写入、停止等阶段执行校验流程。

3. 缺少可观测性与可维护元数据
- 无版本、作者、仓库、许可证等信息，不适合分发或团队治理（`manifest-reference.md:42-208`）。

### 改进建议
1. 补充推荐 metadata
- 至少增加 `version`、`description`、`author`、`license`，便于发布与审计。

2. 引入轻量 Stop hook
- 增加一个 `Stop` prompt/command hook，用于完成前的基本一致性检查（参照 `hook-development/SKILL.md:181-209`）。

3. 保持“最小模板”定位不变
- 不建议直接把此文件改成“全能模板”；应保持教学最小闭环，并在文档中明确升级路径到 `standard-plugin.md`。
