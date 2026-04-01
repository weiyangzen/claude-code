# 研究文档：src/utils/sanitization.ts

## 场景与职责

`sanitization.ts` 是 Claude Code 的**Unicode 安全清洗层**，专门防御基于不可见 Unicode 字符的隐藏指令注入攻击（Hidden Character Attack / ASCII Smuggling）。这类攻击利用 Tag 字符、格式控制符、私有使用区等 invisible Unicode 在用户不可见的文本中嵌入恶意 prompt，已被证实可针对 MCP（Model Context Protocol）实现进行利用（参考 HackerOne #3086545）。

模块的职责是：
- 对所有进入系统敏感路径的字符串进行 Unicode 规范化与危险字符剥离。
- 支持递归清洗复杂嵌套数据结构（对象、数组）。
- 保证清洗过程幂等且有限步收敛，防止无限循环。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `partiallySanitizeUnicode(prompt)` | 对单个字符串执行 NFKC 规范化 + 危险 Unicode 类别/范围移除，最多迭代 10 次直到收敛。 |
| `recursivelySanitizeUnicode(value)` | 递归遍历字符串、数组、对象，对所有键和值调用 `partiallySanitizeUnicode`；保留数字、布尔、null、undefined 不变。 |

---

## 具体技术实现

### 1. `partiallySanitizeUnicode`

迭代清洗逻辑（最多 `MAX_ITERATIONS = 10`）：

```ts
while (current !== previous && iterations < MAX_ITERATIONS) {
  previous = current

  // 1. NFKC 规范化：处理组合字符序列
  current = current.normalize('NFKC')

  // 2. 按 Unicode 属性类剥离：格式控制符(Cf)、私有使用(Co)、非字符(Cn)
  current = current.replace(/[\p{Cf}\p{Co}\p{Cn}]/gu, '')

  // 3. 显式字符范围兜底（兼容不支持 Unicode property 的正则环境）
  current = current
    .replace(/[\u200B-\u200F]/g, '')   // 零宽空格、LTR/RTL marks
    .replace(/[\u202A-\u202E]/g, '')   // 双向格式控制符
    .replace(/[\u2066-\u2069]/g, '')   // 双向隔离符
    .replace(/[\uFEFF]/g, '')          // BOM
    .replace(/[\uE000-\uF8FF]/g, '')   // BMP 私有使用区

  iterations++
}
```

**收敛检测**：若 10 次迭代后仍未收敛，直接抛 `Error` 并附带输入前 100 字符，作为“ loudly crash ”策略——开发者认为这只能是 bug 或恶意构造。

### 2. `recursivelySanitizeUnicode`

使用 TypeScript 函数重载提供类型安全：

```ts
export function recursivelySanitizeUnicode(value: string): string
export function recursivelySanitizeUnicode<T>(value: T[]): T[]
export function recursivelySanitizeUnicode<T extends object>(value: T): T
export function recursivelySanitizeUnicode<T>(value: T): T
export function recursivelySanitizeUnicode(value: unknown): unknown
```

