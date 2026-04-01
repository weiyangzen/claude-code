# platform.ts 研究文档

## 场景与职责

本模块提供平台检测和系统信息获取功能。核心职责包括：

1. **平台检测**：检测当前运行平台（macOS、Windows、WSL、Linux）
2. **WSL 版本检测**：检测 WSL 版本（1 或 2）
3. **Linux 发行版信息**：获取 Linux 发行版 ID、版本和内核信息
4. **版本控制系统检测**：检测项目使用的 VCS（Git、Mercurial、SVN 等）

该模块是跨平台兼容性的基础设施，用于条件编译、功能开关和遥测数据收集。

## 功能点目的

### 1. `getPlatform()` - 平台检测
- **目的**：检测当前运行的操作系统平台
- **返回值**：`'macos'` | `'windows'` | `'wsl'` | `'linux'` | `'unknown'`
- **检测逻辑**：
  - `process.platform === 'darwin'` → macOS
  - `process.platform === 'win32'` → Windows
  - `process.platform === 'linux'` → 检查 `/proc/version` 是否含 "microsoft" 或 "wsl" → WSL 或 Linux
- **缓存**：使用 `lodash/memoize` 缓存结果

### 2. `getWslVersion()` - WSL 版本检测
- **目的**：检测 WSL 版本（1 或 2）
- **检测方式**：
  1. 检查 `/proc/version` 中的 `WSL\d+` 模式
  2. 无显式版本但含 "microsoft" → 假设 WSL1
- **缓存**：使用 `lodash/memoize` 缓存结果

### 3. `getLinuxDistroInfo()` - Linux 发行版信息
- **目的**：获取 Linux 发行版详细信息
- **信息来源**：`/etc/os-release`
- **返回值**：`{ linuxDistroId?, linuxDistroVersion?, linuxKernel? }`
- **缓存**：使用 `lodash/memoize` 缓存结果

### 4. `detectVcs()` - VCS 检测
- **目的**：检测项目使用的版本控制系统
- **检测方式**：
  - 环境变量：`P4PORT` → Perforce
  - 目录标记：检查 `.git`、`.hg`、`.svn`、`.p4config`、`$tf`、`.jj`、`.sl` 等
- **返回值**：检测到的 VCS 名称数组（可能多个）

## 具体技术实现

### 关键流程

#### 平台检测流程

```
getPlatform()
    ↓
process.platform === 'darwin'?
    是 → 返回 'macos'
    ↓
process.platform === 'win32'?
    是 → 返回 'windows'
    ↓
process.platform === 'linux'?
    是 → 读取 /proc/version
        ↓
        包含 'microsoft' 或 'wsl'?
            是 → 返回 'wsl'
            否 → 返回 'linux'
    ↓
其他 → 返回 'unknown'
```

#### WSL 版本检测流程

```
getWslVersion()
    ↓
process.platform !== 'linux'?
    是 → 返回 undefined
    ↓
读取 /proc/version
    ↓
匹配 /WSL(\d+)/i?
    是 → 返回匹配的数字
    ↓
包含 'microsoft'?
    是 → 返回 '1'（假设 WSL1）
    ↓
返回 undefined
```

#### VCS 检测流程

```
detectVcs(dir?)
    ↓
检查 process.env.P4PORT
    存在 → 添加 'perforce'
    ↓
读取目标目录（默认 cwd）
    ↓
遍历 VCS_MARKERS
    标记存在? → 添加对应 VCS
    ↓
返回检测到的 VCS 数组
```

### 数据结构

```typescript
// 平台类型
export type Platform = 'macos' | 'windows' | 'wsl' | 'linux' | 'unknown'

// 支持的平台列表
export const SUPPORTED_PLATFORMS: Platform[] = ['macos', 'wsl']

// Linux 发行版信息
export type LinuxDistroInfo = {
  linuxDistroId?: string      // 如 'ubuntu', 'debian'
  linuxDistroVersion?: string // 如 '20.04', '11'
  linuxKernel?: string        // 内核版本
}

// VCS 标记映射
const VCS_MARKERS: Array<[string, string]> = [
  ['.git', 'git'],
  ['.hg', 'mercurial'],
  ['.svn', 'svn'],
  ['.p4config', 'perforce'],
  ['$tf', 'tfs'],
  ['.tfvc', 'tfs'],
  ['.jj', 'jujutsu'],
  ['.sl', 'sapling'],
]
```

### `/etc/os-release` 解析

