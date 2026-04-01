# figures.ts 深度研究文档

## 场景与职责

`figures.ts` 是 Claude Code CLI 中定义 Unicode 符号和图标的核心常量文件。它提供跨平台一致的视觉符号，用于状态指示、UI 装饰和交互反馈。该文件处理 macOS 与其他平台（Windows/Linux）之间的字符支持差异。

### 核心使用场景
1. **状态指示**：使用符号指示任务状态（运行中、暂停、完成、失败）
2. **UI 装饰**：为界面元素添加视觉标识（列表标记、箭头、图标）
3. **努力级别显示**：用圆形符号表示 AI 模型的努力程度设置
4. **MCP 订阅指示**：显示资源更新、频道消息和跨会话注入
5. **审查状态**：在代码审查流程中标记运行/完成状态
6. **桥接状态**：显示桥接连接状态（加载中、就绪、失败）

---

## 功能点目的

### 1. 基础符号

| 常量 | 符号 | Unicode | 用途 |
|------|------|---------|------|
| `BLACK_CIRCLE` | ⏺ (macOS) / ● (其他) | U+23FA / U+25CF | 主要圆点指示器 |
| `BULLET_OPERATOR` | ∙ | U+2219 | 列表项标记 |
| `TEARDROP_ASTERISK` | ✻ | U+273B | 装饰性星号 |
| `UP_ARROW` | ↑ | U+2191 | 向上指示（Opus 1M 合并通知） |
| `DOWN_ARROW` | ↓ | U+2193 | 向下指示（滚动提示） |
| `LIGHTNING_BOLT` | ↯ | U+21AF | 快速模式指示 |

**平台适配**：`BLACK_CIRCLE` 在 macOS 使用 `⏺`（更美观），其他平台使用 `●`（兼容性更好）。

### 2. 努力级别符号

| 常量 | 符号 | Unicode | 含义 |
|------|------|---------|------|
| `EFFORT_LOW` | ○ | U+25CB | 低努力级别 |
| `EFFORT_MEDIUM` | ◐ | U+25D0 | 中等努力级别 |
| `EFFORT_HIGH` | ● | U+25CF | 高努力级别 |
| `EFFORT_MAX` | ◉ | U+25C9 | 最大努力级别（仅 Opus 4.6） |

**视觉设计**：使用填充程度表示努力级别，从空圆到实心圆再到双圆。

### 3. 媒体/触发状态

| 常量 | 符号 | Unicode | 用途 |
|------|------|---------|------|
| `PLAY_ICON` | ▶ | U+25B6 | 播放/运行中 |
| `PAUSE_ICON` | ⏸ | U+23F8 | 暂停 |

### 4. MCP 订阅指示器

| 常量 | 符号 | Unicode | 用途 |
|------|------|---------|------|
| `REFRESH_ARROW` | ↻ | U+21BB | 资源更新指示 |
| `CHANNEL_ARROW` | ← | U+2190 | 入站频道消息 |
| `INJECTED_ARROW` | → | U+2192 | 跨会话注入消息 |
| `FORK_GLYPH` | ⑂ | U+2442 | Fork 指令指示 |

### 5. 审查状态（UltraReview）

| 常量 | 符号 | Unicode | 状态 |
|------|------|---------|------|
| `DIAMOND_OPEN` | ◇ | U+25C7 | 运行中 |
| `DIAMOND_FILLED` | ◆ | U+25C6 | 已完成/失败 |
| `REFERENCE_MARK` | ※ | U+203B | 离线摘要回顾标记（komejirushi） |

### 6. 问题标记

| 常量 | 符号 | Unicode | 用途 |
|------|------|---------|------|
| `FLAG_ICON` | ⚑ | U+2691 | 问题标记横幅 |

### 7. 引用块装饰

| 常量 | 符号 | Unicode | 用途 |
|------|------|---------|------|
| `BLOCKQUOTE_BAR` | ▎ | U+258E | 引用块行前缀（左四分之一块） |
| `HEAVY_HORIZONTAL` | ━ | U+2501 | 粗线绘制水平线 |

### 8. 桥接状态

| 常量 | 值 | 用途 |
|------|-----|------|
| `BRIDGE_SPINNER_FRAMES` | `['·|·', '·/·', '·—·', '·\·']` | 加载动画帧 |
| `BRIDGE_READY_INDICATOR` | `·✔︎·` | 就绪指示 |
| `BRIDGE_FAILED_INDICATOR` | `×` | 失败指示 |

