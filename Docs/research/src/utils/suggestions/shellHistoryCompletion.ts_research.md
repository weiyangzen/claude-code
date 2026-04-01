# shellHistoryCompletion.ts 深度研究文档

## 场景与职责

`shellHistoryCompletion.ts` 是 Claude Code CLI 的**Shell 历史命令补全模块**，专门处理以 `!` 开头的历史命令引用（bash 风格的 history expansion）。该模块在用户输入 `!` 后提供来自历史记录的命令补全建议：

1. **历史命令补全**：当用户输入 `!` 后跟部分命令时，从历史记录中匹配并补全
2. **幽灵文本显示**：将匹配命令的剩余部分作为幽灵文本（ghost text）显示
3. **缓存管理**：缓存历史命令以避免重复的异步读取
4. **缓存更新**：支持向缓存前端添加新命令（去重并置顶）

该模块是命令行历史体验的重要组成部分，帮助用户快速重用之前的命令。

## 功能点目的

### 1. Shell 历史补全 (`getShellHistoryCompletion`)
- **目的**：根据用户输入返回最佳匹配的历史命令
- **匹配规则**：
  - 输入必须以 `!` 开头（调用方处理）
  - 至少 2 个字符才触发匹配
  - 精确前缀匹配（包括空格）
  - 排除与输入完全相同的命令
- **返回格式**：`{ fullCommand, suffix }`，其中 `suffix` 用于幽灵文本显示

### 2. 历史命令获取 (`getShellHistoryCommands`)
- **目的**：从全局历史记录中提取 bash 命令
- **过滤逻辑**：
  - 仅包含以 `!` 开头的历史条目
  - 去除 `!` 前缀获取实际命令
  - 去重（保留最近的一次）
  - 限制 50 条最近命令
- **缓存策略**：60 秒 TTL，避免重复读取

### 3. 缓存管理
- **清除缓存** (`clearShellHistoryCache`): 历史更新时调用
- **前置添加** (`prependToShellHistoryCache`): 新命令提交时添加到缓存前端

## 具体技术实现

### 关键数据结构

```typescript
// Shell 历史匹配结果
type ShellHistoryMatch = {
  fullCommand: string  // 完整命令
  suffix: string       // 幽灵文本部分（用户输入之后的部分）
}

// 缓存状态
let shellHistoryCache: string[] | null = null
let shellHistoryCacheTimestamp = 0
const CACHE_TTL_MS = 60000  // 60 秒
```

### 核心算法流程

#### 1. 历史命令获取流程
```
getShellHistoryCommands()
├── 缓存检查
│   ├── 缓存存在且未过期 → 返回缓存
│   └── 缓存过期/不存在 → 继续读取
├── 读取历史
│   ├── 遍历 getHistory() 生成器
│   ├── 过滤：entry.display.startsWith('!')
│   ├── 处理：command = entry.display.slice(1).trim()
│   ├── 去重：使用 Set 跟踪已见命令
│   └── 限制：最多 50 条
├── 更新缓存
│   ├── 存储命令列表
│   └── 更新时间戳
└── 返回结果
```

#### 2. 补全匹配流程
```
getShellHistoryCompletion(input)
├── 输入验证
│   ├── 空或 < 2 字符 → 返回 null
│   └── 仅空白字符 → 返回 null
├── 获取历史命令（含缓存）
├── 遍历匹配
│   └── 寻找第一个以 input 开头且不等于 input 的命令
├── 返回匹配结果
│   └── { fullCommand, suffix: command.slice(input.length) }
└── 无匹配 → 返回 null
```

#### 3. 缓存前置添加流程
```
prependToShellHistoryCache(command)
├── 缓存未初始化 → 无操作（下次读取会获取完整历史）
├── 命令已存在
│   └── 移除旧位置，添加到前端
└── 命令不存在
    └── 直接添加到前端
```

### 关键代码路径

| 功能 | 函数 | 行号 |
|------|------|------|
| Shell 历史补全 | `getShellHistoryCompletion` | 91-119 |
| 获取历史命令 | `getShellHistoryCommands` | 23-57 |
| 清除缓存 | `clearShellHistoryCache` | 62-65 |
| 前置添加 | `prependToShellHistoryCache` | 74-83 |

### 历史记录格式处理

