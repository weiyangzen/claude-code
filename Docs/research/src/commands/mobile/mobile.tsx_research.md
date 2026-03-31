# 研究文档: src/commands/mobile/mobile.tsx

## 场景与职责

本文件实现 `/mobile` slash 命令的交互式 UI 组件，核心职责包括：
1. **QR 码生成**：使用 `qrcode` 库为 iOS 和 Android 平台生成 UTF-8 文本格式二维码
2. **平台切换**：支持用户在 iOS 和 Android 之间切换查看对应平台的下载链接
3. **终端渲染**：使用 Ink（React for CLI）在终端内渲染可交互的二维码面板
4. **键盘交互**：处理 Tab/方向键切换平台，ESC/Q/Ctrl+C 关闭面板

该组件是 Claude Code CLI 中典型的 `local-jsx` 类型命令实现，为用户提供无需离开终端即可获取移动应用下载链接的便捷方式。

## 功能点目的

### 1. 双平台 QR 码展示
- **iOS**：链接至 Apple App Store (`https://apps.apple.com/app/claude-by-anthropic/id6473753684`)
- **Android**：链接至 Google Play Store (`https://play.google.com/store/apps/details?id=com.anthropic.claude`)
- **异步生成**：使用 `Promise.all` 并行生成两个平台的二维码，提升加载速度

### 2. 交互式平台切换
- **Tab/左右方向键**：在 iOS 和 Android 之间循环切换
- **视觉反馈**：当前选中平台显示为粗体+下划线，未选中平台暗淡显示
- **URL 显示**：在二维码下方显示当前平台的完整下载链接

### 3. 键盘导航与关闭
- **ESC 键**：关闭面板（通过 `useKeybinding('confirm:no', ...)` 绑定）
- **Q 键 / Ctrl+C**：关闭面板（自定义键盘事件处理）
- **操作提示**：界面底部显示 `(tab to switch, esc to close)` 提示

## 具体技术实现

### 关键数据结构

#### 平台配置
```typescript
type Platform = 'ios' | 'android';

const PLATFORMS: Record<Platform, { url: string }> = {
  ios: {
    url: 'https://apps.apple.com/app/claude-by-anthropic/id6473753684'
  },
  android: {
    url: 'https://play.google.com/store/apps/details?id=com.anthropic.claude'
  }
};
```

#### 组件 Props
```typescript
type Props = {
  onDone: () => void;  // 回调函数，关闭面板时调用
};
```

#### 状态管理
```typescript
const [platform, setPlatform] = useState<Platform>('ios');
const [qrCodes, setQrCodes] = useState<Record<Platform, string>>({
  ios: '',
  android: ''
});
```

### 核心流程

#### 1. QR 码生成流程
```
组件挂载
  → useEffect 触发
  → Promise.all([
      qrToString(PLATFORMS.ios.url, { type: 'utf8', errorCorrectionLevel: 'L' }),
      qrToString(PLATFORMS.android.url, { type: 'utf8', errorCorrectionLevel: 'L' })
    ])
  → setQrCodes({ ios, android })
  → 触发重新渲染显示二维码
```

**QR 码配置说明**：
- `type: 'utf8'`：生成可在终端直接打印的文本格式（使用 Unicode 块字符）
- `errorCorrectionLevel: 'L'`：低容错级别，生成更紧凑的二维码

#### 2. 键盘事件处理流程
```
用户按键
  → onKeyDown 处理器
  → 判断按键类型：
    - 'q' 或 'ctrl+c' → 调用 onDone() 关闭
    - 'tab'/'left'/'right' → 切换平台（ios ↔ android）
  → preventDefault() 阻止默认行为
```

#### 3. 平台切换逻辑
```typescript
setPlatform(prev => prev === 'ios' ? 'android' : 'ios');
```

### React Compiler 优化

代码经过 React Compiler（React 19+）编译，包含大量自动生成的缓存逻辑：
- `_c(52)`：创建包含 52 个缓存槽的编译器上下文
- `$[n]` 访问模式：读取第 n 个缓存槽的值
- `Symbol.for("react.memo_cache_sentinel")`：缓存未命中标记

这种编译方式确保组件在状态不变时避免不必要的重新渲染，对频繁键盘交互的 TUI 组件尤为重要。

## 关键代码路径与文件引用

### 导入依赖
| 路径 | 用途 |
|------|------|
| `qrcode` | QR 码生成库，使用 `toString` 函数 |
| `react` | React 核心库，使用 `useCallback`, `useEffect`, `useState` |
| `../../components/design-system/Pane.js` | 面板容器组件，提供统一的边框样式 |
| `../../ink/events/keyboard-event.js` | 键盘事件类型定义 |
| `../../ink.js` | Ink 核心，导入 `Box`, `Text` 组件 |
| `../../keybindings/useKeybinding.js` | 键盘绑定 Hook |
| `../../types/command.js` | `LocalJSXCommandOnDone` 类型 |

### 导出接口
```typescript
export async function call(onDone: LocalJSXCommandOnDone): Promise<React.ReactNode> {
  return <MobileQRCode onDone={onDone} />;
}
```

### 调用链
```
用户执行 /mobile 命令
  → commands.ts 匹配命令
  → 调用 load() → import('./mobile.js')
  → 执行 call(onDone)
  → 返回 <MobileQRCode> React 元素
  → Ink 渲染引擎渲染到终端
  → 用户交互（键盘事件）
  → onDone() 被调用 → 组件卸载
```

