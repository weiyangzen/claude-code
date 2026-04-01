# 研究文档: src/utils/settings/mdm/rawRead.ts

## 场景与职责

本模块是 Claude Code MDM (Mobile Device Management) 设置系统的**子进程读取层**。作为 MDM 子系统的核心 I/O 模块，它承担着以下核心职责：

1. **最小化模块依赖** - 仅导入 `child_process`、`fs` 和 `mdmConstants`，确保在 `main.tsx` 模块评估阶段即可安全执行
2. **非阻塞子进程管理** - 通过异步子进程读取 MDM 配置，避免阻塞 Node.js 事件循环
3. **跨平台 MDM 读取** - 支持 macOS (plutil)、Windows (reg query)、Linux (无操作)
4. **启动时预加载** - 在 `main.tsx` 导入阶段即启动子进程，与后续模块加载并行执行
5. **按需刷新支持** - 提供 `fireRawRead()` 供变更检测器和 SDK 入口点使用

该模块采用**双模式设计**：
- **启动模式**: `startMdmRawRead()` 在模块评估时触发，结果通过 `getMdmRawReadPromise()` 延迟消费
- **轮询/回退模式**: `fireRawRead()` 创建全新的读取操作，用于变更检测和 SDK 入口

## 功能点目的

### 1. 子进程执行封装

**函数**: `execFilePromise(cmd, args)`

```typescript
function execFilePromise(
  cmd: string,
  args: string[],
): Promise<{ stdout: string; code: number | null }>
```

- 封装 Node.js `execFile` 为 Promise API
- 统一设置 `encoding: 'utf-8'` 和超时（5秒）
- 错误处理：任何错误（包括非零退出码）都 resolve 而非 reject，由调用方判断

### 2. 原始 MDM 数据读取

**函数**: `fireRawRead(): Promise<RawReadResult>`

**RawReadResult 结构**:
```typescript
export type RawReadResult = {
  plistStdouts: Array<{ stdout: string; label: string }> | null  // macOS
  hklmStdout: string | null                                      // Windows HKLM
  hkcuStdout: string | null                                      // Windows HKCU
}
```

**平台特定实现**:

| 平台 | 命令 | 说明 |
|------|------|------|
| macOS | `plutil -convert json -o - -- <plist>` | 并行读取所有 plist 路径，首个成功即返回 |
| Windows | `reg query <key> /v Settings` | 并行查询 HKLM 和 HKCU |
| Linux | 无操作 | 返回空结果 |

### 3. macOS 快速路径优化

```typescript
// 在 spawn 前检查文件存在性
if (!existsSync(path)) {
  return { stdout: '', label, ok: false }
}
```

- **优化动机**: `plutil` 即使对不存在的文件也会耗时 ~5ms 返回 ENOENT
- **优化效果**: 非 MDM 机器（大多数用户）完全跳过子进程创建
- **实现细节**: 使用同步 `existsSync` 以保持"首个 await 前完成 spawn"的不变性

### 4. 启动时预加载机制

```typescript
let rawReadPromise: Promise<RawReadResult> | null = null

export function startMdmRawRead(): void {
  if (rawReadPromise) return
  rawReadPromise = fireRawRead()
}

export function getMdmRawReadPromise(): Promise<RawReadResult> | null {
  return rawReadPromise
}
```

- **调用时机**: `main.tsx` 第 16 行在模块评估阶段调用
- **设计目的**: 子进程与后续 ~135ms 的模块导入并行执行
- **结果消费**: `settings.ts` 的 `startMdmSettingsLoad()` 通过 `getMdmRawReadPromise()` 获取

## 具体技术实现

### 关键流程: macOS 读取

```typescript
if (process.platform === 'darwin') {
  const plistPaths = getMacOSPlistPaths()

  const allResults = await Promise.all(
    plistPaths.map(async ({ path, label }) => {
      // 快速路径：文件不存在则跳过
      if (!existsSync(path)) {
        return { stdout: '', label, ok: false }
      }
      const { stdout, code } = await execFilePromise(PLUTIL_PATH, [
        ...PLUTIL_ARGS_PREFIX,
        path,
      ])
      return { stdout, label, ok: code === 0 && !!stdout }
    }),
  )

  // 首个成功源获胜（数组已按优先级排序）
  const winner = allResults.find(r => r.ok)
  return {
    plistStdouts: winner
      ? [{ stdout: winner.stdout, label: winner.label }]
      : [],
    hklmStdout: null,
    hkcuStdout: null,
  }
}
```

### 关键流程: Windows 读取

```typescript
if (process.platform === 'win32') {
  const [hklm, hkcu] = await Promise.all([
    execFilePromise('reg', [
      'query', WINDOWS_REGISTRY_KEY_PATH_HKLM,
      '/v', WINDOWS_REGISTRY_VALUE_NAME,
    ]),
    execFilePromise('reg', [
      'query', WINDOWS_REGISTRY_KEY_PATH_HKCU,
      '/v', WINDOWS_REGISTRY_VALUE_NAME,
    ]),
  ])
  return {
    plistStdouts: null,
    hklmStdout: hklm.code === 0 ? hklm.stdout : null,
    hkcuStdout: hkcu.code === 0 ? hkcu.stdout : null,
  }
}
```

