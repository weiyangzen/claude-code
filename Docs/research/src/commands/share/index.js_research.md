# src/commands/share/index.js 研究报告

## 执行摘要

`src/commands/share/index.js` 是一个**存根/占位命令实现**，当前处于完全禁用状态。该文件仅作为命令系统的占位符存在，实际功能（会话转录分享）由 `src/components/FeedbackSurvey/` 目录下的组件系统实现。

---

## 1. 场景与职责

### 1.1 命令定位

| 属性 | 说明 |
|------|------|
| 文件路径 | `src/commands/share/index.js` |
| 命令名称 | `share` |
| 当前状态 | **已禁用** (`isEnabled: () => false`) |
| 可见性 | **隐藏** (`isHidden: true`) |
| 类型 | 存根/占位符 (stub) |
| 实际功能承载 | `src/components/FeedbackSurvey/` 组件系统 |

### 1.2 命令分类

该命令属于 **INTERNAL_ONLY_COMMANDS** 类别（见 `src/commands.ts:244`），这意味着：
- 仅当 `process.env.USER_TYPE === 'ant'` 且非演示模式时才会被加载
- 仅限 Anthropic 内部员工使用
- 不会出现在外部构建版本中

### 1.3 历史与演进

该命令可能曾经是一个活跃的功能命令，用于主动分享会话转录。当前设计将转录分享功能从显式命令调用转变为**被动触发机制**：
- 不再通过 `/share` 命令主动调用
- 转而在特定用户行为后（如满意度调查） probabilistically 触发

---

## 2. 功能点目的

### 2.1 原始设计目的（推测）

基于代码结构和命名，该命令的原始目的可能是：
- 允许用户**主动**分享当前会话的转录给 Anthropic
- 用于产品改进、调试分析或用户反馈收集

### 2.2 当前实际功能

转录分享功能现通过以下方式实现：

| 触发场景 | 实现文件 | 触发条件 |
|----------|----------|----------|
| 会话满意度调查 | `useFeedbackSurvey.tsx` | 用户评分后概率触发 |
| 记忆功能调查 | `useMemorySurvey.tsx` | 提及 memory 且读取记忆文件后 |
| 沮丧情绪检测 | `useFrustrationDetection.tsx` | 检测到用户沮丧表达（ant-only） |
| 压缩后调查 | `usePostCompactSurvey.tsx` | 会话压缩后（不含转录分享） |

### 2.3 分享数据内容

```typescript
{
  trigger: 'bad_feedback_survey' | 'good_feedback_survey' | 'frustration' | 'memory_survey',
  version: MACRO.VERSION,
  platform: process.platform,
  transcript: Message[],           // 标准化后的消息记录
  subagentTranscripts: { [agentId: string]: Message[] },
  rawTranscriptJsonl?: string      // 原始 JSONL 文件内容（≤50MB）
}
```

---

## 3. 具体技术实现

### 3.1 存根实现

```javascript
// src/commands/share/index.js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

**关键属性解析：**

| 属性 | 值 | 说明 |
|------|-----|------|
| `isEnabled` | `() => false` | 函数返回 false，命令始终被禁用 |
| `isHidden` | `true` | 命令在帮助、自动补全中隐藏 |
| `name` | `'stub'` | 占位符名称，非实际命令名 |

### 3.2 命令类型定义

根据 `src/types/command.ts`，该存根符合 `CommandBase` 接口的最小实现：

```typescript
type CommandBase = {
  availability?: CommandAvailability[]
  description: string  // 缺失，但存根可能不需要
  isEnabled?: () => boolean  // 返回 false
  isHidden?: boolean  // true
  name: string  // 'stub'
  aliases?: string[]
  // ... 其他可选属性
}
```

**注意：** 该存根缺少 `description` 和完整的命令类型定义（`type: 'prompt' | 'local' | 'local-jsx'`），这在技术上是不完整的，但由于 `isEnabled: () => false`，它永远不会被实际调用。

### 3.3 命令注册流程

```
src/commands/share/index.js
    ↓
src/commands.ts (line 42)
    import share from './commands/share/index.js'
    ↓
src/commands.ts (line 244)
    INTERNAL_ONLY_COMMANDS 数组
    ↓
src/commands.ts (line 343-345)
    条件加载：process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
    ↓
src/commands.ts (line 258-346)
    COMMANDS() 函数返回的命令列表
    ↓
src/commands.ts (line 476-517)
    getCommands() 过滤后返回（因 isEnabled()=false 被过滤）
