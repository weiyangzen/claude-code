# teammateModeSnapshot.ts 深度研究文档

## 场景与职责

teammateModeSnapshot.ts 是 **Teammate 模式的快照管理模块**，负责在会话启动时捕获 teammate 模式配置，并在整个会话期间保持该值不变。这遵循与 `hooksConfigSnapshot.ts` 相同的模式，确保运行时配置更改不会影响当前会话。

**核心定位：**
- 会话级配置快照的单一可信源
- 隔离运行时配置变化与会话执行
- 支持 CLI 覆盖和配置持久化的协调

**设计原则：**
- 启动时捕获一次，之后不再改变
- CLI 覆盖优先于配置文件
- 用户 UI 更改可以清除覆盖并应用新值

---

## 功能点目的

### 1. 模式快照捕获
- `captureTeammateModeSnapshot()`: 在会话启动时捕获 teammate 模式
- 从 CLI 覆盖或全局配置读取
- 记录捕获来源用于调试

### 2. CLI 覆盖管理
- `setCliTeammateModeOverride()`: 设置 CLI 覆盖值
- `getCliTeammateModeOverride()`: 获取当前 CLI 覆盖
- `clearCliTeammateModeOverride()`: 清除覆盖并应用新值

### 3. 模式值获取
- `getTeammateModeFromSnapshot()`: 获取会话的 teammate 模式
- 如果快照未捕获，自动捕获并记录错误

---

## 具体技术实现

### 关键数据结构

```typescript
// Teammate 模式类型
export type TeammateMode = 'auto' | 'tmux' | 'in-process'

// 模块级状态
let initialTeammateMode: TeammateMode | null = null  // 捕获的快照值
let cliTeammateModeOverride: TeammateMode | null = null  // CLI 覆盖值
```

### 核心流程

#### 1. 快照捕获（captureTeammateModeSnapshot）

```
1. 检查是否有 CLI 覆盖
   - 有 → 使用 CLI 覆盖值
   - 无 → 从全局配置读取
2. 记录捕获来源和值（用于调试）
3. 存储到 initialTeammateMode
```

**代码实现（行 56-69）：**

```typescript
export function captureTeammateModeSnapshot(): void {
  if (cliTeammateModeOverride) {
    initialTeammateMode = cliTeammateModeOverride
    logForDebugging(
      `[TeammateModeSnapshot] Captured from CLI override: ${initialTeammateMode}`,
    )
  } else {
    const config = getGlobalConfig()
    initialTeammateMode = config.teammateMode ?? 'auto'
    logForDebugging(
      `[TeammateModeSnapshot] Captured from config: ${initialTeammateMode}`,
    )
  }
}
```

**优先级逻辑：**

```
CLI Override (最高优先级)
    ↓
Global Config (teammateMode 字段)
    ↓
Default Value ('auto')
```

#### 2. CLI 覆盖设置（setCliTeammateModeOverride）

```typescript
export function setCliTeammateModeOverride(mode: TeammateMode): void {
  cliTeammateModeOverride = mode
}
```

**调用时机：**
- 在 CLI 参数解析后调用
- 必须在 `captureTeammateModeSnapshot()` 之前调用

#### 3. CLI 覆盖清除（clearCliTeammateModeOverride）

```typescript
export function clearCliTeammateModeOverride(newMode: TeammateMode): void {
  cliTeammateModeOverride = null
  initialTeammateMode = newMode
  logForDebugging(
    `[TeammateModeSnapshot] CLI override cleared, new mode: ${newMode}`,
  )
}
```

**使用场景：**
- 用户在设置 UI 中更改 teammate 模式
- 清除 CLI 覆盖，使新设置立即生效
- 传递 `newMode` 避免竞态条件

#### 4. 模式值获取（getTeammateModeFromSnapshot）

```typescript
export function getTeammateModeFromSnapshot(): TeammateMode {
  if (initialTeammateMode === null) {
    // 初始化 bug，自动捕获并记录错误
    logError(
      new Error(
        'getTeammateModeFromSnapshot called before capture - this indicates an initialization bug',
      ),
    )
    captureTeammateModeSnapshot()
  }
  return initialTeammateMode ?? 'auto'
}
```

**防御性编程：**
- 如果快照未捕获，自动捕获并记录错误
- 返回 'auto' 作为安全默认值
- 帮助发现初始化顺序问题

---

## 关键代码路径与文件引用

### 内部依赖

