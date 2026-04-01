# gitSafety.ts 研究文档

## 场景与职责

gitSafety.ts 实现 PowerShell 的 **Git 安全检测**，防止通过 Git 命令进行沙箱逃逸攻击。

### 攻击背景：Bare-Repo 攻击

攻击者可利用 Git 的两种机制进行沙箱逃逸：

1. **Bare-repo 攻击**：
   - 如果当前目录包含 `HEAD` + `objects/` + `refs/` 但没有有效的 `.git/HEAD`
   - Git 会将当前目录视为 bare repository
   - 执行 `git` 命令时会运行当前目录下的 hooks

2. **Git-internal 写入 + git 执行**：
   - 复合命令先创建 `HEAD/objects/refs/hooks/` 目录结构
   - 然后运行 `git` 子命令
   - Git 执行刚创建的恶意 hooks

### 本模块的职责

1. **检测 Git 内部路径**：判断参数是否解析到 Git 内部文件/目录
2. **处理路径变体**：应对各种 PowerShell 路径语法和编码技巧
3. **CWD 重入解析**：处理通过 `../<cwd-basename>/` 重新进入工作目录的路径

## 功能点目的

### 1. Git 内部路径检测（isGitInternalPathPS）

**目的**：检测参数是否指向 Git 内部路径（bare-repo 和标准仓库）。

**检测的前缀**：
```typescript
const GIT_INTERNAL_PREFIXES = ['head', 'objects', 'refs', 'hooks']
```

**匹配规则**：
- `head` 或 `.git` → 匹配
- `.git/` 开头 → 匹配
- `git~N`（NTFS 短名）→ 匹配
- 上述前缀 + `/` → 匹配

**路径变体处理**：
1. **CWD 重入解析**：`../project/hooks` 可能实际指向 `./hooks`
2. **绝对路径逃逸**：`C:\full\path\to\HEAD` 可能实际在 CWD 内
3. **NTFS 短名**：`.git` 可能显示为 `GIT~1`

### 2. .git 目录检测（isDotGitPathPS）

**目的**：专门检测指向 `.git/` 目录的路径（标准仓库的元数据目录）。

与 `isGitInternalPathPS` 的区别：
- `isGitInternalPathPS`：匹配 bare-repo 根级路径（`hooks/`, `refs/` 等）和标准 `.git/`
- `isDotGitPathPS`：只匹配 `.git/` 路径

### 3. 路径规范化（normalizeGitPathArg）

**目的**：将各种 PowerShell 路径语法统一为可比较的标准形式。

**处理步骤**（按顺序）：

| 步骤 | 处理 | 示例 |
|------|------|------|
| 1 | 参数前缀剥离 | `/Path:hooks` → `hooks` |
| 2 | Unicode dash 处理 | `–Path:hooks` → `hooks` |
| 3 | 引号移除 | `'hooks'` → `hooks` |
| 4 | 反引号转义移除 | `` `n `` → `n` |
| 5 | Provider 前缀移除 | `FileSystem::hooks` → `hooks` |
| 6 | 驱动器相对路径 | `C:foo` → `foo`（如果无分隔符）|
| 7 | 反斜杠转斜杠 | `hooks\pre-commit` → `hooks/pre-commit` |
| 8 | Win32 每组件处理 | `hooks . ` → `hooks` |
| 9 | POSIX 规范化 | `hooks/../HEAD` → `HEAD` |
| 10 | 小写化 | `HEAD` → `head` |

**Win32 CreateFileW 每组件处理**：
```typescript
// 迭代移除尾部空格和点
while (c !== prev) {
  c = c.replace(/ +$/, '')     // 尾部空格
  if (c === '.' || c === '..') return c
  c = c.replace(/\.+$/, '')   // 尾部点
}
```

这是 Windows 文件系统的实际行为：`.. ` → `..`，`...` → `` → `.`

### 4. CWD 重入解析（resolveCwdReentry）

**目的**：处理通过 `../<cwd-basename>/` 重新进入工作目录的路径。

**问题场景**：
```
CWD: /x/project
路径: ../project/hooks
posix.normalize: 保持 ../project/hooks（无 CWD 上下文）
实际解析: /x/project/hooks（在 CWD 内！）
```

**解决方案**：
```typescript
function resolveCwdReentry(normalized: string): string {
  if (!normalized.startsWith('../')) return normalized
  const cwdBase = basename(getCwd()).toLowerCase()
  const prefix = '../' + cwdBase + '/'
  
  // 迭代剥离 ../<cwd-basename>/ 对
  let s = normalized
  while (s.startsWith(prefix)) {
    s = s.slice(prefix.length)
  }
  return s
}
```

### 5. 逃逸路径解析（resolveEscapingPathToCwdRelative）

**目的**：解析以 `../` 开头或绝对路径的参数，判断是否实际落在 CWD 内。

**这是唯一的 bare-repo HEAD 攻击防护**：
- `path-validation.ts` 的 `DANGEROUS_FILES` 故意排除 bare `HEAD`（误报风险）
- `DANGEROUS_DIRECTORIES` 只匹配 `.git` 段
- 所以 `<cwd>/HEAD` 会通过路径验证层
- 本函数是阻止此类攻击的唯一防线

## 具体技术实现

### 核心数据结构

```typescript
const GIT_INTERNAL_PREFIXES = ['head', 'objects', 'refs', 'hooks'] as const

