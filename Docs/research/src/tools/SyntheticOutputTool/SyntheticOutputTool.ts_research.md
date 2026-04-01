# SyntheticOutputTool 深度研究文档

> 研究对象：`src/tools/SyntheticOutputTool/SyntheticOutputTool.ts`  
> 研究范围：代码实现、调用链路、配置入口、测试上下文、依赖交互  
> 执行器：kimi (k2p5)  
> 生成时间：2026-04-01

---

## 一、场景与职责

### 1.1 定位
`SyntheticOutputTool`（对外工具名 `StructuredOutput`）是一个**合成输出工具**，专门用于**非交互式会话**（headless / SDK / `--print` 模式）。它的核心职责是：
- 接收运行时动态注入的 JSON Schema；
- 将该 Schema 作为工具参数描述暴露给大模型；
- 在工具调用阶段对模型传入的数据进行 **Ajv 校验**；
- 将校验通过的结构化数据作为 `structured_output` 返回给调用方（SDK/CLI）。

### 1.2 使用场景
| 场景 | 说明 |
|------|------|
| `--json-schema` CLI 参数 | 用户在 `claude -p` 时传入自定义 JSON Schema，要求模型按 Schema 返回最终答案。 |
| SDK `jsonSchema` 初始化 | SDK 消费者通过控制协议传入 `jsonSchema`，触发结构化输出能力。 |
| Agent Hook 验证 | `execAgentHook` 内部使用该工具强制 hook agent 返回 `{ok, reason?}` 格式的验证结果。 |
| Coordinator 模式 | 协调器（coordinator mode）被允许使用该工具，以便在编排多 worker 后返回结构化汇总。 |

### 1.3 设计约束
- **仅非交互式可用**：`isSyntheticOutputToolEnabled()` 显式检查 `isNonInteractiveSession`。
- **非用户可控工具**：在 `src/tools.ts` 的 `getTools()` 中被列入 `specialTools`，不会出现在普通工具列表里；它由代码路径显式注入。
- **只读 & 并发安全**：`isReadOnly = true`，`isConcurrencySafe = true`，不会触发权限提示，也不会阻塞其他工具。

---

## 二、功能点目的

### 2.1 功能清单

| 功能 | 目的 |
|------|------|
| **动态 Schema 绑定** | 同一份代码支持任意 JSON Schema，无需为每种输出格式写死一个工具。 |
| **Ajv 编译缓存** | 对相同 Schema 对象引用做 `WeakMap` 缓存，避免 80 次工作流调用重复编译（~110ms → ~4ms）。 |
| **输入校验** | 模型若返回不符合 Schema 的数据，工具调用会抛出 `TelemetrySafeError`，并在结果中提示 Schema mismatch。 |
| **强制调用 Enforcement** | 通过 `registerStructuredOutputEnforcement` 注册 Stop hook，若模型在最终响应前未调用该工具，会被系统提示强制调用。 |
| **重试熔断** | `QueryEngine` 统计本次查询内 `StructuredOutput` 的调用次数，超过 `MAX_STRUCTURED_OUTPUT_RETRIES`（默认 5，可 env 覆盖）即返回 `error_max_structured_output_retries`。 |

### 2.2 与相关功能的边界
- 不同于 **API 原生 `response_format: json_schema`**：该工具走的是 Claude 的 Function Calling 路径，兼容所有支持 tool use 的模型，不依赖后端特定字段。
- 不同于普通业务工具（Bash/Edit/Read）：它不参与文件系统或网络操作，仅做数据校验与透传。
- 不同于 `TaskOutputTool`：`TaskOutputTool` 用于子 agent 向父 agent 汇报结果；`SyntheticOutputTool` 用于主会话向 SDK/CLI 消费者返回结构化数据。

---

## 三、具体技术实现

### 3.1 文件结构
```
src/tools/SyntheticOutputTool/SyntheticOutputTool.ts
├── 基础 Schema 定义 (inputSchema / outputSchema)
├── isSyntheticOutputToolEnabled()
├── SyntheticOutputTool (buildTool 基础定义)
├── createSyntheticOutputTool()      ← 对外工厂
├── buildSyntheticOutputTool()       ← 内部构造 + Ajv 编译
└── toolCache (WeakMap 缓存)
```

### 3.2 核心类型与数据结构

#### 3.2.1 工具定义（`ToolDef`）
```ts
export const SyntheticOutputTool = buildTool({
  isMcp: false,
  isEnabled: () => true,           // 创建后始终启用
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  isOpenWorld: () => false,
  name: SYNTHETIC_OUTPUT_TOOL_NAME, // 'StructuredOutput'
  searchHint: 'return the final response as structured JSON',
  maxResultSizeChars: 100_000,
  // ... description / prompt / schemas / call / checkPermissions / render methods
})
```

