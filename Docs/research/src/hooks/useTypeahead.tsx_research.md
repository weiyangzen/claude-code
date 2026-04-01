# useTypeahead.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useTypeahead` 是 Claude Code CLI 中 **PromptInput 组件的智能输入补全引擎**，负责处理用户在命令行输入时的所有自动补全功能。它是连接用户输入与各类数据源（文件、命令、历史记录、MCP 资源等）的桥梁。

### 1.2 使用场景
- **Slash Command 补全**: 用户输入 `/` 时提供命令建议（如 `/commit`, `/help`）
- **文件路径补全**: 用户输入 `@` 时提供项目文件、MCP 资源、Agent 建议
- **目录补全**: `/add-dir` 等命令的参数补全
- **Shell 补全**: Bash 模式下的命令、变量、文件补全
- **历史记录补全**: Bash 模式下基于历史命令的幽灵文本提示
- **Slack 频道补全**: `#` 触发 Slack 频道搜索
- **Agent/队友补全**: `@` 触发团队成员或命名子代理建议
- **会话标题补全**: `/resume` 命令的自定义标题搜索

### 1.3 架构位置
```
PromptInput.tsx (UI 层)
    ↓ 调用
useTypeahead.tsx (业务逻辑层)
    ↓ 调用
├── fileSuggestions.ts (文件索引管理)
├── unifiedSuggestions.ts (统一建议聚合)
├── commandSuggestions.ts (命令建议生成)
├── directoryCompletion.ts (目录补全)
├── shellCompletion.ts (Shell 补全)
├── shellHistoryCompletion.ts (历史补全)
└── slackChannelSuggestions.ts (Slack 频道)
```

---

## 2. 功能点目的

### 2.1 主要功能模块

| 功能模块 | 触发条件 | 目的 |
|---------|---------|------|
| **Command Suggestions** | 输入以 `/` 开头 | 帮助用户快速找到并执行内置命令 |
| **File Suggestions** | 输入包含 `@` | 快速引用项目文件、MCP 资源、Agent |
| **Directory Completion** | `/add-dir` 等命令 | 精确选择目录路径 |
| **Shell Completion** | Bash 模式下输入 | 提供 bash/zsh 风格的命令补全 |
| **Ghost Text** | Bash 模式或 mid-input `/` | 提供内联预览和快速补全 |
| **Slack Channel** | 输入 `#` + 频道名 | 快速引用 Slack 频道 |
| **Agent/Teammate** | 输入 `@` + 名称 | 向团队成员发送消息 |
| **Session Resume** | `/resume` + 标题 | 按自定义标题搜索历史会话 |

### 2.2 核心设计目标

1. **低延迟响应**: 文件搜索使用 Rust/Nucleo 风格的模糊匹配算法，50ms debounce
2. **渐进式加载**: 文件索引后台构建，先返回部分结果，完成后自动刷新
3. **智能排序**: 基于使用频率、匹配精度、命令类型排序
4. **多模式支持**: 同时支持 Prompt 模式和 Bash 模式的不同补全策略
5. **缓存优化**: 多级缓存策略（内存、LRU、Git 索引签名）

---

## 3. 具体技术实现

### 3.1 核心数据结构

#### Props 定义
```typescript
type Props = {
  onInputChange: (value: string) => void;        // 输入变更回调
  onSubmit: (value: string, isSubmittingSlashCommand?: boolean) => void;
  setCursorOffset: (offset: number) => void;     // 光标位置控制
  input: string;                                  // 当前输入值
  cursorOffset: number;                          // 光标位置
  commands: Command[];                           // 可用命令列表
  mode: string;                                   // 'prompt' | 'bash'
  agents: AgentDefinition[];                     // Agent 定义列表
  setSuggestionsState: Function;                 // 建议状态更新
  suggestionsState: {                            // 建议状态
    suggestions: SuggestionItem[];
    selectedSuggestion: number;
    commandArgumentHint?: string;
  };
  suppressSuggestions?: boolean;                 // 是否禁用建议
  markAccepted: () => void;                      // 接受建议标记
  onModeChange?: (mode: PromptInputMode) => void;
};
```