// Git 内部路径检测
export function isGitInternalPathPS(arg: string): boolean {
  const n = resolveCwdReentry(normalizeGitPathArg(arg))
  if (matchesGitInternalPrefix(n)) return true
  
  // 处理逃逸路径
  if (n.startsWith('../') || n.startsWith('/') || /^[a-z]:/.test(n)) {
    const rel = resolveEscapingPathToCwdRelative(n)
    if (rel !== null && matchesGitInternalPrefix(rel)) return true
  }
  return false
}

// .git 路径检测
export function isDotGitPathPS(arg: string): boolean {
  const n = resolveCwdReentry(normalizeGitPathArg(arg))
  if (matchesDotGitPrefix(n)) return true
  
  // 同样的逃逸路径处理
  if (n.startsWith('../') || n.startsWith('/') || /^[a-z]:/.test(n)) {
    const rel = resolveEscapingPathToCwdRelative(n)
    if (rel !== null && matchesDotGitPrefix(rel)) return true
  }
  return false
}
```

### 路径规范化流程

```
原始参数: -Path:'..\project\hooks\pre-commit'
    ↓
参数前缀剥离: '..\project\hooks\pre-commit'
    ↓
引号移除: ..\project\hooks\pre-commit
    ↓
反引号处理: （无变化）
    ↓
Provider 前缀移除: （无变化）
    ↓
驱动器相对路径: （无变化）
    ↓
反斜杠转斜杠: ../project/hooks/pre-commit
    ↓
Win32 组件处理: （无变化）
    ↓
POSIX 规范化: ../project/hooks/pre-commit
    ↓
小写化: ../project/hooks/pre-commit
    ↓
CWD 重入解析: hooks/pre-commit
    ↓
前缀匹配: hooks/ 匹配 → 返回 true
```

## 关键代码路径与文件引用

### 调用链

```
1. 路径验证
   src/tools/PowerShellTool/pathValidation.ts: checkPathConstraints()
   → 提取命令参数中的路径
   → 调用 isGitInternalPathPS() 检查是否为 Git 内部路径

2. 写入命令保护
   src/tools/PowerShellTool/powershellPermissions.ts
   → GIT_SAFETY_WRITE_CMDLETS 列表
   → 对写入命令检查参数是否指向 Git 内部路径

3. 复合命令保护
   → 检测 cd + git 模式
   → 检测 tar/unzip + git 模式
```

### 相关文件

- `src/tools/PowerShellTool/pathValidation.ts` - 调用路径检查
- `src/tools/PowerShellTool/powershellPermissions.ts` - 权限检查主流程
- `src/utils/powershell/parser.ts` - 提供 PS_TOKENIZER_DASH_CHARS

## 依赖与外部交互

### 外部依赖

```typescript
import { basename, posix, resolve, sep } from 'path'
import { getCwd } from '../../utils/cwd.js'
import { PS_TOKENIZER_DASH_CHARS } from '../../utils/powershell/parser.js'
```

### 被依赖方

```typescript
// pathValidation.ts
import { isDotGitPathPS, isGitInternalPathPS } from './gitSafety.js'

// powershellPermissions.ts
import { isDotGitPathPS, isGitInternalPathPS } from './gitSafety.js'
```

## 风险、边界与改进建议

### 已知风险

1. **TOCTOU（检查时间到使用时间）**：
   - 路径验证时文件可能还不存在
   - 验证通过后、执行前，攻击者可能创建恶意结构
   - 缓解：复合命令检测（tar + git 等）

2. **路径规范化复杂性**：
   - PowerShell 路径语法极其复杂
   - 可能遗漏某些编码技巧

3. **性能考虑**：
   - 每个路径参数都要经过多层正则和字符串处理
   - 对长命令可能有性能影响

### 边界情况

| 场景 | 处理行为 |
|------|----------|
| 空字符串 | 不匹配任何前缀 |
| 只有 `.` 或 `..` | 特殊处理，返回原值 |
| NTFS 数据流 | `file.txt:stream` 未特殊处理 |
| 非常长的路径 | 正常处理，但可能性能下降 |
| 包含 null 字节 | 未特殊处理 |

### 改进建议

1. **更全面的路径测试**：
   - 添加针对各种编码技巧的测试用例
   - 红队测试寻找绕过方法

2. **性能优化**：
   - 缓存规范化结果
   - 提前退出（最明显的模式先检查）

3. **与其他工具统一**：
   - 与 BashTool 的 gitSafety.ts 共享核心逻辑
   - 避免重复实现

4. **更多攻击向量覆盖**：
   ```typescript
   // 考虑添加
   - .gitattributes 操纵
   - .gitignore 操纵
   - Git 工作树攻击
   ```

5. **文档增强**：
   - 添加攻击向量示意图
   - 记录每个规范化步骤的安全理由

### 测试要点

- 各种路径格式（相对、绝对、UNC）
- PowerShell 特殊语法（provider 路径、驱动器相对）
- Unicode dash 字符
- 反引号转义
- NTFS 短名（git~1）
- CWD 重入路径（../project/hooks）
- 多层父目录（../../project/project/hooks）
- 大小写不敏感匹配