#### 3.2.2 动态构造结果
```ts
type CreateResult = { tool: Tool<InputSchema> } | { error: string }
```
- 成功：返回一个 **继承自 `SyntheticOutputTool` 但覆盖 `inputJSONSchema` 和 `call`** 的新工具对象。
- 失败：返回 `{error}`，包含 Ajv 的 schema 诊断信息（如 `data/properties/bugs should be object`）。

#### 3.2.3 Ajv 校验流程
```ts
const ajv = new Ajv({ allErrors: true })
const isValidSchema = ajv.validateSchema(jsonSchema)
if (!isValidSchema) return { error: ajv.errorsText(ajv.errors) }

const validateSchema = ajv.compile(jsonSchema)

// 在 tool.call 中
const isValid = validateSchema(input)
if (!isValid) {
  const errors = validateSchema.errors?.map(e => `${e.instancePath || 'root'}: ${e.message}`).join(', ')
  throw new TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS(...)
}
```

### 3.3 关键流程：从 CLI 参数到结构化输出返回

#### 3.3.1 CLI 入口（`src/main.tsx`）
1. 解析 `--json-schema` 参数（字符串）。
2. 判断 `isSyntheticOutputToolEnabled({ isNonInteractiveSession })`。
3. 若启用，用 `jsonParse(options.jsonSchema)` 得到对象，再调用 `createSyntheticOutputTool(jsonSchema)`。
4. 若返回 `tool`，则 `tools = [...tools, syntheticOutputResult.tool]`。
5. 记录 telemetry：`tengu_structured_output_enabled` 或 `tengu_structured_output_failure`。
6. 将 `jsonSchema` 一路透传给 `runHeadless`。

#### 3.3.2 SDK/Headless 入口（`src/cli/print.ts`）
1. 控制协议初始化时可能调用 `setInitJsonSchema(request.jsonSchema)`。
2. 在 `runHeadlessStreaming` 准备工具池时：
   ```ts
   const initJsonSchema = getInitJsonSchema()
   if (initJsonSchema && !options.jsonSchema) {
     const syntheticOutputResult = createSyntheticOutputTool(initJsonSchema)
     if ('tool' in syntheticOutputResult) {
       allTools = [...allTools, syntheticOutputResult.tool]
     }
   }
   ```

#### 3.3.3 QueryEngine 执行与 Enforcement（`src/QueryEngine.ts`）
1. `submitMessage` 中检查：
   ```ts
   const hasStructuredOutputTool = tools.some(t => toolMatchesName(t, SYNTHETIC_OUTPUT_TOOL_NAME))
   if (jsonSchema && hasStructuredOutputTool) {
     registerStructuredOutputEnforcement(setAppState, getSessionId())
   }
   ```
2. `registerStructuredOutputEnforcement`（`src/utils/hooks/hookHelpers.ts`）注册一个 `Stop` 事件 function hook：
   - 检查消息历史中是否存在成功的 `StructuredOutput` 工具调用（`hasSuccessfulToolCall`）。
   - 若无，则向模型追加系统提示：`You MUST call the StructuredOutput tool to complete this request. Call this tool now.`
3. 在 `query()` 的消息循环中，当遇到 `attachment.type === 'structured_output'` 时：
   ```ts
   structuredOutputFromTool = message.attachment.data
   ```
4. 最终 `result` 消息包含：
   ```ts
   {
     type: 'result',
     subtype: 'success',
     structured_output: structuredOutputFromTool,
     // ... 其他字段
   }
   ```

#### 3.3.4 重试熔断逻辑（`src/QueryEngine.ts`）
```ts
const initialStructuredOutputCalls = jsonSchema
  ? countToolCalls(this.mutableMessages, SYNTHETIC_OUTPUT_TOOL_NAME)
  : 0

// 在 message loop 中，每遇到 user message 时检查
if (message.type === 'user' && jsonSchema) {
  const currentCalls = countToolCalls(this.mutableMessages, SYNTHETIC_OUTPUT_TOOL_NAME)
  const callsThisQuery = currentCalls - initialStructuredOutputCalls
  const maxRetries = parseInt(process.env.MAX_STRUCTURED_OUTPUT_RETRIES || '5', 10)
  if (callsThisQuery >= maxRetries) {
    yield { type: 'result', subtype: 'error_max_structured_output_retries', ... }
    return
  }
}
```

### 3.4 Agent Hook 中的使用（`src/utils/hooks/execAgentHook.ts`）
1. `execAgentHook` 调用 `createStructuredOutputTool()`（来自 `hookHelpers.ts` 的简化版本，固定 Schema 为 `{ok: boolean, reason?: string}`）。
2. 为避免与父上下文的 `--json-schema` 冲突，先过滤掉已有的 `StructuredOutput`：
   ```ts
   const filteredTools = toolUseContext.options.tools.filter(
     tool => !toolMatchesName(tool, SYNTHETIC_OUTPUT_TOOL_NAME)
   )
   ```
