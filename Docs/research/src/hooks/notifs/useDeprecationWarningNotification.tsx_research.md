# useDeprecationWarningNotification.tsx 深度研究

## 场景与职责

`useDeprecationWarningNotification` 是一个 React Hook，用于在用户使用已弃用的 AI 模型时显示警告通知。当用户选择的模型被标记为弃用时，此 Hook 会检测并显示相应的弃用警告信息。

### 核心场景
1. **模型弃用提示**：用户通过 `/model` 命令或其他方式选择了一个已弃用的模型
2. **重复警告防止**：使用 ref 追踪上次显示的警告，避免重复显示相同的警告
3. **动态模型切换**：当用户切换模型时，自动检测新模型的弃用状态

## 功能点目的

### 1. 弃用模型检测
- 监听当前选中的模型变化
- 通过 `getModelDeprecationWarning()` 获取模型的弃用警告信息
- 仅在非远程模式下显示警告

### 2. 智能重复防止
- 使用 `lastWarningRef` 追踪上次显示的警告内容
- 只有当警告内容变化时才显示新通知
- 当模型切换到非弃用状态时重置追踪

### 3. 高优先级通知
- 使用 `priority: 'high'` 确保用户注意到弃用警告
- 使用 `color: 'warning'` 提供视觉提示

## 具体技术实现

### 关键数据结构

```typescript
// 通知类型
interface Notification {
  key: string
  text: string
  color: 'warning' | 'error' | 'suggestion' | 'text'
  priority: 'low' | 'medium' | 'high' | 'immediate'
}

// 模型弃用信息（由 getModelDeprecationWarning 返回）
type DeprecationWarning = string | null
```

### 核心流程

```
useEffect 监听 model 变化
    ↓
检查是否为远程模式 (getIsRemoteMode)
    ↓
获取模型弃用警告 (getModelDeprecationWarning)
    ↓
判断是否需要显示：
    - 警告存在且与上次不同
    ↓
添加通知 (addNotification)
    ↓
更新 lastWarningRef
    ↓
如果模型不再弃用，重置 lastWarningRef
```

### 关键代码路径

```typescript
export function useDeprecationWarningNotification(model: string): void {
  const { addNotification } = useNotifications()
  const lastWarningRef = useRef<string | null>(null)

  useEffect(() => {
    // 远程模式下禁用
    if (getIsRemoteMode()) return
    
    // 获取弃用警告
    const deprecationWarning = getModelDeprecationWarning(model)
    
    // 显示警告（如果存在且与上次不同）
    if (deprecationWarning && deprecationWarning !== lastWarningRef.current) {
      lastWarningRef.current = deprecationWarning
      addNotification({
        key: "model-deprecation-warning",
        text: deprecationWarning,
        color: "warning",
        priority: "high"
      })
    }
    
    // 重置追踪（如果模型不再弃用）
    if (!deprecationWarning) {
      lastWarningRef.current = null
    }
  }, [model, addNotification])
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `useEffect`, `useRef` | `react` | React Hook API |
| `useNotifications` | `src/context/notifications.js` | 通知系统 |
| `getModelDeprecationWarning` | `src/utils/model/deprecation.js` | 获取模型弃用信息 |
| `getIsRemoteMode` | `src/bootstrap/state.js` | 远程模式检测 |

### 依赖模块详解

#### 1. getModelDeprecationWarning (src/utils/model/deprecation.js)
此函数负责：
- 检查模型是否在弃用列表中
- 返回相应的弃用警告文本
- 可能包含弃用时间表、替代模型建议等信息

#### 2. useNotifications (src/context/notifications.tsx)
提供通知系统的核心功能：
- `addNotification`: 添加通知到队列
- 支持优先级队列
- 支持通知超时和自动清除

#### 3. getIsRemoteMode (src/bootstrap/state.ts)
检测当前是否运行在远程模式（如通过 SSH 或远程控制）：
```typescript
export function getIsRemoteMode(): boolean {
  return STATE.isRemoteMode
}
```

## 风险、边界与改进建议

### 潜在风险

1. **Ref 持久化问题**
   - `lastWarningRef` 是组件级别的 ref
   - 如果组件重新挂载，可能导致重复显示警告
   - 建议：考虑使用全局状态或 sessionStorage 持久化

2. **远程模式检测**
   - 依赖 `getIsRemoteMode()` 的返回值
   - 如果远程模式状态在会话中动态变化，可能导致不一致行为

3. **弃用信息获取失败**
   - 如果 `getModelDeprecationWarning` 抛出异常，整个 effect 会失败
   - 建议：添加 try-catch 保护

### 边界情况

1. **快速模型切换**
   - 如果用户快速切换多个弃用模型，可能显示多个通知
   - 当前实现会显示每个不同的警告

2. **相同弃用信息**
   - 如果两个不同模型的弃用警告文本相同，可能只显示一次
   - 这是预期行为（避免重复）

3. **空模型值**
   - 如果 model 参数为空或 undefined，应该安全处理

### 改进建议

1. **增加错误处理**
   ```typescript
   useEffect(() => {
     if (getIsRemoteMode()) return
     
     try {
       const deprecationWarning = getModelDeprecationWarning(model)
       // ... 现有逻辑
     } catch (error) {
       logForDebugging(`[DeprecationWarning] Failed to check model: ${error}`)
     }
   }, [model, addNotification])
   ```

2. **使用全局状态持久化**
   ```typescript
   // 考虑使用已显示警告的 Set
   const shownWarnings = useRef<Set<string>>(new Set())
   ```

3. **添加分析事件**
   ```typescript
   if (deprecationWarning && !shownWarnings.current.has(model)) {
     logEvent('tengu_model_deprecation_shown', { model })
     shownWarnings.current.add(model)
   }
   ```

4. **考虑通知折叠**
   - 如果用户连续切换多个弃用模型，可以考虑折叠通知
   - 使用通知的 `fold` 功能

### 相关文件引用

- **实现文件**: `src/hooks/notifs/useDeprecationWarningNotification.tsx`
- **通知系统**: `src/context/notifications.tsx`
- **模型弃用工具**: `src/utils/model/deprecation.js`
- **应用状态**: `src/bootstrap/state.ts`
