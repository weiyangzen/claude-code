# commonParameters.ts 研究文档

## 场景与职责

commonParameters.ts 定义了 PowerShell 的**通用参数（Common Parameters）**，这些参数通过 `[CmdletBinding()]` 属性在所有 cmdlet 上可用。

### 为什么需要这个模块？

在 PowerShell 权限验证中，需要区分：
1. **路径参数**（需要验证文件系统访问）
2. **开关参数**（不消耗后续参数）
3. **值参数**（消耗后续参数，但非路径）

通用参数属于第 2 和第 3 类，需要在多个验证模块中共享定义，避免重复和循环依赖。

### 模块定位

```
pathValidation.ts ← 使用 COMMON_PARAMETERS
readOnlyValidation.ts ← 使用 COMMON_PARAMETERS
         ↓
    commonParameters.ts（本模块，打破循环依赖）
```

## 功能点目的

### 1. 通用开关参数（COMMON_SWITCHES）

**目的**：定义不消耗后续值的开关参数。

```typescript
export const COMMON_SWITCHES = ['-verbose', '-debug']
```

**行为**：当解析器遇到这些参数时，知道下一个 token 不是它们的值。

**示例**：
```powershell
Get-Process -Verbose -Name chrome
#           ^^^^^^^^ 开关，不消耗 "chrome"
#                    ^^^^^ 是 -Name 的值
```

### 2. 通用值参数（COMMON_VALUE_PARAMS）

**目的**：定义消耗后续值的参数，但这些值不是文件路径。

```typescript
export const COMMON_VALUE_PARAMS = [
  '-erroraction',      # 错误处理动作：Continue, Stop, SilentlyContinue...
  '-warningaction',    # 警告处理动作
  '-informationaction',# 信息处理动作
  '-progressaction',   # 进度处理动作
  '-errorvariable',    # 存储错误的变量名
  '-warningvariable',  # 存储警告的变量名
  '-informationvariable', # 存储信息的变量名
  '-outvariable',      # 存储输出的变量名
  '-outbuffer',        # 输出缓冲区大小
  '-pipelinevariable', # 管道变量名
]
```

**行为**：当解析器遇到这些参数时，知道下一个 token 是它们的值，且该值不应被当作路径验证。

**示例**：
```powershell
Get-Content -Path ./file.txt -ErrorAction Stop
#                              ^^^^^^^^^^^ 值参数
#                                          ^^^^ 值（不是路径）
```

### 3. 合并集合（COMMON_PARAMETERS）

**目的**：提供统一的参数集合用于快速查找。

```typescript
export const COMMON_PARAMETERS: ReadonlySet<string> = new Set([
  ...COMMON_SWITCHES,
  ...COMMON_VALUE_PARAMS,
])
```

## 具体技术实现

### 数据结构

```typescript
/**
 * PowerShell Common Parameters (available on all cmdlets via [CmdletBinding()]).
 * Source: about_CommonParameters (PowerShell docs) + Get-Command output.
 *
 * Shared between pathValidation.ts (merges into per-cmdlet known-param sets)
 * and readOnlyValidation.ts (merges into safeFlags check). Split out to break
 * what would otherwise be an import cycle between those two files.
 *
 * Stored lowercase with leading dash — callers `.toLowerCase()` their input.
 */

export const COMMON_SWITCHES = ['-verbose', '-debug']

export const COMMON_VALUE_PARAMS = [
  '-erroraction',
  '-warningaction',
  '-informationaction',
  '-progressaction',
  '-errorvariable',
  '-warningvariable',
  '-informationvariable',
  '-outvariable',
  '-outbuffer',
  '-pipelinevariable',
]

export const COMMON_PARAMETERS: ReadonlySet<string> = new Set([
  ...COMMON_SWITCHES,
  ...COMMON_VALUE_PARAMS,
])
```

### 使用模式