### 相关文件
| 文件 | 关系 |
|------|------|
| `src/commands/mobile/index.ts` | 命令入口，懒加载本文件 |
| `src/commands.ts` | 命令注册中心，包含 `mobile` 命令定义 |
| `src/types/command.ts` | `LocalJSXCommandOnDone`, `LocalJSXCommandCall` 类型定义 |
| `src/keybindings/useKeybinding.ts` | `useKeybinding` Hook 实现 |
| `src/components/design-system/Pane.tsx` | 面板 UI 组件 |
| `src/ink/events/keyboard-event.ts` | `KeyboardEvent` 类定义 |

## 依赖与外部交互

### 外部 npm 依赖
| 包名 | 版本要求 | 用途 |
|------|----------|------|
| `qrcode` | ^1.x | 生成 UTF-8 文本格式二维码 |
| `react` | ^18.x / ^19.x | 组件运行时 |

### 项目内部依赖

#### UI 组件层
- **Pane**：来自 `design-system`，提供带顶部边框的面板容器
  - 支持 `color` 属性设置主题色
  - 自动处理模态框内外的布局差异

- **Box**：来自 `ink.js`，Ink 的布局容器
  - `flexDirection="column"`：垂直布局
  - `paddingX={2}`：水平内边距
  - `tabIndex={0}`：可聚焦
  - `autoFocus={true}`：自动获取焦点

- **Text**：来自 `ink.js`，文本渲染组件
  - `bold`：粗体
  - `underline`：下划线
  - `dimColor`：暗淡颜色（用于提示文字）

#### 键盘交互层
- **useKeybinding**：声明式键盘绑定 Hook
  - `useKeybinding('confirm:no', handleClose, { context: 'Confirmation' })`
  - 将 ESC 键映射到关闭操作

- **onKeyDown**：底层键盘事件处理
  - 处理 Tab/方向键切换平台
  - 处理 Q/Ctrl+C 关闭

### 类型系统

#### LocalJSXCommandOnDone
```typescript
type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  }
) => void
```

当用户关闭面板时，`onDone()` 被调用，通知命令系统组件已完成。

## 风险、边界与改进建议

### 当前风险

| 风险点 | 描述 | 等级 |
|--------|------|------|
| **QR 生成失败无反馈** | `generateQRCodes().catch(_temp)` 静默吞掉异常，若 `qrcode` 包损坏或内存不足，用户将看到空白区域 | 中 |
| **硬编码 URL** | App Store 和 Play Store 链接硬编码，若链接变更需重新发布版本 | 低 |
| **无加载状态** | QR 码生成期间无 loading 指示，网络慢或性能差时用户体验不佳 | 低 |
| **编译后代码可读性差** | React Compiler 生成的缓存逻辑使代码难以调试 | 低 |

### 边界情况

1. **终端宽度不足**：QR 码需要一定宽度才能正确显示，窄终端可能导致二维码变形
2. **终端不支持 Unicode**：`type: 'utf8'` 依赖 Unicode 块字符，老旧终端可能显示为乱码
3. **快速切换平台**：用户快速按 Tab 键可能导致状态更新竞争（React 18+ Concurrent Mode 已缓解）
4. **焦点管理**：`autoFocus` 依赖 Ink 的焦点系统，若与其他组件冲突可能导致键盘事件失效

### 改进建议

#### 1. 错误处理增强
```typescript
// 建议：添加错误状态和用户反馈
const [error, setError] = useState<string | null>(null);

const generateQRCodes = async () => {
  try {
    const [ios, android] = await Promise.all([...]);
    setQrCodes({ ios, android });
  } catch (err) {
    setError('Failed to generate QR codes. Please try again.');
    logError(toError(err));
  }
};
```

#### 2. 加载状态指示
```typescript
const [isLoading, setIsLoading] = useState(true);

useEffect(() => {
  setIsLoading(true);
  generateQRCodes().finally(() => setIsLoading(false));
}, []);

// 渲染时显示 loading 状态
{isLoading && <Text dimColor>Generating QR codes...</Text>}
```

#### 3. 终端宽度检测
```typescript
import { useTerminalViewport } from '../../ink.js';

const { width } = useTerminalViewport();
const minQRWidth = 40;

{width < minQRWidth && (
  <Text color="warning">Terminal too narrow for QR code. Please resize.</Text>
)}
```

#### 4. URL 配置化
考虑将 URL 移至配置文件或环境变量，便于热更新：
```typescript
const PLATFORMS = {
  ios: { 
    url: process.env.CLAUDE_IOS_APP_URL || 'https://apps.apple.com/...' 
  },
  android: { 
    url: process.env.CLAUDE_ANDROID_APP_URL || 'https://play.google.com/...' 
  }
};
```

#### 5. 测试覆盖
当前文件无单元测试，建议添加：
- QR 码生成成功/失败场景
- 键盘事件处理（Tab 切换、ESC 关闭）
- 平台状态切换逻辑

### 性能考量

1. **并行生成**：使用 `Promise.all` 同时生成两个平台的二维码，减少等待时间
2. **编译器优化**：React Compiler 的自动缓存避免不必要的重新渲染
3. **懒加载**：通过 `index.ts` 的动态导入，确保 `qrcode` 库仅在需要时加载

---

*文档生成时间：2026-04-01*
*关联文件：index.ts, commands.ts, types/command.ts, keybindings/useKeybinding.ts, components/design-system/Pane.tsx*
