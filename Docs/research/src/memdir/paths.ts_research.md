# paths.ts 研究文档

## 场景与职责

`paths.ts` 是记忆系统的**路径解析和配置中心**，负责确定记忆文件的存储位置、验证路径安全性、并提供各种路径相关的工具函数。这是记忆系统的"文件系统网关"。

### 核心职责
1. **记忆目录路径解析**：根据多种来源确定自动记忆目录位置
2. **启用状态检查**：判断自动记忆功能是否启用
3. **路径验证**：验证路径安全性，防止路径遍历攻击
4. **提取模式检查**：判断记忆提取后台代理是否激活
5. **每日日志路径**：为 KAIROS 模式生成日期命名的日志路径

### 使用场景
- 系统启动时确定记忆存储位置
- 文件系统操作前验证路径
- 记忆功能启用/禁用判断
- 团队记忆路径计算（基于自动记忆路径）

---

## 功能点目的

### 1. `isAutoMemoryEnabled()` - 自动记忆启用检查
**目的**：判断自动记忆功能是否启用

**优先级链**（第一个定义的值获胜）：
1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` 环境变量（`1/true` → 关闭，`0/false` → 开启）
2. `CLAUDE_CODE_SIMPLE`（`--bare`）→ 关闭
3. `CLAUDE_CODE_REMOTE` 且无 `CLAUDE_CODE_REMOTE_MEMORY_DIR` → 关闭
4. `settings.json` 中的 `autoMemoryEnabled`（支持项目级退出）
5. 默认：启用

### 2. `isExtractModeActive()` - 提取模式检查
**目的**：判断记忆提取后台代理是否会在此会话运行

**条件**：
- Feature flag `tengu_passport_quail` 启用
- 且（非非交互式会话 或 `tengu_slate_thimble` 启用）

### 3. `getMemoryBaseDir()` - 记忆基础目录
**目的**：返回持久记忆存储的基础目录

**解析顺序**：
1. `CLAUDE_CODE_REMOTE_MEMORY_DIR` 环境变量（显式覆盖，CCR 中设置）
2. `~/.claude`（默认配置主目录）

### 4. `getAutoMemPath()` - 自动记忆目录路径
**目的**：返回自动记忆目录的完整路径

**解析顺序**：
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 环境变量（Cowork 使用的完整路径覆盖）
2. `settings.json` 中的 `autoMemoryDirectory`（仅受信任来源）
3. `<memoryBase>/projects/<sanitized-git-root>/memory/`

**Memoized**：基于 `projectRoot` 缓存，避免重复计算

### 5. `getAutoMemDailyLogPath()` - 每日日志路径
**目的**：为给定日期（默认为今天）生成每日日志文件路径

**路径格式**：`<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md`

**用途**：KAIROS 助手模式使用追加式日志而非维护 `MEMORY.md`

### 6. `getAutoMemEntrypoint()` - 自动记忆入口点
**目的**：返回 `MEMORY.md` 的完整路径

### 7. `isAutoMemPath()` - 自动记忆路径检查
**目的**：检查绝对路径是否在自动记忆目录内

**安全考虑**：
- 规范化路径（`normalize`）防止 `..` 段绕过
- 使用前缀匹配

### 8. `validateMemoryPath()` - 路径验证（内部）
**目的**：验证候选记忆目录路径的安全性

**安全检查**：
- 相对路径（`!isAbsolute`）：拒绝 `"../foo"`
- 根/近根路径（`length < 3`）：拒绝 `"/"`、`"/a"`
- Windows 驱动器根（`C:` 正则）：拒绝 `"C:\"`
- UNC 路径（`\\\\server\share`）：拒绝网络路径
- 空字节：拒绝包含 `\0` 的路径

**Tilde 扩展**：
- `~/` 或 `~\` 前缀支持扩展为用户主目录
- 拒绝 `"~"`、`"~/"`、`"~/."`、`"~/.."` 等危险扩展

---

## 具体技术实现

### 路径解析流程

```
getAutoMemPath()
  ↓
