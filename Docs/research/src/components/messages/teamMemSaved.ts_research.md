# teamMemSaved.ts 研究文档

## 场景与职责

`teamMemSaved.ts` 是 Claude Code CLI 中负责**团队内存保存 UI 文本生成**的轻量级工具模块。它为系统内存保存消息生成团队内存部分的显示文本。

### 核心职责

1. **生成团队内存文本**：为内存保存 UI 创建团队内存计数的显示文本
2. **返回计数信息**：同时返回计数，使调用方可以计算私有内存计数
3. **纯函数设计**：无副作用，易于测试和推理

### 使用场景

- 当 Agent Swarms 功能启用且 `feature('TEAMMEM')` 为 true 时加载
- 在 `SystemTextMessage.tsx` 中用于显示内存保存消息的团队内存部分
- 与 `teamMemCollapsed.tsx` 一起构成团队内存 UI 显示功能

---

## 功能点目的

### 1. 团队内存保存文本生成 (`teamMemSavedPart`)

**目的**：为内存保存消息生成团队内存部分的显示文本。

**输入**：
```typescript
message: SystemMemorySavedMessage
// 包含 teamCount 属性（可选）
```

**输出**：
```typescript
{
  segment: string;  // 显示文本，如 "3 team memories"
  count: number;    // 团队内存计数
} | null            // 如果计数为 0，返回 null
```

**文本格式**：
```
"1 team memory"      // 单数
"3 team memories"    // 复数
```

### 2. 计数返回

**目的**：使调用方能够计算私有内存计数，而无需直接访问 `teamCount` 属性。

**使用模式**：
```typescript
const teamPart = teamMemSavedPart(message);
if (teamPart) {
  const privateCount = totalCount - teamPart.count;
  // 显示："Saved 3 team memories and 2 private memories"
}
```

---

## 具体技术实现

### 关键数据结构

```typescript
// 函数签名
export function teamMemSavedPart(
  message: SystemMemorySavedMessage,
): { segment: string; count: number } | null

// SystemMemorySavedMessage 类型（来自 types/message.js）
type SystemMemorySavedMessage = {
  type: 'system';
  subtype: 'memory_saved';
  writtenPaths: string[];    // 写入的文件路径
  teamCount?: number;        // 团队内存计数（可选）
  timestamp: string;
  uuid: string;
  isMeta: boolean;
};
```

### 实现逻辑

```typescript
export function teamMemSavedPart(
  message: SystemMemorySavedMessage,
): { segment: string; count: number } | null {
  const count = message.teamCount ?? 0;  // 使用空值合并，默认 0
  
  if (count === 0) {
    return null;  // 无团队内存，不显示
  }
  
  return {
    segment: `${count} team ${count === 1 ? 'memory' : 'memories'}`,
    count,
  };
}
```

### 复数处理

```typescript
// 单复数规则
const noun = count === 1 ? 'memory' : 'memories';

// 示例
// count = 1  → "1 team memory"
// count = 5  → "5 team memories"
// count = 0  → null (不显示)
```

---

## 关键代码路径与文件引用

### 直接依赖

| 文件 | 导出内容 | 用途 |
|------|----------|------|
| `src/types/message.ts` | `SystemMemorySavedMessage` | 消息类型定义 |

### 调用方

**主要调用方**：`SystemTextMessage.tsx`

```typescript
// 在 SystemTextMessage.tsx 中的使用（假设）
import { teamMemSavedPart } from './teamMemSaved.js';

function SystemMemorySavedMessage({ message }) {
  const teamPart = teamMemSavedPart(message);
  
  if (!teamPart) {
    return <Text>Saved {message.writtenPaths.length} memories</Text>;
  }
  
  const privateCount = message.writtenPaths.length - teamPart.count;
  
  return (
    <Text>
      Saved {teamPart.segment}
      {privateCount > 0 && ` and ${privateCount} private memories`}
    </Text>
  );
}
```

### 与 teamMemCollapsed.tsx 的关系

| 特性 | teamMemSaved.ts | teamMemCollapsed.tsx |
|------|-----------------|---------------------|
| **用途** | 内存保存消息 | 折叠读/搜索组 |
| **返回** | 文本段 + 计数 | React 节点 |
| **复杂度** | 简单纯函数 | React 组件 |
| **时态** | 无（静态文本） | 动态（现在时/过去时） |
| **操作类型** | 仅写入 | 读、搜索、写 |

---

## 依赖与外部交互

### 模块依赖图

```
teamMemSaved.ts
└── types/message.ts (SystemMemorySavedMessage)

SystemTextMessage.tsx
└── teamMemSaved.ts
    └── teamMemSavedPart()

createMemorySavedMessage() (in utils/messages.ts)
└── 创建 SystemMemorySavedMessage
    └── teamCount 属性
```

### 数据流

