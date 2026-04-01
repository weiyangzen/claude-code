# filesystem.ts 研究文档

## 场景与职责

`filesystem.ts` 是 Claude Code 权限系统的核心文件系统权限检查模块，负责处理所有与文件系统相关的权限决策。该模块是权限系统中最大、最复杂的文件之一（约 1778 行），涵盖了文件读写权限的完整检查流程。

核心职责：

1. **文件读取权限检查** (`checkReadPermissionForTool`)：12 步检查流程
2. **文件写入权限检查** (`checkWritePermissionForTool`)：5 步检查流程
3. **路径安全验证** (`checkPathSafetyForAutoEdit`)：Windows 路径模式、危险文件检测
4. **内部路径处理**：会话内存、计划文件、暂存目录等特殊路径
5. **模式匹配**：基于 gitignore 风格的权限规则匹配
6. **建议生成**：为用户生成权限规则建议

## 功能点目的

### 1. 危险文件/目录定义

```typescript
export const DANGEROUS_FILES = [
  '.gitconfig', '.gitmodules', '.bashrc', '.bash_profile',
  '.zshrc', '.zprofile', '.profile', '.ripgreprc', '.mcp.json', '.claude.json',
] as const

export const DANGEROUS_DIRECTORIES = ['.git', '.vscode', '.idea', '.claude'] as const
```

### 2. 文件读取权限检查流程 (12 步)

1. **UNC 路径拦截** - 阻止网络路径访问
2. **Windows 可疑模式检查** - NTFS 流、8.3 短名、长路径前缀等
3. **读取特定拒绝规则检查**
4. **读取特定询问规则检查**
5. **编辑权限隐含读取** - 如果有编辑权限则允许读取
6. **工作目录检查** - 允许读取工作目录内文件
7. **内部可读路径检查** - 会话内存、计划文件等
8. **允许规则检查**
9. **默认询问** - 以上都不满足时询问用户

### 3. 文件写入权限检查流程 (5 步)

1. **拒绝规则检查**
2. **内部可编辑路径检查** - 计划文件、暂存目录等
3. **.claude 目录允许规则检查** - 会话级规则可绕过安全检查
4. **综合安全验证** - Windows 模式、Claude 配置、危险文件
5. **询问规则检查**
6. **acceptEdits 模式检查**
7. **允许规则检查**
8. **默认询问**

### 4. 路径安全验证

检测以下可疑 Windows 路径模式：
- **NTFS 备用数据流** (`file.txt::$DATA`)
- **8.3 短名** (`GIT~1`, `CLAUDE~1`)
- **长路径前缀** (`\\?\C:\...`, `//?/C:/...`)
- **尾部点/空格** (`.git.`, `.claude `)
- **DOS 设备名** (`.git.CON`, `settings.json.PRN`)
- **连续三个点** (`.../file.txt`)
- **UNC 路径** (`\\server\share`)

### 5. Claude 技能范围检测

```typescript
export function getClaudeSkillScope(
  filePath: string,
): { skillName: string; pattern: string } | null
```

检测文件是否位于 `.claude/skills/{name}/` 目录下，返回技能名称和限定范围的权限模式。

### 6. 临时目录管理

- `getClaudeTempDir()` - 获取 Claude 临时目录（带 memoization）
- `getBundledSkillsRoot()` - 获取捆绑技能根目录（带随机 nonce）
- `getProjectTempDir()` - 获取项目临时目录
- `getScratchpadDir()` - 获取暂存目录

## 具体技术实现

### 读取权限检查核心逻辑

