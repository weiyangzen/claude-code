# registry.ts 深度研究文档

## 场景与职责

registry.ts 是 Agent Swarm 后端系统的 **中央注册表和调度中心**，负责后端检测、选择、缓存和生命周期管理。它是上层代码（如 TeammateTool）与具体后端实现之间的抽象层。

**核心定位：**
- 后端检测和选择的单一入口
- 后端实例的缓存和管理
- 执行模式决策（in-process vs pane-based）
- 与配置系统的集成

**关键职责：**
1. 检测并选择合适的 PaneBackend（tmux/iTerm2）
2. 提供统一的 TeammateExecutor 接口
3. 管理后端实例缓存
4. 处理执行模式回退（fallback）

---

## 功能点目的

### 1. 后端注册管理
- `ensureBackendsRegistered()`: 确保后端类已动态导入
- `registerTmuxBackend()`: 注册 TmuxBackend 类
- `registerITermBackend()`: 注册 ITermBackend 类
- 使用延迟导入避免循环依赖

### 2. 后端检测与选择
- `detectAndGetBackend()`: 检测环境并选择最佳后端
- 优先级：tmux (inside) > iTerm2 (native) > tmux (fallback) > error
- 缓存检测结果避免重复检测

### 3. 执行器获取
- `getTeammateExecutor()`: 获取 TeammateExecutor 实例
- 根据配置和环境选择 InProcessBackend 或 PaneBackendExecutor
- `getInProcessBackend()`: 获取进程内执行器
- `getPaneBackendExecutor()`: 获取窗口执行器

### 4. 执行模式决策
- `isInProcessEnabled()`: 判断是否启用进程内模式
- 考虑：teammateMode 配置、环境检测、fallback 状态
- `getResolvedTeammateMode()`: 获取解析后的执行模式

### 5. 缓存和状态管理
- `getCachedBackend()`: 获取缓存的后端
- `getCachedDetectionResult()`: 获取缓存的检测结果
- `markInProcessFallback()`: 标记进程内回退状态
- `resetBackendDetection()`: 重置所有缓存（测试用）

---

## 具体技术实现

### 关键数据结构

```typescript
// 缓存
let cachedBackend: PaneBackend | null = null
let cachedDetectionResult: BackendDetectionResult | null = null
let cachedInProcessBackend: TeammateExecutor | null = null
let cachedPaneBackendExecutor: TeammateExecutor | null = null

// 状态
let backendsRegistered = false
let inProcessFallbackActive = false  // 是否已回退到 in-process

// 后端类占位符（避免循环依赖）
let TmuxBackendClass: (new () => PaneBackend) | null = null
let ITermBackendClass: (new () => PaneBackend) | null = null
```

### 后端检测优先级流程

```
detectAndGetBackend():
  1. 确保后端已注册
  2. 检查缓存
  3. 检测环境：
     a. 是否在 tmux 内部？
        - 是 → 使用 TmuxBackend (isNative: true)
     b. 是否在 iTerm2？
        - 用户是否偏好 tmux？
          - 是 → 跳过 iTerm2 检测
          - 否 → it2 是否可用？
            - 是 → 使用 ITermBackend (isNative: true)
            - 否 → tmux 是否可用？
              - 是 → 使用 TmuxBackend (isNative: false, needsIt2Setup: true)
              - 否 → 抛出错误（需要安装 it2）
     c. tmux 是否可用？
        - 是 → 使用 TmuxBackend (isNative: false)
        - 否 → 抛出错误（需要安装 tmux）
```

**代码实现（行 136-254）：**

