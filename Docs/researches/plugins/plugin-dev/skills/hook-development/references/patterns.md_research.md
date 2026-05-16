# plugins/plugin-dev/skills/hook-development/references/patterns.md 研究

## 场景与职责

`patterns.md` 是 `hook-development` 的“标准模式库”，职责是提供高频 hook 需求的可复用模板，帮助开发者在最短路径内组装出可用策略。

它在知识架构中的角色：
- `SKILL.md` 给原则与协议，`patterns.md` 给“按问题分类的落地模板”（`plugins/plugin-dev/skills/hook-development/SKILL.md:665-673`）。
- `plugin-dev` README 把 hook-development 定位为自动化核心，并强调 scripts 工具；`patterns.md` 对应的是“实现模板层”，scripts 对应“验证层”（`plugins/plugin-dev/README.md:56-75,271-284`）。

在流程中的职责：
- 为 `/plugin-dev:create-plugin` 的 hooks 实施阶段提供直接可抄的事件配置结构（`plugins/plugin-dev/commands/create-plugin.md:201-209`）。
- 作为 skill-development 提倡的 progressive disclosure 典型参考，即把细节模式下沉到 references，而不挤在 SKILL.md 主体（`plugins/plugin-dev/skills/skill-development/SKILL.md:190-194`）。

边界：
- 文档是“模式集合”，不是完整工程脚手架。
- 每个模式都需按插件真实协议（hooks 格式、路径、测试流程）做适配。

## 功能点目的

`patterns.md` 共给出 10 个模式，覆盖从安全到治理再到可配置化：

1. Pattern 1 安全写入校验（`PreToolUse` + prompt）
- 防止写系统目录/凭据文件/路径穿越（`patterns.md:5-25`）。

2. Pattern 2 停止前测试门禁（`Stop` + prompt）
- 若发生代码改动但未测试则阻断（`patterns.md:27-47`）。

3. Pattern 3 会话起始上下文加载（`SessionStart` + command）
- 自动识别项目类型并写入环境变量（`patterns.md:49-85`）。

4. Pattern 4 通知日志（`Notification` + command）
- 统一记录提醒事件（`patterns.md:86-107`）。

5. Pattern 5 MCP 删除类操作防护（`PreToolUse` + regex matcher）
- 对高风险 MCP 行为做二次确认（`patterns.md:108-129`）。

6. Pattern 6 构建门禁（`Stop` + prompt）
- 代码改动后要求 build 成功（`patterns.md:130-151`）。

7. Pattern 7 危险操作人工确认（`PreToolUse` + prompt）
- 对 rm/delete/drop 等返回 ask（`patterns.md:152-173`）。

8. Pattern 8 编辑后质量检查（`PostToolUse` + command）
- 对文件修改后触发 linter/formatter（`patterns.md:174-207`）。

9. Pattern 9 临时启用 hooks（flag file）
- 通过开关文件控制策略是否生效（`patterns.md:261-299`）。

10. Pattern 10 配置驱动 hooks（local json）
- 用户/项目可配置阈值和策略参数（`patterns.md:300-346`）。

这些模式的共同目标是：
- 缩短从需求到可运行 hook 的路径。
- 把“典型策略”表达成可组合单元。
- 为后续迁移到 advanced/hybrid 提供基础积木。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 模式骨架：事件选择 + matcher 约束 + hook 类型

几乎所有模式都遵循固定结构：
1. 选择触发事件（PreToolUse/PostToolUse/Stop/SessionStart/Notification）
2. 配置 matcher（工具名、通配符或 regex）
3. 选 prompt 或 command 执行体

这与技能协议一致（`plugins/plugin-dev/skills/hook-development/SKILL.md:121-209`, `385-425`）。

### 2) 事件分层思路

从模式分布可以看出作者的事件分层策略：
- `PreToolUse`：做风险预防（Pattern 1/5/7）。
- `PostToolUse`：做结果治理（Pattern 8）。
- `Stop`：做任务完整性门禁（Pattern 2/6）。
- `SessionStart`：做上下文初始化（Pattern 3）。
- `Notification`：做审计与分析（Pattern 4）。

这套分层与主技能“各事件用途”定义一致（`SKILL.md:123-276`）。

### 3) prompt 模式与 command 模式的分工

- prompt 模式：适合复杂语义判断（Pattern 1/2/5/6/7）。
- command 模式：适合确定性初始化、日志、工具执行（Pattern 3/4/8/9/10）。

该分工与技能主文档“prompt 推荐 + command 用于确定性任务”的原则一致（`SKILL.md:22-58`）。

### 4) 可组合性实现

`Pattern Combinations` 展示了多事件组合：
- PreToolUse 双通道（写入 + Bash）
- Stop 完整性检查
- SessionStart 上下文初始化

这形成“前置防护 + 结束门禁 + 会话引导”的三层策略（`patterns.md:208-259`）。

### 5) 可开关与可配置化机制

Pattern 9/10 是工程化关键：

1. Flag 文件启停
- 通过存在性判断快速开关 hook（`patterns.md:267-273`）。
- 文档额外提示需要重启会话生效（`patterns.md:298`），与技能生命周期约束一致（`SKILL.md:574-589`）。