#### SuggestionItem 结构
```typescript
type SuggestionItem = {
  id: string;                    // 唯一标识
  displayText: string;           // 显示文本
  tag?: string;                  // 标签（如 'workflow'）
  description?: string;          // 描述
  metadata?: unknown;            // 元数据（命令对象、分数等）
  color?: keyof Theme;           // 主题颜色
};
```

#### SuggestionType 枚举
```typescript
type SuggestionType = 
  | 'command'      // 命令建议
  | 'file'         // 文件建议
  | 'directory'    // 目录建议
  | 'agent'        // Agent/队友建议
  | 'shell'        // Shell 补全
  | 'custom-title' // 会话标题
  | 'slack-channel'// Slack 频道
  | 'none';        // 无建议
```

### 3.2 关键正则表达式

```typescript
// Unicode-aware 文件路径字符类
const AT_TOKEN_HEAD_RE = /^@[\p{L}\p{N}\p{M}_\-./\\()[\]~:]*/u;
const PATH_CHAR_HEAD_RE = /^[\p{L}\p{N}\p{M}_\-./\\()[\]~:]+/u;
const TOKEN_WITH_AT_RE = /(@[\p{L}\p{N}\p{M}_\-./\\()[\]~:]*|[\p{L}\p{N}\p{M}_\-./\\()[\]~:]+)$/u;
const HAS_AT_SYMBOL_RE = /(^|\s)@([\p{L}\p{N}\p{M}_\-./\\()[\]~:]*|"[^"]*"?)$/u;
const HASH_CHANNEL_RE = /(^|\s)#([a-z0-9][a-z0-9_-]*)$/;
const DM_MEMBER_RE = /(^|\s)@[\w-]*$/;
```

### 3.3 核心流程

#### 3.3.1 建议更新流程 (`updateSuggestions`)

```
1. 检查 suppressSuggestions - 禁用则清空
2. 检查 mid-input slash command (prompt 模式)
   - 发现 ghost text 候选 → 清空下拉建议
3. Bash 模式历史补全检查
   - 匹配历史命令 → 设置 inlineGhostText
4. @ 触发队友/Agent 建议
   - 搜索 teammates 和 agentNameRegistry
5. # 触发 Slack 频道建议
6. /add-dir 命令的目录补全
7. /resume 命令的会话标题搜索
8. 普通命令建议生成
9. @ 触发文件/MCP/Agent 统一建议
```

#### 3.3.2 Tab 键处理流程 (`handleTab`)

```
1. 优先处理 ghost text
   - Bash 模式: 替换整个输入
   - Prompt 模式: 替换 mid-input 命令
2. 根据 suggestionType 处理:
   - 'command': applyCommandSuggestion
   - 'custom-title': buildResumeInputFromSuggestion
   - 'directory': applyDirectorySuggestion
   - 'shell': applyShellSuggestion
   - 'agent': applyTriggerSuggestion
   - 'slack-channel': applyTriggerSuggestion
   - 'file': 处理公共前缀或完整建议
```

#### 3.3.3 Enter 键处理流程 (`handleEnter`)

```
1. 验证有选中建议
2. 根据 suggestionType 执行:
   - 'command': 应用并执行（无参数命令）
   - 'custom-title': 应用并执行 /resume
   - 'shell': 应用建议
   - 'agent': 应用建议
   - 'slack-channel': 应用建议
   - 'file': 应用文件建议
   - 'directory': 应用目录建议或提交命令
```

### 3.4 文件索引技术

#### FileIndex 类 (native-ts/file-index/index.ts)

**核心算法**: Nucleo 风格的模糊匹配

```typescript
class FileIndex {
  private paths: string[] = [];
  private lowerPaths: string[] = [];
  private charBits: Int32Array = new Int32Array(0);
  private pathLens: Uint16Array = new Uint16Array(0);
  private readyCount = 0;  // 渐进式加载标记

  // 位图加速: a-z 字母存在性位图
  // 快速排除不包含查询字符的路径
  
  search(query: string, limit: number): SearchResult[] {
    // 1. 位图过滤 O(1)
    // 2. indexOf 扫描找匹配位置
    // 3. 边界/驼峰 bonus 计算
    // 4. Top-K 维护（避免全排序）
  }
}
```

