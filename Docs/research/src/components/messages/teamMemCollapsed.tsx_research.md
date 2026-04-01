# teamMemCollapsed.tsx 研究文档

## 场景与职责

`teamMemCollapsed.tsx` 是 Claude Code CLI 中负责**团队内存操作计数显示**的专用组件模块。它在折叠的读/搜索 UI 中显示团队内存（Team Memory）的读取、搜索和写入操作计数。

### 核心职责

1. **团队内存操作检测**：检查消息是否包含团队内存操作
2. **计数显示**：在折叠的读/搜索组 UI 中显示团队内存操作统计
3. **动态时态**：根据组是否处于活动状态，动态调整动词时态（现在时 vs 过去时）

### 使用场景

- 当 Agent Swarms 功能启用且 `feature('TEAMMEM')` 为 true 时加载
- 在 `CollapsedReadSearchContent.tsx` 中用于显示团队内存操作计数
- 仅用于折叠的读/搜索消息组，不用于单个消息

---

## 功能点目的

### 1. 团队内存操作检测 (`checkHasTeamMemOps`)

**目的**：快速检查消息是否包含任何团队内存操作。

**检测逻辑**：
```typescript
export function checkHasTeamMemOps(message: CollapsedReadSearchGroup): boolean {
  return (
    (message.teamMemorySearchCount ?? 0) > 0 ||
    (message.teamMemoryReadCount ?? 0) > 0 ||
    (message.teamMemoryWriteCount ?? 0) > 0
  );
}
```

**设计决策**：
- 使用普通函数而非 React 组件
- 避免 React Compiler 将属性访问提升为记忆化依赖
- 确保在 `feature('TEAMMEM')` 为 false 时能被死代码消除（DCE）

### 2. 团队内存计数显示 (`TeamMemCountParts`)

**目的**：渲染团队内存操作的计数文本。

**显示格式**：
```
Recalled 3 team memories, searching team memories, wrote 1 team memory
Recalling 2 team memories, wrote 1 team memory
Searched team memories
```

**时态规则**：
- **活动组**（`isActiveGroup = true`）：使用现在进行时（Recalling, Searching, Writing）
- **非活动组**（`isActiveGroup = false`）：使用过去时（Recalled, Searched, Wrote）

**标点规则**：
- 第一项：大写开头
- 后续项：小写，前面加逗号
- 使用 `hasPrecedingParts` 参数处理与其他计数部分的衔接

### 3. 复数处理

**目的**：正确处理 "memory/memories" 的单复数形式。

```typescript
const noun = count === 1 ? 'memory' : 'memories';
```

---

## 具体技术实现

### 关键数据结构

```typescript
// 组件 Props
type TeamMemCountPartsProps = {
  message: CollapsedReadSearchGroup;     // 折叠的读/搜索组消息
  isActiveGroup: boolean | undefined;    // 是否处于活动状态
  hasPrecedingParts: boolean;            // 前面是否有其他计数部分
};

// CollapsedReadSearchGroup 类型（来自 types/message.js）
type CollapsedReadSearchGroup = {
  // ... 其他属性
  teamMemorySearchCount?: number;   // 团队内存搜索计数
  teamMemoryReadCount?: number;     // 团队内存读取计数
  teamMemoryWriteCount?: number;    // 团队内存写入计数
};
```

### 动词时态逻辑

```typescript
// 读取操作的动词
const verb = isActiveGroup
  ? count === 0 ? 'Recalling' : 'recalling'    // 现在进行时
  : count === 0 ? 'Recalled' : 'recalled';     // 过去时

// 搜索操作的动词
const verb = isActiveGroup
  ? count === 0 ? 'Searching' : 'searching'
  : count === 0 ? 'Searched' : 'searched';

// 写入操作的动词
const verb = isActiveGroup
  ? count === 0 ? 'Writing' : 'writing'
  : count === 0 ? 'Wrote' : 'wrote';
```

### React Compiler 记忆化

组件使用 23 个缓存槽位进行细粒度记忆化：

```typescript
export function TeamMemCountParts(t0) {
  const $ = _c(23);  // 23 个缓存槽位
  
  // 输入参数缓存
  const { message, isActiveGroup, hasPrecedingParts } = t0;
  
  // 派生值缓存
  const tmReadCount = message.teamMemoryReadCount ?? 0;
  const tmSearchCount = message.teamMemorySearchCount ?? 0;
  const tmWriteCount = message.teamMemoryWriteCount ?? 0;
  
  // 条件返回缓存
  if (tmReadCount === 0 && tmSearchCount === 0 && tmWriteCount === 0) {
    return null;
  }
  
  // 复杂渲染逻辑缓存...
}
```

### 渲染流程

```
TeamMemCountParts
│
├─→ 检查是否有任何团队内存操作
│   └─→ 无 → 返回 null
│
├─→ 初始化 nodes 数组和计数器
│
├─→ 读取操作 (tmReadCount > 0)
│   ├─→ 添加逗号（如果需要）
│   ├─→ 计算动词时态
│   ├─→ 渲染 "{verb} {count} team {memories}"
│   └─→ 递增计数器
│
├─→ 搜索操作 (tmSearchCount > 0)
│   ├─→ 添加逗号（如果需要）
│   ├─→ 计算动词时态
│   ├─→ 渲染 "{verb} team memories"
│   └─→ 递增计数器
│
├─→ 写入操作 (tmWriteCount > 0)
│   ├─→ 添加逗号（如果需要）
│   ├─→ 计算动词时态
│   ├─→ 渲染 "{verb} {count} team {memories}"
│   └─→ 递增计数器
│
└─→ 返回 nodes 数组
```

