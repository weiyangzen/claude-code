# 研究文档: src/commands/stickers/stickers.ts

## 场景与职责

`src/commands/stickers/stickers.ts` 是 Claude Code 项目中 `/stickers` 命令的实际实现文件。它负责在用户执行 `/stickers` 命令时，打开系统默认浏览器并导航到 Sticker Mule 上的 Claude Code 贴纸订购页面。

这是一个轻量级的本地命令实现，展示了 Claude Code 中本地命令的标准实现模式：
1. 导入必要的类型和工具函数
2. 导出 `call` 函数作为命令入口点
3. 执行具体操作（打开浏览器）
4. 返回标准化的 `LocalCommandResult` 结果

## 功能点目的

1. **品牌推广**: 为用户提供一种简单的方式获取 Claude Code 品牌贴纸
2. **用户体验**: 通过简单的斜杠命令 `/stickers` 即可访问订购页面，无需手动输入 URL
3. **浏览器集成**: 智能检测系统平台，使用适当的命令打开默认浏览器
4. **优雅降级**: 当浏览器无法打开时，向用户显示 URL 以便手动访问

## 具体技术实现

### 关键流程

```
用户输入 /stickers
    ↓
命令系统调用 stickers.ts 的 call() 函数
    ↓
构建目标 URL: https://www.stickermule.com/claudecode
    ↓
调用 openBrowser(url) 打开浏览器
    ↓
根据执行结果返回 LocalCommandResult
    ├── 成功: { type: 'text', value: 'Opening sticker page in browser…' }
    └── 失败: { type: 'text', value: 'Failed to open browser. Visit: ${url}' }
```

### 核心代码实现

```typescript
import type { LocalCommandResult } from '../../types/command.js'
import { openBrowser } from '../../utils/browser.js'

export async function call(): Promise<LocalCommandResult> {
  const url = 'https://www.stickermule.com/claudecode'
  const success = await openBrowser(url)

  if (success) {
    return { type: 'text', value: 'Opening sticker page in browser…' }
  } else {
    return {
      type: 'text',
      value: `Failed to open browser. Visit: ${url}`,
    }
  }
}
```

### 数据结构

#### LocalCommandResult
定义在 `src/types/command.ts`（第16-23行）：
```typescript
export type LocalCommandResult =
  | { type: 'text'; value: string }
  | {
      type: 'compact'
      compactionResult: CompactionResult
      displayText?: string
    }
  | { type: 'skip' }  // Skip messages
```

本命令使用 `type: 'text'` 变体，返回纯文本消息给用户。

### 浏览器打开机制

`openBrowser()` 函数定义在 `src/utils/browser.ts`（第39-67行）：

```typescript
export async function openBrowser(url: string): Promise<boolean> {
  try {
    // 1. URL 验证
    validateUrl(url)
    
    // 2. 平台检测和命令选择
    const browserEnv = process.env.BROWSER
    const platform = process.platform
    
    // 3. 执行平台特定的打开命令
    if (platform === 'win32') {
      // Windows: 使用 rundll32 或 BROWSER 环境变量
    } else {
      // macOS: open 命令
      // Linux: xdg-open 命令
    }
  } catch (_) {
    return false
  }
}
```

#### 平台支持

| 平台 | 命令 | 说明 |
|-----|------|------|
| Windows | `rundll32 url,OpenURL` | 系统内置，无需额外依赖 |
| Windows (自定义) | `%BROWSER%` | 通过环境变量指定浏览器 |
| macOS | `open` | 系统内置命令 |
| Linux | `xdg-open` | xdg-utils 包提供 |

#### URL 验证

```typescript
function validateUrl(url: string): void {
  let parsedUrl: URL
  try {
    parsedUrl = new URL(url)
  } catch (_error) {
    throw new Error(`Invalid URL format: ${url}`)
  }
  
  // 仅允许 http: 和 https: 协议
  if (parsedUrl.protocol !== 'http:' && parsedUrl.protocol !== 'https:') {
    throw new Error(`Invalid URL protocol: must use http:// or https://`)
  }
}
```

## 关键代码路径与文件引用

### 当前文件
- **路径**: `src/commands/stickers/stickers.ts`
- **行数**: 16 行
- **导出**: `call` 函数（异步）

### 依赖文件

| 文件路径 | 用途 | 关键内容 |
|---------|------|---------|
| `src/types/command.ts` | 类型定义 | `LocalCommandResult` 类型 |
| `src/utils/browser.ts` | 浏览器工具 | `openBrowser()` 函数 |
| `src/utils/execFileNoThrow.ts` | 进程执行 | 被 `browser.ts` 使用，安全执行外部命令 |

### 调用链

```
src/commands/stickers/index.ts (命令定义)
    ↓ load()
src/commands/stickers/stickers.ts (本文件)
    ↓ import
src/utils/browser.ts (浏览器工具)
    ↓ import
