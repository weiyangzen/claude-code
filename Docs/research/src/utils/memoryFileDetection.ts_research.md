# memoryFileDetection.ts 研究文档

## 场景与职责

本模块负责检测和分类 Claude Code 管理的内存文件路径。核心职责包括：

1. **会话文件检测**：识别会话内存文件（session-memory）和会话转录文件（session_transcript）
2. **自动内存检测**：检测路径是否在自动内存（memdir）目录中
3. **代理内存检测**：识别代理内存目录中的文件
4. **作用域判定**：确定内存文件属于个人还是团队作用域
5. **Shell 命令分析**：检测 Shell 命令是否针对内存文件

该模块是内存管理系统的安全网关，用于权限控制、UI 折叠（collapse）和遥测分类。

## 功能点目的

### 1. `detectSessionFileType()` - 会话文件类型检测
- **目的**：检测路径是否为 Claude 会话相关文件
- **识别类型**：
  - `session_memory`：`~/.claude/session-memory/*.md`
  - `session_transcript`：`~/.claude/projects/*.jsonl`
- **跨平台处理**：统一使用正斜杠，Windows 下大小写不敏感比较

### 2. `detectSessionPatternType()` - 会话模式检测
- **目的**：检测 glob/模式字符串是否表示会话文件访问意图
- **场景**：Grep/Glob 工具检查模式而非实际文件路径
- **启发式**：检查模式字符串中是否包含 `session-memory`、`.jsonl` 等关键字

### 3. `isAutoMemFile()` / `memoryScopeForPath()` - 自动内存检测
- **目的**：检测路径是否在自动内存目录中
- **作用域判定**：
  - `team`：团队内存（TEAMMEM feature 启用时）
  - `personal`：个人自动内存
  - `null`：不在内存目录中

