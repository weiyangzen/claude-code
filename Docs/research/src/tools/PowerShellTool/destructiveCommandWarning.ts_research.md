# destructiveCommandWarning.ts 研究文档

## 场景与职责

destructiveCommandWarning.ts 提供**破坏性命令检测**功能，用于在权限请求对话框中向用户显示警告信息。

### 重要定位

> **本模块纯粹是信息性的** —— 它不影响权限逻辑，不阻止命令执行，仅用于 UI 展示。

```
权限决策流程：
1. 安全检查（powershellSecurity.ts）→ 可能阻止
2. 路径验证（pathValidation.ts）→ 可能阻止
3. 模式验证（modeValidation.ts）→ 可能自动允许
4. 破坏性检测（本模块）→ 仅添加警告文本，不阻止
```

### 与 BashTool 的对应关系

本模块功能类似于 BashTool 中的破坏性命令检测，针对 PowerShell 语法进行了适配。

## 功能点目的

### 1. 递归强制删除检测

**目的**：检测 `Remove-Item` 配合 `-Recurse` 和/或 `-Force` 的危险组合。

**检测的命令别名**：
- `Remove-Item`（完整名）
- `rm`（Unix 风格别名）
- `del`（DOS 风格别名）
- `rd`, `rmdir`（目录删除别名）
- `ri`（PowerShell 标准别名）

**正则模式详解**：
```typescript
/(?:^|[|;&\n({])\s*(Remove-Item|rm|del|rd|rmdir|ri)\b[^|;&\n}]*-Recurse\b[^|;&\n}]*-Force\b/i
```

- `(?:^|[|;&\n({])` - 语句开始或分隔符
- `\s*` - 可选空白
- `(Remove-Item|rm|...)` - 命令名
- `\b` - 单词边界
- `[^|;&\n}]*` - 非分隔符字符（防止跨语句匹配）
- `-Recurse\b` - Recurse 参数
- `-Force\b` - Force 参数

**警告级别**：
| 检测到的模式 | 警告信息 |
|-------------|----------|
| `-Recurse` + `-Force` | "Note: may recursively force-remove files" |
| `-Recurse` | "Note: may recursively remove files" |
| `-Force` | "Note: may force-remove files" |

### 2. Clear-Content 检测

**目的**：检测清空多个文件内容的操作。

```typescript
{
  pattern: /\bClear-Content\b[^|;&\n]*\*/i,
  warning: 'Note: may clear content of multiple files',
}
```

检测 `Clear-Content` 配合通配符 `*` 的使用。

### 3. 磁盘操作检测

**目的**：检测格式化磁盘的危险操作。

```typescript
{ pattern: /\bFormat-Volume\b/i, warning: 'Note: may format a disk volume' }
{ pattern: /\bClear-Disk\b/i, warning: 'Note: may clear a disk' }
```

### 4. Git 破坏性操作检测

**目的**：检测可能导致数据丢失的 Git 操作。

| 模式 | 警告 |
|------|------|
| `git reset --hard` | "Note: may discard uncommitted changes" |
| `git push --force` / `--force-with-lease` / `-f` | "Note: may overwrite remote history" |
| `git clean -f`（非 dry-run） | "Note: may permanently delete untracked files" |
| `git stash drop/clear` | "Note: may permanently remove stashed changes" |

**`git clean` 检测的复杂性**：
```typescript
/\bgit\s+clean\b(?![^|;&\n]*(?:-[a-zA-Z]*n|--dry-run))[^|;&\n]*-[a-zA-Z]*f/i
```

- `(?![^|;&\n]*(?:-[a-zA-Z]*n|--dry-run))` - 负向前瞻，排除 dry-run 模式
- `-[a-zA-Z]*f` - 匹配包含 `f` 的任何标志组合

### 5. 数据库操作检测

**目的**：检测 SQL 破坏性操作。

```typescript
{ pattern: /\b(DROP|TRUNCATE)\s+(TABLE|DATABASE|SCHEMA)\b/i, 
  warning: 'Note: may drop or truncate database objects' }
```

### 6. 系统操作检测