```typescript
export function checkReadPermissionForTool(
  tool: Tool,
  input: { [key: string]: unknown },
  toolPermissionContext: ToolPermissionContext,
): PermissionDecision {
  const path = tool.getPath(input)
  const pathsToCheck = getPathsForPermissionCheck(path)

  // 1. UNC 路径防御
  for (const pathToCheck of pathsToCheck) {
    if (pathToCheck.startsWith('\\') || pathToCheck.startsWith('//')) {
      return { behavior: 'ask', message: '...', decisionReason: {...} }
    }
  }

  // 2. Windows 可疑模式
  for (const pathToCheck of pathsToCheck) {
    if (hasSuspiciousWindowsPathPattern(pathToCheck)) {
      return { behavior: 'ask', message: '...', decisionReason: {...} }
    }
  }

  // 3. 读取特定拒绝规则
  for (const pathToCheck of pathsToCheck) {
    const denyRule = matchingRuleForInput(pathToCheck, toolPermissionContext, 'read', 'deny')
    if (denyRule) {
      return { behavior: 'deny', message: '...', decisionReason: {...} }
    }
  }

  // 4. 读取特定询问规则
  // 5. 编辑权限隐含读取
  // 6. 工作目录检查
  // 7. 内部可读路径
  // 8. 允许规则
  // 9. 默认询问
}
```

### 模式匹配实现

使用 `ignore` 库实现 gitignore 风格的模式匹配：

```typescript
export function matchingRuleForInput(
  path: string,
  toolPermissionContext: ToolPermissionContext,
  toolType: 'edit' | 'read',
  behavior: 'allow' | 'deny' | 'ask',
): PermissionRule | null {
  const patternsByRoot = getPatternsByRoot(toolPermissionContext, toolType, behavior)
  
  for (const [root, patternMap] of patternsByRoot.entries()) {
    const patterns = Array.from(patternMap.keys()).map(pattern => {
      let adjustedPattern = pattern
      if (adjustedPattern.endsWith('/**')) {
        adjustedPattern = adjustedPattern.slice(0, -3)
      }
      return adjustedPattern
    })

    const ig = ignore().add(patterns)
    const relativePathStr = relativePath(root ?? getCwd(), fileAbsolutePath)
    
    if (relativePathStr.startsWith(`..${DIR_SEP}`)) continue
    
    const igResult = ig.test(relativePathStr)
    if (igResult.ignored && igResult.rule) {
      // 返回匹配的规则
      return patternMap.get(igResult.rule.pattern) ?? null
    }
  }
  return null
}
```

### 路径安全检查

```typescript
function hasSuspiciousWindowsPathPattern(path: string): boolean {
  // NTFS 备用数据流（Windows/WSL  only）
  if (getPlatform() === 'windows' || getPlatform() === 'wsl') {
    const colonIndex = path.indexOf(':', 2)
    if (colonIndex !== -1) return true
  }

  // 8.3 短名
  if (/~\d/.test(path)) return true

  // 长路径前缀
  if (path.startsWith('\\\\?\\') || path.startsWith('\\\\.\\') ||
      path.startsWith('//?/') || path.startsWith('//./')) return true

  // 尾部点/空格
  if (/[.\s]+$/.test(path)) return true

  // DOS 设备名
  if (/\.(CON|PRN|AUX|NUL|COM[1-9]|LPT[1-9])$/i.test(path)) return true

  // 连续三个点
  if (/(^|\/|\\)\.{3,}(\/|\\|$)/.test(path)) return true

  // UNC 路径
  if (containsVulnerableUncPath(path)) return true

  return false
}
```

### 内部可编辑路径检查

```typescript
export function checkEditableInternalPath(
  absolutePath: string,
  input: { [key: string]: unknown },
): PermissionResult {
  const normalizedPath = normalize(absolutePath)

  // 计划文件
  if (isSessionPlanFile(normalizedPath)) {
    return { behavior: 'allow', updatedInput: input, decisionReason: {...} }
  }

  // 暂存目录
  if (isScratchpadPath(normalizedPath)) {
    return { behavior: 'allow', updatedInput: input, decisionReason: {...} }
  }

  // 作业目录（TEMPLATES 功能）
  if (feature('TEMPLATES')) {
    const jobDir = process.env.CLAUDE_JOB_DIR
    // ... 验证和允许逻辑
  }

  // Agent 内存
  if (isAgentMemoryPath(normalizedPath)) {
    return { behavior: 'allow', updatedInput: input, decisionReason: {...} }
  }

  // Memdir
  if (!hasAutoMemPathOverride() && isAutoMemPath(normalizedPath)) {
    return { behavior: 'allow', updatedInput: input, decisionReason: {...} }
  }

  // launch.json
  if (isLaunchJsonPath(normalizedPath)) {
    return { behavior: 'allow', updatedInput: input, decisionReason: {...} }
  }

  return { behavior: 'passthrough', message: '' }
}
```

