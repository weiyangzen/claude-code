# plugins/plugin-dev/agents 目录研究（DIR）

## 场景与职责

`plugins/plugin-dev/agents` 是 `plugin-dev` 插件中的“专家子代理层”，目录内包含 3 个 agent 定义文件：

- `agent-creator.md`：把用户对“新 agent 能力”的自然语言需求转成可落地的 agent 配置与系统提示词（`plugins/plugin-dev/agents/agent-creator.md:1-176`）。
- `plugin-validator.md`：做插件级结构与配置质量审查（manifest、目录、commands/agents/skills/hooks/mcp、安全项）（`plugins/plugin-dev/agents/plugin-validator.md:1-184`）。
- `skill-reviewer.md`：做技能（SKILL.md）质量评审，重点看触发描述与渐进式披露（`plugins/plugin-dev/agents/skill-reviewer.md:1-184`）。

在 `plugin-dev` 的主工作流里，该目录承担“创建 -> 评审 -> 校验”闭环中的 agent 角色：

- `/plugin-dev:create-plugin` 明确要求使用这 3 个专用 agent（`plugins/plugin-dev/commands/create-plugin.md:15,180,194-200,238-251,351`）。
- `plugin-dev` README 的 8 阶段流程将其列为核心特性与验证阶段能力（`plugins/plugin-dev/README.md:25-40`）。
- 仓库级 `plugins/README.md` 对 `plugin-dev` 的摘要也直接列出这 3 个 agents（`plugins/README.md:24`）。

结论：该目录不是业务代码目录，而是“插件开发流程中的角色协议目录”，通过 frontmatter + 提示词定义触发条件、工具权限、流程模板和输出格式。

## 功能点目的

### 1) `agent-creator`：标准化 agent 生成过程

目的：把“我想要一个什么 agent”的模糊需求，收敛成结构化产物（identifier、description/examples、system prompt、model/color/tools），并直接产出 `agents/[identifier].md`（`plugins/plugin-dev/agents/agent-creator.md:75-123`）。

它的价值在于：
- 统一命名、示例、输出模板和质量门槛（`plugins/plugin-dev/agents/agent-creator.md:132-173`）。
- 内建“结合 CLAUDE.md 项目上下文”的约束，降低生成 agent 与项目规范冲突概率（`plugins/plugin-dev/agents/agent-creator.md:39,43,53`）。

### 2) `plugin-validator`：在发布前做全量健康检查

目的：将插件质量检查从“人工凭经验”变成“可重复流程”，覆盖：
- manifest 字段与命名规范；
- 目录布局与自动发现前提；
- commands/agents/skills/hooks/mcp 各组件规范；
- 安全项（凭据、协议、脚本引用等）。

见 `plugins/plugin-dev/agents/plugin-validator.md:51-135`。

该 agent 是 `/plugin-dev:create-plugin` Phase 6 的第一道质量门（`plugins/plugin-dev/commands/create-plugin.md:233-246`）。

### 3) `skill-reviewer`：提升 skill 的触发命中率与可维护性

目的：针对 `skills/*/SKILL.md` 做专门评审，重点并非“语法是否能过”，而是：
- 描述触发词是否具体、第三人称是否正确；
- SKILL.md 是否过胖；
- references/examples/scripts 的渐进式披露是否合理。

见 `plugins/plugin-dev/agents/skill-reviewer.md:60-105`。

该 agent 与 skill-development 规范形成配套：skill-development 直接建议使用它做质量复核（`plugins/plugin-dev/skills/skill-development/SKILL.md:225-230`）。

### 4) 三者协同目标

以“生成（agent-creator）-> 组件评审（skill-reviewer）-> 全插件验收（plugin-validator）”形成闭环，支撑 `/plugin-dev:create-plugin` 在 Phase 5/6 的质量收敛（`plugins/plugin-dev/commands/create-plugin.md:153-269`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 本目录 -> 被调用方）

