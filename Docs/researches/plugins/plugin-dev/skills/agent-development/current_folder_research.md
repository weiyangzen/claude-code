# plugins/plugin-dev/skills/agent-development 研究

## 场景与职责

`agent-development` 是 `plugin-dev` 工具链中专门负责“如何定义与生成 Claude Code agent”的技能目录，服务于以下典型场景：

1. 用户要在插件中新增自治 agent（如代码审查、测试生成、安全分析）。
2. `/plugin-dev:create-plugin` 工作流进入 Agent 实现阶段，需要标准化产出 agent 文件。
3. 需要统一 agent frontmatter 规范（`name/description/model/color/tools`）与触发描述格式（`<example>/<commentary>`）。
4. 需要用脚本快速校验 agent 文件结构、必填字段和 prompt 质量。

该目录在 `plugin-dev` 里的职责是“规范 + 模板 + 参考 + 校验工具”四合一：

- 规范：`SKILL.md` 定义必填字段、命名规则、触发描述规则、系统提示词结构（`plugins/plugin-dev/skills/agent-development/SKILL.md:20-357`）。
- 模板：`examples/` 给出 AI 生成与完整 agent 示例（`.../examples/agent-creation-prompt.md`，`.../examples/complete-agent-examples.md`）。
- 参考：`references/` 提供更深层的 prompt 设计模式与触发样例库（`.../references/*.md`）。
- 校验：`scripts/validate-agent.sh` 提供命令行校验入口（`.../scripts/validate-agent.sh:1-217`）。

## 功能点目的

### 1. 统一 agent 文件格式

通过 `SKILL.md` 的“Complete Format”要求 agent 采用 markdown + YAML frontmatter + system prompt 的统一结构（`SKILL.md:22-58`），目的：

- 降低创建 agent 的随意性，减少触发失败。
- 为自动发现机制提供一致输入（`SKILL.md:299-306`）。

### 2. 强化触发质量（description）

`description` 被定义为最关键字段，要求包含触发条件和多个 `<example>` 场景（`SKILL.md:82-114`），目的：

- 让 Claude 在真实对话中更稳定地匹配 agent。
- 支持“显式请求 + 主动触发”两类调度策略（`references/triggering-examples.md:125-189,212-230`）。

### 3. 标准化 system prompt 设计

通过 `system-prompt-design.md` 提供可复用结构（角色、职责、流程、质量标准、输出格式、边界条件）（`references/system-prompt-design.md:5-38`），目的：

- 提升 agent 自治执行质量。
- 减少生成后反复返工。

### 4. 提供 AI 辅助创建路径

通过 `agent-creation-system-prompt.md` + `examples/agent-creation-prompt.md`，把“需求 -> JSON -> agent 文件”的流程模板化（`references/agent-creation-system-prompt.md:55-60,91-120`；`examples/agent-creation-prompt.md:19-53`），目的：

- 快速得到结构完整、可直接落盘的 agent。
- 复用 Claude Code 生产环境沉淀的提示词模式。

### 5. 脚本化质量门禁

`validate-agent.sh` 目标是把“结构正确性 + 基本质量要求”前置到脚本层（`scripts/validate-agent.sh:11-17,51-203`），目的：

- 在提交前快速发现缺字段/格式错误/prompt 过短等问题。
- 作为 `create-plugin`、`plugin-validator` 的可执行校验能力。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1. 目录分层实现（Progressive Disclosure）

- `SKILL.md`：核心规则与快速流程。
- `references/`：深度设计模式与触发策略库。
- `examples/`：可直接改造的完整样例。
- `scripts/`：可执行校验工具。

该分层与 `skill-development` 里倡导的技能组织方式一致（`plugins/plugin-dev/skills/skill-development/SKILL.md:318-360`）。

### 2. 关键流程 A：AI 辅助生成

1. 输入自然语言需求。
2. 使用 `agent-creation-system-prompt` 让模型输出固定 JSON：`identifier/whenToUse/systemPrompt`（`references/agent-creation-system-prompt.md:55-60`）。
3. 把 JSON 映射到 frontmatter + body（`references/agent-creation-system-prompt.md:91-120`）。
4. 设置 `model/color/tools` 并落盘到 `agents/*.md`。
5. 用 `validate-agent.sh` 做本地校验。

这一流程同时被 `agent-creator` agent 内嵌复用（`plugins/plugin-dev/agents/agent-creator.md:37-123`）。

### 3. 关键流程 B：手工创建

`SKILL.md` 明确手工路径 7 步（命名 -> description/examples -> model/color/tools -> system prompt -> 保存）（`SKILL.md:250-258`），其后进入校验与触发测试（`SKILL.md:307-327,401-414`）。

