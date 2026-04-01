# useSkillImprovementSurvey.ts 深度研究文档

## 场景与职责

`useSkillImprovementSurvey` 是一个 React Hook，用于管理技能改进建议的调查问卷 UI。当系统检测到用户的偏好或更正可以被永久添加到技能定义中时，显示一个调查问卷让用户确认是否应用这些改进。

### 核心职责

1. **建议检测**: 监听 AppState 中的技能改进建议
2. **UI 状态管理**: 控制调查问卷的显示/隐藏状态
3. **分析事件**: 记录调查问卷的展示和用户响应
4. **改进应用**: 用户确认后应用技能改进
5. **状态清理**: 应用或关闭后清理 AppState

### 使用场景

- **技能执行期间**: 用户在使用技能（可重复流程）时提出偏好或更正
- **改进建议**: 系统分析对话后建议将某些偏好永久化到技能定义
- **用户确认**: 用户确认后，系统自动更新技能文件

---

## 功能点目的

### 1. 建议展示

当 `AppState.skillImprovement.suggestion` 存在时：
- 自动打开调查问卷
- 记录分析事件（仅首次）

### 2. 用户响应处理

支持三种响应：
- **应用改进**: 调用 `applySkillImprovement` 更新技能文件
- **关闭/忽略**: 仅关闭调查问卷，不应用改进
- **分析记录**: 记录用户的响应类型

### 3. 系统消息反馈

应用改进后，向对话中添加系统消息告知用户技能已更新。

---

## 具体技术实现

### 关键数据结构

```typescript
// 建议类型
interface SkillImprovementSuggestion {
  skillName: string
  updates: SkillUpdate[]
}

interface SkillUpdate {
  section: string    // 要修改的步骤/部分
  change: string     // 修改内容
  reason: string     // 触发此修改的用户消息
}

// Hook 返回类型
interface UseSkillImprovementSurveyResult {
  isOpen: boolean                                    // 调查问卷是否打开
  suggestion: SkillImprovementSuggestion | null      // 当前建议
  handleSelect: (selected: FeedbackSurveyResponse) => void  // 响应处理器
}

// 反馈响应类型
 type FeedbackSurveyResponse = 'applied' | 'dismissed' | 'dismissed_forever'
```

### 核心流程

#### 1. 建议检测流程
```
组件渲染
  ↓
检查 suggestion 是否存在
  ↓
存在且 isOpen=false → setIsOpen(true)
  ↓
记录分析事件（仅首次，通过 loggedAppearanceRef 控制）
```

**代码实现**（行 32-51）：
```typescript
// Track the suggestion for display even after clearing AppState
if (suggestion) {
  lastSuggestionRef.current = suggestion
}

// Open when a new suggestion arrives
if (suggestion && !isOpen) {
  setIsOpen(true)
  if (!loggedAppearanceRef.current) {
    loggedAppearanceRef.current = true
    logEvent('tengu_skill_improvement_survey', {
      event_type: 'appeared',
      _PROTO_skill_name: suggestion.skillName ?? 'unknown',
    })
  }
}
```

#### 2. 用户响应处理流程
```
handleSelect 调用
  ↓
获取当前建议（从 lastSuggestionRef）
  ↓
判断响应类型:
  ├── applied → 应用改进
  │             ↓
  │           调用 applySkillImprovement(skillName, updates)
  │             ↓
  │           成功后添加系统消息
  │
  └── dismissed/dismissed_forever → 仅记录分析
  ↓
记录分析事件
  ↓
关闭调查问卷
  ↓
清理 AppState（设置 suggestion 为 null）
```

