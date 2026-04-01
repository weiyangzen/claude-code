# LSPTool.ts 研究文档

## 场景与职责

LSPTool.ts 是 Claude Code 中 LSP（Language Server Protocol）工具的核心实现文件，负责为 AI 模型提供代码智能功能。该工具使 Claude 能够利用语言服务器进行代码分析、导航和理解，是 IDE 级代码智能能力的桥梁。

**主要职责：**
- 提供 9 种 LSP 操作：跳转到定义、查找引用、悬停提示、文档符号、工作区符号、跳转到实现、调用层次结构准备、入站调用、出站调用
- 管理与 LSP 服务器的通信和状态同步
- 处理文件打开/关闭通知以维护 LSP 服务器状态
- 过滤 gitignore 文件的结果
- 格式化 LSP 响应为可读的文本输出

**使用场景：**
- 用户询问"这个函数在哪里定义"
- 需要查找某个符号的所有引用
- 获取类型信息或文档注释
- 分析代码结构和调用关系

---

## 功能点目的

### 1. LSP 操作支持（9种）

| 操作 | LSP 方法 | 用途 |
|------|----------|------|
| goToDefinition | textDocument/definition | 查找符号定义位置 |
| findReferences | textDocument/references | 查找所有引用 |
| hover | textDocument/hover | 获取类型信息和文档 |
| documentSymbol | textDocument/documentSymbol | 获取文件内所有符号 |
| workspaceSymbol | workspace/symbol | 工作区符号搜索 |
| goToImplementation | textDocument/implementation | 查找接口实现 |
| prepareCallHierarchy | textDocument/prepareCallHierarchy | 准备调用层次 |
| incomingCalls | callHierarchy/incomingCalls | 查找调用者 |
| outgoingCalls | callHierarchy/outgoingCalls | 查找被调用者 |

### 2. 文件大小限制
- 最大支持 10MB 文件（`MAX_LSP_FILE_SIZE_BYTES = 10_000_000`）
- 超过限制返回友好错误信息

### 3. 安全机制
- UNC 路径跳过（防止 NTLM 凭证泄漏）
- 文件存在性和类型验证
- 权限检查集成

### 4. gitignore 过滤
- 使用 `git check-ignore` 批量检查
- 每批 50 个文件，避免命令行过长
- 自动从结果中排除被忽略的文件

---

## 具体技术实现

### 关键数据结构

```typescript
// 输入参数结构
interface Input {
  operation: 'goToDefinition' | 'findReferences' | ... | 'outgoingCalls'
  filePath: string
  line: number      // 1-based, 用户友好
  character: number // 1-based, 用户友好
}

// 输出结果结构
interface Output {
  operation: string
  result: string    // 格式化后的文本结果
  filePath: string
  resultCount?: number  // 结果数量统计
  fileCount?: number    // 涉及文件数统计
}
```

### 核心流程

#### 1. 工具调用流程（call 方法）

```
call(input, context)
  ├── 等待 LSP 管理器初始化（如需要）
  ├── 获取 LSP 服务器管理器
  ├── 映射操作到 LSP 方法和参数
  ├── 确保文件已在 LSP 服务器打开
  │   └── 如未打开：读取文件内容 → 发送 didOpen 通知
  ├── 发送 LSP 请求
  ├── 特殊处理：incomingCalls/outgoingCalls 需要两步
  │   └── 先 prepareCallHierarchy → 再发送 calls 请求
  ├── 过滤 gitignore 文件（如适用）
  └── 格式化结果并返回
```

#### 2. 坐标转换

```typescript
// 用户输入是 1-based（编辑器显示）
// LSP 协议要求 0-based
const position = {
  line: input.line - 1,
  character: input.character - 1
}
```

#### 3. gitignore 过滤实现

```typescript
async function filterGitIgnoredLocations<T extends Location>(
  locations: T[],
  cwd: string
): Promise<T[]> {
  // 1. 收集唯一文件路径
  // 2. 批量执行 git check-ignore（每批50个）
  // 3. 过滤掉被忽略的文件
  // 4. 返回过滤后的结果
}
```

### 关键代码路径

#### 操作映射（getMethodAndParams 函数）

```typescript
function getMethodAndParams(input: Input, absolutePath: string) {
  const uri = pathToFileURL(absolutePath).href
  const position = { line: input.line - 1, character: input.character - 1 }
  
  switch (input.operation) {
    case 'goToDefinition':
      return {
        method: 'textDocument/definition',
        params: { textDocument: { uri }, position }
      }
    // ... 其他操作
  }
}
```

#### 调用层次特殊处理

```typescript
if (input.operation === 'incomingCalls' || input.operation === 'outgoingCalls') {
  // 第一步：获取 CallHierarchyItem
  const callItems = result as CallHierarchyItem[]
  
  // 第二步：使用第一个 item 请求实际调用
  const callMethod = input.operation === 'incomingCalls' 
    ? 'callHierarchy/incomingCalls' 
    : 'callHierarchy/outgoingCalls'
  
  result = await manager.sendRequest(absolutePath, callMethod, {
    item: callItems[0]
  })
}
```

#### 结果格式化分发