**评分常量**:
```typescript
const SCORE_MATCH = 16;
const BONUS_BOUNDARY = 8;      // 路径边界奖励
const BONUS_CAMEL = 6;         // 驼峰转换奖励
const BONUS_CONSECUTIVE = 4;   // 连续匹配奖励
const BONUS_FIRST_CHAR = 8;    // 首字符匹配奖励
const PENALTY_GAP_START = 3;   // 间隔惩罚
const PENALTY_GAP_EXTENSION = 1;
```

### 3.5 缓存策略

#### 文件建议缓存 (fileSuggestions.ts)

```typescript
// 全局缓存变量
let fileIndex: FileIndex | null = null;
let fileListRefreshPromise: Promise<FileIndex> | null = null;
let cacheGeneration = 0;
let cachedTrackedFiles: string[] = [];
let cachedConfigFiles: string[] = [];
let cachedTrackedDirs: string[] = [];
let ignorePatternsCache: ReturnType<typeof ignore> | null = null;

// 刷新节流
const REFRESH_THROTTLE_MS = 5_000;

// 路径列表签名（检测变更）
function pathListSignature(paths: string[]): string {
  // 采样哈希: 每 500 个路径采样一个 + 首尾
  // 346k 路径只需哈希 ~700 个
}
```

#### 目录缓存 (directoryCompletion.ts)

```typescript
const CACHE_SIZE = 500;
const CACHE_TTL = 5 * 60 * 1000; // 5 分钟

const directoryCache = new LRUCache<string, DirectoryEntry[]>({
  max: CACHE_SIZE,
  ttl: CACHE_TTL,
});
```

### 3.6 统一建议聚合 (unifiedSuggestions.ts)

**多源数据融合**:

```typescript
export async function generateUnifiedSuggestions(
  query: string,
  mcpResources: Record<string, ServerResource[]>,
  agents: AgentDefinition[],
  showOnEmpty = false,
): Promise<SuggestionItem[]> {
  // 并行获取文件建议和 Agent 建议
  const [fileSuggestions, agentSources] = await Promise.all([
    generateFileSuggestions(query, showOnEmpty),
    Promise.resolve(generateAgentSuggestions(agents, query, showOnEmpty)),
  ]);

  // MCP 资源转换
  const mcpSources = Object.values(mcpResources).flat().map(...);
  
  // 文件使用 Nucleo 分数 (0-1, 越低越好)
  // 非文件源使用 Fuse.js 评分
  // 合并后按分数排序
}
```

**Fuse.js 配置**:
```typescript
const fuse = new Fuse(nonFileSources, {
  includeScore: true,
  threshold: 0.6,
  keys: [
    { name: 'displayText', weight: 2 },
    { name: 'name', weight: 3 },
    { name: 'server', weight: 1 },
    { name: 'description', weight: 1 },
    { name: 'agentType', weight: 3 },
  ],
});
```

---

## 4. 关键代码路径与文件引用

### 4.1 主文件

| 文件 | 职责 | 行数 |
|-----|------|------|
| `src/hooks/useTypeahead.tsx` | 主 Hook 实现 | 1385 |
| `src/hooks/fileSuggestions.ts` | 文件索引管理 | 811 |
| `src/hooks/unifiedSuggestions.ts` | 统一建议聚合 | 202 |
| `src/native-ts/file-index/index.ts` | 模糊搜索算法 | 370 |

### 4.2 辅助文件

