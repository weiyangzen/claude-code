# Ratchet.tsx 深度研究文档

## 场景与职责

Ratchet 是 Claude Code 设计系统中一个智能高度管理组件，用于解决终端 UI 中动态内容高度变化导致的布局抖动问题。它通过"棘轮"机制（只增不减）锁定内容的最大高度，确保内容区域在增长后不会收缩，从而提供稳定的视觉体验。

**核心职责：**
1. **高度锁定**：记录并锁定内容的最大高度，防止收缩
2. **视口感知**：检测内容是否在终端视口内，支持离屏锁定
3. **平滑增长**：允许内容高度自然增长，但阻止回缩
4. **布局稳定**：消除因内容高度变化导致的界面跳动

**典型使用场景：**
- 消息响应区域 (`src/components/MessageResponse.tsx`)
- 动态内容展示，如逐步生成的响应、流式输出等
- 任何需要防止高度收缩导致布局抖动的场景

---

## 功能点目的

### 1. 棘轮高度机制
- **目的**：防止内容高度收缩导致的布局抖动
- **实现**：记录历史最大高度，设置 `minHeight` 为该值
- **类比**：像棘轮一样，只能向前（增长），不能后退（收缩）

### 2. 视口感知锁定
- **目的**：只在需要时启用高度锁定
- **实现**：通过 `useTerminalViewport` 检测内容可见性
- **模式**：
  - `always`：始终锁定高度
  - `offscreen`：仅在内容离开视口时锁定

### 3. 终端尺寸适配
- **目的**：确保锁定高度不超过终端可用行数
- **实现**：`Math.min(height, rows)`
- **价值**：避免锁定高度超出终端可视范围

### 4. 精确测量
- **目的**：获取内容的实际渲染高度
- **实现**：使用 Ink 的 `measureElement` API
- **触发**：在 `useLayoutEffect` 中测量

---

## 具体技术实现

### 关键流程

```
初始化 → 视口检测 → 高度测量 → 最大高度更新 → 渲染锁定
```

**核心逻辑详解：**

1. **状态初始化**
   ```typescript
   const [viewportRef, { isVisible }] = useTerminalViewport();
   const { rows } = useTerminalSize();
   const innerRef = useRef<DOMElement | null>(null);
   const maxHeight = useRef(0);  // 历史最大高度
   const [minHeight, setMinHeight] = useState(0);  // 当前最小高度
   ```

2. **锁定模式判断**
   ```typescript
   const engaged = lock === 'always' || !isVisible;
   ```
   - `always`：始终启用高度锁定
   - `offscreen`：仅在内容离开视口时锁定

3. **高度测量与更新**
   ```typescript
   useLayoutEffect(() => {
     if (!innerRef.current) return;
     
     const { height } = measureElement(innerRef.current);
     
     if (height > maxHeight.current) {
       // 更新最大高度（受终端行数限制）
       maxHeight.current = Math.min(height, rows);
       setMinHeight(maxHeight.current);
     }
   });
   ```

4. **渲染结构**
   ```
   Box (minHeight={engaged ? minHeight : undefined}, ref={outerRef})
   └── Box (flexDirection="column", ref={innerRef})
       └── children
   ```

### 数据结构

**Props 接口：**
```typescript
type Props = {
  children: React.ReactNode;           // 内容子元素
  lock?: 'always' | 'offscreen';       // 锁定模式（默认 'always'）
}
```

**核心状态：**
```typescript
const maxHeight = useRef(0);           // 历史最大高度（mutable ref）
const [minHeight, setMinHeight] = useState(0);  // 当前最小高度（state）
```

### 布局结构

```
Box (minHeight={conditional}, ref={outerRef})
└── Box (flexDirection="column", ref={innerRef})
    └── children
```

- **outerRef**：外层容器，应用 minHeight 约束
- **innerRef**：内层容器，用于测量实际内容高度
- **viewportRef**：与 useTerminalViewport 关联，检测可见性

---

## 关键代码路径与文件引用

### 当前文件
- **路径**：`src/components/design-system/Ratchet.tsx`
- **大小**：约 7.2KB（含 source map）

### 核心代码段

**主渲染逻辑（编译后）：**
```javascript
export function Ratchet(t0) {
  const $ = _c(10);
  const { children, lock: t1 } = t0;
  const lock = t1 === undefined ? "always" : t1;
  
  // 视口检测
  const [viewportRef, t2] = useTerminalViewport();
  const { isVisible } = t2;
  
  // 终端尺寸
  const { rows } = useTerminalSize();
  
  // Refs 和 State
  const innerRef = useRef(null);
  const maxHeight = useRef(0);
  const [minHeight, setMinHeight] = useState(0);
  
  // outerRef 回调
  let t3;
  if ($[0] !== viewportRef) {
    t3 = el => { viewportRef(el); };
    $[0] = viewportRef;
    $[1] = t3;
  } else {
    t3 = $[1];
  }
  const outerRef = t3;
  
  // 锁定判断
  const engaged = lock === "always" || !isVisible;
  
  // 高度测量 effect
  let t4;
  if ($[2] !== rows) {
    t4 = () => {
      if (!innerRef.current) return;
      const { height } = measureElement(innerRef.current);
      if (height > maxHeight.current) {
        maxHeight.current = Math.min(height, rows);
        setMinHeight(maxHeight.current);
      }
    };
    $[2] = rows;
    $[3] = t4;
  } else {
    t4 = $[3];
  }
  useLayoutEffect(t4);
  
  // 渲染
  const t5 = engaged ? minHeight : undefined;
  // ... 缓存优化后的渲染逻辑
}
```

### 调用方文件