3. 将过滤后的工具 + 新的 `structuredOutputTool` 注入 hook agent 的工具池。
4. hook agent 的系统提示明确要求最终调用 `StructuredOutput` 工具返回验证结果。
5. 消息循环中监听 `attachment.type === 'structured_output'`，解析后决定 hook 的 `outcome`（`success` / `blocking` / `cancelled`）。

---

## 四、关键代码路径与文件引用

### 4.1 目标文件
- `src/tools/SyntheticOutputTool/SyntheticOutputTool.ts` — 工具本体、工厂函数、Ajv 编译与缓存。

### 4.2 直接调用方
| 文件 | 调用点 | 作用 |
|------|--------|------|
| `src/main.tsx` | `createSyntheticOutputTool(jsonSchema)` | CLI `--json-schema` 参数注入工具。 |
| `src/cli/print.ts` | `createSyntheticOutputTool(initJsonSchema)` | SDK headless 模式注入工具。 |
| `src/QueryEngine.ts` | `registerStructuredOutputEnforcement` | 注册强制调用 hook；提取 `structured_output`；重试熔断。 |
| `src/utils/hooks/execAgentHook.ts` | `createStructuredOutputTool()` + 过滤已有工具 | Agent hook 内部验证。 |
| `src/utils/hooks/hookHelpers.ts` | `createStructuredOutputTool()` / `registerStructuredOutputEnforcement` | Hook 辅助函数与 enforcement 注册。 |

### 4.3 策略/常量配置方
| 文件 | 引用方式 | 作用 |
|------|----------|------|
| `src/constants/tools.ts` | `SYNTHETIC_OUTPUT_TOOL_NAME` | 定义 `ASYNC_AGENT_ALLOWED_TOOLS`、`COORDINATOR_MODE_ALLOWED_TOOLS` 包含该工具。 |
| `src/coordinator/coordinatorMode.ts` | `SYNTHETIC_OUTPUT_TOOL_NAME` | Coordinator 的系统提示与允许工具列表包含它。 |
| `src/tools.ts` | `SYNTHETIC_OUTPUT_TOOL_NAME` | `getTools()` 中将其加入 `specialTools` 集合，排除在普通工具过滤之外。 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `SYNTHETIC_OUTPUT_TOOL_NAME` | 进度跟踪中排除该工具（视为内部工具，不展示在 activity preview）。 |

