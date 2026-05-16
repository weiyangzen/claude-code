# plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh 研究

## 场景与职责

`validate-agent.sh` 是 `plugin-dev` 工具链中用于校验 agent 定义文件（`agents/*.md`）的轻量静态校验器，定位是“开发阶段质量闸门”，而不是运行时组件。

它所在的上下文角色：

- 在插件开发主流程中，`/plugin-dev:create-plugin` 的 Phase 5（组件实现）与 Phase 6（验证）都要求对 agent 文件运行该脚本（`plugins/plugin-dev/commands/create-plugin.md:193-200,252-256`）。
- `plugin-validator` agent 在“Validate Agents”步骤中将该脚本作为优先校验工具（`plugins/plugin-dev/agents/plugin-validator.md:86-97`）。
- `agent-development` skill 将其列为 scripts 工具并纳入标准创建工作流（`plugins/plugin-dev/skills/agent-development/SKILL.md:394-412`）。
- `README` 将它归入 toolkit 的 validation utilities（`plugins/plugin-dev/README.md:35-40,167-173`）。

职责边界：

- 仅做单文件静态结构检查，不做自动修复。
- 仅依赖 shell 文本工具解析 frontmatter，不执行 YAML 语义级解析。
- 不直接集成 CI，也未发现仓库内自动化调用链（主要通过文档/agent 指引人工执行）。

## 功能点目的

脚本覆盖的功能点与目的如下（源于 `validate-agent.sh:7-217`）：

1. 输入与文件基本合法性
- 目的：避免空参数、路径错误导致后续检查无意义。
- 行为：无参数输出 usage 并 `exit 1`；文件不存在时失败。

2. frontmatter 边界完整性
- 目的：确保 agent 文件采用 `---` 包裹 YAML frontmatter 的标准结构。
- 行为：检查首行必须为 `---`，并检查存在第二个 `---`。

3. 必填字段检查（`name/description/model/color`）
- 目的：保证 agent 可被框架识别且具备最小触发/显示配置。
- 行为：逐项提取并判空，缺失计入 error。

4. 字段约束检查
- `name`：格式、长度、泛化命名告警。
- `description`：长度阈值、是否包含 `<example>`、是否包含 “Use this agent when”。
- `model`：是否在 `inherit/sonnet/opus/haiku`。
- `color`：是否在 `blue/cyan/green/yellow/magenta/red`。
- `tools`：可选，存在则打印。

5. system prompt 检查
- 目的：避免空提示词或过短提示词导致 agent 行为不稳定。
- 行为：检查 prompt 非空、长度范围、是否含二人称表达与结构关键词（responsibilities/process/steps/output）。

6. 汇总与退出码
- 设计目标：
  - 无 error 且无 warning：`exit 0`
  - 仅 warning：`exit 0`
  - 有 error：`exit 1`
- 实际行为见“风险”章节（存在提前退出问题）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

校验流程是顺序式 short-circuit：

1. 参数检查与文件存在检查。
2. frontmatter 起止检查。
3. `sed/awk` 提取 frontmatter 与正文。
4. 逐字段检查并累计 `error_count` / `warning_count`。
5. 检查 system prompt。
6. 统一汇总输出。

该流程在设计上能区分 hard error 与 soft warning；但由于 `set -e` 与算术表达式组合，warning/error 自增时可能在中途退出，导致流程未走到汇总。

### 2) 数据结构与解析方式

输入协议（文件协议）：

- 目标文件是 Markdown：
  - 顶部 YAML frontmatter（`---` 包围）
  - 后续正文作为 system prompt

解析实现：

- `FRONTMATTER=$(sed -n '/^---$/,/^---$/{ /^---$/d; p; }' "$AGENT_FILE")`
- `SYSTEM_PROMPT=$(awk '/^---$/{i++; next} i>=2' "$AGENT_FILE")`

字段提取实现（示例）：

- `name`: `grep '^name:' | sed ...`
- `description`: `grep '^description:' | sed ...`
- `model/color/tools`: 同类 `grep '^key:'`

这是一种“行匹配”解析策略，优点是轻量、零额外依赖；缺点是对 YAML 多行值、缩进、引号/块标量兼容性弱。

### 3) 规则实现细节

- 名称正则：`^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$`
- 名称长度：3-50
- 描述长度建议：10-5000
- prompt 长度：20-10000
- model/color 为枚举白名单检查（未知值仅 warning）

### 4) 外部命令依赖

脚本显式/隐式使用的命令：

- `bash`（解释器）
- `head`, `tail`, `grep`, `sed`, `awk`, `echo`

无 `jq/yq/python` 依赖，易移植；但解析鲁棒性受限。

### 5) 实测行为（复现实验）

在仓库根目录执行：

```bash
bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh \
  plugins/plugin-dev/agents/plugin-validator.md
```

观察到：

- 输出到 `description should include <example> blocks for triggering` 后即结束；
- 未进入汇总段落（`All checks passed / Validation passed / Validation failed`）；
- 命令退出码为 `1`。

同样行为可在 `plugins/plugin-dev/agents/agent-creator.md` 复现。

根因：`set -euo pipefail` 下使用 `((warning_count++))` / `((error_count++))`；当表达式值为 0 时返回非零状态，触发 `-e` 提前退出。

