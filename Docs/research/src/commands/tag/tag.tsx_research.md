# 研究文档: src/commands/tag/tag.tsx

## 场景与职责

本文件是 `/tag` 命令的核心实现，采用 React + Ink 技术栈构建交互式终端 UI。该命令实现了会话标签的添加、移除和查询功能，支持以下交互模式：

**主要使用场景：**
1. **添加标签**：`/tag <tag-name>` 为当前会话添加标签
2. **移除标签**：再次执行相同命令触发 toggle 行为，显示确认对话框后移除
3. **查看帮助**：`/tag help` 或空参数显示使用说明
4. **会话识别**：标签显示在 `/resume` 列表的分支名之后，便于快速识别会话

**核心职责：**
- 标签名称的 Unicode 安全处理（防止隐藏字符攻击）
- 标签状态管理（添加/移除/替换）
- 用户确认交互（移除前二次确认）
- 分析事件上报（操作追踪）
- 会话存储集成（持久化标签数据）

## 功能点目的

### 1. 标签管理
- **添加标签**：为新会话或已有标签的会话添加标签
- **移除标签**：通过 toggle 机制移除已有标签
- **替换标签**：自动检测并处理标签替换场景

### 2. 安全防护
- **Unicode 净化**：使用 `recursivelySanitizeUnicode` 防止隐藏字符注入攻击
- **输入验证**：确保标签名称非空且格式合法

### 3. 用户体验
- **确认对话框**：移除标签前显示确认提示，防止误操作
- **实时反馈**：操作成功后显示彩色确认消息
- **帮助文档**：内置完整的使用说明和示例

### 4. 数据持久化
- **JSONL 存储**：标签数据以结构化格式写入会话转录文件
- **内存缓存**：当前会话标签缓存在内存中以提升性能

## 具体技术实现

### 核心组件架构

```
tag.tsx
├── ConfirmRemoveTag    # 标签移除确认对话框组件
├── ToggleTagAndClose   # 标签切换主逻辑组件
├── ShowHelp           # 帮助信息显示组件
└── call               # 命令入口函数
```

### 关键数据结构

#### 标签存储格式（JSONL Entry）
```typescript
{
  type: 'tag',
  tag: string,        // 标签名称
  sessionId: UUID     // 会话 ID
}
```

#### 组件 Props 定义
```typescript
// ConfirmRemoveTag
interface ConfirmRemoveTagProps {
  tagName: string
  onConfirm: () => void
  onCancel: () => void
}

// ToggleTagAndClose
interface ToggleTagAndCloseProps {
  tagName: string
  onDone: LocalJSXCommandOnDone
}

// ShowHelp
interface ShowHelpProps {
  onDone: LocalJSXCommandOnDone
}
```

### 核心流程详解

#### 1. 命令入口 (`call` 函数)

```typescript
export async function call(
  onDone: LocalJSXCommandOnDone,
  _context: unknown,
  args?: string
): Promise<React.ReactNode> {
  args = args?.trim() || ''
  
  // 帮助参数处理
  if (COMMON_INFO_ARGS.includes(args) || COMMON_HELP_ARGS.includes(args)) {
    return <ShowHelp onDone={onDone} />
  }
  
  // 空参数显示帮助
  if (!args) {
    return <ShowHelp onDone={onDone} />
  }
  
  // 执行标签切换
  return <ToggleTagAndClose tagName={args} onDone={onDone} />
}
```

**流程说明：**
1. 参数修剪处理
2. 检测帮助请求（`help`, `-h`, `--help`, `?` 等）
3. 空参数默认显示帮助
4. 有效参数进入标签切换逻辑

#### 2. 标签切换逻辑 (`ToggleTagAndClose` 组件)

```typescript
function ToggleTagAndClose({ tagName, onDone }) {
  const [showConfirm, setShowConfirm] = useState(false)
  const [sessionId, setSessionId] = useState(null)
  
  // Unicode 净化处理
  const normalizedTag = recursivelySanitizeUnicode(tagName).trim()
  
  useEffect(() => {
    const id = getSessionId() as UUID
    if (!id) {
      onDone("No active session to tag", { display: "system" })
      return
    }
    
    if (!normalizedTag) {
      onDone("Tag name cannot be empty", { display: "system" })
      return
    }
    
    setSessionId(id)
    const currentTag = getCurrentSessionTag(id)
    
    if (currentTag === normalizedTag) {
      // 相同标签 → 触发移除确认
      logEvent("tengu_tag_command_remove_prompt", {})
      setShowConfirm(true)
    } else {
      // 新标签或不同标签 → 直接添加
      const isReplacing = !!currentTag
      logEvent("tengu_tag_command_add", { is_replacing: isReplacing })
      
      (async () => {
        const fullPath = getTranscriptPath()
        await saveTag(id, normalizedTag, fullPath)
        onDone(`Tagged session with #${normalizedTag}`, {
          display: "system"
        })
      })()
    }
  }, [normalizedTag, onDone])
  
  // 显示确认对话框
  if (showConfirm && sessionId) {
    // ... 确认处理逻辑
  }
  
  return null
}
```

**状态流转：**
```
初始状态
    ↓
