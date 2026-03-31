# src/commands/release-notes/release-notes.ts 研究文档

## 场景与职责

该文件是 Claude Code CLI 中 `/release-notes` 命令的实际实现模块，负责获取、解析和格式化产品发布说明（changelog）。它实现了智能的发布说明获取策略，优先尝试从网络获取最新内容，同时提供本地缓存作为降级方案。

**核心职责：**
1. 异步获取远程发布说明（GitHub 上的 CHANGELOG.md）
2. 管理本地缓存的发布说明数据
3. 格式化发布说明为可读的文本输出
4. 处理网络超时和错误，确保命令始终可用

## 功能点目的

### 1. 智能发布说明获取
实现分层获取策略：
- **首选**: 尝试从 GitHub 获取最新发布说明（500ms 超时）
- **降级**: 使用本地缓存的发布说明
- **兜底**: 若都不可用，返回 CHANGELOG 链接

### 2. 非阻塞用户体验
- 使用 `Promise.race` 实现超时控制，避免用户长时间等待
- 网络请求失败静默处理，不影响命令可用性

### 3. 版本感知显示
与 `src/utils/releaseNotes.ts` 协作，根据用户上次查看的版本智能筛选需要显示的更新内容。

## 具体技术实现

### 关键数据结构

```typescript
// 发布说明条目格式: [版本号, 该版本的更新项数组]
type ReleaseNoteEntry = [string, string[]];

// 本地命令结果类型
interface LocalCommandResult {
  type: 'text' | 'compact' | 'skip';
  value?: string;
  // ... 其他字段
}
```

### 核心函数实现

#### 1. 格式化函数
```typescript
function formatReleaseNotes(notes: Array<[string, string[]]>): string {
  return notes
    .map(([version, notes]) => {
      const header = `Version ${version}:`
      const bulletPoints = notes.map(note => `· ${note}`).join('\n')
      return `${header}\n${bulletPoints}`
    })
    .join('\n\n')
}
```

**输出示例：**
```
Version 0.2.45:
· Added support for custom keybindings
· Improved file search performance

Version 0.2.44:
· Fixed memory leak in long sessions
· Updated default model to Claude 3.5 Sonnet
```

#### 2. 主执行函数 `call()`

```typescript
export async function call(): Promise<LocalCommandResult> {
  // 阶段 1: 尝试快速获取最新发布说明
  let freshNotes: Array<[string, string[]]> = []
  
  try {
    const timeoutPromise = new Promise<void>((_, reject) => {
      setTimeout(rej => rej(new Error('Timeout')), 500, reject)
    })
    
    await Promise.race([fetchAndStoreChangelog(), timeoutPromise])
    freshNotes = getAllReleaseNotes(await getStoredChangelog())
  } catch {
    // 静默处理失败 - 使用缓存数据
  }
  
  // 阶段 2: 返回获取到的新数据
  if (freshNotes.length > 0) {
    return { type: 'text', value: formatReleaseNotes(freshNotes) }
  }
  
  // 阶段 3: 尝试使用缓存数据
  const cachedNotes = getAllReleaseNotes(await getStoredChangelog())
  if (cachedNotes.length > 0) {
    return { type: 'text', value: formatReleaseNotes(cachedNotes) }
  }
  
  // 阶段 4: 兜底 - 返回链接
  return {
    type: 'text',
    value: `See the full changelog at: ${CHANGELOG_URL}`,
  }
}
```

### 超时控制机制

使用 `Promise.race` 实现竞态超时：
```typescript
const timeoutPromise = new Promise<void>((_, reject) => {
  setTimeout(rej => rej(new Error('Timeout')), 500, reject)
})

await Promise.race([fetchAndStoreChangelog(), timeoutPromise])
```

- **超时时间**: 500ms（硬编码）
- **行为**: 超时后静默失败，继续使用缓存数据

## 关键代码路径与文件引用

### 当前文件
- **路径**: `src/commands/release-notes/release-notes.ts`
- **行数**: 50 行
- **导出**: `call` 函数（`LocalCommandCall` 类型）

### 依赖工具模块

| 文件路径 | 导入内容 | 用途 |
|----------|----------|------|
| `src/types/command.ts` | `LocalCommandResult` | 返回类型定义 |
| `src/utils/releaseNotes.ts` | `CHANGELOG_URL` | GitHub CHANGELOG 链接常量 |
| `src/utils/releaseNotes.ts` | `fetchAndStoreChangelog` | 获取并缓存远程 changelog |
| `src/utils/releaseNotes.ts` | `getAllReleaseNotes` | 解析所有发布说明 |
| `src/utils/releaseNotes.ts` | `getStoredChangelog` | 读取本地缓存的 changelog |

### 被调用关系

```
src/commands/release-notes/index.ts
    ↓ (通过 load() 动态导入)
本文件 (release-notes.ts)
    ↓ 调用
src/utils/releaseNotes.ts 中的工具函数
```

### 完整调用链

