# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 是 ToolSearchTool 模块的核心辅助文件，承担以下职责：

1. **提示词生成**：构建 ToolSearchTool 的系统提示词（system prompt）
2. **延迟工具判断**：定义 `isDeferredTool()` 函数，决定哪些工具需要延迟加载
3. **工具名称管理**：导入并重新导出 `TOOL_SEARCH_TOOL_NAME` 常量
4. **特性开关集成**：根据 GrowthBook 特性标志和运行时条件调整行为

## 功能点目的

### 1. 提示词生成（getPrompt）

为 ToolSearchTool 生成系统提示词，告诉模型：
- 工具的作用：获取延迟工具的完整 schema 定义
- 延迟工具的位置提示（system-reminder vs available-deferred-tools）
- 查询语法：`select:` 直接选择、关键词搜索、`+` 必需词前缀
- 结果格式：`<functions>` 块中的 JSON 格式

### 2. 延迟工具判断（isDeferredTool）

决定一个工具是否应该延迟加载，判断逻辑（按优先级）：

| 条件 | 结果 | 说明 |
|------|------|------|
| `alwaysLoad === true` | 不延迟 | 显式选择加入，如 MCP 工具设置 `_meta['anthropic/alwaysLoad']` |
| `isMcp === true` | 延迟 | MCP 工具默认延迟（工作流特定） |
| `name === 'ToolSearch'` | 不延迟 | ToolSearchTool 本身必须立即可用 |
| `FORK_SUBAGENT` 特性且 `name === 'Agent'` | 不延迟 | Fork-first 实验：Agent 工具必须首回合可用 |
| `KAIROS`/`KAIROS_BRIEF` 特性且 `name === BRIEF_TOOL_NAME` | 不延迟 | Brief 工具是主要通信渠道 |
| `KAIROS` 特性且 `name === SEND_USER_FILE_TOOL_NAME` | 不延迟 | SendUserFile 是文件传递通道 |
| `shouldDefer === true` | 延迟 | 显式标记为延迟的工具 |
| 默认 | 不延迟 | 普通工具 |

### 3. 工具行格式化（formatDeferredToolLine）

格式化延迟工具在 `<available-deferred-tools>` 消息中的显示：
- 当前实现仅返回工具名称
- 注释提到 searchHint 实验（exp_xenhnnmn0smrx4）显示无益处，已停止

## 具体技术实现

### 提示词结构

```typescript
const PROMPT_HEAD = `Fetches full schema definitions for deferred tools so they can be called.

`;

function getToolLocationHint(): string {
  const deltaEnabled = process.env.USER_TYPE === 'ant' ||
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_glacier_2xr', false);
  return deltaEnabled
    ? 'Deferred tools appear by name in <system-reminder> messages.'
    : 'Deferred tools appear by name in <available-deferred-tools> messages.';
}

const PROMPT_TAIL = ` Until fetched, only the name is known — there is no parameter schema...

Result format: each matched tool appears as one <function>...</function> line...

Query forms:
- "select:Read,Edit,Grep" — fetch these exact tools by name
- "notebook jupyter" — keyword search, up to max_results best matches
- "+slack send" — require "slack" in the name, rank by remaining terms`;

export function getPrompt(): string {
  return PROMPT_HEAD + getToolLocationHint() + PROMPT_TAIL;
}
```

### 延迟工具判断实现

```typescript
export function isDeferredTool(tool: Tool): boolean {
  // 1. 显式选择加入（最高优先级）
  if (tool.alwaysLoad === true) return false;

  // 2. MCP 工具默认延迟
  if (tool.isMcp === true) return true;

  // 3. ToolSearchTool 本身不能延迟
  if (tool.name === TOOL_SEARCH_TOOL_NAME) return false;

  // 4. Fork-first 实验：Agent 工具首回合可用
  if (feature('FORK_SUBAGENT') && tool.name === AGENT_TOOL_NAME) {
    type ForkMod = typeof import('../AgentTool/forkSubagent.js');
    const m = require('../AgentTool/forkSubagent.js') as ForkMod;
    if (m.isForkSubagentEnabled()) return false;
  }

  // 5. Brief 工具（KAIROS 特性）
  if ((feature('KAIROS') || feature('KAIROS_BRIEF')) &&
      BRIEF_TOOL_NAME && tool.name === BRIEF_TOOL_NAME) {
    return false;
  }

  // 6. SendUserFile 工具（KAIROS 特性）
  if (feature('KAIROS') &&
      SEND_USER_FILE_TOOL_NAME &&
      tool.name === SEND_USER_FILE_TOOL_NAME &&
      isReplBridgeActive()) {
    return false;
  }

  // 7. 显式标记为延迟
  return tool.shouldDefer === true;
}
```

### 特性标志懒加载

为避免循环导入和死代码，使用动态 require：

```typescript
const BRIEF_TOOL_NAME: string | null =
  feature('KAIROS') || feature('KAIROS_BRIEF')
    ? (require('../BriefTool/prompt.js') as typeof import('../BriefTool/prompt.js')).BRIEF_TOOL_NAME
    : null;

const SEND_USER_FILE_TOOL_NAME: string | null = feature('KAIROS')
  ? (require('../SendUserFileTool/prompt.js') as typeof import('../SendUserFileTool/prompt.js')).SEND_USER_FILE_TOOL_NAME
  : null;
```

### Bun 死代码消除

```typescript
import { feature } from 'bun:bundle';
```

使用 Bun 的 `feature()` 函数进行编译时特性检测，未启用的特性代码会被消除。

## 关键代码路径与文件引用

