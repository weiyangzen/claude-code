# commandSemantics.ts 研究文档

## 场景与职责

`commandSemantics.ts` 是 BashTool 的**命令语义解释模块**，负责处理不同 shell 命令的退出码语义差异。许多 Unix 命令使用退出码传达超出简单成功/失败的信息（例如 `grep` 返回 1 表示未找到匹配项，这不是错误），该模块提供统一的解释层，将原始退出码转换为结构化的结果状态。

### 核心职责
1. **退出码语义映射**：为常见命令（grep、find、diff、test 等）定义特定的退出码解释规则
2. **命令结果解释**：根据命令类型和退出码判断是否为错误，并生成人类可读的消息
3. **复合命令处理**：从管道命令中提取最后一个命令作为决定退出码的主体

## 功能点目的

### 1. 命令语义类型定义
```typescript
export type CommandSemantic = (
  exitCode: number,
  stdout: string,
  stderr: string,
) => { isError: boolean; message?: string }
```

该类型定义了语义解释器的签名，允许基于退出码、标准输出和标准错误进行灵活的结果判断。

### 2. 默认语义 vs 特定语义
- **DEFAULT_SEMANTIC**: 仅将退出码 0 视为成功，其他均为错误
- **COMMAND_SEMANTICS**: Map 结构存储命令特定的语义规则

### 3. 支持的命令语义

| 命令 | 退出码 0 | 退出码 1 | 退出码 2+ |
|------|---------|---------|----------|
| grep/rg | 匹配成功 | 无匹配（非错误） | 错误 |
| find | 成功 | 部分成功（某些目录不可访问） | 错误 |
| diff | 无差异 | 有差异（非错误） | 错误 |
| test/[ | 条件为真 | 条件为假（非错误） | 错误 |

## 具体技术实现

### 关键流程

#### 1. 命令语义获取流程
```typescript
function getCommandSemantic(command: string): CommandSemantic {
  const baseCommand = heuristicallyExtractBaseCommand(command)
  const semantic = COMMAND_SEMANTICS.get(baseCommand)
  return semantic !== undefined ? semantic : DEFAULT_SEMANTIC
}
```

#### 2. 启发式命令提取
```typescript
function heuristicallyExtractBaseCommand(command: string): string {
  const segments = splitCommand_DEPRECATED(command)
  // 取最后一个命令，因为管道中最后一个命令决定退出码
  const lastCommand = segments[segments.length - 1] || command
  return extractBaseCommand(lastCommand)
}
```

**安全说明**: 注释明确指出此方法"可能完全错误——不要将其用于安全决策"，仅用于语义解释。

#### 3. 结果解释流程
```typescript
export function interpretCommandResult(
  command: string,
  exitCode: number,
  stdout: string,
  stderr: string,
): { isError: boolean; message?: string }
```

## 关键代码路径与文件引用

### 导出函数
- `interpretCommandResult`: 主入口函数，用于解释命令执行结果
- `CommandSemantic`: 类型定义

### 依赖关系
```typescript
import { splitCommand_DEPRECATED } from '../../utils/bash/commands.js'
```

### 调用方
- `src/tools/BashTool/BashTool.tsx`: 在命令执行后调用 `interpretCommandResult` 解释结果
- `src/tools/PowerShellTool/PowerShellTool.tsx`: PowerShell 工具也使用类似的语义解释

### 相关文件
- `src/utils/bash/commands.ts`: 提供 `splitCommand_DEPRECATED` 用于命令分割
- `src/tools/PowerShellTool/commandSemantics.ts`: PowerShell 版本的语义解释

## 依赖与外部交互

### 运行时依赖
| 依赖 | 用途 |
|------|------|
| `splitCommand_DEPRECATED` | 分割复合命令（管道）以提取最后一个命令 |

### 数据结构
```typescript
const COMMAND_SEMANTICS: Map<string, CommandSemantic> = new Map([
  ['grep', (exitCode) => ({ isError: exitCode >= 2, message: exitCode === 1 ? 'No matches found' : undefined })],
  ['rg', /* 同 grep */],
  ['find', (exitCode) => ({ isError: exitCode >= 2, message: exitCode === 1 ? 'Some directories were inaccessible' : undefined })],
  ['diff', (exitCode) => ({ isError: exitCode >= 2, message: exitCode === 1 ? 'Files differ' : undefined })],
  ['test', (exitCode) => ({ isError: exitCode >= 2, message: exitCode === 1 ? 'Condition is false' : undefined })],
  ['[', /* 同 test */],
])
```

## 风险、边界与改进建议

### 已知风险

1. **启发式提取的局限性**
   - `heuristicallyExtractBaseCommand` 使用简单的字符串分割，可能被复杂的 shell 语法误导
   - 注释明确警告不要用于安全决策

2. **命令覆盖不完整**
   - 仅覆盖 grep、find、diff、test 等常见命令
   - 其他命令（如 ack、ag、rg 的变体）使用默认语义，可能产生误导性错误报告

3. **复合命令语义**
   - 管道命令的语义仅基于最后一个命令，中间命令的失败可能被忽略
   - 例如 `false | true` 返回 0，但 `false` 实际上失败了

### 边界情况

1. **空命令**: `extractBaseCommand` 处理空字符串时返回空字符串
2. **纯空白命令**: `trim().split(/\s+/)` 处理纯空白输入
3. **未知命令**: 回退到 DEFAULT_SEMANTIC

### 改进建议

1. **扩展命令覆盖**
   ```typescript
   // 建议添加
   ['ack', /* 同 grep */],
   ['ag', /* 同 grep */],
   ['git diff', /* 同 diff */],
   ```

2. **管道失败检测**
   - 考虑使用 `pipefail` 选项的语义，检测管道中任何命令的失败
   - 或提供配置选项让用户选择管道语义

3. **与 AST 解析集成**
   - 当前依赖 `splitCommand_DEPRECATED`，建议迁移到 tree-sitter AST 解析
   - 更准确地识别命令结构和类型

4. **国际化支持**
   - 当前消息为硬编码英文，考虑支持多语言错误消息

5. **测试覆盖**
   - 建议添加单元测试覆盖各种退出码组合
   - 特别是边界情况（exitCode = 0, 1, 2, 255）
