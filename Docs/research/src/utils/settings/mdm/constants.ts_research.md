# 研究文档: src/utils/settings/mdm/constants.ts

## 场景与职责

本模块是 Claude Code MDM (Mobile Device Management) 设置系统的**共享常量与路径构建器**。作为 MDM 子系统的最底层模块，它承担着以下核心职责：

1. **零重量级依赖设计** - 仅导入 `os` 和 `path` 模块，确保可以被 `mdmRawRead.ts` 安全导入而不触发重量级模块加载
2. **跨平台常量定义** - 统一封装 macOS、Windows、Linux 三大平台的 MDM 相关路径和配置
3. **plist 路径优先级构建** - 根据运行环境动态构建 macOS MDM 配置文件的查找路径（按优先级排序）
4. **测试环境支持** - 通过 `USER_TYPE=ant` 环境变量启用用户可写偏好设置路径，支持内部测试

该模块是 MDM 子系统的"基石"，被 `rawRead.ts` 和 `settings.ts` 共同导入以避免重复定义。

## 功能点目的

### 1. 平台特定的 MDM 域/注册表路径

**macOS Preference Domain**
```typescript
export const MACOS_PREFERENCE_DOMAIN = 'com.anthropic.claudecode'
```
- 用途：macOS MDM 配置文件的标识域，用于 `defaults read` 或 plist 文件命名
- 文件位置：`/Library/Managed Preferences/${DOMAIN}.plist`

**Windows 注册表路径**
```typescript
export const WINDOWS_REGISTRY_KEY_PATH_HKLM = 'HKLM\\SOFTWARE\\Policies\\ClaudeCode'
export const WINDOWS_REGISTRY_KEY_PATH_HKCU = 'HKCU\\SOFTWARE\\Policies\\ClaudeCode'
```
- 设计考量：
  - 使用 `SOFTWARE\Policies` 而非 `SOFTWARE\ClaudeCode`
  - `Policies` 位于 WOW64 共享密钥列表，32/64 位进程可见相同值
  - 避免 32 位进程被重定向到 `WOW6432Node` 导致读取失败
