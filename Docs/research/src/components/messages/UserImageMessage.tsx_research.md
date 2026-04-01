# UserImageMessage.tsx 研究文档

## 场景与职责

`UserImageMessage` 是一个 React 组件，用于渲染用户消息中的图片附件。该组件显示图片引用标签，并在终端支持超链接时提供可点击的链接以打开图片。

**核心职责：**
- 根据 imageId 显示图片引用标签（如 `[Image #123]` 或 `[Image]`）
- 如果图片已存储且终端支持超链接，显示为可点击的链接
- 根据 addMargin 参数决定是否与上方消息连接或作为新用户轮次开始
- 使用 `MessageResponse` 组件保持与消息系统的视觉一致性

## 功能点目的

1. **图片引用显示**：显示图片 ID 或通用图片标签
2. **超链接支持**：在支持的终端中提供可点击的图片链接
3. **视觉连接**：使用 MessageResponse 与上方消息保持视觉连接
4. **边距控制**：通过 addMargin 控制是否作为新用户轮次的开始

## 具体技术实现

### 关键流程

```
输入: imageId (number?), addMargin (boolean?)
  ↓
生成 label = imageId ? `[Image #${imageId}]` : "[Image]"
  ↓
获取图片路径 imagePath = imageId ? getStoredImagePath(imageId) : null
  ↓
判断是否支持超链接：imagePath && supportsHyperlinks()
  ↓
如果支持：
  渲染 <Link url={pathToFileURL(imagePath).href}><Text>{label}</Text></Link>
否则：
  渲染 <Text>{label}</Text>
  ↓
如果 addMargin 为 true：
  用 <Box marginTop={1}> 包裹
否则：
  用 <MessageResponse> 包裹（与上方消息连接）
```

### 数据结构

**Props 接口：**
```typescript
{
  imageId?: number    // 图片 ID，用于获取存储路径
  addMargin?: boolean // 是否添加顶部边距
}
```

### 关键代码路径

**文件位置：** `src/components/messages/UserImageMessage.tsx`

**标签生成逻辑：**
```typescript
const label = imageId ? `[Image #${imageId}]` : '[Image]'
```

**超链接判断与渲染：**
```typescript
const imagePath = imageId ? getStoredImagePath(imageId) : null

const content =
  imagePath && supportsHyperlinks() ? (
    <Link url={pathToFileURL(imagePath).href}>
      <Text>{label}</Text>
    </Link>
  ) : (
    <Text>{label}</Text>
  )
```

**边距处理：**
```typescript
if (addMargin) {
  return <Box marginTop={1}>{content}</Box>
}
return <MessageResponse>{content}</MessageResponse>
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| React | 'react' | UI 框架 |
| pathToFileURL | 'url' | 将文件路径转换为 file:// URL |
| Link | '../../ink/components/Link.js' | 超链接组件 |
| supportsHyperlinks | '../../ink/supports-hyperlinks.js' | 检测终端超链接支持 |
| Box, Text | '../../ink.js' | 终端 UI 组件 |
| getStoredImagePath | '../../utils/imageStore.js' | 获取存储图片的路径 |
| MessageResponse | '../MessageResponse.js' | 消息响应包装组件 |

### 超链接支持检测

**supports-hyperlinks.ts：**
```typescript
export function supportsHyperlinks(options?: SupportsHyperlinksOptions): boolean {
  // 检查终端是否支持 OSC 8 超链接
  // 支持：ghostty, Hyper, kitty, alacritty, iTerm.app, iTerm2 等
}
```

**支持的终端列表：**
- ghostty
- Hyper
- kitty
- alacritty
- iTerm.app
- iTerm2
- TERM 包含 'kitty' 的终端

### 图片存储系统

**imageStore.js：**
- `getStoredImagePath(imageId: number): string | null` - 根据 ID 获取图片存储路径

### Link 组件

**Link.tsx：**
- 使用 `<ink-link>` 标签渲染超链接
- 在不支持超链接的终端回退到纯文本显示

## 风险、边界与改进建议

### 潜在风险

1. **图片路径失效**：getStoredImagePath 返回的路径可能已不存在
2. **权限问题**：即使有路径，用户可能没有权限打开图片
3. **终端兼容性**：超链接检测可能不准确，某些支持的终端未被识别

### 边界情况

1. **无 imageId**：显示通用 `[Image]` 标签，无超链接
2. **图片未存储**：getStoredImagePath 返回 null，显示纯文本标签
3. **终端不支持超链接**：显示纯文本标签，用户无法点击
4. **addMargin 未提供**：默认为 false，使用 MessageResponse 连接样式

### 改进建议

1. **图片预览**：在支持的终端中尝试显示图片预览（如 kitty 图形协议）
2. **路径验证**：在生成链接前验证文件是否存在
3. **错误处理**：添加图片打开失败的错误提示
4. **多图片支持**：支持一次显示多个图片附件
5. **图片元数据**：显示图片尺寸、大小等信息
6. **复制功能**：添加复制图片路径的功能

### 测试建议

1. 各种 imageId 情况测试（有/无，有效/无效）
2. 不同终端的超链接支持测试
3. 图片文件存在/不存在场景测试
4. addMargin 参数测试
5. 特殊字符路径测试
