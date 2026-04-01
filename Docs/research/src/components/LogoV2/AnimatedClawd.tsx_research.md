# AnimatedClawd.tsx 研究文档

## 场景与职责

`AnimatedClawd` 是 Claude Code 终端 UI 中的交互式吉祥物组件，是 `Clawd` 组件的动画增强版本。该组件在以下场景中使用：

1. **全屏模式下的 Logo 展示** - 作为 CondensedLogo 的一部分，在 `isFullscreenEnvEnabled()` 返回 true 时显示
2. **交互反馈** - 用户点击时触发动画，提供愉悦的交互体验
3. **品牌情感连接** - 通过生动的动画增强用户对产品的情感连接

组件设计遵循以下原则：
- 保持与基础 `Clawd` 相同的视觉尺寸（3行高度）
- 点击触发动画仅在鼠标跟踪启用时生效（全屏/备用屏幕模式）
- 尊重减少动画偏好设置

## 功能点目的

### 1. 点击触发动画系统
- **跳跃挥手动画 (JUMP_WAVE)**: 蹲下后弹起，双臂举起，重复两次
- **环顾四周动画 (LOOK_AROUND)**: 眼睛先看右，再看左，最后回到中间
- **随机选择**: 每次点击随机选择一种动画播放

### 2. 固定高度容器
- **目的**: 防止动画过程中布局偏移
- **实现**: 使用固定高度 3 的容器，通过 `marginTop` 偏移实现"蹲下"效果
- **视觉技巧**: 蹲下时脚部被容器裁剪，产生"缩到画面下方"的视觉效果

### 3. 减少动画支持
- 读取 `prefersReducedMotion` 设置
- 启用时跳过动画，保持默认姿态

### 4. React Compiler 优化
- 使用 `_c` 编译器运行时进行记忆化
- 减少不必要的重渲染

## 具体技术实现

### 关键流程

```
点击 → 检查 reducedMotion → [true] → 忽略点击
            ↓
      [false] && frameIndex === -1 → 随机选择动画序列
            ↓
      设置 frameIndex = 0 → 启动定时器
            ↓
      每 60ms 推进帧索引 → 渲染对应姿态
            ↓
      序列结束 → 重置 frameIndex = -1 (返回 IDLE 状态)
```

### 数据结构

```typescript
// 帧定义
type Frame = {
  pose: ClawdPose;    // 'default' | 'arms-up' | 'look-left' | 'look-right'
  offset: number;     // marginTop 偏移量 (0 = 正常, 1 = 蹲下)
};

// 动画常量
const FRAME_MS = 60;           // 每帧持续时间
const CLAWD_HEIGHT = 3;        // 容器固定高度
const IDLE: Frame = { pose: 'default', offset: 0 };

// 动画序列 (帧数组)
const JUMP_WAVE: readonly Frame[] = [
  // 第一次蹲下-弹起
  ...hold('default', 1, 2),   // 蹲下 2 帧
  ...hold('arms-up', 0, 3),   // 弹起举臂 3 帧
  // 第二次蹲下-弹起
  ...hold('default', 1, 2),   // 蹲下 2 帧
  ...hold('arms-up', 0, 3),   // 弹起举臂 3 帧
  ...hold('default', 0, 1),   // 恢复 1 帧
];

const LOOK_AROUND: readonly Frame[] = [
  ...hold('look-right', 0, 5),  // 看右 5 帧
  ...hold('look-left', 0, 5),   // 看左 5 帧
  ...hold('default', 0, 1),     // 恢复 1 帧
];
```

### 核心算法

**hold 函数** - 生成保持同一姿态的多帧:
```typescript
function hold(pose: ClawdPose, offset: number, frames: number): Frame[] {
  return Array.from({ length: frames }, () => ({ pose, offset }));
}
```

**动画状态机**:
```typescript
const [frameIndex, setFrameIndex] = useState(-1);  // -1 = IDLE

// 当前帧计算
const current = frameIndex >= 0 && frameIndex < seq.length 
  ? seq[frameIndex] 
  : IDLE;
```

### 关键代码路径

1. **点击处理** (line 102-105):
   ```typescript
   const onClick = () => {
     if (reducedMotion || frameIndex !== -1) return;
     sequenceRef.current = CLICK_ANIMATIONS[Math.floor(Math.random() * CLICK_ANIMATIONS.length)];
     setFrameIndex(0);
   };
   ```
   - 防止动画期间重复触发
   - 随机选择动画序列