---

## 具体技术实现

### 数据结构

```typescript
// 平台检测
import { env } from '../utils/env.js'

// 基础符号（平台适配）
export const BLACK_CIRCLE = env.platform === 'darwin' ? '⏺' : '●'

// 标准符号
export const BULLET_OPERATOR = '∙'
export const TEARDROP_ASTERISK = '✻'
// ... 更多符号

// 数组类型（动画帧）
export const BRIDGE_SPINNER_FRAMES = [
  '\u00b7|\u00b7',
  '\u00b7/\u00b7',
  '\u00b7\u2014\u00b7',
  '\u00b7\\\u00b7',
]
```

### 关键代码路径

#### 1. 努力级别显示路径

```
设置/显示努力级别
    ↓
src/components/EffortIndicator.ts
    ↓
根据级别选择 EFFORT_LOW / EFFORT_MEDIUM / EFFORT_HIGH / EFFORT_MAX
    ↓
渲染到状态栏或设置界面
```

**关键文件引用**：
- `src/components/EffortIndicator.ts`: 努力级别指示器

#### 2. 消息渲染路径

```
渲染各种消息类型
    ↓
src/components/messages/*.tsx
    ↓
使用 BLACK_CIRCLE, BULLET_OPERATOR, FLAG_ICON 等
    ↓
渲染到终端 UI
```

**关键文件引用**：
- `src/components/messages/AssistantToolUseMessage.tsx`
- `src/components/messages/AssistantTextMessage.tsx`
- `src/components/messages/UserTextMessage.tsx`
- `src/components/messages/AttachmentMessage.tsx`
- `src/components/messages/UserToolResultMessage/UserToolErrorMessage.tsx`
- `src/components/messages/UserAgentNotificationMessage.tsx`
- `src/components/messages/UserChannelMessage.tsx`
- `src/components/messages/UserResourceUpdateMessage.tsx`
- `src/components/messages/UserLocalCommandOutputMessage.tsx`
- `src/components/messages/SystemTextMessage.tsx`

#### 3. 桥接 UI 路径

```
桥接连接状态变化
    ↓
src/bridge/bridgeUI.ts
    ↓
使用 BRIDGE_SPINNER_FRAMES 显示加载动画
使用 BRIDGE_READY_INDICATOR 显示就绪状态
使用 BRIDGE_FAILED_INDICATOR 显示失败状态
    ↓
更新桥接对话框 UI
```

**关键文件引用**：
- `src/bridge/bridgeUI.ts`: 桥接 UI 状态管理
- `src/components/BridgeDialog.tsx`: 桥接对话框组件

#### 4. 工具加载路径

```
工具执行中
    ↓
src/components/ToolUseLoader.tsx
    ↓
使用 TEARDROP_ASTERISK 等符号
    ↓
显示加载状态
```

**关键文件引用**：
- `src/components/ToolUseLoader.tsx`: 工具使用加载指示

#### 5. 任务标签路径

```
生成任务标签
    ↓
src/tasks/pillLabel.ts
    ↓
使用各种符号装饰标签
    ↓
显示在任务列表中
```

**关键文件引用**：
- `src/tasks/pillLabel.ts`: 任务标签生成

#### 6. 权限模式路径

```
显示权限模式
    ↓
src/utils/permissions/PermissionMode.ts
    ↓
使用相关符号
    ↓
渲染权限指示器
```

**关键文件引用**：
- `src/utils/permissions/PermissionMode.ts`: 权限模式显示

---

## 依赖与外部交互

### 内部依赖

| 导入 | 用途 |
|------|------|
| `../utils/env.js` | 平台检测（`env.platform`） |

### 被依赖方