```

### 3.4 同类存根命令

以下命令采用相同的存根模式：

| 命令 | 文件路径 | 状态 |
|------|----------|------|
| `share` | `src/commands/share/index.js` | stub |
| `summary` | `src/commands/summary/index.js` | stub |
| `teleport` | `src/commands/teleport/index.js` | stub |
| `ant-trace` | `src/commands/ant-trace/index.js` | stub |
| `perf-issue` | `src/commands/perf-issue/index.js` | stub |
| `good-claude` | `src/commands/good-claude/index.js` | stub |
| `onboarding` | `src/commands/onboarding/index.js` | stub |
| `env` | `src/commands/env/index.js` | stub |
| `issue` | `src/commands/issue/index.js` | stub |
| `ctx_viz` | `src/commands/ctx_viz/index.js` | stub |
| `bughunter` | `src/commands/bughunter/index.js` | stub |

这些命令均为 **INTERNAL_ONLY_COMMANDS** 成员，采用统一的存根实现模式。

---

## 4. 关键代码路径与文件引用

### 4.1 直接引用

| 引用位置 | 行号 | 用途 |
|----------|------|------|
| `src/commands.ts` | 42 | 导入 share 命令 |
| `src/commands.ts` | 244 | 加入 INTERNAL_ONLY_COMMANDS 数组 |

### 4.2 间接引用（通过 INTERNAL_ONLY_COMMANDS）

| 引用位置 | 用途 |
|----------|------|
| `src/commands.ts:343-345` | 条件加载内部命令 |
| `src/components/HelpV2/HelpV2.tsx` | 导入 INTERNAL_ONLY_COMMANDS（未实际使用） |

### 4.3 实际功能相关文件

虽然存根本身无功能，但相关功能实现在：

| 文件 | 功能 |
|------|------|
| `src/components/FeedbackSurvey/submitTranscriptShare.ts` | 转录分享 API 调用 |
| `src/components/FeedbackSurvey/useSurveyState.tsx` | 调查状态管理 |
| `src/components/FeedbackSurvey/useFeedbackSurvey.tsx` | 会话满意度调查 |
| `src/components/FeedbackSurvey/useMemorySurvey.tsx` | 记忆功能调查 |
| `src/components/FeedbackSurvey/TranscriptSharePrompt.tsx` | 转录分享 UI |

---

## 5. 依赖与外部交互

### 5.1 模块依赖

该存根为纯 JavaScript 对象导出，无外部依赖：

```
src/commands/share/index.js
└── 无依赖（纯对象字面量）
```

### 5.2 被依赖关系

```
src/commands.ts
└── import share from './commands/share/index.js'
```

### 5.3 运行时行为

由于 `isEnabled: () => false`，该命令在运行时：
- 不会被添加到可用命令列表
- 不会出现在帮助文档中
- 不会被用户调用
- 不消耗运行时资源

---

## 6. 风险、边界与改进建议

### 6.1 当前风险

| 风险 | 严重程度 | 说明 |
|------|----------|------|
| 类型不完整 | 低 | 缺少 `type` 和 `description` 字段，但 `isEnabled: false` 规避了调用风险 |
| 技术债务 | 中 | 存根文件与目录结构占用维护成本 |
| 功能分散 | 中 | 实际功能分散在 components 目录，与命令目录结构不一致 |

### 6.2 边界情况

1. **模块加载**：虽然命令被禁用，但模块仍会被导入（构建时）
2. **类型检查**：TypeScript 可能因不完整类型而报错（如有类型检查）
3. **命名冲突**：`name: 'stub'` 可能与调试或日志中的其他 stub 混淆

### 6.3 改进建议

#### 短期（维护）

1. **统一存根实现**
   - 考虑创建统一的 `createStubCommand(name)` 工厂函数
   - 确保所有存根命令类型完整（添加 `type: 'local'` 和 `description`）

2. **添加注释**
   ```javascript
   // Stub command - actual functionality moved to src/components/FeedbackSurvey/
   // See useFeedbackSurvey.tsx and submitTranscriptShare.ts for transcript sharing
   export default { isEnabled: () => false, isHidden: true, name: 'stub' };
   ```

#### 中期（重构）

3. **目录清理**
   - 评估是否可以删除存根目录，改为在 `commands.ts` 中内联定义
   - 或移动存根命令到 `src/commands/_internal/stubs/` 统一管理

4. **功能整合**
   - 考虑将转录分享功能重新实现为可激活的命令
   - 允许用户在任意时刻主动触发转录分享（而非仅被动触发）

#### 长期（架构）

5. **功能开关系统**
   - 使用 GrowthBook 或类似功能开关系统替代硬编码的 `isEnabled: () => false`
   - 允许动态启用/禁用命令而无需代码变更

6. **代码生成**
   - 对于大量存根命令，考虑使用代码生成或配置文件驱动
   - 减少样板文件数量

### 6.4 相关决策记录

| 决策 | 状态 | 建议 |
|------|------|------|
| 保留存根 vs 删除 | 待定 | 如功能完全迁移，可考虑删除；如可能恢复，保留存根 |
| 统一存根模式 | 建议 | 所有存根命令应遵循相同模式（类型完整、有注释） |
| 功能归属 | 建议 | 明确文档化实际功能位置，便于维护 |

---

## 附录：文件清单

### 本命令相关

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/commands/share/index.js` | 1 | 存根实现 |

### 实际功能实现（参考）

| 文件 | 说明 |
|------|------|
| `src/components/FeedbackSurvey/submitTranscriptShare.ts` | 转录分享 API 调用 |
| `src/components/FeedbackSurvey/useFeedbackSurvey.tsx` | 会话满意度调查逻辑 |
| `src/components/FeedbackSurvey/useMemorySurvey.tsx` | 记忆功能调查逻辑 |
| `src/components/FeedbackSurvey/TranscriptSharePrompt.tsx` | 转录分享询问 UI |

### 同类存stub命令

| 文件 | 说明 |
|------|------|
| `src/commands/summary/index.js` | 存根 |
| `src/commands/teleport/index.js` | 存根 |
| `src/commands/ant-trace/index.js` | 存根 |
| `src/commands/perf-issue/index.js` | 存根 |
| `src/commands/good-claude/index.js` | 存根 |
| `src/commands/onboarding/index.js` | 存根 |
| `src/commands/env/index.js` | 存根 |
| `src/commands/issue/index.js` | 存根 |
| `src/commands/ctx_viz/index.js` | 存根 |
| `src/commands/bughunter/index.js` | 存根 |

---

*文档生成时间: 2026-04-01*
*研究范围: 代码、配置、测试及实现上下文*
