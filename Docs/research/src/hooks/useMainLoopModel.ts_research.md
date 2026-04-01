# useMainLoopModel.ts 深度研究文档

## 场景与职责

`useMainLoopModel` 是一个用于获取和监听主循环模型（Main Loop Model）的 React 钩子。它处理模型配置的动态变化，特别是与 GrowthBook 功能标志系统的集成。

### 核心场景

1. **模型选择**：根据用户设置和订阅级别确定使用哪个 AI 模型
2. **动态更新**：当 GrowthBook 功能标志刷新时重新解析模型
3. **别名解析**：支持模型别名（如 'sonnet', 'opus'）到实际模型 ID 的转换

### 与其他组件的关系

- 被 `REPL.tsx` 和其他组件使用，获取当前会话使用的模型
- 与 `GrowthBook` 功能标志系统集成，支持动态模型配置
- 与 `AppState` 集成，读取用户模型设置

---

## 功能点目的

### 1. 模型解析优先级

模型解析遵循以下优先级（从高到低）：

1. `mainLoopModelForSession` - 会话期间通过 `/model` 命令设置的模型
2. `mainLoopModel` - 启动时通过 `--model` 标志或环境变量设置的模型
3. `getDefaultMainLoopModelSetting()` - 基于用户订阅的默认模型

### 2. GrowthBook 集成

- 监听 `onGrowthBookRefresh` 事件
- 当功能标志更新时强制重新渲染
- 重新解析模型别名（如 `tengu_ant_model_override`）

### 3. 模型别名支持

支持的别名：
- `'sonnet'` → 默认 Sonnet 模型
- `'opus'` → 默认 Opus 模型
- `'haiku'` → 默认 Haiku 模型
- `'best'` → 最佳可用模型
- `'opusplan'` → 计划模式使用 Opus，其他使用 Sonnet

---

## 具体技术实现

### 关键数据结构

```typescript
export type ModelName = string
export type ModelAlias = 'sonnet' | 'opus' | 'haiku' | 'best' | 'opusplan'
export type ModelSetting = ModelName | ModelAlias | null
```

### 核心实现

```typescript
export function useMainLoopModel(): ModelName {
  // 从 AppState 获取模型设置
  const mainLoopModel = useAppState(s => s.mainLoopModel)
  const mainLoopModelForSession = useAppState(s => s.mainLoopModelForSession)
  
  // 订阅 GrowthBook 刷新事件
  const [, forceRerender] = useReducer(x => x + 1, 0)
  useEffect(() => onGrowthBookRefresh(forceRerender), [])
  
  // 解析模型
  const model = parseUserSpecifiedModel(
    mainLoopModelForSession ?? 
    mainLoopModel ?? 
    getDefaultMainLoopModelSetting()
  )
  return model
}
```

### 依赖函数

#### `parseUserSpecifiedModel`

位于 `src/utils/model/model.ts`：

```typescript
export function parseUserSpecifiedModel(modelInput: ModelName | ModelAlias): ModelName {
  const normalizedModel = modelInput.trim().toLowerCase()
  
  // 检查 [1m] 后缀（1M 上下文窗口）
  const has1mTag = has1mContext(normalizedModel)
  const modelString = has1mTag 
    ? normalizedModel.replace(/\[1m]$/i, '').trim() 
    : normalizedModel
  
  // 解析别名
  if (isModelAlias(modelString)) {
    switch (modelString) {
      case 'opusplan':
        return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
      case 'sonnet':
        return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
      case 'haiku':
        return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
      case 'opus':
        return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
      case 'best':
        return getBestModel()
    }
  }
  
  // 处理内部模型（Ant-only）
  if (process.env.USER_TYPE === 'ant') {
    const antModel = resolveAntModel(baseAntModel)
    if (antModel) return antModel.model + suffix
  }
  
  return modelInputTrimmed
}
```

#### `getDefaultMainLoopModelSetting`