获取 sessionId
    ↓
检查当前标签
    ├─ 相同标签 ─→ 显示确认对话框 ─→ 确认移除 / 取消
    └─ 新标签 ───→ 直接保存 ─────→ 显示成功消息
```

#### 3. 确认对话框 (`ConfirmRemoveTag` 组件)

使用 `Dialog` + `Select` 组件构建确认交互：

```typescript
function ConfirmRemoveTag({ tagName, onConfirm, onCancel }) {
  const handleChange = (value: string) => {
    value === "yes" ? onConfirm() : onCancel()
  }
  
  return (
    <Dialog title="Remove tag?" subtitle={`Current tag: #${tagName}`}>
      <Box flexDirection="column" gap={1}>
        <Text>This will remove the tag from the current session.</Text>
        <Select onChange={handleChange} options={[
          { label: "Yes, remove tag", value: "yes" },
          { label: "No, keep tag", value: "no" }
        ]} />
      </Box>
    </Dialog>
  )
}
```

#### 4. 帮助信息 (`ShowHelp` 组件)

```typescript
function ShowHelp({ onDone }) {
  useEffect(() => {
    onDone(
      "Usage: /tag <tag-name>\n\n" +
      "Toggle a searchable tag on the current session.\n" +
      "Run the same command again to remove the tag.\n" +
      "Tags are displayed after the branch name in /resume and can be searched with /.\n\n" +
      "Examples:\n" +
      "  /tag bugfix        # Add tag\n" +
      "  /tag bugfix        # Remove tag (toggle)\n" +
      "  /tag feature-auth\n" +
      "  /tag wip",
      { display: "system" }
    )
  }, [onDone])
  
  return null
}
```

### 分析事件追踪

| 事件名称 | 触发时机 | 附加数据 |
|---------|---------|---------|
| `tengu_tag_command_remove_prompt` | 用户尝试移除标签时 | - |
| `tengu_tag_command_add` | 添加新标签时 | `is_replacing: boolean` |
| `tengu_tag_command_remove_confirmed` | 确认移除标签时 | - |
| `tengu_tag_command_remove_cancelled` | 取消移除操作时 | - |
| `tengu_session_tagged` | 标签保存成功后（在 `saveTag` 中） | - |

## 关键代码路径与文件引用

### 当前文件
- **路径**: `src/commands/tag/tag.tsx`
- **大小**: ~21KB（编译后包含 React Compiler 优化代码）
- **原始源码**: 约 70 行 TypeScript/TSX

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/commands.js` | `CommandResultDisplay`, `Command` 类型定义 |
| `src/types/command.ts` | `LocalJSXCommandOnDone` 类型定义 |
| `src/bootstrap/state.ts` | `getSessionId` 获取当前会话 ID |
| `src/utils/sessionStorage.ts` | `getCurrentSessionTag`, `getTranscriptPath`, `saveTag` |
| `src/utils/sanitization.ts` | `recursivelySanitizeUnicode` Unicode 净化 |
| `src/services/analytics/index.ts` | `logEvent` 分析事件上报 |
| `src/constants/xml.ts` | `COMMON_HELP_ARGS`, `COMMON_INFO_ARGS` |
| `src/components/CustomSelect/select.tsx` | `Select` 选择组件 |
| `src/components/design-system/Dialog.tsx` | `Dialog` 对话框组件 |
| `src/ink.ts` | `Box`, `Text` Ink UI 组件 |
| `chalk` | 终端颜色输出 |

### 调用关系图

```
tag.tsx
├── imports
│   ├── chalk (颜色输出)
│   ├── React (UI 框架)
│   ├── src/bootstrap/state.ts → getSessionId()
│   ├── src/commands.js → CommandResultDisplay
│   ├── src/components/CustomSelect/select.tsx → Select
│   ├── src/components/design-system/Dialog.tsx → Dialog
│   ├── src/constants/xml.ts → COMMON_HELP_ARGS, COMMON_INFO_ARGS
│   ├── src/ink.ts → Box, Text
│   ├── src/services/analytics/index.ts → logEvent
│   ├── src/types/command.ts → LocalJSXCommandOnDone
│   └── src/utils/sanitization.ts → recursivelySanitizeUnicode
│   └── src/utils/sessionStorage.ts → getCurrentSessionTag, getTranscriptPath, saveTag
│
├── components
│   ├── ConfirmRemoveTag (确认对话框)
│   ├── ToggleTagAndClose (主逻辑)
│   └── ShowHelp (帮助信息)
│
└── exports
    └── call (命令入口)
```

## 依赖与外部交互

### 核心依赖详解

#### 1. 会话状态管理 (`src/bootstrap/state.ts`)
```typescript
import { getSessionId } from '../../bootstrap/state.js'
```
- 获取当前会话的唯一标识符 (UUID)
- 用于验证会话存在性和关联标签数据

#### 2. 会话存储 (`src/utils/sessionStorage.ts`)
```typescript
import {
  getCurrentSessionTag,
  getTranscriptPath,
  saveTag
} from '../../utils/sessionStorage.js'
```

