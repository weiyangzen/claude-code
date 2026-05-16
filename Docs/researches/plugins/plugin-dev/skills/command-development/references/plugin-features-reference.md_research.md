# plugin-features-reference.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md` 是 command-development 里“插件命令特性层”规范，负责把通用 slash command 语法落到插件生态（commands/agents/skills/hooks/resources）中。

其核心职责：

1. 定义插件命令自动发现与命名空间组织（`13-53`）。
2. 定义 `${CLAUDE_PLUGIN_ROOT}` 路径可移植协议（`75-220`）。
3. 提供插件命令模式库（配置驱动、模板驱动、多脚本、环境感知、数据管理，`221-329`）。
4. 规定与 plugin agents/skills/hooks 的协同方式（`330-437`）。
5. 统一输入、资源、输出、错误处理的验证模式（`439-561`）。

调用方主要是：

1. `plugins/plugin-dev/skills/command-development/SKILL.md:326,527-829,833`
2. `plugins/plugin-dev/skills/command-development/README.md:53,101`
3. `plugins/plugin-dev/skills/command-development/examples/plugin-commands.md:1-560`（示例大量复用此文模式）
4. `plugins/plugin-dev/commands/create-plugin.md:183-191`（命令实现阶段按此类规范落地）

## 功能点目的

1. 自动发现与命名空间
- 降低注册成本，统一插件加载行为（`17-31`）。
- 通过子目录 namespace 降低命令冲突（`33-53`）。

2. `${CLAUDE_PLUGIN_ROOT}`
- 解决跨安装路径不可预测问题（`79-85`）。
- 统一脚本、模板、配置、资源访问模式（`87-204`）。

3. 插件命令模式库
- 将高频需求抽象成可复用模板，降低命令设计复杂度（`223-329`）。

4. 组件协同
- 明确命令如何触发 agent（Task）与 skill 提示、如何与 hook 联动（`334-407`）。

5. 验证模板
- 把“失败后排障”前置到命令实现阶段（参数校验、文件存在、资源可用、输出检查）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 插件命令发现协议

定义了插件命令目录协议：`commands/**/*.md` 自动发现，子目录转换为命名空间标签（`17-53`）。

运行流程可概括为：

1. 插件加载时扫描命令目录。
2. 注册命令名与 namespace。
3. `/help` 暴露插件标签。

### 2) `${CLAUDE_PLUGIN_ROOT}` 路径协议

实现要点：

1. 在 Bash 执行中引用：`!`node ${CLAUDE_PLUGIN_ROOT}/scripts/...``（`97,119`）。
2. 在文件包含中引用：`@${CLAUDE_PLUGIN_ROOT}/templates/...`（`99,144`）。
3. 结合参数拼装环境配置路径：`@${CLAUDE_PLUGIN_ROOT}/config/$1.json`（`158,300`）。

### 3) 组件协同协议

1. Agent 协同：命令文案中声明“使用某 agent”，运行时由 Task 工具拉起（`334-356`）。
2. Skill 协同：命令内显式点名 skill，提示运行时加载对应知识（`358-384`）。
3. Hook 协同：命令触发的事件（如 git commit）交由 hooks 自动处理（`385-407`）。

### 4) 验证命令模式

文档给出标准校验命令组合：

1. 参数格式校验：`grep -E`（`451-456`）。
2. 文件存在校验：`test -f`（`474-482`）。
3. 资源完整性校验：`test -f/-d/-x`（`513-516`）。
4. 输出可用性校验：exit code + 文件数量（`533-537`）。
5. 错误兜底：`... || echo "ERROR:$?"`（`551`）。

## 关键代码路径与文件引用

核心文件：

1. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:1-609`

直接调用方：

1. `plugins/plugin-dev/skills/command-development/SKILL.md:326,645,715,833`
2. `plugins/plugin-dev/skills/command-development/README.md:53,101`
3. `plugins/plugin-dev/skills/command-development/examples/plugin-commands.md:20-560`

关键配套文件：

1. `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 正确路径规范）
2. `plugins/README.md:49-61`（插件标准目录结构）
3. `plugins/plugin-dev/commands/create-plugin.md:126-144,183-191`（命令创建实作流程）

## 依赖与外部交互

1. 依赖 Claude Code 插件发现机制与命名空间展示机制。
2. 依赖运行时注入环境变量 `${CLAUDE_PLUGIN_ROOT}`。
3. 依赖工具与命令协议：`Read/Write/Bash/Task/Skill`。
4. 依赖外部 CLI（`git/node/bash/test/find/wc` 等）执行具体检查。
5. 与 hooks 的交互依赖 hook 配置加载（见 `plugins/plugin-dev/skills/hook-development/SKILL.md`）。

## 风险、边界与改进建议

### 风险

1. manifest 路径示例冲突：
- 本文件示例写 `plugin-name/plugin.json`（`20-25`）。
- 官方与 plugin-structure 规范要求 `.claude-plugin/plugin.json`（`plugins/README.md:51-55`，`manifest-reference.md:7-10`）。
- 这是高风险误导点。

2. 多处 `Bash(*)`（如 `129,154,230,274,317,528`）与最小权限策略冲突。

3. “路径有空格无需额外 quoting”（`216-219`）在复杂 shell 组合下并不总是安全。

### 边界

1. 该文档描述“模式”，不提供实际脚本与测试资产。
2. Agent/Skill/Hook 协同只是约定，不保证目标组件实际存在。

### 改进建议

1. 统一修正文档树中的 manifest 示例到 `.claude-plugin/plugin.json`。
2. 默认示例改为 `Bash(prefix:*)`，把 `Bash(*)` 收敛到“特例”章节。
3. 补充“引用路径加引号”的 shell 安全模板（如 `"${CLAUDE_PLUGIN_ROOT}/path"`）。
4. 增加一节“与 plugin-validator 的一致性规则”，避免文档与校验器偏差。
