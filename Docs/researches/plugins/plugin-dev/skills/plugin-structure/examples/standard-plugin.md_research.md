# plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md 研究

## 场景与职责
`standard-plugin.md` 是三档示例中的“生产常见组合模板”，职责是提供中等复杂度团队插件的标准化组织方式：

- 同时覆盖 `commands`、`agents`、`skills`、`hooks`、`scripts` 五类组件（`standard-plugin.md:7-35`）。
- 通过完整 metadata manifest 展示“可分发插件”应具备的信息面（`standard-plugin.md:41-55`）。
- 通过 hooks 脚本演示“流程护栏”与“质量门禁”在插件中的落地（`standard-plugin.md:426-455,457-504`）。

它在文档体系中的定位：

- 被 `plugin-structure/README.md` 作为 production 级模板指向（`README.md:59-64`）。
- 对应 `plugin-structure/SKILL.md` 的 full-featured pattern（`SKILL.md:417-432`）。
- 与 `component-patterns.md` 的“角色型 agent 组织”“资源丰富 skill”“单文件 hooks 配置”模式一致（`component-patterns.md:135-152,261-290,293-320`）。

## 功能点目的
1. 目录分层与职责清晰
- 将用户入口（commands）、任务执行者（agents）、知识库（skills）、事件控制（hooks）、通用脚本（scripts）分离，降低耦合（`standard-plugin.md:7-35`）。

2. 通过命令连接执行脚本
- `/lint` 命令明确调用 `${CLAUDE_PLUGIN_ROOT}/scripts/run-linter.sh`，体现可移植路径实践（`standard-plugin.md:78-82`，`SKILL.md:256-277`）。

3. 通过 agent + skill 协同提升专业性
- `code-reviewer` 自动加载 `code-standards`；`test-generator` 自动加载 `testing-patterns`（`standard-plugin.md:163-166,210-213`）。

4. 通过 Stop hook 形成完成前质量闸门
- 会话停止时执行 `validate-commit.sh`，在变更范围内跑语言特定 lint 并给出 JSON 反馈（`standard-plugin.md:442-454,466-503`）。

5. 给出用户视角运行结果
- `/lint`、`/test`、自动 agent 选择的输出示例用于预期管理（`standard-plugin.md:506-572`）。

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 组件加载与调用主流程
- 加载阶段：
1. Claude Code 读取 `.claude-plugin/plugin.json`（`SKILL.md:343`）。
2. 因未显式覆盖 `commands/agents/hooks` 路径，依赖默认目录自动发现（`SKILL.md:100-101,344-348`）。
3. 注册命令、agent、skill，并载入 `hooks/hooks.json`。

- 执行阶段（典型链路）：
1. 用户执行 `/lint`。
2. 命令指令要求执行 `bash ${CLAUDE_PLUGIN_ROOT}/scripts/run-linter.sh`（`standard-plugin.md:80-82`）。
3. 完成后用户结束任务触发 `Stop` hook。
4. `hooks/scripts/validate-commit.sh` 根据 staged 变更调用 `eslint`/`pylint` 并返回 `systemMessage` JSON（`standard-plugin.md:472-503`）。

### 2) 关键数据结构与协议
- `plugin.json`（metadata 为主）
- 字段覆盖 `name/version/description/author/homepage/repository/license/keywords`，符合 manifest 参考“推荐可分发配置”（`standard-plugin.md:42-54`，`manifest-reference.md:42-208,463-481`）。

- `commands/*.md` 协议
- YAML frontmatter：`name + description`（`standard-plugin.md:60-63,98-101`）。
- Markdown body 承载“执行步骤 + 输出格式 + 后续动作”。

- `agents/*.md` 协议
- 当前示例采用 `description + capabilities` frontmatter（`standard-plugin.md:133-141,180-188`）。
- 这是 `plugin-structure` skill 当前示例格式（`SKILL.md:150-160`），但与 `agent-development` 的新必填 `name/model/color` 规范存在偏差（`agent-development/SKILL.md:62-141,351-357`）。

