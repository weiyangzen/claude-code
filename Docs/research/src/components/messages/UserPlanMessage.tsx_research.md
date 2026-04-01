# UserPlanMessage.tsx 研究文档

## 场景与职责

`UserPlanMessage` 是一个 React 组件，用于渲染用户提交的计划消息。在 Plan Mode（计划模式）中，用户输入实现计划，该组件以带边框的卡片形式展示计划内容。

**核心职责：**
- 以视觉突出的卡片形式显示计划内容
- 使用 Markdown 渲染计划文本
- 提供 "Plan to implement" 标题标识
- 使用计划模式主题颜色进行样式区分

## 功能点目的

1. **计划展示**：将计划内容以结构化方式呈现
2. **视觉区分**：使用边框和主题色区分计划消息与普通消息
3. **Markdown 支持**：计划内容支持 Markdown 格式
4. **模式标识**：明确标识这是计划实现消息

## 具体技术实现

### 关键流程

```
输入: addMargin (boolean), planContent (string)
  ↓
渲染带圆角边框的容器
  ↓
显示标题 "Plan to implement"
  ↓
使用 Markdown 组件渲染 planContent
```

### 数据结构

**Props 接口：**
```typescript
{
  addMargin: boolean   // 是否在顶部添加边距
  planContent: string  // 计划内容（支持 Markdown）
}
```

### 关键代码路径

**文件位置：** `src/components/messages/UserPlanMessage.tsx`

**渲染结构（源码）：**
```tsx
<Box
  flexDirection="column"
  borderStyle="round"
  borderColor="planMode"
  marginTop={addMargin ? 1 : 0}
  paddingX={1}
>
  <Box marginBottom={1}>
    <Text bold color="planMode">
      Plan to implement
    </Text>
  </Box>
  <Markdown>{planContent}</Markdown>
</Box>
```

**样式属性：**
- `borderStyle="round"` - 圆角边框
- `borderColor="planMode"` - 计划模式主题色边框
- `paddingX={1}` - 水平内边距
- `marginTop={addMargin ? 1 : 0}` - 可选顶部边距

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| React | 'react' | UI 框架 |
| Box, Text | '../../ink.js' | 终端 UI 组件 |
| Markdown | '../Markdown.js' | Markdown 渲染组件 |

### 主题颜色

**计划模式相关颜色：**
- `planMode` - 边框和标题颜色

### 相关命令

与 `/plan` 命令相关：
- 用户通过 `/plan` 进入计划模式
- 在计划模式中输入多行文本作为实现计划
- 提交后由 `UserPlanMessage` 渲染显示

### Markdown 组件

**Markdown.tsx：**
- 支持标准 Markdown 语法
- 代码块高亮
- 表格渲染
- 列表和标题

## 风险、边界与改进建议

### 潜在风险

1. **Markdown 解析错误**：如果 planContent 包含格式错误的 Markdown，可能导致渲染问题
2. **长计划内容**：没有截断或折叠机制，长计划可能占用大量屏幕空间
3. **嵌套边框**：如果计划内容包含其他带边框的元素，可能导致视觉混乱

### 边界情况

1. **空计划内容**：渲染空的 Markdown 区域
2. **纯空格内容**：显示边框和标题，内容区域为空
3. **超长单行**：没有自动换行处理（依赖 Ink 的文本包装）
4. **特殊 Markdown**：复杂表格或代码块可能影响布局

### 改进建议

1. **内容验证**：验证 planContent 非空
2. **长度限制**：添加计划内容长度限制或折叠功能
3. **编辑功能**：支持点击计划消息进行编辑
4. **版本对比**：显示计划的版本历史
5. **任务列表**：特殊处理 Markdown 任务列表，显示完成进度
6. **导出功能**：支持将计划导出为文件
7. **协作标记**：在团队环境中显示计划作者

### 测试建议

1. 各种 Markdown 内容测试
2. 空内容和边界测试
3. 长内容渲染性能测试
4. 特殊字符和编码测试
5. 主题颜色兼容性测试
