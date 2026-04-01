# BriefTool.ts 深度研究文档

## 场景与职责

BriefTool（正式名称为 `SendUserMessage`，曾用名 `Brief`）是 Claude Code CLI 中**最核心的用户消息输出通道**。它是模型向用户发送可读消息的主要工具，承载着以下关键职责：

1. **用户可见消息输出**：作为模型与用户沟通的主要界面，所有需要用户阅读的内容（回复、状态更新、结果通知）都通过此工具发送
2. **附件支持**：支持发送图片、截图、差异文件、日志文件等附件，增强消息的信息承载能力
3. **Proactive 消息**：支持"主动式"消息（`status: 'proactive'`），用于后台任务完成、阻塞通知等场景
4. **功能门控（Feature Gating）**：通过 GrowthBook 和构建时特性标志控制工具的可用性

该工具是 **Kairos 项目**（助手模式）的核心依赖，也是 `--brief` / `--tools` 模式下的主要输出方式。

## 功能点目的

### 1. 工具定义与命名
- **主名称**：`SendUserMessage` - 明确表达工具用途
- **别名**：`Brief` - 向后兼容旧版本
- **用户可见名称**：返回空字符串 `''`，使 UI 渲染时隐藏工具名称，呈现更自然的对话效果

### 2. 输入/输出 Schema

**输入参数**：
```typescript
{
  message: string;           // 支持 Markdown 格式的消息内容
  attachments?: string[];    // 可选文件路径（绝对路径或相对于 cwd）
  status: 'normal' | 'proactive';  // 消息意图标识
}
```

**输出结构**：
```typescript
{
  message: string;
  attachments?: {
    path: string;
    size: number;
    isImage: boolean;
    file_uuid?: string;      // 上传后返回的 UUID（用于 Web 预览）
  }[];
  sentAt?: string;           // ISO 时间戳
}
```

### 3. 功能门控体系

工具实现了**双重门控**机制：

#### isBriefEntitled() - 资格检查
决定用户**是否被允许**使用 Brief 功能：
- 构建时标志：`feature('KAIROS')` 或 `feature('KAIROS_BRIEF')`
- 运行时标志：`getKairosActive()`（助手模式激活）
- 环境变量覆盖：`CLAUDE_CODE_BRIEF`（开发/测试用途）
- GrowthBook 标志：`tengu_kairos_brief`（远程功能开关，5分钟缓存）

#### isBriefEnabled() - 激活检查
决定工具**当前是否激活**：
- 必须满足 `isBriefEntitled()`
- 用户明确 opt-in：`getUserMsgOptIn()`（通过 `--brief`、`defaultView: 'chat'`、`/brief` 命令等）
- 或处于助手模式：`getKairosActive()`

### 4. 附件处理流程

1. **验证阶段**（`validateInput`）：调用 `validateAttachmentPaths` 检查文件存在性和可访问性
2. **解析阶段**（`call`）：调用 `resolveAttachments` 获取文件元数据并可选上传
3. **上传条件**：仅在 `BRIDGE_MODE` 构建且 `replBridgeEnabled` 或 `CLAUDE_CODE_BRIEF_UPLOAD` 环境变量设置时上传

## 具体技术实现

### 关键流程

#### 工具调用流程
```
call({ message, attachments, status }, context)
  ├── 记录分析事件：tengu_brief_send
  ├── 若无附件：直接返回 { message, sentAt }
  └── 若有附件：
      ├── 获取 appState
      ├── resolveAttachments(attachments, { replBridgeEnabled, signal })
      └── 返回 { message, attachments: resolved, sentAt }
```

#### 权限检查
- `isConcurrencySafe`: 返回 `true`，支持并发执行
- `isReadOnly`: 返回 `true`，不修改系统状态
- `checkPermissions`: 使用默认实现（`buildTool` 提供），始终允许

### 数据结构

#### 工具定义对象
```typescript
buildTool({
  name: BRIEF_TOOL_NAME,           // 'SendUserMessage'
  aliases: [LEGACY_BRIEF_TOOL_NAME], // ['Brief']
  searchHint: 'send a message to the user...',
  maxResultSizeChars: 100_000,     // 结果大小限制
  userFacingName: () => '',        // 隐藏工具名称
  inputSchema: lazySchema(...),    // 延迟加载的 Zod schema
  outputSchema: lazySchema(...),
  isEnabled: isBriefEnabled,       // 动态启用检查
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  validateInput: async ({ attachments }, _context) => {...},
  description: async () => DESCRIPTION,
  prompt: async () => BRIEF_TOOL_PROMPT,
  mapToolResultToToolResultBlockParam: (output, toolUseID) => {...},
  renderToolUseMessage,            // 来自 UI.tsx
  renderToolResultMessage,         // 来自 UI.tsx
  call: async ({ message, attachments, status }, context) => {...},
})
```

