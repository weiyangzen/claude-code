# teamMemPaths.ts 研究文档

## 场景与职责

`teamMemPaths.ts` 是团队记忆系统的**安全路径验证中心**，专门处理团队记忆目录的路径计算和安全验证。相比 `paths.ts` 的通用路径处理，该模块专注于团队记忆特有的安全需求，特别是防止路径遍历和符号链接逃逸攻击。

### 核心职责
1. **团队记忆启用检查**：判断团队记忆功能是否启用
2. **团队记忆路径计算**：计算团队记忆目录和入口点路径
3. **路径遍历防护**：多层验证防止目录逃逸攻击
4. **符号链接解析**：检测并阻止通过符号链接的目录逃逸（PSR M22186）
5. **服务器密钥验证**：验证来自服务器的相对路径密钥

### 使用场景
- 团队记忆同步服务验证写入路径
- 文件系统操作前验证团队记忆路径安全性
- 团队记忆文件检测和分类

---

## 功能点目的

### 1. `isTeamMemoryEnabled()` - 团队记忆启用检查
**目的**：判断团队记忆功能是否启用

**依赖关系**：
- 团队记忆是自动记忆的子目录
- 因此需要 `isAutoMemoryEnabled()` 返回 true
- 且 feature flag `tengu_herring_clock` 启用

### 2. `getTeamMemPath()` - 团队记忆目录路径
**目的**：返回团队记忆目录路径

**路径格式**：`<autoMemPath>/team/`

### 3. `getTeamMemEntrypoint()` - 团队记忆入口点
**目的**：返回团队记忆 `MEMORY.md` 的完整路径

**路径格式**：`<autoMemPath>/team/MEMORY.md`

### 4. `isTeamMemPath()` - 团队记忆路径检查（字符串级）
**目的**：检查路径是否在团队记忆目录内（字符串级验证）

**实现**：
- 使用 `path.resolve()` 转换为绝对路径并消除 `..` 段
- 前缀匹配检查

**限制**：
- **不解析符号链接**
- 仅用于读取验证，不用于写入验证

### 5. `validateTeamMemWritePath()` - 写入路径验证
**目的**：验证绝对文件路径对于写入团队记忆目录是否安全

**验证流程**：
1. 空字节检查
2. 第一遍：`path.resolve()` 规范化并检查字符串级包含
3. 第二遍：`realpathDeepestExisting()` 解析最深存在祖先的符号链接
4. 验证解析后的路径仍在真实团队目录内

**返回**：解析后的绝对路径（如果有效）
**抛出**：`PathTraversalError`（如果无效）

### 6. `validateTeamMemKey()` - 服务器密钥验证
**目的**：验证来自服务器的相对路径密钥

**验证流程**：
1. `sanitizePathKey()` - 清理路径密钥（见下文）
2. 与团队目录拼接
3. 第一遍：`path.resolve()` 检查字符串级包含
4. 第二遍：符号链接解析和真实路径验证

**安全场景**：防止恶意服务器通过构造特殊密钥实现路径遍历

### 7. `sanitizePathKey()` - 路径密钥清理（内部）
**目的**：清理文件路径密钥，拒绝危险模式

**检查项**：
- 空字节（`\0`）
- URL 编码遍历（`%2e%2e%2f` = `../`）
- Unicode 规范化攻击（全角 `．．／`）
- 反斜杠（Windows 路径分隔符）
- 绝对路径（`/` 开头）

### 8. `realpathDeepestExisting()` - 最深存在祖先解析（内部）
**目的**：解析路径最深存在祖先的符号链接

**算法**：
- 从目标路径向上遍历目录树
- 对每层尝试 `realpath()`
- 处理 `ENOENT`（不存在）、`ENOTDIR`（非目录）、`ELOOP`（符号链接循环）
- 检测悬空符号链接（dangling symlink）

