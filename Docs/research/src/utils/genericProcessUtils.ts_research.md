# genericProcessUtils.ts 深度研究文档

## 场景与职责

`genericProcessUtils.ts` 提供跨平台的进程管理工具函数，用于：

1. **进程状态检查**：检测指定 PID 的进程是否正在运行
2. **进程树遍历**：获取进程的祖先或后代进程链
3. **命令行获取**：获取进程及其祖先的命令行信息

该模块是进程管理、锁恢复、进程监控等功能的基础设施，支持 Windows (PowerShell) 和 Unix (ps/pgrep) 平台。

## 功能点目的

### 1. 进程状态检测
- `isProcessRunning`: 通过信号 0 探测检查进程是否存在

### 2. 进程树遍历
- `getAncestorPidsAsync`: 异步获取进程的祖先 PID 链
- `getChildPids`: 获取进程的子进程 PID 列表

### 3. 命令行获取
- `getProcessCommand`: 获取指定进程的命令行（已弃用）
- `getAncestorCommandsAsync`: 异步获取进程及其祖先的命令行链

## 具体技术实现

### 关键流程

#### 进程存在检测
```typescript
export function isProcessRunning(pid: number): boolean {
  if (pid <= 1) return false  // 0 = 当前进程组, 1 = init
  try {
    process.kill(pid, 0)  // 信号 0 不发送信号，仅检查权限
    return true
  } catch {
    return false
  }
}
```

**注意**：`process.kill(pid, 0)` 在进程存在但属于其他用户时会抛出 `EPERM`，此时返回 `false`（保守策略，避免窃取活动锁）。

#### 祖先 PID 获取（跨平台）

**Windows 实现**：
```powershell
$pid = ${inputPid}
$ancestors = @()
for ($i = 0; $i -lt ${maxDepth}; $i++) {
  $proc = Get-CimInstance Win32_Process -Filter "ProcessId=$pid" -ErrorAction SilentlyContinue
  if (-not $proc -or -not $proc.ParentProcessId -or $proc.ParentProcessId -eq 0) { break }
  $pid = $proc.ParentProcessId
  $ancestors += $pid
}
$ancestors -join ','
```

**Unix 实现**：
```bash
pid=${inputPid}
for i in $(seq 1 ${maxDepth}); do
  ppid=$(ps -o ppid= -p $pid 2>/dev/null | tr -d ' ')
  if [ -z "$ppid" ] || [ "$ppid" = "0" ] || [ "$ppid" = "1" ]; then break; fi
  echo $ppid
  pid=$ppid
done
```

#### 祖先命令行获取（跨平台）

**Windows 实现**：
- 使用 `Get-CimInstance Win32_Process` 获取进程信息
- 使用空字符 (`\0`) 分隔命令行（处理包含换行符的命令）

**Unix 实现**：
- 使用 `ps -o command= -p $pid` 获取命令行
- 使用 `printf '%s\0'` 输出空字符分隔的命令

## 关键代码路径与文件引用

### 核心导出
- `isProcessRunning(pid: number): boolean` - 检查进程是否运行
- `getAncestorPidsAsync(pid: string | number, maxDepth?: number): Promise<number[]>` - 获取祖先 PID
- `getProcessCommand(pid: string | number): string | null` - 获取进程命令（已弃用）
- `getAncestorCommandsAsync(pid: string | number, maxDepth?: number): Promise<string[]>` - 获取祖先命令链
- `getChildPids(pid: string | number): number[]` - 获取子进程 PID

### 依赖关系

**被以下模块导入**：
- `src/services/autoDream/consolidationLock.ts` - 锁恢复
- `src/utils/nativeInstaller/pidLock.ts` - PID 锁管理
- `src/utils/envDynamic.ts` - 动态环境检测
- `src/utils/concurrentSessions.ts` - 并发会话管理
- `src/utils/ide.ts` - IDE 集成
- `src/utils/cronTasksLock.ts` - Cron 任务锁

**依赖的模块**：
- `src/utils/execFileNoThrow.ts` - 安全的命令执行

### 文件位置
- 源码：`src/utils/genericProcessUtils.ts` (184 行)

## 依赖与外部交互

### Node.js 内置模块
- 无直接内置模块依赖

### 项目内部依赖
- `src/utils/execFileNoThrow.ts` - `execFileNoThrowWithCwd`, `execSyncWithDefaults_DEPRECATED`

### 外部依赖
- 无直接外部依赖

### 平台依赖
- **Windows**: `powershell.exe`, `Get-CimInstance Win32_Process`
- **Unix**: `ps`, `pgrep`, `sh`

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**：进程树在遍历过程中可能发生变化
2. **权限限制**：无法获取其他用户的进程信息
3. **平台差异**：Windows 和 Unix 的输出格式差异需要仔细处理
4. **已弃用函数**：`getProcessCommand` 使用同步执行，可能阻塞事件循环

### 边界情况

1. **PID ≤ 1**：直接返回 false（0 是当前进程组，1 是 init）
2. **循环父进程**：`maxDepth` 限制防止无限循环
3. **命令行中的空字符**：使用空字符作为分隔符处理包含换行符的命令
4. **超时处理**：所有异步调用都有 3000ms 超时

### 改进建议

1. **缓存机制**：对进程树信息添加短期缓存，减少重复查询
2. **批量查询**：支持批量 PID 查询，减少子进程创建开销
3. **错误分类**：细化错误处理，区分权限错误和其他错误
4. **测试覆盖**：当前没有专门的测试文件，建议添加单元测试
5. **性能优化**：考虑使用原生模块替代 shell 命令提高性能
6. **类型安全**：改进 PID 参数类型，使用 branded number 类型

### 平台特定注意事项

1. **Windows**：
   - `Get-CimInstance` 可能比 `Get-WmiObject` 慢
   - 需要考虑 PowerShell 执行策略限制

2. **Unix**：
   - BSD 和 GNU ps 的选项差异
   - 某些系统可能缺少 `pgrep`
