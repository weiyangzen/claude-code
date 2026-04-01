# 研究文档: src/utils/settings/mdm/settings.ts

## 场景与职责

本模块是 Claude Code MDM (Mobile Device Management) 设置系统的**核心解析与缓存层**。作为 MDM 子系统的"大脑"，它承担着以下核心职责：

1. **原始数据解析** - 将 `rawRead.ts` 获取的原始 stdout 转换为结构化的 `SettingsJson`
2. **多源优先级管理** - 实现"首个源获胜"策略（First Source Wins）：remote → HKLM/plist → file → HKCU
3. **设置缓存管理** - 提供同步缓存读取和异步刷新能力
4. **跨平台注册表解析** - 解析 Windows `reg query` 输出提取 JSON 值
5. **验证与错误处理** - 使用 Zod schema 验证 MDM 设置，收集验证错误
6. **变更检测支持** - 为 `changeDetector.ts` 提供刷新接口和缓存更新机制

**架构位置**:
```
constants.ts (零依赖常量)
    ↓
rawRead.ts (子进程 I/O)
    ↓
settings.ts (本模块: 解析、缓存、优先级逻辑)
    ↓
settings.ts (上层: 合并所有设置源)
```

## 功能点目的

### 1. 启动时异步加载

**函数**: `startMdmSettingsLoad()` / `ensureMdmSettingsLoaded()`

```typescript
export function startMdmSettingsLoad(): void {
  if (mdmLoadPromise) return
  mdmLoadPromise = (async () => {
    profileCheckpoint('mdm_load_start')
    const startTime = Date.now()

    // 使用预加载的 rawRead 结果，或触发新读取
    const rawPromise = getMdmRawReadPromise() ?? fireRawRead()
    const { mdm, hkcu } = consumeRawReadResult(await rawPromise)
    mdmCache = mdm
    hkcuCache = hkcu
    
    profileCheckpoint('mdm_load_end')
    logForDebugging(`MDM settings load completed in ${Date.now() - startTime}ms`)
  })()
}
```

- **调用时机**: `main.tsx` 的 `preAction` hook（第 914 行）
- **设计目的**: 确保在首次设置读取前完成 MDM 加载
- **性能优化**: 优先使用 `main.tsx` 预启动的 `rawRead` 结果

### 2. 同步缓存读取

**函数**: `getMdmSettings()` / `getHkcuSettings()`

```typescript
export function getMdmSettings(): MdmResult {
  return mdmCache ?? EMPTY_RESULT
}

export function getHkcuSettings(): MdmResult {
  return hkcuCache ?? EMPTY_RESULT
}
```

- **使用场景**: 上层 `settings.ts` 的 `loadSettingsFromDisk()` 同步调用
- **返回值**: `{ settings: SettingsJson; errors: ValidationError[] }`
- **注意**: 返回的是缓存引用，调用方不应修改

### 3. 原始数据解析

**函数**: `consumeRawReadResult(raw: RawReadResult)`

**优先级链**（从高到低）：

| 优先级 | 源 | 条件 | 说明 |
|--------|-----|------|------|
| 1 | macOS plist | `raw.plistStdouts?.length > 0` | 首个成功读取的 plist |
| 2 | Windows HKLM | `raw.hklmStdout` | 管理员注册表 |
| 3 | File-based | `hasManagedSettingsFile()` | managed-settings.json |
| 4 | Windows HKCU | `raw.hkcuStdout` | 用户注册表 |

**关键逻辑**:
```typescript
// 1. macOS plist（已按优先级排序）
if (raw.plistStdouts && raw.plistStdouts.length > 0) {
  const { stdout, label } = raw.plistStdouts[0]!
  const result = parseCommandOutputAsSettings(stdout, label)
  if (Object.keys(result.settings).length > 0) {
    return { mdm: result, hkcu: EMPTY_RESULT }
  }
}

// 2. Windows HKLM
if (raw.hklmStdout) {
  const jsonString = parseRegQueryStdout(raw.hklmStdout)
  // ...
}

// 3. 检查 file-based 设置（阻止回退到 HKCU）
if (hasManagedSettingsFile()) {
  return { mdm: EMPTY_RESULT, hkcu: EMPTY_RESULT }
}

// 4. Windows HKCU
if (raw.hkcuStdout) {
  // ...
}
```

