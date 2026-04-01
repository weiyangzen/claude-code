# pluginCliCommands.ts 研究文档

## 场景与职责

`pluginCliCommands.ts` 是 Claude Code 插件系统的 **CLI 命令包装层**，为 `claude plugin *` 子命令提供非交互式的命令行接口。

### 核心定位
- **CLI 专用层**：处理 CLI 特有的副作用（console 输出、进程退出）
- **薄包装器**：核心逻辑委托给 `pluginOperations.ts`，本层只处理 CLI 表现
- **遥测收集点**：所有 CLI 操作都记录分析事件

### 提供的命令

| 命令 | 函数 | 描述 |
|------|------|------|
| `claude plugin install` | `installPlugin()` | 安装插件到指定 scope |
| `claude plugin uninstall` | `uninstallPlugin()` | 卸载指定 scope 的插件 |
| `claude plugin enable` | `enablePlugin()` | 启用插件 |
| `claude plugin disable` | `disablePlugin()` | 禁用插件 |
| `claude plugin disable --all` | `disableAllPlugins()` | 禁用所有插件 |
| `claude plugin update` | `updatePluginCli()` | 更新插件到最新版本 |

---

## 功能点目的

### 1. 统一错误处理 (`handlePluginCommandError`)

**目的**：为所有插件 CLI 命令提供一致的错误处理和遥测上报。

**行为**：
- 记录错误日志
- 输出用户友好的错误信息（使用 `figures.cross` 图标）
- 发送 `tengu_plugin_command_failed` 分析事件
- 进程以退出码 1 退出

**遥测字段**：
```typescript
{
  command: PluginCliCommand,
  error_category: string,
  _PROTO_plugin_name?: string,      // PII 标记
  _PROTO_marketplace_name?: string, // PII 标记
  // 其他遥测字段...
}
```

### 2. 插件安装 (`installPlugin`)

**目的**：非交互式安装插件。

**关键特性**：
- 支持 scope 参数（user/project/local，默认 user）
- 安装成功后输出成功信息并退出
- 发送 `tengu_plugin_installed_cli` 分析事件

### 3. 插件卸载 (`uninstallPlugin`)

**目的**：非交互式卸载插件。

**关键特性**：
- 支持 `--keep-data` 标志保留插件数据目录
- 发送 `tengu_plugin_uninstalled_cli` 分析事件

### 4. 插件启用/禁用 (`enablePlugin` / `disablePlugin`)

**目的**：切换插件启用状态。

**关键特性**：
- scope 可选，省略时自动检测最具体的 scope
- 发送 `tengu_plugin_enabled_cli` / `tengu_plugin_disabled_cli` 分析事件

### 5. 禁用所有插件 (`disableAllPlugins`)

**目的**：一键禁用所有已启用的插件。

**使用场景**：
- 故障排查时快速隔离所有插件
- 发送 `tengu_plugin_disabled_all_cli` 分析事件

### 6. 插件更新 (`updatePluginCli`)

**目的**：非交互式更新插件到最新版本。

**关键特性**：
- 使用 `gracefulShutdown()` 而非直接 `process.exit()`，确保优雅关闭
- 发送 `tengu_plugin_updated_cli` 分析事件（包含版本信息）

---

## 具体技术实现

### 命令类型定义

```typescript
type PluginCliCommand =
  | 'install'
  | 'uninstall'
  | 'enable'
  | 'disable'
  | 'disable-all'
  | 'update'
```

### 错误处理流程

```
命令执行
    ↓
try {
    ↓
  调用 pluginOperations.ts 对应函数
    ↓
  检查 result.success
    ↓
  输出成功信息 + 发送成功事件
    ↓
  process.exit(0)
} catch (error) {
    ↓
  handlePluginCommandError(error, command, plugin)
    ↓
  1. logError(error)
  2. console.error(`${figures.cross} Failed to ...`)
  3. logEvent('tengu_plugin_command_failed', {...})
  4. process.exit(1)
}
```

### 遥测实现细节

**PII 处理**：
```typescript
// _PROTO_* 前缀表示 PII 标记字段，路由到特权 BQ 列
logEvent('tengu_plugin_installed_cli', {
  _PROTO_plugin_name: name as AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED,
  _PROTO_marketplace_name: marketplace as AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED,
  scope: result.scope as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  install_source: 'cli-explicit' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  ...buildPluginTelemetryFields(name, marketplace, getManagedPluginNames()),
})
```

**错误分类**：
```typescript
import { classifyPluginCommandError } from '../../utils/telemetry/pluginTelemetry.js'

logEvent('tengu_plugin_command_failed', {
  command,
  error_category: classifyPluginCommandError(error),
  ...
})
```

### 与 pluginOperations.ts 的分工

| 职责 | pluginCliCommands.ts | pluginOperations.ts |
|------|---------------------|---------------------|
| console 输出 | ✅ 负责 | ❌ 不输出 |
| process.exit() | ✅ 负责 | ❌ 不调用 |
| 核心逻辑 | ❌ 委托 | ✅ 实现 |
| 遥测发送 | ✅ 包装层添加 | ❌ 不发送 |
| 结果对象处理 | ✅ 转换为用户消息 | ✅ 返回结果对象 |