| 文件路径 | 用途 |
|---------|------|
| `src/utils/config.ts` | `getGlobalConfig()` |
| `src/utils/debug.ts` | `logForDebugging()` |
| `src/utils/log.ts` | `logError()` |

### 关键代码位置

- **CLI 覆盖设置**: 行 25-27
- **CLI 覆盖获取**: 行 33-35
- **CLI 覆盖清除**: 行 43-49
- **快照捕获**: 行 56-69
- **模式获取**: 行 75-87

---

## 依赖与外部交互

### 与 Registry 的交互

```typescript
// registry.ts
import { getTeammateModeFromSnapshot } from './teammateModeSnapshot.js'

function getTeammateMode(): 'auto' | 'tmux' | 'in-process' {
  return getTeammateModeFromSnapshot()
}
```

### 与 CLI 解析的交互

```typescript
// 在 CLI 参数解析后
if (args['--teammate-mode']) {
  setCliTeammateModeOverride(args['--teammate-mode'])
}

// 在 main.tsx 或 setup 中
captureTeammateModeSnapshot()
```

### 与设置 UI 的交互

```typescript
// 用户更改设置时
function onTeammateModeChange(newMode: TeammateMode) {
  clearCliTeammateModeOverride(newMode)
  // 保存到全局配置
  saveGlobalConfig(config => ({ ...config, teammateMode: newMode }))
}
```

### 与 Spawn Utils 的交互

```typescript
// spawnUtils.ts
import { getTeammateModeFromSnapshot } from './backends/teammateModeSnapshot.js'

export function buildInheritedCliFlags() {
  const sessionMode = getTeammateModeFromSnapshot()
  flags.push(`--teammate-mode ${sessionMode}`)
}
```

---

## 时序图

```
CLI 解析阶段                    启动阶段                         运行阶段
   │                              │                                │
   ├── setCliTeammateModeOverride ─┤                                │
   │   (如果提供了 --teammate-mode)│                                │
   │                              │                                │
   │                              ├── captureTeammateModeSnapshot ──┤
   │                              │   (读取 CLI 覆盖或配置)          │
   │                              │                                │
   │                              │                                ├── getTeammateModeFromSnapshot
   │                              │                                │   (返回快照值)
   │                              │                                │
   │                              │                                │
   │         用户更改设置          │                                │
   │◄─────────────────────────────┤                                │
   │                              │                                │
   ├── clearCliTeammateModeOverride(newMode) ────────────────────────┤
   │   (清除覆盖，更新快照)         │                                │
   │                              │                                │
```

---

## 风险、边界与改进建议

### 已知风险

1. **初始化顺序依赖**
   - `setCliTeammateModeOverride` 必须在 `captureTeammateModeSnapshot` 之前调用
   - 错误的顺序会导致 CLI 覆盖被忽略
   - **缓解**: 文档说明和调试日志

2. **竞态条件**
   - `clearCliTeammateModeOverride` 需要传递 `newMode` 参数
   - 避免读取配置和更新快照之间的竞态
   - **缓解**: 参数传递模式

3. **状态泄漏**
   - 模块级变量在测试间可能泄漏
   - **缓解**: 需要添加重置函数用于测试

4. **调试困难**
   - 如果快照值不符合预期，难以追踪来源
   - **缓解**: 详细的调试日志

### 边界情况

| 场景 | 处理方式 |
|-----|---------|
| CLI 覆盖在捕获后设置 | 覆盖被忽略（设计如此） |
| 多次调用捕获 | 使用首次捕获的值（检查 null） |
| 配置值为 null/undefined | 回退到 'auto' |
| 获取时未捕获 | 自动捕获并记录错误 |
| 清除覆盖时未传递 newMode | 需要参数，强制调用方考虑 |

### 改进建议

1. **测试支持**
   - 添加 `resetTeammateModeSnapshot()` 函数
   - 允许测试重置状态

2. **更严格的类型**
   - 考虑使用 branded type 区分捕获前后的状态
   - 编译时防止未捕获就获取

3. **来源追踪**
   - 记录快照来源（CLI、配置、默认）
   - 帮助调试配置问题

4. **验证逻辑**
   - 添加模式值验证
   - 拒绝无效的模式值

5. **与 hooksConfigSnapshot 统一**
   - 两个快照模块模式相似
   - 可考虑提取通用抽象

6. **热重载支持**
   - 当前设计明确禁止运行时更改
   - 可考虑添加显式的重新捕获 API
   - 用于高级调试场景
