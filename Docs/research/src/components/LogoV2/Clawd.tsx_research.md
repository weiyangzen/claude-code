# Clawd.tsx 研究文档

## 场景与职责

`Clawd` 是 Claude Code 终端 UI 中的吉祥物组件，以 ASCII 艺术形式呈现 Claude 的品牌形象。该组件在以下场景中使用：

1. **Logo 展示** - 作为 CondensedLogo 和 LogoV2 的核心视觉元素
2. **品牌识别** - 提供独特的视觉标识，增强产品辨识度
3. **交互反馈** - AnimatedClawd 基于此组件添加点击动画

组件设计遵循以下原则：
- 跨平台兼容：针对 Apple Terminal 有特殊渲染路径
- 可配置姿态：支持多种表情和动作姿态
- 固定尺寸：9列 x 3行的标准尺寸

## 功能点目的

### 1. 多姿态 ASCII 艺术
支持 4 种不同的姿态：

| 姿态 | 描述 | 应用场景 |
|------|------|----------|
| `default` | 默认姿态，双臂下垂 | 正常状态 |
| `arms-up` | 双臂举起 | 跳跃/庆祝动画 |
| `look-left` | 双眼向左看 | 环顾动画 |
| `look-right` | 双眼向右看 | 环顾动画 |

### 2. Apple Terminal 适配
- **问题**: Apple Terminal 的背景填充渲染方式不同，导致标准姿态显示异常
- **解决方案**: 检测到 `env.terminal === "Apple_Terminal"` 时使用简化的仅眼睛姿态
- **限制**: Apple Terminal 不支持手臂姿态，回退到 default 的眼睛样式

### 3. 分段渲染优化
- 将 3 行 ASCII 艺术分割为多个 Text 组件
- 允许独立变化的部分（眼睛、手臂）与稳定部分（身体）分离
- 减少 React 重渲染的范围

### 4. React Compiler 优化
- 使用 `_c` 编译器运行时进行细粒度的记忆化
- 每个 Text 片段独立缓存，仅在实际变化时更新

## 具体技术实现

### 关键流程

```
接收 pose 属性
      ↓
检查终端类型
      ↓
[Apple Terminal] → 使用 APPLE_EYES 简化渲染
      ↓
[其他终端] → 使用 POSES 标准渲染
      ↓
分段渲染为 Text 组件
      ↓
应用主题颜色 (clawd_body, clawd_background)
```

### 数据结构

```typescript
// 姿态类型
export type ClawdPose = 
  | 'default' 
  | 'arms-up'    // 双臂举起（跳跃时使用）
  | 'look-left'  // 双眼向左看
  | 'look-right'; // 双眼向右看

// 分段定义
type Segments = {
  r1L: string;  // 第1行左侧（无背景）: 可选举起的臂 + 侧边
  r1E: string;  // 第1行眼睛（有背景）: 左眼、额头、右眼
  r1R: string;  // 第1行右侧（无背景）: 侧边 + 可选举起的臂
  r2L: string;  // 第2行左侧（无背景）: 臂 + 身体曲线
  r2R: string;  // 第2行右侧（无背景）: 身体曲线 + 臂
};

// 标准终端姿态定义
const POSES: Record<ClawdPose, Segments> = {
  default: {
    r1L: ' ▐',      r1E: '▛███▜',    r1R: '▌',
    r2L: '▝▜',      r2R: '▛▘'
  },
  'look-left': {
    r1L: ' ▐',      r1E: '▟███▟',    r1R: '▌',
    r2L: '▝▜',      r2R: '▛▘'
  },
  'look-right': {
    r1L: ' ▐',      r1E: '▙███▙',    r1R: '▌',
    r2L: '▝▜',      r2R: '▛▘'
  },
  'arms-up': {
    r1L: '▗▟',      r1E: '▛███▜',    r1R: '▙▖',
    r2L: ' ▜',      r2R: '▛ '
  }
};

// Apple Terminal 专用眼睛定义
const APPLE_EYES: Record<ClawdPose, string> = {
  default:    ' ▗   ▖ ',
  'look-left': ' ▘   ▘ ',
  'look-right': ' ▝   ▝ ',
  'arms-up':  ' ▗   ▖ '  // 回退到 default
};
```

### 核心算法

**分段渲染逻辑**:
```typescript
// 标准终端渲染
const p = POSES[pose];
return (
  <Box flexDirection="column">
    {/* 第1行：左 + 眼睛(带背景) + 右 */}
    <Text>
      <Text color="clawd_body">{p.r1L}</Text>
      <Text color="clawd_body" backgroundColor="clawd_background">{p.r1E}</Text>
      <Text color="clawd_body">{p.r1R}</Text>
    </Text>
    {/* 第2行：左 + 身体(带背景) + 右 */}
    <Text>
      <Text color="clawd_body">{p.r2L}</Text>
      <Text color="clawd_body" backgroundColor="clawd_background">█████</Text>
      <Text color="clawd_body">{p.r2R}</Text>
    </Text>
    {/* 第3行：脚部 */}
    <Text color="clawd_body">{"  "}▘▘ ▝▝{"  "}</Text>
  </Box>
);
```