getAutoMemPathOverride() ?? getAutoMemPathSetting()
  ↓
如果存在覆盖值 → 返回覆盖路径
  ↓
join(getMemoryBaseDir(), 'projects', sanitizePath(getAutoMemBase()), 'memory') + sep
```

### 路径验证实现

```typescript
function validateMemoryPath(
  raw: string | undefined,
  expandTilde: boolean,
): string | undefined {
  if (!raw) return undefined
  
  let candidate = raw
  
  // Tilde 扩展（仅 settings.json 路径）
  if (expandTilde && (candidate.startsWith('~/') || candidate.startsWith('~\\'))) {
    const rest = candidate.slice(2)
    const restNorm = normalize(rest || '.')
    if (restNorm === '.' || restNorm === '..') return undefined
    candidate = join(homedir(), rest)
  }
  
  // 规范化并去除尾部分隔符
  const normalized = normalize(candidate).replace(/[/\\]+$/, '')
  
  // 安全检查
  if (
    !isAbsolute(normalized) ||
    normalized.length < 3 ||
    /^[A-Za-z]:$/.test(normalized) ||
    normalized.startsWith('\\\\') ||
    normalized.startsWith('//') ||
    normalized.includes('\0')
  ) {
    return undefined
  }
  
  return (normalized + sep).normalize('NFC')
}
```

### Memoization 实现

```typescript
export const getAutoMemPath = memoize(
  (): string => {
    // ... 路径计算逻辑
  },
  () => getProjectRoot(),  // 缓存键基于 projectRoot
)
```

---

## 关键代码路径与文件引用

### 内部依赖
无（纯工具模块）

### 外部依赖
| 文件 | 用途 |
|------|------|
| `lodash-es/memoize.js` | 路径计算缓存 |
| `os` | `homedir()` |
| `path` | 路径操作（`join`, `normalize`, `sep`, `isAbsolute`） |
| `../bootstrap/state.ts` | `getIsNonInteractiveSession()`, `getProjectRoot()` |
| `../services/analytics/growthbook.ts` | `getFeatureValue_CACHED_MAY_BE_STALE()` |
| `../utils/envUtils.ts` | `getClaudeConfigHomeDir()`, `isEnvDefinedFalsy()`, `isEnvTruthy()` |
| `../utils/git.ts` | `findCanonicalGitRoot()` |
| `../utils/path.ts` | `sanitizePath()` |
| `../utils/settings/settings.ts` | `getInitialSettings()`, `getSettingsForSource()` |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/memdir/memdir.ts` | `getAutoMemPath()`, `isAutoMemoryEnabled()` |
| `src/memdir/teamMemPaths.ts` | `getAutoMemPath()`, `isAutoMemoryEnabled()` |
| `src/memdir/teamMemPrompts.ts` | `getAutoMemPath()` |
| `src/utils/claudemd.ts` | `getAutoMemEntrypoint()`, `isAutoMemoryEnabled()` |
| `src/utils/memoryFileDetection.ts` | `isAutoMemPath()`, `getAutoMemPath()` |
| `src/utils/messages.ts` | `isAutoMemoryEnabled()` |
| `src/utils/config.ts` | `getMemoryBaseDir()` |
| `src/services/extractMemories/extractMemories.ts` | `getAutoMemPath()`, `isAutoMemoryEnabled()`, `isAutoMemPath()` |
| `src/services/autoDream/autoDream.ts` | `getAutoMemPath()` |
| `src/services/teamMemorySync/index.ts` | `getAutoMemPath()`（间接） |
| `src/tools/FileReadTool/FileReadTool.ts` | `isAutoMemPath()`（间接） |
| `src/utils/permissions/filesystem.ts` | `isAutoMemPath()` |

---

