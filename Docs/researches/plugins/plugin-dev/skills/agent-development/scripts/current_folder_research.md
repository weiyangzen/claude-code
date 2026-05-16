# DIR `plugins/plugin-dev/skills/agent-development/scripts` 研究

## 场景与职责

该目录是 `plugin-dev` 工具链中针对 Agent 定义文件的“静态质量闸门”，当前仅包含一个可执行脚本：

- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`

在插件开发流程中，它承担两个核心职责：

1. 在 Agent 文件落盘后做结构化校验，尽早拦截格式错误。
2. 将 `agent-development` 文档中的部分规范（frontmatter 字段、命名、prompt 基本质量）落实成可执行检查。

其上游使用场景主要来自：

- `plugins/plugin-dev/commands/create-plugin.md:193-200,252-256`（Phase 5/6 明确要求校验 agent）
- `plugins/plugin-dev/agents/plugin-validator.md:86-97`（plugin-validator 在 agent 校验阶段引用该工具）
- `plugins/plugin-dev/skills/agent-development/SKILL.md:401-413`（agent 创建工作流第 7 步）

## 功能点目的

`validate-agent.sh` 的功能点可分为 5 类：

1. 输入与文件存在性校验
- 目的：保证脚本只在目标文件存在时执行后续逻辑。
- 位置：`validate-agent.sh:7-30`

2. frontmatter 边界校验
- 目的：确保 agent 文件是“YAML frontmatter + markdown body”形态。
- 规则：第一行必须是 `---`，且存在第二个 `---` 作为闭合。
- 位置：`validate-agent.sh:32-46`

3. frontmatter 字段校验
- 必填字段：`name/description/model/color`
- 可选字段：`tools`
- 目的：把 agent 可发现性、触发描述和运行配置在创建阶段校验掉。
- 位置：`validate-agent.sh:51-168`

4. system prompt 基础质量校验
- 目的：避免空 prompt、过短 prompt，推动提示词具备可执行结构。
- 规则：长度阈值、二人称提示、结构化关键词建议（responsibilities/process/output）。
- 位置：`validate-agent.sh:170-203`

5. 汇总与退出码
- 目的：将错误与警告分级，并输出统一结果。
- 预期：error=0 时成功（有 warning 也应返回 0）。
- 位置：`validate-agent.sh:205-217`

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 命令协议（CLI Contract）

脚本接口是：

```bash
bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh <path/to/agent.md>
```

- 参数数量为 0 时打印 usage 并 `exit 1`（`validate-agent.sh:8-17`）。
- 成功路径会输出每项检查结论与最终汇总（`validate-agent.sh:205-217`）。

### 2) 关键数据结构

1. frontmatter 文本块
- 提取方式：
  - `sed -n '/^---$/,/^---$/{ /^---$/d; p; }'`（`validate-agent.sh:48`）
- 用途：后续 `grep '^key:'` 拉取字段。

2. system prompt 文本块
- 提取方式：
  - `awk '/^---$/{i++; next} i>=2'`（`validate-agent.sh:49`）
- 用途：长度和文风检查（`validate-agent.sh:174-203`）。

3. 计数器
- `error_count` / `warning_count`（`validate-agent.sh:55-57`）
- 用于最终退出逻辑分流（`validate-agent.sh:208-216`）。

### 3) 关键流程拆解

1. 预检阶段
- 文件存在（`-f`）
- 首行 `---`
- 存在 closing `---`

2. 字段阶段
- `name`：正则、长度、泛化命名告警（`validate-agent.sh:58-88`）
- `description`：长度、`<example>`、`Use this agent when`（`validate-agent.sh:90-119`）
- `model`：允许集 `inherit|sonnet|opus|haiku`（`validate-agent.sh:121-139`）
- `color`：允许集 `blue|cyan|green|yellow|magenta|red`（`validate-agent.sh:141-159`）
- `tools`：仅做存在性提示，不校验数组语法（`validate-agent.sh:161-168`）

3. Prompt 阶段
- 非空 + 长度阈值（20~10000）
- 文风 heuristics：`You are|You will|Your`
- 结构建议：关键词匹配 `responsibilities|process|steps|output`

4. 收尾阶段
- `error=0 && warning=0` => 通过
- `error=0 && warning>0` => 警告通过
- `error>0` => 失败

### 4) 复现到的行为差异（实测）

在仓库根目录实测：

```bash
bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh plugins/plugin-dev/agents/plugin-validator.md
```

输出在第一条 warning（description 未检测到 `<example>`）后提前终止，`exit_code=1`，未进入脚本末尾汇总分支。

该现象与以下实现相关：

- 全局 `set -euo pipefail`（`validate-agent.sh:5`）
- warning/error 自增使用 `((warning_count++))` / `((error_count++))`

在 Bash 中，`((x++))` 当表达式结果为 `0` 时返回状态码 `1`，会触发 `set -e` 提前退出。

### 5) 与文档协议的实现偏差

1. 多行 `description` 解析不足
- 当前实现仅抓 `grep '^description:'` 首行（`validate-agent.sh:91`）。
- 但该技能规范要求 `description` 内嵌多段 `<example>`（`SKILL.md:82-106`）。
- 结果：大量合法 agent 可能被误告警“缺少 `<example>`”。

2. 路径示例不一致
- 文档多处写 `scripts/validate-agent.sh` 或 `./scripts/validate-agent.sh`（如 `SKILL.md:411`，`examples/agent-creation-prompt.md:199-202`）。
- 实际仓库不存在 `plugins/plugin-dev/scripts/`，脚本位于 `skills/agent-development/scripts/`。

## 关键代码路径与文件引用

### 目录内核心文件

- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:1-217`

