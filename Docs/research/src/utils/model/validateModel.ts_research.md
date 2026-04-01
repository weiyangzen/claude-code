# Research: src/utils/model/validateModel.ts

## 场景与职责

当用户在 `/model` 命令或设置中输入一个非 alias 的自定义模型名称（如完整的模型 ID 或 Bedrock ARN）时，需要验证该模型是否真实可用。本模块通过发起一次真实的 API 调用（side query）来完成验证，并对常见错误提供友好的错误提示和 3P fallback 建议。

## 功能点目的

### `validateModel(model): Promise<{ valid: boolean; error?: string }>`

验证流程：
1. **空值检查**: 空字符串直接返回无效。
2. **Allowlist 检查**: 调用 `isModelAllowed()`，若不在白名单则提前拒绝。
3. **已知 Alias 检查**: 若输入是 `MODEL_ALIASES` 中的标准 alias，直接视为有效（无需浪费 API 调用）。
4. **自定义模型选项检查**: 若输入等于 `ANTHROPIC_CUSTOM_MODEL_OPTION` 环境变量，视为有效（假设管理员已预先验证）。
5. **缓存检查**: `validModelCache` 内存缓存，命中则直接返回有效。
6. **API 验证**: 通过 `sideQuery()` 发起一次最小化 API 调用：
   - `model`: 用户输入的模型名
   - `max_tokens: 1`
   - `maxRetries: 0`
   - `querySource: 'model_validation'`
   - 消息内容: `"Hi"`，并附加 `cache_control: { type: 'ephemeral' }`
7. **成功**: 写入缓存，返回 `valid: true`。
8. **失败**: 进入 `handleValidationError()`。

### `handleValidationError(error, modelName)`

按错误类型提供差异化提示：
- `NotFoundError` (404) → `"Model 'x' not found"`，并附加 3P fallback 建议（如 Opus 4.6 不可用时建议回退到 Opus 4.1）。
- `AuthenticationError` → 认证失败提示。
- `APIConnectionError` → 网络错误提示。
- 其他 APIError → 检查 error body 是否为 `not_found_error` 且 message 包含 `"model:"`，若是则返回模型未找到；否则返回通用 API 错误。
- 未知错误 → `"Unable to validate model: <message>"`。

### `get3PFallbackSuggestion(model): string | undefined`

仅对 3P provider 生效，按模型版本提供降级建议链：
- `opus-4-6` / `opus_4_6` → `getModelStrings().opus41`
- `sonnet-4-6` / `sonnet_4_6` → `getModelStrings().sonnet45`
- `sonnet-4-5` / `sonnet_4_5` → `getModelStrings().sonnet40`

## 关键代码路径与文件引用

- **入口**: `src/utils/model/validateModel.ts`
- **被调用方**:
  - `src/commands/model/model.tsx` — `/model <customModel>` 命令在设置模型前验证。
  - `src/tools/ConfigTool/supportedSettings.ts` — 配置工具中验证模型设置。
  - `src/commands/advisor.ts` — Advisor 相关模型验证。
- **依赖**:
  - `src/utils/model/aliases.ts` (`MODEL_ALIASES`)
  - `src/utils/model/modelAllowlist.ts` (`isModelAllowed`)
  - `src/utils/model/providers.ts` (`getAPIProvider`)
  - `src/utils/model/modelStrings.ts` (`getModelStrings`)
  - `src/utils/sideQuery.ts` (`sideQuery`)
  - `@anthropic-ai/sdk` (`NotFoundError`, `APIError`, `APIConnectionError`, `AuthenticationError`)

## 依赖与外部交互

- **网络**: 通过 `sideQuery()` 调用 Anthropic API（或 3P provider 的等效端点），产生一次真实的模型验证请求。
- **状态**: 使用模块级 `Map<string, boolean>` 内存缓存有效模型，进程重启后失效。
- **环境**: 读取 `ANTHROPIC_CUSTOM_MODEL_OPTION`。

## 风险、边界与改进建议

- **风险**: API 验证调用带有 `cache_control: { type: 'ephemeral' }`，虽然请求极小（1 token），但仍会计费并产生 prompt cache 写入成本。对高频场景（如批量设置验证）不够经济。不过当前调用频率低（仅用户手动输入时触发），可接受。
- **风险**: `validModelCache` 只有“有效”缓存，没有“无效”缓存。这意味着如果用户反复输入同一个错误模型名，每次都会发起新的 API 调用。建议增加负缓存（带 TTL），减少无效请求。
- **边界**: `maxRetries: 0` 意味着网络抖动会直接导致验证失败；用户可能看到 `"Network error"` 而非模型真正不可用。这是为了快速反馈，但可能误伤。
- **边界**: 对 `MODEL_ALIASES` 的短路检查使用 `(MODEL_ALIASES as readonly string[]).includes(lowerModel)`，因此大小写不敏感（已转小写）。但 `[1m]` 后缀的 alias（如 `sonnet[1m]`）已在 `MODEL_ALIASES` 中，无需额外处理。
- **改进建议**: 为 `validateModel` 增加单元测试，mock `sideQuery` 的各种异常类型，确保错误提示文案稳定。
- **改进建议**: 3P fallback 建议链目前硬编码了 Opus/Sonnet 的降级路径，若未来模型矩阵变化需要同步维护。可考虑将 fallback 规则与 `configs.ts` 中的模型版本顺序关联，自动生成降级建议。