src/utils/execFileNoThrow.ts (进程执行)
```

## 依赖与外部交互

### 内部依赖

```typescript
import type { LocalCommandResult } from '../../types/command.js'
import { openBrowser } from '../../utils/browser.js'
```

1. **`LocalCommandResult`**: 命令结果类型，确保所有本地命令返回统一格式的结果
2. **`openBrowser`**: 跨平台浏览器打开工具函数

### 外部系统交互

1. **操作系统命令**:
   - `open` (macOS)
   - `xdg-open` (Linux)
   - `rundll32` (Windows)
   - 自定义浏览器（通过 `BROWSER` 环境变量）

2. **网络请求**:
   - 打开 `https://www.stickermule.com/claudecode`
   - 依赖外部网站可用性

### 环境变量

| 变量名 | 作用 | 平台 |
|-------|------|------|
| `BROWSER` | 指定自定义浏览器可执行文件 | 所有平台 |

## 风险、边界与改进建议

### 当前风险

1. **外部依赖风险**:
   - Sticker Mule 网站可用性：如果网站宕机或 URL 变更，命令将失去意义
   - 硬编码 URL 需要代码更新才能修改

2. **平台兼容性问题**:
   - Linux 系统可能未安装 `xdg-open`（属于 xdg-utils 包）
   - 某些受限环境（如容器、WSL 无 GUI）可能无法打开浏览器
   - Windows 的 `rundll32` 方法在某些企业环境中可能被策略限制

3. **无参数化**:
   - 命令不接受任何参数，功能固定
   - 无法让用户选择打开特定页面或获取更多信息

4. **缺乏遥测**:
   - 无法统计命令使用频率
   - 无法追踪用户是否成功完成订购

### 边界情况处理

| 场景 | 当前行为 | 评估 |
|-----|---------|------|
| 浏览器打开成功 | 显示 "Opening sticker page in browser…" | ✅ 良好 |
| 浏览器打开失败 | 显示 "Failed to open browser. Visit: ${url}" | ✅ 优雅降级 |
| URL 格式错误 | 由 `validateUrl` 捕获，返回失败 | ✅ 安全 |
| 非标准协议 | 被拒绝（仅允许 http/https） | ✅ 安全 |
| 无图形界面环境 | 返回失败，显示 URL | ⚠️ 可接受 |

### 改进建议

#### 高优先级

1. **URL 配置化**:
   ```typescript
   const url = process.env.CLAUDE_STICKERS_URL || 'https://www.stickermule.com/claudecode'
   ```
   允许通过环境变量覆盖默认 URL。

2. **添加遥测**:
   ```typescript
   import { trackEvent } from '../../utils/analytics.js'
   
   export async function call(): Promise<LocalCommandResult> {
     trackEvent('stickers_command_invoked')
     // ... 现有逻辑
   }
   ```

3. **支持预览模式**:
   ```typescript
   export async function call(args: string): Promise<LocalCommandResult> {
     if (args.includes('--preview') || args.includes('-p')) {
       return { type: 'text', value: `Sticker page URL: ${url}` }
     }
     // ... 现有逻辑
   }
   ```

#### 中优先级

4. **添加更多贴纸选项**:
   - 支持多个贴纸供应商
   - 允许用户选择不同风格的贴纸

5. **改进错误信息**:
   ```typescript
   return {
     type: 'text',
     value: `Unable to open browser automatically. Please visit: ${url}\n\nTip: You can set the BROWSER environment variable to specify your preferred browser.`,
   }
   ```

6. **异步确认**:
   - 在 TUI 环境中，可以添加确认提示，让用户确认是否打开外部链接

#### 低优先级

7. **缓存机制**:
   - 缓存页面元数据（标题、描述），即使无法打开浏览器也能显示信息

8. **国际化**:
   - 支持多语言的错误消息和提示

### 测试建议

1. **单元测试**:
   ```typescript
   describe('stickers command', () => {
     it('should return success message when browser opens', async () => {
       vi.mocked(openBrowser).mockResolvedValue(true)
       const result = await call()
       expect(result).toEqual({ type: 'text', value: 'Opening sticker page in browser…' })
     })
     
     it('should return URL when browser fails to open', async () => {
       vi.mocked(openBrowser).mockResolvedValue(false)
       const result = await call()
       expect(result.value).toContain('https://www.stickermule.com/claudecode')
     })
   })
   ```

2. **集成测试**:
   - 验证 `openBrowser` 在各种平台上的正确调用
   - 验证 URL 验证逻辑

3. **手动测试矩阵**:
   - macOS + Safari/Chrome/Firefox
   - Windows + Edge/Chrome
   - Linux + Firefox/Chrome
   - WSL 环境
   - 无 GUI 环境（SSH 远程）

---

**文档生成时间**: 2026-04-01
**关联版本**: Claude Code 内部版本
**维护者**: Claude Code 团队