- `hooks/hooks.json` 协议
- 顶层为事件键：`PreToolUse` + `Stop`，每项包含 `matcher` 与 `hooks[]`（`standard-plugin.md:429-454`）。
- hook 子项包含 `type/prompt|command/timeout`，与 `hook-development` 的事件结构示例一致（`hook-development/SKILL.md:121-209,344-381`）。

- `validate-commit.sh` 命令流程
1. `git status -s` 判断是否有工作区变更（`standard-plugin.md:466-469`）。
2. 仅对 staged 的 `js|ts|py` 文件处理：`git diff --name-only --cached`（`standard-plugin.md:472`）。
3. JS/TS 调 `npx eslint --quiet`，Python 调 `python -m pylint --errors-only`（`standard-plugin.md:485,490`）。
4. 汇总错误数并 `exit 1` 拦截结束，成功则 `exit 0`（`standard-plugin.md:497-503`）。

### 3) 设计取舍
- 优先“目录约定 + 默认发现”，减少 manifest 路径配置复杂度。
- 在 Stop 阶段集中校验而非每次写入校验，执行开销更低，但反馈更晚。
- 输出以可读文本为主，协议化结构（machine-readable）较少，仅 hook 返回 JSON message。

## 关键代码路径与文件引用
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:7-35`：整体目录组织。
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:41-55`：完整 manifest metadata。
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:57-128`：`lint/test` 命令流程定义。
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:130-222`：两个 agent 的角色与流程。
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:224-424`：skills 与 reference 样例组织。
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:426-455`：hook 事件配置。
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:457-504`：`validate-commit.sh` 逻辑。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-356`：默认目录 + 自定义路径补充规则。
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:135-152,261-290,293-320`：组织模式映射。

## 依赖与外部交互
### 仓库内依赖（文档与规范）
- `plugin-structure/SKILL.md`：组件格式、自动发现、路径可移植规则。
- `references/manifest-reference.md`：metadata 与 path 字段规范。
- `hook-development/SKILL.md`：hook 事件模型与 command/prompt hook 行为。
- `agent-development/SKILL.md`：agent frontmatter 新规范（用于一致性比对）。

### 运行时外部交互
- Git 命令：`git status -s`、`git diff --name-only --cached`。
- Node 工具链：`npx eslint`。
- Python 工具链：`python -m pylint`。
- 这些都要求宿主环境具备对应可执行与依赖，否则 Stop hook 会失败或误报。

## 风险、边界与改进建议
### 风险与边界
1. Stop hook 的变更源不一致风险
- 脚本先看 `git status -s`（工作区），再只检查 staged 文件（`--cached`）。若仅有 unstaged 变更，会出现“有变更但无 staged 代码文件”分支，可能给出“通过”但实际未检。

2. 仅覆盖 JS/TS/Python
- 其他语言（Go/Rust/Ruby 等）被跳过，质量门禁不完整。

3. 工具可用性耦合
- `npx eslint`、`python -m pylint` 依赖项目和环境安装，不具备时会产生噪声失败。

4. agent schema 演进不一致
- 示例仍为旧式 `description + capabilities`；按 `agent-development` 当前规范，`name/model/color` 是 required，存在新项目照抄后触发兼容问题的风险。

5. hooks 格式文档漂移
- `hook-development` 文档对插件 hooks 是否需要 `{"hooks": {...}}` wrapper 有前后不一致描述（`hook-development/SKILL.md:64-81` vs `340-381`），该示例采用“事件顶层直出”格式，容易导致读者混淆。

### 改进建议
1. 统一 staged/unstaged 语义
- 若策略是“提交前校验”，应先检查 `git diff --name-only --cached` 是否为空，并在提示中明确“仅校验已暂存文件”。

2. 扩展语言与可插拔 lint
- 增加基于文件后缀到命令映射表（例如 `go test`、`cargo clippy`、`rubocop`），并支持通过配置开关禁用。

3. 对缺失依赖做软失败
- 在脚本前置检测 `command -v`，缺失时输出可操作指引，而非直接失败。

4. 升级 agent 样例 frontmatter
- 在示例中增加 `name/model/color`，并保留 `capabilities` 作为可选扩展字段，降低跨技能认知断层。

5. 明确 hooks 示例格式
- 在本示例旁加入注释说明“当前采用直出事件格式”，并链接到 `hook-development` 的最终规范段落，减少误用。