| 文件路径 | 使用场景 |
|---------|---------|
| `src/components/MessageResponse.tsx` | 消息响应区域高度稳定 |

---

## 依赖与外部交互

### 直接依赖

```typescript
import React, { useCallback, useLayoutEffect, useRef, useState } from 'react';
import { useTerminalSize } from '../../hooks/useTerminalSize.js';
import { useTerminalViewport } from '../../ink/hooks/use-terminal-viewport.js';
import { Box, type DOMElement, measureElement } from '../../ink.js';
```

### 依赖详情

**1. useTerminalSize**
- **路径**：`src/hooks/useTerminalSize.ts`
- **提供**：终端尺寸（rows, columns）
- **实现**：
  ```typescript
  export function useTerminalSize(): TerminalSize {
    const size = useContext(TerminalSizeContext);
    if (!size) {
      throw new Error('useTerminalSize must be used within an Ink App component');
    }
    return size;
  }
  ```

**2. useTerminalViewport**
- **路径**：`src/ink/hooks/use-terminal-viewport.ts`
- **提供**：视口检测能力
- **返回**：`[ref, { isVisible }]`
- **实现细节**：
  - 使用 `useLayoutEffect` 检测元素位置
  - 遍历 DOM 祖先链计算绝对位置
  - 考虑 scroll containers 的 scrollTop

**3. measureElement**
- **来源**：`src/ink.ts`
- **功能**：测量 DOMElement 的渲染尺寸
- **返回**：`{ width, height }`

**4. DOMElement**
- **来源**：`src/ink.ts`
- **类型**：Ink 内部 DOM 元素类型

### 依赖关系图

```
Ratchet
├── useTerminalSize → TerminalSizeContext
├── useTerminalViewport → TerminalSizeContext + DOM 遍历
├── measureElement → Ink 内部 API
└── Box → Ink 组件
```

---

## 风险、边界与改进建议

### 潜在风险

1. **测量时机问题**
   - `useLayoutEffect` 在每次渲染后运行，可能导致性能问题
   - 如果内容包含异步加载的图片/数据，初始测量可能不准确

2. **终端尺寸变化**
   - `rows` 变化会触发重新测量，但 `maxHeight` 不会自动缩小
   - 如果终端缩小，锁定高度可能超出可视区域

3. **内存泄漏**
   - `maxHeight` 使用 ref 存储，组件卸载时不会自动清理
   - 虽然数值类型不会导致严重泄漏，但长期运行的应用需注意

4. **嵌套使用风险**
   - 多个 Ratchet 嵌套可能导致高度计算复杂化
   - 父级 Ratchet 的 minHeight 可能影响子级测量

### 边界情况

| 场景 | 行为 | 建议 |
|------|------|------|
| children 为空 | minHeight 保持 0 | 符合预期 |
| rows = 0 | maxHeight 被约束为 0 | 罕见，但处理正确 |
| 内容高度超过 rows | 锁定在 rows | 防止超出终端 |
| 终端快速 resize | 可能触发多次测量 | 考虑添加防抖 |
| lock = 'offscreen' 且始终在视口 | minHeight 始终为 0 | 等同于无 Ratchet |

### 改进建议

1. **添加高度重置机制**
   ```typescript
   type Props = {
     // ...
     resetKey?: string | number;  // 变化时重置 maxHeight
   }
   
   useEffect(() => {
     maxHeight.current = 0;
     setMinHeight(0);
   }, [resetKey]);
   ```

2. **支持最大高度限制**
   ```typescript
   type Props = {
     // ...
     maxLockHeight?: number;  // 限制最大锁定高度
   }
   
   maxHeight.current = Math.min(
     height, 
     rows, 
     maxLockHeight ?? Infinity
   );
   ```

3. **添加过渡动画**
   ```typescript
   // 在高度增长时添加平滑过渡
   const [isGrowing, setIsGrowing] = useState(false);
   
   useLayoutEffect(() => {
     // ...
     if (height > maxHeight.current) {
       setIsGrowing(true);
       // ...
       setTimeout(() => setIsGrowing(false), 300);
     }
   });
   ```

4. **性能优化**
   ```typescript
   // 使用 requestAnimationFrame 批量测量
   useLayoutEffect(() => {
     let rafId: number;
     const measure = () => {
       // 测量逻辑
       rafId = requestAnimationFrame(measure);
     };
     rafId = requestAnimationFrame(measure);
     return () => cancelAnimationFrame(rafId);
   });
   ```

5. **添加调试模式**
   ```typescript
   type Props = {
     // ...
     debug?: boolean;  // 显示高度信息
   }
   
   {debug && (
     <Text dimColor>height: {minHeight}, max: {maxHeight.current}</Text>
   )}
   ```

### 测试建议

1. **单元测试**
   ```typescript
   test('height only increases', () => {
     const { rerender } = render(<Ratchet><Content height={10} /></Ratchet>);
     expect(getMinHeight()).toBe(10);
     
     rerender(<Ratchet><Content height={5} /></Ratchet>);
     expect(getMinHeight()).toBe(10); // 不收缩
     
     rerender(<Ratchet><Content height={15} /></Ratchet>);
     expect(getMinHeight()).toBe(15); // 可增长
   });
   
   test('respects terminal rows', () => {
     mockTerminalSize({ rows: 20 });
     render(<Ratchet><Content height={30} /></Ratchet>);
     expect(getMinHeight()).toBe(20);
   });
   ```

2. **集成测试**
   - 在真实消息流中测试高度稳定性
   - 测试终端 resize 时的行为
   - 测试快速内容变化时的性能

3. **视觉回归测试**
   - 验证高度锁定后内容不跳动
   - 验证不同锁定模式的行为差异
