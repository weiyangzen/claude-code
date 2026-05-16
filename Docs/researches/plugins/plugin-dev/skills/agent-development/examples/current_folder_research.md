# plugins/plugin-dev/skills/agent-development/examples 研究

## 场景与职责

`plugins/plugin-dev/skills/agent-development/examples` 是 `agent-development` 技能的“可复用样例层”，面向两类场景：

1. 需要快速创建新 agent 的用户，按模板完成 `需求 -> 配置 -> agent 文件`。
2. 需要对齐高质量 agent 写法的用户，直接复制完整模板再做定制。

该目录职责不是运行时执行，而是为 `SKILL.md` 提供落地示例与操作脚手架：

- `agent-creation-prompt.md`：强调 AI 辅助生成流程（`plugins/plugin-dev/skills/agent-development/examples/agent-creation-prompt.md:5-53`）。
- `complete-agent-examples.md`：提供 4 个完整 agent 样板（Code Review/Test/Docs/Security），可直接改造（`plugins/plugin-dev/skills/agent-development/examples/complete-agent-examples.md:5-427`）。

从引用关系看，examples 由 `agent-development/SKILL.md` 作为官方参考入口显式指向（`plugins/plugin-dev/skills/agent-development/SKILL.md:248,387-393`），并被 `plugin-dev/README.md` 计入该技能资源清单（`plugins/plugin-dev/README.md:154-171`）。

## 功能点目的

### 1. 把抽象规范转为可执行步骤

`agent-creation-prompt.md` 把创建流程拆成 4 步（描述需求、发送生成提示、接收 JSON、转成 agent 文件），降低上手成本（`.../agent-creation-prompt.md:7-53`）。

### 2. 固化最小数据契约

文档要求模型输出固定 JSON 三元组：`identifier`、`whenToUse`、`systemPrompt`（`.../agent-creation-prompt.md:31-37`）。这与参考文档中的系统提示词契约一致（`plugins/plugin-dev/skills/agent-development/references/agent-creation-system-prompt.md:55-60`）。

### 3. 提供“可直接粘贴”的生产级样本

`complete-agent-examples.md` 直接给出完整 frontmatter + system prompt + 边界处理，覆盖四类高频职责（`.../complete-agent-examples.md:5-386`），减少从零写 prompt 的不确定性。

### 4. 统一触发描述写法

两个 examples 都使用 `<example> + <commentary>` 结构表达触发逻辑（如 `.../agent-creation-prompt.md:78-97`，`.../complete-agent-examples.md:14-42`），与触发参考规范一致（`plugins/plugin-dev/skills/agent-development/references/triggering-examples.md:9-19`）。

### 5. 把校验前置到交付流程

examples 明确要求生成后执行 `validate-agent.sh`（`.../agent-creation-prompt.md:195-205`，`.../complete-agent-examples.md:417-425`），与 `create-plugin` Phase 5/6 的 agent 校验流程一致（`plugins/plugin-dev/commands/create-plugin.md:192-200,252-256`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程 A：AI 辅助创建（文档化工作流）

1. 输入自然语言需求（角色、触发时机、职责）。
2. 使用固定指令让模型只返回 JSON（`.../agent-creation-prompt.md:17-23`）。
3. 按字段映射生成 `agents/[identifier].md`，填充 `name/description/model/color/tools` + prompt body（`.../agent-creation-prompt.md:41-53`）。
4. 运行结构校验，失败则迭代（`.../agent-creation-prompt.md:195-219`）。

### 关键流程 B：模板拷贝式创建

1. 从 `complete-agent-examples.md` 选取最接近场景的模板（`.../complete-agent-examples.md:5-386`）。
2. 修改触发 examples、流程步骤、输出格式、工具集（`.../complete-agent-examples.md:388-425`）。
3. 执行校验并做触发手测。

### 数据结构与约束

1. JSON 输出契约：
   - `identifier`：agent 标识。
   - `whenToUse`：以 `Use this agent when...` 起始并含 `<example>`。
   - `systemPrompt`：第二人称、结构化指令。
   参考：`plugins/plugin-dev/skills/agent-development/references/agent-creation-system-prompt.md:55-60`。

2. Agent frontmatter 结构（映射目标）：
   - `name`
   - `description`
   - `model`
   - `color`
   - `tools`（可选）
   参考模板：`plugins/plugin-dev/skills/agent-development/examples/agent-creation-prompt.md:43-53`。

3. 触发协议（软协议）：
   - `<example>` 内应含 Context/user/assistant/commentary/触发动作。
   规范来源：`plugins/plugin-dev/skills/agent-development/references/triggering-examples.md:9-19,80-121`。

### 命令与脚本

examples 给出的关键命令：

```bash
./scripts/validate-agent.sh agents/your-agent.md
```

见 `plugins/plugin-dev/skills/agent-development/examples/agent-creation-prompt.md:199-202`。

在仓库根目录实际可用命令是：

```bash
bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh <agent-file>
```

脚本校验能力包括 frontmatter 边界、必填字段、名称/颜色/模型约束、prompt 长度与二人称检查（`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:25-217`）。

