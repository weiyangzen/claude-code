# 研究文档：src/utils/semanticNumber.ts

## 场景与职责

`semanticNumber.ts` 是 Claude Code 中用于**Zod number schema 的语义化预处理辅助工厂**，与 `semanticBoolean.ts` 成对出现。它解决的核心问题是：

> LLM 在生成 JSON tool inputs 时，偶尔会将数字写成字符串字面量，例如 `"head_limit": "30"` 而非 `"head_limit": 30`。原生 `z.number()` 会拒绝这种输入；而 `z.coerce.number()` 虽然能转换，但它对 `""`、`null` 等值使用 `Number()` 进行转换，会掩盖真正的数据错误（例如 `""` → `0`、`null` → `0`）。

该模块提供一个**严格且有限**的强制策略：仅当字符串匹配十进制数字字面量正则 `^-?\d+(\.\d+)?$` 且转换后为有限数时，才进行 `string → number` 映射；其余值原样透传给内层 schema，由其决定合法与否。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `semanticNumber(inner?)` | 生成 `ZodPipe`，预处理合法数字字符串为 `number`，再交给内层 schema（默认 `z.number()`）校验。 |

---

## 具体技术实现

### 1. 实现代码

```ts
import { z } from 'zod/v4'

export function semanticNumber<T extends z.ZodType>(
  inner: T = z.number() as unknown as T,
) {
  return z.preprocess((v: unknown) => {
    if (typeof v === 'string' && /^-?\d+(\.\d+)?$/.test(v)) {
      const n = Number(v)
      if (Number.isFinite(n)) return n
    }
    return v
  }, inner)
}
```

### 2. 关键设计决策

- **正则严格匹配**：`/^-?\d+(\.\d+)?$/`
  - 允许： `"30"`, `"-5"`, `"3.14"`
  - 拒绝： `""`, `" 30"`, `"30px"`, `"1e5"`, `"0x1A"`, `"NaN"`, `"Infinity"`
- **`Number.isFinite` 二次保护**：防止极端输入（如超长的全 9 字符串在某些 JS 引擎中的行为差异）产生非有限数。
- **透传非匹配值**：如果值不是字符串或不符合正则，直接返回原值，让 `inner` schema 处理。这意味着：
  - 真正的数字 `30` → 透传 → `z.number()` 通过
  - 字符串 `"30"` → 预处理为 `30` → `z.number()` 通过
  - 字符串 `""` → 透传 → `z.number()` 拒绝（正确行为，避免 `z.coerce.number()` 的 `""`→`0` bug）

### 3. 与 API Schema 的兼容性

与 `semanticBoolean` 相同，`z.preprocess` 在生成 API JSON Schema 时通常仍会被底层 `inner` 的类型覆盖。模型看到的仍然是 `{"type":"number"}`，字符串容忍对模型不可见。

### 4. 使用模式

```ts
semanticNumber()                              // → number
semanticNumber(z.number().optional())         // → number | undefined
semanticNumber(z.number().default(0))         // → number
```

`.optional()` / `.default()` 必须放在 `inner` 上，不能 chain 到 `ZodPipe` 外，否则 Zod v4 会推断 `z.output<>` 为 `unknown`。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/semanticNumber.ts:26-36` | `semanticNumber` 工厂函数。 |
| `src/tools/FileReadTool/FileReadTool.ts:230-235` | 使用方：`offset` 与 `limit` 参数。 |
| `src/tools/BashTool/BashTool.tsx` | 使用方：Bash 工具参数。 |
| `src/tools/GrepTool/GrepTool.ts` | 使用方：Grep 工具的 `head_limit` 等参数。 |
| `src/tools/PowerShellTool/PowerShellTool.tsx` | 使用方：PowerShell 工具参数。 |

---

## 依赖与外部交互

- **第三方库**：`zod/v4`。
- **无 Node.js 内置模块依赖**。
- **调用方**：FileReadTool、BashTool、GrepTool、PowerShellTool 等工具的 input schema 定义。

---

## 风险、边界与改进建议

### 风险与边界

1. **科学计数法被拒绝**：正则 `^-?\d+(\.\d+)?$` 不允许 `1e5` 或 `-3.14e-10`。若模型偶尔输出科学计数法字符串，校验会失败。不过考虑到工具参数的语义（如 `offset`、`limit`、`head_limit`），模型通常输出整数或简单小数，此限制是合理的。

2. **前导零与八进制歧义**：`Number("010")` 在严格模式下是 `10`（ES5+ 已废弃八进制隐式转换），所以当前行为是安全的。但正则允许 `"010"`，若某些业务场景需要严格拒绝前导零，当前实现做不到。

3. **小数点后无数字被拒绝**：`"3."` 不匹配 `\.\d+`，会被拒绝。这与 JSON 数字规范一致（JSON 不允许 `3.`），是正确行为。

4. **大整数精度丢失**：当字符串表示的整数超过 `Number.MAX_SAFE_INTEGER`（`9007199254740991`）时，`Number(v)` 会丢失精度。若工具参数需要处理大整数（如某些 ID），当前实现会静默产生错误数值。`Number.isFinite` 无法检测精度丢失。

5. **与 Zod v4 的兼容性假设**：同 `semanticBoolean`，`z.preprocess` 的 JSON Schema 生成行为依赖 Zod 内部实现，未来版本变更可能影响 API 侧的类型提示。

### 改进建议

1. **增加单元测试**：覆盖以下场景：
   - `"30"` → `30`
   - `"-5"` → `-5`
   - `"3.14"` → `3.14`
   - `""` → 校验失败
   - `"30px"` → 校验失败
   - `"1e5"` → 校验失败（或根据需求调整）
   - `30` → `30`（透传）
   - `null` → 校验失败
   - `.optional()` 与 `.default()` 的类型推断

2. **大整数安全检测**：若某些参数可能涉及大整数，可在预处理中加入 `Number.isSafeInteger(n)` 检查：
   ```ts
   if (Number.isFinite(n) && Number.isSafeInteger(n)) return n
   ```
   但这需要按参数决定是否启用，不适合作为全局默认行为。

3. **考虑支持 `BigInt` 字符串**：未来若需要处理超大整数，可新增 `semanticBigInt` 工厂，使用 `BigInt(v)` 转换并配合 `z.bigint()` inner schema。

4. **统一工具参数规范**：在团队文档中明确“所有数值型 tool 参数必须使用 `semanticNumber`，禁止使用 `z.coerce.number()`”，并配合 lint 规则或 code review checklist 执行。

5. **收集模型输出偏差 telemetry**：分析工具调用失败日志，统计因数字被引号包裹导致的校验失败率，验证 `semanticNumber` 的实际覆盖效果。
