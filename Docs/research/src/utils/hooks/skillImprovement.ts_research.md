# 研究文档：src/utils/hooks/skillImprovement.ts

> 生成时间：2026-04-01  
> 研究范围：代码、调用方、被调用方、特性开关、状态存储、LLM 交互链路  
> 文件大小：约 8.4 KB

---

## 1. 场景与职责

`skillImprovement.ts` 实现 Claude Code 的**技能自动改进（Skill Improvement）**功能。其核心目标是在用户执行某个 skill 的过程中，自动检测用户表达出的**偏好、修正或补充要求**，并将这些改进建议持久化回 skill 的定义文件（`SKILL.md`），从而让用户在下次运行同一 skill 时获得更符合其意图的体验。

该模块包含两个互补的能力：
1. **检测（Detect）**：作为 `post-sampling hook`，每隔一定轮次分析最近的用户消息，判断是否存在应被记住的改进点；
2. **应用（Apply）**：当用户在 UI 中确认改进建议后，通过侧信道 LLM 调用将改进合并回 skill 文件。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| `createSkillImprovementHook` | 构造一个 `ApiQueryHook`，在 REPL 主线程的每次采样结束后被调用，分析最近 N 条消息并提取改进建议。 |
| `initSkillImprovement` | 在应用启动时（background housekeeping）检查特性开关，若开启则向 `postSamplingHooks` 注册检测钩子。 |
| `applySkillImprovement` | 接收用户确认的改进列表，读取本地 `.claude/skills/<name>/SKILL.md`，调用 LLM 重写文件内容，并写回磁盘。 |
| `formatRecentMessages` | 将消息历史格式化为供 LLM 分析的文本摘要，过滤非 user/assistant 消息并截断至 500 字符。 |
| `findProjectSkill` | 从全局 `invokedSkills` 中定位当前正在执行的、以 `projectSettings:` 为前缀的 skill。 |

---

## 3. 具体技术实现

### 3.1 检测钩子：基于批次的触发策略

```ts
const TURN_BATCH_SIZE = 5
```

检测钩子的触发条件（`shouldRun`）有三重过滤：
1. **`querySource === 'repl_main_thread'`**：只在主线程 REPL 查询中运行，避免在子 agent、hook agent、后台任务等侧信道中重复触发；
2. **`findProjectSkill()` 必须存在**：确保当前会话确实在执行某个 project-level skill；
3. **用户消息数达到批次阈值**：统计 `messages` 中 `type === 'user'` 的数量，只有当 `userCount - lastAnalyzedCount >= 5` 时才触发，并将 `lastAnalyzedCount` 更新为当前值。

> 该状态（`lastAnalyzedCount`、`lastAnalyzedIndex`）通过闭包保存在 `createSkillImprovementHook()` 内部，因此**不依赖 AppState**，也不会在进程重启后恢复。这意味着若应用重启，计数会从头开始，但这不影响功能正确性，只是可能延迟一次检测。

### 3.2 LLM 提示词设计

`buildMessages` 构造了一条单用户消息，包含：
- `<skill_definition>`：当前 skill 的完整内容（`projectSkill.content`）；
- `<recent_messages>`：自上次分析以来的新消息（`context.messages.slice(lastAnalyzedIndex)`）。

模型被指示：
- 寻找用户提出的**新增/修改/删除步骤**、**偏好设置**、**纠正性指令**；
- 忽略一次性闲聊和 skill 已具备的能力；
- 输出格式必须是 `<updates>[...]</updates>` 包裹的 JSON 数组，元素结构为 `{"section": "...", "change": "...", "reason": "..."}`。

### 3.3 结果解析与状态落盘

`parseResponse` 使用 `extractTag(content, 'updates')` 提取标签内容，再通过 `jsonParse` 反序列化。若解析失败，返回空数组 `[]`。

`logResult` 在检测到非空建议时做两件事：
1. **埋点**：发送 `tengu_skill_improvement_detected` 事件，附带 `updateCount`、UUID 与 skill 名称（`_PROTO_skill_name` 标记为 PII，进入特权 BQ 列）；
2. **写入 AppState**：将建议暂存到 `appState.skillImprovement.suggestion`，供 UI 层展示确认弹窗。

```ts
context.toolUseContext.setAppState(prev => ({
  ...prev,
  skillImprovement: {
    suggestion: { skillName, updates: result.result },
  },
}))
```

### 3.4 应用改进：fire-and-forget 的 LLM 文件重写

`applySkillImprovement` 是一个 `async` 函数，设计上**不阻塞主对话**：

