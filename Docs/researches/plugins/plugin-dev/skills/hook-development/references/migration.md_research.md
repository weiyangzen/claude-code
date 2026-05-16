# plugins/plugin-dev/skills/hook-development/references/migration.md 研究

## 场景与职责

`migration.md` 是 `hook-development` 资源中的“演进路径文档”，核心职责是把团队已有的 command hook 资产，平滑迁移到 prompt hook 或 hybrid 方案，而不是从零介绍 hooks。

在技能体系中的位置：
- 主技能文档把它定义为“从基础到高级 hooks 的迁移指南”（`plugins/plugin-dev/skills/hook-development/SKILL.md:671-673`）。
- `skill-development` 文档把“详细迁移内容放到 references/”作为 progressive disclosure 的标准实践（`plugins/plugin-dev/skills/skill-development/SKILL.md:190-194`）。

在工作流中的责任链：
- `/plugin-dev:create-plugin` 要求 hooks 组件落地后做验证（`plugins/plugin-dev/commands/create-plugin.md:201-209`, `257-260`）。
- `migration.md` 提供的是“如何改造策略本身”，与 `scripts/test-hook.sh`、`validate-hook-schema.sh` 构成变更闭环。

边界定义：
- 文档给的是策略迁移框架和样例，不负责自动转换旧脚本。
- 迁移后的正确性仍需靠样例回归与运行时测试确认。

## 功能点目的

`migration.md` 的功能目标可拆为 6 组：

1. 先解释迁移价值，降低改造阻力
- 以可维护性、语义理解、边界覆盖能力解释为何从硬编码规则迁移（`migration.md:5-13`）。

2. 用“前后对照”降低认知成本
- Bash 校验从 command 迁移到 prompt 的前后配置与问题清单（`migration.md:14-82`）。
- 写文件校验的同类迁移示例（`migration.md:83-154`）。

3. 明确“不要盲目迁移”的边界
- 保留 command hook 的三种典型场景：确定性数学判断、外部工具返回值、极低延迟快检（`migration.md:155-204`）。

4. 给出 hybrid 中间态
- 同时挂 command + prompt，把快检和深判组合使用，降低一次性迁移风险（`migration.md:205-232`）。

5. 提供迁移操作清单
- 从“抽取旧逻辑”到“清理旧脚本”的 checklist，强调落地步骤（`migration.md:233-244`）。

6. 提供规则翻译方法
- 将字符串 contains/regex/多条件 if 转换为自然语言判定标准（`migration.md:320-365`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 迁移方法论流程（文档隐含流程）

从全文可归纳出 7 步流程：

1. 盘点旧 command hook 的显式规则与漏判点（`migration.md:48-54`, `123-128`）。
2. 将“模式匹配规则”转为“自然语言 criteria”（`migration.md:322-365`）。
3. 选择目标事件与 matcher（Bash、Write|Edit 等）（`migration.md:60-71`, `134-144`）。
4. 设置 prompt timeout（建议 15-30s）（`migration.md:67`, `241`）。
5. 保留必须确定性的 command 分支（`migration.md:159-204`）。
6. 使用 hybrid 过渡避免大爆炸切换（`migration.md:205-232`）。
7. 完成文档与脚本归档（`migration.md:242-243`, `267-278`）。

### 2) 数据结构与配置协议

文档中迁移后的目标结构仍是事件->matcher->hooks[]：
- hook 类型从 `command` 切换为 `prompt`（`migration.md:65-67`, `139-141`）。
- 通过 prompt 内容承载策略逻辑，而非 shell if/regex。

该结构与技能协议一致（`plugins/plugin-dev/skills/hook-development/SKILL.md:121-209`），但注意格式层面：
- `migration.md` 示例使用 direct 事件结构。
- 技能文档对插件场景建议 wrapper 格式 `{"description":..., "hooks": {...}}`（`SKILL.md:62-80,119`）。

因此迁移落地时需要在“示例结构”和“插件真实结构”之间做一次包装转换。

### 3) 命令到提示的“规则语义升级”

文档核心技术不是语法替换，而是语义升级：
- 从字面串（`"rm -rf"`）升级到意图类别（destructive ops、privilege escalation、network without consent）（`migration.md:41-45`, `66-67`）。
- 从路径字符串模式升级到“路径 + 内容联合判定”（`migration.md:110-121`, `140-141`）。

这使得规则可维护性变好，但也引入模型不确定性与提示设计质量依赖。

### 4) 与工具链的结合方式（推荐实践）

