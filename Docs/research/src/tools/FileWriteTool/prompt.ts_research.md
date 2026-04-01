# prompt.ts 深度研究文档

## 场景与职责

prompt.ts 是 FileWriteTool 的轻量级配置模块，负责定义工具的常量标识和系统提示描述。虽然代码量很小（仅 18 行），但它在工具识别和模型指导方面起着关键作用。

### 核心定位
- **工具标识**：定义 FileWriteTool 的常量名称
- **提示描述**：为模型提供工具使用说明和行为准则
- **跨工具协调**：与 FileReadTool 协作，确保正确的使用顺序

### 使用场景
1. 模型初始化时获取工具描述
2. 工具注册时识别工具类型
3. 生成系统提示时嵌入使用说明

---

## 功能点目的

### 1. 工具名称常量（FILE_WRITE_TOOL_NAME）
```typescript
export const FILE_WRITE_TOOL_NAME = 'Write'
```
- **标识符**：作为工具在系统中的唯一名称
- **模型可见**：模型使用此名称调用工具
- **跨文件引用**：被 FileWriteTool.ts、权限系统、分析日志等引用

### 2. 基础描述（DESCRIPTION）
```typescript
export const DESCRIPTION = 'Write a file to the local filesystem.'
```
- **简短描述**：一句话概括工具功能
- **搜索提示**：用于工具搜索和分类

### 3. 预读取指令（getPreReadInstruction）
```typescript
function getPreReadInstruction(): string {
  return `\n- If this is an existing file, you MUST use the ${FILE_READ_TOOL_NAME} tool first to read the file's contents. This tool will fail if you did not read the file first.`
}
```
- **强制读取要求**：强调必须先读取现有文件才能写入
- **跨工具引用**：引用 FileReadTool 的名称，确保一致性
- **失败预警**：明确告知不读取会导致工具失败

### 4. 完整工具描述（getWriteToolDescription）
```typescript
export function getWriteToolDescription(): string {
  return `Writes a file to the local filesystem.

Usage:
- This tool will overwrite the existing file if there is one at the provided path.${getPreReadInstruction()}
- Prefer the Edit tool for modifying existing files — it only sends the diff. Only use this tool to create new files or for complete rewrites.
- NEVER create documentation files (*.md) or README files unless explicitly requested by the User.
- Only use emojis if the user explicitly requests it. Avoid writing emojis to files unless asked.`
}
```

包含以下指导原则：

| 指导原则 | 目的 |
|---------|------|
| 覆盖写入警告 | 提醒工具会覆盖现有文件 |
| 必须先读后写 | 强制要求读取现有文件 |
| 推荐使用 Edit 工具 | 引导模型使用更高效的局部编辑 |
| 限制使用场景 | 仅用于创建新文件或完全重写 |
| 禁止自动创建文档 | 防止未经请求创建 README/文档 |
| 表情符号限制 | 避免未经请求使用表情符号 |

---

## 具体技术实现

### 模块结构
```
prompt.ts
├── 导入: FILE_READ_TOOL_NAME (来自 FileReadTool/prompt.ts)
├── 导出: FILE_WRITE_TOOL_NAME
├── 导出: DESCRIPTION
├── 内部函数: getPreReadInstruction()
└── 导出函数: getWriteToolDescription()
```

### 依赖关系
```
FileWriteTool/prompt.ts
  └── 导入: FileReadTool/prompt.ts (FILE_READ_TOOL_NAME)

FileWriteTool/FileWriteTool.ts
  └── 导入: ./prompt.ts (FILE_WRITE_TOOL_NAME, getWriteToolDescription)
```

### 调用链
```
1. 系统初始化 / 模型查询
   └── FileWriteTool.prompt() (在 FileWriteTool.ts 中定义)
       └── getWriteToolDescription()
           └── getPreReadInstruction()

2. 工具注册
   └── tools.ts
       └── 引用 FILE_WRITE_TOOL_NAME

3. 分析日志
   └── 各处 analytics 调用
       └── 引用 FILE_WRITE_TOOL_NAME 作为工具标识
