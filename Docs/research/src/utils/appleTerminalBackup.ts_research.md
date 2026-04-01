# appleTerminalBackup.ts 研究文档

## 场景与职责

`appleTerminalBackup.ts` 提供 macOS Terminal.app 配置的安全备份和恢复机制，用于 `/terminal-setup` 命令。当用户运行终端设置时，该模块确保在配置失败时可以恢复到原始状态。

## 功能点目的

### 1. 配置备份 (`backupTerminalPreferences`)
- 使用 `defaults export` 导出 Terminal.app 的 plist 配置
- 创建备份文件（`com.apple.Terminal.plist.bak`）
- 在全局配置中标记备份状态

### 2. 配置恢复 (`checkAndRestoreTerminalBackup`)
- 检查是否有中断的设置过程
- 使用 `defaults import` 恢复备份的配置
- 刷新系统偏好缓存（`killall cfprefsd`）
- 清理备份状态标记

### 3. 状态管理
- `markTerminalSetupInProgress` - 标记设置进行中
- `markTerminalSetupComplete` - 标记设置完成
- `getTerminalRecoveryInfo` - 获取恢复信息

## 具体技术实现

### 关键数据结构

```typescript
type RestoreResult =
  | { status: 'restored' | 'no_backup' }
  | { status: 'failed'; backupPath: string }
```

### 备份流程

```typescript
export async function backupTerminalPreferences(): Promise<string | null> {
  const terminalPlistPath = getTerminalPlistPath()
  const backupPath = `${terminalPlistPath}.bak`

  // 首先导出到原始路径（确保文件存在）
  const { code } = await execFileNoThrow('defaults', [
    'export',
    'com.apple.Terminal',
    terminalPlistPath,
  ])

  if (code !== 0) return null

  // 验证文件存在
  try {
    await stat(terminalPlistPath)
  } catch {
    return null
  }

  // 创建备份
  await execFileNoThrow('defaults', [
    'export',
    'com.apple.Terminal',
    backupPath,
  ])

  markTerminalSetupInProgress(backupPath)
  return backupPath
}
```

### 恢复流程

```typescript
export async function checkAndRestoreTerminalBackup(): Promise<RestoreResult> {
  const { inProgress, backupPath } = getTerminalRecoveryInfo()
  
  // 无进行中状态
  if (!inProgress) return { status: 'no_backup' }
  
  // 无备份路径
  if (!backupPath) {
    markTerminalSetupComplete()
    return { status: 'no_backup' }
  }

  // 验证备份文件存在
  try {
    await stat(backupPath)
  } catch {
    markTerminalSetupComplete()
    return { status: 'no_backup' }
  }

  // 执行恢复
  try {
    const { code } = await execFileNoThrow('defaults', [
      'import',
      'com.apple.Terminal',
      backupPath,
    ])

    if (code !== 0) {
      return { status: 'failed', backupPath }
    }

    // 刷新偏好缓存
    await execFileNoThrow('killall', ['cfprefsd'])
    markTerminalSetupComplete()
    return { status: 'restored' }
  } catch (restoreError) {
    markTerminalSetupComplete()
    return { status: 'failed', backupPath }
  }
}
```

### 状态持久化

```typescript
export function markTerminalSetupInProgress(backupPath: string): void {
  saveGlobalConfig(current => ({
    ...current,
    appleTerminalSetupInProgress: true,
    appleTerminalBackupPath: backupPath,
  }))
}

export function markTerminalSetupComplete(): void {
  saveGlobalConfig(current => ({
    ...current,
    appleTerminalSetupInProgress: false,
  }))
}
```

## 关键代码路径与文件引用

### 本文件导出
- `backupTerminalPreferences(): Promise<string | null>` - 备份配置
- `checkAndRestoreTerminalBackup(): Promise<RestoreResult>` - 检查并恢复
- `markTerminalSetupInProgress(backupPath: string): void` - 标记进行中
- `markTerminalSetupComplete(): void` - 标记完成
- `getTerminalPlistPath(): string` - 获取 plist 路径

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/utils/config.ts` | 全局配置读写（`getGlobalConfig`, `saveGlobalConfig`） |
| `src/utils/execFileNoThrow.ts` | 安全执行外部命令 |
| `src/utils/log.ts` | 错误日志（`logError`） |

### 调用方
- `src/setup.ts` - 启动时检查并恢复备份
- `src/commands/terminalSetup/terminalSetup.tsx` - 终端设置命令

## 依赖与外部交互

### 外部系统命令
| 命令 | 用途 |
|------|------|
| `defaults export com.apple.Terminal` | 导出 Terminal.app 配置 |
| `defaults import com.apple.Terminal` | 导入/恢复配置 |
| `killall cfprefsd` | 刷新偏好系统缓存 |

### 文件路径
```
~/Library/Preferences/com.apple.Terminal.plist
~/Library/Preferences/com.apple.Terminal.plist.bak
```

## 风险、边界与改进建议

### 已知限制
1. **macOS 专属**：依赖 macOS 的 `defaults` 命令
2. **单备份**：只保留一个备份，新设置会覆盖旧备份
3. **无自动清理**：备份文件不会自动删除

### 边界条件
1. **权限问题**：`defaults` 命令可能因权限失败
2. **文件系统错误**：备份文件可能损坏或被删除
3. **并发设置**：不支持并发设置操作

### 安全风险
1. **配置泄露**：备份文件包含 Terminal.app 的所有配置
2. **路径遍历**：plist 路径硬编码，无用户输入

### 改进建议
1. **多备份保留**：保留最近 N 个备份，支持回滚到任意版本
2. **备份加密**：对敏感配置进行加密存储
3. **自动清理**：设置完成后自动删除备份文件
4. **跨平台支持**：为 Linux/Windows 终端提供类似功能

### 测试建议
1. 测试备份和恢复的完整流程
2. 测试备份文件不存在时的恢复行为
3. 测试权限不足时的错误处理
4. 测试并发设置操作的边界情况
