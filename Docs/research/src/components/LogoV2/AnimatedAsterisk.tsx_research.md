# AnimatedAsterisk.tsx 研究文档

## 场景与职责

`AnimatedAsterisk` 是 Claude Code 终端 UI 中的一个动画装饰组件，用于在 Logo 区域显示一个带有彩虹色扫过动画的星号字符。该组件主要应用于：

1. **LogoV2 欢迎界面** - 作为品牌标识的装饰元素
2. **启动/加载场景** - 提供视觉反馈，表明系统正在活跃状态
3. **无障碍支持** - 尊重用户的减少动画偏好设置

组件的设计目标是在不干扰用户操作的前提下，提供愉悦的视觉体验。

## 功能点目的

### 1. 彩虹色扫过动画
- **目的**: 创建动态的视觉效果，吸引用户注意力
- **实现**: 通过 HSL 色相循环实现彩虹色效果
- **参数**: 
  - 单次扫过持续时间: 1500ms
  - 扫过次数: 2 次
  - 总动画时长: 3000ms

### 2. 减少动画支持
- **目的**: 尊重用户的无障碍偏好
- **实现**: 读取 `prefersReducedMotion` 设置，如果启用则直接显示静态灰色星号
- **技术细节**: 使用 `useState` 在挂载时一次性读取设置，避免订阅设置变化导致的重渲染

### 3. 视口暂停机制
- **目的**: 当组件离开可视区域时自动暂停动画，节省资源并防止闪烁
- **实现**: 利用 `useAnimationFrame` 的 viewport-pause 功能
- **触发条件**: 用户提交消息后，组件进入 scrollback 区域时自动停止动画

## 具体技术实现

### 关键流程

```
挂载 → 读取 prefersReducedMotion → [true] → 显示静态灰色星号
                    ↓
              [false] → 启动动画帧 → 计算色相 → 渲染彩色星号
                              ↓
                        3秒后 → 显示静态灰色星号
```

### 数据结构

```typescript
// 动画常量
const SWEEP_DURATION_MS = 1500;  // 单次扫过时长
const SWEEP_COUNT = 2;           // 扫过次数
const TOTAL_ANIMATION_MS = SWEEP_DURATION_MS * SWEEP_COUNT;

// 静态颜色
const SETTLED_GREY = toRGBColor({ r: 153, g: 153, b: 153 });
```

### 核心算法

**色相计算**:
```typescript
const elapsed = time - startTimeRef.current;
const hue = (elapsed / SWEEP_DURATION_MS * 360) % 360;
```

**颜色转换**:
- 使用 `hueToRgb()` 将 HSL 色相转换为 RGB
- 使用 `toRGBColor()` 将 RGB 对象格式化为 CSS 颜色字符串

### 关键代码路径

1. **动画启动** (line 30):
   ```typescript
   const [ref, time] = useAnimationFrame(done ? null : 50);
   ```
   - 50ms 更新间隔
   - `done` 为 true 时传入 null 暂停动画

2. **时间偏移捕获** (line 41-44):
   ```typescript
   if (startTimeRef.current === null) {
     startTimeRef.current = time;
   }
   const elapsed = time - startTimeRef.current;
   ```
   - 捕获开始时间偏移，确保动画从色相 0 开始
   - 避免共享时钟导致的不同步问题

3. **动画完成处理** (line 31-35):
   ```typescript
   useEffect(() => {
     if (done) return;
     const t = setTimeout(setDone, TOTAL_ANIMATION_MS, true);
     return () => clearTimeout(t);
   }, [done]);
   ```
   - 使用 setTimeout 标记动画完成
   - 清理函数防止内存泄漏

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| TEARDROP_ASTERISK | `../../constants/figures.js` | 默认星号字符常量 |
| Box, Text, useAnimationFrame | `../../ink.js` | UI 组件和动画钩子 |
| getInitialSettings | `../../utils/settings/settings.js` | 读取用户设置 |
| hueToRgb, toRGBColor | `../Spinner/utils.js` | 颜色转换工具 |

### 依赖详情

**`useAnimationFrame` hook** (`src/ink/hooks/use-animation-frame.ts`):
- 提供共享的动画时钟
- 自动处理视口可见性检测
- 返回 `[ref, time]` 元组，ref 用于绑定 DOM 元素

**`TEARDROP_ASTERISK`** (`src/constants/figures.ts`):
- Unicode 字符: `✻`
- 用于品牌标识的一致性

**`getInitialSettings`** (`src/utils/settings/settings.ts`):
- 提供 `prefersReducedMotion` 设置读取
- 返回会话启动时的设置快照

**`hueToRgb`** (`src/components/Spinner/utils.ts`):
- HSL 到 RGB 的颜色空间转换
- 使用饱和度 0.7、亮度 0.6 的语音模式波形参数

## 风险、边界与改进建议

### 已知风险

1. **时钟漂移**: 如果 `useAnimationFrame` 的时钟在动画期间重置，可能导致色相跳跃
   - 缓解: 使用 `startTimeRef` 捕获相对时间偏移

2. **颜色可访问性**: 彩虹色在某些终端配色方案下可能对比度不足
   - 缓解: 动画结束后使用固定的灰色

3. **性能**: 50ms 的更新间隔在低端设备上可能仍有压力
   - 缓解: 视口暂停机制确保不可见时不更新

### 边界情况

| 场景 | 行为 |
|------|------|
| prefersReducedMotion = true | 立即显示灰色星号，无动画 |
| 组件快速挂载/卸载 | 通过 ref 和 cleanup 函数安全处理 |
| 终端宽度变化 | 不影响，组件尺寸固定 |
| 动画期间设置变化 | 不响应，使用挂载时的快照值 |

### 改进建议

1. **配置化动画参数**: 当前动画时长和颜色固定，可考虑通过设置暴露配置选项

2. **主题适配**: 考虑根据当前主题动态调整 `SETTLED_GREY` 的颜色值

3. **动画曲线**: 当前使用线性色相变化，可考虑添加缓动函数使动画更自然

4. **测试覆盖**: 建议添加以下测试用例:
   - 减少动画偏好下的静态渲染
   - 动画完成后的状态转换
   - 视口暂停/恢复行为

### 相关文件引用

- 主实现: `src/components/LogoV2/AnimatedAsterisk.tsx`
- 动画钩子: `src/ink/hooks/use-animation-frame.ts`
- 颜色工具: `src/components/Spinner/utils.ts`
- 字符常量: `src/constants/figures.ts`
- 设置系统: `src/utils/settings/settings.ts`
