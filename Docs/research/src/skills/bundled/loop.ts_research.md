# loop.ts 研究文档

## 场景与职责

`loop.ts` 实现了 `/loop` 内置技能，用于将用户的提示或 slash 命令设置为按指定时间间隔重复执行。它是 Kairos 定时任务系统面向用户的入口之一，将自然语言描述的时间间隔（如 "5m"、"every 2 hours"）转换为 cron 表达式，并通过 `CronCreate` 工具创建周期性任务。

## 功能点目的

1. **自然语言解析**：从用户输入中提取时间间隔和实际提示内容。
2. **间隔到 cron 的转换**：将 `Ns`/`Nm`/`Nh`/`Nd` 格式转换为标准 5 字段 cron 表达式。
3. **任务创建引导**：生成详细的系统提示，指导模型调用 `CronCreate` 工具并立即执行一次任务。
4. **用法提示**：无参数或参数为空时返回使用说明。

## 具体技术实现

### 关键流程

- `registerLoopSkill()` → `registerBundledSkill({ name: 'loop', ... })`
- `getPromptForCommand(args)` 入口：
  1. `args.trim()` 判空 → 返回 `USAGE_MESSAGE`
  2. 否则返回 `buildPrompt(trimmed)`

### 解析规则（优先级顺序）

1. **前导 token**：若第一个空白分隔的 token 匹配 `^\d+[smhd]$`（如 `5m`、`2h`），则作为 interval，其余为 prompt。
2. **尾部 "every" 子句**：若输入以 `every <N><unit>` 或 `every <N> <unit-word>` 结尾，提取为 interval 并剥离。
3. **默认**：interval 为 `10m`，整个输入作为 prompt。

### Cron 转换表

| 间隔模式 | Cron 表达式 | 说明 |
|---------|------------|------|
| `Nm` (N ≤ 59) | `*/N * * * *` | 每 N 分钟 |
| `Nm` (N ≥ 60) | `0 */H * * *` | 转为小时（H=N/60，须整除 24） |
| `Nh` (N ≤ 23) | `0 */N * * *` | 每 N 小时 |
| `Nd` | `0 0 */N * *` | 每 N 天午夜 |
| `Ns` | `ceil(N/60)m` | cron 最小粒度为 1 分钟 |

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'loop'` |
| `argumentHint` | `'[interval] <prompt>'` |
| `userInvocable` | `true` |
| `isEnabled` | `isKairosCronEnabled` |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/loop.ts`
- 注册入口：`src/skills/bundled/index.ts`（条件注册：`feature('AGENT_TRIGGERS')`）
- 核心注册器：`src/skills/bundledSkills.ts`
- Cron 工具常量：`src/tools/ScheduleCronTool/prompt.ts`（`CRON_CREATE_TOOL_NAME`、`CRON_DELETE_TOOL_NAME`、`DEFAULT_MAX_AGE_DAYS`、`isKairosCronEnabled`）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `isKairosCronEnabled` | 统一门控，结合构建时 `feature('AGENT_TRIGGERS')` 与运行时 GrowthBook `tengu_kairos_cron` |
| `CRON_CREATE_TOOL_NAME` / `CRON_DELETE_TOOL_NAME` | 在 prompt 中引用正确的工具名 |
| `DEFAULT_MAX_AGE_DAYS` | 告知用户循环任务自动过期时间 |
| `registerBundledSkill` | 注册技能 |

- **无直接网络调用**：仅生成 prompt，实际的任务调度由 `CronCreate` 工具通过 `cronScheduler`/`cronTasks` 模块完成。
- **与 durable cron 的关系**：`/loop` 创建的 cron 任务默认是 session-only（`durable` 由模型根据 prompt 指令决定，但 `/loop` 的 prompt 未显式要求 durable）。

## 风险、边界与改进建议

1. **边界：cron 粒度限制**：秒级间隔会被强制进位到分钟，且非整除的分钟/小时数会产生不均匀的触发间隔。当前实现要求模型向用户说明并取最近的可表达值，但 prompt 中的规则较复杂，模型可能执行不一致。
2. **边界：空 prompt 拦截**：`buildPrompt` 会在 prompt 为空时要求模型展示用法并停止，不调用 `CronCreate`。这是纯 prompt 级约束，依赖模型遵循。
3. **风险：立即执行与 cron 创建的原子性**：prompt 要求模型 "先创建 cron，再立即执行 parsed prompt"，但这两个动作是独立的工具调用，存在创建成功但立即执行失败（或相反）的中间状态，用户可能看到不一致的结果。
4. **改进建议**：
   - 在 `loop.ts` 中增加一个辅助函数用于 interval 到 cron 的确定性转换，并在 prompt 中要求模型直接调用该逻辑（或通过工具参数传递），减少模型自由发挥导致的错误。
   - 考虑将 `/loop` 与 `CronCreate` 工具深度集成，在工具层直接支持 `loop` 语义（如自动追加 `recurring: true` 和默认 interval），而不是完全依赖 prompt 引导。
   - 增加对 "每工作日"、"每周一" 等常见自然语言表达的显式解析规则，提升用户体验。
