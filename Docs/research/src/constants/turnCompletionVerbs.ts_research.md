# turnCompletionVerbs.ts 深度研究文档

## 场景与职责

`src/constants/turnCompletionVerbs.ts` 是 Claude Code CLI 的回合完成动词常量模块，定义了一组用于显示 Agent/模型完成工作时的趣味动词列表。这是一个轻量级的 UI/UX 增强模块，为终端输出增添个性化和趣味性。

**主要使用场景：**
1. **回合完成消息显示**：当 Agent 完成一轮工作（turn）时，显示带有持续时间的完成消息
2. **加载状态指示**：在长时间运行的任务中提供视觉反馈
3. **队友活动指示**：在 Agent Swarm 场景中显示队友的活动状态

**设计目标：**
- 提供多样化的动词避免重复和单调
- 使用过去时态以自然配合 "for [duration]" 句式
- 保持专业性的同时增添个性

## 功能点目的

### 1. 趣味化完成消息

传统的完成消息可能是简单的 "Completed" 或 "Done"，而该模块提供了更多样化的表达：

- "Baked for 5s"
- "Cogitated for 12s"
- "Sautéed for 3s"

这种表达方式：
- 减轻长时间等待的焦虑感
- 为技术工具增添人性化元素
- 通过多样化避免视觉疲劳

### 2. 过去时态设计

所有动词都使用过去时态，因为：
- 与 "for [duration]" 句式自然配合
- 表示动作已经完成
- 符合英语语法习惯

## 具体技术实现

### 常量定义

```typescript
// Past tense verbs for turn completion messages
// These verbs work naturally with "for [duration]" (e.g., "Worked for 5s")
export const TURN_COMPLETION_VERBS = [
  'Baked',
  'Brewed',
  'Churned',
  'Cogitated',
  'Cooked',
  'Crunched',
  'Sautéed',
  'Worked',
]
```

### 动词选择 rationale

| 动词 | 含义/联想 | 使用场景 |
|------|----------|----------|
| Baked | 烘焙，精心制作 | 代码生成、构建任务 |
| Brewed | 酿造，酝酿 | 思考、规划任务 |
| Churned | 搅拌，快速处理 | 数据处理、批量操作 |
| Cogitated | 深思熟虑 | 复杂问题分析 |
| Cooked | 烹饪，准备 | 文件操作、准备工作 |
| Crunched |  crunch，处理数据 | 计算密集型任务 |
| Sautéed | 煎炒，快速处理 | 快速编辑、小修改 |
| Worked | 工作（保底选项） | 通用场景 |

### 使用模式

```typescript
// 典型使用方式
import sample from 'lodash-es/sample.js'
import { TURN_COMPLETION_VERBS } from '../constants/turnCompletionVerbs.js'

const verb = sample(TURN_COMPLETION_VERBS)
const message = `${verb} for ${duration}s`
// 输出: "Cogitated for 5s" 或 "Baked for 12s" 等
```

## 关键代码路径与文件引用

### 调用方分析

| 调用文件 | 调用内容 | 用途 |
|----------|----------|------|
| `src/components/messages/SystemTextMessage.tsx` | `TURN_COMPLETION_VERBS` | 系统消息中的回合完成显示 |
| `src/components/Spinner/TeammateSpinnerLine.tsx` | `TURN_COMPLETION_VERBS` | 队友活动指示器 |
| `src/utils/swarm/spawnInProcess.ts` | `TURN_COMPLETION_VERBS` | 进程内队友生成时的活动描述 |

### 核心消费代码示例

```typescript
// src/components/messages/SystemTextMessage.tsx
import sample from 'lodash-es/sample.js'
import { TURN_COMPLETION_VERBS } from '../../constants/turnCompletionVerbs.js'

// 在回合完成消息组件中
function TurnDurationMessage({ message, addMargin }) {
  const verb = sample(TURN_COMPLETION_VERBS)
  const duration = formatDuration(message.durationMs)
  
  return (
    <Text dimColor>
      {verb} for {duration}
    </Text>
  )
}
```