### 内部依赖

```
prompt.ts
├── constants.ts              # TOOL_SEARCH_TOOL_NAME
├── ../../bootstrap/state.js  # isReplBridgeActive
├── ../../services/analytics/growthbook.js  # getFeatureValue_CACHED_MAY_BE_STALE
├── ../../Tool.js             # Tool 类型
└── ../AgentTool/constants.js # AGENT_TOOL_NAME
```

### 动态依赖（条件加载）

```
../AgentTool/forkSubagent.js  # FORK_SUBAGENT 特性启用时
../BriefTool/prompt.js        # KAIROS/KAIROS_BRIEF 特性启用时
../SendUserFileTool/prompt.js # KAIROS 特性启用时
```

### 被依赖关系

| 文件 | 导入内容 |
|------|----------|
| `ToolSearchTool.ts` | `getPrompt`, `isDeferredTool`, `TOOL_SEARCH_TOOL_NAME` |
| `src/utils/toolSearch.ts` | `isDeferredTool`, `formatDeferredToolLine`, `TOOL_SEARCH_TOOL_NAME` |

## 依赖与外部交互

### 导入依赖

```typescript
// Bun 编译时特性
import { feature } from 'bun:bundle';

// 运行时状态
import { isReplBridgeActive } from '../../bootstrap/state.js';

// 特性标志
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../../services/analytics/growthbook.js';

// 类型
import type { Tool } from '../../Tool.js';

// Agent 工具常量
import { AGENT_TOOL_NAME } from '../AgentTool/constants.js';

// 本地常量
export { TOOL_SEARCH_TOOL_NAME } from './constants.js';
import { TOOL_SEARCH_TOOL_NAME } from './constants.js';
```

### 外部交互

1. **GrowthBook 特性标志**
   - `tengu_glacier_2xr`：控制延迟工具通过 delta attachments 还是传统消息块展示

2. **环境变量**
   - `USER_TYPE === 'ant'`：内部用户特殊处理

3. **运行时状态**
   - `isReplBridgeActive()`：检查 REPL 桥接是否激活（影响 SendUserFileTool）

4. **Bun 编译时特性**
   - `feature('KAIROS')` / `feature('KAIROS_BRIEF')` / `feature('FORK_SUBAGENT')`
   - 未启用的特性代码在打包时被消除

## 风险、边界与改进建议

### 潜在风险

1. **循环导入风险**
   - 风险：`isDeferredTool()` 中动态 require `../AgentTool/forkSubagent.js` 是为了避免循环导入
   - 当前：静态导入 `forkSubagent` 会导致通过 `coordinatorMode` 在模块初始化时产生循环
   - 缓解：使用动态 require，但失去了类型安全
   - 建议：考虑重构以消除循环依赖

2. **特性标志缓存**
   - 风险：`getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过期值
   - 影响：`getToolLocationHint()` 可能返回过时的位置提示
   - 缓解：用于非关键 UI 提示，不影响核心功能
   - 建议：在关键路径上使用实时特性检查

3. **动态 require 的类型安全**
   - 风险：动态 require 绕过 TypeScript 的类型检查
   - 当前：使用类型断言 `as typeof import(...)`
   - 建议：添加运行时类型检查或单元测试

4. **重复导入**
   - 问题：第 23-25 行存在重复导入
   ```typescript
   export { TOOL_SEARCH_TOOL_NAME } from './constants.js'
   import { TOOL_SEARCH_TOOL_NAME } from './constants.js'
   ```
   - 建议：合并为一条语句

### 边界情况

1. **工具名称匹配**
   - `isDeferredTool` 使用严格相等 (`===`) 匹配工具名称
   - 不考虑别名（aliases），即使工具有别名，判断仍基于主名称

2. **特性组合**
   - `FORK_SUBAGENT` + `KAIROS` 同时启用时，Agent 和 Brief 工具都首回合可用
   - 优先级顺序：`alwaysLoad` > `isMcp` > `TOOL_SEARCH` > `FORK_SUBAGENT` > `KAIROS`

3. **MCP 工具的特殊性**
   - MCP 工具默认延迟，但可通过 `_meta['anthropic/alwaysLoad']` 选择加入
   - 这是 MCP 协议级别的元数据，不是工具定义的一部分

4. **REPL 桥接状态**
   - `SendUserFileTool` 的延迟判断还依赖 `isReplBridgeActive()`
   - 如果桥接未激活，即使 KAIROS 启用也会延迟加载

### 改进建议

1. **代码清理**
   - 修复第 23-25 行的重复导入
   - 统一导入风格（全部使用命名空间导入或默认导入）

2. **类型安全**
   - 为动态 require 添加运行时类型守卫
   - 考虑使用 `import()` 动态导入语法替代 require

3. **可测试性**
   - `isDeferredTool` 依赖多个全局状态（feature flags、环境变量）
   - 建议：重构为纯函数，将依赖作为参数传入
   ```typescript
   export function isDeferredTool(tool: Tool, context: DeferredToolContext): boolean
   ```

4. **文档完善**
   - 为 `isDeferredTool` 的每个判断分支添加 JSDoc 说明
   - 解释各特性标志的用途和交互

5. **性能优化**
   - `BRIEF_TOOL_NAME` 和 `SEND_USER_FILE_TOOL_NAME` 在每次调用时重新计算
   - 建议：如果特性标志在运行时不变，可以缓存结果

6. **错误处理**
   - 动态 require 可能失败（如文件不存在）
   - 当前：没有 try-catch，失败会抛出异常
   - 建议：添加错误处理并记录警告