1. 用户执行 `/plugin-dev:create-plugin`，命令前置允许使用 `Task`/`Skill` 等工具（`plugins/plugin-dev/commands/create-plugin.md:1-5`）。
2. Phase 5 在“Agents”子步骤要求通过 `agent-creator` 产出 agent 文件，并调用 `validate-agent.sh` 做结构校验（`plugins/plugin-dev/commands/create-plugin.md:192-200`）。
3. Phase 6 先跑 `plugin-validator` 做全局检查，再按需用 `skill-reviewer` 复核 skills（`plugins/plugin-dev/commands/create-plugin.md:238-251`）。
4. 按插件能力参考文档，命令触发 agent 的机制基于 Task 工具，agent 文件需位于 `agents/` 目录（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`）。

### B. 数据结构与协议

#### 1. Agent 文件协议（frontmatter + system prompt）

该目录 3 个文件都遵循同一结构：
- frontmatter 字段：`name`、`description`（含 `<example>`）、`model`、`color`、`tools`；
- 正文：角色定义、步骤化流程、质量标准、输出格式、边界情况。

依据：
- 目录内 3 个 agent 文件（`plugins/plugin-dev/agents/*.md`）
- agent-development 的规范定义（`plugins/plugin-dev/skills/agent-development/SKILL.md:20-58,60-160`）

#### 2. 三个 agent 的能力协议差异

- `agent-creator` 输出协议偏“生成型”：要求创建文件并输出配置摘要（`plugins/plugin-dev/agents/agent-creator.md:142-165`）。
- `plugin-validator` 输出协议偏“审计型”：要求按严重级别给问题清单与 PASS/FAIL 结论（`plugins/plugin-dev/agents/plugin-validator.md:143-173`）。
- `skill-reviewer` 输出协议偏“评审改进型”：要求给 description 改写建议、内容质量评估、优先级建议（`plugins/plugin-dev/agents/skill-reviewer.md:107-174`）。

#### 3. 关键命令与脚本约定

- agent 文件校验脚本：`bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh <agent.md>`（`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:7-17`）。
- 脚本检查维度：frontmatter 边界、必填字段、命名规则、prompt 长度与二人称风格等（`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:25-203`）。

### C. 配置、测试、脚本、文档上下文

1. 配置：
- 本目录“配置”就是 3 个 agent frontmatter。
- 上游命令通过 `allowed-tools` 放行 `Task`/`Skill` 等能力，使 agent 编排可执行（`plugins/plugin-dev/commands/create-plugin.md:4`）。

2. 测试：
- 本目录无独立测试目录/测试脚本。
- 主要依赖通用校验器 `validate-agent.sh`（`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`）。

3. 文档：
- `plugin-dev/README.md` 说明这 3 个 agent 在整体流程中的定位（`plugins/plugin-dev/README.md:31,38,154-174`）。
- `agent-development`/`skill-development` 两个 skill 文档提供设计与验证原则（`plugins/plugin-dev/skills/agent-development/SKILL.md:217-260`，`plugins/plugin-dev/skills/skill-development/SKILL.md:212-230`）。

## 关键代码路径与文件引用

### 目标目录核心文件

- `plugins/plugin-dev/agents/agent-creator.md`
- `plugins/plugin-dev/agents/plugin-validator.md`
- `plugins/plugin-dev/agents/skill-reviewer.md`

### 上游调用方

- `plugins/plugin-dev/commands/create-plugin.md:15,180,194-200,238-251,351`
- `plugins/plugin-dev/README.md:25-40,154-174`
- `plugins/README.md:24`

### 规范与参考来源

- `plugins/plugin-dev/skills/agent-development/SKILL.md:20-58,60-160,260-357`
- `plugins/plugin-dev/skills/agent-development/references/agent-creation-system-prompt.md:7-71,195-205`
- `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:330-357`
- `plugins/plugin-dev/skills/skill-development/SKILL.md:77-85,212-230,269-277`

### 下游脚本与工具链

- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`
- （文档提及但当前不存在）`plugins/plugin-dev/skills/agent-development/scripts/test-agent-trigger.sh`（`plugins/plugin-dev/skills/agent-development/SKILL.md:398-400`）

## 依赖与外部交互

### 运行时依赖

1. Claude Code 对 `agents/*.md` 的发现与触发机制（规范层定义于 agent-development：`plugins/plugin-dev/skills/agent-development/SKILL.md:299-306`）。
2. 命令对 Task 工具的依赖，用于实际拉起 agent（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`）。
3. 脚本依赖：`validate-agent.sh` 需要标准 Unix 工具链（`bash`、`grep`、`sed`、`awk` 等，见脚本实现）。

### 目录内 agent 的工具权限交互面

- `agent-creator`：`["Write", "Read"]`，能直接生成 agent 文件（`plugins/plugin-dev/agents/agent-creator.md:34`）。
- `plugin-validator`：`["Read", "Grep", "Glob", "Bash"]`，可做跨目录检索与命令校验（`plugins/plugin-dev/agents/plugin-validator.md:36`）。
- `skill-reviewer`：`["Read", "Grep", "Glob"]`，纯读审查路径（`plugins/plugin-dev/agents/skill-reviewer.md:35`）。

### 外部交互边界

- 本目录自身不包含 hook/mcp 可执行代码，不直接处理网络协议。
- 但 `plugin-validator` 的流程会要求检查 MCP/Hooks 配置与安全项，属于“对外部交互配置的审核”，不是直接调用外部服务（`plugins/plugin-dev/agents/plugin-validator.md:107-134`）。

## 风险、边界与改进建议

### 风险

1. `validate-agent.sh` 在首个 warning 处提前退出，无法完成完整报告。
- 脚本开启 `set -euo pipefail`（`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:5`）。
- 使用 `((warning_count++))`/`((error_count++))` 计数（如 `:87,112,117`），在 bash 中当表达式结果为 `0` 时返回非零，可能触发 `set -e` 立即退出。
- 实测：对 `agent-creator.md` 执行后只输出到首个 warning 即退出，返回码 `1`，没有最终 summary（本次研究执行结果）。

2. 多行 `description` 的 `<example>` 检查逻辑失真。
- 当前仅通过 `grep '^description:'` 抽取 description 首行（`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:91`），拿不到后续多行 `<example>` 块。
- 导致对当前三个 agent 都出现“缺少 `<example>`”告警，即使文件实际包含 `<example>`（agent 文件见 `plugins/plugin-dev/agents/*.md`）。

3. 文档引用了不存在的测试脚本。
- `agent-development` 文档将 `test-agent-trigger.sh` 列为 utility script（`plugins/plugin-dev/skills/agent-development/SKILL.md:398-400`），但目录内实际只有 `validate-agent.sh`。

4. 3 个 agent 文件尾部存在疑似编辑残留，影响提示词整洁性。
- `agent-creator.md:174-176`、`plugin-validator.md:182-184`、`skill-reviewer.md:182-184` 出现孤立代码围栏或总结性句子，非核心协议内容。

5. `plugin-dev` 目录当前缺少 `.claude-plugin/plugin.json`，与仓库 README 声明的标准插件结构不一致。
- 标准结构要求插件包含 manifest（`plugins/README.md:49-61,67-70`）。
- 当前 `plugins/plugin-dev` 未发现该文件（本次目录检查）。这会影响其按“标准插件目录”直接加载的一致性。

### 边界

1. 本目录只定义“agent 行为合同”，不直接提供业务脚本执行逻辑。
2. 触发时机与是否调用由上游命令（如 `/plugin-dev:create-plugin`）和运行时调度决定。
3. 校验质量受调用上下文影响较大（例如是否给出明确 plugin 根目录、是否具备可读文件权限）。

### 改进建议

1. 修复 `validate-agent.sh` 的计数与退出语义。
- 方案：将 `((warning_count++))` 改为 `warning_count=$((warning_count+1))`（或 `((warning_count+=1)) || true`），避免 `set -e` 提前退出。
- 目标：无论有无 warning，都能输出完整汇总并给出稳定退出码策略。

2. 改进 frontmatter 解析为“多行安全”实现。
- 方案：使用 `awk` 解析 frontmatter 段并按 key 提取 block scalar；或引入 `yq`（若环境允许）准确读取 YAML。
- 目标：正确识别 `description` 中 `<example>`，降低误报。

3. 对齐文档与实际脚本清单。
- 若不计划提供 `test-agent-trigger.sh`，应从 `agent-development/SKILL.md` 移除该条；
- 若计划提供，则补齐脚本并在 README/skill 中给出调用示例。

4. 清理三份 agent 文件尾部残留文本。
- 保留结构化协议主体，移除非必要尾句与孤立围栏，降低后续维护歧义。

5. 增加“命令-代理映射”回归检查脚本。
- 自动校验 `create-plugin.md` 中声明的 agent 名称在 `agents/` 实际存在；
- 作为 docs/tooling 回归的一部分，避免命名漂移。