```typescript
// src/utils/swarm/spawnInProcess.ts
import sample from 'lodash-es/sample.js'
import { TURN_COMPLETION_VERBS } from '../../constants/turnCompletionVerbs.js'

// 创建活动描述
const activityVerb = sample(TURN_COMPLETION_VERBS)
const activityDescription = `${activityVerb}...`
```

```typescript
// src/components/Spinner/TeammateSpinnerLine.tsx
import { TURN_COMPLETION_VERBS } from '../../constants/turnCompletionVerbs.js'

// 在队友旋转指示器中使用
const verbs = TURN_COMPLETION_VERBS
```

## 依赖与外部交互

### 外部依赖

该模块是纯常量模块，无直接依赖。

### 消费依赖

| 模块 | 依赖方式 |
|------|----------|
| `lodash-es/sample.js` | 调用方使用来随机选择动词 |

### 国际化考虑

当前模块仅提供英文动词，未来如需国际化：
- 可将动词列表移至本地化文件
- 根据用户语言设置加载对应的动词列表

## 风险、边界与改进建议

### 潜在风险

1. **文化适应性**
   - 某些动词（如 Sautéed）可能不是所有用户都熟悉
   - 烹饪相关的隐喻可能不适用于所有文化背景

2. **专业性平衡**
   - 过于随意的动词可能降低专业感
   - 需要在趣味性和专业性之间取得平衡

3. **可访问性**
   - 对于屏幕阅读器用户，多样化的动词可能增加认知负担
   - 需要确保辅助技术能正确朗读

### 边界情况

1. **空数组**：如果数组为空，`sample()` 返回 undefined
2. **持续时间极短**："Baked for 0s" 可能显得奇怪
3. **持续时间极长**：长时间任务可能需要不同的表达方式

### 改进建议

1. **分类动词**
   ```typescript
   export const TURN_COMPLETION_VERBS_BY_CATEGORY = {
     thinking: ['Cogitated', 'Pondered', 'Deliberated'],
     creating: ['Baked', 'Cooked', 'Crafted'],
     processing: ['Churned', 'Crunched', 'Processed'],
     quick: ['Sautéed', 'Whipped', 'Zapped'],
     fallback: ['Worked'],
   }
   
   // 根据任务类型选择动词类别
   export function getVerbForTaskType(taskType: TaskType): string {
     // 返回对应类别的随机动词
   }
   ```

2. **动态添加动词**
   ```typescript
   // 允许用户自定义动词
   export function addCustomVerbs(verbs: string[]): void {
     TURN_COMPLETION_VERBS.push(...verbs)
   }
   ```

3. **智能选择**
   ```typescript
   // 根据持续时间智能选择动词
   export function getVerbForDuration(durationMs: number): string {
     if (durationMs < 1000) {
       return sample(QUICK_VERBS)
     } else if (durationMs < 10000) {
       return sample(MEDIUM_VERBS)
     } else {
       return sample(LONG_VERBS)
     }
   }
   ```

4. **本地化支持**
   ```typescript
   // 简单的本地化框架
   const VERBS_BY_LOCALE: Record<string, string[]> = {
     en: ['Baked', 'Brewed', /* ... */],
     zh: ['烘焙', '酝酿', /* ... */],
     // ...
   }
   ```

5. **用户偏好**
   ```typescript
   // 允许用户选择风格
   export type VerbStyle = 'fun' | 'professional' | 'minimal'
   
   export function getVerbsForStyle(style: VerbStyle): string[] {
     switch (style) {
       case 'fun': return TURN_COMPLETION_VERBS
       case 'professional': return ['Completed', 'Processed', 'Finished']
       case 'minimal': return ['Done']
     }
   }
   ```

6. **动画效果配合**
   - 为不同的动词配合不同的终端动画
   - 例如 "Baked" 可以配合加热效果的进度条

7. **统计和反馈**
   - 跟踪哪些动词最常被显示
   - 收集用户对趣味化消息的反馈
