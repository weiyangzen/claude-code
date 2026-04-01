# LogSelector.tsx 研究文档

## 1. 场景与职责

### 1.1 核心定位
`LogSelector` 是一个 React 组件，用于在终端 UI 中提供会话（Session）选择器功能。它是 `/resume` 命令的核心 UI 组件，允许用户从历史会话列表中选择并恢复特定会话。

### 1.2 使用场景
- **会话恢复**: 用户执行 `/resume` 命令时，展示可恢复的历史会话列表
- **跨项目恢复**: 支持切换显示当前项目或所有项目的会话
- **会话搜索**: 提供实时搜索过滤功能，支持标题、分支、标签、PR 信息等多维度搜索
- **会话预览**: 允许用户在恢复前预览会话内容
- **会话重命名**: 支持为会话设置自定义标题

### 1.3 调用方
| 调用方 | 路径 | 用途 |
|--------|------|------|
| ResumeCommand | `src/commands/resume/resume.tsx` | `/resume` 命令的交互式选择器 |
| ResumeConversation | `src/screens/ResumeConversation.tsx` | 应用启动时的会话恢复界面 |

---

## 2. 功能点目的

### 2.1 主要功能模块

#### 2.1.1 列表视图 (List View)
- **目的**: 以树形或扁平列表展示会话
- **功能**:
  - 按会话 ID 分组显示相关会话（fork 的会话）
  - 支持展开/折叠会话组
  - 显示会话标题、元数据（时间、分支、消息数等）

#### 2.1.2 搜索模式 (Search Mode)
- **目的**: 快速过滤会话列表
- **功能**:
  - 实时标题搜索（支持标题、分支、标签、PR 号）
  - 深度搜索（已禁用，代码保留）
  - Agentic 搜索（已禁用，代码保留）- 使用 Claude AI 进行语义搜索

#### 2.1.3 标签过滤 (Tag Filter)
- **目的**: 按用户定义的标签分类筛选
- **功能**:
  - 显示所有可用标签
  - Tab 键循环切换标签
  - "All" 标签显示全部会话

#### 2.1.4 分支过滤 (Branch Filter)
- **目的**: 只显示当前 Git 分支的会话
- **快捷键**: `Ctrl+B`

#### 2.1.5 Worktree 过滤
- **目的**: 在多 worktree 场景中，只显示当前 worktree 的会话
- **快捷键**: `Ctrl+W`

#### 2.1.6 会话预览 (Session Preview)
- **目的**: 在恢复前查看会话完整内容
- **快捷键**: `Ctrl+V`
- **实现**: 调用 `SessionPreview` 组件

#### 2.1.7 会话重命名 (Session Rename)
- **目的**: 为会话设置自定义标题
- **快捷键**: `Ctrl+R`
- **依赖**: `isCustomTitleEnabled()` 功能开关

---

## 3. 具体技术实现

### 3.1 关键数据结构

```typescript
// 组件 Props 定义
export type LogSelectorProps = {
  logs: LogOption[];                    // 会话列表
  maxHeight?: number;                   // 最大高度
  forceWidth?: number;                  // 强制宽度
  onCancel?: () => void;                // 取消回调
  onSelect: (log: LogOption) => void;   // 选择回调
  onLogsChanged?: () => void;           // 日志变化回调（用于刷新）
  onLoadMore?: (count: number) => void; // 加载更多回调
  initialSearchQuery?: string;          // 初始搜索词
  showAllProjects?: boolean;            // 是否显示所有项目
  onToggleAllProjects?: () => void;     // 切换项目范围回调
  onAgenticSearch?: (query: string, logs: LogOption[], signal?: AbortSignal) => Promise<LogOption[]>;
};

// Agentic 搜索状态机
type AgenticSearchState = 
  | { status: 'idle' }
  | { status: 'searching' }
  | { status: 'results'; results: LogOption[]; query: string }
  | { status: 'error'; message: string };

// 树节点类型
type LogTreeNode = TreeNode<{
  log: LogOption;
  indexInFiltered: number;
}>;
```

### 3.2 关键流程