## 关键代码路径与文件引用

核心实现：

- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:5`
  - 严格模式 `set -euo pipefail`
- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:32-49`
  - frontmatter 边界检测与 frontmatter/prompt 抽取
- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:55-159`
  - 必填字段与枚举/长度/格式校验
- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:170-203`
  - system prompt 质量检查
- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:205-217`
  - 结果汇总与退出语义

调用方/规范来源：

- `plugins/plugin-dev/commands/create-plugin.md:193-200,252-256`
- `plugins/plugin-dev/agents/plugin-validator.md:86-97`
- `plugins/plugin-dev/skills/agent-development/SKILL.md:260-286,394-412`
- `plugins/plugin-dev/README.md:35-40,167-173`

示例与使用说明：

- `plugins/plugin-dev/skills/agent-development/examples/agent-creation-prompt.md:195-205`
- `plugins/plugin-dev/skills/agent-development/examples/complete-agent-examples.md:417-425`
- `plugins/plugin-dev/agents/agent-creator.md:162`

上下文一致性问题相关引用：

- `plugins/plugin-dev/skills/agent-development/SKILL.md:398-400`（提及 `test-agent-trigger.sh`）
- `plugins/plugin-dev/skills/agent-development/scripts` 目录当前仅有 `validate-agent.sh`

## 依赖与外部交互

1. 文件系统交互
- 输入：用户指定的 agent markdown 文件路径。
- 输出：stdout 文本报告 + 进程退出码（0/1）。

2. 与人/流程交互
- 该脚本主要由开发者手工运行，或由上层 agent（如 `plugin-validator`）在建议流程中调用。
- 目前未发现自动化测试或 CI 直接执行该脚本的配置。

3. 与规范文档交互
- 校验规则与 `agent-development/SKILL.md` 的字段规范基本同源；
- 但实现细节与规范存在偏差（见风险章节）。

4. 与其他脚本的关系
- 无内部子脚本调用（单体脚本）。
- 在工具链层面与 `validate-hook-schema.sh` 同属“组件校验脚本”角色（并列能力，而非函数调用关系）。

## 风险、边界与改进建议

### 高风险 1：计数器自增在 `set -e` 下导致提前退出

现象：首次 warning/error 发生即可能退出，导致：

- 报告不完整（看不到后续检查和汇总）；
- 退出码偏离“warning 也可通过”的设计目标；
- 上层流程（人工或 agent）被误导为硬失败。

建议：

- 将 `((warning_count++))` / `((error_count++))` 改为不会触发 `-e` 的写法，如：
  - `warning_count=$((warning_count + 1))`
  - `error_count=$((error_count + 1))`
  - 或 `((warning_count+=1)) || true`。

### 高风险 2：`description` 仅按单行提取，导致对多行 `<example>` 误判

现象：脚本只取 `description:` 所在行，拿不到后续多行 `<example>` 块，因此会误报 “description should include <example> blocks”。

建议：

- 使用更稳健的 frontmatter 解析策略：
  - 方案 A（轻依赖）：`awk` 按 frontmatter 区块提取 `description` 多行；
  - 方案 B（高鲁棒）：使用 `yq` 解析 YAML。
- 对 description 的 `<example>` 校验应基于完整字段内容而不是首行文本。

### 中风险 3：`name` 校验与技能文档规范不一致

现状：

- 脚本允许大写（`[a-zA-Z0-9-]`）；
- `SKILL.md` 标注应为 lowercase（`plugins/plugin-dev/skills/agent-development/SKILL.md:66-69,269-273`）。

建议：

- 将正则收紧为 `^[a-z0-9][a-z0-9-]*[a-z0-9]$`，保持与规范一致。

### 中风险 4：路径示例容易误导仓库根目录执行者

现状：多个文档写 `scripts/validate-agent.sh` 或 `./scripts/validate-agent.sh`，但从仓库根目录运行并不存在该相对路径。

建议：

- 统一文档示例为可从仓库根目录直接执行的形式：
  - `bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh agents/xxx.md`
- 或明确“需在 `plugins/plugin-dev/skills/agent-development/` 下执行”。

### 中风险 5：文档列出的 `test-agent-trigger.sh` 缺失

现状：`SKILL.md` 列出 `test-agent-trigger.sh`，目录中实际不存在。

建议：

- 二选一：补齐脚本；或从文档移除该条并改为“手工触发测试指南”。

### 低风险 6：解析边界较脆弱

包括但不限于：

- `grep '^key:'` 对缩进键值不兼容；
- frontmatter 中若出现额外 `---` 行可能导致截断歧义；
- 二人称检查仅覆盖英文关键词且大小写/表达变体有限。

建议：

- 用 YAML parser 替代纯文本 grep/sed；
- 为“结构建议类检查”保留 info/warn，不影响主流程；
- 增加一组脚本级回归样例（valid/warn/error/multiline-description）。

---

研究结论：`validate-agent.sh` 在“规则意图”层面覆盖较完整，但当前实现存在两个阻断级问题（计数器退出语义、description 多行误判），会直接影响其作为质量闸门的可信度。优先修复这两点后，再做规范一致性和文档路径统一，可显著提升可用性与结果稳定性。
