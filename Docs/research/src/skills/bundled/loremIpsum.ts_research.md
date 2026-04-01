# loremIpsum.ts 研究文档

## 场景与职责

`loremIpsum.ts` 实现了 `/lorem-ipsum` 内置技能，用于生成长上下文测试所需的填充文本。该技能仅对 Anthropic 内部员工（`USER_TYPE === 'ant'`）可用，通过组合经过 API token 计数验证的单 token 英文单词，生成接近指定 token 数量的伪随机文本。

## 功能点目的

1. **长上下文测试支持**：为内部测试提供可控制长度的大段文本，用于验证模型在处理长上下文时的行为。
2. **近似 token 数量**：通过使用已验证的单 token 单词，生成的文本 token 数与请求值大致相等。
3. **安全上限**：默认 10,000 tokens，最大允许 500,000 tokens，防止意外生成超大内容。

## 具体技术实现

### 关键流程

- `registerLoremIpsumSkill()`：
  1. 若 `process.env.USER_TYPE !== 'ant'` 直接返回（技能不注册）
  2. 否则 `registerBundledSkill({ name: 'lorem-ipsum', ... })`
- `getPromptForCommand(args)` 入口：
  1. `parseInt(args)`，判空则默认 `10000`
  2. 若 `NaN` 或 `<= 0` 返回错误提示
  3. `cappedTokens = Math.min(targetTokens, 500_000)`
  4. 调用 `generateLoremIpsum(cappedTokens)` 生成文本并返回

### 数据结构

```ts
const ONE_TOKEN_WORDS: string[] = [
  // 约 200 个经 API token 计数验证的英文单词
  'the', 'a', 'an', 'I', 'you', 'is', 'are', 'time', 'year', ...
]
```

### 生成算法

```ts
function generateLoremIpsum(targetTokens: number): string {
  let tokens = 0
  let result = ''
  while (tokens < targetTokens) {
    const sentenceLength = 10 + Math.floor(Math.random() * 11) // 10-20 词
    for (let i = 0; i < sentenceLength && tokens < targetTokens; i++) {
      const word = ONE_TOKEN_WORDS[Math.floor(Math.random() * ONE_TOKEN_WORDS.length)]
      result += word
      tokens++
      // 句末加 '. '，否则加 ' '
    }
    // 每句结束后 20% 概率插入段落换行 '\n\n'
  }
  return result.trim()
}
```

### 注册参数

| 字段 | 值 |
|------|-----|
| `name` | `'lorem-ipsum'` |
| `argumentHint` | `'[token_count]'` |
| `userInvocable` | `true` |

## 关键代码路径与文件引用

- 源文件：`src/skills/bundled/loremIpsum.ts`
- 注册入口：`src/skills/bundled/index.ts`
- 核心注册器：`src/skills/bundledSkills.ts`
- 无其他外部依赖

## 依赖与外部交互

- **纯本地算法**：不依赖任何外部文件或网络调用。
- **Ant-only 门控**：通过 `process.env.USER_TYPE !== 'ant'` 在模块加载时直接短路，技能对外部用户完全不可见。

## 风险、边界与改进建议

1. **边界：token 计数仅为近似值**：`ONE_TOKEN_WORDS` 中的单词在特定模型/tokenizer 下被验证为单 token，但不同模型（尤其是多语言模型或新版 tokenizer）可能将同一单词切分为不同数量的 token。因此 "10,000 词 ≈ 10,000 tokens" 的假设在模型切换时可能失效。
2. **边界：随机性不可复现**：使用 `Math.random()` 无种子，相同参数两次调用结果不同，不利于回归测试中的可复现性。
3. **风险：大文本内存占用**：生成 500,000 tokens 的字符串在 Node.js/Bun 中可能占用数十 MB 内存，在资源受限环境中有 OOM 风险。
4. **改进建议**：
   - 引入可配置的随机种子（如通过参数 `--seed`），使测试结果可复现。
   - 在生成超大文本时采用流式返回或分块生成，避免一次性构造大字符串。
   - 增加对当前模型 tokenizer 的动态校验：在生成前用实际 tokenizer 对 `ONE_TOKEN_WORDS` 做快速采样验证，给出偏差警告。
