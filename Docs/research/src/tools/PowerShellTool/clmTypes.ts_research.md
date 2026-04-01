# clmTypes.ts 研究文档

## 场景与职责

clmTypes.ts 实现了 PowerShell **约束语言模式（Constrained Language Mode, CLM）** 的类型安全检测机制。

### 背景：什么是 CLM？

PowerShell 的 Constrained Language Mode 是 Microsoft 为 AppLocker/WDAC 系统锁定环境设计的安全机制。在此模式下：
- 只允许使用白名单中的 .NET 类型
- 阻止反射、P/Invoke、进程创建等危险操作
- 由 Microsoft 维护允许的类型列表

### 本模块的职责

1. **维护 CLM 允许类型白名单**（`CLM_ALLOWED_TYPES`）
2. **提供类型名称规范化**（`normalizeTypeName`）
3. **判断类型是否在白名单中**（`isClmAllowedType`）

**安全模型反转**：Microsoft 用白名单允许安全类型；我们用同一白名单检测危险类型——不在白名单中的类型触发 "ask" 权限请求。

## 功能点目的

### 1. CLM 允许类型白名单（CLM_ALLOWED_TYPES）

**目的**：定义 PowerShell 中可安全使用的 .NET 类型集合。

**类型分类**：

| 类别 | 示例 | 说明 |
|------|------|------|
| 类型加速器（短名） | `int`, `string`, `bool` | PowerShell 内置别名 |
| 完整类型名 | `System.Int32`, `System.String` | .NET 完整限定名 |
| CIM 类型 | `ciminstance`, `cimclass` | WMI/CIM 相关 |
| PS 特定类型 | `psobject`, `pscredential` | PowerShell 自定义类型 |
| 验证属性 | `validatepattern`, `validaterange` | 参数验证属性 |

**安全移除的类型**（注释详细说明原因）：

```typescript
// REMOVED: 'adsi', 'adsisearcher'
// 原因：Active Directory Service Interface 类型会执行网络绑定
// [adsi]'LDAP://evil.com/...' → 连接到 LDAP 服务器

// REMOVED: 'wmi', 'wmiclass', 'wmisearcher', 'cimsession'
// 原因：WMI 类型可针对远程计算机执行查询
// [wmi]'\\evil-host\root\cimv2:Win32_Process.Handle="1"' → 远程 WMI
```

### 2. 类型名称规范化（normalizeTypeName）

**目的**：处理 PowerShell 类型语法的变体，统一用于白名单查找。

**处理规则**：
1. 转小写：`String` → `string`
2. 移除数组后缀：`String[]` → `string`
3. 移除泛型参数：`List[int]` → `list`
4. 修剪空白

**安全考量**：泛型容器保守处理——即使类型参数安全，容器本身可能不安全。

### 3. 类型白名单检查（isClmAllowedType）

**目的**：判断给定的类型字面量是否在 CLM 白名单中。

**使用场景**：
```typescript
// powershellSecurity.ts 中的调用
function checkTypeLiterals(parsed: ParsedPowerShellCommand): PowerShellSecurityResult {
  for (const t of parsed.typeLiterals ?? []) {
    if (!isClmAllowedType(t)) {
      return {
        behavior: 'ask',
        message: `Command uses .NET type [${t}] outside the ConstrainedLanguage allowlist`
      }
    }
  }
  return { behavior: 'passthrough' }
}
```

## 具体技术实现

### 数据结构

```typescript
// 只读集合，防止运行时修改
export const CLM_ALLOWED_TYPES: ReadonlySet<string> = new Set([
  // 约 180+ 个类型名称（全部小写存储）
  'alias', 'array', 'bool', 'byte', 'char', ...
])

// 类型名称规范化函数
export function normalizeTypeName(name: string): string {
  return name
    .toLowerCase()
    .replace(/\[\]$/, '')      // 移除数组后缀
    .replace(/\[.*\]$/, '')    // 移除泛型参数
    .trim()
}

// 白名单检查函数
export function isClmAllowedType(typeName: string): boolean {
  return CLM_ALLOWED_TYPES.has(normalizeTypeName(typeName))
}
```

### 类型分类详解