虽然 `migration.md` 未直接写工具命令，但结合 `hook-development/scripts` 可形成稳定迁移流水线：
- 结构验证：`validate-hook-schema.sh`（`scripts/README.md:5-27`）。
- 样本测试：`test-hook.sh --create-sample` + 回放（`scripts/README.md:29-53`, `test-hook.sh:25-100`, `183-252`）。
- 脚本遗留质量检查：`hook-linter.sh`（`scripts/README.md:62-90`）。

### 5) 与 create-plugin / validator 的交汇点

- create-plugin Hooks 阶段明确要求“优先 prompt hooks + 用脚本测试”（`create-plugin.md:201-209`）。
- plugin-validator 的 hooks 校验条目关注 `matcher/hooks/type/command` 等结构项（`plugin-validator.md:107-115`）。

因此 `migration.md` 实际扮演的是“把策略从硬编码搬到 prompt 同时仍满足结构校验约束”的说明书。

## 关键代码路径与文件引用

- 目标文档：`plugins/plugin-dev/skills/hook-development/references/migration.md:1-369`
- 上游入口：`plugins/plugin-dev/skills/hook-development/SKILL.md:665-673`
- 协议依据：
  - hooks 格式（wrapper/direct）：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-119`
  - PreToolUse/Stop 输出与退出码：`plugins/plugin-dev/skills/hook-development/SKILL.md:144-209,294-299`
  - 输入字段与变量：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-339`
- 工具链：
  - `plugins/plugin-dev/skills/hook-development/scripts/README.md:5-164`
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1-159`
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:1-252`
  - `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:1-153`
- 调用链：
  - `plugins/plugin-dev/commands/create-plugin.md:201-209,257-260`
  - `plugins/plugin-dev/README.md:56-75,249-257,271-284`
  - `plugins/plugin-dev/agents/plugin-validator.md:107-115`

## 依赖与外部交互

1. 技术依赖
- Prompt hook 依赖 Claude/LLM 的自然语言判定能力。
- Command hook 示例依赖 `bash`, `jq`, `stat` 与外部扫描器（`security-scanner`）等（`migration.md:161-186`）。

2. 协议依赖
- 依赖 Claude Code hook 运行协议：stdin JSON 输入、exit code 决策、prompt 变量替换。
- 依赖 plugin 结构约定：`hooks/hooks.json` 需要与插件格式对齐。

3. 外部交互
- `migration.md` 本身无网络调用，但外部工具示例（如安全扫描器）反映了与企业工具链集成的迁移场景。

4. 本次仓内实测关联
- 使用 `validate-hook-schema.sh` 对真实插件 hooks（wrapper 格式）校验时会报 `Unknown event type: description/hooks` 并出现 `jq` 索引失败（exit=5），这意味着迁移完成后若直接套用脚本，验证环节可能误失败。

## 风险、边界与改进建议

1. 风险：示例与插件真实格式存在隐式转换步骤
- 文档示例是 direct 结构（`migration.md:19-33`, `58-73`），插件推荐 wrapper（`SKILL.md:62-80`）。
- 改进：在每个 “After” 示例后附一段 wrapper 版本，避免落地时格式错配。

2. 风险：迁移后策略可解释性提升但确定性下降
- prompt hook 对边界案例表现受提示词质量和上下文影响。
- 改进：对高风险策略采用 hybrid 常驻；用“确定性前置 + 语义后置”分层。

3. 风险：输出 schema 与主技能文档示例存在差异
- 部分 command 示例输出 `{"decision":"deny"}`，而主文档对 PreToolUse 强调 `hookSpecificOutput.permissionDecision`（`SKILL.md:144-153`）。
- 改进：在迁移文档增加“各事件推荐输出字段对照表”。

4. 风险：迁移 checklist 缺少“回归基线”定义
- 当前 checklist 强调过程，但未强制定义通过标准（误报率、漏报率、耗时预算）。
- 改进：补一项“建立旧/新策略对照样本集并给出通过阈值”。

5. 风险：成本与延迟预算未显式量化
- 文档建议 prompt timeout（15-30s），但未给“最大允许阻塞时间”与并发场景预算。
- 改进：补充建议矩阵：风险等级 -> timeout -> 是否必须保留 command 快检。

6. 边界
- `migration.md` 关注策略迁移，不覆盖 hooks 生态所有问题（如 schema 校验脚本兼容性、插件加载时机、热更新限制）。这些需配合 `SKILL.md` 生命周期章节和 scripts 工具文档处理（`SKILL.md:572-599`）。

7. 综合改进建议
- 在“Complete Migration Example”里补上：
  - wrapper 格式版本
  - test-hook.sh 样本回放命令
  - validate-hook-schema/hook-linter 命令
  - 回滚开关（flag file）
- 让迁移指南从“策略对比文档”升级为“可执行迁移 runbook”。