```typescript
export function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Anthropic 内部员工
  if (process.env.USER_TYPE === 'ant') {
    return getAntModelOverrideConfig()?.defaultModel ?? getDefaultOpusModel() + '[1m]'
  }
  
  // Max 订阅用户
  if (isMaxSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  
  // Team Premium 用户
  if (isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  
  // 其他用户（PAYG, Pro, Enterprise, Team Standard）
  return getDefaultSonnetModel()
}
```

---

## 关键代码路径与文件引用

```
src/hooks/useMainLoopModel.ts
├── useMainLoopModel()             # 行 13-34: 主钩子
│   ├── useAppState - mainLoopModel # 行 14
│   ├── useAppState - mainLoopModelForSession # 行 15
│   ├── useReducer - forceRerender  # 行 25
│   ├── useEffect - GrowthBook 订阅 # 行 26
│   └── parseUserSpecifiedModel    # 行 28-32
```

### 依赖文件

```
src/utils/model/model.ts
├── parseUserSpecifiedModel()      # 行 445-506: 模型解析
├── getDefaultMainLoopModelSetting() # 行 178-200: 默认模型
├── getDefaultOpusModel()          # 行 105-116
├── getDefaultSonnetModel()        # 行 119-128
├── getDefaultHaikuModel()         # 行 131-138
└── resolveAntModel()              # Ant-only 模型解析

src/services/analytics/growthbook.ts
└── onGrowthBookRefresh()          # GrowthBook 刷新事件

src/state/AppState.ts
├── mainLoopModel                  # 用户设置的模型
└── mainLoopModelForSession        # 会话期间覆盖的模型

src/utils/auth.ts
├── isMaxSubscriber()
├── isTeamPremiumSubscriber()
└── isClaudeAISubscriber()
```

---

## 依赖与外部交互

### React Hooks 使用

- `useAppState`: 获取模型设置
- `useReducer`: 强制重渲染机制
- `useEffect`: 订阅 GrowthBook 刷新事件

### 与 GrowthBook 的交互

```typescript
// 订阅刷新事件，强制重新解析模型
const [, forceRerender] = useReducer(x => x + 1, 0)
useEffect(() => onGrowthBookRefresh(forceRerender), [])
```

### 与 AppState 的交互

```typescript
const mainLoopModel = useAppState(s => s.mainLoopModel)
const mainLoopModelForSession = useAppState(s => s.mainLoopModelForSession)
```

### 模型解析流程

```
mainLoopModelForSession ?? mainLoopModel ?? getDefaultMainLoopModelSetting()
    ↓
parseUserSpecifiedModel()
    ↓
ModelName (实际 API 使用的模型 ID)
```

---

## 风险、边界与改进建议

### 已知风险

1. **GrowthBook 延迟**
   - `_CACHED_MAY_BE_STALE` 可能在 GB 初始化前返回旧值
   - 缓解：订阅刷新事件强制重新渲染

2. **模型别名歧义**
   - 不同时间点的 'sonnet' 可能指向不同版本
   - 缓解：文档说明，用户可使用具体模型 ID

3. **订阅状态变化**
   - 用户订阅级别可能在会话期间变化
   - 缓解：每次重新解析时检查当前订阅状态

### 边界情况

| 场景 | 行为 |
|-----|------|
| 无模型设置 | 使用默认模型 |
| 无效别名 | 原样返回（API 可能报错）|
| GB 刷新时 | 重新解析模型 |
| 订阅降级 | 下次解析时切换到可用模型 |

### 改进建议

1. **缓存优化**
   - 缓存解析结果，避免重复解析相同输入
   - 订阅状态变化时失效缓存

2. **验证增强**
   - 验证模型 ID 是否在当前 API 提供商可用
   - 提前提示用户模型不可用

3. **迁移提示**
   - 当默认模型变化时通知用户
   - 提供保持旧模型的选项

4. **遥测**
   - 记录模型切换事件
   - 分析用户使用模式

### 测试建议

1. **单元测试**：
   - 各种别名解析
   - 优先级顺序
   - [1m] 后缀处理

2. **集成测试**：
   - GrowthBook 刷新流程
   - 与 REPL 的集成

3. **边界测试**：
   - 无效输入处理
   - 订阅状态变化