**安全意义**：
- `path.resolve()` 不解析符号链接
- 攻击者可放置指向目录外的符号链接
- 通过解析最深存在祖先，确保比较的是真实文件系统位置

### 9. `isRealPathWithinTeamDir()` - 真实路径包含检查（内部）
**目的**：检查真实（符号链接解析后）路径是否在真实团队记忆目录内

**特殊处理**：
- 如果团队目录不存在，返回 true（跳过检查）
- 安全依据：符号链接逃逸需要预存在的符号链接，而预存在符号链接需要目录存在

### 10. `isTeamMemFile()` - 团队记忆文件检查
**目的**：检查文件路径是否是启用的团队记忆文件

**条件**：`isTeamMemoryEnabled() && isTeamMemPath(filePath)`

---

## 具体技术实现

### 路径遍历防护架构

```
用户输入路径
  ↓
sanitizePathKey()  // 清理注入向量
  ↓
join(teamDir, key) // 拼接完整路径
  ↓
resolve()          // 第一遍：规范化，字符串级检查
  ↓
realpathDeepestExisting()  // 第二遍：符号链接解析
  ↓
isRealPathWithinTeamDir()  // 真实路径验证
```

### 符号链接解析算法

```typescript
async function realpathDeepestExisting(absolutePath: string): Promise<string> {
  const tail: string[] = []
  let current = absolutePath
  
  for (let parent = dirname(current); current !== parent; parent = dirname(current)) {
    try {
      const realCurrent = await realpath(current)
      // 重新拼接未存在的尾部
      return tail.length === 0
        ? realCurrent
        : join(realCurrent, ...tail.reverse())
    } catch (e: unknown) {
      const code = getErrnoCode(e)
      
      if (code === 'ENOENT') {
        // 可能是真正不存在，或悬空符号链接
        try {
          const st = await lstat(current)
          if (st.isSymbolicLink()) {
            throw new PathTraversalError(`Dangling symlink detected: "${current}"`)
          }
        } catch (lstatErr) {
          if (lstatErr instanceof PathTraversalError) throw lstatErr
          // lstat 也失败，安全向上遍历
        }
      } else if (code === 'ELOOP') {
        throw new PathTraversalError(`Symlink loop detected: "${current}"`)
      } else if (code !== 'ENOTDIR' && code !== 'ENAMETOOLONG') {
        // EACCES, EIO 等 - 无法验证，失败关闭
        throw new PathTraversalError(`Cannot verify path containment (${code}): "${current}"`)
      }
      
      tail.push(current.slice(parent.length + sep.length))
      current = parent
    }
  }
  
  return absolutePath  // 到达根目录未找到存在祖先
}
```

### 路径密钥清理

```typescript
function sanitizePathKey(key: string): string {
  // 1. 空字节检查
  if (key.includes('\0')) throw new PathTraversalError(...)
  
  // 2. URL 编码遍历检查
  let decoded: string
  try {
    decoded = decodeURIComponent(key)
  } catch {
    decoded = key
  }
  if (decoded !== key && (decoded.includes('..') || decoded.includes('/'))) {
    throw new PathTraversalError(`URL-encoded traversal: "${key}"`)
  }
  
  // 3. Unicode 规范化攻击检查
  const normalized = key.normalize('NFKC')
  if (normalized !== key && [...]) {
    throw new PathTraversalError(`Unicode-normalized traversal: "${key}"`)
  }
  
  // 4. 反斜杠检查
  if (key.includes('\\')) throw new PathTraversalError(...)
  
  // 5. 绝对路径检查
  if (key.startsWith('/')) throw new PathTraversalError(...)
  
  return key
}
```

