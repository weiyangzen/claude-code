# ErrorOverview.tsx 研究文档

## 场景与职责

`ErrorOverview` 是 Ink 框架中的错误展示组件，用于在应用发生错误时显示友好的错误信息。它是 React Error Boundary 的 fallback UI，提供：

1. **错误信息展示**: 显示错误消息和 ERROR 标签
2. **源代码上下文**: 解析错误堆栈，读取并展示出错位置的源代码片段
3. **堆栈跟踪**: 格式化展示完整的调用堆栈
4. **文件定位**: 显示出错的文件路径、行号和列号

ErrorOverview 在 App.tsx 中作为 Error Boundary 的 fallback 使用，当组件树中的任何组件抛出错误时显示。

## 功能点目的

### 1. 错误信息展示
- **ERROR 标签**: 红色背景白色文字的高可见度标签
- **错误消息**: 直接显示 error.message
- **视觉层次**: 使用 Box 和 Text 组件构建清晰的视觉层次

### 2. 源代码上下文
- **堆栈解析**: 使用 stack-utils 解析错误堆栈
- **文件读取**: 使用 code-excerpt 读取出错位置的源代码
- **代码高亮**: 高亮显示出错的行，其他行使用 dim 样式
- **行号对齐**: 动态计算行号宽度，保持对齐

### 3. 堆栈跟踪展示
- **函数名**: 粗体显示函数名
- **文件位置**: 灰色显示文件路径、行号、列号
- **不可解析行**: 对于无法解析的堆栈行，原样显示

### 4. 路径处理
- **file:// 协议移除**: 清理堆栈中的 file:// 前缀
- **cwd 相对化**: 将绝对路径转换为相对于 cwd 的路径

## 具体技术实现

### 关键数据结构

```typescript
type Props = {
  readonly error: Error;
};

// 堆栈解析结果（来自 stack-utils）
interface StackLine {
  function?: string;
  file?: string;
  line?: number;
  column?: number;
}

// 代码片段（来自 code-excerpt）
interface CodeExcerpt {
  line: number;
  value: string;
}
```

### 关键流程

1. **堆栈解析**
   ```typescript
   const stack = error.stack ? error.stack.split('\n').slice(1) : undefined;
   const origin = stack ? getStackUtils().parseLine(stack[0]!) : undefined;
   const filePath = cleanupPath(origin?.file);
   ```

2. **源代码读取**
   ```typescript
   if (filePath && origin?.line) {
     try {
       const sourceCode = readFileSync(filePath, 'utf8');
       excerpt = codeExcerpt(sourceCode, origin.line);
       // 计算行号宽度用于对齐
       if (excerpt) {
         for (const { line } of excerpt) {
           lineWidth = Math.max(lineWidth, String(line).length);
         }
       }
     } catch {
       // 文件不可读 - 跳过源代码上下文
     }
   }
   ```

3. **渲染结构**
   ```tsx
   <Box flexDirection="column" padding={1}>
     {/* 错误标题 */}
     <Box>
       <Text backgroundColor="ansi:red" color="ansi:white"> ERROR </Text>
       <Text> {error.message}</Text>
     </Box>
     
     {/* 文件位置 */}
     {origin && filePath && <Box marginTop={1}>
       <Text dim>{filePath}:{origin.line}:{origin.column}</Text>
     </Box>}
     
     {/* 源代码片段 */}
     {origin && excerpt && <Box marginTop={1} flexDirection="column">
       {excerpt.map(({ line, value }) => (
         <Box key={line}>
           <Box width={lineWidth + 1}>
             <Text dim={line !== origin.line} 
                   backgroundColor={line === origin.line ? 'ansi:red' : undefined}
                   color={line === origin.line ? 'ansi:white' : undefined}>
               {String(line).padStart(lineWidth, ' ')}:
             </Text>
           </Box>
           <Text backgroundColor={line === origin.line ? 'ansi:red' : undefined}
                 color={line === origin.line ? 'ansi:white' : undefined}>
             {' ' + value}
           </Text>
         </Box>
       ))}
     </Box>}
     
     {/* 完整堆栈 */}
     {error.stack && <Box marginTop={1} flexDirection="column">
       {error.stack.split('\n').slice(1).map(line => {
         const parsed = getStackUtils().parseLine(line);
         if (!parsed) {
           return <Box key={line}>
             <Text dim>- </Text>
             <Text bold>{line}</Text>
           </Box>;
         }
         return <Box key={line}>
           <Text dim>- </Text>
           <Text bold>{parsed.function}</Text>
           <Text dim> ({cleanupPath(parsed.file)}:{parsed.line}:{parsed.column})</Text>
         </Box>;
       })}
     </Box>}
   </Box>
   ```

### 代码路径