```typescript
function formatResult(operation, result, cwd) {
  switch (operation) {
    case 'goToDefinition':
      return formatGoToDefinitionResult(result, cwd)
    case 'findReferences':
      return formatFindReferencesResult(result, cwd)
    // ... 其他操作
  }
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `./formatters.js` | 格式化各种 LSP 结果为可读文本 |
| `./schemas.js` | Zod 输入验证 schema（discriminated union） |
| `./prompt.js` | 工具描述文本 |
| `./UI.js` | React 组件渲染工具使用和结果消息 |
| `./symbolContext.js` | 从文件位置提取符号名称 |
| `../../services/lsp/manager.js` | LSP 服务器管理器（单例） |
| `../../Tool.js` | Tool 类型定义和 buildTool 辅助函数 |

### 工具定义（buildTool）

```typescript
export const LSPTool = buildTool({
  name: LSP_TOOL_NAME,           // 'LSP'
  searchHint: 'code intelligence (definitions, references, symbols, hover)',
  maxResultSizeChars: 100_000,
  isLsp: true,                   // 标记为 LSP 工具
  shouldDefer: true,             // 需要 ToolSearch 延迟加载
  
  isEnabled() {
    return isLspConnected()      // 检查是否有可用 LSP 服务器
  },
  
  isConcurrencySafe() {
    return true                  // 支持并发调用
  },
  
  isReadOnly() {
    return true                  // 只读操作
  },
  
  // 验证输入
  async validateInput(input: Input): Promise<ValidationResult> {
    // 1. 使用 discriminated union schema 验证
    // 2. 检查文件存在性和类型
    // 3. 跳过 UNC 路径安全检查
  },
  
  // 检查权限
  async checkPermissions(input, context) {
    return checkReadPermissionForTool(LSPTool, input, appState.toolPermissionContext)
  },
  
  // 核心调用逻辑
  async call(input: Input, _context) {
    // ... 见上文流程
  }
})
```

---

## 依赖与外部交互

### LSP 服务层（src/services/lsp/）

```
manager.ts              # LSP 管理器单例和生命周期
  ├── LSPServerManager.ts  # 多服务器管理和路由
  │     ├── LSPServerInstance.ts  # 单个服务器实例
  │     │     └── LSPClient.ts    # JSON-RPC 通信
  │     └── config.ts             # 服务器配置加载
  └── passiveFeedback.ts    # 诊断通知处理
```

### 关键外部函数

| 函数 | 来源 | 用途 |
|------|------|------|
| `getLspServerManager()` | manager.ts | 获取管理器实例 |
| `isLspConnected()` | manager.ts | 检查 LSP 是否可用 |
| `waitForInitialization()` | manager.ts | 等待初始化完成 |
| `manager.sendRequest()` | LSPServerManager.ts | 发送 LSP 请求 |
| `manager.openFile()` | LSPServerManager.ts | 同步文件打开 |
| `manager.isFileOpen()` | LSPServerManager.ts | 检查文件状态 |

### 工具系统集成

- **Tool.ts**: 基础工具接口和 `buildTool` 辅助函数
- **权限系统**: `checkReadPermissionForTool` 集成
- **UI 渲染**: 通过 `UI.tsx` 提供 React 组件

---

## 风险、边界与改进建议

### 已知风险

1. **LSP 服务器不可用**
   - 风险：某些文件类型没有配置 LSP 服务器
   - 处理：返回友好错误信息 "No LSP server available for file type: .ext"

2. **文件过大**
   - 风险：大文件导致内存和性能问题
   - 处理：10MB 限制，超过返回错误

3. **UNC 路径安全**
   - 风险：Windows UNC 路径可能导致 NTLM 凭证泄漏
   - 处理：跳过文件系统操作，直接返回成功

4. **LSP 服务器崩溃**
   - 风险：服务器崩溃后状态不一致
   - 处理：LSPServerInstance 有崩溃恢复和重启限制

5. **Content Modified 错误**
   - 风险：服务器索引中返回临时错误
   - 处理：LSPServerInstance 自动重试（指数退避）

### 边界情况

| 场景 | 行为 |
|------|------|
| 文件不存在 | validateInput 返回错误码 1 |
| 路径不是文件 | validateInput 返回错误码 2 |
| 输入验证失败 | 返回错误码 3 |
| 文件系统错误 | 返回错误码 4 |
| 无 LSP 服务器 | 返回 "No LSP server available" |
| 空结果 | 返回友好说明（如 "No definition found..."） |
| 无效位置 | 返回空结果或错误 |

### 改进建议

1. **缓存优化**
   - 考虑缓存 documentSymbol 结果以减少重复请求
   - 实现智能预加载常用文件

2. **错误恢复**
   - 添加 LSP 服务器自动重启后的请求重试
   - 改进部分失败场景的错误信息

3. **性能优化**
   - 大文件处理可考虑分块读取
   - git check-ignore 可进一步优化批量大小

4. **功能扩展**
   - 支持代码操作（code actions）
   - 支持重命名符号
   - 支持代码补全建议

5. **监控增强**
   - 添加 LSP 请求延迟指标
   - 跟踪各服务器类型的成功率

### 测试注意事项

- 需要模拟 LSP 服务器进行单元测试
- 测试各种文件类型和边界情况
- 验证 gitignore 过滤逻辑
- 测试大文件处理
- 测试网络/服务器故障场景