```
用户执行 /release-notes
    ↓
commands.ts 路由到 releaseNotes 命令
    ↓
index.ts 的 load() 动态导入本文件
    ↓
call() 函数执行
    ├── 尝试 fetchAndStoreChangelog() (500ms 超时)
    │   └── 成功 → 获取最新数据
    │   └── 超时/失败 → 静默忽略
    ├── 尝试使用 freshNotes
    ├── 尝试使用 cachedNotes
    └── 兜底返回 CHANGELOG_URL
```

## 依赖与外部交互

### 工具函数详解（src/utils/releaseNotes.ts）

#### 1. `fetchAndStoreChangelog()`
```typescript
export async function fetchAndStoreChangelog(): Promise<void>
```
- 从 `https://raw.githubusercontent.com/anthropics/claude-code/refs/heads/main/CHANGELOG.md` 获取原始内容
- 存储到 `~/.claude/cache/changelog.md`
- 在非交互式模式或隐私模式下跳过

#### 2. `getStoredChangelog()`
```typescript
export async function getStoredChangelog(): Promise<string>
```
- 读取本地缓存的 changelog
- 维护内存缓存避免重复磁盘读取

#### 3. `getAllReleaseNotes()`
```typescript
export function getAllReleaseNotes(
  changelogContent: string = getStoredChangelogFromMemory()
): Array<[string, string[]]>
```
- 解析 markdown 格式的 changelog
- 返回按版本排序的发布说明数组
- 版本按 semver 排序（旧版本在前）

### 网络交互

| 目标 | URL | 方法 | 用途 |
|------|-----|------|------|
| GitHub Raw | `raw.githubusercontent.com/anthropics/claude-code/refs/heads/main/CHANGELOG.md` | GET | 获取最新 changelog |

### 文件系统交互

| 路径 | 操作 | 说明 |
|------|------|------|
| `~/.claude/cache/changelog.md` | 读取 | 本地缓存文件 |

## 风险、边界与改进建议

### 潜在风险

1. **硬编码超时时间**
   - **风险**: 500ms 超时在网络较慢环境下可能导致频繁降级
   - **影响**: 用户总是看到缓存数据而非最新内容
   - **建议**: 考虑根据网络状况动态调整，或允许用户配置

2. **静默错误处理**
   - **风险**: `catch {}` 块完全静默，无法追踪获取失败的原因
   - **影响**: 调试困难，无法区分网络问题、超时还是解析错误
   - **建议**: 添加日志记录（debug 级别）用于故障排查

3. **无重试机制**
   - **风险**: 单次失败即放弃，临时网络抖动影响体验
   - **建议**: 添加指数退避重试（1-2 次）

### 边界情况

1. **空缓存场景**
   - 首次使用或缓存被清除时，`getStoredChangelog()` 返回空字符串
   - `getAllReleaseNotes('')` 返回空数组
   - 最终返回 GitHub 链接作为兜底

2. **changelog 格式变更**
   - 若 GitHub 上的 CHANGELOG.md 格式变更，解析可能失败
   - `parseChangelog` 有 try-catch 保护，返回空对象

3. **并发调用**
   - 命令可被多次调用，`fetchAndStoreChangelog` 内部有非交互式检查
   - 多次快速调用可能导致重复网络请求

### 改进建议

1. **添加诊断日志**
   ```typescript
   import { logForDebugging } from '../../utils/debug.js';
   
   try {
     // ... fetch logic
   } catch (error) {
     logForDebugging('Release notes fetch failed:', error);
     // 继续静默处理
   }
   ```

2. **可配置超时**
   ```typescript
   const FETCH_TIMEOUT = 
     parseInt(process.env.CLAUDE_CHANGELOG_TIMEOUT_MS) || 500;
   ```

3. **添加版本信息到输出**
   ```typescript
   return { 
     type: 'text', 
     value: `Release Notes (cached)\n\n${formatReleaseNotes(cachedNotes)}` 
   };
   ```

4. **考虑添加强制刷新选项**
   ```typescript
   // 支持 /release-notes --refresh 强制跳过缓存
   export async function call(args: string): Promise<LocalCommandResult> {
     const forceRefresh = args.includes('--refresh');
     // ... 相应逻辑
   }
   ```

5. **内存缓存优化**
   当前每次调用都重新读取缓存文件，可考虑在模块级别维护缓存：
   ```typescript
   let cachedNotes: Array<[string, string[]]> | null = null;
   
   export async function call(): Promise<LocalCommandResult> {
     if (cachedNotes) {
       return { type: 'text', value: formatReleaseNotes(cachedNotes) };
     }
     // ... 原有逻辑
   }
   ```

### 测试建议

1. **单元测试场景**
   - 网络成功且 500ms 内返回
   - 网络超时（>500ms）
   - 网络立即失败（断网）
   - 缓存存在且有效
   - 缓存为空/损坏

2. **集成测试**
   - 验证与 `src/utils/releaseNotes.ts` 的集成
   - 验证命令注册和加载流程