#### 3.2.1 会话过滤流程
```
原始日志列表
    ↓
[isResumeWithRenameEnabled] 过滤无效会话
    ↓
[tagFilter] 标签过滤
    ↓
[branchFilter] 分支过滤
    ↓
[showAllWorktrees] Worktree 过滤
    ↓
[searchQuery] 标题搜索过滤
    ↓
[deepSearch] 深度搜索（已禁用）
    ↓
[agenticSearch] AI 搜索（已禁用）
    ↓
最终显示列表
```

#### 3.2.2 键盘输入处理流程
```
useInput 捕获键盘事件
    ↓
根据 viewMode 分发处理:
  - "preview": 忽略（SessionPreview 处理）
  - "search": 搜索框输入处理
  - "rename": 重命名输入处理
  - "list": 列表导航处理
    ↓
快捷键映射:
  - "/" 或任意字符: 进入搜索模式
  - "Ctrl+R": 进入重命名模式
  - "Ctrl+V": 进入预览模式
  - "Ctrl+B": 切换分支过滤
  - "Ctrl+W": 切换 Worktree 过滤
  - "Ctrl+A": 切换所有项目（如提供 onToggleAllProjects）
  - "Tab": 切换标签
```

#### 3.2.3 树形列表构建流程
```
filteredLogs
    ↓
groupLogsBySessionId() 按会话 ID 分组
    ↓
对每个分组:
  - 单一会话: 创建叶节点
  - 多会话: 创建父节点（最新）+ 子节点（其余）
    ↓
buildLogLabel() 构建显示标签
    ↓
buildLogMetadata() 构建元数据描述
    ↓
TreeNode[] 树形结构
```

### 3.3 核心算法

#### 3.3.1 会话分组算法
```typescript
function groupLogsBySessionId(filteredLogs: LogOption[]): Map<string, LogOption[]> {
  const groups = new Map<string, LogOption[]>();
  for (const log of filteredLogs) {
    const sessionId = getSessionIdFromLog(log);
    if (sessionId) {
      const existing = groups.get(sessionId);
      if (existing) {
        existing.push(log);
      } else {
        groups.set(sessionId, [log]);
      }
    }
  }
  // 按修改时间排序（最新的在前）
  groups.forEach(logs => logs.sort((a, b) => 
    new Date(b.modified).getTime() - new Date(a.modified).getTime()
  ));
  return groups;
}
```

#### 3.3.2 搜索文本构建算法
```typescript
function buildSearchableText(log: LogOption): string {
  // 裁剪长会话以提高性能
  const searchableMessages = log.messages.length <= DEEP_SEARCH_MAX_MESSAGES 
    ? log.messages 
    : [...log.messages.slice(0, DEEP_SEARCH_CROP_SIZE), 
       ...log.messages.slice(-DEEP_SEARCH_CROP_SIZE)];
  
  const messageText = searchableMessages
    .map(extractSearchableText)
    .filter(Boolean)
    .join(' ');
  
  const metadata = [
    log.customTitle, log.summary, log.firstPrompt,
    log.gitBranch, log.tag, log.prNumber ? `PR #${log.prNumber}` : undefined,
    log.prRepository
  ].filter(Boolean).join(' ');
  
  const fullText = `${metadata} ${messageText}`.trim();
  return fullText.length > DEEP_SEARCH_MAX_TEXT_LENGTH 
    ? fullText.slice(0, DEEP_SEARCH_MAX_TEXT_LENGTH) 
    : fullText;
}
```

### 3.4 性能优化

#### 3.4.1 React Compiler 缓存
组件使用 React Compiler（`_c` 函数）进行自动记忆化，大量依赖 `$[n]` 缓存槽位避免重复计算。

#### 3.4.2 虚拟滚动
- 通过 `visibleCount` 计算可见选项数量
- 配合 `onLoadMore` 实现分页加载
- 当焦点接近列表底部时触发加载更多

#### 3.4.3 防抖搜索
```typescript
// 300ms 防抖
const timeoutId = setTimeout(setDebouncedDeepSearchQuery, 300, deferredSearchQuery);
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件
| 文件 | 职责 |
|------|------|
| `src/components/LogSelector.tsx` | 主组件实现 |
| `src/types/logs.ts` | `LogOption`, `SerializedMessage` 类型定义 |

