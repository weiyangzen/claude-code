# FILE 研究：plugins/plugin-dev/skills/hook-development/scripts/README.md

## 场景与职责

该文件是 `hook-development` 技能中 3 个工具脚本的统一入口文档，面向“正在开发插件 hooks 的作者”。它的职责不是定义协议本身，而是把常见开发路径整理成可执行流程：

- 先静态检查脚本（`hook-linter.sh`）
- 再构造输入并运行脚本（`test-hook.sh`）
- 最后校验 `hooks.json`（`validate-hook-schema.sh`）

上游调用方主要是文档和流程：

- `plugins/plugin-dev/README.md:273-284` 把这三个脚本列为工具链
- `plugins/plugin-dev/commands/create-plugin.md:201-209,257-260` 在创建/验证阶段要求运行 schema + test
- `plugins/plugin-dev/skills/hook-development/SKILL.md:685-709` 把该目录脚本纳入实现流程

下游被调用方是同目录 3 个脚本本体与插件中的 hook 脚本/配置文件。

## 功能点目的

`README.md` 的功能点可以分为 4 类：

1. 工具说明：分别解释 3 个脚本做什么（`README.md:5-90`）。
2. 标准流程：给出从编写到 Claude Code 中验证的 7 步流程（`README.md:92-128`）。
3. 经验提示：通过 Tips/常见问题减少误用（`README.md:130-164`）。
4. 命令模板：提供可直接复制的 CLI 示例，降低上手成本。

其核心目的是“降低 hooks 开发失败率与调试成本”，而不是提供 hooks API 全量语义（那部分由 `SKILL.md` 和 references 承担）。

## 具体技术实现（关键流程/数据结构/协议/命令）

该文档本质是“命令编排层”，实现方式是把协议约束映射为可执行命令：

1. 配置校验命令
- `./validate-hook-schema.sh path/to/hooks.json`（`README.md:9-27`）
- 对应协议对象：`hooks.json` 事件配置

2. 运行时仿真命令
- `./test-hook.sh [options] <hook-script> <test-input.json>`（`README.md:33-53`）
- 对应协议对象：hook stdin JSON、退出码语义、环境变量注入

3. 静态扫描命令
- `./hook-linter.sh <hook-script.sh> [...]`（`README.md:66-90`）
- 对应协议对象：bash 脚本约定（shebang、变量引用、stderr 输出等）

4. 端到端工作流
- 文档把三个脚本串成“写脚本 -> lint -> 造样例 -> 回放 -> 写 hooks.json -> schema 校验 -> `claude --debug` 联调”（`README.md:92-128`）。

## 关键代码路径与文件引用

核心文件：

- `plugins/plugin-dev/skills/hook-development/scripts/README.md:1-164`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:1-153`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:1-252`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1-159`

关键上下文：

- `plugins/plugin-dev/skills/hook-development/SKILL.md:64-80`（插件 hooks wrapper 格式说明）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:685-709`（工具脚本入口与推荐流程）
- `plugins/plugin-dev/README.md:273-284`（工具脚本在 toolkit 中的定位）
- `plugins/plugin-dev/commands/create-plugin.md:257-260`（验证阶段要求）
- `plugins/plugin-dev/agents/plugin-validator.md:107-115`（validator 对 hooks 的校验指引）

## 依赖与外部交互

文档间接声明了以下外部依赖：

- Shell：`bash`
- JSON 工具：`jq`（由 `test-hook.sh`、`validate-hook-schema.sh` 使用）
- 超时工具：`timeout`（由 `test-hook.sh` 使用）
- Claude CLI：`claude --debug`

对外部系统无网络调用；主要与本地文件系统交互：

- 读取 hook 脚本与 `hooks.json`
- 通过 stdin 回放测试输入
- 可读取/展示 `CLAUDE_ENV_FILE` 产物

## 风险、边界与改进建议

1. 文档与实现存在漂移风险。
- 文档默认“插件 `hooks/hooks.json` 可直接用 schema 脚本校验”（`README.md:122`），但当前 `validate-hook-schema.sh` 对常见 wrapper 结构（`{"description","hooks":{...}}`）并不兼容。
- 实测：`bash .../validate-hook-schema.sh plugins/security-guidance/hooks/hooks.json` 在遍历阶段触发 `jq: Cannot index string with number` 并退出。

2. “功能声明”与实际行为有细微不一致。
- `README.md:58` 写“Validates output JSON”，但 `test-hook.sh` 实际是“若可解析则美化输出”，并不会因输出非 JSON 而判失败（失败判定仍以退出码为主）。

3. 工作流未显式提示配置格式分歧。
- `SKILL.md` 同时出现“wrapper 格式（插件）”和“direct 格式（settings）”两种描述；README 未强调脚本当前偏向 direct 格式，易导致误判。

建议：

- 在 `README.md` 增加“当前 schema 脚本支持范围”说明（direct vs wrapper）。
- 在“Typical Workflow”第 6 步附一个 wrapper 预处理命令或明确限制。
- 在 `test-hook.sh` 章节补一句“输出 JSON 仅做可解析展示，不作为通过条件”。