### 直接上游（调用方/流程方）

- `plugins/plugin-dev/commands/create-plugin.md:193-200`
  - Agent 生成后要求执行 validate-agent 校验。
- `plugins/plugin-dev/commands/create-plugin.md:252-256`
  - Phase 6 质量检查再次要求运行该脚本。
- `plugins/plugin-dev/agents/plugin-validator.md:86-97`
  - plugin-validator 在验证 agents 时优先建议调用该脚本。
- `plugins/plugin-dev/agents/agent-creator.md:162`
  - agent-creator 输出中建议验证命令。

### 规范与资源耦合

- `plugins/plugin-dev/skills/agent-development/SKILL.md:60-160`
  - frontmatter 字段规则来源。
- `plugins/plugin-dev/skills/agent-development/SKILL.md:281-286`
  - system prompt 长度与结构性要求来源。
- `plugins/plugin-dev/skills/agent-development/SKILL.md:398-399`
  - utility scripts 声明（含当前缺失脚本）。
- `plugins/plugin-dev/README.md:154-173`
  - 对外声明该技能包含 `validate-agent.sh`。

### 文档/示例中的命令参考

- `plugins/plugin-dev/skills/agent-development/examples/agent-creation-prompt.md:195-205`
- `plugins/plugin-dev/skills/agent-development/examples/complete-agent-examples.md:417-425`

### 间接被依赖关系

- `plugins/plugin-dev/skills/skill-development/SKILL.md:305-310,608-611`
  - 将 `agent-development` 作为“可复用最佳实践技能”示例，间接要求 scripts 层可用。

## 依赖与外部交互

### 运行时依赖

脚本使用的外部命令：

- `bash`
- `head` / `tail`
- `grep`
- `sed`
- `awk`

特点：

- 无 Python/Node/jq 依赖，跨环境复用成本低。
- 对 GNU/BSD 工具行为有轻微耦合（文本处理正则语义）。

### 文件系统交互

- 只读目标 agent 文件。
- 不写回任何源码文件。
- 通过退出码向调用方传递结果（0 成功，1 失败）。

### 与其他系统的协议边界

- 与 `create-plugin`、`plugin-validator` 的接口是“约定命令 + 文本输出 + 退出码”。
- 当前没有机器可读输出（JSON/SARIF），不利于 CI 聚合与自动修复流程。

### 测试现状

- 目标目录无单元测试/集成测试脚本。
- 仅有人工命令式验证。
- `SKILL.md` 提及 `test-agent-trigger.sh`，但目录中并不存在该脚本。

## 风险、边界与改进建议

### 风险 1（高）：`set -e` 与 `((count++))` 组合导致提前退出

- 证据：`validate-agent.sh:5,63,70,77,80,86,95,102,105,111,117,126,136,146,156,176,183,186,192`
- 影响：warning/error 一旦首次触发，脚本可能直接中断，无法输出完整报告，且“warning-only 场景”被误判失败。
- 建议：统一改为 `count=$((count + 1))` 或 `((count+=1)) || true`。

### 风险 2（高）：多行 description 误解析导致误告警

- 证据：`validate-agent.sh:91` 仅抓首行；而规范要求 `<example>` 多段块（`SKILL.md:82-106`）。
- 影响：合法 agent 被误报“缺少 `<example>`”，削弱工具可信度。
- 建议：用 YAML 解析器读取 `description`（例如 `yq`），或改成 frontmatter 区域内“字段块提取”。

### 风险 3（中）：文档命令路径不统一，易误用

- 证据：`agent-creation-prompt.md:201` 使用 `./scripts/...`；实际脚本位于 `skills/agent-development/scripts/`。
- 影响：从仓库根目录直接执行时失败，增加排障成本。
- 建议：统一改为仓库根相对路径示例：
  - `bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh <agent.md>`

### 风险 4（中）：规范与实现在 `name` 大小写约束上存在漂移

- 证据：文档声明 lowercase（`SKILL.md:66`），但正则允许大写（`validate-agent.sh:68`）。
- 影响：可能放行不符合文档规范的 agent 标识符。
- 建议：将正则收敛到 `^[a-z0-9][a-z0-9-]*[a-z0-9]$`。

### 风险 5（中）：缺失触发测试脚本

- 证据：`SKILL.md:399` 声明 `test-agent-trigger.sh`，实际目录无该文件。
- 影响：使用者按文档操作会遇到“文档承诺 > 实际能力”问题。
- 建议：二选一：
  1. 实现并补齐 `test-agent-trigger.sh`；
  2. 从文档移除该脚本并改成手工测试指引。

### 边界说明

- 该脚本只覆盖“静态文本规范”，不验证 agent 真实触发行为与执行质量。
- `tools` 仅检查存在，不校验 JSON/YAML 数组格式与工具名合法性。
- 未提供 CI 友好输出，当前更适合本地开发阶段人工使用。

### 优先级改进路线（建议）

1. 先修复计数器导致的提前退出（阻断级）。
2. 改造 description 解析，支持多行/块文本。
3. 统一所有文档中的校验命令路径。
4. 对齐 `name` 大小写规则并补充 `tools` 语法校验。
5. 增加最小回归测试（至少覆盖：合法 agent、warning-only、error-only、多行 description）。