- 参考文档：[Microsoft Shared Registry Keys](https://learn.microsoft.com/en-us/windows/win32/winprog64/shared-registry-keys)

### 2. plutil 命令配置

```typescript
export const PLUTIL_PATH = '/usr/bin/plutil'
export const PLUTIL_ARGS_PREFIX = ['-convert', 'json', '-o', '-', '--'] as const
```
- 用途：将 macOS plist 文件转换为 JSON 输出到 stdout
- 命令示例：`plutil -convert json -o - -- /path/to/file.plist`

### 3. 子进程超时控制

```typescript
export const MDM_SUBPROCESS_TIMEOUT_MS = 5000
```
- 用途：限制 `plutil` 和 `reg query` 子进程的最大执行时间
- 设计目的：防止 MDM 读取阻塞主事件循环，确保启动性能

### 4. macOS Plist 路径构建器

**函数签名**：`getMacOSPlistPaths(): Array<{ path: string; label: string }>`

**优先级顺序**（从高到低）：

| 优先级 | 路径 | 标签 | 条件 |
|--------|------|------|------|
| 1 | `/Library/Managed Preferences/${username}/${DOMAIN}.plist` | per-user managed preferences | `username` 存在 |
| 2 | `/Library/Managed Preferences/${DOMAIN}.plist` | device-level managed preferences | 始终包含 |
| 3 | `~/Library/Preferences/${DOMAIN}.plist` | user preferences (ant-only) | `USER_TYPE === 'ant'` |

**设计考量**：
- 使用 `userInfo().username` 获取当前用户名（失败时静默忽略）
- 用户可写路径（优先级3）仅在内部测试环境启用，生产环境禁用
- 返回数组按优先级排序，供 `rawRead.ts` 实现"首个成功即获胜"策略

## 具体技术实现

### 关键数据结构

```typescript
// 路径条目结构
interface PlistPathEntry {
  path: string   // 绝对路径
  label: string  // 人类可读标签（用于日志/调试）
}
```

### 路径构建逻辑

```typescript
export function getMacOSPlistPaths(): Array<{ path: string; label: string }> {
  let username = ''
  try {
    username = userInfo().username  // 安全获取用户名
  } catch {
    // 失败时静默处理，不影响后续路径
  }

  const paths: Array<{ path: string; label: string }> = []

  // 1. 用户级托管偏好（最高优先级）
  if (username) {
    paths.push({
      path: `/Library/Managed Preferences/${username}/${MACOS_PREFERENCE_DOMAIN}.plist`,
      label: 'per-user managed preferences',
    })
  }

  // 2. 设备级托管偏好
  paths.push({
    path: `/Library/Managed Preferences/${MACOS_PREFERENCE_DOMAIN}.plist`,
    label: 'device-level managed preferences',
  })

  // 3. 用户可写偏好（仅内部测试）
  if (process.env.USER_TYPE === 'ant') {
    paths.push({
      path: join(homedir(), 'Library', 'Preferences', `${MACOS_PREFERENCE_DOMAIN}.plist`),
      label: 'user preferences (ant-only)',
    })
  }

  return paths
}
```

## 关键代码路径与文件引用

### 被引用关系

| 引用方 | 导入内容 | 用途 |
|--------|----------|------|
| `src/utils/settings/mdm/rawRead.ts` | 全部常量 + `getMacOSPlistPaths` | 执行子进程读取 MDM 配置 |
| `src/utils/settings/mdm/settings.ts` | Windows 注册表常量 | 解析 Windows 注册表输出 |

### 引用链

```
main.tsx:13
  └── import { startMdmRawRead } from './utils/settings/mdm/rawRead.js'
      └── rawRead.ts 导入 constants.ts
```

## 依赖与外部交互

### 内部依赖

| 模块 | 导入项 | 用途 |
|------|--------|------|
| `os` | `homedir`, `userInfo` | 获取用户主目录和用户名 |
| `path` | `join` | 路径拼接 |

### 外部系统交互

本模块**不直接**与外部系统交互，仅提供常量定义。实际的外部交互发生在：

- **macOS**: `rawRead.ts` 调用 `plutil` 读取 `/Library/Managed Preferences/*.plist`
- **Windows**: `rawRead.ts` 调用 `reg query` 读取注册表

### 环境变量依赖

| 变量名 | 用途 |
|--------|------|
| `USER_TYPE` | 值为 `'ant'` 时启用用户可写偏好路径（内部测试） |

## 风险、边界与改进建议

### 已知风险

1. **用户名获取失败**
   - 风险：`userInfo()` 可能在某些容器/受限环境中抛出异常
   - 缓解：已使用 try-catch 包裹，失败时仅跳过用户级路径

2. **硬编码路径**
   - 风险：macOS 系统路径未来可能变更
   - 缓解：使用标准 macOS MDM 路径，变更概率极低

3. **Windows WOW64 依赖**
   - 风险：若 Microsoft 更改共享注册表密钥行为
   - 缓解：使用官方文档推荐的 `SOFTWARE\Policies` 路径

### 边界条件

| 场景 | 行为 |
|------|------|
| 非 macOS 平台调用 `getMacOSPlistPaths()` | 正常返回路径数组（调用方负责平台判断） |
| `userInfo()` 抛出异常 | 仅跳过用户级路径，返回设备级路径 |
| `USER_TYPE !== 'ant'` | 不返回用户可写路径 |
| 路径指向的文件不存在 | 本模块不检查，由 `rawRead.ts` 处理 |

### 改进建议

1. **路径存在性检查**
   - 当前：仅返回路径，不检查存在性
   - 建议：考虑添加可选的同步存在性检查（需权衡性能影响）

2. **日志记录**
   - 当前：无日志输出
   - 建议：在调试模式下输出构建的路径列表（需避免循环依赖）

3. **平台抽象**
   - 当前：仅提供 macOS 路径构建器
   - 建议：若未来支持更多平台，可统一为 `getManagedConfigPaths()` 接口

4. **测试覆盖**
   - 当前：无直接单元测试（通过集成测试间接覆盖）
   - 建议：添加单元测试验证路径构建逻辑，特别是 `USER_TYPE` 条件分支

### 安全考量

1. **路径注入防护**
   - `username` 来自系统 API，不直接拼接用户输入，无注入风险
   - `MACOS_PREFERENCE_DOMAIN` 为硬编码常量，不可篡改

2. **信息泄露**
   - 路径信息可能泄露用户名（通过 `userInfo().username`）
   - 该信息仅用于本地文件读取，不上传或记录