| 文件 | 职责 |
|-----|------|
| `src/utils/suggestions/commandSuggestions.ts` | 命令建议生成、Fuse 索引 |
| `src/utils/suggestions/directoryCompletion.ts` | 目录扫描、LRU 缓存 |
| `src/utils/suggestions/shellHistoryCompletion.ts` | 历史命令缓存与匹配 |
| `src/utils/suggestions/slackChannelSuggestions.ts` | Slack 频道搜索缓存 |
| `src/utils/bash/shellCompletion.ts` | Bash/Zsh 补全命令生成 |
| `src/utils/argumentSubstitution.ts` | 参数解析与提示生成 |
| `src/utils/sessionStorage.ts` | 会话标题搜索 |

### 4.3 调用方

| 文件 | 调用方式 |
|-----|---------|
| `src/components/PromptInput/PromptInput.tsx` | 主要消费者，传递状态和回调 |
| `src/components/mcp/ElicitationDialog.tsx` | 独立的 typeahead 实现（枚举字段） |

### 4.4 关键函数调用链

**文件建议路径**:
```
useTypeahead.updateSuggestions
  → debouncedFetchFileSuggestions
    → fetchFileSuggestions
      → generateUnifiedSuggestions
        → generateFileSuggestions
          → startBackgroundCacheRefresh
            → getPathsForSuggestions
              → getFilesUsingGit / getProjectFiles
                → FileIndex.loadFromFileListAsync
        → generateAgentSuggestions
```

**命令建议路径**:
```
useTypeahead.updateSuggestions
  → generateCommandSuggestions (commandSuggestions.ts)
    → getCommandFuse (Fuse.js 缓存)
    → fuse.search(query)
```

---

## 5. 依赖与外部交互

### 5.1 React Hooks 依赖

```typescript
// 状态管理
const [suggestionType, setSuggestionType] = useState<SuggestionType>('none');
const [maxColumnWidth, setMaxColumnWidth] = useState<number | undefined>();
const [inlineGhostText, setInlineGhostText] = useState<InlineGhostText | undefined>();

// Refs（避免不必要的重渲染）
const cursorOffsetRef = useRef(cursorOffset);
const latestSearchTokenRef = useRef<string | null>(null);
const suggestionsRef = useRef(suggestions);

// Debounce
const debouncedFetchFileSuggestions = useDebounceCallback(fetchFileSuggestions, 50);
const debouncedFetchSlackChannels = useDebounceCallback(fetchSlackChannels, 150);
```

### 5.2 外部状态依赖

```typescript
// AppState
const mcpResources = useAppState(s => s.mcp.resources);
const promptSuggestion = useAppState(s => s.promptSuggestion);
const isViewingTeammate = useAppState(s => !!s.viewingAgentTaskId);
const store = useAppStateStore();  // 用于 imperative 读取

// Context
const { addNotification } = useNotifications();
const keybindingContext = useOptionalKeybindingContext();

// 快捷键
useKeybindings(autocompleteHandlers, {
  context: 'Autocomplete',
  isActive: isAutocompleteActive && !isModalOverlayActive
});
```

### 5.3 系统交互

| 交互对象 | 方式 | 用途 |
|---------|------|------|
| Git | `git ls-files` | 快速获取跟踪文件列表 |
| Ripgrep | `rg --files` | 非 Git 仓库的文件发现 |
| Shell | `compgen` (bash) / `print -rl` (zsh) | Shell 补全 |
| MCP Server | `client.callTool` | Slack 频道搜索 |
| 文件系统 | `fs.readdir`, `fs.stat` | 目录扫描、索引构建 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 性能风险

| 风险点 | 影响 | 缓解措施 |
|-------|------|---------|
| 大仓库索引构建 | 270k+ 文件可能阻塞 UI | 渐进式构建 (CHUNK_MS=4ms)，异步加载 |
| 频繁 Git 调用 | 每次按键可能触发 `git ls-files` | 5s 节流 + .git/index mtime 检测 |
| Fuse.js 重建 | 命令列表变更时重建索引 | 按 commands 数组引用缓存 |
| 内存泄漏 | 文件索引持续增长 | 缓存签名检测，无变更跳过重建 |

#### 6.1.2 并发风险