```typescript
export async function detectAndGetBackend(): Promise<BackendDetectionResult> {
  await ensureBackendsRegistered()
  
  if (cachedDetectionResult) return cachedDetectionResult
  
  const insideTmux = await isInsideTmux()
  const inITerm2 = isInITerm2()
  
  // Priority 1: Inside tmux
  if (insideTmux) {
    const backend = createTmuxBackend()
    cachedDetectionResult = { backend, isNative: true, needsIt2Setup: false }
    return cachedDetectionResult
  }
  
  // Priority 2: In iTerm2
  if (inITerm2) {
    const preferTmux = getPreferTmuxOverIterm2()
    if (!preferTmux) {
      const it2Available = await isIt2CliAvailable()
      if (it2Available) {
        const backend = createITermBackend()
        cachedDetectionResult = { backend, isNative: true, needsIt2Setup: false }
        return cachedDetectionResult
      }
    }
    
    // Fallback to tmux
    const tmuxAvailable = await isTmuxAvailable()
    if (tmuxAvailable) {
      const backend = createTmuxBackend()
      cachedDetectionResult = { 
        backend, 
        isNative: false, 
        needsIt2Setup: !preferTmux  // 只有用户未明确偏好 tmux 时才提示
      }
      return cachedDetectionResult
    }
    
    throw new Error('iTerm2 detected but it2 CLI not installed...')
  }
  
  // Priority 3: External tmux
  const tmuxAvailable = await isTmuxAvailable()
  if (tmuxAvailable) {
    const backend = createTmuxBackend()
    cachedDetectionResult = { backend, isNative: false, needsIt2Setup: false }
    return cachedDetectionResult
  }
  
  throw new Error(getTmuxInstallInstructions())
}
```

### 执行模式决策逻辑

```
isInProcessEnabled():
  1. 非交互式会话？
     - 是 → 返回 true（强制 in-process）
  2. 获取 teammateMode（从 snapshot）
     - 'in-process' → 返回 true
     - 'tmux' → 返回 false
     - 'auto' → 继续判断
  3. 'auto' 模式：
     a. 之前是否已回退到 in-process？
        - 是 → 返回 true（环境不会变）
     b. 是否在 tmux 或 iTerm2 中？
        - 是 → 返回 false（使用 pane backend）
        - 否 → 返回 true（使用 in-process）
```

**代码实现（行 351-389）：**

```typescript
export function isInProcessEnabled(): boolean {
  // 强制 in-process 用于非交互式会话
  if (getIsNonInteractiveSession()) {
    return true
  }
  
  const mode = getTeammateMode()  // 从 snapshot 获取
  
  if (mode === 'in-process') return true
  if (mode === 'tmux') return false
  
  // 'auto' 模式
  if (inProcessFallbackActive) {
    return true  // 已回退，保持 in-process
  }
  
  const insideTmux = isInsideTmuxSync()
  const inITerm2 = isInITerm2()
  return !insideTmux && !inITerm2  // 无 pane 环境时使用 in-process
}
```

### 执行器获取流程

```
getTeammateExecutor(preferInProcess):
  1. preferInProcess 为 true 且 in-process 已启用？
     - 是 → 返回 InProcessBackend
  2. 否则 → 返回 PaneBackendExecutor
```

**代码实现（行 425-436）：**

```typescript
export async function getTeammateExecutor(
  preferInProcess: boolean = false,
): Promise<TeammateExecutor> {
  if (preferInProcess && isInProcessEnabled()) {
    return getInProcessBackend()
  }
  return getPaneBackendExecutor()
}
```

### 后端类注册机制

**为什么使用延迟注册？**

避免循环依赖：
- `registry.ts` 需要 `TmuxBackend` 和 `ITermBackend`
- `TmuxBackend.ts` 和 `ITermBackend.ts` 需要 `registry.ts` 中的注册函数

**解决方案：**

