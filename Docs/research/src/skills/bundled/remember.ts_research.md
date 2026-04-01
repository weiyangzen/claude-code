# remember.ts 研究文档

## 场景与职责

`remember.ts` 实现了 `/remember` 内置技能，用于帮助用户审查、整理和升级自动记忆（auto-memory）条目。该技能分析多层记忆来源（`CLAUDE.md`、`CLAUDE.local.md`、team memory、auto-memory），识别重复、过时、冲突的条目，并向用户提出整理建议，但**不会未经批准直接修改文件**。

## 功能点目的

1. **记忆分层审查**：读取项目根目录的 `CLAUDE.md`/`CLAUDE.local.md`，结合系统提示中已有的 auto-memory 内容，进行跨层对比。
2. **条目分类与晋升**：为每个 auto-memory 条目建议最佳归宿（项目级 `CLAUDE.md`、个人级 `CLAUDE.local.md`、team memory 或保留在 auto-memory）。
3. **冲突检测**：发现不同记忆层之间的重复、矛盾或过时的内容，并提示用户如何决议。
4. **结构化报告**：以 "Promotions / Cleanup / Ambiguous / No action needed" 四类输出报告，便于用户逐条审阅。

## 具体技术实现

### 关键流程

- `registerRememberSkill()`：
  1. 若 `process.env.USER_TYPE !== 'ant'` 直接返回（技能不注册）
  2. 否则 `registerBundledSkill({ name: 'remember', ... })`
- `getPromptForCommand(args)` 入口：
  1. 返回固定的 `SKILL_PROMPT`（若用户有额外参数则追加 `## Additional context`）

### Prompt 结构（SKILL_PROMPT）

1. **Goal**：审查记忆景观，产出变更建议报告，分组按 action type，**不应用变更**。
2. **Step 1: Gather all memory layers**：
   - 读取 `CLAUDE.md`、`CLAUDE.local.md`
   - auto-memory 内容已在系统提示中
3. **Step 2: Classify each auto-memory entry**：
   - 提供四象限分类表（CLAUDE.md / CLAUDE.local.md / Team memory / Stay in auto-memory）
4. **Step 3: Identify cleanup opportunities**：
   - 查找重复、过时、冲突
5. **Step 4: Present the report**：
   - Promotions、Cleanup、Ambiguous、No action needed

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'remember'` |
| `userInvocable` | `true` |
| `isEnabled` | `() => isAutoMemoryEnabled()` |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/remember.ts`
- 注册入口：`src/skills/bundled/index.ts`
- 核心注册器：`src/skills/bundledSkills.ts`
- 自动记忆开关：`src/memdir/paths.ts`（`isAutoMemoryEnabled`）

## 依赖与外部交互

| 依赖 | 作用 |
|------|------|
| `isAutoMemoryEnabled` | 判断自动记忆功能是否开启（环境变量、settings.json、--bare 模式等） |
| `registerBundledSkill` | 注册技能 |

- **无直接文件读取**：`remember.ts` 本身不读取 `CLAUDE.md` 等文件，而是通过在 prompt 中指示模型使用 `Read` 工具去读取。
- **无网络调用**：纯 prompt 生成器。

## 风险、边界与改进建议

1. **边界：Ant-only 限制**：与 `lorem-ipsum`、`skillify`、`stuck` 一样，`remember` 技能仅对内部员工注册。外部用户即使 auto-memory 开启也无法调用 `/remember` 进行记忆整理。
2. **边界：纯 prompt 约束的"不修改文件"**：prompt 中反复强调 "Do NOT modify files without explicit user approval"，但这是软性约束，依赖模型遵循。恶意或错误的模型行为仍可能直接编辑文件。
3. **风险：记忆层对比的准确性**：模型需要同时处理系统提示中的 auto-memory 摘要和本地文件内容，若系统提示中的摘要被截断或压缩，可能导致跨层对比遗漏。
4. **改进建议**：
   - 在 `remember.ts` 中直接读取 `CLAUDE.md`/`CLAUDE.local.md` 的内容并注入 prompt，减少模型自行读取的失败率，同时确保对比的完整性。
   - 增加对 auto-memory 文件路径的显式引用（如 `getAutoMemEntrypoint()`），让模型知道去哪里读取原始记忆日志。
   - 考虑将 `/remember` 的 ant-only 限制放宽，因为记忆整理对所有使用 auto-memory 的用户都有价值。
