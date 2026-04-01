# 研究文档：src/utils/semanticBoolean.ts

## 场景与职责

`semanticBoolean.ts` 是一个**Zod schema 辅助工厂**，专门解决 LLM 生成 JSON tool inputs 时偶尔将布尔值写成字符串 `"true"` / `"false"` 的问题。

在 Claude Code 的工具调用链中，模型输出的 JSON 会经过 Zod 校验。若 schema 使用 `z.boolean()`，而模型输出了 `"replace_all": "false"`（带引号），Zod 会报类型错误。常见的错误修复 `z.coerce.boolean()` 更危险：它使用 JS 真值语义，`"false"` 会被判定为 `true`。

该模块提供一种**对模型透明、对客户端宽容**的解决方案：在 Zod 校验前做预处理，仅当值为 `"true"` / `"false"` 字符串时映射为对应布尔值，其他值原样透传给内层 schema。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `semanticBoolean(inner?)` | 生成一个 `ZodPipe`，预处理 `"true"`→`true`、`"false"`→`false`，然后交给内层 schema（默认 `z.boolean()`）校验。 |

---

## 具体技术实现

### 1. 实现代码

```ts
import { z } from 'zod/v4'

export function semanticBoolean<T extends z.ZodType>(
  inner: T = z.boolean() as unknown as T,
) {
  return z.preprocess(
    (v: unknown) => (v === 'true' ? true : v === 'false' ? false : v),
    inner,
  )
}
```

### 2. 关键设计决策

- **仅接受精确字符串匹配**：`v === 'true'` / `v === 'false'`，不使用 `JSON.parse` 或 `Boolean()`，避免 `"0"`、`""`、`"FALSE"` 等边缘情况被错误解释。
- **透传其他值**：非 `"true"/"false"` 的值原样传给 `inner`，让 `inner` 自行决定合法与否。这意味着如果模型输出了数字 `1` 或 `0`，默认的 `z.boolean()` 仍会拒绝——这是有意为之，保持布尔类型的严格性。
- **ZodPipe 与 API Schema 的兼容性**：`z.preprocess` 在 Zod v4 的 OpenAPI/JSON Schema 生成中通常仍会被底层 schema 的类型覆盖。注释说明 `{"type":"boolean"}` 仍会发给 API，模型被告知这是布尔值，字符串容忍是“不可见的客户端强制转换”。
- **`.optional()` / `.default()` 必须放在 inner 上**：注释强调 chaining 到 `ZodPipe` 会在 Zod v4 中将 `z.output<>` 推断为 `unknown`。使用方式：
  ```ts
  semanticBoolean(z.boolean().optional())  // → boolean | undefined
  semanticBoolean(z.boolean().default(false)) // → boolean
  ```

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/semanticBoolean.ts:22-29` | `semanticBoolean` 工厂函数。 |
| `src/tools/FileEditTool/types.ts` | 使用方：FileEditTool 的 `replace_all` 等布尔参数。 |
| `src/tools/BashTool/BashTool.tsx` | 使用方：Bash 工具参数。 |
| `src/tools/GrepTool/GrepTool.ts` | 使用方：Grep 工具参数。 |
| `src/tools/PowerShellTool/PowerShellTool.tsx` | 使用方：PowerShell 工具参数。 |
| `src/tools/ScheduleCronTool/CronCreateTool.ts` | 使用方：Cron 创建工具参数。 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | 使用方：消息发送工具参数。 |
| `src/tools/TaskOutputTool/TaskOutputTool.tsx` | 使用方：任务输出工具参数。 |

---

## 依赖与外部交互

- **第三方库**：`zod/v4`。
- **无 Node.js 内置模块依赖**。
- **调用方**：多个 Tool 定义文件中的 input schema 构建。

---

## 风险、边界与改进建议

### 风险与边界

1. **大小写敏感**：只匹配小写的 `"true"` / `"false"`。若模型输出 `"True"` 或 `"FALSE"`，预处理不会转换，底层 `z.boolean()` 会拒绝。考虑到 LLM 通常遵循 schema 类型提示且输出稳定为小写，这是可接受的严格性。

2. **Zod v4 的 schema 生成行为依赖实现细节**：注释声称 `z.preprocess` 会 emit `{"type":"boolean"}`，但这取决于 Zod v4 的具体版本和 schema 序列化逻辑。若未来 Zod 更新改变了 `ZodPipe` 的 JSON Schema 生成，API 可能将字段类型暴露为 `any` 或 `unknown`，导致模型更频繁地输出字符串。

3. **无单元测试覆盖（从代码库中未找到）**：该模块逻辑简单，但它是安全 gate（防止 `"false"` 被 coerce 成 `true`）的核心。缺少显式测试意味着如果某人误将 `z.coerce.boolean()` 重新引入，问题可能直到生产环境才暴露。

4. **与 `semanticNumber` 的对称性**：两者设计模式完全一致，但分别放在两个文件中。这是为了保持导入路径清晰，但也意味着任何模式改进需要在两处同步修改。

### 改进建议

1. **增加单元测试**：至少覆盖以下场景：
   - `"true"` → `true`
   - `"false"` → `false`
   - `true` → `true`（透传）
   - `false` → `false`（透传）
   - `"True"` → 校验失败（或根据策略决定是否支持）
   - `1` / `0` → 校验失败（保持布尔严格性）
   - `.optional()` 与 `.default()` 的 chaining 行为

2. **考虑支持 `"1"` / `"0"` 映射**：某些模型在少数情况下会用 `"1"` / `"0"` 表示布尔值。若 telemetry 显示这是常见问题，可扩展预处理逻辑：
   ```ts
   v === 'true' || v === '1' ? true
   : v === 'false' || v === '0' ? false
   : v
   ```
   但需权衡：过度宽容会降低类型系统的防护能力。

3. **统一文档化使用模式**：在 `SKILL.md` 或 `AGENTS.md` 中增加“如何定义 tool boolean/number 参数”的规范，强制要求使用 `semanticBoolean` / `semanticNumber` 而非原生 `z.boolean()` / `z.number()`，防止新工具开发者遗漏。

4. **监控模型输出偏差**：通过工具调用失败日志分析，统计因 `"true"`/"false"` 字符串导致的 Zod 校验失败率，评估当前预处理的实际效果。
