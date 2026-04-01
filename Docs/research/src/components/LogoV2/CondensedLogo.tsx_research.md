# CondensedLogo.tsx 研究文档

## 场景与职责

`CondensedLogo` 是 Claude Code 终端 UI 中的紧凑型 Logo 组件，用于在会话期间持续显示关键状态信息。该组件在以下场景中使用：

1. **主输入区域顶部** - 作为 PromptInput 上方的状态栏
2. **紧凑信息展示** - 在终端宽度有限时提供关键信息
3. **持续状态反馈** - 显示当前模型、计费方式、工作目录、Agent 状态等

组件设计遵循以下原则：
- 信息密度高：在有限空间内展示多个状态维度
- 响应式布局：根据终端宽度自动调整显示格式
- 交互感知：全屏模式下显示可点击的 AnimatedClawd

## 功能点目的

### 1. 多维度状态展示
组件展示以下信息：

| 信息项 | 来源 | 用途 |
|--------|------|------|
| Claude Code 版本 | `process.env.DEMO_VERSION` / `MACRO.VERSION` | 版本识别 |
| 当前模型 | `useMainLoopModel()` | 显示正在使用的 AI 模型 |
| 计费方式 | `isClaudeAISubscriber()` | Pro/Max/Team/Enterprise/API Usage |
| 工作目录 | `getCwd()` | 显示当前工作位置 |
| Agent 名称 | `useAppState(s => s.agent)` | 显示当前激活的 Agent |
| 努力级别 | `useAppState(s => s.effortValue)` | 显示当前努力设置 |

### 2. 响应式文本截断
- **版本号**: 根据可用宽度动态截断
- **模型名称 + 计费**: 空间不足时拆分为两行显示
- **工作目录**: 智能路径截断（保留首尾，中间省略）
- **Agent 路径**: 考虑 Agent 名称后的可用空间

### 3. 营销组件集成
- **GuestPassesUpsell**: 显示访客通行证推广（条件触发）
- **OverageCreditUpsell**: 显示超额额度推广（条件触发）
- **计数追踪**: 记录推广展示次数用于频率控制

### 4. 全屏模式适配
- **标准模式**: 显示静态 `Clawd`
- **全屏模式**: 显示可点击的 `AnimatedClawd`（支持跳跃/环顾动画）

## 具体技术实现

### 关键流程

```
获取终端尺寸
      ↓
获取应用状态 (agent, effort, model)
      ↓
获取 Logo 显示数据 (version, cwd, billing, agentName)
      ↓
计算文本宽度 = columns - 15
      ↓
截断/格式化各文本元素
      ↓
useEffect 更新推广展示计数
      ↓
渲染布局：Clawd + 信息列 + 可选推广
```

### 数据结构

```typescript
// Logo 显示数据结构
interface LogoDisplayData {
  version: string;           // 版本号
  cwd: string;               // 当前工作目录（或带服务器 URL）
  billingType: string;       // 计费类型描述
  agentName: string | undefined;  // Agent 名称
}

// 模型和计费格式化结果
interface FormattedModelBilling {
  shouldSplit: boolean;      // 是否需要拆分为两行
  truncatedModel: string;    // 截断后的模型名称
  truncatedBilling: string;  // 截断后的计费类型
}
```

### 核心算法

**文本宽度计算**:
```typescript
const textWidth = Math.max(columns - 15, 20);
```
- 预留 15 列给 Clawd 艺术和间距
- 最小宽度 20 列确保基本可读性

**模型和计费格式化**:
```typescript
function formatModelAndBilling(
  modelName: string,
  billingType: string,
  availableWidth: number
): FormattedModelBilling {
  const separator = ' · ';
  const combinedWidth = stringWidth(modelName) + separator.length + stringWidth(billingType);
  const shouldSplit = combinedWidth > availableWidth;
  
  if (shouldSplit) {
    return {
      shouldSplit: true,
      truncatedModel: truncate(modelName, availableWidth),
      truncatedBilling: truncate(billingType, availableWidth),
    };
  }
  
  return {
    shouldSplit: false,
    truncatedModel: truncate(modelName, availableWidth - stringWidth(billingType) - separator.length),
    truncatedBilling: billingType,
  };
}
```