## 关键代码路径与文件引用

### 导入依赖
- `ignore`：gitignore 风格模式匹配
- `lodash-es/memoize`：记忆化工具
- `zod/v4`：类型定义
- 各种工具常量文件
- 设置、路径、平台等工具模块

### 导出函数
| 函数 | 用途 |
|------|------|
| `checkReadPermissionForTool` | 检查工具读取权限 |
| `checkWritePermissionForTool` | 检查工具写入权限 |
| `checkPathSafetyForAutoEdit` | 检查路径自动编辑安全性 |
| `matchingRuleForInput` | 匹配输入与权限规则 |
| `generateSuggestions` | 生成权限建议 |
| `getFileReadIgnorePatterns` | 获取文件读取忽略模式 |
| `getClaudeTempDir` | 获取 Claude 临时目录 |
| `getBundledSkillsRoot` | 获取捆绑技能根目录 |
| `getClaudeSkillScope` | 获取 Claude 技能范围 |
| `normalizeCaseForComparison` | 大小写规范化 |
| `toPosixPath` | 转换为 POSIX 路径 |

### 被引用位置
- `src/utils/permissions/permissions.ts`：权限检查调用
- `src/tools/FileReadTool/FileReadTool.ts`：文件读取权限
- `src/tools/FileEditTool/FileEditTool.ts`：文件编辑权限
- `src/tools/GlobTool/GlobTool.ts`：文件匹配权限
- `src/tools/GrepTool/GrepTool.ts`：文本搜索权限
- 其他文件系统相关工具

## 依赖与外部交互

### 上游依赖
| 模块 | 用途 |
|------|------|
| `ignore` | gitignore 模式匹配 |
| `lodash-es/memoize` | 记忆化 |
| `zod/v4` | 类型定义 |
| 工具常量文件 | 工具名称 |
| `settings/settings.js` | 设置路径 |
| `path.js` | 路径操作 |
| `platform.js` | 平台检测 |

### 下游消费者
| 模块 | 用途 |
|------|------|
| `permissions.ts` | 权限检查 |
| 文件系统工具 | 权限验证 |
| UI 组件 | 权限建议 |

## 风险、边界与改进建议

### 风险点
1. **复杂度**：文件过大（1778 行），包含多个职责，维护困难
2. **安全检查顺序**：检查顺序至关重要，错误的顺序可能导致安全绕过
3. **Windows 路径处理**：Windows 路径的复杂性增加了安全风险
4. **符号链接**：`getPathsForPermissionCheck` 解析符号链接，可能存在 TOCTOU 竞态条件

### 边界条件
1. **路径规范化**：多处使用 `normalize()` 和 `expandPath()`，需要确保一致性
2. **大小写敏感**：macOS/Windows 不区分大小写，使用 `normalizeCaseForComparison()` 处理
3. **跨平台路径**：Windows 使用 `\`，POSIX 使用 `/`，需要统一处理
4. **内存使用**：`memoize` 缓存路径解析结果，可能占用大量内存

### 改进建议
1. **模块拆分**：将文件拆分为多个小模块：
   - `pathSecurity.ts` - 路径安全检查
   - `internalPaths.ts` - 内部路径处理
   - `patternMatching.ts` - 模式匹配逻辑
   - `tempDirectories.ts` - 临时目录管理

2. **安全检查审计**：定期进行安全审计，确保检查顺序和逻辑正确

3. **性能优化**：
   - `matchingRuleForInput` 每次调用都创建新的 `ignore` 实例，考虑缓存
   - `getPathsForPermissionCheck` 的 memoization 键需要仔细设计

4. **测试覆盖**：增加边界条件测试：
   - 各种 Windows 路径模式
   - 符号链接场景
   - 大小写混合路径
   - 特殊字符路径

5. **文档完善**：
   - 添加更多实现注释说明为什么某些检查顺序很重要
   - 为每个安全检查添加参考的安全问题描述

6. **配置化**：
   - 将危险文件/目录列表改为可配置
   - 允许用户自定义安全检查严格程度
