# iTermBackup.ts 研究文档

## 场景与职责

`iTermBackup.ts` 是 Claude Code CLI 的 **iTerm2 终端设置备份与恢复工具**。该模块专门处理 iTerm2（macOS 上的流行终端模拟器）的偏好设置文件备份和恢复逻辑，确保在 Claude Code 修改 iTerm2 设置后，能够在异常情况下恢复用户的原始配置。

### 核心使用场景

1. **设置修改前的备份**：在 Claude Code 修改 iTerm2 设置（如启用粘贴大括号模式）前，自动备份原始 `com.googlecode.iterm2.plist` 文件
2. **异常恢复**：如果设置修改过程中断（如用户强制退出、系统崩溃），下次启动时自动检测并恢复备份
3. **状态标记**：使用全局配置标记备份/恢复状态，防止重复操作

### iTerm2 设置文件位置

```
~/Library/Preferences/com.googlecode.iterm2.plist
```

这是 macOS 标准的 plist 偏好设置文件，包含 iTerm2 的所有配置信息。

---

## 功能点目的

### 1. 标记设置完成 (`markITerm2SetupComplete`)

**设计目标**：在设置修改成功完成后，清除进行中的标记，防止不必要的恢复操作。

**实现逻辑**：
```typescript
export function markITerm2SetupComplete(): void {
  saveGlobalConfig(current => ({
    ...current,
    iterm2SetupInProgress: false,
  }))
}
```

### 2. 备份恢复检查 (`checkAndRestoreITerm2Backup`)

**设计目标**：启动时检测是否存在未完成的状态和有效的备份文件，自动恢复原始设置。

**恢复条件**：
1. `iterm2SetupInProgress` 标记为 `true`（上次设置未完成）
2. `iterm2BackupPath` 存在且指向有效文件
3. 备份文件可通过 `stat` 访问

**恢复结果类型**：
```typescript
type RestoreResult =
  | { status: 'restored' | 'no_backup' }
  | { status: 'failed'; backupPath: string }
```

---

## 具体技术实现

### 核心函数实现

#### 获取恢复信息
```typescript
function getIterm2RecoveryInfo(): {
  inProgress: boolean
  backupPath: string | null
} {
  const config = getGlobalConfig()
  return {
    inProgress: config.iterm2SetupInProgress ?? false,
    backupPath: config.iterm2BackupPath || null,
  }
}
```

#### iTerm2 plist 文件路径
```typescript
function getITerm2PlistPath(): string {
  return join(
    homedir(),
    'Library',
    'Preferences',
    'com.googlecode.iterm2.plist',
  )
}
```

#### 备份恢复主逻辑
```typescript
export async function checkAndRestoreITerm2Backup(): Promise<RestoreResult> {
  const { inProgress, backupPath } = getIterm2RecoveryInfo()
  
  // 无进行中的设置操作
  if (!inProgress) {
    return { status: 'no_backup' }
  }

  // 无备份路径
  if (!backupPath) {
    markITerm2SetupComplete()
    return { status: 'no_backup' }
  }

  // 验证备份文件存在
  try {
    await stat(backupPath)
  } catch {
    markITerm2SetupComplete()
    return { status: 'no_backup' }
  }

  // 执行恢复
  try {
    await copyFile(backupPath, getITerm2PlistPath())
    markITerm2SetupComplete()
    return { status: 'restored' }
  } catch (restoreError) {
    logError(new Error(`Failed to restore iTerm2 settings with: ${restoreError}`))
    markITerm2SetupComplete()
    return { status: 'failed', backupPath }
  }
}
```

### 状态流转图

```
[正常启动] --无进行中标记--> [no_backup]
    |
    v
[检查 iterm2SetupInProgress]
    |
    +-- true --+--> [检查备份文件存在]
    |           |           |
    |           v           v
    |      [不存在]    [存在]
    |           |           |
    |           v           v
    |    [no_backup]  [copyFile]
    |                       |
    |                       v
    |                  [restored]
    |
    +-- false --> [no_backup]
```

---

## 关键代码路径与文件引用

### 导出位置
- **文件**：`src/utils/iTermBackup.ts`
- **导出函数**：
  - `markITerm2SetupComplete()` - 标记设置完成
  - `checkAndRestoreITerm2Backup()` - 检查并恢复备份

### 调用方分布

| 文件路径 | 使用函数 | 使用场景 |
|---------|---------|---------|
| `src/setup.ts` | `checkAndRestoreITerm2Backup` | 应用启动时恢复中断的设置修改，行 41 |

### 调用示例（setup.ts）