**工作目录可用宽度计算**:
```typescript
const cwdAvailableWidth = agentName 
  ? textWidth - 1 - stringWidth(agentName) - 3  // 减去 "@"、Agent 名和 " · "
  : textWidth;
```

**努力级别后缀**:
```typescript
const effortSuffix = getEffortSuffix(model, effortValue);
// 返回: "" | " (Low)" | " (Medium)" | " (High)" | " (Max)"
```

### 关键代码路径

1. **推广展示计数** (line 37-71):
   ```typescript
   useEffect(() => {
     if (showGuestPassesUpsell) {
       incrementGuestPassesSeenCount();
     }
   }, [showGuestPassesUpsell]);
   
   useEffect(() => {
     if (showOverageCreditUpsell && !showGuestPassesUpsell) {
       incrementOverageCreditUpsellSeenCount();
     }
   }, [showOverageCreditUpsell, showGuestPassesUpsell]);
   ```
   - 互斥展示：当 GuestPassesUpsell 显示时，OverageCreditUpsell 不增加计数

2. **Clawd 选择** (line 82-88):
   ```typescript
   let t4;
   if ($[7] === Symbol.for("react.memo_cache_sentinel")) {
     t4 = isFullscreenEnvEnabled() ? <AnimatedClawd /> : <Clawd />;
     $[7] = t4;
   }
   ```
   - 全屏环境下显示可交互的 AnimatedClawd
   - 使用 React Compiler 记忆化

3. **模型计费布局** (line 105-113):
   ```typescript
   let t7;
   if ($[11] !== shouldSplit || $[12] !== truncatedBilling || $[13] !== truncatedModel) {
     t7 = shouldSplit 
       ? <><Text dimColor>{truncatedModel}</Text><Text dimColor>{truncatedBilling}</Text></>
       : <Text dimColor>{truncatedModel} · {truncatedBilling}</Text>;
   }
   ```
   - 根据空间决定单行或双行显示

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| useMainLoopModel | `../../hooks/useMainLoopModel.js` | 获取当前模型 |
| useTerminalSize | `../../hooks/useTerminalSize.js` | 获取终端尺寸 |
| stringWidth | `../../ink/stringWidth.js` | 计算字符串显示宽度 |
| Box, Text | `../../ink.js` | UI 组件 |
| useAppState | `../../state/AppState.js` | 获取应用状态 |
| getEffortSuffix | `../../utils/effort.js` | 努力级别格式化 |
| truncate, truncatePath | `../../utils/logoV2Utils.js` | 文本截断工具 |
| isFullscreenEnvEnabled | `../../utils/fullscreen.js` | 全屏环境检测 |
| renderModelSetting | `../../utils/model/model.js` | 模型名称渲染 |
| OffscreenFreeze | `../OffscreenFreeze.js` | 离屏冻结优化 |
| AnimatedClawd, Clawd | `./AnimatedClawd.js`, `./Clawd.js` | 吉祥物组件 |
| GuestPassesUpsell, OverageCreditUpsell | `./GuestPassesUpsell.js`, `./OverageCreditUpsell.js` | 推广组件 |

### 依赖详情

**`useMainLoopModel`** (`src/hooks/useMainLoopModel.ts`):
- 返回当前使用的主循环模型
- 订阅 GrowthBook 刷新以响应模型配置变化

**`useTerminalSize`** (`src/hooks/useTerminalSize.ts`):
- 返回 `{ columns, rows }`
- 终端尺寸变化时触发重渲染

**`getLogoDisplayData`** (`src/utils/logoV2Utils.ts`):
```typescript
function getLogoDisplayData(): {
  version: string;
  cwd: string;
  billingType: string;
  agentName: string | undefined;
}
```