```
ErrorOverview.tsx
├── 导入依赖
│   ├── code-excerpt: 提取代码片段
│   ├── fs.readFileSync: 读取源文件
│   ├── stack-utils: 解析堆栈
│   ├── Box: 布局容器
│   └── Text: 文本渲染
├── cleanupPath 函数: 清理 file:// 前缀
├── getStackUtils 函数: 延迟初始化 StackUtils
├── Props 类型定义
├── ErrorOverview 组件
│   ├── 解析堆栈获取 origin
│   ├── 读取源代码片段
│   ├── 计算行号宽度
│   └── 渲染错误信息
└── 默认导出
```

## 关键代码路径与文件引用

### 核心依赖

| 模块/文件 | 用途 |
|-----------|------|
| `code-excerpt` | 从源代码中提取出错位置周围的代码片段 |
| `stack-utils` | 解析 JavaScript 堆栈跟踪 |
| `fs` | 同步读取源文件 |
| `src/ink/components/Box.tsx` | 布局容器 |
| `src/ink/components/Text.tsx` | 文本渲染 |

### 调用方

ErrorOverview 在 `src/ink/components/App.tsx` 中作为 Error Boundary 的 fallback：

```tsx
// App.tsx
class App extends PureComponent<Props, State> {
  static getDerivedStateFromError(error: Error) {
    return { error };
  }
  
  state = { error: undefined };
  
  render() {
    return (
      <CursorDeclarationContext.Provider value={...}>
        {this.state.error 
          ? <ErrorOverview error={this.state.error} />
          : this.props.children
        }
      </CursorDeclarationContext.Provider>
    );
  }
}
```

### 渲染管线

```
React Error Boundary 捕获错误
├── getDerivedStateFromError 设置 state.error
├── 触发重新渲染
├── App.render 检查 state.error
├── 渲染 ErrorOverview 替代 children
└── ErrorOverview
    ├── 解析堆栈
    ├── 读取源代码
    └── 渲染错误 UI
```

## 依赖与外部交互

### 运行时依赖

1. **code-excerpt**: 提取代码片段，支持多种语言
2. **stack-utils**: 解析堆栈跟踪，支持多种 Node.js 版本
3. **fs**: 同步文件读取（渲染路径不能异步）

### 路径处理

```typescript
// 错误堆栈中的路径格式：file:///home/user/project/file.js
// 需要转换为：/home/user/project/file.js 或相对路径

const cleanupPath = (path: string | undefined): string | undefined => {
  return path?.replace(`file://${process.cwd()}/`, '');
};
```

### StackUtils 配置

```typescript
const getStackUtils = (): StackUtils => {
  return stackUtils ??= new StackUtils({
    cwd: process.cwd(),
    internals: StackUtils.nodeInternals(),  // 排除 Node.js 内部帧
  });
};
```

## 风险、边界与改进建议

### 已知风险

1. **同步文件读取**: 使用 `readFileSync` 可能在文件较大时阻塞事件循环
2. **堆栈解析依赖**: stack-utils 的解析可能不适用于所有 JavaScript 引擎或转换后的代码
3. **source map 不支持**: 当前不处理 source map，显示的是编译后的位置

### 边界情况

1. **堆栈不可用**: error.stack 可能不存在，组件优雅处理
2. **文件不可读**: 文件可能被删除或无权限，try-catch 保护
3. **堆栈帧解析失败**: 某些堆栈行可能无法解析，原样显示
4. **无源代码上下文**: 如果无法读取文件，只显示错误消息和堆栈

### 改进建议

1. **Source Map 支持**: 添加 source-map-support，将编译后的位置映射回源代码
   ```typescript
   import { SourceMapConsumer } from 'source-map';
   // 解析原始位置并显示源代码
   ```

2. **异步文件读取**: 考虑使用 Suspense 和异步文件读取（需要重构 Ink 渲染管线）

3. **代码语法高亮**: 集成语法高亮库（如 highlight.js）为代码片段添加语法着色

4. **可折叠堆栈**: 对于深层堆栈，提供折叠/展开功能

5. **错误代码链接**: 对于常见错误类型，提供文档链接

6. **错误上报**: 添加将错误信息发送到错误追踪服务的选项

7. **文件大小限制**: 对大文件添加读取大小限制，避免内存问题
   ```typescript
   const stats = statSync(filePath);
   if (stats.size > MAX_FILE_SIZE) {
     // 跳过或只读取部分内容
   }
   ```

### 代码质量

- 使用了 eslint-disable 注释处理 process.cwd() 使用
- 同步读取有明确的注释说明原因（渲染路径不能异步）
- 错误处理完善，不会因为文件读取失败而崩溃

### 测试建议

- 测试各种错误类型的展示
- 测试文件不可读时的降级行为
- 测试堆栈解析失败的边界情况
- 测试不同 cwd 下的路径处理