1. **定位文件**：`join(getCwd(), '.claude', 'skills', skillName, 'SKILL.md')`；
2. **读取当前内容**：通过动态导入 `fs/promises` 读取；失败则 `logError` 并直接返回；
3. **构造改进列表**：将 `updates` 格式化为 `- {section}: {change}`；
4. **调用 `queryModelWithoutStreaming`**：
   - 使用 `getSmallFastModel()` 轻量模型；
   - `temperatureOverride: 0` 保证确定性；
   - `thinkingConfig: { type: 'disabled' }` 关闭思考模式；
   - `querySource: 'skill_improvement_apply'` 便于追踪；
   - 提示词要求模型保留 frontmatter、保留格式、仅自然融入改进，并输出在 `<updated_file>` 标签内；
5. **写回磁盘**：提取 `<updated_file>` 内容后 `fs.writeFile`；失败仅 `logError`，不向用户抛异常。

> 该函数被 `useSkillImprovementSurvey.ts` 的 `handleSelect` 以 `void applySkillImprovement(...).then(...)` 方式调用，充分体现了 fire-and-forget 的设计意图。

### 3.5 特性开关

```ts
export function initSkillImprovement(): void {
  if (
    feature('SKILL_IMPROVEMENT') &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_copper_panda', false)
  ) {
    registerPostSamplingHook(createSkillImprovementHook())
  }
}
```

- `feature('SKILL_IMPROVEMENT')`：编译期/Bundle 级特性开关（来自 `bun:bundle`）；
- `getFeatureValue_CACHED_MAY_BE_STALE('tengu_copper_panda', false)`：运行时 GrowthBook feature flag，用于灰度发布。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件导出的符号

| 符号 | 类型 | 说明 |
|------|------|------|
| `SkillUpdate` | type alias | 改进建议项 `{ section, change, reason }` |
| `createSkillImprovementHook` | function (内部) | 构造检测用的 `ApiQueryHook` |
| `initSkillImprovement` | function | 初始化入口，注册 post-sampling hook |
| `applySkillImprovement` | function | 应用改进到 skill 文件 |

### 4.2 上游调用方

| 文件 | 调用符号 | 用途 |
|------|----------|------|
| `src/utils/backgroundHousekeeping.ts` | `initSkillImprovement` | 应用启动时初始化所有后台/延迟任务，包括 skill improvement。 |
| `src/hooks/useSkillImprovementSurvey.ts` | `applySkillImprovement` | UI 弹窗中用户点击“应用”后触发实际文件改写。 |

### 4.3 下游消费与状态联动

| 文件 | 关联符号/路径 | 用途 |
|------|---------------|------|
| `src/state/AppStateStore.ts` | `skillImprovement.suggestion` | 存储检测到的建议，供 UI 订阅。 |
| `src/utils/hooks/apiQueryHookHelper.ts` | `createApiQueryHook`, `ApiQueryHookConfig` | 检测钩子的底层执行框架。 |
| `src/utils/hooks/postSamplingHooks.ts` | `registerPostSamplingHook` | 将检测钩子挂接到采样后链路。 |
| `src/services/api/claude.ts` | `queryModelWithoutStreaming` | 应用阶段调用 LLM 重写文件。 |
| `src/bootstrap/state.ts` | `getInvokedSkillsForAgent` | 获取当前会话已调用的 skills。 |
| `src/services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE` | 运行时特性开关读取。 |

---

## 5. 依赖与外部交互

### 5.1 直接依赖

| 模块 | 用途 |
|------|------|
| `bun:bundle` | `feature('SKILL_IMPROVEMENT')` 编译期开关 |
| `src/bootstrap/state.js` | `getInvokedSkillsForAgent` |
| `src/services/analytics/growthbook.js` | GrowthBook 运行时开关 |
| `src/services/analytics/index.js` | `logEvent` 埋点 |
| `src/services/api/claude.js` | `queryModelWithoutStreaming` |
| `src/Tool.js` | `getEmptyToolPermissionContext` |
| `src/types/message.js` | `Message` 类型 |
| `src/utils/abortController.js` | `createAbortController` |
| `src/utils/array.js` | `count` 统计 user 消息数 |
| `src/utils/cwd.js` | `getCwd` |
| `src/utils/errors.js` | `toError` |
| `src/utils/log.js` | `logError` |
| `src/utils/messages.js` | `createUserMessage`, `extractTag`, `extractTextContent` |
| `src/utils/model/model.js` | `getSmallFastModel` |
| `src/utils/slowOperations.js` | `jsonParse` |
| `src/utils/systemPromptType.js` | `asSystemPrompt` |
| `src/utils/hooks/apiQueryHookHelper.ts` | `createApiQueryHook`, `ApiQueryHookConfig` |
| `src/utils/hooks/postSamplingHooks.ts` | `registerPostSamplingHook` |

