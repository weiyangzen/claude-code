# src/tools/WebSearchTool/prompt.ts 研究文档

> 文件路径：`/home/sansha/Github/claude-code-instructkr/src/tools/WebSearchTool/prompt.ts`
> 文件大小：1545 bytes
> 研究日期：2026-04-01

---

## 1. 场景与职责

`prompt.ts` 是 **WebSearchTool** 的提示文本与常量定义文件。职责极其单一：
1. 导出工具名称常量 `WEB_SEARCH_TOOL_NAME`；
2. 提供 `getWebSearchPrompt()` 函数，返回注入到系统提示中的 Web Search 能力说明。

该文件被 `WebSearchTool.ts`（用于 `buildTool` 的 `prompt` 方法）和 `src/constants/tools.ts`（用于权限/过滤集合）等多个模块引用，是 WebSearchTool 的**命名与提示语源头**。

---

## 2. 功能点目的

### 2.1 `WEB_SEARCH_TOOL_NAME`
- **值**：`'WebSearch'`
- **用途**：
  - 作为 `buildTool({ name: WEB_SEARCH_TOOL_NAME, ... })` 的工具唯一标识；
  - 在权限系统、compact 逻辑、代理工具白名单等场景中作为字符串常量引用，避免魔法字符串散落各处。

### 2.2 `getWebSearchPrompt()`
- **目的**：生成一段系统提示文本，告知模型：
  - 该工具允许 Claude 搜索网页并使用结果辅助回答；
  - 适用于获取时事、最新数据等超出知识截止日期的信息；
  - 搜索结果以搜索块形式返回，链接需以 markdown 超链接呈现；
  - **强制要求（CRITICAL REQUIREMENT）**：回答完毕后必须在末尾添加 `Sources:` 章节，列出所有相关 URL；
  - 支持域名过滤（允许/排除特定网站）；
  - 仅在美国可用；
  - **时间敏感提示**：使用 `getLocalMonthYear()` 获取当前年月，提醒模型在搜索近期信息时必须使用当前年份，而非去年。

---

## 3. 具体技术实现

### 3.1 源码实现

```typescript
import { getLocalMonthYear } from 'src/constants/common.js'

export const WEB_SEARCH_TOOL_NAME = 'WebSearch'

export function getWebSearchPrompt(): string {
  const currentMonthYear = getLocalMonthYear()
  return `
- Allows Claude to search the web and use the results to inform responses
- Provides up-to-date information for current events and recent data
- Returns search result information formatted as search result blocks, including links as markdown hyperlinks
- Use this tool for accessing information beyond Claude's knowledge cutoff
- Searches are performed automatically within a single API call

