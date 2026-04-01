# destructiveCommandWarning.ts 研究文档

## 场景与职责

`destructiveCommandWarning.ts` 是 BashTool 的**破坏性命令警告模块**，负责检测可能具有破坏性的 bash 命令并返回警告字符串。这些警告纯粹是信息性的——它们不影响权限逻辑或自动批准，仅用于在权限对话框中向用户显示风险提示。

### 核心职责
1. **破坏性模式检测**：通过正则表达式匹配识别危险的命令模式
2. **风险分类**：覆盖 Git 操作、文件删除、数据库操作、基础设施操作等
3. **警告生成**：返回人类可读的警告消息，帮助用户理解潜在风险

## 功能点目的

### 1. 警告信息性设计
> "This is purely informational — it doesn't affect permission logic or auto-approval."

警告仅用于增强用户意识，不改变权限决策流程。这是有意的设计选择，确保：
- 用户了解潜在风险
- 权限系统保持独立和可预测
- 不会因为警告而意外地阻止合法操作

### 2. 风险类别覆盖

| 类别 | 示例命令 | 警告内容 |
|------|---------|----------|
| Git - 数据丢失 | `git reset --hard` | "may discard uncommitted changes" |
| Git - 历史重写 | `git push --force` | "may overwrite remote history" |
| Git - 文件删除 | `git clean -f` | "may permanently delete untracked files" |
| Git - 工作区丢弃 | `git checkout .` | "may discard all working tree changes" |
| Git - 安全绕过 | `git commit --no-verify` | "may skip safety hooks" |
| 文件删除 | `rm -rf` | "may recursively force-remove files" |
| 数据库 | `DROP TABLE` | "may drop or truncate database objects" |
| 基础设施 | `kubectl delete` | "may delete Kubernetes resources" |

## 具体技术实现

### 数据结构
```typescript
type DestructivePattern = {
  pattern: RegExp
  warning: string
}
```

### 关键流程

#### 1. 模式匹配
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

线性扫描所有模式，返回第一个匹配的警告。

#### 2. 正则表达式模式详解

**Git 重置（--hard）**
```typescript
pattern: /\bgit\s+reset\s+--hard\b/
```
- `\b`: 单词边界，确保匹配完整单词
- 匹配：`git reset --hard`

**Git 强制推送**
```typescript
pattern: /\bgit\s+push\b[^;&|\n]*[ \t](--force|--force-with-lease|-f)\b/
```
- `[^;&|\n]*`: 匹配直到控制字符或换行
- 支持 `--force`、`-f`、`--force-with-lease`

**Git 清理（排除 dry-run）**
```typescript
pattern: /\bgit\s+clean\b(?![^;&|\n]*(?:-[a-zA-Z]*n|--dry-run))[^;&|\n]*-[a-zA-Z]*f/
```
- `(?!...)`: 负向前瞻，排除包含 `-n` 或 `--dry-run` 的命令
- 确保只在真正会删除文件时警告

**Git checkout/restore 丢弃更改**
```typescript
pattern: /\bgit\s+checkout\s+(--\s+)?\.[ \t]*($|[;&|\n])/
```
- `(--\s+)?`: 可选的 `--` 分隔符
- `\.`: 匹配当前目录 `.`

**递归强制删除（rm）**
```typescript
pattern: /(^|[;&|\n]\s*)rm\s+-[a-zA-Z]*[rR][a-zA-Z]*f|(^|[;&|\n]\s*)rm\s+-[a-zA-Z]*f[a-zA-Z]*[rR]/
```
- 匹配 `-rf` 或 `-fr` 的任何变体
- 考虑命令分隔符后的上下文

**数据库操作**
```typescript
pattern: /\b(DROP|TRUNCATE)\s+(TABLE|DATABASE|SCHEMA)\b/i
```
- 不区分大小写（`i` 标志）
- 匹配 SQL 的 DROP/TRUNCATE 操作

## 关键代码路径与文件引用

### 导出函数
- `getDestructiveCommandWarning`: 主入口函数

### 调用方
- `src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx`: 在权限请求对话框中显示警告
- `src/components/permissions/PowerShellPermissionRequest/PowerShellPermissionRequest.tsx`: PowerShell 权限请求

### 相关文件
- `src/tools/PowerShellTool/destructiveCommandWarning.ts`: PowerShell 版本的破坏性命令检测

## 依赖与外部交互

### 运行时依赖
该模块**无外部依赖**，是纯正则表达式匹配模块。

### 模式列表（DESTRUCTIVE_PATTERNS）