### 4.2 依赖组件
| 组件 | 路径 | 用途 |
|------|------|------|
| TreeSelect | `src/components/ui/TreeSelect.tsx` | 树形选择器 UI |
| Select | `src/components/CustomSelect/select.tsx` | 扁平列表选择器 UI |
| SessionPreview | `src/components/SessionPreview.tsx` | 会话预览 |
| TagTabs | `src/components/TagTabs.tsx` | 标签页切换 |
| SearchBox | `src/components/SearchBox.tsx` | 搜索输入框 |
| Spinner | `src/components/Spinner.tsx` | 加载指示器 |

### 4.3 依赖 Hooks
| Hook | 路径 | 用途 |
|------|------|------|
| useSearchInput | `src/hooks/useSearchInput.ts` | 搜索输入管理 |
| useTerminalSize | `src/hooks/useTerminalSize.ts` | 终端尺寸监听 |
| useExitOnCtrlCDWithKeybindings | `src/hooks/useExitOnCtrlCDWithKeybindings.ts` | Ctrl+C 退出处理 |
| useKeybinding | `src/keybindings/useKeybinding.js` | 键盘快捷键绑定 |

### 4.4 依赖工具函数
| 函数 | 路径 | 用途 |
|------|------|------|
| getLogDisplayTitle | `src/utils/log.ts` | 获取会话显示标题 |
| formatLogMetadata | `src/utils/format.ts` | 格式化会话元数据 |
| getSessionIdFromLog | `src/utils/sessionStorage.ts` | 从日志提取会话 ID |
| isCustomTitleEnabled | `src/utils/sessionStorage.ts` | 检查自定义标题功能开关 |
| saveCustomTitle | `src/utils/sessionStorage.ts` | 保存自定义标题 |
| getWorktreePaths | `src/utils/getWorktreePaths.ts` | 获取 worktree 路径列表 |
| getBranch | `src/utils/git.ts` | 获取当前 Git 分支 |
| agenticSessionSearch | `src/utils/agenticSessionSearch.ts` | AI 语义搜索 |

