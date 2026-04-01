# 研究文档：src/utils/userPromptKeywords.ts

## 场景与职责

本模块提供基于正则表达式的**用户输入关键词检测**，用于识别两类特定模式：

1. **负面情绪关键词**：检测用户输入中是否包含脏话、沮丧表达或强烈负面情感；
2. **继续指令关键词**：检测用户是否输入了类似 "continue"、"keep going"、"go on" 的延续性指令。

这些检测结果可能被 analytics 系统记录，用于评估用户体验、模型表现，或在特定交互路径中触发不同的处理逻辑（如自动确认、情绪反馈收集等）。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `matchesNegativeKeyword(input)` | 检测输入是否匹配负面情绪/脏话正则模式。 |
| `matchesKeepGoingKeyword(input)` | 检测输入是否匹配继续/延续指令模式。 |

## 具体技术实现

### 1. 负面情绪检测 `matchesNegativeKeyword`

- 先将输入转为小写：`const lowerInput = input.toLowerCase()`。
- 使用一个较复杂的正则表达式进行全局词边界匹配（`\b`），覆盖的词汇/短语包括：
  - 缩写脏话：`wtf`, `wth`, `ffs`, `omfg`
  - 常见粗口及变体：`shit`, `shitty`, `dumbass`, `horrible`, `awful`
  - 情绪短语：`piss off`, `piece of shit/crap/junk`, `what the fuck/hell`
  - 组合表达：`fucking broken/useless/terrible/awful/horrible`, `fuck you`, `screw this/you`, `so frustrating`, `this sucks`, `damn it`
- 返回正则的 `.test(lowerInput)` 结果。

### 2. 继续指令检测 `matchesKeepGoingKeyword`

- 同样先转小写并去除首尾空白：`const lowerInput = input.toLowerCase().trim()`。
- **精确匹配**：若输入完全等于 `"continue"`（整句），返回 `true`。
- **子串匹配**：使用正则 `\b(keep going|go on)\b` 检测输入中是否包含这两个短语（不要求整句匹配）。
- 注意：`"please continue"` 不会匹配，因为 `"continue"` 要求整句唯一；但 `"please keep going"` 会匹配，因为 `"keep going"` 使用子串检测。

## 关键代码路径与文件引用

- **主实现**：`src/utils/userPromptKeywords.ts`（27 行）
- **调用方（文本输入处理）**：`src/utils/processUserInput/processTextPrompt.ts`

## 依赖与外部交互

- 无内部模块依赖。
- 无外部 npm 依赖。

## 风险、边界与改进建议

### 风险

1. **英语中心主义**：正则仅覆盖英语脏话和短语。对于中文、日语、西班牙语等非英语用户的负面情绪输入，模块完全无法检测，可能导致 analytics 中的负面情绪样本严重偏向英语用户，产生统计偏差。
2. **误报（False Positives）**：`\b` 词边界在某些语言或特殊字符组合中行为不一致。例如，包含 `"this sucks"` 的短语会匹配，但用户可能在讨论真空吸尘器（"this vacuum sucks"）——虽然这在日常对话中也可能被理解为双关，但在技术讨论中可能是完全中性的。
3. **`"continue"` 的严格整句匹配过于保守**：用户输入 `"just continue"`、`"continue please"` 或 `"continue with the task"` 都不会被识别为继续指令，可能错过大量自然的延续表达。

### 边界

- **无情感强度分级**：模块只返回布尔值，不区分轻微不满（如 "damn it"）和强烈侮辱（如 "fuck you"）。调用方无法根据情绪强度采取不同策略。
- **无上下文感知**：检测是纯字符串匹配，不理解对话历史。例如，用户引用一段包含脏话的代码或文档时，会被错误标记为负面情绪。
- **正则硬编码**：模式字符串直接写在源码中，没有外部配置或国际化文件，更新和扩展都需要修改代码并重新发版。

### 改进建议

1. **增加多语言支持**：
   - 引入一个基于 Unicode 属性的轻量级脏话检测库（如 `bad-words` 的扩展版），或维护按语言分组的正则映射表。
   - 至少覆盖中文、西班牙语、德语等用户量较大的语言。
2. **放宽 `"continue"` 匹配规则**：将 `"continue"` 的检测从精确整句改为子串匹配（但仍保留词边界 `\bcontinue\b`），以捕获 `"please continue"`、`"continue working"` 等常见变体。
3. **引入情感强度/分类**：将 `matchesNegativeKeyword` 拆分为更细粒度的函数，如 `getSentimentCategory(input)`，返回 `'frustration' | 'anger' | 'mild_discomfort' | 'neutral'`，让调用方可以差异化响应（如对高强度愤怒提示主动提供帮助或降级处理）。
4. **上下文白名单**：在调用方（`processTextPrompt.ts`）增加一层过滤，例如如果当前消息是在引用代码块（被 triple backtick 包裹），则跳过负面情绪检测，减少误报。
5. **外置正则配置**：将关键词模式迁移到项目配置或 GrowthBook flag 中，支持运营团队在不发版的情况下动态调整敏感词列表和继续指令模式。
