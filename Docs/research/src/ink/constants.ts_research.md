# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 Ink 终端 UI 框架的共享常量定义文件。它包含整个渲染管线使用的全局常量，确保各模块对关键数值有一致的定义。

### 核心职责

1. **帧率控制**：定义渲染节流和动画的帧间隔
2. **全局共享**：提供跨模块访问的单一事实来源
3. **性能调优**：集中管理影响性能的定时参数

## 功能点目的

### 1. 帧间隔常量

`FRAME_INTERVAL_MS = 16` 定义了约 60fps 的渲染帧率：
- 16ms ≈ 60 帧/秒
- 用于渲染节流（throttling）
- 用于动画帧调度

### 2. 使用场景

- **渲染节流**：限制 React 渲染触发实际终端更新的频率
- **动画同步**：`use-animation-frame.ts` 等 hooks 使用此间隔
- **滚动动画**：平滑滚动的帧率控制

## 具体技术实现

### 常量定义

```typescript
// Shared frame interval for render throttling and animations (~60fps)
export const FRAME_INTERVAL_MS = 16
```

### 数值选择依据

- **16ms**：1000ms / 60fps ≈ 16.67ms，取整为 16ms
- 平衡流畅度和性能
- 与显示器刷新率（通常 60Hz）匹配

## 关键代码路径与文件引用

### 入口与导出
- **文件**：`src/ink/constants.ts`
- **导出常量**：`FRAME_INTERVAL_MS`

### 使用位置

通过 grep 搜索，该常量被以下模块使用：

| 模块 | 用途 |
|------|------|
| `hooks/use-animation-frame.ts` | 动画帧调度 |
| `ink.tsx` | 渲染节流控制 |
| `renderer.ts` | 帧率限制 |

### 相关文件

- `src/ink/hooks/use-animation-frame.ts` - 动画 hook
- `src/ink/ink.tsx` - 主渲染循环
- `src/ink/renderer.ts` - 渲染器实现

## 依赖与外部交互

### 模块关系

```
constants.ts
    ↓
被多个模块导入
    ↓
渲染节流、动画控制
```

### 设计考量

1. **单一职责**：文件仅包含常量定义，无逻辑代码
2. **最小化依赖**：不导入其他模块，避免循环依赖
3. **可预测性**：常量在模块加载时确定，运行时不变

## 风险、边界与改进建议

### 已知风险

1. **硬编码值**：16ms 是固定的，不适应高刷新率显示器（120Hz/144Hz）
2. **全局影响**：修改此值会影响整个应用的渲染行为
3. **单位混淆**：毫秒 vs 秒，需要文档明确

### 边界情况

1. **高负载场景**：16ms 间隔可能无法保证，实际帧率可能下降
2. **低功耗模式**：某些系统可能降低刷新率，导致不必要的渲染
3. **测试环境**：测试可能需要不同的帧率设置

### 改进建议

1. **动态帧率**：
   - 检测显示器刷新率，自适应调整
   - 提供配置选项覆盖默认值
   
   ```typescript
   // 示例：动态帧率
   export const FRAME_INTERVAL_MS = 
     detectRefreshRate() > 60 ? 8 : 16
   ```

2. **环境感知**：
   - 测试环境使用更短的间隔加速测试
   - CI 环境禁用节流
   
   ```typescript
   export const FRAME_INTERVAL_MS = 
     process.env.NODE_ENV === 'test' ? 0 : 16
   ```

3. **扩展常量**：
   - 添加更多共享常量（最大滚动速度、缓冲区大小等）
   - 按功能分组（RENDER_、ANIMATION_、SCROLL_ 前缀）
   
   ```typescript
   export const RENDER_FRAME_INTERVAL_MS = 16
   export const SCROLL_MAX_DELTA_PER_FRAME = 10
   export const ANIMATION_DEFAULT_DURATION_MS = 200
   ```

4. **文档完善**：
   - 添加 JSDoc 注释说明数值选择依据
   - 记录与其他系统的交互（如 React 的调度器）

5. **类型安全**：
   - 使用 branded types 区分毫秒和其他时间单位
   
   ```typescript
   type Milliseconds = number & { __brand: 'ms' }
   export const FRAME_INTERVAL_MS: Milliseconds = 16 as Milliseconds
   ```