```typescript
// registry.ts - 提供注册函数
export function registerTmuxBackend(backendClass: new () => PaneBackend): void {
  TmuxBackendClass = backendClass
}

// TmuxBackend.ts - 模块加载时自注册
registerTmuxBackend(TmuxBackend)

// registry.ts - 使用时动态导入
export async function ensureBackendsRegistered(): Promise<void> {
  if (backendsRegistered) return
  await import('./TmuxBackend.js')
  await import('./ITermBackend.js')
  backendsRegistered = true
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 文件路径 | 用途 |
|---------|------|
| `src/utils/swarm/backends/types.ts` | `PaneBackend`, `TeammateExecutor`, `BackendDetectionResult` 类型 |
| `src/utils/swarm/backends/detection.ts` | `isInsideTmux()`, `isInITerm2()`, `isTmuxAvailable()`, `isIt2CliAvailable()` |
| `src/utils/swarm/backends/teammateModeSnapshot.ts` | `getTeammateModeFromSnapshot()` |
| `src/utils/swarm/backends/it2Setup.ts` | `getPreferTmuxOverIterm2()` |
| `src/utils/swarm/backends/InProcessBackend.ts` | `createInProcessBackend()` |
| `src/utils/swarm/backends/PaneBackendExecutor.ts` | `createPaneBackendExecutor()` |
| `src/utils/swarm/backends/TmuxBackend.ts` | 动态导入，自注册 |
| `src/utils/swarm/backends/ITermBackend.ts` | 动态导入，自注册 |
| `src/bootstrap/state.ts` | `getIsNonInteractiveSession()` |
| `src/utils/platform.ts` | `getPlatform()` |
| `src/utils/debug.ts` | `logForDebugging()` |

### 关键代码位置

- **后端注册**: 行 74-100
- **后端创建**: 行 106-126
- **检测逻辑**: 行 136-254
- **安装指导**: 行 259-285
- **执行模式判断**: 行 351-389
- **执行器获取**: 行 404-451
- **缓存重置**: 行 457-464

---

## 依赖与外部交互

### 与 TeammateTool 的交互

```typescript
// TeammateTool 调用
const executor = await getTeammateExecutor(preferInProcess)
const result = await executor.spawn(config)
```

### 与配置系统的交互

```typescript
// 从 teammateModeSnapshot 获取模式
function getTeammateMode(): 'auto' | 'tmux' | 'in-process' {
  return getTeammateModeFromSnapshot()
}

// 从 it2Setup 获取用户偏好
const preferTmux = getPreferTmuxOverIterm2()
```

### 与状态系统的交互

```typescript
// 非交互式会话检测
if (getIsNonInteractiveSession()) {
  return true  // 强制 in-process
}
```

### 缓存策略

| 缓存变量 | 用途 | 生命周期 |
|---------|------|---------|
| `cachedBackend` | 缓存 PaneBackend 实例 | 进程级 |
| `cachedDetectionResult` | 缓存检测结果 | 进程级 |
| `cachedInProcessBackend` | 缓存 InProcessBackend | 进程级 |
| `cachedPaneBackendExecutor` | 缓存 PaneBackendExecutor | 进程级 |

---

## 风险、边界与改进建议

### 已知风险

1. **检测顺序依赖**
   - 检测顺序固定，可能影响某些边缘场景
   - 例如：在 tmux 中运行的 iTerm2 会优先使用 tmux

2. **缓存不一致**
   - 环境理论上不会变，但缓存可能导致测试困难
   - **缓解**: 提供 `resetBackendDetection()` 用于测试

3. **循环依赖风险**
   - 虽然使用延迟导入，但仍需小心维护
   - 错误的导入顺序可能导致类未注册

4. **Fallback 状态持久化**
   - `inProcessFallbackActive` 是内存状态，重启后重置
   - 用户可能需要再次经历检测流程

5. **错误消息平台差异**
   - 安装指导根据平台定制，但可能不完整
   - 某些 Linux 发行版可能有不同的包管理器

### 边界情况

| 场景 | 处理方式 |
|-----|---------|
| 后端类未注册 | 抛出错误，提示导入顺序问题 |
| 检测缓存存在 | 直接返回缓存结果 |
| iTerm2 + 偏好 tmux | 跳过 iTerm2 检测，避免重复提示 |
| 非交互式会话 | 强制 in-process，无视配置 |
| 无可用后端 | 抛出错误，提供安装指导 |

### 改进建议

1. **配置热重载**
   - 当前 teammateMode 在启动时快照
   - 可考虑支持运行时切换（需谨慎）

2. **检测重试机制**
   - 对临时性检测失败实现重试
   - 例如：tmux 命令超时

3. **更细粒度的回退控制**
   - 当前只有全局 inProcessFallbackActive
   - 可考虑按团队或按队友控制

4. **检测原因追踪**
   - 记录为什么选择特定后端
   - 帮助用户理解和调试

5. **后端健康检查**
   - 定期验证后端仍然可用
   - 提前发现窗口被关闭等情况

6. **A/B 测试支持**
   - 支持随机选择后端进行测试
   - 收集不同后端的性能和用户体验数据

7. **更好的错误恢复**
   - 创建队友失败时自动尝试其他后端
   - 而非直接失败