**Apple Terminal 简化渲染**:
```typescript
function AppleTerminalClawd({ pose }: { pose: ClawdPose }) {
  const eyes = APPLE_EYES[pose];
  return (
    <Box flexDirection="column" alignItems="center">
      <Text>
        <Text color="clawd_body">▗</Text>
        <Text color="clawd_background" backgroundColor="clawd_body">{eyes}</Text>
        <Text color="clawd_body">▖</Text>
      </Text>
      <Text backgroundColor="clawd_body">{" ".repeat(7)}</Text>
      <Text color="clawd_body">▘▘ ▝▝</Text>
    </Box>
  );
}
```

### 关键代码路径

1. **终端检测** (line 87):
   ```typescript
   if (env.terminal === "Apple_Terminal") {
     return <AppleTerminalClawd pose={pose} />;
   }
   ```

2. **分段缓存** (line 99-122):
   ```typescript
   let t3;
   if ($[4] !== p.r1L) {
     t3 = <Text color="clawd_body">{p.r1L}</Text>;
     $[4] = p.r1L;
     $[5] = t3;
   } else {
     t3 = $[5];
   }
   ```
   - React Compiler 生成的记忆化代码
   - 每个片段独立比较和缓存

3. **颜色应用**:
   - `clawd_body`: 主体颜色（通常是黄色/金色）
   - `clawd_background`: 背景填充色

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| Box, Text | `../../ink.js` | UI 组件 |
| env | `../../utils/env.js` | 终端类型检测 |

### 依赖详情

**`env.terminal`** (`src/utils/env.ts`):
- 通过 `detectTerminal()` 函数检测终端类型
- 检查 `TERM_PROGRAM` 环境变量
- Apple Terminal 返回 `"Apple_Terminal"`

**主题颜色**:
- `clawd_body`: 在主题系统中定义的主体颜色
- `clawd_background`: 在主题系统中定义的背景颜色

### 调用关系

```
CondensedLogo.tsx / LogoV2.tsx
    ↓
Clawd.tsx (基础渲染)
    ↓
AnimatedClawd.tsx (动画包装)
    ↓
Clawd.tsx (姿态渲染)
```

## 风险、边界与改进建议

### 已知风险

1. **终端检测可靠性**: 依赖 `TERM_PROGRAM` 环境变量，某些 SSH 或嵌套 shell 场景可能检测失败
   - 缓解: 检测失败时回退到标准渲染，Apple Terminal 用户可能看到轻微渲染问题

2. **字符显示差异**: Unicode 方块字符在不同字体中的显示效果差异很大
   - 缓解: 使用常见的方块字符（▛▜▝▞▟▙▖▗▘）

3. **主题颜色依赖**: 硬编码依赖 `clawd_body` 和 `clawd_background` 主题键
   - 风险: 如果主题未定义这些键，显示可能异常

### 边界情况

| 场景 | 行为 |
|------|------|
| 未知 pose 值 | 通过 `POSES[pose]` 访问返回 undefined，可能导致渲染错误 |
| 终端宽度不足 | 组件宽度固定为 9 列，超出部分可能被截断或换行 |
| 非彩色终端 | 颜色代码被忽略，显示为纯 ASCII 字符 |
| 字体不支持 Unicode | 显示为 tofu 或替代字符 |

### 改进建议

1. **姿态回退**: 为未知的 pose 值添加安全回退
   ```typescript
   const p = POSES[pose] ?? POSES['default'];
   ```

2. **更多姿态**: 可扩展更多表情和动作:
   - `blink`: 眨眼（交替显示睁眼/闭眼）
   - `happy`: 开心的表情
   - `surprised`: 惊讶的表情
   - `sleeping`: 睡眠状态（用于空闲时）

3. **动画支持**: 当前姿态是瞬时的，可考虑添加过渡动画:
   ```typescript
   // 可能的实现
   function AnimatedClawdTransition({ from, to, duration }) {
     // 使用 useAnimationFrame 插值过渡
   }
   ```

4. **终端适配扩展**: 考虑检测更多终端的特殊渲染需求:
   - Windows Terminal
   - VS Code 集成终端
   - JetBrains 终端

5. **尺寸可配置**: 当前尺寸固定，可考虑添加 `scale` 属性支持放大/缩小:
   ```typescript
   type Props = {
     pose?: ClawdPose;
     scale?: 1 | 2;  // 1 = 正常, 2 = 双倍大小
   };
   ```

6. **测试覆盖**: 建议添加:
   - 各姿态的渲染快照测试
   - Apple Terminal 检测的单元测试
   - 颜色应用的视觉测试

### 字符参考

组件使用的 Unicode 方块字符：

| 字符 | Unicode | 描述 |
|------|---------|------|
| ▛ | U+259B | 象限左上右 |
| ▜ | U+259C | 象限右上左 |
| ▝ | U+259D | 象限右上 |
| ▞ | U+259E | 象限对角 |
| ▟ | U+259F | 象限右下左 |
| ▙ | U+2599 | 象限左下右 |
| ▖ | U+2596 | 象限左下 |
| ▗ | U+2597 | 象限右下 |
| ▘ | U+2598 | 象限左上 |

### 相关文件引用

- 主实现: `src/components/LogoV2/Clawd.tsx`
- 动画包装: `src/components/LogoV2/AnimatedClawd.tsx`
- 调用方: `src/components/LogoV2/CondensedLogo.tsx`
- 环境检测: `src/utils/env.ts`
