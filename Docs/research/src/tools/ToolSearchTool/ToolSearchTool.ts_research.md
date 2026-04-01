# ToolSearchTool.ts 研究文档

## 场景与职责

ToolSearchTool 是 Claude Code CLI 中的核心工具发现机制，用于支持**延迟加载（Deferred Loading）**架构。当系统启用工具搜索功能时，MCP（Model Context Protocol）工具和标记为 `shouldDefer` 的工具不会直接加载到初始系统提示中，而是仅显示工具名称。模型需要通过 ToolSearchTool 查询来获取这些工具的完整 JSONSchema 定义后才能调用它们。

主要应用场景：
1. **MCP 工具管理**：当用户配置了多个 MCP 服务器时，相关工具数量可能很多，延迟加载可以减少上下文窗口占用
2. **动态工具发现**：模型可以通过关键词搜索找到所需的功能工具
3. **工具选择优化**：支持直接选择特定工具（`select:ToolName`）或关键词搜索

## 功能点目的

### 1. 工具搜索与匹配
- **关键词搜索**：支持多关键词匹配，包括工具名称、描述和 searchHint
- **精确匹配**：支持 `select:ToolName` 语法直接获取特定工具
- **多选支持**：支持 `select:A,B,C` 批量选择多个工具
- **MCP 工具特殊处理**：处理 `mcp__server__action` 格式的 MCP 工具名称

### 2. 智能评分系统
搜索评分考虑以下因素（按优先级排序）：
- 名称部分精确匹配（MCP 工具权重 12，普通工具权重 10）
- 名称部分包含匹配（MCP 工具权重 6，普通工具权重 5）
- searchHint 匹配（权重 4）
- 描述中的词边界匹配（权重 2）
- 完整名称包含（权重 3，仅作为后备）

### 3. 缓存管理
- **描述缓存**：使用 lodash memoize 缓存工具描述，避免重复生成
- **缓存失效**：当延迟工具集合变化时自动清除缓存
- **手动清除**：提供 `clearToolSearchDescriptionCache()` 函数供外部调用

### 4. 结果格式化
- 成功返回：返回 `tool_reference` 块数组，API 会将其展开为完整工具定义
- 无匹配：返回友好提示，包括正在连接的 MCP 服务器信息
- 结果限制：支持通过 `max_results` 参数限制返回数量（默认 5）

## 具体技术实现

### 关键数据结构

```typescript
// 输入模式
{
  query: string;        // 搜索查询，支持 "select:" 前缀或关键词
  max_results?: number; // 最大结果数，默认 5
}

// 输出模式
{
  matches: string[];           // 匹配的工具名称列表
  query: string;               // 原始查询
  total_deferred_tools: number; // 延迟工具总数
  pending_mcp_servers?: string[]; // 正在连接的 MCP 服务器
}
```

### 核心流程

#### 1. 工具构建（buildTool）
```typescript
export const ToolSearchTool = buildTool({
  isEnabled() { return isToolSearchEnabledOptimistic(); },
  isConcurrencySafe() { return true; },
  isReadOnly() { return true; },
  name: TOOL_SEARCH_TOOL_NAME,  // 'ToolSearch'
  maxResultSizeChars: 100_000,
  // ... 其他方法
});
```

#### 2. 调用处理流程
1. **解析输入**：提取 `query` 和 `max_results`
2. **获取延迟工具**：从所有工具中筛选 `isDeferredTool` 返回 true 的工具
3. **缓存失效检查**：比较当前延迟工具集合与缓存的键
4. **查询类型判断**：
   - 以 `select:` 开头 → 直接选择模式
   - 否则 → 关键词搜索模式
5. **结果构建**：返回匹配的工具名称列表

#### 3. 直接选择模式（select:）
```typescript
const selectMatch = query.match(/^select:(.+)$/i);
if (selectMatch) {
  const requested = selectMatch[1]!
    .split(',')
    .map(s => s.trim())
    .filter(Boolean);
  // 在延迟工具和全部工具中查找
  const tool = findToolByName(deferredTools, toolName) ?? 
               findToolByName(tools, toolName);
  // ...
}
```

#### 4. 关键词搜索算法
```typescript
async function searchToolsWithKeywords(
  query: string,
  deferredTools: Tools,
  tools: Tools,
  maxResults: number,
): Promise<string[]> {
  // 1. 快速路径：精确名称匹配
  const exactMatch = deferredTools.find(t => t.name.toLowerCase() === queryLower) ??
                     tools.find(t => t.name.toLowerCase() === queryLower);
  if (exactMatch) return [exactMatch.name];

  // 2. MCP 前缀匹配
  if (queryLower.startsWith('mcp__')) { ... }

  // 3. 解析查询词（支持 +required 语法）
  const requiredTerms: string[] = [];
  const optionalTerms: string[] = [];
  // ...

  // 4. 预过滤：必须包含所有 required 词
  let candidateTools = deferredTools;
  if (requiredTerms.length > 0) { ... }

  // 5. 评分排序
  const scored = await Promise.all(
    candidateTools.map(async tool => {
      // 计算分数...
      return { name: tool.name, score };
    }),
  );

  return scored
    .filter(item => item.score > 0)
    .sort((a, b) => b.score - a.score)
    .slice(0, maxResults)
    .map(item => item.name);
}
```