---

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `paths.ts` | `getAutoMemPath()`, `isAutoMemoryEnabled()` |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `fs/promises` | `lstat`, `realpath` |
| `path` | `dirname`, `join`, `resolve`, `sep` |
| `../services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE()` |
| `../utils/errors.ts` | `getErrnoCode()` |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/services/teamMemorySync/index.ts` | `getTeamMemPath()`, `validateTeamMemKey()`, `PathTraversalError` |
| `src/services/teamMemorySync/watcher.ts` | `isTeamMemoryEnabled()`, `getTeamMemPath()` |
| `src/services/teamMemorySync/teamMemSecretGuard.ts` | `isTeamMemPath()` |
| `src/memdir/memdir.ts` | `isTeamMemoryEnabled()`, `getTeamMemPath()`（条件加载） |
| `src/memdir/teamMemPrompts.ts` | `getTeamMemPath()` |
| `src/utils/teamMemoryOps.ts` | `isTeamMemoryEnabled()`, `getTeamMemPath()`, `isTeamMemFile()` |
| `src/utils/memoryFileDetection.ts` | `isTeamMemFile()` |
| `src/utils/claudemd.ts` | `isTeamMemoryEnabled()`, `getTeamMemPath()`（条件加载） |

---

## 依赖与外部交互

### Feature Flags
| Flag | 用途 |
|------|------|
| `tengu_herring_clock` | 启用团队记忆功能 |

### 安全参考
- **PSR M22186**: 符号链接逃逸防护
- **PSR M22187**: Unicode 规范化攻击防护（向量4）

### 错误类型
```typescript
export class PathTraversalError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'PathTraversalError'
  }
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **TOCTOU 竞争条件**
   - `realpathDeepestExisting` 和实际文件操作之间有时间窗口
   - 攻击者可能在验证后、写入前切换符号链接
   - 缓解：文件系统操作使用 `O_NOFOLLOW` 等标志

2. **性能开销**
   - 符号链接解析需要多次系统调用
   - 批量操作时可能成为瓶颈

3. **平台差异**
   - Windows 的符号链接行为与 Unix 不同
   - `realpath` 在某些平台上的行为可能有差异

4. **悬空符号链接检测复杂性**
   - 需要区分"真正不存在"和"悬空符号链接"
   - `lstat` + `realpath` 组合逻辑复杂

### 边界情况

| 场景 | 行为 |
|------|------|
| 团队目录不存在 | `isRealPathWithinTeamDir` 返回 true（跳过检查） |
| 路径指向团队目录本身 | 允许（`realCandidate === realTeamDir`） |
| 符号链接指向团队目录内 | 允许（解析后仍在范围内） |
| 符号链接指向团队目录外 | 抛出 `PathTraversalError` |
| 符号链接循环 | 抛出 `PathTraversalError` |
| 无法访问的父目录 | 抛出 `PathTraversalError`（失败关闭） |

### 改进建议

1. **原子操作**
   ```typescript
   // 使用 O_NOFOLLOW 等标志确保操作时不跟随符号链接
   await fs.writeFile(path, content, { flag: 'wx' })
   ```

2. **缓存优化**
   ```typescript
   // 缓存符号链接解析结果
   const realpathCache = new Map<string, string>()
   ```

3. **更细粒度的错误**
   ```typescript
   export type PathValidationError =
     | { type: 'null_byte' }
     | { type: 'url_encoded_traversal'; decoded: string }
     | { type: 'unicode_normalization'; normalized: string }
     | { type: 'symlink_escape'; realPath: string; expectedDir: string }
     // ...
   ```

4. **异步验证流**
   ```typescript
   // 支持批量路径验证
   export async function validateTeamMemKeys(keys: string[]): Promise<ValidationResult[]>
   ```

5. **审计日志**
   ```typescript
   // 记录所有路径验证失败尝试
   logSecurityEvent('path_validation_failed', { attemptedPath, reason })
   ```

6. **测试覆盖**
   ```typescript
   // 应该测试的攻击场景
   describe('path traversal prevention', () => {
     it('rejects .. segments')
     it('rejects URL-encoded traversals')
     it('rejects Unicode-normalized traversals')
     it('rejects symlink escapes')
     it('rejects dangling symlinks')
     it('rejects symlink loops')
   })
   ```