```typescript
// 竞态条件防护: 丢弃过期结果
if (latestSearchTokenRef.current !== searchToken) {
  return;  // 丢弃过期结果
}

// AbortController 取消进行中的请求
if (currentShellCompletionAbortController) {
  currentShellCompletionAbortController.abort();
}
```

#### 6.1.3 边界情况

1. **空输入处理**: `@` 单独输入时应显示顶级目录
2. **引号路径**: `@"path with spaces"` 需要特殊解析
3. **光标位置**: mid-input 补全需要精确计算 token 位置
4. **模式切换**: Bash/Prompt 模式切换时清理建议状态
5. **测试环境**: `NODE_ENV=test` 跳过后台索引构建

### 6.2 改进建议

#### 6.2.1 性能优化

1. **Web Worker**: 将 FileIndex 搜索移至 Worker 线程，完全避免主线程阻塞
2. **增量索引**: 使用文件系统监听（如 chokidar）实现真正增量更新
3. **预加载**: 启动时预加载常用目录索引
4. **智能节流**: 根据输入速度动态调整 debounce 时间

#### 6.2.2 功能增强

1. **模糊匹配增强**: 支持拼音首字母匹配（对中文用户友好）
2. **最近使用优先**: 基于使用频率的动态排序
3. **多选支持**: 允许一次选择多个文件
4. **预览功能**: 选中文件时显示内容预览
5. **上下文感知**: 根据当前命令智能过滤建议类型

#### 6.2.3 代码质量

1. **类型安全**: `suggestion.metadata` 使用 `unknown` 类型，需要运行时类型守卫
2. **测试覆盖**: 缺少单元测试，特别是复杂交互场景
3. **错误边界**: 部分异步错误仅记录日志，用户体验可优化
4. **文档注释**: 复杂算法（如 FileIndex 评分）需要更多注释

#### 6.2.4 架构优化

```typescript
// 建议: 提取建议策略为插件架构
interface SuggestionStrategy {
  readonly trigger: RegExp;
  readonly priority: number;
  generateSuggestions(input: string, context: Context): Promise<SuggestionItem[]>;
  applySuggestion(suggestion: SuggestionItem, input: string): string;
}

// 当前实现是 monolithic 的 switch-case，可拆分为独立策略类
```

### 6.3 监控与调试

**现有日志**:
```typescript
logEvent('tengu_file_suggestions_query', {
  duration_ms: duration,
  cache_hit: !wasBuilding,
  result_count: matches.length,
  query_length: partialPath.length,
});

logForDebugging(`[FileIndex] generateFileSuggestions: ${matches.length} results in ${duration}ms`);
```

**建议增加**:
- 建议接受率统计
- 各类型建议的延迟分布
- 缓存命中率监控
- 用户输入模式分析

---

## 7. 附录

### 7.1 快捷键映射

| 快捷键 | 功能 |
|-------|------|
| Tab | 接受当前建议 / 触发补全 |
| Enter | 执行选中的命令建议 |
| ↑/↓ | 在建议列表中导航 |
| Ctrl+N/P | 备选导航方式 |
| Esc | 关闭建议列表 |
| → (右箭头) | 接受 ghost text 提示 |

### 7.2 配置项

```typescript
// 项目设置
interface ProjectSettings {
  respectGitignore?: boolean;  // 是否尊重 .gitignore
  fileSuggestion?: {
    type: 'command' | 'internal';
    command?: string;  // 自定义文件建议命令
  };
}
```

### 7.3 相关常量

```typescript
const MAX_SUGGESTIONS = 15;           // 文件建议上限
const MAX_UNIFIED_SUGGESTIONS = 15;   // 统一建议上限
const MAX_SHELL_COMPLETIONS = 15;     // Shell 补全上限
const OVERLAY_MAX_ITEMS = 5;          // 悬浮层显示上限
const REFRESH_THROTTLE_MS = 5_000;    // 刷新节流
const DEBOUNCE_MS = 50;               // 输入防抖
const CHUNK_MS = 4;                   // 事件循环让步阈值
```

---

*文档生成时间: 2026-04-01*
*研究范围: src/hooks/useTypeahead.tsx 及其直接依赖*