2. JSON 配置驱动
- 从 `.claude/my-plugin.local.json` 读取 `strictMode`、`maxFileSize`（`patterns.md:306-316`）。
- 根据配置控制逻辑分支与阈值（`patterns.md:318-330`）。

### 6) 命令与协议细节

常见命令族：
- `jq`：输入解析和配置读取（`patterns.md:197-199`, `310-311`, `325`）。
- `npx eslint`：编辑后质量校验（`patterns.md:201-203`）。
- shell exit code：`exit 0/2` 控制放行与阻断。

协议层依赖：
- `CLAUDE_PLUGIN_ROOT` 用于可移植路径（`patterns.md:61`, `98`, `186`, `251`）。
- `CLAUDE_PROJECT_DIR`/`CLAUDE_ENV_FILE` 用于会话上下文和配置读取（`patterns.md:72,77`, `268,306`）。

## 关键代码路径与文件引用

- 目标文档：`plugins/plugin-dev/skills/hook-development/references/patterns.md:1-346`
- 上游入口：`plugins/plugin-dev/skills/hook-development/SKILL.md:665-673`
- 关联协议：
  - hook 类型与适用事件：`plugins/plugin-dev/skills/hook-development/SKILL.md:20-58,121-276`
  - matcher 规则：`plugins/plugin-dev/skills/hook-development/SKILL.md:385-425`
  - 生命周期限制（需重启）：`plugins/plugin-dev/skills/hook-development/SKILL.md:572-589`
- 对应样例实现：
  - `plugins/plugin-dev/skills/hook-development/examples/validate-write.sh:1-38`
  - `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh:1-43`
  - `plugins/plugin-dev/skills/hook-development/examples/load-context.sh:1-55`
- 验证工具链：
  - `plugins/plugin-dev/skills/hook-development/scripts/README.md:5-164`
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1-159`
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:1-252`
  - `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:1-153`
- 调用/治理链：
  - `plugins/plugin-dev/README.md:56-75,249-257,271-284`
  - `plugins/plugin-dev/commands/create-plugin.md:201-209,257-260`
  - `plugins/plugin-dev/agents/plugin-validator.md:107-115`

## 依赖与外部交互

1. 本地运行依赖
- shell 工具：`bash`, `jq`
- 生态工具：`npx eslint`（Pattern 8 例子）
- 依赖 Claude Code 注入的环境变量（`CLAUDE_PLUGIN_ROOT`, `CLAUDE_PROJECT_DIR`, `CLAUDE_ENV_FILE`）

2. 协议交互
- 通过 stdin 获取 hook 输入 JSON。
- 通过退出码与 stderr/stdout 控制阻断与反馈。
- 通过 matcher 把模式绑定到特定工具或事件。

3. 外部交互
- `patterns.md` 本身主要是本地模式，外部交互较轻；如需日志/审计对接通常会下沉到 advanced 文档里的外部系统模式。

4. 与仓内真实插件配置的兼容关联
- 仓内部分插件 hooks 采用 wrapper 格式（如 `plugins/security-guidance/hooks/hooks.json`），而本文件示例采用 direct 结构。落地到插件时需包装到 `hooks` 字段下，或按所在宿主格式调整。

## 风险、边界与改进建议

1. 风险：示例格式与插件常见 wrapper 结构不一致
- `patterns.md` 全文用 direct 事件结构；插件文档则推荐 wrapper（`SKILL.md:62-80`）。
- 建议：在每个 Pattern 后补“插件版 wrapper 示例”，降低复制误用。

2. 风险：部分模式对 LLM 判断质量高度敏感
- Pattern 1/2/5/6/7 的核心决策在 prompt 中，若提示词模糊可能导致误判。
- 建议：给每个 prompt 模式补“必须包含的判定维度清单”和“失败时兜底动作”。

3. 风险：Pattern 8 性能与副作用未约束
- `npx eslint` 对大仓库/未安装依赖场景可能明显增时或报错噪音。
- 建议：增加文件白名单、超时和失败降级策略。

4. 风险：Pattern 9/10 易与会话加载时机冲突
- 文档虽然提示“需要重启”，但未给出自动提醒机制。
- 建议：提供一个 `Notification` hook 提醒“配置变化需重启会话”。

5. 风险：Pattern 3 的目录判断逻辑可更严谨
- 示例中项目检测覆盖有限，且某些存在性判断在真实文件系统语义下需要区分文件/目录。
- 建议：统一使用 `-f`/`-d` 正确判断并补充 monorepo/多语言场景。

6. 边界
- 模式库不承担安全审计闭环，不代替 validator 和脚本测试。
- 模式片段不保证可直接生产，需要结合团队策略、工具链、延迟预算做裁剪。

7. 综合改进建议
- 在文档末尾新增“模式选型矩阵”：
  - 风险等级
  - 推荐事件
  - 建议 hook 类型
  - 超时建议
  - 是否需要 hybrid
- 为每个模式配一条最小可运行验证命令（基于 `test-hook.sh`），把模式库升级为可执行手册。