#### 1. 基础类型加速器（行 27-121）
```typescript
'int', 'int16', 'int32', 'int64',      // 整数
'uint', 'uint16', 'uint32', 'uint64',  // 无符号整数
'float', 'double', 'decimal',          // 浮点数
'string', 'char', 'byte', 'bool',      // 基础类型
'array', 'hashtable', 'ordered',       // 集合
'datetime', 'timespan', 'guid',        // 时间和标识
'regex', 'version', 'uri',             // 实用类型
```

#### 2. 完整 .NET 类型名（行 123-155）
```typescript
'system.array', 'system.boolean', 'system.byte',
'system.collections.hashtable',
'system.text.regularexpressions.regex',
'system.net.ipaddress', 'system.net.mail.mailaddress',
'system.security.securestring',
'system.xml.xmldocument'
```

#### 3. PowerShell 特定类型（行 157-167）
```typescript
'system.management.automation.pscredential',
'system.management.automation.pscustomobject',
'system.management.automation.psobject',
'system.management.automation.switchparameter',
```

#### 4. CIM 类型（行 170-173）
```typescript
'microsoft.management.infrastructure.cimclass',
'microsoft.management.infrastructure.ciminstance',
// 注意：cimsession 被移除（网络安全风险）
```

## 关键代码路径与文件引用

### 调用链

```
1. PowerShell 命令解析
   src/utils/powershell/parser.ts: parsePowerShellCommand()
   → 提取 typeLiterals（AST 中的 TypeExpressionAst + TypeConstraintAst）

2. 安全检查
   src/tools/PowerShellTool/powershellSecurity.ts: checkTypeLiterals()
   → 遍历 parsed.typeLiterals
   → 调用 isClmAllowedType() 检查每个类型

3. New-Object 类型检查
   src/tools/PowerShellTool/powershellSecurity.ts: checkComObject()
   → 提取 -TypeName 参数值
   → 调用 isClmAllowedType() 检查
```

### 相关文件

- `src/utils/powershell/parser.ts` - 解析 PowerShell 命令，提取类型字面量
- `src/tools/PowerShellTool/powershellSecurity.ts` - 使用 CLM 检查进行安全验证
- `src/tools/PowerShellTool/powershellPermissions.ts` - 权限检查主流程

## 依赖与外部交互

### 无外部依赖

本模块是纯数据定义，不依赖其他模块：
```typescript
// 无任何 import
```

### 被依赖方

```typescript
// powershellSecurity.ts
import { isClmAllowedType } from './clmTypes.js'
```

## 风险、边界与改进建议

### 已知风险

1. **白名单维护负担**：
   - PowerShell/.NET 版本更新可能引入新类型
   - 需要跟踪 Microsoft 的 CLM 文档更新

2. **类型加速器解析差异**：
   - AST 可能输出短名（`[int]`）或全名（`[System.Int32]`）
   - 白名单包含两者，但边缘情况可能遗漏

3. **泛型保守处理**：
   - `List[int]` 被归一化为 `list` 后检查
   - 如果 `list` 不在白名单，安全类型参数也被拒

### 边界情况

| 场景 | 处理行为 |
|------|----------|
| 空字符串 | `normalizeTypeName('')` → `''`，不在白名单中 |
| 只有数组后缀 `[]` | `normalizeTypeName('[]')` → `''` |
| 嵌套泛型 | `List[Dict[string,int]]` → `list` |
| 带空格的名称 | 先 trim 再处理 |
| 大写混合 | 全部转小写后比较 |

### 改进建议

1. **自动化同步**：
   - 编写脚本从 Microsoft 文档自动提取 CLM 类型列表
   - CI 检查白名单与官方文档的差异

2. **更细粒度的泛型处理**：
   - 考虑允许 `List[T]` 其中 T 是白名单类型
   - 需要递归检查类型参数

3. **类型别名扩展**：
   - 添加更多 PowerShell 社区常用类型
   - 考虑用户自定义类型白名单配置

4. **安全注释增强**：
   - 为每个被移除的类型添加攻击向量示例
   - 便于安全审计和新成员培训

### 测试要点

- 各种类型名称变体的规范化
- 数组类型：`int[]`, `string[][]`
- 泛型类型：`List[int]`, `Dictionary[string,object]`
- 大小写混合：`STRING`, `System.STRING`
- 空白处理：`  string  `, `string []`
- 被移除的危险类型检测
