# promptEditor.ts 深度研究文档

## 场景与职责

`promptEditor.ts` 是 Claude Code 中负责**提示编辑器集成**的核心工具模块。它提供了在外部编辑器中编辑文件和提示的功能，支持 GUI 编辑器（如 VS Code）和终端编辑器（如 vim、nano）的无缝集成。

### 核心职责
1. **文件编辑**：在外部编辑器中打开并编辑文件
2. **提示编辑**：支持提示文本的展开、编辑和重新折叠
3. **粘贴内容处理**：处理粘贴内容的展开和重新折叠
4. **终端管理**：在编辑器使用时暂停/恢复 Ink 渲染

### 使用场景
- 用户通过 `/edit` 命令编辑提示
- 在外部编辑器中编辑计划文件
- 编辑 Agent 提示
- 编辑用户问题

---

## 功能点目的

### 1. 外部编辑器支持

支持两类编辑器：

| 类型 | 示例 | 处理方式 |
|-----|------|---------|
| GUI 编辑器 | VS Code、Sublime Text | 分离式启动，暂停 Ink |
| 终端编辑器 | vim、nano、emacs | 交替屏幕，接管终端 |

### 2. 编辑器命令覆盖

```typescript
const EDITOR_OVERRIDES: Record<string, string> = {
  code: 'code -w',      // VS Code: 等待文件关闭
  subl: 'subl --wait',  // Sublime Text: 等待文件关闭
}
```

**目的**：确保编辑器在文件关闭后才返回。

### 3. 粘贴内容处理

```typescript
export function editPromptInEditor(
  currentPrompt: string,
  pastedContents?: Record<number, PastedContent>,
): EditorResult
```

**流程**：
1. 展开粘贴内容引用
2. 写入临时文件
3. 调用编辑器
4. 读取编辑后的内容
5. 重新折叠粘贴内容

---

## 具体技术实现

### 数据结构

```typescript
export type EditorResult = {
  content: string | null
  error?: string
}

type PastedContent = {
  type: 'text'
  content: string
}
```

### 核心流程

#### 1. 文件编辑流程

```
editFileInEditor(filePath)
├── 获取 Ink 实例
├── 获取外部编辑器配置
├── statSync(filePath) → 验证文件存在
├── classifyGuiEditor(editor) → 判断编辑器类型
├── 根据类型处理终端
│   ├── GUI 编辑器 → inkInstance.pause() + suspendStdin()
│   └── 终端编辑器 → inkInstance.enterAlternateScreen()
├── execSync_DEPRECATED(`${editorCommand} "${filePath}"`)
├── readFileSync(filePath) → 读取编辑后内容
└── 恢复终端状态
```

#### 2. 提示编辑流程

```
editPromptInEditor(currentPrompt, pastedContents)
├── generateTempFilePath() → 生成临时文件路径
├── 展开粘贴内容引用（如果有）
├── writeFileSync_DEPRECATED(tempFile, expandedPrompt)
├── editFileInEditor(tempFile) → 调用文件编辑
├── 处理编辑结果
│   ├── 移除尾部单个换行符
│   └── 重新折叠粘贴内容
└── 清理临时文件
```

#### 3. 粘贴内容重新折叠

```typescript
function recollapsePastedContent(
  editedPrompt: string,
  originalPrompt: string,
  pastedContents: Record<number, PastedContent>,
): string
```

**逻辑**：
- 遍历所有粘贴内容
- 在编辑后的文本中查找完整内容
- 替换为引用格式 `[Pasted text 1: N lines]`

### Ink 终端管理