**`isFullscreenEnvEnabled`** (`src/utils/fullscreen.ts`):
- 检测全屏环境是否启用
- 考虑 `CLAUDE_CODE_NO_FLICKER` 环境变量
- 检测 tmux -CC 模式并自动禁用

### 调用关系

```
PromptInput.tsx / REPL.tsx
    ↓
CondensedLogo.tsx
    ├─→ hooks/useMainLoopModel.ts (模型)
    ├─→ hooks/useTerminalSize.ts (终端尺寸)
    ├─→ state/AppState.js (agent, effort)
    ├─→ utils/logoV2Utils.js (显示数据)
    ├─→ utils/fullscreen.ts (全屏检测)
    ├─→ LogoV2/AnimatedClawd.tsx (动画吉祥物)
    ├─→ LogoV2/Clawd.tsx (静态吉祥物)
    ├─→ LogoV2/GuestPassesUpsell.tsx (推广)
    └─→ LogoV2/OverageCreditUpsell.tsx (推广)
```

## 风险、边界与改进建议

### 已知风险

1. **宽度计算精度**: `stringWidth` 对于某些 Unicode 字符（如 emoji）的宽度计算可能不准确
   - 缓解: 使用 `Math.max` 确保最小宽度，留出安全边距

2. **信息过载**: 在极窄终端（< 40 列）下信息可能过于拥挤
   - 缓解: 最小宽度 20 列的限制，但超窄终端体验仍不佳

3. **推广组件竞争**: GuestPassesUpsell 和 OverageCreditUpsell 同时显示时布局可能混乱
   - 缓解: 代码逻辑确保互斥，但需维护者注意

### 边界情况

| 场景 | 行为 |
|------|------|
| columns < 35 | textWidth = 20，信息可能被大量截断 |
| agentName 很长 | 工作目录显示空间被压缩 |
| 模型名 + effortSuffix 很长 | 强制拆分为两行显示 |
| 非交互式会话 | 可能不渲染或渲染简化版本 |
| 远程模式 | cwd 显示包含服务器 URL |

### 改进建议

1. **优先级信息**: 在极窄终端下，可考虑隐藏低优先级信息（如版本号）
   ```typescript
   const showVersion = columns >= 50;
   const showBilling = columns >= 40;
   ```

2. **工具提示**: 截断的文本可考虑添加悬停提示显示完整内容（如果在支持鼠标的终端）

3. **配置化显示**: 允许用户通过设置选择显示/隐藏特定信息项
   ```typescript
   // 可能的设置
   {
     "condensedLogo": {
       "showVersion": false,
       "showBilling": true,
       "showAgent": true
     }
   }
   ```

4. **动态刷新优化**: 当前每次渲染都重新计算截断，可考虑缓存上次结果
   ```typescript
   const lastTruncation = useRef({ text, width, result });
   ```

5. **主题适配**: 当前使用固定的 `dimColor` 和 `bold`，可考虑支持主题自定义

6. **测试覆盖**: 建议添加:
   - 不同终端宽度下的布局快照测试
   - 文本截断逻辑的单元测试
   - 推广组件显示条件的测试

### 性能考虑

1. **React Compiler 优化**: 组件经过 React Compiler 编译，大量记忆化减少重渲染
   - 每个 Text 片段独立缓存
   - 仅在实际数据变化时更新

2. **OffscreenFreeze**: 使用 `OffscreenFreeze` 包装，当组件离屏时暂停更新

3. **useEffect 计数器**: 推广计数使用 `useEffect` 确保只在显示状态变化时触发

### 相关文件引用

- 主实现: `src/components/LogoV2/CondensedLogo.tsx`
- 调用方: `src/components/PromptInput/PromptInput.tsx`
- 模型钩子: `src/hooks/useMainLoopModel.ts`
- 终端尺寸: `src/hooks/useTerminalSize.ts`
- 工具函数: `src/utils/logoV2Utils.ts`
- 全屏检测: `src/utils/fullscreen.ts`