---

## 关键代码路径与文件引用

### 核心函数

| 函数 | 行号 | 描述 |
|------|------|------|
| `handlePluginCommandError` | 53-96 | 统一错误处理 |
| `installPlugin` | 103-146 | 安装命令 |
| `uninstallPlugin` | 153-188 | 卸载命令 |
| `enablePlugin` | 195-229 | 启用命令 |
| `disablePlugin` | 236-270 | 禁用命令 |
| `disableAllPlugins` | 275-293 | 禁用全部命令 |
| `updatePluginCli` | 300-344 | 更新命令 |

### 依赖文件

```typescript
// CLI 输出
import figures from 'figures'
import { writeToStdout } from '../../utils/process.js'

// 错误处理
import { errorMessage } from '../../utils/errors.js'
import { gracefulShutdown } from '../../utils/gracefulShutdown.js'
import { logError } from '../../utils/log.js'

// 插件工具
import { getManagedPluginNames } from '../../utils/plugins/managedPlugins.js'
import { parsePluginIdentifier } from '../../utils/plugins/pluginIdentifier.js'
import type { PluginScope } from '../../utils/plugins/schemas.js'

// 遥测
import {
  buildPluginTelemetryFields,
  classifyPluginCommandError,
} from '../../utils/telemetry/pluginTelemetry.js'
import {
  type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED,
  logEvent,
} from '../analytics/index.js'

// 核心操作
import {
  disableAllPluginsOp,
  disablePluginOp,
  enablePluginOp,
  type InstallableScope,
  installPluginOp,
  uninstallPluginOp,
  updatePluginOp,
  VALID_INSTALLABLE_SCOPES,
  VALID_UPDATE_SCOPES,
} from './pluginOperations.js'
```

### 调用方

- `src/cli/handlers/plugins.ts` - CLI 子命令处理器

```typescript
// src/cli/handlers/plugins.ts
import {
  disableAllPlugins,
  disablePlugin,
  enablePlugin,
  installPlugin,
  uninstallPlugin,
  updatePluginCli,
  VALID_INSTALLABLE_SCOPES,
  VALID_UPDATE_SCOPES,
} from '../../services/plugins/pluginCliCommands.js'
```

---

## 依赖与外部交互

### 与 pluginOperations.ts 的关系

```
pluginCliCommands.ts (CLI 包装层)
    ↓ 调用
pluginOperations.ts (核心逻辑层)
    ↓ 调用
各种 utils/plugins/* 工具函数
```

### 与 CLI 系统的关系

```
main.tsx 命令注册
    ↓
cli/handlers/plugins.ts (参数解析/验证)
    ↓
pluginCliCommands.ts (执行 + 输出 + 退出)
```

### 遥测流向

```
pluginCliCommands.ts
    ↓
services/analytics/index.ts: logEvent()
    ↓
分析平台 (BigQuery)
```

---

## 风险、边界与改进建议

### 已知风险

1. **进程退出问题**
   - 风险：`process.exit()` 可能中断正在进行的异步操作
   - 缓解：`updatePluginCli` 使用 `gracefulShutdown()`，其他命令待改进

2. **PII 泄露风险**
   - 风险：插件名和市场名可能包含敏感信息
   - 缓解：使用 `_PROTO_*` 前缀标记，路由到特权列

3. **错误分类准确性**
   - 风险：`classifyPluginCommandError` 可能无法准确分类所有错误
   - 现状：依赖错误消息字符串匹配

### 边界情况

| 场景 | 处理 |
|------|------|
| 插件已安装/已启用/已禁用 | 由下层 `pluginOperations.ts` 返回失败结果，本层转换为错误消息 |
| 网络失败 | 下层抛出异常，本层捕获并显示友好错误 |
| 权限不足 | 下层抛出异常，本层捕获并显示友好错误 |
| 无效 scope | 由上层 `cli/handlers/plugins.ts` 验证 |

### 改进建议

1. **统一使用 gracefulShutdown**
   - 当前：只有 `updatePluginCli` 使用 `gracefulShutdown()`
   - 建议：所有命令都使用，确保异步操作完成后再退出

2. **支持 JSON 输出**
   - 当前：仅支持人类可读的文本输出
   - 建议：增加 `--json` 标志，方便脚本集成

3. **批量操作**
   - 当前：一次只能操作一个插件
   - 建议：支持 `claude plugin install plugin1 plugin2 ...`

4. ** dry-run 模式**
   - 当前：所有操作立即执行
   - 建议：增加 `--dry-run` 标志，预览将要执行的操作

5. **更好的进度反馈**
   - 当前：安装完成后一次性输出结果
   - 建议：对于长时间操作（如下载），显示进度条

### 代码质量观察

1. **注释质量**：文件头部有清晰的职责说明，与 `pluginOperations.ts` 的分工明确
2. **类型安全**：使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_*` 类型标记确保遥测类型安全
3. **一致性**：所有命令遵循相同的模式（try/catch + 成功事件/失败事件）