2. **帧推进** (line 107-115):
   ```typescript
   useEffect(() => {
     if (frameIndex === -1) return;
     if (frameIndex >= sequenceRef.current.length) {
       setFrameIndex(-1);
       return;
     }
     const timer = setTimeout(setFrameIndex, FRAME_MS, incrementFrame);
     return () => clearTimeout(timer);
   }, [frameIndex]);
   ```
   - 使用 setTimeout 链式推进
   - 自动清理防止内存泄漏

3. **布局偏移防止** (line 73-74):
   ```typescript
   <Box height={CLAWD_HEIGHT} flexDirection="column" onClick={onClick}>
     <Box marginTop={bounceOffset} flexShrink={0}>
   ```
   - 固定高度容器
   - 通过 marginTop 实现偏移而非改变内容高度

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| Box | `../../ink.js` | 布局容器 |
| getInitialSettings | `../../utils/settings/settings.js` | 读取减少动画设置 |
| Clawd, ClawdPose | `./Clawd.js` | 基础吉祥物组件和姿态类型 |

### 依赖详情

**`Clawd` 组件** (`src/components/LogoV2/Clawd.tsx`):
- 提供基础的 ASCII 艺术渲染
- 支持 4 种姿态: 'default', 'arms-up', 'look-left', 'look-right'
- 针对 Apple Terminal 有特殊渲染路径

**`getInitialSettings`** (`src/utils/settings/settings.ts`):
- 提供 `prefersReducedMotion` 布尔值
- 在组件挂载时一次性读取

### 调用关系

```
CondensedLogo.tsx
    ↓ (条件渲染)
AnimatedClawd.tsx
    ↓ (导入)
Clawd.tsx
    ↓ (导入)
ink.js (Box, Text)
```

## 风险、边界与改进建议

### 已知风险

1. **快速点击丢失**: 动画期间（约 300-600ms）的点击会被忽略
   - 缓解: 这是设计意图，防止动画队列堆积
   - 改进: 可考虑添加点击反馈（如音效或视觉提示）

2. **定时器精度**: 60ms 的帧间隔依赖 JavaScript 事件循环
   - 缓解: 动画设计允许一定的时间偏差
   - 风险: 高负载时可能出现卡顿

3. **内存泄漏风险**: `sequenceRef` 和 `startTimeRef` 使用
   - 缓解: 组件卸载时 React 自动清理
   - 注意: 确保 `useEffect` 清理函数正确执行

### 边界情况

| 场景 | 行为 |
|------|------|
| prefersReducedMotion = true | 点击无响应，始终显示 default 姿态 |
| 非全屏模式 | onClick 不会触发（Ink 点击事件仅在备用屏幕可用） |
| 动画中组件卸载 | setFrameIndex 在已卸载组件上调用（React 18 自动处理） |
| 连续快速点击 | 第二次及后续点击被忽略，直到当前动画完成 |

### 改进建议

1. **动画队列**: 当前实现忽略动画期间的点击，可考虑添加简单的队列机制

2. **更多动画**: 当前只有 2 种动画，可扩展更多姿态组合:
   - 眨眼动画
   - 点头动画
   - 庆祝动画（如任务完成时自动触发）

3. **自动触发**: 可考虑在特定事件时自动播放动画:
   - 任务完成
   - 错误发生
   - 长时间空闲后的唤醒

4. **性能优化**: 考虑使用 `requestAnimationFrame` 替代 `setTimeout`:
   ```typescript
   // 当前实现
   const timer = setTimeout(setFrameIndex, FRAME_MS, incrementFrame);
   
   // 可能的优化
   const startTime = performance.now();
   const tick = (now: number) => {
     if (now - startTime >= FRAME_MS) {
       setFrameIndex(i => i + 1);
     } else {
       requestAnimationFrame(tick);
     }
   };
   requestAnimationFrame(tick);
   ```

5. **测试覆盖**: 建议添加:
   - 点击触发动画的集成测试
   - 减少动画偏好下的行为测试
   - 动画序列完成的断言

### 相关文件引用

- 主实现: `src/components/LogoV2/AnimatedClawd.tsx`
- 基础组件: `src/components/LogoV2/Clawd.tsx`
- 调用方: `src/components/LogoV2/CondensedLogo.tsx`
- 设置系统: `src/utils/settings/settings.ts`
