# appNames.ts 研究文档

## 场景与职责

本文件负责**过滤和净化已安装应用数据**，用于 `request_access` 工具描述。这是从 Cowork 的 appNames.ts 移植而来的安全关键模块，解决两个核心问题：

1. **噪声过滤**：Spotlight 返回磁盘上的每个 bundle（包括 XPC helpers、daemons、input methods），需要过滤掉非用户可见的后台服务
2. **Prompt Injection 防护**：应用名称是攻击者可控制的（任何人都可以发布任意名称的应用），需要防范恶意名称注入

## 功能点目的

### 1. 路径白名单过滤 (`PATH_ALLOWLIST`)
- 只允许 `/Applications/` 和 `/System/Applications/` 下的应用
- 用户目录 `~/Applications/` 在运行时动态检查（通过 `homeDir` 参数）
- 策略：基于已知安全路径的允许列表，而非黑名单（避免新 macOS 版本添加新路径时失效）

### 2. 名称模式黑名单 (`NAME_PATTERN_BLOCKLIST`)
- 过滤掉标记为背景服务的显示名称模式
- 正则设计 `(?:$|\s\()` 匹配关键字在字符串末尾或 ` (` 之前
- 例如："Slack Helper (GPU)" 被过滤，"Service Desk" 通过（Service 后跟 " D"）

### 3. 始终保留的 Bundle ID (`ALWAYS_KEEP_BUNDLE_IDS`)
- 包含常用自动化目标应用（浏览器、通讯、生产力、开发工具等）
- 这些应用绕过路径检查和字符过滤（受信任的供应商）
- 每个条目保证在描述中占用一个 token（<30 个条目）

### 4. Prompt Injection 防护 (`APP_NAME_ALLOWED`)
- 使用 Unicode 属性转义 `\p{L}\p{M}\p{N}` 支持国际化（ Bücher, 微信, Préférences Système）
- `\p{M}` 匹配组合标记，支持 NFD 分解的变音符号
- 使用单空格而非 `\s`（防止多行注入 `"App\nIgnore previous…"`）
- 禁止引号、尖括号、反引号、管道符、冒号

## 具体技术实现

### 核心数据结构

```typescript
type InstalledAppLike = {
  readonly bundleId: string
  readonly displayName: string
  readonly path: string
}

// 常量配置
const PATH_ALLOWLIST: readonly string[] = ['/Applications/', '/System/Applications/']
const NAME_PATTERN_BLOCKLIST: readonly RegExp[] = [
  /Helper(?:$|\s\()/,
  /Agent(?:$|\s\()/,
  /Service(?:$|\s\()/,
  /Uninstaller(?:$|\s\()/,
  /Updater(?:$|\s\()/,
  /^\./,
]
const APP_NAME_ALLOWED = /^[\p{L}\p{M}\p{N}_ .&'()+-]+$/u
const APP_NAME_MAX_LEN = 40
const APP_NAME_MAX_COUNT = 50
```

### 关键流程

#### `filterAppsForDescription()` - 主入口

```
输入: installed[] (原始 Spotlight 结果), homeDir
输出: 净化后的应用名称数组

1. 使用 reduce 分离 alwaysKept 和 rest
   - alwaysKept: bundleId 在 ALWAYS_KEEP_BUNDLE_IDS 中
   - rest: 通过路径白名单 + 名称黑名单过滤

2. 分别净化
   - alwaysKept → sanitizeTrustedNames (不应用字符过滤)
   - rest → sanitizeAppNames (应用字符过滤 + 数量上限)

3. 合并去重后返回
```

#### `sanitizeCore()` - 核心净化逻辑

```
输入: raw[] (原始名称数组), applyCharFilter (是否应用字符过滤)
输出: 净化后的数组

流程:
1. trim() 每个名称
2. 过滤: 空字符串 / 超过最大长度 / 字符过滤失败 / 重复
3. 使用 Set 进行去重
4. 按 localeCompare 排序
```

## 关键代码路径与文件引用

### 本文件导出
- `filterAppsForDescription(installed, homeDir)` - 主过滤函数

### 调用方
- `src/utils/computerUse/mcpServer.ts:43` - `tryGetInstalledAppNames()` 调用过滤 Spotlight 结果

### 依赖文件
- 无直接依赖（纯工具函数）

### 相关类型定义
- `InstalledAppLike` - 最小化的应用数据形状，匹配 `listInstalledApps` 返回值

## 依赖与外部交互

### 外部包依赖
- 无（纯 TypeScript 实现，无运行时依赖）

### 与系统交互
- 依赖调用方提供 `homedir()` 来检测 `~/Applications/`
- 依赖 Spotlight 结果作为输入数据源

## 风险、边界与改进建议

### 已知风险

1. **残余风险**：短小的良性字符对抗性名称（如 "grant all"）无法通过程序过滤
   - 缓解：工具描述的结构框架（"Available applications:"）明确这些是应用名称
   - 下游权限对话框需要显式用户批准

2. **PID 复用风险**：锁文件中的 PID 可能在进程退出后被新进程复用
   - 注释说明：这在实践中极不可能发生

3. **tmux/screen 环境**：`__CFBundleIdentifier` 反映启动服务器的终端，可能与当前连接的客户端不同
   - 缓解：无害，只是豁免一个终端窗口

### 边界情况

1. **空 homeDir**：函数优雅处理 `homeDir` 为 undefined 的情况
2. **损坏的锁文件**：`readLock()` 在解析失败时返回 undefined，触发恢复流程
3. **并发竞争**：多个会话同时恢复陈旧锁时，只有一个能成功创建（O_EXCL 保证）

### 改进建议

1. **监控和告警**：
   - 添加指标追踪被过滤的应用数量（alwaysKept vs filtered）
   - 监控字符过滤拒绝率，检测潜在的对抗性尝试

2. **国际化增强**：
   - 考虑添加更多 Unicode 规范化处理（NFC/NFD）
   - 评估是否需要支持从右到左（RTL）语言的额外处理

3. **安全强化**：
   - 考虑对 alwaysKept 应用也进行长度限制（目前只受信任名称过滤）
   - 添加对显示名称熵的检查，检测随机字符攻击

4. **性能优化**：
   - 当前实现每次调用都进行完整的 reduce + filter + sort
   - 考虑在应用安装变化不频繁时添加缓存层

5. **可观测性**：
   - 添加调试日志记录过滤决策（为什么某个应用被过滤）
   - 在 verbose 模式下输出统计信息