### 超时控制

```typescript
execFile(
  cmd,
  args,
  { encoding: 'utf-8', timeout: MDM_SUBPROCESS_TIMEOUT_MS },  // 5000ms
  (err, stdout) => { /* ... */ }
)
```

## 关键代码路径与文件引用

### 调用关系

```
main.tsx:13-16
  └── import { startMdmRawRead } from './utils/settings/mdm/rawRead.js'
      └── startMdmRawRead()  // 模块评估时调用

settings.ts:75
  └── const rawPromise = getMdmRawReadPromise() ?? fireRawRead()
      └── 消费预加载结果，或触发新读取

changeDetector.ts:395
  └── import { refreshMdmSettings } from './mdm/settings.js'
      └── settings.ts:170 调用 fireRawRead()
```

### 被引用关系

| 引用方 | 导入内容 | 用途 |
|--------|----------|------|
| `src/main.tsx` | `startMdmRawRead` | 启动时预加载 |
| `src/utils/settings/mdm/settings.ts` | `fireRawRead`, `getMdmRawReadPromise`, `RawReadResult` | 解析和消费原始数据 |

## 依赖与外部交互

### 内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `child_process` | `execFile` | 执行子进程 |
| `fs` | `existsSync` | 快速路径文件存在性检查 |
| `./constants.js` | 全部常量 + `getMacOSPlistPaths` | 路径和配置 |

### 外部系统交互

| 平台 | 命令 | 参数 | 输出格式 |
|------|------|------|----------|
| macOS | `/usr/bin/plutil` | `-convert json -o - -- <path>` | JSON stdout |
| Windows | `reg` | `query <key> /v Settings` | 文本表格 |

**macOS 示例输出**:
```json
{
  "permissions": {
    "allow": ["Bash", "Edit"],
    "defaultMode": "auto"
  }
}
```

**Windows 示例输出**:
```
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\ClaudeCode
    Settings    REG_SZ    {"permissions":{"allow":["Bash"]}}
```

## 风险、边界与改进建议

### 已知风险

1. **子进程超时**
   - 风险：5秒超时可能在高负载系统上不足
   - 影响：超时后返回空结果，可能导致 MDM 设置被忽略
   - 缓解：超时后静默失败，不影响主流程；下次轮询会重试

2. **并发读取竞争**
   - 风险：`startMdmRawRead()` 和 `fireRawRead()` 可能同时执行
   - 缓解：`startMdmRawRead()` 检查 `rawReadPromise` 存在性，避免重复启动

3. **Windows 注册表权限**
   - 风险：非管理员用户读取 HKLM 可能失败
   - 缓解：这是预期行为，失败时返回 null，由 `settings.ts` 回退到 HKCU

4. **macOS plutil 不可用**
   - 风险：极少数定制系统可能缺少 `/usr/bin/plutil`
   - 缓解：命令失败时返回空结果，不影响主流程

### 边界条件

| 场景 | 行为 |
|------|------|
| 非 macOS/Windows 平台 | 返回 `{ plistStdouts: null, hklmStdout: null, hkcuStdout: null }` |
| 所有 plist 文件不存在 | 返回空数组 `plistStdouts: []` |
| 子进程超时 | `execFile` 自动终止，返回空 stdout |
| 子进程非零退出码 | `ok: false`，结果不参与后续解析 |
| `startMdmRawRead()` 被调用多次 | 仅首次生效，后续调用直接返回 |

### 改进建议

1. **渐进式超时**
   - 当前：固定 5 秒超时
   - 建议：首次读取使用较短超时（如 3 秒），失败后重试使用较长超时

2. **缓存原始输出**
   - 当前：每次 `fireRawRead()` 都执行子进程
   - 建议：考虑在短时间窗口内缓存结果（需权衡实时性）

3. **更详细的错误分类**
   - 当前：仅返回 `ok: boolean`
   - 建议：区分"文件不存在"、"权限拒绝"、"超时"等错误类型

4. **Windows 注册表解析优化**
   - 当前：返回原始 stdout 文本
   - 建议：考虑在此层解析注册表输出，返回结构化数据

5. **单元测试覆盖**
   - 当前：无直接单元测试
   - 建议：添加模拟子进程的单元测试，覆盖各种边界条件

### 性能考量

1. **启动性能**
   - 预加载机制确保 MDM 读取与模块导入并行
   - 实测：子进程通常在 ~135ms 模块导入期间完成

2. **快速路径效率**
   - `existsSync` 检查避免不必要的子进程创建
   - 非 MDM 机器：零子进程开销

3. **内存占用**
   - 原始 stdout 字符串可能较大（企业配置）
   - 建议：考虑在 `settings.ts` 解析后立即释放原始数据