---

## 关键代码路径与文件引用

### 直接依赖

| 文件 | 导出内容 | 用途 |
|------|----------|------|
| `src/ink.ts` | `Text` | Ink 文本组件 |
| `src/types/message.ts` | `CollapsedReadSearchGroup` | 消息类型定义 |

### 调用方

**主要调用方**：`CollapsedReadSearchContent.tsx`

```typescript
// 在 CollapsedReadSearchContent.tsx 中的使用（假设）
import { TeamMemCountParts, checkHasTeamMemOps } from './teamMemCollapsed.js';

function CollapsedReadSearchContent({ message, isActiveGroup }) {
  const hasTeamMemOps = checkHasTeamMemOps(message);
  
  return (
    <>
      {/* 其他计数部分 */}
      {hasTeamMemOps && (
        <TeamMemCountParts
          message={message}
          isActiveGroup={isActiveGroup}
          hasPrecedingParts={hasOtherParts}
        />
      )}
    </>
  );
}
```

### 与 collapseReadSearch.ts 的关系

**`src/utils/collapseReadSearch.ts`**：
- 创建 `CollapsedReadSearchGroup` 对象
- 计算 `teamMemorySearchCount`、`teamMemoryReadCount`、`teamMemoryWriteCount`
- 聚合多个文件读/搜索操作到折叠组

---

## 依赖与外部交互

### 模块依赖图

```
teamMemCollapsed.tsx
├── ink.ts (Text 组件)
├── types/message.ts (CollapsedReadSearchGroup)
└── React

collapseReadSearch.ts
└── 创建 CollapsedReadSearchGroup
    ├── teamMemorySearchCount
    ├── teamMemoryReadCount
    └── teamMemoryWriteCount

CollapsedReadSearchContent.tsx
├── teamMemCollapsed.tsx
│   ├── checkHasTeamMemOps()
│   └── TeamMemCountParts
└── 其他计数组件
```

### 功能开关集成

```typescript
// 此模块仅在 feature('TEAMMEM') 为 true 时加载
// 确保外部构建中不包含团队内存相关代码

// 在 CollapsedReadSearchContent.tsx 中的条件加载
if (feature('TEAMMEM')) {
  const { TeamMemCountParts } = require('./teamMemCollapsed.js');
  // ...
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **React Compiler 优化问题**
   - 注释明确说明使用普通函数而非组件的原因
   - 如果改为 React 组件，React Compiler 可能错误地将属性访问提升为记忆化依赖
   - **风险**：可能导致不必要的重新渲染或错误的缓存行为

2. **功能开关依赖**
   - 模块假设调用方已检查 `feature('TEAMMEM')`
   - 如果直接导入而不检查功能开关，可能导致外部构建包含不必要的代码
   - **缓解**：文档明确说明使用条件

3. **类型定义耦合**
   - 依赖 `CollapsedReadSearchGroup` 类型定义
   - 如果类型变更，此模块需要同步更新

### 边界情况

| 场景 | 当前行为 |
|------|----------|
| 所有计数为 0 | 返回 `null`，不渲染任何内容 |
| `isActiveGroup` 为 `undefined` | 视为非活动组（使用过去时） |
| `hasPrecedingParts` 为 `true` | 在开头添加逗号 |
| 计数为 1 | 使用单数 "memory" |
| 计数大于 1 | 使用复数 "memories" |
| 只有搜索操作 | 不显示数字（搜索操作无计数） |

### 改进建议

1. **国际化支持**
   ```typescript
   // 建议：支持多语言
   const t = useTranslation();
   const verb = isActiveGroup ? t('recalling') : t('recalled');
   const noun = count === 1 ? t('memory') : t('memories');
   ```

2. **可配置显示**
   ```typescript
   // 建议：允许用户选择显示哪些操作类型
   type DisplayConfig = {
     showRead: boolean;
     showSearch: boolean;
     showWrite: boolean;
   };
   ```

3. **更丰富的信息**
   ```typescript
   // 建议：显示具体的内存键或摘要
   type TeamMemoryOperation = {
     type: 'read' | 'search' | 'write';
     key?: string;
     summary?: string;
   };
   ```

4. **测试覆盖**
   ```typescript
   // 建议：添加单元测试
   describe('TeamMemCountParts', () => {
     it('returns null when all counts are 0', () => {});
     it('uses present tense for active groups', () => {});
     it('uses past tense for inactive groups', () => {});
     it('handles singular/plural correctly', () => {});
     it('adds commas correctly based on hasPrecedingParts', () => {});
   });
   ```

5. **代码生成优化**
   ```typescript
   // 建议：简化重复的模式
   function renderOperation(
     count: number,
     isActive: boolean,
     verbs: { active: string; inactive: string },
     noun: string,
   ) {
     const verb = isActive ? verbs.active : verbs.inactive;
     return `${verb} ${count} team ${count === 1 ? noun : noun + 'ies'}`;
   }
   ```

### 架构考虑

当前实现是**功能开关驱动的条件模块**：
- 优点：外部构建不包含团队内存代码
- 缺点：增加了代码复杂性，需要条件导入

**替代方案**：
- 始终包含模块，但在运行时根据功能开关决定是否渲染
- 使用更通用的"操作计数"组件，通过配置驱动