### 4.4 基础设施依赖
| 文件 | 依赖项 | 说明 |
|------|--------|------|
| `src/Tool.ts` | `buildTool`, `ToolDef`, `Tool`, `toolMatchesName` | 工具类型系统与构建工厂。 |
| `src/utils/lazySchema.ts` | `lazySchema` | 延迟初始化 Zod schema，避免顶层副作用。 |
| `src/utils/errors.ts` | `TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | 用于 Ajv 校验失败时抛出可安全上报的错误。 |
| `src/utils/slowOperations.ts` | `jsonStringify` | UI 渲染中序列化输入字段。 |
| `src/utils/messages.ts` | `countToolCalls`, `hasSuccessfulToolCall` | 重试计数与 enforcement 条件判断。 |
| `src/utils/hooks/sessionHooks.ts` | `addFunctionHook` | enforcement 的底层 hook 注册机制。 |
| `src/utils/attachments.ts` | `type: 'structured_output'` | Attachment 类型定义。 |
| `src/bootstrap/state.ts` | `setInitJsonSchema`, `getInitJsonSchema` | SDK 模式下 schema 的跨调用状态存储。 |

---

## 五、依赖与外部交互

### 5.1 第三方库
- **`ajv`**（`Ajv`）：JSON Schema 的元校验（`validateSchema`）与运行期校验（`compile`）。使用 `{ allErrors: true }` 以收集全部错误信息。
- **`zod/v4`**：定义基础 `inputSchema`（`z.object({}).passthrough()`）与 `outputSchema`（`z.string()`）。

### 5.2 与系统其他模块的交互
1. **权限系统**：`checkPermissions` 永远返回 `allow`，因为工具只读且不涉及外部资源；不会触发用户确认弹窗。
2. **Telemetry**：
   - `tengu_structured_output_enabled` — 启用时上报，附带 `schema_property_count`、`has_required_fields`。
   - `tengu_structured_output_failure` — Schema 无效时上报。
   - `tengu_agent_stop_hook_*` — Agent hook 路径中的成功/失败/超时事件。
3. **Prompt Cache**：`inputJSONSchema` 是动态生成的，会参与系统提示的构造；在 `src/tools.ts` 的 `assembleToolPool` 中通过按名称排序保证缓存稳定性。
4. **MCP / 插件工具池**：`SyntheticOutputTool` 不是 MCP 工具（`isMcp: false`），但在 `assembleToolPool` 时与 MCP 工具合并；若名称冲突，内置工具优先（`uniqBy` 保留插入顺序）。

---

## 六、风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 | 代码体现 |
|------|------|----------|
| **Schema 注入失败静默处理** | `createSyntheticOutputTool` 返回 `{error}` 时，`main.tsx` 仅记录 telemetry，未向用户/stderr 输出错误，导致 `--json-schema` 无效时模型看不到该工具，用户也无感知。 | `src/main.tsx:1886-1899` |
| **Ajv JIT 开销** | 虽然做了 `WeakMap` 缓存，但如果每次传入**不同的 schema 对象**（即使内容相同），仍会重新 `new Ajv() + compile()`。 | `toolCache` 基于对象引用而非内容哈希。 |
| **重试计数包含历史消息** | `countToolCalls` 扫描整个 `mutableMessages`，若会话很长，计数逻辑是 O(N)；虽然 `maxCount` 可提前退出，但此处未传 `maxCount`。 | `src/QueryEngine.ts:1006-1009` |
| **Enforcement 与模型行为博弈** | Stop hook 只是追加系统提示，无法 100% 保证模型下一次一定调用工具；极端情况下可能反复触发 Stop hook 直到达到 `maxRetries`。 | `registerStructuredOutputEnforcement` 使用软性提示而非硬性协议约束。 |
| **并发场景下的 Schema 冲突** | 在 Agent Hook 中，父上下文若已有 `StructuredOutput`，会被过滤掉再替换成 hook 自己的 Schema；若父上下文也需要结构化输出，子 agent 的返回格式可能与父预期不同。 | `src/utils/hooks/execAgentHook.ts:91-95` |

### 6.2 边界条件
- **非交互式限制**：如果在交互式 REPL 中传入 `--json-schema`，`isSyntheticOutputToolEnabled` 返回 `false`，工具不会被创建；但 `main.tsx` 中 `options.jsonSchema` 仍会被解析并透传，只是 `QueryEngine` 检测不到对应工具，导致 `registerStructuredOutputEnforcement` 不触发。行为不一致。
- **空 Schema / 无 properties**：Ajv 允许 `{}` 作为合法 schema，此时模型可返回任意对象，工具校验永远通过。
- **超大 Schema**：`inputJSONSchema` 直接序列化进系统提示；若 Schema 极大，可能显著增加 token 消耗，甚至触碰上下文上限。目前无大小限制或截断逻辑。

### 6.3 改进建议
1. **显式错误暴露**：当 `createSyntheticOutputTool` 返回 `{error}` 时，在 CLI 入口向 `stderr` 输出诊断信息并建议用户检查 Schema，避免静默降级为普通文本输出。
2. **缓存增强**：考虑在 `WeakMap` 之外增加一层基于 `jsonStringify(schema)` 内容哈希的 `Map` 缓存，以应对内容相同但引用不同的 schema（常见于 SDK 每次新建对象）。
3. **性能优化**：`countToolCalls` 在重试检查处可传入 `maxCount = maxRetries + initialCalls`，减少长会话的扫描开销。
4. **Schema 大小限制**：增加对 `inputJSONSchema` 序列化后字符数的上限检查（如 8K/16K），超过时拒绝或提示用户，防止 token 爆炸。
5. **交互式支持评估**：若产品策略允许，可评估在 REPL 模式下也启用 `SyntheticOutputTool`（例如通过 `/json-schema` 命令动态切换），使交互式用户也能获得结构化输出能力。
6. **测试覆盖**：当前仓库中未找到针对 `SyntheticOutputTool` 的单元测试（`.test.ts` 搜索无结果）。建议补充：
   - Ajv 编译失败/成功的工厂函数测试；
   - `WeakMap` 缓存命中测试；
   - `QueryEngine` 重试熔断路径的集成测试；
   - `execAgentHook` 中 Schema 替换逻辑的测试。

---

## 七、附录：快速参考

### 7.1 环境变量
- `MAX_STRUCTURED_OUTPUT_RETRIES` — 结构化输出调用重试上限，默认 `5`。

### 7.2 CLI 参数
- `--json-schema <schema>` — 传入 JSON Schema 字符串，触发结构化输出。

### 7.3 关键常量
- `SYNTHETIC_OUTPUT_TOOL_NAME = 'StructuredOutput'`

### 7.4 结果子类型
- `error_max_structured_output_retries` — 超过最大重试次数时的结果类型。