### 协议与命令

#### 分析事件
- **事件名**：`tengu_brief_send`
- **元数据**：
  - `proactive: boolean` - 是否为主动消息
  - `attachment_count: number` - 附件数量

#### 工具结果映射
将工具输出映射为 Anthropic API 的 `tool_result` 块：
```typescript
{
  tool_use_id: toolUseID,
  type: 'tool_result',
  content: `Message delivered to user.${suffix}`,  // 包含附件数量信息
}
```

## 关键代码路径与文件引用

### 核心文件
| 文件 | 职责 |
|------|------|
| `BriefTool.ts` | 工具定义、门控逻辑、调用实现 |
| `prompt.ts` | 工具名称常量、描述文本、系统提示片段 |
| `UI.tsx` | React 渲染组件（工具使用和结果消息） |
| `attachments.ts` | 附件验证和解析逻辑 |
| `upload.ts` | 附件上传到私有 API（用于 Web 预览） |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/Tool.ts` | `buildTool` 工厂函数、`ToolDef` 类型、`ValidationResult` 类型 |
| `src/bootstrap/state.ts` | `getKairosActive()`, `getUserMsgOptIn()` |
| `src/services/analytics/growthbook.ts` | `getFeatureValue_CACHED_WITH_REFRESH()` |
| `src/services/analytics/index.ts` | `logEvent()` |
| `src/utils/lazySchema.ts` | 延迟 schema 加载 |
| `src/utils/stringUtils.ts` | `plural()` 函数 |
| `src/utils/envUtils.ts` | `isEnvTruthy()` |

### 调用方
- `main.tsx`：初始化时设置 `userMsgOptIn`，调用 `maybeActivateBrief()`
- `brief.ts`：`/brief` 斜杠命令处理
- `Config.tsx`：`/config` 中的 `defaultView` 选择器
- 系统提示生成：包含 `BRIEF_PROACTIVE_SECTION`

## 依赖与外部交互

### 构建时特性标志（bun:bundle）
- `KAIROS`：助手模式主标志
- `KAIROS_BRIEF`：Brief 工具独立标志
- `BRIDGE_MODE`：附件上传功能标志

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_BRIEF` | 强制启用 Brief 功能（开发/测试） |
| `CLAUDE_CODE_BRIEF_UPLOAD` | 允许非 TTY 宿主上传附件 |

### GrowthBook 功能标志
- `tengu_kairos_brief`：远程控制 Brief 功能可用性

### 外部 API
- **私有 API 上传**：`/api/oauth/file_upload`（用于 Web 查看器预览附件）

## 风险、边界与改进建议

### 风险点

1. **功能门控复杂性**
   - 双重门控（entitled + enabled）逻辑复杂，容易混淆
   - GrowthBook 缓存 5 分钟，可能导致功能状态切换延迟
   - 建议：添加更详细的调试日志，简化门控逻辑

2. **附件上传失败静默处理**
   - 上传失败时返回 `undefined`，不会向用户显示错误
   - 本地终端仍可正常显示，但 Web 查看器无法预览
   - 建议：区分"可恢复失败"和"需要通知用户的失败"

3. **TOCTOU 竞争条件**
   - `validateInput` 和 `call` 之间文件可能被修改或删除
   - 代码注释已说明此问题，错误会传播给模型
   - 建议：在关键路径添加重试逻辑

4. **会话恢复兼容性**
   - `attachments` 和 `sentAt` 字段设计为可选，以兼容恢复会话
   - 但旧版本可能无法正确解析新字段
   - 建议：添加版本兼容性测试

### 边界情况

1. **大文件处理**
   - 单个附件上限 30MB（与后端限制一致）
   - 超过限制时静默跳过上传

2. **路径解析**
   - 支持绝对路径和相对于 cwd 的路径
   - 使用 `expandPath()` 处理 `~` 展开和平台差异

3. **并发安全**
   - 标记为并发安全，但附件上传是并行执行的
   - 大量附件可能导致内存压力

### 改进建议

1. **性能优化**
   - 附件验证和 stat 操作目前是串行的，可考虑并行化
   - 添加附件大小预检查，避免读取大文件

2. **可观测性**
   - 添加上传成功率指标
   - 记录附件类型分布（图片 vs 文件）

3. **用户体验**
   - 上传失败时向用户提供反馈选项
   - 支持附件压缩或缩略图生成

4. **代码结构**
   - `isBriefEntitled()` 和 `isBriefEnabled()` 的职责边界可更清晰
   - 考虑将门控逻辑抽取到独立模块