```typescript
for (const line of content.split('\n')) {
  const match = line.match(/^(ID|VERSION_ID)=(.*)$/)
  if (match && match[1] && match[2]) {
    const value = match[2].replace(/^"|"$/g, '')  // 去除引号
    if (match[1] === 'ID') {
      result.linuxDistroId = value
    } else {
      result.linuxDistroVersion = value
    }
  }
}
```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `fs/promises` | 异步文件读取 |
| `lodash-es/memoize.js` | 结果缓存 |
| `os` | `release()` 获取内核版本 |
| `./fsOperations.js` | 文件系统操作（同步） |
| `./log.js` | 错误日志 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/cli/handlers/mcp.tsx` | CLI 平台适配 |
| `src/services/tips/tipRegistry.ts` | 平台相关提示 |
| `src/services/mcp/xaaIdpLogin.ts` | MCP 登录平台适配 |
| `src/services/mcp/auth.ts` | 认证平台适配 |
| `src/services/voice.ts` | 语音功能平台适配 |
| `src/services/analytics/metadata.ts` | 遥测平台信息 |
| `src/services/mcp/config.ts` | MCP 配置平台适配 |
| `src/services/mcp/oauthPort.ts` | OAuth 端口平台适配 |
| `src/services/analytics/firstPartyEventLogger.ts` | 事件日志平台信息 |
| `src/utils/telemetry/instrumentation.ts` | 遥测平台检测 |
| `src/main.tsx` | 主程序平台初始化 |
| `src/hooks/usePasteHandler.ts` | 粘贴处理平台适配 |
| `src/hooks/useDiffInIDE.ts` | IDE 差异平台适配 |
| `src/keybindings/reservedShortcuts.ts` | 快捷键平台适配 |
| `src/keybindings/defaultBindings.ts` | 默认绑定平台适配 |
| 以及 10+ 其他模块 | 各种平台适配场景 |

### 系统文件访问

| 文件 | 用途 | 容错 |
|------|------|------|
| `/proc/version` | WSL 检测、内核版本 | 读取失败视为普通 Linux |
| `/etc/os-release` | Linux 发行版信息 | 读取失败返回部分信息 |
| `cwd()` | VCS 检测 | 读取失败返回空数组 |

## 风险、边界与改进建议

### 已知风险

1. **平台检测误判**
   - 风险：WSL 检测依赖 `/proc/version` 内容
   - 潜在问题：未来 WSL 版本可能改变格式
   - 缓解：多层检测（显式版本 + microsoft 字符串）

2. **缓存失效**
   - 风险：`memoize` 缓存的平台信息在运行时不变
   - 潜在问题：容器环境可能动态改变
   - 现状：通常不是问题，平台在进程生命周期内不变

3. **VCS 检测不完整**
   - 风险：仅检查根目录标记
   - 潜在问题：子目录使用不同 VCS 无法检测
   - 建议：递归向上检查

4. **权限问题**
   - 风险：`/proc/version` 或 `/etc/os-release` 可能无权限读取
   - 处理：try-catch，失败返回默认值

### 边界情况

| 场景 | 行为 |
|------|------|
| `/proc/version` 不存在 | Linux 平台返回 'linux'（非 WSL） |
| `/proc/version` 无 WSL 标记 | 返回 'linux' |
| `/etc/os-release` 不存在 | 返回仅有 `linuxKernel` 的对象 |
| `/etc/os-release` 格式异常 | 部分解析或返回空值 |
| 目录不可读 | VCS 检测返回空数组 |
| 多个 VCS 标记 | 返回所有检测到的 VCS |
| 未知平台 | 返回 'unknown' |

### 改进建议

1. **更精确的 WSL 检测**
   - 当前：基于 `/proc/version` 字符串匹配
   - 建议：
     - 检查 `WSL_DISTRO_NAME` 环境变量
     - 检查 `/proc/sys/kernel/osrelease`
     - 使用多种指标综合判断

2. **容器检测**
   - 建议：添加容器环境检测（Docker、LXC、Kubernetes）
   - 用途：容器特定的功能适配

3. **VCS 检测增强**
   - 建议：
     - 递归向上查找 VCS 根目录
     - 检测嵌套 VCS（子模块）
     - 检测 VCS 状态（干净/有变更）

4. **架构检测**
   - 建议：添加 CPU 架构检测（x64、ARM64、ARM）
   - 用途：原生模块加载、功能开关

5. **虚拟化检测**
   - 建议：检测虚拟机环境（VMware、VirtualBox、Hyper-V）
   - 用途：性能优化、功能适配

6. **缓存刷新**
   - 建议：提供手动刷新缓存的接口
   - 用途：长时间运行的进程、环境变化

7. **遥测增强**
   - 建议：
     - 记录平台分布统计
     - 记录 WSL 版本使用
     - 记录 Linux 发行版分布

8. **测试覆盖**
   - 建议：
     - 模拟不同平台环境测试
     - 边界条件测试（文件不存在、权限不足）
     - VCS 检测场景测试