### 4.5 关键常量
```typescript
// 深度搜索限制
const DEEP_SEARCH_MAX_MESSAGES = 2000;      // 最大消息数
const DEEP_SEARCH_CROP_SIZE = 1000;         // 裁剪大小
const DEEP_SEARCH_MAX_TEXT_LENGTH = 50000;  // 最大文本长度
const FUSE_THRESHOLD = 0.3;                 // Fuse.js 模糊搜索阈值
const DATE_TIE_THRESHOLD_MS = 60 * 1000;    // 时间平局阈值（1分钟）
const SNIPPET_CONTEXT_CHARS = 50;           // 搜索片段上下文字符数

// 树形前缀宽度
const PARENT_PREFIX_WIDTH = 2;  // '▼ ' or '▶ '
const CHILD_PREFIX_WIDTH = 4;   // '  ▸ '
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

#### 5.1.1 NPM 包
| 包名 | 用途 |
|------|------|
| `chalk` | 终端颜色输出 |
| `figures` | 终端符号（箭头、指针等） |
| `fuse.js` | 模糊搜索（深度搜索功能，当前禁用） |
| `react` | React 核心 |

#### 5.1.2 内部模块
- **Ink UI 框架**: `src/ink.js`, `src/ink/*.ts`
- **主题系统**: `src/utils/theme.ts`
- **分析埋点**: `src/services/analytics/index.js`
- **键盘绑定系统**: `src/keybindings/useKeybinding.js`

### 5.2 状态管理
- **本地状态**: 使用 `React.useState` 管理组件内部状态
- **全局状态**: 通过 `useTheme()` 获取当前主题
- **持久化状态**: 通过 `sessionStorage.ts` 读写会话元数据

### 5.3 事件系统
- **键盘事件**: 通过 `useInput` 和 `useKeybinding` 处理
- **分析事件**: 通过 `logEvent` 上报用户行为
  - `tengu_session_search_toggled`
  - `tengu_session_tag_filter_changed`
  - `tengu_session_all_projects_toggled`
  - `tengu_session_branch_filter_toggled`
  - `tengu_session_worktree_filter_toggled`
  - `tengu_session_rename_started`
  - `tengu_session_preview_opened`
  - `tengu_session_group_expanded`
  - `tengu_agentic_search_started`
  - `tengu_agentic_search_completed`
  - `tengu_agentic_search_error`
  - `tengu_agentic_search_cancelled`

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 功能开关风险
- `isDeepSearchEnabled` 和 `isAgenticSearchEnabled` 被硬编码为 `false`
- 相关代码路径虽被禁用但仍保留，增加维护负担
- 深度搜索的 Fuse.js 依赖被引入但未实际使用

#### 6.1.2 性能风险
- 大日志列表（>1000 条）的过滤和分组操作在主线程同步执行
- `buildSearchableText` 可能处理大量文本（最大 50KB 每会话）
- React Compiler 缓存虽优化渲染，但计算逻辑仍可能阻塞

#### 6.1.3 状态一致性风险
- `expandedGroupSessionIds` 使用 Set 存储，但在分支过滤模式下强制展开所有组
- 视图模式切换（list/search/rename/preview）时状态重置逻辑复杂

### 6.2 边界情况

#### 6.2.1 空状态处理
```typescript
if (logs.length === 0) {
  return null;  // 直接返回 null，无空状态提示
}
```
- 调用方需自行处理空列表情况

#### 6.2.2 跨项目恢复
- 组件本身不处理跨项目恢复逻辑，由调用方 `ResumeCommand` 处理
- 仅通过 `showAllProjects` 和 `onToggleAllProjects` 提供 UI 支持

#### 6.2.3 Lite 日志
- Lite 日志（`isLite: true`）的消息列表不完整
- 预览模式会自动加载完整日志（`loadFullLog`）

### 6.3 改进建议

#### 6.3.1 代码清理
1. **移除死代码**: 删除深度搜索和 Agentic 搜索的禁用代码，或提取为独立组件
2. **简化条件**: `isResumeWithRenameEnabled` 检查分散在多处，可集中处理
3. **类型优化**: 部分临时函数（`_temp`, `_temp2` 等）可内联或命名优化

#### 6.3.2 性能优化
1. **虚拟化**: 大列表使用虚拟滚动（当前仅分页）
2. **Web Worker**: 将搜索过滤移至 Worker 线程
3. **记忆化**: `filteredLogs` 的计算可进一步优化

#### 6.3.3 功能增强
1. **多选支持**: 当前仅支持单选，可考虑多选批量操作
2. **排序选项**: 当前仅按时间排序，可增加按名称、消息数排序
3. **键盘导航**: 增加更多 Vim 风格快捷键（如 `gg`, `G`）

#### 6.3.4 可测试性
1. **单元测试**: 当前无直接测试，建议添加
2. **逻辑拆分**: 将过滤逻辑、分组逻辑提取为纯函数便于测试
3. **Mock 支持**: 提供测试用的 Mock 数据生成器

### 6.4 架构建议

```
建议重构结构:

LogSelector/
├── index.tsx           # 主组件，仅负责布局
├── useLogFiltering.ts  # 过滤逻辑 Hook
├── useLogGrouping.ts   # 分组逻辑 Hook
├── useLogSearch.ts     # 搜索逻辑 Hook（含深度/Agentic）
├── LogList.tsx         # 列表渲染组件
├── LogTreeItem.tsx     # 树节点渲染组件
└── utils.ts            # 纯工具函数
```

这种拆分将提高可维护性和可测试性，同时便于后续功能扩展。

---

## 7. 总结

`LogSelector` 是一个功能丰富的会话选择器组件，支撑了 Claude Code CLI 的核心会话恢复流程。其设计考虑了多种使用场景（搜索、过滤、预览、重命名），并通过 React Compiler 优化了渲染性能。

主要特点：
- **功能完整**: 支持多种过滤维度（标签、分支、worktree、项目）
- **交互友好**: 丰富的键盘快捷键，树形/扁平列表切换
- **性能优化**: 防抖搜索、分页加载、React Compiler 缓存

主要问题：
- **代码膨胀**: 包含大量已禁用功能的代码
- **复杂度**: 单文件 1500+ 行，逻辑耦合度高
- **测试缺失**: 无直接单元测试覆盖

建议优先级：
1. 高: 清理死代码，降低维护成本
2. 中: 拆分逻辑，提高可测试性
3. 低: 功能增强（多选、排序等）