```typescript
import { checkAndRestoreITerm2Backup } from './utils/iTermBackup.js'

export async function setup(...): Promise<void> {
  // ... 其他初始化代码 ...
  
  // 恢复可能中断的 iTerm2 设置修改
  await checkAndRestoreITerm2Backup()
  
  // ... 继续初始化 ...
}
```

### 依赖导入

```typescript
import { copyFile, stat } from 'fs/promises'    // 文件操作
import { homedir } from 'os'                     // 获取用户主目录
import { join } from 'path'                      // 路径拼接
import { getGlobalConfig, saveGlobalConfig } from './config.js'  // 配置管理
import { logError } from './log.js'              // 错误日志
```

---

## 依赖与外部交互

### Node.js 内置模块

| 模块 | 用途 |
|------|------|
| `fs/promises` | `copyFile` - 恢复备份；`stat` - 验证备份存在 |
| `os` | `homedir()` - 获取用户主目录路径 |
| `path` | `join()` - 构建 plist 文件路径 |

### 内部依赖

| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `utils/config.js` | `getGlobalConfig`, `saveGlobalConfig` | 读写备份状态和路径 |
| `utils/log.js` | `logError` | 记录恢复失败错误 |

### 全局配置字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `iterm2SetupInProgress` | `boolean` | 标记是否有正在进行的设置修改 |
| `iterm2BackupPath` | `string` | 备份文件的完整路径 |

---

## 风险、边界与改进建议

### 已知风险

1. **备份文件残留**
   - 恢复成功后，备份文件不会被自动删除
   - 长期积累可能占用磁盘空间
   - **缓解**：备份文件通常较小（plist 文件一般 < 1MB）

2. **并发修改冲突**
   - 如果用户在 Claude Code 运行时手动修改 iTerm2 设置，可能导致配置冲突
   - 恢复操作会覆盖用户的修改
   - **缓解**：设置修改通常在启动早期完成，用户并发修改概率低

3. **权限问题**
   - `~/Library/Preferences/` 目录可能需要特定权限
   - `copyFile` 可能因权限不足失败
   - **缓解**：错误被捕获并记录，不会阻塞启动流程

4. **跨设备同步问题**
   - 如果用户通过 iCloud 同步 iTerm2 设置，备份恢复可能导致同步冲突
   - **缓解**：iTerm2 的 plist 文件通常不参与 iCloud 同步

### 边界情况

| 场景 | 处理 |
|------|------|
| 无进行中标记 | 直接返回 `no_backup`，不执行任何操作 |
| 无备份路径 | 清除标记，返回 `no_backup` |
| 备份文件不存在 | 清除标记，返回 `no_backup` |
| 恢复成功 | 清除标记，返回 `restored` |
| 恢复失败 | 清除标记，记录错误，返回 `failed` |
| iTerm2 未安装 | 恢复操作会失败（文件不存在），返回 `failed` |

### 改进建议

1. **备份文件清理**
   ```typescript
   // 恢复成功后删除备份文件
   try {
     await copyFile(backupPath, getITerm2PlistPath())
     await unlink(backupPath)  // 删除备份
     markITerm2SetupComplete()
     return { status: 'restored' }
   } catch (restoreError) {
     // ...
   }
   ```

2. **添加备份过期机制**
   ```typescript
   // 检查备份文件年龄，过期则忽略
   const stats = await stat(backupPath)
   const ageMs = Date.now() - stats.mtime.getTime()
   const MAX_BACKUP_AGE_MS = 7 * 24 * 60 * 60 * 1000  // 7天
   
   if (ageMs > MAX_BACKUP_AGE_MS) {
     logForDebugging('iTerm2 backup expired, ignoring')
     markITerm2SetupComplete()
     return { status: 'no_backup' }
   }
   ```

3. **支持其他终端**
   - 当前仅支持 iTerm2
   - 可考虑扩展支持其他支持粘贴大括号模式的终端（如 Kitty、Alacritty）

4. **添加恢复确认提示**
   ```typescript
   // 在交互式会话中询问用户是否恢复
   if (isInteractive()) {
     const shouldRestore = await askUser(`Detected incomplete iTerm2 setup. Restore from backup?`)
     if (!shouldRestore) {
       markITerm2SetupComplete()
       return { status: 'no_backup' }
     }
   }
   ```

5. **更详细的日志记录**
   ```typescript
   logForDebugging('iTerm2 backup check', {
     inProgress,
     backupPath,
     plistPath: getITerm2PlistPath(),
   })
   ```

6. **单元测试覆盖**
   - 模拟各种配置状态的恢复行为
   - 文件操作失败的错误处理
   - 并发调用场景测试