```

---

## 关键代码路径与文件引用

### 核心实现文件
- `/src/tools/FileWriteTool/prompt.ts` - 主实现（18 行）

### 依赖文件
- `/src/tools/FileReadTool/prompt.ts` - FILE_READ_TOOL_NAME 常量

### 调用方
- `/src/tools/FileWriteTool/FileWriteTool.ts` - 工具定义中使用
- `/src/tools.ts` - 工具注册
- `/src/constants/tools.ts` - 工具常量
- `/src/services/tools/toolExecution.ts` - 工具执行
- `/src/utils/fileOperationAnalytics.ts` - 文件操作分析
- `/src/utils/collapseReadSearch.ts` - 读取搜索折叠
- `/src/services/compact/microCompact.ts` - 紧凑模式
- `/src/services/extractMemories/prompts.ts` - 记忆提取提示
- `/src/tools/AgentTool/built-in/exploreAgent.ts` - 探索 Agent
- `/src/tools/AgentTool/built-in/verificationAgent.ts` - 验证 Agent
- `/src/tools/AgentTool/built-in/planAgent.ts` - 计划 Agent
- `/src/tools/REPLTool/primitiveTools.ts` - REPL 原语工具
- `/src/tools/REPLTool/constants.ts` - REPL 常量

---

## 依赖与外部交互

### 与 FileReadTool 的协作
```typescript
import { FILE_READ_TOOL_NAME } from '../FileReadTool/prompt.js'
```
- **名称依赖**：FileWriteTool 的提示需要引用 FileReadTool 的名称
- **行为协调**：提示中强调必须先使用 Read 工具
- **循环依赖风险**：注意避免与 FileReadTool 形成循环导入

### 与 FileWriteTool.ts 的集成
```typescript
// 在 FileWriteTool.ts 中
import { FILE_WRITE_TOOL_NAME, getWriteToolDescription } from './prompt.js'

async prompt() {
  return getWriteToolDescription()
}
```
- **延迟调用**：prompt 函数在需要时调用 getWriteToolDescription
- **动态生成**：支持未来扩展为根据上下文动态生成描述

---

## 风险、边界与改进建议

### 已知风险

1. **循环导入风险**
   - 风险：如果 FileReadTool/prompt.ts 反向依赖 FileWriteTool，会形成循环
   - 当前状态：安全，FileReadTool/prompt.ts 是独立的
   - 缓解：保持 prompt.ts 文件的简单性，避免复杂依赖

2. **提示内容重复**
   - 风险：类似的"必须先读"逻辑可能在多个地方重复
   - 现状：FileEditTool 也有类似的读取要求
   - 建议：考虑提取通用的"文件操作前置条件"提示

3. **国际化缺失**
   - 风险：提示文本硬编码为英文，不支持多语言
   - 影响：非英语用户可能难以理解工具使用规则
   - 建议：未来考虑 i18n 支持

### 边界情况

1. **空返回值处理**
   - 边界：getWriteToolDescription 始终返回字符串
   - 安全：没有 null/undefined 风险

2. **模板字符串注入**
   - 边界：getPreReadInstruction 使用模板字符串插入 FILE_READ_TOOL_NAME
   - 安全：FILE_READ_TOOL_NAME 是常量，无注入风险

### 改进建议

1. **提示内容增强**
   - 添加文件大小限制说明
   - 添加支持的编码格式说明
   - 添加二进制文件处理指导

2. **动态提示生成**
   ```typescript
   // 示例：根据项目类型生成定制提示
   export function getWriteToolDescription(projectType?: string): string {
     const baseDescription = `Writes a file to the local filesystem...`
     if (projectType === 'web') {
       return baseDescription + '\n- For web projects, avoid writing to node_modules.'
     }
     return baseDescription
   }
   ```

3. **提示版本控制**
   - 添加版本号，便于追踪提示变更对模型行为的影响
   - 支持 A/B 测试不同的提示表述

4. **与 FileEditTool 提示统一**
   - 考虑将共同的"必须先读"逻辑提取到共享模块
   - 统一文件操作工具的提示风格

5. **文档化提示设计原则**
   - 记录提示设计的最佳实践
   - 建立提示审查流程，确保新工具提示的一致性

---

## 总结

prompt.ts 虽然代码量极小，但承担着重要的职责：

1. **单一职责**：专注于工具标识和提示描述，不涉及业务逻辑
2. **高内聚**：相关常量和方法集中在一个文件
3. **低耦合**：仅依赖 FileReadTool 的名称常量，无其他复杂依赖
4. **可扩展**：函数式导出便于未来扩展为动态提示生成

这种设计模式值得在其他工具模块中借鉴：将工具配置（prompt.ts）与业务逻辑（FileWriteTool.ts）分离，提高代码的可维护性和可测试性。
