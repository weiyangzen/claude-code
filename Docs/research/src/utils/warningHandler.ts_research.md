# 研究文档：src/utils/warningHandler.ts

## 场景与职责

本模块负责安装和管理 **Node.js 全局 `process.on('warning')` 事件处理器**。在 Claude Code 这类长期运行的交互式 CLI 应用中，Node.js 运行时可能产生各种警告（如 `MaxListenersExceededWarning`、`DeprecationWarning` 等）。默认情况下，这些警告会输出到 `stderr`，干扰用户体验。本模块的职责是：

- **拦截所有运行时警告**，将其转化为结构化的 analytics 事件（`tengu_node_warning`）供后台监控；
- **对终端用户隐藏警告输出**（非开发模式下移除默认的 `stderr` 处理器）；
- **对已知内部警告进行标记和抑制**，避免用户被无害的 AbortSignal/EventTarget 监听器超限警告困扰；
- **防止内存泄漏**：对警告 key 进行有界计数，避免无限增长的 Map。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `initializeWarningHandler()` | 安装全局 warning 处理器。若已安装则跳过；非开发模式下会先移除 Node.js 默认的 `stderr` warning 监听器。 |
| `resetWarningHandler()` | 测试专用：移除已安装的处理器并清空计数状态。 |

## 具体技术实现

### 1. 开发模式检测

`isRunningFromBuildDirectory()` 通过检查 `process.argv[1]` 和 `process.execPath` 是否包含以下构建目录路径来判断：

- `/build-ant/`
- `/build-external/`
- `/build-external-native/`
- `/build-ant-native/`

在 Windows 上，先将反斜杠统一替换为正斜杠后再匹配。

若 `NODE_ENV === 'development'` 或 `isRunningFromBuildDirectory()` 为 `true`，则认为是开发模式，**保留** Node.js 默认的 `stderr` warning 输出；否则**移除**所有默认监听器，避免用户看到噪音。

### 2. 警告处理器逻辑

安装自定义处理器后，每次触发 `warning` 事件时执行：

1. **生成 warning key**：`${warning.name}: ${warning.message.slice(0, 50)}`，取前 50 个字符作为去重标识。
2. **有界计数**：
   - `MAX_WARNING_KEYS = 1000`。
   - 若 key 已存在，或 Map 大小未达上限，则递增计数；否则忽略（该 key 的 `occurrence_count` 将永远显示为 1）。
3. **内部警告标记**：
   - `INTERNAL_WARNINGS` 列表包含两个正则：
     - `MaxListenersExceededWarning.*AbortSignal`
     - `MaxListenersExceededWarning.*EventTarget`
   - 匹配到的警告标记 `is_internal: 1`，否则 `is_internal: 0`。
4. **Analytics 上报**：
   - 事件名：`tengu_node_warning`。
   - 字段：`is_internal`、`occurrence_count`、`classname`。
   - **Ant-only**：额外附带完整 `warning.message`（因为可能包含代码或文件路径，外部用户隐私敏感）。
5. **Debug 输出**：
   - 若 `CLAUDE_DEBUG` 为 truthy，调用 `logForDebugging` 输出带 `[Warning]` 或 `[Internal Warning]` 前缀的日志。
6. **静默失败**：整个处理器包裹在 `try/catch` 中，任何异常都被吞掉，防止处理器自身故障引发级联问题。

### 3. 去重安装

`initializeWarningHandler()` 会检查 `process.listeners('warning')` 中是否已包含本模块的 `warningHandler` 引用，避免重复安装。

## 关键代码路径与文件引用

- **主实现**：`src/utils/warningHandler.ts`（121 行）
- **启动调用**：`src/main.tsx`（进程启动时调用 `initializeWarningHandler()`）
- **Analytics**：`src/services/analytics/index.ts`（`logEvent`）
- **Debug 日志**：`src/utils/debug.ts`（`logForDebugging`）
- **平台检测**：`src/utils/platform.ts`（`getPlatform`）
- **环境工具**：`src/utils/envUtils.ts`（`isEnvTruthy`）

## 依赖与外部交互

- **`path`**（`posix`、`win32`）：路径格式转换。
- **`src/services/analytics/index.js`**：`logEvent`。
- **`src/utils/debug.js`**：`logForDebugging`。
- **`src/utils/envUtils.js`**：`isEnvTruthy`。
- **`src/utils/platform.js`**：`getPlatform`。
- 无外部 npm 依赖。

## 风险、边界与改进建议

### 风险

1. **开发模式检测的脆弱性**：`isRunningFromBuildDirectory()` 依赖硬编码的构建目录名称。若 CI/CD 流程或本地构建脚本将产物输出到自定义目录（如 `/dist/`、`/out/`），该函数会返回 `false`，导致开发构建中的警告被静默吞掉，开发者可能长时间看不到重要的运行时警告。
2. **`process.removeAllListeners('warning')` 的副作用**：非开发模式下，该调用会移除**所有** `warning` 监听器，包括第三方库（如 OpenTelemetry、测试框架）可能合法安装的监听器。虽然当前架构中这是预期行为，但引入新的依赖时可能导致隐蔽的调试困难。
3. **警告 key 的粒度问题**：key 由 `warning.name + message.slice(0, 50)` 组成。对于包含动态路径或动态数值的警告（如 `ENOENT: no such file or directory, open '/tmp/abc123'`），前 50 个字符可能包含变化部分，导致同一类警告产生大量不同的 key，迅速耗尽 1000 的 key 上限，后续 genuinely different 的警告被丢弃。

### 边界

- **仅处理 `process.on('warning')`**：V8 的 `UnhandledPromiseRejectionWarning`、未捕获的异常、或 `console.warn` 输出不会被此处理器捕获。
- **1000 key 上限是全局有界**：一旦达到上限，任何新的唯一 key 都不会再被跟踪，其 `occurrence_count` 恒为 1。这是内存安全的设计，但会丢失高频动态警告的聚合统计。
- **Ant-only 完整 message**：外部构建中，`warning.message` 不会发送到 analytics，仅发送分类名和计数，符合隐私政策。

### 改进建议

1. **增强开发模式检测**：除了硬编码目录名，还可检查 `process.env.CLAUDE_CODE_DEV` 或 `process.env.NODE_ENV === 'development'` 作为显式开关，让自定义构建目录的开发者也能保留默认警告输出。
2. **更安全的监听器移除**：将 `process.removeAllListeners('warning')` 替换为遍历并仅移除 Node.js 内置的默认 warning 监听器（可通过引用比较识别），保留第三方库安装的监听器。
3. **规范化警告 key**：在生成 key 前，对 `warning.message` 进行简单的动态内容剥离，例如将文件路径替换为 `<PATH>`、将随机 ID 替换为 `<ID>`。可维护一个小的正则替换列表，显著减少 key 爆炸问题。
4. **增加警告采样/限流**：除了 1000 key 的上限，可对单个 key 的 `occurrence_count` 增加对数采样（如只上报 1, 10, 100, 1000...），减少高频率重复警告对 analytics 后端的冲击。
5. **暴露警告计数查询接口**：当前 `warningCounts` Map 是模块级私有状态。可考虑提供一个 `getWarningStats()` 函数，供调试命令（如 `/debug`）或崩溃报告时导出最近的警告摘要，帮助排查问题。