## 依赖与外部交互

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动记忆 |
| `CLAUDE_CODE_SIMPLE` | 简化模式（禁用记忆） |
| `CLAUDE_CODE_REMOTE` | 远程模式 |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | 远程记忆目录 |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | Cowork 路径覆盖 |

### Feature Flags
| Flag | 用途 |
|------|------|
| `tengu_passport_quail` | 启用记忆提取 |
| `tengu_slate_thimble` | 非交互式会话也启用提取 |

### Settings.json
| 键 | 来源 | 说明 |
|----|------|------|
| `autoMemoryEnabled` | policy/flag/local/user | 启用/禁用自动记忆 |
| `autoMemoryDirectory` | policy/flag/local/user | 自定义记忆目录（**排除 projectSettings**） |

### 安全考虑

1. **ProjectSettings 排除**：
   - `autoMemoryDirectory` 不接受 `projectSettings`
   - 防止恶意仓库通过提交 `.claude/settings.json` 获取敏感目录写入权限

2. **路径遍历防护**：
   - `validateMemoryPath` 多层检查
   - `isAutoMemPath` 规范化后前缀匹配

3. **Tilde 扩展限制**：
   - 仅 `settings.json` 路径支持 `~/` 扩展
   - 环境变量路径不支持（应由调用方传递绝对路径）

---

## 风险、边界与改进建议

### 已知风险

1. **缓存失效**
   - `getAutoMemPath` 是 memoized 的
   - 测试需要手动清除缓存
   - 运行时环境变量/设置变化不会触发重新计算

2. **Git 根目录变化**
   - 使用 `findCanonicalGitRoot` 确保所有工作树共享同一记忆目录
   - 但 Git 仓库结构变化可能导致记忆"丢失"

3. **路径规范化差异**
   - 不同平台的 `normalize` 行为可能不同
   - Windows 的 `\` vs Unix 的 `/`

4. **设置来源复杂性**
   - 四个设置来源（policy/flag/local/user）
   - 优先级逻辑复杂，容易出错

### 边界情况

| 场景 | 行为 |
|------|------|
| 无 Git 仓库 | 使用 `getProjectRoot()` 作为基础 |
| 路径验证失败 | 返回 `undefined`，回退到默认路径 |
| 环境变量为 `""` | 视为未设置 |
| 环境变量为 `"0"` | `isEnvDefinedFalsy` 识别为 false |
| 路径包含 Unicode | NFC 规范化处理 |

### 改进建议

1. **配置对象模式**
   ```typescript
   // 替代多个独立函数
   export interface MemoryPathConfig {
     baseDir: string
     autoMemPath: string
     isEnabled: boolean
     isExtractModeActive: boolean
   }
   export function getMemoryPathConfig(): MemoryPathConfig
   ```

2. **更好的缓存控制**
   ```typescript
   // 显式缓存管理
   export function clearMemoryPathCache(): void
   export function getAutoMemPath(options?: { skipCache?: boolean }): string
   ```

3. **路径验证增强**
   ```typescript
   // 返回详细错误信息
   export type PathValidationResult = 
     | { valid: true; path: string }
     | { valid: false; reason: PathValidationError }
   ```

4. **原子设置读取**
   ```typescript
   // 避免多次调用 getSettingsForSource
   const allSettings = getAllSettingsSources()
   const autoMemoryDirectory = allSettings.find(s => s.autoMemoryDirectory)?.autoMemoryDirectory
   ```

5. **路径监控**
   ```typescript
   // 检测路径变化并发出事件
   export function watchMemoryPathChanges(callback: (oldPath: string, newPath: string) => void): void
   ```

6. **文档化优先级**
   ```typescript
   // 在代码中明确展示优先级链
   const ENABLE_PRIORITY_CHAIN = [
     { source: 'env:CLAUDE_CODE_DISABLE_AUTO_MEMORY', check: () => ... },
     { source: 'env:CLAUDE_CODE_SIMPLE', check: () => ... },
     // ...
   ] as const
   ```