递归规则：
- `string` → `partiallySanitizeUnicode(value)`
- `Array` → `value.map(recursivelySanitizeUnicode)`
- `object`（且非 null）→ 遍历 `Object.entries`，对 key 和 value 均递归清洗后组装新对象
- 其他（`number`, `boolean`, `null`, `undefined`）→ 原样返回

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/sanitization.ts:25-65` | `partiallySanitizeUnicode` 核心实现。 |
| `src/utils/sanitization.ts:67-91` | `recursivelySanitizeUnicode` 递归清洗实现。 |
| `src/services/mcp/client.ts` | MCP 客户端调用清洗层处理 tool inputs/outputs。 |
| `src/commands/tag/tag.tsx` | Tag 命令处理用户输入时调用。 |
| `src/utils/deepLink/parseDeepLink.ts` | 深度链接解析时对参数做清洗。 |
| `src/utils/sessionStoragePortable.ts` | 会话存储数据反序列化时清洗。 |
| `src/commands/security-review.ts` | 安全审查相关输入清洗。 |
| `src/tools/FileEditTool/utils.ts` | 文件编辑工具内部使用。 |

---

## 依赖与外部交互

- **无第三方依赖**：纯 ECMAScript 内置 API（`String.prototype.normalize`、`String.prototype.replace`、`Object.entries`）。
- **无 Node.js 内置模块依赖**：可在浏览器或任何 JS 运行时运行。
- **调用方**：
  - `src/services/mcp/client.ts`（MCP 输入/输出安全 gate）
  - `src/commands/tag/tag.tsx`
  - `src/utils/deepLink/parseDeepLink.ts`
  - `src/utils/sessionStoragePortable.ts`
  - `src/commands/security-review.ts`
  - `src/tools/FileEditTool/utils.ts`

---

## 风险、边界与改进建议

### 风险与边界

1. **Unicode property 正则的兼容性**：`\p{Cf}\p{Co}\p{Cn}` 需要 ES2018+ 的 `u` flag 与 Unicode property escapes。当前项目运行环境为 Node.js 18+ / Bun，完全支持。若未来需要降级到更老环境，显式范围兜底已提供基础保护，但会漏掉大量危险字符。

2. **NFKC 的语义副作用**：NFKC 规范化会将某些视觉上相似但语义不同的字符统一（如全角数字 `１` → `1`，上标 `²` → `2`）。这在安全场景下通常是可接受的，甚至是有益的（防混淆攻击），但可能对需要保留原始 Unicode 精确性的场景（如处理特定语言文本）造成意外修改。

3. **递归深度风险**：`recursivelySanitizeUnicode` 对深层嵌套对象存在调用栈溢出风险。虽然正常业务数据不会达到 V8 的栈深度极限（通常 ~10k+ 层），但恶意构造的极深嵌套 JSON 可能导致 `RangeError: Maximum call stack size exceeded`。

4. **对象原型链丢失**：递归实现使用 `Object.entries` 并组装到新的 plain object `{}`，会丢失原对象的原型链、不可枚举属性、Symbol 键。对于 tool 输入这类通常来自 JSON 的数据不是问题，但对更复杂的对象结构可能不够完备。

5. **10 次迭代上限的合理性**：NFKC + 正则替换通常是幂等的（一次即可收敛），10 次是极度保守的安全网。若输入被刻意构造为每次迭代只去掉一个字符且 NFKC 不断生成新字符，则 10 次后 crash。这种输入在现实中几乎不存在，但 crash 行为意味着任何触及该路径的代码都会抛出未捕获异常。

### 改进建议

1. **增加循环/Map/Set 支持**：当前递归只处理 Array 和 plain Object。若业务数据中出现 `Map`、`Set` 或自定义类实例，会被当作 plain object 处理（`Object.entries` 只能提取可枚举字符串属性）。可考虑扩展支持：
   ```ts
   if (value instanceof Map) return new Map([...value].map(...))
   if (value instanceof Set) return new Set([...value].map(...))
   ```

2. **栈安全化**：对极深嵌套对象，可改用迭代式遍历（如维护一个待处理队列）替代递归，彻底消除栈溢出风险。

3. **收敛失败降级而非 Crash**：当前 10 次未收敛直接抛 Error。考虑到这是安全层，更稳健的做法可能是：记录严重警告日志后返回最后一次迭代结果，而不是让整条调用链崩溃。当然，若这是“防御性编程”故意设计的 fail-closed 行为，则保持现状亦可，但需在文档中明确。

4. **补充更多危险范围**：随着 Unicode 版本更新，新的不可见/控制字符可能未被覆盖。可定期审计并补充范围，例如：
   - Tag characters: `U+E0001` + `U+E0020–U+E007F`
   - Variation selectors: `U+FE00–U+FE0F`, `U+E0100–U+E01EF`
   当前 NFKC + `\p{Cf}\p{Co}\p{Cn}` 已覆盖大部分，但显式范围可作为双重保险。

5. **性能基准测试**：对于高频调用路径（如 MCP 的每条消息），建议增加 micro-benchmark，确保清洗不会成为吞吐量瓶颈。当前实现基于正则，对典型输入（<10KB）开销可忽略，但对大文本（如代码文件全文）应评估是否需要在更高层做截断后再清洗。
