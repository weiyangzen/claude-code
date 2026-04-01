# TeleportStash.tsx 深度研究文档

## 场景与职责

TeleportStash 是 Claude Code CLI 中 **Teleport 功能** 的 Git 状态处理对话框。当用户尝试使用 `--teleport` 恢复远程会话时，如果当前工作目录有未提交的更改，此对话框会提示用户 stash 这些更改以继续操作。

该组件的核心职责：
1. **Git 状态检测**：检查当前工作目录的更改状态
2. **更改展示**：向用户展示被修改的文件列表
3. **Stash 操作**：执行 Git stash 保存更改
4. **流程控制**：根据用户选择继续或取消 Teleport 流程

## 功能点目的

### 1. 工作目录保护
Teleport 操作需要切换到特定分支，未提交的更改可能导致：
- 更改丢失
- 分支切换失败
- 合并冲突

通过 stash 操作，用户的更改被安全保存，Teleport 完成后可以恢复。

### 2. 用户确认
强制用户确认 stash 操作，避免意外数据丢失：
- 展示具体被修改的文件
- 提供 "Stash changes and continue" 和 "Exit" 两个明确选项

### 3. 错误处理
处理 stash 过程中的各种错误：
- Git 命令失败
- 文件系统错误
- 权限问题

## 具体技术实现

### 关键数据结构

```typescript
// 组件 Props
type TeleportStashProps = {
  onStashAndContinue: () => void;  // stash 成功后继续
  onCancel: () => void;            // 取消操作
};

// Git 文件状态（来自 git.ts）
export type GitFileStatus = {
  tracked: string[];    // 已跟踪的修改文件
  untracked: string[];  // 未跟踪的新文件
};
```

### 核心流程

#### 1. 组件挂载时加载 Git 状态
```typescript
useEffect(() => {
  const loadChangedFiles = async () => {
    try {
      const fileStatus = await getFileStatus();
      setGitFileStatus(fileStatus);
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : String(err);
      logForDebugging(`Error getting changed files: ${errorMessage}`, {
        level: 'error'
      });
      setError('Failed to get changed files');
    } finally {
      setLoading(false);
    }
  };
  void loadChangedFiles();
}, []);
```

#### 2. Stash 操作处理
```typescript
const handleStash = async () => {
  setStashing(true);
  try {
    logForDebugging('Stashing changes before teleport...');
    const success = await stashToCleanState('Teleport auto-stash');
    
    if (success) {
      logForDebugging('Successfully stashed changes');
      onStashAndContinue();
    } else {
      setError('Failed to stash changes');
    }
  } catch (err) {
    const errorMessage = err instanceof Error ? err.message : String(err);
    logForDebugging(`Error stashing changes: ${errorMessage}`, {
      level: 'error'
    });
    setError('Failed to stash changes');
  } finally {
    setStashing(false);
  }
};
```

#### 3. 选择处理
```typescript
const handleSelectChange = (value: string) => {
  if (value === 'stash') {
    void handleStash();
  } else {
    onCancel();
  }
};
```

#### 4. 文件列表展示
```typescript
const changedFiles = gitFileStatus !== null 
  ? [...gitFileStatus.tracked, ...gitFileStatus.untracked] 
  : [];

const showFileCount = changedFiles.length > 8;

// 条件渲染
{changedFiles.length > 0 ? (
  showFileCount ? (
    <Text>{changedFiles.length} files changed</Text>
  ) : (
    changedFiles.map((file: string, index: number) => (
      <Text key={index}>{file}</Text>
    ))
  )
) : (
  <Text dimColor>No changes detected</Text>
)}
```

## 关键代码路径与文件引用

### 本文件导出
- `TeleportStash` - Stash 对话框组件
- `TeleportStashProps` - 组件 Props 类型

### 依赖文件
| 文件路径 | 用途 |
|---------|------|
| `figures` | 终端图标（省略号） |
| `../ink.js` | Ink UI 组件（Box, Text） |
| `../utils/debug.js` | `logForDebugging` 调试日志 |
| `../utils/git.js` | Git 操作（`getFileStatus`, `stashToCleanState`） |
| `./CustomSelect/index.js` | Select 组件 |
| `./design-system/Dialog.js` | 对话框容器 |
| `./Spinner.js` | Loading 动画 |

### 调用方
| 文件路径 | 调用方式 |
|---------|---------|
| `src/components/TeleportError.tsx` | 作为错误处理流程的一部分 |

### 依赖的 Git 工具函数

#### git.ts - getFileStatus
```typescript
export const getFileStatus = async (): Promise<GitFileStatus> => {
  const { stdout } = await execFileNoThrow(
    gitExe(),
    ['--no-optional-locks', 'status', '--porcelain'],
    { preserveOutputOnError: false }
  );

  const tracked: string[] = [];
  const untracked: string[] = [];

  stdout
    .trim()
    .split('\n')
    .filter(line => line.length > 0)
    .forEach(line => {
      const status = line.substring(0, 2);
      const filename = line.substring(2).trim();

      if (status === '??') {
        untracked.push(filename);
      } else if (filename) {
        tracked.push(filename);
      }
    });

  return { tracked, untracked };
};
```