## 关键代码路径与文件引用

### 目标目录核心文件

- `plugins/plugin-dev/skills/agent-development/examples/agent-creation-prompt.md`
  - 4 步 AI 创建流程：`5-53`
  - JSON 契约示例：`31-37,63-68,142-147`
  - 校验与迭代：`195-238`

- `plugins/plugin-dev/skills/agent-development/examples/complete-agent-examples.md`
  - Code reviewer 模板：`5-112`
  - Test generator 模板：`114-209`
  - Docs generator 模板：`211-294`
  - Security analyzer 模板：`296-386`
  - 定制与落地步骤：`388-427`

### 上下文调用方与被调用方

- 调用方（引用 examples 的入口文档）：
  - `plugins/plugin-dev/skills/agent-development/SKILL.md:248,389-393`
  - `plugins/plugin-dev/README.md:154-171`

- 间接调用链（examples 指导的执行流最终落到这些组件）：
  - Agent 生成代理：`plugins/plugin-dev/agents/agent-creator.md:37-123`
  - 创建工作流：`plugins/plugin-dev/commands/create-plugin.md:192-200,252-256`
  - 插件校验代理：`plugins/plugin-dev/agents/plugin-validator.md:86-96`
  - 结构校验脚本：`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:1-217`

### 关联参考资料（被 examples 复用的方法学）

- `plugins/plugin-dev/skills/agent-development/references/agent-creation-system-prompt.md:7-71,91-120`
- `plugins/plugin-dev/skills/agent-development/references/triggering-examples.md:9-19,123-189`
- `plugins/plugin-dev/skills/agent-development/references/system-prompt-design.md:5-38,40-132`

## 依赖与外部交互

### 内部依赖

1. 依赖 `agent-development/SKILL.md` 作为导航与规范入口（`.../SKILL.md:377-415`）。
2. 依赖 `references/` 提供触发、prompt 结构、JSON 契约基线。
3. 依赖 `scripts/validate-agent.sh` 做静态质量门禁。

### 与其他模块交互

1. 与 `/plugin-dev:create-plugin`：examples 提供 Phase 5 产出 agent 时的模板知识，最终由 workflow 组织执行（`plugins/plugin-dev/commands/create-plugin.md:153-200`）。
2. 与 `agent-creator`：examples 与 agent-creator 的方法论同构（提取意图、设计触发样例、生成系统提示）（`plugins/plugin-dev/agents/agent-creator.md:41-101`）。
3. 与 `plugin-validator`：examples 产物会被 plugin-validator 的“Validate Agents”步骤检查（`plugins/plugin-dev/agents/plugin-validator.md:86-96`）。

### 配置、测试、脚本现状

1. 配置层：无独立配置文件；通过 markdown 模板内 frontmatter 字段传递配置。
2. 测试层：该目录无自动化测试；依赖脚本校验 + 人工触发场景测试（`plugins/plugin-dev/skills/agent-development/SKILL.md:307-327`）。
3. 脚本层：目录自身无脚本；对外依赖 `../scripts/validate-agent.sh`。

## 风险、边界与改进建议

### 风险 1（高）：示例命令路径对“仓库根目录执行”不友好

- examples 使用 `./scripts/validate-agent.sh`（`.../agent-creation-prompt.md:199-202`，`.../complete-agent-examples.md:423`）。
- 仓库中仅存在 `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`，根目录没有 `scripts/validate-agent.sh`。

改进建议：

1. 在 examples 同时给出“在 skill 目录执行”和“在仓库根目录执行”两个命令版本。
2. 或统一改成绝对相对路径：`bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh ...`。

### 风险 2（中）：示例与校验器存在潜在规范漂移

- examples 强调 `<example>` 多段触发描述，但校验器当前仅抓 `description:` 首行（`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:91,109-111`），对多行描述识别不足。

改进建议：

1. 在 examples 中建议使用 YAML 块标量写法并补充注意事项。
2. 校验器改为完整解析多行 description（`yq` 或更稳健的 `awk` 方案）。

### 风险 3（中）：文档内容重复导致维护成本上升

- `examples`、`references/agent-creation-system-prompt.md`、`agents/agent-creator.md` 都维护相近流程与范式，长期容易分叉。

改进建议：

1. 引入“单一权威模板”源文件，其他文档只做引用。
2. 增加文档一致性检查脚本（关键片段比对）。

### 风险 4（中）：模板覆盖面有限

- 当前仅 4 个样例，尚未覆盖多 agent 协同、MCP 工具调用约束、跨目录 namespacing 等复杂场景。

改进建议：

1. 增加 orchestration 类与 MCP 集成类 agent 完整示例。
2. 增加“反例”章节，明确何时不应触发 agent。

### 边界说明

1. 本目录是文档模板层，不参与运行时自动发现与调度实现。
2. 其质量保证依赖外部脚本与人工验证，不具备 CI 级强制门禁。
3. 示例关注“结构与触发表达”，不保证特定业务领域的事实正确性或安全完备性。
