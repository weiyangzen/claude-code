# frontmatter-reference.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md` 是 command-development 技能里最核心的“命令元数据契约”文档。它的职责不是教用户写具体业务逻辑，而是定义 slash command 文件头（YAML frontmatter）的协议边界，保证命令可发现、可控权、可维护。

在上下文链路中，它被以下调用方消费：

1. `plugins/plugin-dev/skills/command-development/SKILL.md:112-194,832-833`：主技能先给概览，再把字段细节下钻到该参考文档。
2. `plugins/plugin-dev/skills/command-development/README.md:47,100`：作为 references 首要条目公开。
3. `plugins/plugin-dev/commands/create-plugin.md:183-189`：创建命令阶段要求补全 `description`、`argument-hint`、`allowed-tools`，其语义基线来自本文件。
4. 命令校验流程（间接调用）：`plugins/plugin-dev/agents/plugin-validator.md:76-84` 在验证命令 frontmatter 时与本文件字段定义对齐（但存在一处不一致，见风险章节）。

## 功能点目的

本文件围绕 5 个字段给出“类型 + 默认值 + 何时使用 + 错误样例 + 检查清单”，目的如下：

1. `description`（`frontmatter-reference.md:24-59`）
- 解决 `/help` 可发现性问题，要求短、动词开头、语义明确。

2. `allowed-tools`（`frontmatter-reference.md:60-129`）
- 解决权限最小化与命令可执行性平衡问题，支持字符串与数组两种格式。

3. `model`（`frontmatter-reference.md:130-195`）
- 解决成本/速度/能力分层选择问题，用于命令级模型路由。

4. `argument-hint`（`frontmatter-reference.md:196-268`）
- 解决参数接口可读性与自动补全问题，建立参数语义文档。

5. `disable-model-invocation`（`frontmatter-reference.md:270-323`）
- 解决高风险命令自动触发安全问题，强制转为人工显式调用。

此外，`Complete Examples`（`324-414`）和 `Validation`（`416-463`）把抽象字段转成“可复制模板 + 预提交流程”。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 协议结构

该文档定义的协议形态为：

```markdown
---
description: ...
allowed-tools: ...
model: ...
argument-hint: ...
disable-model-invocation: ...
---
```

关键数据约束：

1. `allowed-tools` 可为 `Read, Write` 这种逗号字符串，或 YAML 数组（`62-86`）。
2. `model` 值域限定 `sonnet|opus|haiku`（`135-148`）。
3. `argument-hint` 约定 `[...] [...]` 槽位格式，并要求与 `$1/$2` 顺序一致（`205-236`）。
4. `disable-model-invocation` 默认 `false`，true 时只允许用户手动 `/command`（`309-317`）。

### 2) 字段到行为的映射流程

1. 解析命令 frontmatter。
2. 计算缺省值（如未写 `description` 则取首行，未写 `allowed-tools` 则继承会话权限，见 `28,64`）。
3. 在运行期约束工具调用能力（`allowed-tools`）。
4. 在命令发现/展示阶段影响 `/help` 文案（`description`）。
5. 在执行阶段决定模型和是否允许程序化触发（`model`、`disable-model-invocation`）。

### 3) 验证与错误处理机制

文档明确了三类常见错误（`420-444`）：

1. YAML 语法问题。
2. `Bash` 规格不完整（未使用 `Bash(prefix:*)`）。
3. 非法模型名。

并提供提交前 checklist（`445-453`）作为轻量测试协议。

## 关键代码路径与文件引用

核心文件：

1. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:1-463`

关键调用方：

1. `plugins/plugin-dev/skills/command-development/SKILL.md:112-194,832-833`
2. `plugins/plugin-dev/skills/command-development/README.md:47,100`
3. `plugins/plugin-dev/commands/create-plugin.md:183-189`
4. `plugins/plugin-dev/agents/plugin-validator.md:76-84`

关键上下游对照：

1. 示例命令落地参考：`plugins/plugin-dev/skills/command-development/examples/simple-commands.md`、`plugins/plugin-dev/skills/command-development/examples/plugin-commands.md`
2. 与插件能力字段联动：`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md`

## 依赖与外部交互

1. 运行时依赖 Claude Code slash command frontmatter 解析能力。
2. 安全边界依赖工具权限系统（`allowed-tools`）与 Bash 过滤语法。
3. 文档层对外并不直接调用脚本；但其规则被命令创建与校验流程消费：
- 命令创建：`plugins/plugin-dev/commands/create-plugin.md`
- 插件校验：`plugins/plugin-dev/agents/plugin-validator.md`
4. 与其他参考文档的接口关系：
- 与 `interactive-commands.md` 共同决定“参数 vs 问答”边界。
- 与 `plugin-features-reference.md` 共同决定插件命令下的 `allowed-tools` 组合策略。

## 风险、边界与改进建议

### 风险

1. `allowed-tools` 规范与校验器不一致：
- 本文件允许“字符串或数组”（`62-86`）。
- `plugin-validator` 当前写法要求数组（`plugin-validator.md:82`）。
- 结果：合法命令可能被误报。

2. `model` 值域硬编码（`135-148`）存在演进滞后风险。

3. 样例中出现 `allowed-tools: "*"`（`104`），与最小权限原则存在冲突。

### 边界

1. 该文件是规范文档，不是解析器实现。
2. 文档不保证“写了字段就业务正确”，只保证 frontmatter 层面的契约。

### 改进建议

1. 为 frontmatter 提供机器可校验 schema（JSON Schema 或脚本），并在 `plugin-validator` 复用同一规则源。
2. 把 `model` 值域改为“运行时可配置/可扩展列表”，避免文档与产品脱节。
3. 在示例层默认使用受限 Bash 过滤，把 `*` 放入“特例”章节而非常规模板。
