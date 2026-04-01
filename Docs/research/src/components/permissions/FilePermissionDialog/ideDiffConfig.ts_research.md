# ideDiffConfig.ts 研究文档

## 场景与职责

`ideDiffConfig.ts` 是 Claude Code 中 IDE 差异对比功能的配置定义模块。它定义了文件编辑操作在 IDE（如 VSCode）中展示差异视图所需的配置结构、类型和工具函数。

### 核心职责
1. **类型定义**：定义 IDE diff 配置的核心 TypeScript 接口
2. **配置生成**：提供便捷函数生成单编辑 diff 配置
3. **变更应用**：定义如何将用户在 IDE 中的修改应用回原始输入

### 使用场景
- 文件编辑（FileEditTool）的 IDE diff 支持
- 文件写入（FileWriteTool）的 IDE diff 支持
- 任何需要在 IDE 中展示代码变更预览的文件操作

---

## 功能点目的

### 1. IDEDiffConfig 接口
- **目的**：定义在 IDE 中展示 diff 所需的基本配置
- **字段说明**：
  - `filePath`: 要对比的文件路径
  - `edits`: 文件编辑列表（旧字符串、新字符串、是否全局替换）
  - `editMode`: 编辑模式（'single' 单处编辑 | 'multiple' 多处编辑）

### 2. IDEDiffSupport 泛型接口
- **目的**：为不同类型的工具输入提供类型安全的 diff 支持
- **核心方法**：
  - `getConfig`: 从工具输入生成 diff 配置
  - `applyChanges`: 将用户在 IDE 中的修改应用回工具输入

### 3. createSingleEditDiffConfig 工具函数
- **目的**：简化单处文件编辑的 diff 配置创建
- **适用场景**：大多数文件编辑操作（FileEditTool、FileWriteTool）

---

## 具体技术实现

### 关键数据结构

```typescript
// 文件编辑定义
export interface FileEdit {
  old_string: string    // 原始内容
  new_string: string    // 替换后的内容
  replace_all?: boolean // 是否替换所有匹配项
}

// IDE diff 配置
export interface IDEDiffConfig {
  filePath: string
  edits?: FileEdit[]
  editMode?: 'single' | 'multiple'
}

// IDE diff 变更输入（从 IDE 返回的数据结构）
export interface IDEDiffChangeInput {
  file_path: string
  edits: FileEdit[]
}

// 泛型 diff 支持接口
export interface IDEDiffSupport<TInput extends ToolInput> {
  getConfig(input: TInput): IDEDiffConfig
  applyChanges(input: TInput, modifiedEdits: FileEdit[]): TInput
}
```

### 工具函数实现

```typescript
export function createSingleEditDiffConfig(
  filePath: string,
  oldString: string,
  newString: string,
  replaceAll?: boolean,
): IDEDiffConfig {
  return {
    filePath,
    edits: [
      {
        old_string: oldString,
        new_string: newString,
        replace_all: replaceAll,
      },
    ],
    editMode: 'single',
  }
}
```

### 类型约束

```typescript
// 来自 useFilePermissionDialog.ts
export interface ToolInput {
  [key: string]: unknown
}
```

`IDEDiffSupport` 使用泛型 `TInput extends ToolInput` 确保类型安全：
- 输入类型必须是对象类型（索引签名 `[key: string]: unknown`）
- 允许不同工具定义自己的具体输入类型

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `./useFilePermissionDialog.js` | 导入 ToolInput 基础类型 |

### 被引用文件

| 文件路径 | 用途 |
|---------|------|
| `./FilePermissionDialog.tsx` | 使用 IDEDiffSupport 类型 |
| `../FileEditPermissionRequest/FileEditPermissionRequest.tsx` | 实现 FileEditTool 的 IDEDiffSupport |
| `../FileWritePermissionRequest/FileWritePermissionRequest.tsx` | 实现 FileWriteTool 的 IDEDiffSupport |

### 类型使用示例

#### FileEditPermissionRequest 中的实现
```typescript
// 文件：../FileEditPermissionRequest/FileEditPermissionRequest.tsx
type FileEditInput = z.infer<typeof FileEditTool.inputSchema>;

const ideDiffSupport: IDEDiffSupport<FileEditInput> = {
  getConfig: (input: FileEditInput) => createSingleEditDiffConfig(
    input.file_path,
    input.old_string,
    input.new_string,
    input.replace_all
  ),
  applyChanges: (input: FileEditInput, modifiedEdits: FileEdit[]) => {
    const firstEdit = modifiedEdits[0];
    if (firstEdit) {
      return {
        ...input,
        old_string: firstEdit.old_string,
        new_string: firstEdit.new_string,
        replace_all: firstEdit.replace_all
      };
    }
    return input;
  }
};
```