### 4. 数据结构与“协议”

#### 4.1 Agent frontmatter 结构

最小必填：

- `name`（3-50，字母数字连字符）
- `description`（含触发条件与 examples）
- `model`（`inherit|sonnet|opus|haiku`）
- `color`（`blue|cyan|green|yellow|magenta|red`）

可选：`tools`（数组）（`SKILL.md:60-160`）。

#### 4.2 description 内嵌“触发样例协议”

通过 `<example>...<commentary>...</commentary>...</example>` 传达：上下文、用户原话、assistant 触发前响应、触发理由、触发动作（`references/triggering-examples.md:9-18,80-121`）。

#### 4.3 system prompt 结构协议

建议固定区块：Role -> Responsibilities -> Process -> Quality Standards -> Output Format -> Edge Cases（`references/system-prompt-design.md:9-38`）。

### 5. 命令与脚本实现

核心命令：

```bash
bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh <path/to/agent.md>
```

脚本实现要点：

- 使用 `set -euo pipefail`（`validate-agent.sh:5`）。
- 通过 `head/tail/grep/sed/awk` 检查 frontmatter 边界并抽取字段（`validate-agent.sh:33-49,59,91,122,142,162`）。
- 按 error/warning 计数并尝试在结尾给出汇总（`validate-agent.sh:55-57,205-217`）。

### 6. 触发与发现机制（运行时约束）

- 文档约定 `agents/` 下 `.md` 自动发现（`SKILL.md:299`，`plugin-structure/SKILL.md:138-141`）。
- 名称空间策略：`agent-name` 或 `plugin:subdir:agent-name`（`SKILL.md:303-306`）。
- `create-plugin` 命令在 Phase 5/6 将该技能和校验脚本纳入标准流程（`commands/create-plugin.md:157-200,252-256`）。

## 关键代码路径与文件引用

### 目标目录核心文件

- `plugins/plugin-dev/skills/agent-development/SKILL.md`
  - 结构规范：`22-58`
  - frontmatter 字段规则：`60-160`
  - 设计与创建流程：`162-258`
  - 校验与组织：`260-357`
  - 资源索引与实现流程：`377-415`
- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`
  - 参数与入口：`7-23`
  - frontmatter/system prompt 提取：`47-49`
  - 字段校验逻辑：`58-168`
  - prompt 质量检查：`170-203`
  - 汇总退出：`205-217`
- `plugins/plugin-dev/skills/agent-development/references/agent-creation-system-prompt.md`
  - 生成 JSON 契约：`55-60`
  - JSON 转文件模板：`91-120`
- `plugins/plugin-dev/skills/agent-development/references/system-prompt-design.md`
  - 通用结构模板：`5-38`
  - 四类模式（analysis/generation/validation/orchestration）：`40-230`
- `plugins/plugin-dev/skills/agent-development/references/triggering-examples.md`
  - `<example>` 标准格式：`9-18`
  - 示例类型与策略：`123-249,325-344`
- `plugins/plugin-dev/skills/agent-development/examples/agent-creation-prompt.md`
  - AI 生成操作模板：`17-53`
  - 校验调用示例：`199-205`
- `plugins/plugin-dev/skills/agent-development/examples/complete-agent-examples.md`
  - 4 个完整 agent 模板：`5-427`

### 上下文调用方 / 被调用方

- 调用方：`plugins/plugin-dev/commands/create-plugin.md`
  - 明确“Agents 阶段必须加载 agent-development skill”并调用 `validate-agent.sh`（`160,193-200,252-256`）。
- 调用方：`plugins/plugin-dev/agents/plugin-validator.md`
  - 在 Agent 校验步骤中明确引用本目录脚本（`86-97`，尤其 `89`）。
- 被调用方：`plugins/plugin-dev/agents/agent-creator.md`
  - 直接复用本目录 references 中同款系统提示词方法学（内容高度同构，`37-123`）。
- 文档聚合：`plugins/plugin-dev/README.md`
  - 将其列为 7 大技能之一并声明包含 `validate-agent.sh`（`14,154-173`）。

## 依赖与外部交互

### 1. 对 Claude Code 运行时约束的依赖

- 依赖 agent 自动发现机制（`agents/**/*.md`）。
- 依赖 Skill/Agent 选择器依据 description + examples 进行触发匹配。
- 依赖项目上下文（如 `CLAUDE.md`）参与 agent 生成（`references/agent-creation-system-prompt.md:10,24,156-159`）。

### 2. 对 shell 工具链的依赖

`validate-agent.sh` 依赖：`bash`, `head`, `tail`, `grep`, `sed`, `awk`。无外部二进制依赖，便于跨项目复用。

### 3. 与其他技能/组件交互

- 与 `skill-development`：目录分层策略一致，彼此互为模板（`skill-development/SKILL.md:305-310,610`）。
- 与 `plugin-structure`：共同约束 `agents/` 自动发现位置（`plugin-structure/SKILL.md:138-141`）。
- 与 `create-plugin` 流程：作为 Phase 5/6 的 agent 子流程基础设施。

### 4. 测试与验证现状

- 本目录无独立 automated tests（无 `test/` 或 CI 断言脚本）。
- 仅提供手工/命令式校验脚本 `validate-agent.sh`。

## 风险、边界与改进建议

### 风险 1（高）：`validate-agent.sh` 计数器在 `set -e` 下会提前退出

现象：

- 脚本使用 `set -euo pipefail`（`validate-agent.sh:5`）。
- 多处使用 `((warning_count++))` / `((error_count++))`（如 `63,86,111,126,176`）。
- 在 Bash 中该表达式会以算术结果作为退出码，首次从 `0` 自增时返回非零，触发 `set -e` 直接退出。

实测：

- 执行 `bash .../validate-agent.sh plugins/plugin-dev/agents/plugin-validator.md`，在首个 warning 后中断，未进入末尾汇总逻辑（脚本输出停在“description should include <example> blocks”，且整体退出非预期）。

建议：

- 将 `((warning_count++))` 改为 `warning_count=$((warning_count+1))` 或 `((warning_count+=1)) || true`。
- 同步修复 `error_count` 自增写法，保证脚本可运行到统一汇总出口。

### 风险 2（高）：description 解析只读取首行，导致 `<example>` 校验失真

现象：

- 当前解析：`DESCRIPTION=$(... | grep '^description:' | sed ...)`（`validate-agent.sh:91`）。
- 对多行 description（当前仓库 agent 普遍采用）只读取首行文本，无法看到后续 `<example>` 块。
- 直接导致“明明有 `<example>` 仍被判 warning”，并影响触发质量门禁有效性。

建议：

- 使用真正 YAML 解析器（如 `yq`）或用 `awk` 基于 frontmatter 边界 + 键位缩进读取多行值。
- 若保持纯 POSIX 工具，至少增加对 `description: |` / `description: >` 块标量形式的支持。

### 风险 3（中）：规范与校验器对 `name` 大小写约束不一致

现象：

- 文档要求小写（`SKILL.md:66,271-273`）。
- 脚本正则允许大写（`validate-agent.sh:68` 使用 `[a-zA-Z0-9-]`）。

建议：

- 校验器改为仅允许 `[a-z0-9-]`，与文档一致。
- 为兼容历史文件，可先 warning 后逐步升级为 error。

### 风险 4（中）：文档声明存在 `test-agent-trigger.sh`，实际缺失

现象：

- `SKILL.md` 资源清单写有 `test-agent-trigger.sh`（`SKILL.md:398-400`）。
- `scripts/` 目录实际仅 `validate-agent.sh`。

建议：

- 二选一：
  1. 补齐 `test-agent-trigger.sh`；
  2. 从文档移除该脚本并改写触发测试指南。

### 风险 5（中）：跨技能文档存在 agent frontmatter 规范漂移

现象：

- `agent-development` 强调 `name/model/color` 必填。
- `plugin-structure` 示例仍出现旧式 `description + capabilities` 模板（`plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:134-141`，`.../advanced-plugin.md:229-236`）。

影响：

- 用户按 `plugin-structure` 示例创建 agent 后，会被 `validate-agent.sh` 报缺字段。

建议：

- 统一所有技能中的 agent 示例格式，避免互相打架。
- 在 `README` 或 `create-plugin` 中指向单一“权威规范来源”（建议 `agent-development/SKILL.md`）。

### 边界说明

- 该目录侧重“agent 文件设计与校验”，不覆盖 Claude Code 内部触发算法实现细节。
- 脚本是轻量静态校验，不等价于真实对话触发效果验证；仍需场景化手测。
- references 提供的是模式库与经验法则，不是强约束语法规范。

### 优先级化改进路线

1. 先修复 `validate-agent.sh` 的 `set -e + ((count++))` 退出问题（阻断级）。
2. 再修复 description 多行解析，恢复 `<example>` 校验可信度。
3. 统一跨技能示例格式，消除文档冲突。
4. 明确 `test-agent-trigger.sh` 去留并同步 README/SKILL。
5. 补充最小回归测试（至少 3 个样例：合法、多行 description、缺字段）。