#### 5. 工具名称解析
```typescript
function parseToolName(name: string): { parts: string[]; full: string; isMcp: boolean } {
  if (name.startsWith('mcp__')) {
    // MCP 工具：mcp__server__action → ['server', 'action']
    const withoutPrefix = name.replace(/^mcp__/, '').toLowerCase();
    const parts = withoutPrefix.split('__').flatMap(p => p.split('_'));
    return { parts, full: withoutPrefix.replace(/__/g, ' ').replace(/_/g, ' '), isMcp: true };
  }
  // 普通工具：CamelCase 和 下划线分割
  const parts = name
    .replace(/([a-z])([A-Z])/g, '$1 $2')
    .replace(/_/g, ' ')
    .toLowerCase()
    .split(/\s+/)
    .filter(Boolean);
  return { parts, full: parts.join(' '), isMcp: false };
}
```

### 关键代码路径

| 功能 | 路径 | 说明 |
|------|------|------|
| 工具定义 | `src/tools/ToolSearchTool/ToolSearchTool.ts` | 主实现文件 |
| 工具名称常量 | `src/tools/ToolSearchTool/constants.ts` | `TOOL_SEARCH_TOOL_NAME = 'ToolSearch'` |
| 提示词生成 | `src/tools/ToolSearchTool/prompt.ts` | `getPrompt()` 函数 |
| 延迟工具判断 | `src/tools/ToolSearchTool/prompt.ts` | `isDeferredTool()` 函数 |
| 工具搜索开关 | `src/utils/toolSearch.ts` | `isToolSearchEnabledOptimistic()` |
| 工具基础类型 | `src/Tool.ts` | `Tool`, `ToolDef`, `buildTool` |
| 字符串工具 | `src/utils/stringUtils.ts` | `escapeRegExp()` |
| 调试日志 | `src/utils/debug.ts` | `logForDebugging()` |
| 分析事件 | `src/services/analytics/index.ts` | `logEvent()` |

## 依赖与外部交互

### 导入依赖

```typescript
// 外部库
import type { ToolResultBlockParam } from '@anthropic-ai/sdk/resources/index.mjs'
import memoize from 'lodash-es/memoize.js'
import { z } from 'zod/v4'

// 内部模块
import { logEvent, type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS } from '../../services/analytics/index.js'
import { buildTool, findToolByName, type Tool, type ToolDef, type Tools } from '../../Tool.js'
import { logForDebugging } from '../../utils/debug.js'
import { lazySchema } from '../../utils/lazySchema.js'
import { escapeRegExp } from '../../utils/stringUtils.js'
import { isToolSearchEnabledOptimistic } from '../../utils/toolSearch.js'
import { getPrompt, isDeferredTool, TOOL_SEARCH_TOOL_NAME } from './prompt.js'
```

### 外部交互

1. **Analytics 服务**
   - 记录 `tengu_tool_search_outcome` 事件
   - 包含查询类型、匹配数量、延迟工具总数等元数据

2. **AppState**
   - 通过 `getAppState()` 获取 MCP 客户端状态
   - 检查正在连接的 MCP 服务器以提供用户反馈

3. **工具系统**
   - 调用 `tool.prompt()` 获取工具描述用于搜索评分
   - 使用 `findToolByName()` 查找工具
   - 使用 `isDeferredTool()` 判断工具是否延迟加载

4. **API 交互**
   - 返回 `tool_reference` 内容块
   - API 负责将引用展开为完整工具定义

## 风险、边界与改进建议

### 潜在风险

1. **缓存一致性问题**
   - 风险：工具描述缓存可能在新 MCP 服务器连接后未正确失效
   - 缓解：`maybeInvalidateCache()` 检查延迟工具名称集合的变化
   - 注意：缓存键仅基于工具名称，不包括描述内容变化

2. **性能问题**
   - 风险：大量延迟工具时，每个搜索都需要调用所有工具的 `prompt()` 方法
   - 缓解：使用 memoize 缓存描述，但首次加载仍可能较慢
   - 建议：考虑预生成搜索索引而非实时计算

3. **评分准确性**
   - 风险：简单关键词匹配可能导致不相关工具排名靠前
   - 当前：依赖词边界匹配和加权评分
   - 建议：考虑引入语义搜索或更复杂的 NLP 技术

4. **MCP 服务器连接状态**
   - 风险：用户搜索时 MCP 服务器可能仍在连接中
   - 缓解：返回 `pending_mcp_servers` 提示用户稍后重试
   - 边界：无法自动重试或通知用户连接完成

### 边界情况

1. **空查询处理**
   - 查询为空字符串时，所有工具都可能匹配（取决于评分逻辑）
   - 实际中模型通常不会发送空查询

2. **大小写敏感性**
   - 所有匹配都转换为小写进行
   - MCP 工具名称中的 `mcp__` 前缀必须小写

3. **特殊字符处理**
   - 使用 `escapeRegExp()` 处理搜索词中的正则特殊字符
   - 但查询词分割仅基于空白字符，不处理标点符号

4. **循环依赖风险**
   - `prompt.ts` 中的 `isDeferredTool()` 使用动态 require 避免循环导入
   - `AgentTool` 和 `BriefTool` 的引用通过懒加载处理

### 改进建议

1. **搜索优化**
   - 引入模糊匹配（如 Levenshtein 距离）处理拼写错误
   - 支持引号精确短语匹配
   - 添加工具类别/标签系统

2. **性能优化**
   - 预构建倒排索引而非实时扫描
   - 考虑使用 Web Worker 处理大量工具的搜索
   - 添加搜索超时机制

3. **用户体验**
   - 支持搜索结果分页
   - 添加搜索历史记录
   - 提供搜索建议/自动补全

4. **可观测性**
   - 添加更多分析事件（如搜索延迟、缓存命中率）
   - 记录搜索失败的原因分类
   - 监控 MCP 服务器连接时间

5. **代码质量**
   - `prompt.ts` 中存在重复的 `import { TOOL_SEARCH_TOOL_NAME } from './constants.js'`
   - 考虑将评分权重提取为可配置常量
   - 增加单元测试覆盖率，特别是边界情况