#### FileWritePermissionRequest 中的实现
```typescript
// 文件：../FileWritePermissionRequest/FileWritePermissionRequest.tsx
type FileWriteToolInput = z.infer<typeof FileWriteTool.inputSchema>;

const ideDiffSupport: IDEDiffSupport<FileWriteToolInput> = {
  getConfig: (input: FileWriteToolInput) => {
    let oldContent: string;
    try {
      oldContent = readFileSync(input.file_path);
    } catch (e) {
      if (!isENOENT(e)) throw e;
      oldContent = '';
    }
    return createSingleEditDiffConfig(
      input.file_path,
      oldContent,
      input.content,
      false // File writes replace entire content
    );
  },
  applyChanges: (input: FileWriteToolInput, modifiedEdits: FileEdit[]) => {
    const firstEdit = modifiedEdits[0];
    if (firstEdit) {
      return {
        ...input,
        content: firstEdit.new_string
      };
    }
    return input;
  }
};
```

---

## 依赖与外部交互

### 类型依赖关系

```
ideDiffConfig.ts
    ↓ 导入
useFilePermissionDialog.ts (ToolInput)
    ↓ 被导入
FilePermissionDialog.tsx (使用 IDEDiffSupport<T>)
FileEditPermissionRequest.tsx (实现 IDEDiffSupport<FileEditInput>)
FileWritePermissionRequest.tsx (实现 IDEDiffSupport<FileWriteToolInput>)
```

### 与 IDE 的交互流程

1. **配置生成阶段**：
   ```
   FileEditPermissionRequest
       ↓
   ideDiffSupport.getConfig(input) → IDEDiffConfig
       ↓
   FilePermissionDialog
   ```

2. **Diff 展示阶段**：
   ```
   FilePermissionDialog
       ↓
   useDiffInIDE(diffParams)
       ↓
   IDE 扩展 (MCP)
   ```

3. **变更应用阶段**：
   ```
   IDE 扩展 (用户保存/关闭)
       ↓
   useDiffInIDE onChange callback
       ↓
   ideDiffSupport.applyChanges(input, modifiedEdits)
       ↓
   更新后的 input
   ```

---

## 风险、边界与改进建议

### 潜在风险

#### 1. 类型安全边界
- **风险**：`ToolInput` 使用 `[key: string]: unknown` 过于宽松
- **影响**：编译时无法捕获类型错误，依赖运行时验证
- **建议**：考虑使用更严格的泛型约束或 branded types

#### 2. FileEdit 数组越界风险
- **风险**：`applyChanges` 实现依赖 `modifiedEdits[0]`，数组为空时行为未定义
- **现有防护**：实现中检查了 `firstEdit` 是否存在
- **建议**：在类型层面强制要求非空数组，或添加运行时断言

#### 3. 编辑模式一致性
- **风险**：`editMode: 'single'` 但 `edits` 数组可能包含多个编辑
- **影响**：可能导致 IDE diff 展示不正确
- **建议**：在 `createSingleEditDiffConfig` 中添加断言或限制

### 边界情况

#### 1. 空字符串处理
- `oldString` 或 `newString` 为空字符串时的 diff 展示
- 新文件创建（`oldString = ''`）的场景

#### 2. 特殊字符处理
- 包含 Unicode、换行符、制表符的内容
- 大文件（>1MB）的 diff 性能

#### 3. 并发编辑
- 用户在 IDE 中编辑时，文件被外部修改
- 网络延迟导致的竞态条件

### 改进建议

#### 1. 类型增强
```typescript
// 建议：添加更严格的类型约束
export interface NonEmptyFileEdit extends FileEdit {
  // 确保至少有一个编辑
}

export interface SingleEditConfig extends IDEDiffConfig {
  editMode: 'single'
  edits: [FileEdit] // 元组类型，强制长度为 1
}
```

#### 2. 验证函数
```typescript
// 建议：添加配置验证
export function validateIDEDiffConfig(config: IDEDiffConfig): void {
  if (config.editMode === 'single' && config.edits?.length !== 1) {
    throw new Error('Single edit mode requires exactly one edit');
  }
  // 其他验证...
}
```

#### 3. 文档增强
- 添加 JSDoc 注释说明各字段的用途和约束
- 提供使用示例

#### 4. 性能优化
- 考虑对大文件进行内容截断或分块处理
- 添加 diff 内容哈希用于快速比较

### 架构建议

1. **分离关注点**：将 `FileEdit` 类型移动到独立的 types 模块，供其他模块共享
2. **常量提取**：将 `'single'` 和 `'multiple'` 提取为枚举或常量
3. **工厂模式**：为不同工具类型提供预置的 `IDEDiffSupport` 实现工厂

```typescript
// 建议：工厂模式示例
export const IDEDiffSupportFactory = {
  forFileEditTool(): IDEDiffSupport<FileEditInput> { /* ... */ },
  forFileWriteTool(): IDEDiffSupport<FileWriteToolInput> { /* ... */ },
};
```