CRITICAL REQUIREMENT - You MUST follow this:
  - After answering the user's question, you MUST include a "Sources:" section at the end of your response
  - In the Sources section, list all relevant URLs from the search results as markdown hyperlinks: [Title](URL)
  - This is MANDATORY - never skip including sources in your response
  - Example format:

    [Your answer here]

    Sources:
    - [Source Title 1](https://example.com/1)
    - [Source Title 2](https://example.com/2)

Usage notes:
  - Domain filtering is supported to include or block specific websites
  - Web search is only available in the US

IMPORTANT - Use the correct year in search queries:
  - The current month is ${currentMonthYear}. You MUST use this year when searching for recent information, documentation, or current events.
  - Example: If the user asks for "latest React docs", search for "React documentation" with the current year, NOT last year
`
}
```

### 3.2 动态时间注入

- `getLocalMonthYear()` 来自 `src/constants/common.js`。
- 该函数返回类似 `"April 2026"` 的本地化字符串（基于运行时系统时区）。
- 通过模板字符串在每次调用时动态插入，确保模型始终获得最新的月份/年份提示，减少因知识截止日期导致的"去年"搜索问题。

### 3.3 提示语设计特点

| 特点 | 说明 |
|------|------|
| **指令层级清晰** | 先用 bullet list 说明工具能力，再用 `CRITICAL REQUIREMENT` 大写强调必须遵守的格式规范 |
| **示例驱动** | 提供完整的 `Sources:` 示例，降低模型格式错误概率 |
| **地域限制声明** | 明确告知 `"Web search is only available in the US"`，避免模型在非支持区域过度依赖 |
| **时间校准** | 通过运行时注入 `currentMonthYear`，解决大模型对"当前年份"的幻觉问题 |

---

## 4. 关键代码路径与文件引用

### 4.1 本文件导出

| 导出 | 类型 | 说明 |
|------|------|------|
| `WEB_SEARCH_TOOL_NAME` | `string` | 工具常量名 `'WebSearch'` |
| `getWebSearchPrompt` | `() => string` | 返回系统提示文本 |

### 4.2 直接依赖

| 文件 | 说明 |
|------|------|
| `src/constants/common.js` | `getLocalMonthYear()` 函数 |

### 4.3 调用方

| 文件 | 引用方式 | 用途 |
|------|----------|------|
| `src/tools/WebSearchTool/WebSearchTool.ts` | `import { getWebSearchPrompt, WEB_SEARCH_TOOL_NAME } from './prompt.js'` | `buildTool` 的 `name` 和 `prompt()` 方法 |
| `src/constants/tools.ts` | `import { WEB_SEARCH_TOOL_NAME } from '../tools/WebSearchTool/prompt.js'` | 定义 `ASYNC_AGENT_ALLOWED_TOOLS` 等集合 |
| `src/services/compact/microCompact.ts` | `import { WEB_SEARCH_TOOL_NAME } from '../../tools/WebSearchTool/prompt.js'` | `COMPACTABLE_TOOLS` 判断 |
| `src/services/compact/apiMicrocompact.ts` | `import { WEB_SEARCH_TOOL_NAME } from 'src/tools/WebSearchTool/prompt.js'` | `TOOLS_CLEARABLE_RESULTS` 判断 |
| `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts` | `import { WEB_SEARCH_TOOL_NAME } from 'src/tools/WebSearchTool/prompt.js'` | 在 Guide Agent 的系统提示中引用 |

---

## 5. 依赖与外部交互

- **零网络依赖**：本文件不发起任何 HTTP 请求，也不依赖文件系统。
- **纯函数**：`getWebSearchPrompt` 是无副作用纯函数，仅依赖 `getLocalMonthYear()` 的返回值。
- **国际化/本地化**：`getLocalMonthYear()` 基于系统时区生成本地化月份名称，若用户时区与预期不同，月份名称会相应变化（如中文环境可能显示"四月 2026"，具体取决于 `common.js` 的实现）。

---

## 6. 风险、边界与改进建议

### 6.1 当前风险与边界

1. **提示文本无版本控制/AB 测试入口**：`getWebSearchPrompt` 返回的字符串是硬编码的模板。若产品需要快速调整 Sources 格式要求或地域声明，必须修改源码并重新发版，无法通过 GrowthBook 等配置系统热更新。
2. **`getLocalMonthYear` 的时区敏感性**：该函数依赖运行环境的系统时区。如果服务器/容器时区配置错误（如 UTC 导致日期偏移），可能向模型注入错误的月份/年份，进而影响搜索质量。
3. **地域限制声明的准确性**：提示中声明 `"Web search is only available in the US"`，若 Anthropic 服务端未来扩展支持区域，此提示会成为过时信息，需要同步更新。
4. **无单元测试**：文件虽小，但 `getWebSearchPrompt` 的输出格式对模型行为有直接影响。当前无自动化测试验证输出字符串包含 `CRITICAL REQUIREMENT`、`Sources:` 示例和当前年月占位符。

### 6.2 改进建议

- **配置化提示模板**：可考虑将提示文本主体抽取为常量字符串，并允许通过环境变量或 GrowthBook 开关覆盖关键段落（如地域可用性说明、Sources 格式示例），实现无需发版的快速迭代。
- **增加时区校验/兜底**：在 `getLocalMonthYear()` 或调用方增加断言/日志，确保返回的月份年份与预期时间窗口（如系统时间 ±1 月）一致，防止容器时区漂移导致错误提示。
- **补充单元测试**：建议增加轻量级测试，断言 `getWebSearchPrompt()` 的输出：
  - 包含 `"Sources:"` 关键字；
  - 包含 `"CRITICAL REQUIREMENT"`；
  - 包含 `getLocalMonthYear()` 返回的字符串；
  - 包含 `"[Title](URL)"` 示例格式。
- **建立提示变更审计机制**：由于该提示直接影响模型输出格式，建议在代码审查或发布流程中标记 `prompt.ts` 的变更为"模型行为影响"，防止无意的格式改动破坏 Sources 提取逻辑。
