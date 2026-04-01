# src/utils/queryProfiler.ts 研究文档

## 场景与职责

`queryProfiler.ts` 是 Claude Code 主查询链路（从用户输入到首 token 返回）的性能分析器。通过设置环境变量 `CLAUDE_CODE_PROFILE_QUERY=1` 启用，它利用 Node.js `perf_hooks` 在查询管道的关键节点打 mark，最终输出包含时间线、阶段分解、TTFT（Time To First Token）和内存快照的详细报告。该模块是排查查询卡顿、定位瓶颈的核心工具。

## 功能点目的

1. **查询生命周期打点（`startQueryProfile`、`queryCheckpoint`、`endQueryProfile`）**
   - `startQueryProfile`：清空之前的 mark 和内存快照，递增查询计数，记录 `query_user_input_received`。
   - `queryCheckpoint(name)`：在指定名称处打 mark，并记录当前 `process.memoryUsage()`。
   - `endQueryProfile`：记录 `query_profile_end` 作为收尾。

2. **格式化报告输出（`getQueryProfileReport`）**
   - 以第一个 mark 为 baseline，输出每阶段的相对时间和增量时间。
   - 标注慢操作警告：`>100ms` 标 `SLOW`，`>1000ms` 标 `VERY SLOW`；对 `git_status`、`tool_schema`、`client_creation` 等已知瓶颈降低阈值到 50ms。
   - 计算 TTFT 分解：pre-request overhead vs network latency，并给出百分比。
   - 输出阶段分解（Phase Breakdown）：Context loading、Microcompact、Autocompact、Query setup、Tool schemas、Message normalization、Client creation、Network TTFB、Tool execution 等。

3. **日志输出（`logQueryProfileReport`）**
   - 将完整报告通过 `logForDebugging` 输出，可在 `--debug` 模式或 `~/.claude/debug/latest` 中查看。

## 具体技术实现

- 模块加载时读取 `CLAUDE_CODE_PROFILE_QUERY` 环境变量，决定 `ENABLED` 常量；非启用状态下所有函数都是空操作，开销极小。
- 使用 `profilerBase.ts` 提供的 `getPerformance()` 访问 `perf_hooks.performance`。
- 内存快照存储在 `Map<string, MemoryUsage>` 中，以 mark 名称作为 key；这与 `startupProfiler` 的数组存储方式不同，需要注意同名 mark 重复调用时的覆盖问题（query 路径中同名 mark 通常只出现一次）。
- `firstTokenTime` 单独记录，用于快速获取 TTFT。

## 关键代码路径与文件引用

- **本文件**：`src/utils/queryProfiler.ts`
- **调用方（打点位置）**：
  - `src/services/api/claude.ts` — API 请求发送、响应头到达、首 chunk 到达等关键节点。
  - `src/query.ts` — `query()` 函数入口、递归调用、工具执行等。
  - `src/screens/REPL.tsx` — 用户输入接收、查询结束等。
  - `src/utils/handlePromptSubmit.ts` — 提交 prompt 时调用 `startQueryProfile`。
  - `src/utils/processUserInput/processUserInput.ts` — 处理用户输入的各个子阶段。
- **依赖**：
  - `src/utils/debug.js` — `logForDebugging`。
  - `src/utils/envUtils.js` — `isEnvTruthy`。
  - `src/utils/profilerBase.js` — `formatMs`、`formatTimelineLine`、`getPerformance`。

## 依赖与外部交互

- 仅依赖 Node.js `perf_hooks` 和 `process.memoryUsage()`。
- 无网络或外部进程交互。

## 风险、边界与改进建议

1. **同名 mark 覆盖内存快照**：与 `startupProfiler` 不同，`queryProfiler` 使用 `Map` 存储内存快照。如果某个 checkpoint 名称被多次调用（如 `query_tool_execution_start` 在递归查询中可能出现多次），后一次的内存数据会覆盖前一次，导致报告中只保留最后一次的值。当前 query 路径中大多数 mark 是唯一的，但这不是严格保证的。
2. **`query_user_input_received` 的特殊处理**：该 mark 的 delta 是从进程启动到第一个查询开始的时间，因此 `getSlowWarning` 明确跳过对它的慢操作标记，避免误报。但如果用户长时间空闲后发起查询，这个 delta 可能非常大，报告中却没有任何提示，可能让新用户困惑。
3. **TTFT 计算的假设**：`preRequestOverhead` 被简单定义为 `apiRequestSentTime`（从 baseline 到请求发送的时间），这包含了所有本地处理时间。如果网络层有重试或连接建立耗时，这部分会被算入 `networkLatency`，但实际上可能是本地代理/TLS 握手时间。
4. **阶段分解的硬编码**：`getPhaseSummary` 中的阶段名称和对应 mark 名称是硬编码的，如果 query 流程中 mark 名称变更或新增阶段，这里需要同步维护，容易遗漏。
5. **改进建议**：
   - 将内存快照存储改为数组（与 `startupProfiler` 一致），或在 Map 中存储数组以支持同名 mark 的多值记录。
   - 为 `query_user_input_received` 增加一个特殊注释（如 `[since process start]`），让报告读者明白其含义。
   - 考虑把阶段定义提取为配置文件或常量数组，减少硬编码带来的维护成本。
   - 增加一个 `queryProfilerExport()` 函数，以 JSON 格式导出原始 mark 数据，方便外部工具（如 Chrome DevTools Performance 面板）做可视化分析。
