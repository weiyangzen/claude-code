# UserCommandMessage.tsx 研究文档

## 场景与职责

`UserCommandMessage` 是一个 React 组件，用于渲染用户输入的斜杠命令消息。该组件解析命令消息 XML 标签，支持两种格式：普通命令格式（`/<command> <args>`）和 Skill 格式（`Skill(name)`）。

**核心职责：**
- 解析 `<command-message>` 和 `<command-args>` XML 标签
- 检测 Skill 格式（通过 `<skill-format>true</skill-format>`）
- 以不同样式渲染普通命令和 Skill 调用
- 提供统一的命令输入视觉反馈

## 功能点目的

1. **命令解析**：提取命令名称和参数
2. **格式检测**：识别 Skill 格式与普通命令格式
3. **差异化渲染**：
   - Skill 格式：`Skill(commandName)` 样式
   - 普通命令：`/command args` 样式
4. **视觉反馈**：使用指针图标和背景色区分命令消息

## 具体技术实现

### 关键流程

```
输入: addMargin (boolean), param (TextBlockParam)
  ↓
提取 commandMessage = extractTag(text, COMMAND_MESSAGE_TAG)
  ↓
提取 args = extractTag(text, "command-args")
  ↓
检测 isSkillFormat = extractTag(text, "skill-format") === "true"
  ↓
如果无 commandMessage，返回 null
  ↓
如果是 Skill 格式：
  渲染 Skill(commandMessage) 样式
否则：
  组装 content = "/" + [commandMessage, args].filter(Boolean).join(" ")
  渲染 /command args 样式
```

### 数据结构

**Props 接口：**
```typescript
{
  addMargin: boolean      // 是否在顶部添加边距
  param: TextBlockParam   // 包含 text 字段的对象
}

// TextBlockParam 结构
{
  text: string  // 包含命令标签的 XML 文本
}
```

**XML 标签：**
- `command-message` - 命令名称
- `command-args` - 命令参数（可选）
- `skill-format` - 是否为 Skill 格式标记

### 关键代码路径

**文件位置：** `src/components/messages/UserCommandMessage.tsx`

**命令提取与格式检测：**
```typescript
const commandMessage = extractTag(text, COMMAND_MESSAGE_TAG)
const args = extractTag(text, 'command-args')
const isSkillFormat = extractTag(text, 'skill-format') === 'true'

if (!commandMessage) {
  return null
}
```

**Skill 格式渲染：**
```tsx
<Box
  flexDirection="column"
  marginTop={addMargin ? 1 : 0}
  backgroundColor="userMessageBackground"
  paddingRight={1}
>
  <Text>
    <Text color="subtle">{figures.pointer} </Text>
    <Text color="text">Skill({commandMessage})</Text>
  </Text>
</Box>
```

**普通命令渲染：**
```typescript
const content = `/${[commandMessage, args].filter(Boolean).join(' ')}`

<Box
  flexDirection="column"
  marginTop={addMargin ? 1 : 0}
  backgroundColor="userMessageBackground"
  paddingRight={1}
>
  <Text>
    <Text color="subtle">{figures.pointer} </Text>
    <Text color="text">{content}</Text>
  </Text>
</Box>
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| TextBlockParam | '@anthropic-ai/sdk/resources/index.mjs' | SDK 类型定义 |
| figures | 'figures' | 终端图形符号库 |
| React | 'react' | UI 框架 |
| COMMAND_MESSAGE_TAG | '../../constants/xml.js' | 命令消息标签常量 |
| Box, Text | '../../ink.js' | 终端 UI 组件 |
| extractTag | '../../utils/messages.js' | XML 标签提取工具 |

### 相关常量

**xml.js：**
```typescript
export const COMMAND_MESSAGE_TAG = 'command-message'
export const COMMAND_ARGS_TAG = 'command-args'
```

**figures 库：**
- `figures.pointer` - 指针符号（▶ 或类似）

### 样式系统

使用 Ink 的主题颜色：
- `subtle` - 指针颜色
- `text` - 命令文本颜色
- `userMessageBackground` - 用户消息背景色

## 风险、边界与改进建议

### 潜在风险

1. **空命令处理**：如果 command-message 为空字符串，组件返回 null，调用方需要处理
2. **参数注入**：args 直接拼接，如果包含特殊字符可能影响显示
3. **XML 解析依赖**：依赖 extractTag 正确解析，格式错误可能导致提取失败

### 边界情况

1. **无 command-message**：返回 null，不渲染任何内容
2. **空 args**：只显示命令名，不显示额外空格
3. **skill-format 非 true**：按普通命令处理
4. **超长命令**：没有截断处理，可能超出终端宽度

### 改进建议

1. **命令验证**：验证 commandMessage 是否为有效命令格式
2. **参数转义**：处理 args 中的特殊字符（如 XML 实体）
3. **长度限制**：添加命令总长度限制，超长时截断
4. **可点击命令**：支持命令点击重新执行
5. **命令历史**：与命令历史系统集成
6. **类型安全**：为 Skill 格式添加更严格的类型检查

### 测试建议

1. 各种命令格式测试（有/无参数，Skill/普通）
2. 特殊字符参数测试
3. 空命令名边界测试
4. 超长命令显示测试
5. XML 格式变体测试
