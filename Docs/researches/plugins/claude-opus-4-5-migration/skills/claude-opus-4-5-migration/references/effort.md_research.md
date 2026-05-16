# FILE `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md` 研究文档

## 场景与职责

`effort.md` 是 `claude-opus-4-5-migration` skill 的参数参考文档，定位为“迁移后生成质量/成本控制”的细化说明，而不是迁移入口或执行器本身。

在调用链中：
- 上游调用方是 `SKILL.md` 的迁移流程第 4 步（`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:15`）以及末尾 reference 跳转（`.../SKILL.md:105`）。
- 更上游入口来自插件 README 的自然语言触发示例（`plugins/claude-opus-4-5-migration/README.md:11-13`），并通过 marketplace 注册项暴露（`.claude-plugin/marketplace.json:18-26`）。

该文件职责是把 `output_config.effort` 的取值策略和请求字段组织成可直接落地的 API 片段，供执行迁移时按需拷贝。

## 功能点目的

1. 定义 effort 的作用域与成本含义  
- 指明 effort 影响 thinking、文本响应、函数调用三类 token 消耗（`effort.md:7`）。

2. 给出档位与场景映射  
- 通过 `high/medium/low` -> use case 表，建立性能与成本折中基线（`effort.md:9-13`）。

3. 提供三种协议接入模板  
- 分别给出 Python SDK、TypeScript SDK、Raw API 请求体写法（`effort.md:19-56`），减少迁移时字段误填。

4. 明确与 thinking budget 的关系  
- 说明 effort 与 thinking budget 独立，不是互斥参数（`effort.md:58-64`）。

5. 输出操作建议顺序  
- 先定 effort，再定 thinking budget（`effort.md:67-70`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

该文件本身不执行命令，实际是“文档驱动配置流程”：

1. 迁移执行者按 `SKILL.md` 先完成模型串替换和不支持 beta 移除（`SKILL.md:12-29,31-48`）。  
2. 进入 effort 阶段后，依据 `effort.md` 添加 `output_config.effort` 并补 beta 标识（`effort.md:17,24-27,37-40,50-53`）。  
3. 若已有 thinking 配置，保持 thinking 上限策略，避免把 effort 当作 thinking 替代（`effort.md:58-64`）。  
4. 在迁移总结里说明该参数变更（`SKILL.md:16`）。

### 数据结构

1. 策略映射表  
- Markdown 表格表达 `Effort -> Use Case`（`effort.md:9-13`），等价于三值枚举策略。

2. 请求体结构模板  
- SDK 结构：`model`、`max_tokens`、`betas`、`output_config.effort`、`messages`（`effort.md:21-29,34-42`）。  
- Raw API 结构：`anthropic-beta` 头位 + `output_config` 对象（`effort.md:47-55`）。

3. 行为关系规则  
- “effort 独立于 thinking budget”通过两条例子定义（`effort.md:62-63`），是规则级约束而非代码逻辑。

### 协议与命令

- 关键协议字段：
  - `betas=["effort-2025-11-24"]`（SDK）（`effort.md:24,37`）
  - `anthropic-beta: effort-2025-11-24`（Raw API）（`effort.md:50`）
  - `output_config.effort`（`effort.md:25-27,38-40,51-53`）
- 模型 ID 与主 skill 保持 Opus 4.5 版本一致（`effort.md:22,35,48` 与 `SKILL.md:35-38`）。
- 本文件无脚本、无 CLI 命令、无测试 harness。

## 关键代码路径与文件引用

核心文件：
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`

直接调用方（上游）：
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:15`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:105`

同级协作文件：
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`（问题驱动 prompt 修复，不是 effort 参数配置）

插件装载与文档入口：
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`
- `.claude-plugin/marketplace.json:18-26`
- `plugins/claude-opus-4-5-migration/README.md:1-21`

技能加载机制背景（为何 references 按需读取）：
- `plugins/plugin-dev/skills/skill-development/SKILL.md:271-277`

## 依赖与外部交互

### 内部依赖

1. 依赖 `SKILL.md` 决策  
- `effort.md` 不单独触发，只在迁移流程中被引用（`SKILL.md:15,105`）。

2. 依赖插件发现机制  
- marketplace `source` 指向插件目录（`.claude-plugin/marketplace.json:25`），`plugin.json` 提供基础元数据（`plugin.json:2-8`）。

3. 依赖用户目标仓库存在可改 API 调用点  
- 真正被改写的是用户工程代码，不是插件自身文件。

### 外部交互

1. 与 Anthropic API 协议耦合  
- beta 名称和字段路径必须与服务端能力一致（`effort.md:17,24,37,50`）。

2. 与多平台模型迁移策略并行  
- 主 skill 覆盖 1P/Bedrock/Vertex/Azure 模型串（`SKILL.md:33-38`），但 `effort.md` 示例仅给出 1P 形态，平台语义需执行者自行折算。

### 配置/测试/脚本现状

- 配置：仅文档内请求片段，无独立配置文件。
- 测试：插件目录无测试文件（仓库扫描该插件仅 5 个文件）。
- 脚本：无 `scripts/`，无自动校验命令。

## 风险、边界与改进建议

### 风险

1. 默认策略不一致风险  
- `SKILL.md` 第 4 步要求“添加 high effort”（`SKILL.md:15`），但末尾又写“only if user requests it”（`SKILL.md:105`），`effort.md` 开头也偏向“迁移中默认加 high”（`effort.md:3`）。执行者可能出现分歧。

2. 版本漂移风险  
- `effort-2025-11-24` 与模型日期 `20251101` 均为硬编码（`effort.md:17,22,35,48`），上游更新后易过时。

3. 成本失控风险  
- 如果在高并发/简单任务场景盲目使用 `high`，会偏离 `medium/low` 的成本优化目标（`effort.md:11-13,68-70`）。

4. 平台适配空档风险  
- 文档样例主要面向 Anthropic 1P 请求格式；Bedrock/Vertex/Azure 的 effort 字段映射未在此文件显式给出。

### 边界

1. 这是参数参考，不负责自动修改代码。  
2. 只定义 effort 与 thinking 的关系，不定义业务提示词策略。  
3. 不覆盖 Haiku 迁移边界（由 `SKILL.md` 明确限制，`SKILL.md:48`）。

### 改进建议

1. 统一默认策略文案  
- 统一 `SKILL.md:15` 与 `SKILL.md:105` 对 effort 的默认行为，避免“默认开启”和“仅按需开启”并存。

2. 增加多平台示例  
- 在 `effort.md` 补充 Bedrock/Vertex/Azure 的最小可用请求示例，和模型串映射表形成闭环。

3. 增加最小校验脚本  
- 可在插件目录提供检查脚本，扫描迁移后是否包含 `output_config.effort`、是否残留旧模型串、是否错误处理 Haiku。

4. 提供“成本优先”决策树  
- 在 `Recommendations` 下加入按延迟/成本/质量目标选择 `high|medium|low` 的简短决策树，降低人工判断偏差。