**代码实现**（行 53-98）：
```typescript
const handleSelect = useCallback(
  (selected: FeedbackSurveyResponse) => {
    const current = lastSuggestionRef.current
    if (!current) return

    const applied = selected !== 'dismissed'

    logEvent('tengu_skill_improvement_survey', {
      event_type: 'responded',
      response: applied ? 'applied' : 'dismissed',
      _PROTO_skill_name: current.skillName,
    })

    if (applied) {
      void applySkillImprovement(current.skillName, current.updates).then(() => {
        setMessages(prev => [
          ...prev,
          createSystemMessage(
            `Skill "${current.skillName}" updated with improvements.`,
            'suggestion',
          ),
        ])
      })
    }

    // Close and clear
    setIsOpen(false)
    loggedAppearanceRef.current = false
    setAppState(prev => {
      if (!prev.skillImprovement.suggestion) return prev
      return {
        ...prev,
        skillImprovement: { suggestion: null },
      }
    })
  },
  [setAppState, setMessages],
)
```

### 关键代码路径

#### Ref 使用说明

**lastSuggestionRef**（行 29, 33-35）：
```typescript
const lastSuggestionRef = useRef(suggestion)

// Track the suggestion for display even after clearing AppState
if (suggestion) {
  lastSuggestionRef.current = suggestion
}
```
用途：在 AppState 被清理后仍然保留建议数据，供后续处理使用。

**loggedAppearanceRef**（行 30, 40-50）：
```typescript
const loggedAppearanceRef = useRef(false)

if (!loggedAppearanceRef.current) {
  loggedAppearanceRef.current = true
  logEvent(...)
}
```
用途：确保分析事件只记录一次，避免重复计数。

#### 分析事件字段

```typescript
logEvent('tengu_skill_improvement_survey', {
  event_type: 'appeared' | 'responded',
  response?: 'applied' | 'dismissed',  // 仅 responded 时有
  _PROTO_skill_name: skillName,        // 特殊字段，路由到 BQ skill_name 列
})
```

注释说明 `_PROTO_skill_name` 会路由到特权列，不会出现在 `additional_metadata` 中。

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `../components/FeedbackSurvey/utils.js` | `FeedbackSurveyResponse` 类型 |
| `../services/analytics/index.js` | `logEvent` 分析日志 |
| `../state/AppState.js` | AppState 访问和更新 |
| `../types/message.js` | `Message` 类型 |
| `../utils/hooks/skillImprovement.js` | `SkillUpdate`, `applySkillImprovement` |
| `../utils/messages.js` | `createSystemMessage` |

### 外部交互

1. **AppState**: 
   - 读取 `skillImprovement.suggestion`
   - 更新 `skillImprovement.suggestion` 为 null

2. **分析系统**: 
   - 记录调查问卷展示事件
   - 记录用户响应事件

3. **技能改进系统**: 
   - 调用 `applySkillImprovement` 应用改进
   - 异步更新技能文件

4. **消息系统**: 
   - 应用成功后添加系统消息

---

## 风险、边界与改进建议

### 已知风险

1. **Ref 持久化**: `lastSuggestionRef` 在组件生命周期内持久化，如果建议频繁变化可能导致显示旧数据
2. **异步应用**: `applySkillImprovement` 是异步的，用户可能在应用完成前关闭界面
3. **分析重复**: 如果组件重新挂载，`loggedAppearanceRef` 会重置，可能导致重复记录

### 边界情况

1. **快速连续建议**: 如果新建议在旧建议处理完成前到达
2. **应用失败**: `applySkillImprovement` 失败时没有错误处理或用户通知
3. **空建议处理**: 对 `skillName` 为空的情况有检查，但 `updates` 为空数组时仍会处理

### 改进建议

1. **应用状态反馈**: 添加应用中的加载状态和应用成功/失败的反馈
2. **建议队列**: 支持多个建议排队显示
3. **持久化分析标记**: 将 `loggedAppearanceRef` 持久化到 AppState 避免重复记录
4. **错误处理**: 添加 `applySkillImprovement` 的错误处理和重试机制
5. **预览功能**: 在应用前显示具体的文件变更预览
6. **撤销支持**: 支持撤销已应用的改进

### 测试关注点

1. 建议到达时调查问卷是否正确打开
2. 分析事件是否正确记录（仅一次）
3. 应用改进后的系统消息添加
4. 关闭后的 AppState 清理
5. 快速连续建议的处理
6. 组件重新挂载后的行为