### 4. `isAutoManagedMemoryFile()` - 自动管理内存文件检测
- **目的**：区分 Claude 管理的内存文件和用户管理的指令文件
- **包含**：auto-memory、agent memory、session memory/transcripts
- **排除**：CLAUDE.md、CLAUDE.local.md、.claude/rules/*.md（用户管理）
- **用途**：UI 折叠逻辑中，用户管理文件应显示完整差异

### 5. `isMemoryDirectory()` - 内存目录检测
- **目的**：检测目录路径是否为内存相关目录
- **场景**：Grep/Glob 工具接收目录路径而非特定文件
- **安全检查**：使用 `normalize()` 防止路径遍历绕过

### 6. `isShellCommandTargetingMemory()` - Shell 命令内存检测
- **目的**：检测 Bash/PowerShell 命令是否针对内存文件
- **实现**：
  - 提取命令中的绝对路径标记
  - 支持 Unix 路径（/foo/bar）、Windows 路径（C:\foo）、MinGW 路径（/c/foo）
  - Windows 下将 MinGW 路径转换为原生路径

### 7. `isAutoManagedMemoryPattern()` - 内存模式检测
- **目的**：检测 glob 模式是否仅针对自动管理内存文件
- **用途**：折叠徽章逻辑，用户管理文件不计为"内存"操作

## 具体技术实现

### 关键流程

#### 路径标准化流程

```
输入路径
    ↓
toPosix(): 反斜杠转斜杠
    ↓
toComparable(): Windows 下转小写
    ↓
normalize(): 解析 . 和 .. 段
    ↓
与已知内存路径前缀比较
```

#### Shell 命令分析流程

```
输入命令字符串
    ↓
快速检查：是否包含 configDir/memoryBase/autoMemDir？
    ↓
否 → 返回 false
    ↓
是 → 提取绝对路径标记
    ↓
Windows? → 转换 MinGW /c/... → C:\...
    ↓
逐个路径检查 isAutoManagedMemoryFile() / isMemoryDirectory()
    ↓
任一匹配 → 返回 true
```

### 数据结构

```typescript
// 内存作用域类型
export type MemoryScope = 'personal' | 'team'

// 会话文件类型
export type SessionFileType = 'session_memory' | 'session_transcript' | null

// 关键常量
const IS_WINDOWS = process.platform === 'win32'
```

### 路径比较策略

| 平台 | 标准化步骤 |
|------|------------|
| 所有平台 | `split(win32.sep).join(posix.sep)` |
| Windows | 额外 `toLowerCase()` |
| 比较时 | `startsWith()` 前缀匹配 |

### 关键代码路径

1. **会话文件检测**：
   ```typescript
   const normalized = toComparable(filePath)
   const configDirCmp = toComparable(configDir)
   if (!normalized.startsWith(configDirCmp)) return null
   if (normalized.includes('/session-memory/') && normalized.endsWith('.md'))
     return 'session_memory'
   ```

2. **团队内存检测（条件编译）**：
   ```typescript
   const teamMemPaths = feature('TEAMMEM')
     ? require('../memdir/teamMemPaths.js')
     : null
   if (feature('TEAMMEM') && teamMemPaths!.isTeamMemFile(filePath))
     return 'team'
   ```

3. **路径标记提取**：
   ```typescript
   // 匹配 Unix 绝对路径、Windows 驱动器路径、MinGW 路径
   const matches = command.match(/(?:[A-Za-z]:[/\\]|\/)[^\s'"]+/g)
   ```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `bun:bundle` | Feature flag 检查 |
| `path` | 路径操作（normalize, posix, win32） |
| `../memdir/paths.js` | 自动内存路径获取 |
| `../tools/AgentTool/agentMemory.js` | 代理内存路径检测 |
| `./envUtils.js` | Claude 配置目录获取 |
| `./windowsPaths.js` | Windows/MinGW 路径转换 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/tools/FileReadTool/FileReadTool.ts` | 读取权限检查 |
| `src/utils/collapseReadSearch.ts` | 折叠逻辑判断 |
| `src/utils/sessionFileAccessHooks.ts` | 会话文件访问钩子 |
| `src/components/FeedbackSurvey/useMemorySurvey.tsx` | 内存使用调查 |

### 条件编译依赖

- `../memdir/teamMemPaths.js`：仅在 `feature('TEAMMEM')` 为 true 时加载
- 使用 `require()` 动态导入，支持 tree-shaking

## 风险、边界与改进建议

### 已知风险

1. **路径遍历攻击**
   - 风险：`../../` 可能绕过路径前缀检查
   - 缓解：使用 `normalize()` 解析路径段
   - 潜在问题：符号链接可能绕过检查

2. **大小写敏感性问题**
   - Windows：使用 `toLowerCase()` 进行大小写不敏感比较
   - 风险：某些文件系统（如 macOS APFS）可配置大小写敏感
   - 现状：Windows 特定处理，其他平台保持敏感

3. **MinGW 路径歧义**
   - 风险：`/c/foo` 在 Linux 可能是合法路径（非 MinGW）
   - 缓解：仅在 Windows 平台进行 MinGW 转换
   - 潜在问题：WSL 路径可能被误处理

4. **TeamMem 功能耦合**
   - 风险：条件编译增加代码复杂度
   - 缓解：使用动态 require 和 feature flag
   - 潜在问题：测试覆盖困难

### 边界情况

| 场景 | 行为 |
|------|------|
| 空路径 | 返回 null/false |
| 相对路径 | 不匹配（仅检查绝对路径） |
| 符号链接 | 跟随链接后检查（依赖外部 normalize） |
| UNC 路径 | 支持 `\\server\share` 格式 |
| 自定义内存目录 | 通过 `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 支持 |
| 大小写混合 | Windows 下不敏感，其他平台敏感 |
| 路径含空格 | 正常处理 |
| 路径含特殊字符 | 依赖 normalize() 处理 |

### 改进建议

1. **缓存路径检查结果**
   - 当前：每次调用重新计算
   - 建议：使用 `memoizeWithLRU` 缓存结果
   - 注意：路径可能动态变化，需设置合理 TTL

2. **符号链接安全**
   - 当前：依赖外部 normalize
   - 建议：显式解析符号链接后检查
   - 实现：`fs.realpath()` 或等效操作

3. **更精确的 MinGW 检测**
   - 当前：仅基于平台判断
   - 建议：检测 Git Bash 环境变量（`MSYSTEM` 等）
   - 考虑：支持 MSYS2、Cygwin 等多种环境

4. **配置化路径模式**
   - 当前：硬编码路径模式
   - 建议：支持 settings.json 配置额外内存路径
   - 安全：限制配置来源（仅用户设置）

5. **统一路径处理**
   - 当前：多处重复标准化逻辑
   - 建议：提取 `PathMatcher` 类封装比较逻辑
   - 好处：便于测试和维护

6. **遥测增强**
   - 当前：无直接遥测
   - 建议：
     - 记录内存文件访问频率
     - 记录 Shell 命令内存检测命中率
     - 识别潜在的安全绕过尝试

7. **文档和注释**
   - 当前：部分复杂逻辑缺少详细注释
   - 建议：
     - 添加路径比较示例
     - 说明各检测函数的优先级
     - 记录已知限制

8. **测试覆盖**
   - 建议：
     - 跨平台路径测试矩阵
     - 符号链接场景测试
     - 边界路径（空、根目录、超长路径）
