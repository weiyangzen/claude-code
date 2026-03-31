# extra-usage-noninteractive.ts 研究文档

## 场景与职责

`extra-usage-noninteractive.ts` 是 `/extra-usage` 命令的非交互式（Non-Interactive）版本实现。该模块专门服务于以下场景：

1. **非交互式会话执行**：当 Claude Code 以非交互模式运行（如 CI/CD 管道、脚本调用、`claude -p` 模式）时，提供 `/extra-usage` 命令的文本输出版本
2. **无 TUI 环境**：在不支持或不需要 React/Ink 终端用户界面的环境中执行

该模块是 `extra-usage.tsx`（交互式版本）的平行实现，共享核心逻辑但适配不同的执行环境。

## 功能点目的

### 1. 核心逻辑复用
- **目的**：复用 `extra-usage-core.ts` 中的业务逻辑，避免代码重复
- **实现**：直接导入并调用 `runExtraUsage()` 函数
- **优势**：确保交互式和非交互式版本的行为一致性

### 2. 结果格式转换
- **目的**：将核心逻辑的 `ExtraUsageResult` 转换为非交互式命令期望的 `LocalCommandResult` 格式
- **实现**：根据 `result.type` 进行分支处理，生成纯文本输出

### 3. 浏览器打开结果处理
- **目的**：当需要打开浏览器时，提供适合非交互式环境的文本反馈
- **实现**：根据 `opened` 状态生成不同的提示消息，始终包含 URL

## 具体技术实现

### 关键流程

```
call()
├── 调用 runExtraUsage() 获取结果
├── 分支处理结果类型
│   ├── type === 'message' → 直接返回文本
│   └── type === 'browser-opened' → 构建包含 URL 的提示文本
└── 返回 LocalCommandResult
```

### 数据结构

#### 输入类型（来自 extra-usage-core.ts）
```typescript
type ExtraUsageResult =
  | { type: 'message'; value: string }
  | { type: 'browser-opened'; url: string; opened: boolean }
```

#### 输出类型（LocalCommandResult）
```typescript
// 来自 types/command.ts
type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
  | { type: 'skip' }
```

### 关键代码路径

| 功能 | 代码位置 | 说明 |
|------|----------|------|
| 核心调用 | line 4 | `const result = await runExtraUsage()` |
| 消息类型处理 | line 6-8 | 直接返回 `{ type: 'text', value: result.value }` |
| 浏览器类型处理 | line 10-15 | 根据 `opened` 状态构建不同消息 |

### 代码实现细节

```typescript
export async function call(): Promise<{ type: 'text'; value: string }> {
  const result = await runExtraUsage()

  if (result.type === 'message') {
    return { type: 'text', value: result.value }
  }

  return {
    type: 'text',
    value: result.opened
      ? `Browser opened to manage extra usage. If it didn't open, visit: ${result.url}`
      : `Please visit ${result.url} to manage extra usage.`,
  }
}
```

## 依赖与外部交互

### 导入依赖

| 模块路径 | 导入内容 | 用途 |
|----------|----------|------|
| `./extra-usage-core.js` | `runExtraUsage` | 核心逻辑函数 |

### 无外部 API 调用

该模块本身不直接调用任何外部 API，所有 API 交互都通过 `extra-usage-core.ts` 中的 `runExtraUsage()` 函数间接完成。

### 与命令系统的集成

在 `index.ts` 中，该模块被注册为 `extraUsageNonInteractive` 命令：

```typescript
export const extraUsageNonInteractive = {
  type: 'local',
  name: 'extra-usage',
  supportsNonInteractive: true,  // 标记支持非交互式
  description: 'Configure extra usage to keep working when limits are hit',
  isEnabled: () => isExtraUsageAllowed() && getIsNonInteractiveSession(),
  get isHidden() {
    return !getIsNonInteractiveSession()  // 交互式会话中隐藏
  },
  load: () => import('./extra-usage-noninteractive.js'),
}
```

## 风险、边界与改进建议

### 潜在风险

1. **行为一致性风险**
   - 风险：如果 `extra-usage-core.ts` 的接口变更，此模块需要同步更新
   - 缓解：TypeScript 类型检查会在编译期捕获大部分接口不匹配问题

2. **用户体验差异**
   - 风险：非交互式版本无法提供交互式版本的用户体验（如登录流程）
   - 缓解：这是设计预期，非交互式环境通常不需要复杂的 UI 交互

### 边界情况

1. **浏览器打开失败**：在非交互式环境中（如 Docker 容器、SSH 会话），浏览器通常无法打开
   - 处理：始终返回包含 URL 的文本消息，让用户可以手动访问

2. **无显示器环境**：`openBrowser()` 会返回 `opened: false`
   - 处理：生成 "Please visit ${url}..." 的提示

3. **CI/CD 管道执行**：在自动化脚本中调用
   - 处理：纯文本输出便于日志记录和后续处理

### 改进建议

1. **增加 JSON 输出选项**
   ```typescript
   // 建议：支持 --json 标志，便于脚本解析
   if (process.env.CLAUDE_OUTPUT_JSON) {
     return { type: 'text', value: JSON.stringify(result) }
   }
   ```

2. **增加退出码支持**
   ```typescript
   // 建议：根据结果返回不同的退出码
   // 0 = 成功, 1 = 需要用户操作, 2 = 错误
   ```

3. **与交互式版本的功能对齐**
   - 当前非交互式版本缺少交互式版本的登录流程处理
   - 考虑在非交互式环境中提供 OAuth 设备码流程支持

4. **日志增强**
   ```typescript
   // 建议：增加调试日志
   import { logForDebugging } from '../../utils/debug.js'
   logForDebugging(`extra-usage-noninteractive: result type=${result.type}`)
   ```

5. **URL 验证**
   ```typescript
   // 建议：验证 URL 格式后再返回
   import { validateUrl } from '../../utils/browser.js'
   if (!isValidUrl(result.url)) {
     return { type: 'text', value: 'Error: Invalid URL generated' }
   }
   ```