| 文件 | 使用的符号 | 用途 |
|------|-----------|------|
| `src/components/EffortIndicator.ts` | `EFFORT_*` | 努力级别显示 |
| `src/components/Spinner.tsx` | 多个符号 | 加载指示器 |
| `src/components/ToolUseLoader.tsx` | `TEARDROP_ASTERISK` 等 | 工具加载 |
| `src/components/Messages.tsx` | 多个符号 | 消息渲染 |
| `src/components/BridgeDialog.tsx` | `BRIDGE_*` | 桥接对话框 |
| `src/components/CompactSummary.tsx` | 多个符号 | 摘要显示 |
| `src/components/Passes/Passes.tsx` | `DIAMOND_*` | 审查状态 |
| `src/components/PromptInput/IssueFlagBanner.tsx` | `FLAG_ICON` | 问题标记 |
| `src/components/FastIcon.tsx` | `LIGHTNING_BOLT` | 快速模式图标 |
| `src/components/LogoV2/AnimatedAsterisk.tsx` | `TEARDROP_ASTERISK` | Logo 动画 |
| `src/components/LogoV2/Opus1mMergeNotice.tsx` | `UP_ARROW` | 合并通知 |
| `src/components/permissions/WorkerBadge.tsx` | 多个符号 | 工作器徽章 |
| `src/components/tasks/*.tsx` | 多个符号 | 任务相关 UI |
| `src/bridge/bridgeUI.ts` | `BRIDGE_*` | 桥接状态 |
| `src/tasks/pillLabel.ts` | 多个符号 | 任务标签 |
| `src/utils/permissions/PermissionMode.ts` | 多个符号 | 权限显示 |
| `src/utils/markdown.ts` | `BLOCKQUOTE_BAR` | Markdown 渲染 |
| `src/utils/model/model.ts` | 多个符号 | 模型信息显示 |

### 平台兼容性

| 平台 | 特殊处理 |
|------|---------|
| macOS (`darwin`) | `BLACK_CIRCLE` 使用 `⏺` |
| Windows/Linux | `BLACK_CIRCLE` 使用 `●` |

---

## 风险、边界与改进建议

### 当前风险

1. **字体支持差异**
   - 不同终端/系统对 Unicode 字符的支持程度不同
   - 某些符号可能在旧版 Windows 终端显示为方框或问号

2. **等宽对齐问题**
   - 某些 Unicode 字符（尤其是 emoji 风格的）可能不是严格的单宽度
   - 可能导致 UI 对齐问题

3. **平台检测依赖**
   - `BLACK_CIRCLE` 的平台适配依赖 `env.platform`
   - 如果平台检测不准确，可能导致显示问题

4. **硬编码符号**
   - 符号直接硬编码在文件中
   - 主题定制困难

### 边界情况

| 场景 | 行为 |
|------|------|
| 终端不支持 Unicode | 显示为 `?` 或方框 |
| 等宽字体缺失 | 对齐可能错乱 |
| SSH 到远程服务器 | 依赖远程服务器的字体配置 |
| 非交互式输出 | 符号可能干扰管道处理 |

### 改进建议

1. **主题化支持**
   ```typescript
   // 建议添加主题配置
   export interface FigureTheme {
     bullet: string
     arrow: { up: string; down: string }
     effort: { low: string; medium: string; high: string; max: string }
     // ...
   }
   
   export const THEMES: Record<string, FigureTheme> = {
     default: { /* 当前符号 */ },
     ascii: { /* ASCII 替代 */ },
     minimal: { /* 极简风格 */ }
   }
   ```

2. **运行时检测**
   ```typescript
   // 建议检测终端 Unicode 支持
   export function supportsUnicode(): boolean {
     // 检查 LANG、LC_ALL 环境变量
     // 检查 TERM 类型
   }
   
   export function getSafeFigure(figure: string, fallback: string): string {
     return supportsUnicode() ? figure : fallback
   }
   ```

3. **自动化测试**
   - 验证所有符号在各平台的显示效果
   - 检查等宽属性
   - 确保无重复或冲突的符号分配

4. **文档化符号语义**
   ```typescript
   // 建议添加 JSDoc 说明使用场景
   /**
    * ● / ⏺ - 主要圆点指示器
    * @used_in 状态指示、列表标记
    * @platform macOS 使用 ⏺，其他使用 ●
    */
   export const BLACK_CIRCLE = ...
   ```

5. **与 figures 库的关系**
   - 项目已依赖 `figures` npm 包
   - 考虑评估是否可以迁移到 `figures` 库的标准符号
   - 或明确区分自定义符号与 `figures` 库符号的使用场景

### 与 figures npm 包的关系

```
constants/figures.ts (自定义符号)
    ↓ 部分导出被
outputStyles.ts 等使用
    ↓ 同时
figures npm 包
    ↓ 也被
outputStyles.ts 等使用
```

当前两者并存：
- `figures` 库：提供跨平台兼容的标准符号
- `constants/figures.ts`：提供项目特定的自定义符号

建议定期评估是否需要自定义符号，或可以迁移到标准库。