```typescript
if (useAlternateScreen) {
  // 终端编辑器：进入交替屏幕
  inkInstance.enterAlternateScreen()
} else {
  // GUI 编辑器：暂停 Ink
  inkInstance.pause()
  inkInstance.suspendStdin()
}

// ... 编辑器执行 ...

if (useAlternateScreen) {
  inkInstance.exitAlternateScreen()
} else {
  inkInstance.resumeStdin()
  inkInstance.resume()
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `../history.js` | 粘贴内容展开/格式化 |
| `../ink/instances.js` | Ink 实例管理 |
| `./config.js` | 粘贴内容类型 |
| `./editor.js` | 编辑器分类和获取 |
| `./execSyncWrapper.js` | 同步执行包装 |
| `./fsOperations.js` | 文件系统抽象 |
| `./ide.js` | IDE 显示名称 |
| `./slowOperations.js` | 同步文件写入 |
| `./tempfile.js` | 临时文件生成 |

### 外部调用方

| 调用方 | 用途 |
|-------|------|
| `src/commands/memory/memory.tsx` | 编辑记忆 |
| `src/commands/plan/plan.tsx` | 编辑计划 |
| `src/commands/keybindings/keybindings.ts` | 编辑快捷键 |
| `src/components/PromptInput/PromptInput.tsx` | 编辑提示 |
| `src/components/agents/AgentEditor.tsx` | 编辑 Agent |
| `src/components/agents/new-agent-creation/wizard-steps/ConfirmStepWrapper.tsx` | Agent 创建确认 |
| `src/components/agents/new-agent-creation/wizard-steps/PromptStep.tsx` | Agent 提示编辑 |
| `src/components/agents/new-agent-creation/wizard-steps/DescriptionStep.tsx` | Agent 描述编辑 |
| `src/components/agents/new-agent-creation/wizard-steps/GenerateStep.tsx` | Agent 生成编辑 |
| `src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx` | 退出计划模式 |
| `src/components/permissions/AskUserQuestionPermissionRequest/PreviewQuestionView.tsx` | 预览问题 |
| `src/components/permissions/AskUserQuestionPermissionRequest/QuestionView.tsx` | 编辑问题 |

---

## 依赖与外部交互

### 运行时依赖

1. **内部模块**：
   - Ink 实例管理
   - 编辑器配置
   - 文件系统抽象
   - 历史记录管理

### 编辑器环境变量

| 变量 | 优先级 |
|-----|-------|
| `VISUAL` | 最高 |
| `EDITOR` | 次之 |
| 默认编辑器 | 最低 |

### 支持的编辑器

**GUI 编辑器**：
- VS Code (`code`)
- Cursor (`cursor`)
- Windsurf (`windsurf`)
- VSCodium (`codium`)
- Sublime Text (`subl`)

**终端编辑器**：
- vi/vim/nvim
- nano
- emacs
- pico
- micro
- helix/hx

---

## 风险、边界与改进建议

### 已知风险

1. **命令注入**
   - 编辑器命令直接拼接文件路径
   - 恶意文件名可能导致命令注入

2. **临时文件清理**
   - 使用 `try...finally` 清理
   - 但进程异常退出可能泄漏临时文件

3. **编码问题**
   - 固定使用 UTF-8 编码
   - 其他编码的文件可能损坏

4. **并发编辑**
   - 无文件锁机制
   - 多进程同时编辑同一文件可能冲突

### 边界条件

| 场景 | 处理 |
|------|------|
| 无编辑器配置 | 返回 `{ content: null }` |
| 文件不存在 | 返回 `{ content: null }` |
| 编辑器退出码非 0 | 返回错误信息 |
| 编辑后内容为空 | 正常返回空字符串 |
| 粘贴内容被修改 | 不重新折叠（内容不匹配） |

### 改进建议

1. **命令注入防护**
   ```typescript
   // 使用参数数组而非字符串拼接
   execFileSync(editor, [filePath], { stdio: 'inherit' })
   ```

2. **临时文件管理**
   ```typescript
   // 使用临时文件管理器
   import { withTempFile } from './tempfile.js'
   await withTempFile(async (path) => { ... })
   ```

3. **编码检测**
   ```typescript
   // 自动检测文件编码
   import { detectEncoding } from './encoding.js'
   const encoding = await detectEncoding(filePath)
   ```

4. **文件锁**
   ```typescript
   // 防止并发编辑
   import { lock } from 'proper-lockfile'
   await lock(filePath)
   ```

5. **编辑器配置扩展**
   ```typescript
   // 支持更多编辑器参数
   interface EditorConfig {
     command: string
     args: string[]
     waitFlag: string
     gui: boolean
   }
   ```

### 维护注意事项

1. **编辑器兼容性**：测试新编辑器版本的兼容性
2. **安全审计**：定期审查命令执行逻辑
3. **临时文件监控**：监控临时目录大小
4. **跨平台测试**：确保 Windows、macOS、Linux 行为一致

### 使用示例

```typescript
import { editFileInEditor, editPromptInEditor } from './promptEditor.js'

// 编辑文件
const result = editFileInEditor('/path/to/file.txt')
if (result.content !== null) {
  console.log('Edited content:', result.content)
} else if (result.error) {
  console.error('Editor error:', result.error)
}

// 编辑提示（带粘贴内容）
const pastedContents = {
  1: { type: 'text', content: '粘贴的大段文本...' }
}
const promptResult = editPromptInEditor(
  '请检查这段代码：[Pasted text 1: 100 lines]',
  pastedContents
)
if (promptResult.content !== null) {
  console.log('Edited prompt:', promptResult.content)
}
```