```typescript
const DESTRUCTIVE_PATTERNS: DestructivePattern[] = [
  // Git — 数据丢失 / 难以撤销
  { pattern: /\bgit\s+reset\s+--hard\b/, warning: 'Note: may discard uncommitted changes' },
  { pattern: /\bgit\s+push\b[^;&|\n]*[ \t](--force|--force-with-lease|-f)\b/, warning: 'Note: may overwrite remote history' },
  { pattern: /\bgit\s+clean\b(?![^;&|\n]*(?:-[a-zA-Z]*n|--dry-run))[^;&|\n]*-[a-zA-Z]*f/, warning: 'Note: may permanently delete untracked files' },
  { pattern: /\bgit\s+checkout\s+(--\s+)?\.[ \t]*($|[;&|\n])/, warning: 'Note: may discard all working tree changes' },
  { pattern: /\bgit\s+restore\s+(--\s+)?\.[ \t]*($|[;&|\n])/, warning: 'Note: may discard all working tree changes' },
  { pattern: /\bgit\s+stash[ \t]+(drop|clear)\b/, warning: 'Note: may permanently remove stashed changes' },
  { pattern: /\bgit\s+branch\s+(-D[ \t]|--delete\s+--force|--force\s+--delete)\b/, warning: 'Note: may force-delete a branch' },
  
  // Git — 安全绕过
  { pattern: /\bgit\s+(commit|push|merge)\b[^;&|\n]*--no-verify\b/, warning: 'Note: may skip safety hooks' },
  { pattern: /\bgit\s+commit\b[^;&|\n]*--amend\b/, warning: 'Note: may rewrite the last commit' },
  
  // 文件删除
  { pattern: /(^|[;&|\n]\s*)rm\s+-[a-zA-Z]*[rR][a-zA-Z]*f|(^|[;&|\n]\s*)rm\s+-[a-zA-Z]*f[a-zA-Z]*[rR]/, warning: 'Note: may recursively force-remove files' },
  { pattern: /(^|[;&|\n]\s*)rm\s+-[a-zA-Z]*[rR]/, warning: 'Note: may recursively remove files' },
  { pattern: /(^|[;&|\n]\s*)rm\s+-[a-zA-Z]*f/, warning: 'Note: may force-remove files' },
  
  // 数据库
  { pattern: /\b(DROP|TRUNCATE)\s+(TABLE|DATABASE|SCHEMA)\b/i, warning: 'Note: may drop or truncate database objects' },
  { pattern: /\bDELETE\s+FROM\s+\w+[ \t]*(;|"|'|\n|$)/i, warning: 'Note: may delete all rows from a database table' },
  
  // 基础设施
  { pattern: /\bkubectl\s+delete\b/, warning: 'Note: may delete Kubernetes resources' },
  { pattern: /\bterraform\s+destroy\b/, warning: 'Note: may destroy Terraform infrastructure' },
]
```

## 风险、边界与改进建议

### 已知风险

1. **正则表达式复杂性**
   - 复杂的正则可能导致性能问题（回溯）
   - 某些边缘情况可能无法覆盖

2. **误报和漏报**
   - `git clean -nf`（dry-run）不应警告，但 `git clean -n -f` 可能会
   - 注释中的危险命令可能被误匹配

3. **仅检测已知模式**
   - 新型危险命令需要手动添加
   - 无法检测组合攻击（如 `echo rm -rf /`）

4. **信息性警告的局限性**
   - 用户可能忽视警告
   - 警告不会阻止执行

### 边界情况

| 输入 | 行为 | 说明 |
|------|------|------|
| `""` | `null` | 空命令 |
| `"git reset"` | `null` | 缺少 `--hard` |
| `"git reset --hard"` | 警告 | 匹配 |
| `"# git reset --hard"` | 警告 | 注释中也会匹配 |
| `"echo git reset --hard"` | 警告 | 字符串中也会匹配 |

### 改进建议

1. **添加更多模式**
   ```typescript
   // Docker 破坏性操作
   { pattern: /\bdocker\s+(system\s+prune|volume\s+rm|rm\s+-f)/, warning: 'Note: may remove Docker resources' },
   
   // 系统级操作
   { pattern: /\bsudo\s+rm\s+-rf\s+\//, warning: 'Note: extremely dangerous system deletion' },
   
   // 网络操作
   { pattern: /\biptables\s+-F/, warning: 'Note: may flush all firewall rules' },
   ```

2. **上下文感知检测**
   ```typescript
   // 排除注释和字符串中的匹配
   function getDestructiveCommandWarning(command: string): string | null {
     const codeWithoutComments = stripComments(command)
     const codeWithoutStrings = stripStrings(codeWithoutComments)
     // 在清理后的代码上匹配
   }
   ```

3. **风险等级分级**
   ```typescript
   type RiskLevel = 'low' | 'medium' | 'high' | 'critical'
   type DestructivePattern = {
     pattern: RegExp
     warning: string
     level: RiskLevel
   }
   ```

4. **命令链分析**
   ```typescript
   // 检测组合风险
   // "cd / && rm -rf *" 比单独的 "rm -rf *" 更危险
   ```

5. **性能优化**
   - 对于频繁调用的场景，考虑使用 Trie 或 Aho-Corasick 算法
   - 预编译正则表达式（当前已静态定义）

### 测试建议

```typescript
describe('getDestructiveCommandWarning', () => {
  it('检测 git reset --hard', () => {
    expect(getDestructiveCommandWarning('git reset --hard')).toContain('discard')
  })
  it('排除 git clean dry-run', () => {
    expect(getDestructiveCommandWarning('git clean -n -f')).toBeNull()
  })
  it('检测 rm -rf', () => {
    expect(getDestructiveCommandWarning('rm -rf /tmp')).toContain('recursively')
  })
  it('处理复合命令', () => {
    expect(getDestructiveCommandWarning('echo foo && git push --force')).toContain('overwrite')
  })
})
```