在 `pathValidation.ts` 中：
```typescript
import { COMMON_SWITCHES, COMMON_VALUE_PARAMS } from './commonParameters.js'

const CMDLET_PATH_CONFIG: Record<string, CmdletPathConfig> = {
  'get-content': {
    operationType: 'read',
    pathParams: ['-path', '-literalpath'],
    // 合并通用开关
    knownSwitches: [
      '-force',
      '-raw',
      ...COMMON_SWITCHES,  // 展开通用开关
    ],
    // 合并通用值参数
    knownValueParams: [
      '-totalcount',
      '-encoding',
      ...COMMON_VALUE_PARAMS,  // 展开通用值参数
    ],
  },
}
```

在 `readOnlyValidation.ts` 中：
```typescript
import { COMMON_PARAMETERS } from './commonParameters.js'

// 验证参数时排除通用参数
function validateFlags(cmd: string, args: string[]): boolean {
  for (const arg of args) {
    const lower = arg.toLowerCase()
    if (COMMON_PARAMETERS.has(lower)) {
      continue  // 通用参数自动允许
    }
    // ... 其他验证逻辑
  }
}
```

## 关键代码路径与文件引用

### 依赖关系图

```
commonParameters.ts
    ├──→ pathValidation.ts
    │       └── 合并到每个 cmdlet 的参数配置
    │
    └──→ readOnlyValidation.ts
            └── 用于安全标志验证
```

### 相关文件

- `src/tools/PowerShellTool/pathValidation.ts` - 路径验证，合并通用参数
- `src/tools/PowerShellTool/readOnlyValidation.ts` - 只读命令验证

## 依赖与外部交互

### 无外部依赖

```typescript
// 无任何 import
```

### 被依赖方

```typescript
// pathValidation.ts
import { COMMON_SWITCHES, COMMON_VALUE_PARAMS } from './commonParameters.js'

// readOnlyValidation.ts
import { COMMON_PARAMETERS } from './commonParameters.js'
```

## 风险、边界与改进建议

### 已知风险

1. **参数名称变更**：
   - PowerShell 新版本可能添加新的通用参数
   - 当前列表基于 PowerShell 5.1/7.x 文档

2. **大小写敏感**：
   - 存储为小写，调用方需要 `.toLowerCase()`
   - 如果调用方忘记转换，会导致匹配失败

3. **缩写处理**：
   - PowerShell 支持参数缩写（如 `-ea` 代表 `-ErrorAction`）
   - 本模块只包含完整名称，缩写处理在其他模块

### 边界情况

| 场景 | 处理行为 |
|------|----------|
| 参数大小写混合 | 调用方负责转小写 |
| 带冒号的参数 | `-ErrorAction:Stop` 不在集合中，需要调用方处理 |
| 带空格的参数名 | 不考虑，PowerShell 参数名不包含空格 |

### 改进建议

1. **添加更多通用参数**：
   ```typescript
   // 考虑添加（如果确实通用）
   '-whatif'      // 某些 cmdlet 支持
   '-confirm'     // 某些 cmdlet 支持
   ```
   注意：`-WhatIf` 和 `-Confirm` 不是所有 cmdlet 都支持，只在支持 `ShouldProcess` 的 cmdlet 上可用。

2. **缩写支持**：
   ```typescript
   // 添加常见缩写映射
   export const COMMON_PARAM_ABBREVIATIONS: Record<string, string> = {
     '-ea': '-erroraction',
     '-wa': '-warningaction',
     '-ev': '-errorvariable',
     '-wv': '-warningvariable',
     '-ov': '-outvariable',
     '-pv': '-pipelinevariable',
   }
   ```

3. **参数分类增强**：
   ```typescript
   // 更细粒度的分类
   export const COMMON_ACTION_PARAMS = ['-erroraction', '-warningaction', ...]
   export const COMMON_VARIABLE_PARAMS = ['-errorvariable', '-warningvariable', ...]
   ```

4. **自动化同步**：
   - 编写脚本从 PowerShell 文档提取通用参数列表
   - CI 检查与官方文档的差异

### 测试要点

- 所有通用参数都被正确识别
- 大小写不敏感匹配
- 在 pathValidation 中的合并行为
- 在 readOnlyValidation 中的排除行为
- 不存在的参数不被误匹配