**`getCurrentSessionTag(sessionId)`**: 从内存缓存获取当前标签
**`getTranscriptPath()`**: 获取会话转录文件的完整路径
**`saveTag(sessionId, tag, fullPath)`**: 将标签写入 JSONL 文件并更新缓存

#### 3. Unicode 安全 (`src/utils/sanitization.ts`)
```typescript
import { recursivelySanitizeUnicode } from '../../utils/sanitization.js'
```
- 防御 Unicode 隐藏字符攻击（如 ASCII Smuggling）
- 应用 NFKC 规范化并移除危险字符类别

#### 4. 分析追踪 (`src/services/analytics/index.ts`)
```typescript
import { logEvent } from '../../services/analytics/index.js'
```
- 无依赖设计，事件队列机制
- 支持同步和异步事件上报

#### 5. UI 组件

**Select 组件** (`src/components/CustomSelect/select.tsx`):
- 支持键盘导航的选择列表
- 支持输入模式和选项分组
- 可配置布局和样式

**Dialog 组件** (`src/components/design-system/Dialog.tsx`):
- 模态对话框容器
- 内置键盘快捷键支持（Esc 取消，Enter 确认）
- 支持自定义颜色和输入引导

### 外部系统交互

| 系统 | 交互方式 | 数据 |
|-----|---------|------|
| 文件系统 | 通过 `saveTag` | 写入 JSONL 格式标签数据 |
| 分析后端 | 通过 `logEvent` | 上报操作事件 |
| 终端 UI | 通过 Ink + React | 渲染交互式界面 |

## 风险、边界与改进建议

### 潜在风险

#### 1. 安全风险
- **Unicode 攻击**：虽然使用了 `recursivelySanitizeUnicode`，但仍需持续关注新的 Unicode 攻击向量
- **标签注入**：标签内容直接显示在 UI 中，需确保 HTML/终端转义正确

#### 2. 并发风险
```typescript
// 当前实现：非原子性检查-设置操作
const currentTag = getCurrentSessionTag(id)
if (currentTag === normalizedTag) {
  // 竞态条件：两次快速调用可能导致状态不一致
}
```

#### 3. 持久化风险
- **文件写入失败**：`saveTag` 使用异步写入，失败时用户可能已看到成功消息
- **缓存不一致**：内存缓存与磁盘数据可能短暂不一致

### 边界情况

| 场景 | 当前行为 | 评估 |
|-----|---------|------|
| 空标签输入 | 显示错误消息 "Tag name cannot be empty" | 合理 |
| 无活动会话 | 显示错误消息 "No active session to tag" | 合理 |
| 超长标签 | 无长度限制，直接存储 | 建议增加长度限制 |
| 特殊字符标签 | Unicode 净化后存储 | 安全 |
| 并发标签操作 | 后执行者覆盖前者 | 建议增加乐观锁 |
| 磁盘写入失败 | 用户看到成功消息但数据未保存 | 需要修复 |

### 改进建议

#### 1. 增加标签长度限制
```typescript
const MAX_TAG_LENGTH = 50
if (normalizedTag.length > MAX_TAG_LENGTH) {
  onDone(`Tag name too long (max ${MAX_TAG_LENGTH} characters)`, { display: "system" })
  return
}
```

#### 2. 优化错误处理
```typescript
// 当前实现：无错误处理
await saveTag(id, normalizedTag, fullPath)

// 建议改进：
try {
  await saveTag(id, normalizedTag, fullPath)
  onDone(`Tagged session with #${normalizedTag}`, { display: "system" })
} catch (error) {
  logEvent('tengu_tag_command_error', { error_type: 'save_failed' })
  onDone('Failed to save tag. Please try again.', { display: "system" })
}
```

#### 3. 支持标签列表和验证
```typescript
// 建议：支持查看当前标签
if (args === 'current' || args === 'show') {
  const currentTag = getCurrentSessionTag(getSessionId())
  onDone(currentTag ? `Current tag: #${currentTag}` : 'No tag set', { display: "system" })
  return
}
```

#### 4. 标签历史追踪
- 当前实现：仅保存最新标签
- 建议：保留标签变更历史，支持审计和回滚

#### 5. 性能优化
- 当前 `saveTag` 每次都追加新条目到 JSONL
- 建议：定期压缩或清理历史标签条目

### 测试建议

1. **单元测试**
   - Unicode 净化功能测试（各种隐藏字符）
   - Toggle 逻辑测试（添加→移除→再添加）
   - 边界条件测试（空字符串、超长字符串）

2. **集成测试**
   - 文件系统写入/读取一致性
   - 并发操作行为
   - 会话切换后标签状态

3. **E2E 测试**
   - 完整用户流程：添加→查看→移除
   - 帮助信息显示
   - 确认对话框交互

### 相关 Issue 追踪

- 标签功能目前仅限内部员工（`USER_TYPE === 'ant'`）
- 建议逐步开放给外部用户，需评估安全性和稳定性