```typescript
// 历史条目格式（来自 history.ts）
interface HistoryEntry {
  display: string           // 显示文本（可能以 ! 开头）
  pastedContents: Record<number, PastedContent>
}

// 提取 bash 命令
if (entry.display && entry.display.startsWith('!')) {
  const command = entry.display.slice(1).trim()
  // ...
}
```

## 依赖与外部交互

### 直接依赖模块

```typescript
import { getHistory } from '../../history.js'  // 历史记录读取
import { logForDebugging } from '../debug.js'  // 调试日志
```

### 依赖详解

1. **history.ts** (内部模块)
   - `getHistory()`: 异步生成器，按时间倒序返回历史条目
   - 返回 `HistoryEntry` 对象，包含 `display` 字段
   - 支持跨会话历史（从 `~/.claude/history.jsonl` 读取）
   - 当前会话条目优先于其他会话

2. **debug.ts** (内部模块)
   - `logForDebugging()`: 记录调试信息
   - 仅在调试模式下输出

### 调用方

- `useTypeahead.tsx`: 类型提示钩子，检测 `!` 前缀并调用补全
- `PromptInput.tsx`: 输入处理，应用历史补全建议

### 历史记录存储

历史记录存储在 `~/.claude/history.jsonl`，格式为：
```json
{
  "display": "!git commit -m 'update'",
  "pastedContents": {},
  "timestamp": 1234567890,
  "project": "/path/to/project",
  "sessionId": "uuid"
}
```

## 风险、边界与改进建议

### 已知风险

1. **缓存过期延迟**
   - 60 秒 TTL 意味着新命令可能需要最多 60 秒才出现在补全中
   - **当前处理**：`prependToShellHistoryCache` 用于立即更新缓存
   - **风险**：如果调用方忘记调用前置添加，用户体验不一致

2. **前缀匹配局限性**
   - 仅支持前缀匹配，不支持模糊搜索
   - **示例**：输入 `!com` 匹配 `commit` 但不匹配 `git commit`
   - **影响**：用户需要记住命令开头

3. **历史记录污染**
   - 所有以 `!` 开头的条目都被视为 bash 命令
   - **风险**：如果用户输入 `!important note`，会被当作命令
   - **当前处理**：这是预期行为，符合 bash history expansion 语义

4. **并发读取**
   - 无锁机制保护缓存更新
   - **风险**：理论上可能出现竞态条件（实际影响较小）

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| 输入 < 2 字符 | 不返回建议（避免过多匹配） |
| 仅空白字符 | 不返回建议 |
| 精确匹配（无后缀） | 不返回建议（command !== input） |
| 历史读取失败 | 捕获错误，返回空数组，记录调试日志 |
| 缓存未初始化 | `prependToShellHistoryCache` 无操作 |
| 命令已存在 | 移动到缓存前端（去重） |
| 超过 50 条历史 | 仅保留最近 50 条 |

### 改进建议

1. **匹配算法增强**
   - 支持模糊匹配（类似命令补全的 Fuse.js 实现）
   - 支持子串匹配（如 `!commit` 匹配 `git commit`）
   - 添加最近使用权重（更频繁的命令优先）

2. **缓存策略优化**
   - 减少 TTL 到 30 秒或更短
   - 实现增量更新（仅读取新条目）
   - 添加文件系统监视（`fs.watch`）主动刷新

3. **功能扩展**
   - 支持命令参数补全（如 `!git co` → `git commit`）
   - 添加命令分类（git、npm、shell 等）
   - 支持历史命令的模糊搜索 UI（类似 Ctrl+R）

4. **可配置性**
   - 允许用户配置缓存 TTL
   - 可配置的最大历史条目数
   - 排除特定模式的命令（如 `!secret`）

5. **性能优化**
   - 使用更高效的数据结构（Trie 树）加速前缀匹配
   - 延迟加载历史记录（仅在需要时读取）
   - 预加载常用命令到内存

### 测试要点

- 各种输入长度的边界处理
- 缓存命中/未命中/过期行为
- 前置添加的去重逻辑
- 历史读取失败的错误处理
- 并发访问的安全性
- 大历史文件的性能表现
- 特殊字符和空格的匹配处理

### 相关代码参考

- `history.ts`: 历史记录的完整实现
- `useArrowKeyHistory.tsx`: 上下箭头历史导航
- `shellCompletion.ts`: Shell 命令补全（与历史补全配合使用）