**目的**：检测影响系统状态的操作。

| 模式 | 警告 |
|------|------|
| `Stop-Computer` | "Note: will shut down the computer" |
| `Restart-Computer` | "Note: will restart the computer" |
| `Clear-RecycleBin` | "Note: permanently deletes recycled files" |

## 具体技术实现

### 数据结构

```typescript
type DestructivePattern = {
  pattern: RegExp
  warning: string
}

const DESTRUCTIVE_PATTERNS: DestructivePattern[] = [
  // ... 模式定义
]
```

### 主函数

```typescript
export function getDestructiveCommandWarning(command: string): string | null {
  for (const { pattern, warning } of DESTRUCTIVE_PATTERNS) {
    if (pattern.test(command)) {
      return warning
    }
  }
  return null
}
```

### 正则模式设计原则

1. **锚定到语句边界**：使用 `(?:^|[|;&\n({])` 确保匹配命令开始位置
2. **防止跨语句匹配**：使用 `[^|;&\n}]*` 限制匹配范围在同一句句内
3. **单词边界**：使用 `\b` 防止部分匹配（如 `Remove-ItemX` 不匹配）
4. **大小写不敏感**：使用 `i` 标志

## 关键代码路径与文件引用

### 调用链

```
1. 权限请求生成
   src/components/permissions/PowerShellPermissionRequest/PowerShellPermissionRequest.tsx
   → 调用 getDestructiveCommandWarning()

2. 警告显示
   → 将警告文本添加到权限对话框
   → 用户看到警告后决定是否批准
```

### 相关文件

- `src/components/permissions/PowerShellPermissionRequest/PowerShellPermissionRequest.tsx` - 调用方
- `src/tools/PowerShellTool/powershellPermissions.ts` - 权限检查主流程

## 依赖与外部交互

### 无外部依赖

```typescript
// 无任何 import
```

### 被依赖方

```typescript
// PowerShellPermissionRequest.tsx
import { getDestructiveCommandWarning } from '../../../tools/PowerShellTool/destructiveCommandWarning.js'
```

## 风险、边界与改进建议

### 已知风险

1. **误报（False Positives）**：
   - `Remove-Item ./temp -Recurse -Force` 可能是完全合法的操作
   - 警告可能让用户对正常操作感到不安

2. **漏报（False Negatives）**：
   - 正则无法覆盖所有破坏性命令变体
   - 动态生成的命令名无法检测

3. **正则复杂性**：
   - 复杂的正则可能难以维护
   - 性能问题（虽然通常命令字符串不长）

### 边界情况

| 场景 | 处理行为 |
|------|----------|
| 空命令 | 返回 null |
| 多个危险模式匹配 | 返回第一个匹配的警告 |
| 大小写混合 | 不敏感匹配（i 标志）|
| 命令在字符串中 | 如果匹配正则，仍会触发 |

### 改进建议

1. **优先级系统**：
   ```typescript
   type DestructivePattern = {
     pattern: RegExp
     warning: string
     severity: 'low' | 'medium' | 'high' | 'critical'
   }
   ```
   根据严重程度使用不同的 UI 样式。

2. **更精确的检测**：
   - 结合 AST 分析，而非纯正则
   - 区分 `"./temp"` 和 `"/"` 的路径风险

3. **可配置性**：
   - 允许用户禁用某些警告
   - 允许添加自定义警告模式

4. **上下文感知**：
   - 检测目标路径的风险级别
   - `Remove-Item ./temp` vs `Remove-Item C:\Windows`

5. **更多模式**：
   ```typescript
   // 建议添加
   { pattern: /\bRemove-Module\b.*-Force/i, warning: '...' }
   { pattern: /\bUnregister-ScheduledTask\b/i, warning: '...' }
   { pattern: /\bSet-ExecutionPolicy\b/i, warning: '...' }
   ```

### 测试要点

- 各种命令别名（rm, del, rd, ri）的检测
- 参数顺序变化（-Recurse -Force vs -Force -Recurse）
- 语句边界处理（管道、分号、换行）
- 大小写不敏感
- 无匹配时返回 null
- 多个模式时的优先级