### 4. Windows 注册表解析

**函数**: `parseRegQueryStdout(stdout, valueName)`

**解析逻辑**:
```typescript
const lines = stdout.split(/\r?\n/)
const escaped = valueName.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
const re = new RegExp(`^\\s+${escaped}\\s+REG_(?:EXPAND_)?SZ\\s+(.*)$`, 'i')
for (const line of lines) {
  const match = line.match(re)
  if (match && match[1]) {
    return match[1].trimEnd()
  }
}
```

- 支持 `REG_SZ` 和 `REG_EXPAND_SZ` 类型
- 大小写不敏感匹配
- 返回 JSON 字符串（需进一步解析）

### 5. 设置验证

**函数**: `parseCommandOutputAsSettings(stdout, sourcePath)`

**验证流程**:
1. 使用 `safeParseJSON` 解析 stdout
2. 调用 `filterInvalidPermissionRules` 过滤无效权限规则（避免单个坏规则导致整个设置被拒绝）
3. 使用 `SettingsSchema().safeParse()` 进行 Zod 验证
4. 使用 `formatZodError` 格式化验证错误

### 6. 文件存在性检查

**函数**: `hasManagedSettingsFile()`

检查两个位置：
1. `managed-settings.json`（主文件）
2. `managed-settings.d/*.json`（drop-in 目录）

**用途**: 当没有 admin MDM 时，阻止回退到 HKCU（因为 file-based 设置优先级高于 HKCU）

## 具体技术实现

### 数据流

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  rawRead.ts     │────▶│  consumeRawRead  │────▶│  mdmCache       │
│  (子进程输出)    │     │  (解析与优先级)   │     │  (同步缓存)      │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │  SettingsSchema  │
                        │  (Zod 验证)       │
                        └──────────────────┘
```

### 关键数据结构

```typescript
type MdmResult = { 
  settings: SettingsJson 
  errors: ValidationError[] 
}

const EMPTY_RESULT: MdmResult = Object.freeze({ 
  settings: {}, 
  errors: [] 
})

// 缓存状态
let mdmCache: MdmResult | null = null   // Admin MDM (plist/HKLM)
let hkcuCache: MdmResult | null = null  // User MDM (HKCU)
let mdmLoadPromise: Promise<void> | null = null
```

### 刷新机制

```typescript
export async function refreshMdmSettings(): Promise<{
  mdm: MdmResult
  hkcu: MdmResult
}> {
  const raw = await fireRawRead()
  return consumeRawReadResult(raw)
}
```

- **调用方**: `changeDetector.ts` 的 30 分钟轮询
- **注意**: 不自动更新缓存，由调用方决定是否应用

## 关键代码路径与文件引用

### 调用关系

```
main.tsx:914
  └── await Promise.all([ensureMdmSettingsLoaded(), ...])
      └── settings.ts:104 ensureMdmSettingsLoaded()
          └── 等待 mdmLoadPromise

settings.ts (上层):323-344
  └── getSettingsForSource('policySettings')
      ├── getRemoteManagedSettingsSyncFromCache()  // remote (最高优先级)
      ├── getMdmSettings()                          // 本模块: plist/HKLM
      ├── loadManagedFileSettings()                 // file-based
      └── getHkcuSettings()                         // 本模块: HKCU

changeDetector.ts:395
  └── refreshMdmSettings()
      └── 触发 MDM 轮询更新
