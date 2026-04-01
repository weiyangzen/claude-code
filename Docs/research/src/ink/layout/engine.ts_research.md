# src/ink/layout/engine.ts 研究文档

## 场景与职责

`engine.ts` 是 Ink 布局系统的入口文件，作为**工厂模块**负责创建布局节点。它是上层 DOM 系统与底层 Yoga 布局引擎之间的抽象层，提供统一的 `createLayoutNode()` 函数来实例化布局节点。

**核心定位**：
- 作为布局引擎的**统一入口**，隐藏底层 Yoga 实现细节
- 提供**工厂模式**创建布局节点，便于未来替换或扩展布局引擎
- 保持与 `node.ts` 中定义的 `LayoutNode` 接口兼容

## 功能点目的

### 1. 布局节点工厂函数

```typescript
export function createLayoutNode(): LayoutNode {
  return createYogaLayoutNode()
}
```

**目的**：
- 解耦上层代码与具体布局引擎实现（Yoga）
- 如果需要更换布局引擎，只需修改此文件即可
- 保持 API 稳定性，上层代码通过 `engine.ts` 导入而非直接依赖 `yoga.ts`

## 具体技术实现

### 关键流程

```
调用方 (dom.ts)
    ↓
createLayoutNode() [engine.ts]
    ↓
createYogaLayoutNode() [yoga.ts]
    ↓
new YogaLayoutNode(Yoga.Node.create())
```

### 依赖关系

```
engine.ts
├── imports: LayoutNode (from './node.js')
├── imports: createYogaLayoutNode (from './yoga.js')
└── exports: createLayoutNode
```

## 关键代码路径与文件引用

### 被调用方

| 文件 | 调用方式 | 用途 |
|------|----------|------|
| `src/ink/dom.ts` | `import { createLayoutNode } from './layout/engine.js'` | 创建 DOM 元素时初始化 yogaNode |

**具体调用位置** (`src/ink/dom.ts` 第 121 行):
```typescript
const node: DOMElement = {
  // ...
  yogaNode: needsYogaNode ? createLayoutNode() : undefined,
  // ...
}
```

### 调用链

```
dom.ts:createNode()
  → engine.ts:createLayoutNode()
    → yoga.ts:createYogaLayoutNode()
      → YogaLayoutNode.constructor()
        → Yoga.Node.create()
```

## 依赖与外部交互

### 直接依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `./node.js` | `LayoutNode` (type) | 返回类型定义 |
| `./yoga.js` | `createYogaLayoutNode` | 实际创建 Yoga 布局节点 |

### 与 Yoga 的关系

`engine.ts` 是 Yoga 布局引擎的**薄封装层**：
- 不直接操作 Yoga API
- 通过 `yoga.ts` 中的适配器间接使用
- 符合依赖倒置原则（DIP）

## 风险、边界与改进建议

### 风险点

1. **过度简化风险**：当前实现过于简单，如果未来需要支持多种布局引擎，需要重构
2. **循环依赖风险**：虽然目前没有，但如果 `engine.ts` 开始依赖更多模块，可能引入循环依赖

### 边界情况

- 该文件**无状态**，纯工厂函数
- 不处理错误情况，依赖底层 `yoga.ts` 的错误处理

### 改进建议

1. **配置支持**：扩展 `createLayoutNode` 接受配置参数，支持不同布局模式
   ```typescript
   export function createLayoutNode(config?: LayoutConfig): LayoutNode
   ```

2. **引擎切换**：如果需要支持多种布局引擎，可以实现运行时切换
   ```typescript
   let engine: 'yoga' | 'custom' = 'yoga'
   export function setLayoutEngine(e: typeof engine) { engine = e }
   ```

3. **单例优化**：考虑是否需要 Yoga 实例级别的配置（当前 Yoga 是全局单例）

4. **类型安全**：当前文件非常小（6 行），可以考虑与 `yoga.ts` 合并，或者保持独立以维持清晰的架构边界