#### git.ts - stashToCleanState
```typescript
export const stashToCleanState = async (message?: string): Promise<boolean> => {
  try {
    const stashMessage = message || `Claude Code auto-stash - ${new Date().toISOString()}`;

    // 先检查是否有未跟踪的文件
    const { untracked } = await getFileStatus();

    // 如果有未跟踪文件，先添加到索引
    if (untracked.length > 0) {
      const { code: addCode } = await execFileNoThrow(
        gitExe(),
        ['add', ...untracked],
        { preserveOutputOnError: false }
      );
      if (addCode !== 0) return false;
    }

    // 执行 stash
    const { code } = await execFileNoThrow(
      gitExe(),
      ['stash', 'push', '--message', stashMessage],
      { preserveOutputOnError: false }
    );
    return code === 0;
  } catch (_) {
    return false;
  }
};
```

## 依赖与外部交互

### Git 状态流转

```
用户执行 --teleport
    ↓
检测 Git 状态
    ↓
有未提交更改 → 显示 TeleportStash
    ↓
用户选择 "Stash and continue"
    ↓
执行 stashToCleanState
    ↓
成功 → 继续 Teleport 流程
失败 → 显示错误
    ↓
用户选择 "Exit"
    ↓
取消 Teleport 流程
```

### 文件列表展示策略

- **≤8 个文件**：逐个列出文件名
- **>8 个文件**：仅显示 "X files changed"
- **无更改**：显示 "No changes detected"

这种设计平衡了信息完整性和界面简洁性。

### 错误处理层级

1. **Git 状态获取失败**
   - 显示错误消息
   - 提供 Esc 取消选项

2. **Stash 操作失败**
   - 显示 "Failed to stash changes"
   - 保持在对话框状态，允许重试

3. **未知错误**
   - 捕获并记录到调试日志
   - 显示通用错误消息

## 风险、边界与改进建议

### 已知风险

1. **未跟踪文件处理**
   - `stashToCleanState` 会先将未跟踪文件添加到索引
   - 如果添加失败，stash 会跳过这些文件
   - 可能导致数据丢失

2. **Stash 消息硬编码**
   - 使用固定的 "Teleport auto-stash" 消息
   - 用户难以在 stash 列表中识别

3. **无 stash 恢复提示**
   - Teleport 完成后不提示用户恢复 stash
   - 用户可能忘记恢复更改

### 边界情况

1. **大量文件更改**
   - 仅显示数量，不显示具体文件
   - 用户无法确认是否有重要文件

2. **Stash 冲突**
   - 如果同名 stash 已存在，可能产生冲突
   - 当前实现未处理此情况

3. **权限不足**
   - 如果用户没有 Git 仓库写权限
   - stash 操作会失败

### 改进建议

1. **增强文件展示**
   ```typescript
   // 添加展开/折叠功能
   const [isExpanded, setIsExpanded] = useState(false);
   
   {changedFiles.length > 8 && !isExpanded ? (
     <Text>{changedFiles.length} files changed 
       <Text dimColor>(Press Space to expand)</Text>
     </Text>
   ) : (
     changedFiles.map(file => <Text key={file}>{file}</Text>)
   )}
   ```

2. **区分跟踪/未跟踪文件**
   ```typescript
   <Box flexDirection="column" paddingLeft={2}>
     {gitFileStatus.tracked.length > 0 && (
       <>
         <Text dimColor>Modified:</Text>
         {gitFileStatus.tracked.map(f => <Text key={f}>  {f}</Text>)}
       </>
     )}
     {gitFileStatus.untracked.length > 0 && (
       <>
         <Text dimColor>New files:</Text>
         {gitFileStatus.untracked.map(f => <Text key={f}>  {f}</Text>)}
       </>
     )}
   </Box>
   ```

3. **添加 stash 恢复提示**
   - Teleport 完成后显示提示：
   - "Your changes have been stashed. Run `git stash pop` to restore them."

4. **唯一 stash 标识**
   ```typescript
   const stashMessage = `Teleport auto-stash ${new Date().toISOString()} (${sessionId})`;
   ```

5. **预检查 stash 空间**
   ```typescript
   // 在执行 stash 前检查磁盘空间
   const checkDiskSpace = async () => {
     const { stdout } = await execFileNoThrow('du', ['-sb', '.']);
     const size = parseInt(stdout);
     // 检查是否有足够空间...
   };
   ```

6. **Stash 恢复自动化**
   ```typescript
   // 在 Teleport 会话结束时自动恢复
   useEffect(() => {
     return () => {
       // 清理时检查是否有 Teleport stash
       checkAndPopTeleportStash();
     };
   }, []);
   ```

### 测试建议

1. **单元测试**：
   - Git 状态解析正确性
   - 文件列表展示逻辑
   - 选择处理回调

2. **集成测试**：
   - 与 Git 命令的集成
   - 错误处理路径
   - 加载状态展示

3. **E2E 测试**：
   - 完整 stash 流程
   - 大文件列表处理
   - 权限错误场景

---

**文档生成时间**：2026-04-01  
**组件路径**：`src/components/TeleportStash.tsx`  
**关联研究文件**：`src/utils/git.ts`, `src/components/TeleportError.tsx`
