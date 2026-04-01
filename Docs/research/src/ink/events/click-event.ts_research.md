# click-event.ts 深度研究文档

## 场景与职责

`ClickEvent` 是 Ink 终端 UI 框架中的鼠标点击事件类，专门用于处理终端内的鼠标点击交互。在终端应用中实现鼠标点击功能面临独特挑战：

1. **终端环境限制**：传统终端以键盘输入为主，鼠标支持需要启用特定的 DEC 私有模式（DECSET 1000/1002/1006）
2. **坐标系统转换**：终端使用 1-indexed 坐标，而 Ink 内部使用 0-indexed 坐标
3. **事件冒泡机制**：需要模拟 DOM 事件冒泡，让父组件可以监听子组件的点击事件

该事件类仅在 `<AlternateScreen>` 内有效，因为鼠标追踪必须显式启用。

## 功能点目的

### 1. 点击位置追踪
- `col` / `row`: 0-indexed 屏幕坐标，表示点击发生在终端的哪个单元格
- `localCol` / `localRow`: 相对于当前处理组件的局部坐标，由 `dispatchClick` 在每次处理前重新计算

### 2. 空白单元格检测
- `cellIsBlank`: 标识点击的单元格是否有可见内容。用于防止用户误点文本右侧空白区域时触发状态切换

### 3. 事件传播控制
继承自 `Event` 基类，支持 `stopImmediatePropagation()` 来阻止祖先组件的 `onClick` 处理器执行。

## 具体技术实现

### 数据结构
```typescript
class ClickEvent extends Event {
  readonly col: number          // 0-indexed 屏幕列
  readonly row: number          // 0-indexed 屏幕行
  localCol = 0                  // 相对于当前 Box 的列
  localRow = 0                  // 相对于当前 Box 的行
  readonly cellIsBlank: boolean // 单元格是否空白
}
```

### 关键流程

1. **事件创建**（`hit-test.ts:dispatchClick`）:
   ```typescript
   const event = new ClickEvent(col, row, cellIsBlank)
   ```

2. **局部坐标计算**（`hit-test.ts:70-84`）:
   ```typescript
   const rect = nodeCache.get(target)
   if (rect) {
     event.localCol = col - rect.x
     event.localRow = row - rect.y
   }
   ```

3. **冒泡处理**:
   - 从最深命中的节点开始向上遍历 `parentNode`
   - 每个有 `onClick` 处理器的节点都会触发
   - 调用 `stopImmediatePropagation()` 后停止冒泡

### 坐标系统

| 坐标类型 | 来源 | 用途 |
|---------|------|------|
| 屏幕坐标 (col/row) | 终端 SGR 鼠标序列 | 全局定位 |
| 局部坐标 (localCol/localRow) | 屏幕坐标 - Box 偏移 | 组件内部定位 |

终端发送的 SGR 鼠标序列格式：`CSI < button ; col ; row M/m`
- col/row 是 1-indexed
- Ink 在解析时转换为 0-indexed

## 关键代码路径与文件引用

### 定义位置
- `src/ink/events/click-event.ts` - 事件类定义

### 使用位置
1. `src/ink/hit-test.ts:70` - 创建并分发点击事件
2. `src/ink/components/Box.tsx:30` - Box 组件的 onClick 属性类型
3. `src/ink/components/Button.tsx` - 按钮点击处理

### 调用链
```
App.handleMouseEvent (App.tsx:515)
  → dispatchClick (hit-test.ts:49)
    → new ClickEvent (click-event.ts:32)
    → handler(event) (hit-test.ts:83)
```

## 依赖与外部交互

### 依赖
- `event.ts` - 基类 Event，提供 `stopImmediatePropagation` 支持

### 被依赖
- `hit-test.ts` - 导入 ClickEvent 用于事件分发
- `Box.tsx` - 导入类型用于 Props 定义
- `event-handlers.ts` - 定义 ClickEventHandler 类型

### 终端协议交互
- 依赖 DECSET 1006 (SGR 鼠标协议) 获取带坐标的鼠标事件
- 依赖 DECSET 1002 (按钮事件追踪) 或 1003 (任意鼠标事件) 启用鼠标报告

## 风险、边界与改进建议

### 风险点

1. **鼠标追踪依赖**:
   - 如果终端不支持或不启用鼠标追踪模式，点击事件永远不会触发
   - 某些终端模拟器（如 Windows CMD）可能不完全支持 SGR 鼠标协议

2. **坐标越界**:
   - 终端窗口调整大小时，缓存的坐标可能失效
   - 需要配合 `handleResize` 重置状态

3. **事件竞争**:
   - 快速点击可能导致事件堆积
   - 多点击检测（双击/三击）与单击事件可能冲突

### 边界情况

1. **空白单元格点击**:
   - `cellIsBlank` 用于过滤误点，但依赖屏幕缓冲区状态
   - 如果缓冲区未正确更新，可能误判

2. **Alt 键点击**:
   - macOS 上 Option+点击可能被终端拦截用于原生选择
   - xterm.js 会在此情况下设置 `cellIsBlank` 相关标记

3. **跨组件拖拽**:
   - 点击开始于一个组件，释放于另一个组件
   - 当前实现只在释放时触发点击，可能不符合用户预期

### 改进建议

1. **添加右键/中键支持**:
   ```typescript
   // 当前只处理左键 (button=0)
   // 可考虑扩展支持其他按钮
   readonly button: 0 | 1 | 2
   ```

2. **添加修饰键状态**:
   ```typescript
   readonly shift: boolean
   readonly ctrl: boolean
   readonly meta: boolean
   ```
   这可以让组件实现 Shift+点击等高级交互。

3. **双击事件原生支持**:
   当前双击检测在 `App.tsx` 中手动实现，可考虑封装到 ClickEvent。

4. **触摸板手势**:
   考虑将滚轮事件（wheelup/wheeldown）与点击事件统一处理。

### 测试建议

1. 测试不同终端模拟器（iTerm2、Ghostty、Windows Terminal）的兼容性
2. 测试快速点击、拖拽后释放等边界交互
3. 测试终端窗口调整大小后的坐标准确性
4. 测试 SSH 连接下的鼠标事件传递