```
utils/messages.ts
  └── createMemorySavedMessage(writtenPaths, teamCount?)
      └── SystemMemorySavedMessage

SystemTextMessage.tsx
  └── teamMemSavedPart(message)
      └── { segment: "3 team memories", count: 3 }
          └── 渲染: "Saved 3 team memories and 2 private memories"
```

### 功能开关集成

```typescript
// 此模块仅在 feature('TEAMMEM') 为 true 时加载
// 确保外部构建中不包含团队内存相关代码

// 在 SystemTextMessage.tsx 中的条件加载
if (feature('TEAMMEM')) {
  const { teamMemSavedPart } = require('./teamMemSaved.js');
  const teamPart = teamMemSavedPart(message);
  // ...
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **功能开关依赖**
   - 模块假设调用方已检查 `feature('TEAMMEM')`
   - 如果直接导入而不检查功能开关，可能导致外部构建包含不必要的代码
   - **缓解**：文档明确说明使用条件

2. **类型定义耦合**
   - 依赖 `SystemMemorySavedMessage` 类型定义
   - 如果类型变更（如 `teamCount` 重命名），此模块需要同步更新

3. **复数规则简单**
   - 当前仅支持英语复数规则（加 -ies）
   - 未来国际化可能需要更复杂的复数处理

### 边界情况

| 场景 | 当前行为 |
|------|----------|
| `teamCount` 为 `undefined` | 视为 0，返回 `null` |
| `teamCount` 为 0 | 返回 `null`，不显示 |
| `teamCount` 为 1 | 返回 `"1 team memory"` |
| `teamCount` 为负数 | 返回文本（假设输入已验证） |
| `teamCount` 为很大的数 | 正常返回文本 |

### 改进建议

1. **输入验证**
   ```typescript
   export function teamMemSavedPart(
     message: SystemMemorySavedMessage,
   ): { segment: string; count: number } | null {
     const count = message.teamCount ?? 0;
     
     // 添加验证
     if (count < 0) {
       console.warn('teamCount should not be negative:', count);
       return null;
     }
     
     if (count === 0) return null;
     
     return {
       segment: `${count} team ${count === 1 ? 'memory' : 'memories'}`,
       count,
     };
   }
   ```

2. **国际化支持**
   ```typescript
   // 建议：支持多语言
   import { pluralize } from '../utils/i18n.js';
   
   export function teamMemSavedPart(
     message: SystemMemorySavedMessage,
     locale: string = 'en',
   ) {
     const count = message.teamCount ?? 0;
     if (count === 0) return null;
     
     return {
       segment: `${count} ${pluralize(locale, 'team_memory', count)}`,
       count,
     };
   }
   ```

3. **更丰富的信息**
   ```typescript
   // 建议：返回更多信息供调用方使用
   export function teamMemSavedPart(message: SystemMemorySavedMessage) {
     const count = message.teamCount ?? 0;
     if (count === 0) return null;
     
     return {
       segment: `${count} team ${count === 1 ? 'memory' : 'memories'}`,
       count,
       isSingular: count === 1,
       // 可以添加更多元数据
     };
   }
   ```

4. **测试覆盖**
   ```typescript
   // 建议：添加单元测试
   describe('teamMemSavedPart', () => {
     it('returns null when teamCount is undefined', () => {
       const message = { teamCount: undefined };
       expect(teamMemSavedPart(message)).toBeNull();
     });
     
     it('returns null when teamCount is 0', () => {
       const message = { teamCount: 0 };
       expect(teamMemSavedPart(message)).toBeNull();
     });
     
     it('returns singular for count of 1', () => {
       const message = { teamCount: 1 };
       expect(teamMemSavedPart(message)?.segment).toBe('1 team memory');
     });
     
     it('returns plural for count > 1', () => {
       const message = { teamCount: 5 };
       expect(teamMemSavedPart(message)?.segment).toBe('5 team memories');
     });
   });
   ```

5. **与 teamMemCollapsed.tsx 的代码共享**
   ```typescript
   // 建议：共享复数处理逻辑
   // utils/teamMemoryText.ts
   export function formatTeamMemoryCount(count: number): string {
     return `${count} team ${count === 1 ? 'memory' : 'memories'}`;
   }
   
   // teamMemSaved.ts
   import { formatTeamMemoryCount } from '../utils/teamMemoryText.js';
   
   // teamMemCollapsed.tsx
   import { formatTeamMemoryCount } from '../utils/teamMemoryText.js';
   ```

### 架构考虑

当前实现是**极简的工具函数**：
- 优点：简单、可预测、易于测试
- 缺点：功能单一，可能需要与其他模块协调变更

**设计模式**：
- 使用**纯函数**避免副作用
- 返回**null**表示"无内容"，符合 React 的渲染习惯
- 同时返回**文本和计数**，支持调用方的灵活使用

**与 teamMemCollapsed.tsx 的对比**：
- 两者都是功能开关驱动的模块
- `teamMemSaved.ts` 更简单（纯函数）
- `teamMemCollapsed.tsx` 更复杂（React 组件，动态时态）
- 两者都使用相同的复数规则