```

### 依赖模块

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `./constants.js` | Windows 注册表常量 | 解析注册表输出 |
| `./rawRead.js` | `fireRawRead`, `getMdmRawReadPromise` | 获取原始数据 |
| `../types.js` | `SettingsJson`, `SettingsSchema` | 设置类型和验证 |
| `../validation.js` | `filterInvalidPermissionRules`, `formatZodError` | 验证辅助 |
| `../managedPath.js` | `getManagedFilePath`, `getManagedSettingsDropInDir` | 文件路径 |
| `../../debug.js` | `logForDebugging` | 调试日志 |
| `../../startupProfiler.js` | `profileCheckpoint` | 性能分析 |

## 依赖与外部交互

### 与上层 settings.ts 的协作

上层 `settings.ts` 的 `policySettings` 加载逻辑：

```typescript
// settings.ts:323-344 (上层)
if (source === 'policySettings') {
  // 1. Remote (最高优先级)
  const remoteSettings = getRemoteManagedSettingsSyncFromCache()
  if (remoteSettings && Object.keys(remoteSettings).length > 0) {
    return remoteSettings
  }

  // 2. Admin-only MDM (本模块)
  const mdmResult = getMdmSettings()
  if (Object.keys(mdmResult.settings).length > 0) {
    return mdmResult.settings
  }

  // 3. File-based managed settings
  const { settings: fileSettings } = loadManagedFileSettings()
  if (fileSettings) {
    return fileSettings
  }

  // 4. HKCU (本模块)
  const hkcu = getHkcuSettings()
  if (Object.keys(hkcu.settings).length > 0) {
    return hkcu.settings
  }

  return null
}
```

### 与远程管理设置的集成

```typescript
// 完整优先级链（从高到低）
1. Remote managed settings (API 获取)
2. macOS plist (admin)
3. Windows HKLM (admin)
4. File-based (managed-settings.json)
5. Windows HKCU (user-writable)
```

### 外部系统交互

本模块**不直接**与外部系统交互，所有 I/O 通过：
- `rawRead.ts` - 子进程读取系统 MDM 配置
- `managedPath.ts` - 文件系统读取 managed-settings.json

## 风险、边界与改进建议

### 已知风险

1. **缓存一致性**
   - 风险：`mdmCache` 和 `hkcuCache` 在进程生命周期内可能过期
   - 缓解：`changeDetector.ts` 每 30 分钟轮询并调用 `setMdmSettingsCache()` 更新

2. **验证错误累积**
   - 风险：无效权限规则被过滤但错误信息可能堆积
   - 缓解：错误去重由上层 `settings.ts` 处理

3. **Windows 注册表格式变化**
   - 风险：未来 Windows 版本可能更改 `reg query` 输出格式
   - 缓解：正则表达式相对宽松，但需持续监控

4. **空设置与无效设置的区分**
   - 风险：`{}`（空对象）和解析失败都可能导致空设置
   - 缓解：`parseCommandOutputAsSettings` 返回 `errors` 数组，调用方可检查

### 边界条件

| 场景 | 行为 |
|------|------|
| 非 macOS/Windows 平台 | 返回空结果，不影响主流程 |
| plist 解析失败 | 返回 `{ settings: {}, errors: [...] }` |
| 注册表值不存在 | 返回 `null`，由调用方处理 |
| 所有源都为空 | 返回 `EMPTY_RESULT` |
| `refreshMdmSettings()` 失败 | 抛出异常，由调用方处理 |

### 改进建议

1. **缓存 TTL**
   - 当前：缓存无过期时间，依赖外部轮询
   - 建议：添加可选的 TTL 机制，过期后自动刷新

2. **更细粒度的错误报告**
   - 当前：仅返回验证错误列表
   - 建议：区分"源不存在"、"解析失败"、"验证失败"等状态

3. **设置来源追踪**
   - 当前：调用方无法知道设置来自 plist 还是 HKLM
   - 建议：在 `MdmResult` 中添加 `source` 字段

4. **并发刷新控制**
   - 当前：`refreshMdmSettings()` 可能被并发调用
   - 建议：添加刷新锁或去重机制

5. **单元测试覆盖**
   - 当前：无直接单元测试
   - 建议：添加模拟 `RawReadResult` 的单元测试，覆盖各种优先级场景

### 安全考量

1. **设置注入**
   - plist 和注册表内容由系统 MDM 控制，非用户可控
   - HKCU 可被用户修改，但优先级最低

2. **敏感信息泄露**
   - 调试日志仅记录设置键名，不记录值
   - 验证错误包含路径信息，可能泄露配置结构

3. **权限规则过滤**
   - `filterInvalidPermissionRules` 防止恶意规则注入
   - 无效规则被移除而非导致整个设置被拒绝

### 性能考量

1. **启动性能**
   - 依赖 `rawRead.ts` 的预加载，通常无额外延迟
   - `ensureMdmSettingsLoaded()` 在 `preAction` 中等待，此时模块已加载完成

2. **内存占用**
   - 缓存的设置对象常驻内存
   - 建议：考虑在长时间运行后释放未使用的设置

3. **轮询开销**
   - 每 30 分钟执行一次完整读取
   - 开销可忽略（子进程 + 解析）