### 5.2 外部系统交互

- **LLM API**：检测阶段与应用阶段各发起一次 `queryModelWithoutStreaming` 调用；
- **文件系统**：`applySkillImprovement` 直接读写用户工作目录下的 `.claude/skills/<name>/SKILL.md`；
- **Analytics**：通过 `logEvent` 上报检测与应用相关的事件到内部数据仓库；
- **GrowthBook**：通过 `tengu_copper_panda` flag 控制功能灰度。

---

## 6. 风险、边界与改进建议

### 6.1 风险

1. **文件路径硬编码**
   - `applySkillImprovement` 假设 skill 文件一定位于 `join(getCwd(), '.claude', 'skills', skillName, 'SKILL.md')`。若未来 skill 存储路径变更（如支持全局 skill、marketplace skill 等），此处会静默失败。

2. **Fire-and-forget 导致用户无感知失败**
   - 无论是 LLM 调用失败、标签提取失败还是文件写入失败，均只通过 `logError` 记录，不向用户展示任何错误提示。用户可能误以为“应用成功”，但实际上文件未被修改。

3. **闭包状态在进程重启后丢失**
   - `lastAnalyzedCount` 与 `lastAnalyzedIndex` 是内存闭包变量，进程重启后会导致：
     - 立即再次分析（因为 `userCount - 0` 很容易 >= 5）；
     - 或分析范围扩大到整个历史（`slice(0)`），增加 token 消耗和误检概率。

4. **无去重与冲突解决机制**
   - 若用户在连续多个批次中反复表达同一偏好，检测钩子可能产生重复建议；`applySkillImprovement` 的 LLM 提示词虽要求“自然融入”，但没有显式的去重或冲突消解逻辑。

5. **GrowthBook 缓存值可能过时**
   - `initSkillImprovement` 使用 `_CACHED_MAY_BE_STALE` 版本的 feature flag。若用户在会话中途被加入/移出灰度组，该变更不会生效，直到下次启动。

### 6.2 边界

- **仅支持 projectSettings skill**：`findProjectSkill` 显式过滤 `skillPath.startsWith('projectSettings:')`，用户级或 marketplace skill 的改进不在当前范围内。
- **仅主线程生效**：`querySource !== 'repl_main_thread'` 时直接返回 `false`，子 agent 或后台任务中的 skill 执行不会被分析。
- **仅分析 user/assistant 消息**：`formatRecentMessages` 过滤掉 `tool_result`、`system` 等其他消息类型。
- **无工具调用**：检测钩子的 `useTools: false`，模型只能基于文本推理，不能读取文件或执行命令来验证改进的合理性。

### 6.3 改进建议

1. **将分析状态持久化到 AppState 或磁盘**
   - 把 `lastAnalyzedCount` 和 `lastAnalyzedIndex` 存入 `appState.skillImprovement.lastAnalyzedCount` 等字段，确保进程重启后不会重复分析或过度分析。

2. **增加应用失败的 UI 反馈**
   - `applySkillImprovement` 应返回 `Promise<boolean>` 或抛出可识别的错误类型，`useSkillImprovementSurvey` 在失败时向用户展示“应用失败，请重试”的系统消息，而非仅在成功时提示。

3. **引入建议去重与合并策略**
   - 在写入 `appState.skillImprovement.suggestion` 前，可对比已有建议的 `section + change` 进行去重；或在 `applySkillImprovement` 前将多个批次的建议合并为一份统一的 diff 再提交给 LLM。

4. **使用 skill 的真实文件路径而非硬编码**
   - 若 `InvokedSkillInfo` 中已包含 skill 的物理路径，应直接使用该路径；否则应在 `findProjectSkill` 时解析并缓存真实路径，避免路径假设僵化。

5. **补充单元测试**
   - 建议补充以下测试：
     - `shouldRun` 的批次阈值逻辑（`userCount` 增长与 `lastAnalyzedCount` 更新）；
     - `buildMessages` 正确截取 `lastAnalyzedIndex` 之后的消息；
     - `parseResponse` 对合法/非法 `<updates>` 标签的解析行为；
     - `applySkillImprovement` 的 mock 文件读写与 LLM 调用链路；
     - `findProjectSkill` 对 `projectSettings:` 前缀的过滤。
