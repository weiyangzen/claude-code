# DiffDialog.tsx 研究文档

## 场景与职责

`DiffDialog` 是 Claude Code 中 `/diff` 命令的核心 UI 组件，提供交互式的 diff 查看体验。它是 diff 功能的入口点和状态管理中枢，负责：

1. **双源 Diff 数据整合** - 同时展示当前 Git 工作区变更（`useDiffData`）和历史对话轮次变更（`useTurnDiffs`）
2. **视图模式管理** - 支持列表视图（list）和详情视图（detail）两种模式的切换
3. **键盘导航交互** - 完整的键盘快捷键支持（上下选择、左右切换源、回车查看详情等）
4. **Source 切换导航** - 在 "Current"（当前 Git 变更）和各历史 Turn 之间切换
5. **Overlay 生命周期管理** - 注册为模态覆盖层，协调 Escape 键行为

该组件通过 `/diff` 命令触发，以 JSX 组件形式渲染在终端 UI 中。

## 功能点目的

### 1. 双源 Diff 数据模型
- **Current（当前变更）**：通过 `useDiffData` 获取 Git 工作区相对于 HEAD 的变更
- **Turn Diff（历史轮次）**：通过 `useTurnDiffs` 从对话历史中提取每个用户轮次产生的文件变更
- **目的**：让用户既能查看当前未提交的修改，也能回顾 AI 在之前的对话中做了哪些更改

### 2. 视图状态管理
- **List 模式**：显示文件列表（`DiffFileList`），支持上下选择和 Enter 进入详情
- **Detail 模式**：显示单个文件的详细 diff（`DiffDetailView`），支持 Back 返回列表
- **状态持久化**：切换 Source 时自动重置选中索引，保持用户体验一致性

### 3. 键盘快捷键系统
完整的键盘导航支持：
- `←/→`：在多个 Source 之间切换（当存在历史 Turn 时）
- `↑/↓`：在 List 模式下选择文件
- `Enter`：从 List 进入 Detail 模式
- `Backspace/←`：从 Detail 返回 List 模式
- `Esc`：关闭 Dialog（或返回上级视图）

### 4. Source 导航器
当存在历史 Turn 变更时，在顶部显示 Source 切换栏：
- 显示格式：`◀ Current · T1 · T2 ▶`
- 当前选中 Source 高亮显示
- 支持循环导航（到边界后回绕）

## 具体技术实现

### 类型定义

```typescript
// 视图模式
type ViewMode = 'list' | 'detail';

// Diff 数据来源类型
type DiffSource = 
  | { type: 'current' }                    // 当前 Git 变更
  | { type: 'turn'; turn: TurnDiff };      // 特定轮次的变更

// 组件 Props
type Props = {
  messages: Message[];      // 对话历史，用于提取 TurnDiff
  onDone: (result?: string, options?: { display?: CommandResultDisplay }) => void;
};
```

### 核心状态

```typescript
const [viewMode, setViewMode] = useState<'list' | 'detail'>('list');
const [selectedIndex, setSelectedIndex] = useState(0);      // 当前选中文件索引
const [sourceIndex, setSourceIndex] = useState(0);          // 当前 Source 索引
```

### 数据转换：TurnDiff → DiffData

```typescript
function turnDiffToDiffData(turn: TurnDiff): DiffData {
  const files = Array.from(turn.files.values())
    .map(f => ({
      path: f.filePath,
      linesAdded: f.linesAdded,
      linesRemoved: f.linesRemoved,
      isBinary: false,
      isLargeFile: false,
      isTruncated: false,
      isNewFile: f.isNewFile
    }))
    .sort((a, b) => a.path.localeCompare(b.path));
  
  const hunks = new Map<string, StructuredPatchHunk[]>();
  for (const f of turn.files.values()) {
    hunks.set(f.filePath, f.hunks);
  }
  
  return {
    stats: {
      filesCount: turn.stats.filesChanged,
      linesAdded: turn.stats.linesAdded,
      linesRemoved: turn.stats.linesRemoved
    },
    files,
    hunks,
    loading: false
  };
}
```

### 键盘快捷键绑定

使用 `useKeybindings` hook 批量注册：

```typescript
useKeybindings({
  'diff:previousSource': () => { /* 切换到上一个 Source */ },
  'diff:nextSource': () => { /* 切换到下一个 Source */ },
  'diff:back': () => { /* Detail 模式返回 List */ },
  'diff:viewDetails': () => { /* List 模式进入 Detail */ },
  'diff:previousFile': () => { /* 上一个文件 */ },
  'diff:nextFile': () => { /* 下一个文件 */ }
}, { context: 'DiffDialog' });
```

### 边界保护 Effect

```typescript
// Source 索引越界保护
useEffect(() => {
  if (sourceIndex >= sources.length) {
    setSourceIndex(Math.max(0, sources.length - 1));
  }
}, [sources.length, sourceIndex]);

// Source 切换时重置选中索引
useEffect(() => {
  if (prevSourceIndex.current !== sourceIndex) {
    setSelectedIndex(0);
    prevSourceIndex.current = sourceIndex;
  }
}, [sourceIndex]);
```

### 空状态消息逻辑

```typescript
const emptyMessage = (() => {
  if (diffData.loading) return 'Loading diff…';
  if (currentTurn) return 'No file changes in this turn';
  if (diffData.stats && diffData.stats.filesCount > 0 && diffData.files.length === 0) {
    return 'Too many files to display details';
  }
  return 'Working tree is clean';
})();
```

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `src/components/diff/DiffFileList.tsx` | 文件列表展示组件 |
| `src/components/diff/DiffDetailView.tsx` | 单个文件 diff 详情组件 |
| `src/components/design-system/Dialog.tsx` | 对话框容器组件 |
| `src/components/design-system/Byline.tsx` | 快捷键提示行组件 |
| `src/hooks/useDiffData.ts` | 获取当前 Git diff 数据 |
| `src/hooks/useTurnDiffs.ts` | 从消息历史提取 Turn diff |
| `src/context/overlayContext.tsx` | `useRegisterOverlay` 注册覆盖层 |
| `src/keybindings/useKeybinding.ts` | `useKeybindings` 快捷键绑定 |
| `src/keybindings/useShortcutDisplay.ts` | `useShortcutDisplay` 获取快捷键显示文本 |
| `src/utils/stringUtils.ts` | `plural` 复数格式化 |

### 被调用方

| 文件路径 | 调用场景 |
|---------|---------|
| `src/commands/diff/diff.tsx` | `/diff` 命令入口，动态导入 DiffDialog |

### 命令定义

```typescript
// src/commands/diff/index.ts
export default {
  type: 'local-jsx',
  name: 'diff',
  description: 'View uncommitted changes and per-turn diffs',
  load: () => import('./diff.js'),
} satisfies Command;
```

## 依赖与外部交互

### 与 Git 子系统的交互
通过 `useDiffData` 间接调用：
- `fetchGitDiff()` - 获取 diff 统计信息
- `fetchGitDiffHunks()` - 获取 diff hunks 详情

### 与消息历史的交互
通过 `useTurnDiffs` 解析 `Message[]`，提取：
- `FileEditTool` 的编辑结果
- `FileWriteTool` 的创建/更新结果
- 生成合成 hunk 用于新文件展示

### 键盘事件系统
使用 `useKeybindings` 与全局键盘系统交互：
- 上下文名称：`'DiffDialog'`
- 支持 chord（组合键）序列
- 自动处理事件冒泡（`stopImmediatePropagation`）

### Overlay 系统
调用 `useRegisterOverlay("diff-dialog")`：
- 注册为活动覆盖层
- 影响 Escape 键行为（优先关闭覆盖层而非取消请求）
- 自动清理（unmount 时注销）

## 风险、边界与改进建议

### 已知风险

1. **Source 切换时的选中状态**：
   - 不同 Source 的文件列表长度可能不同
   - 当前实现：切换 Source 时重置 `selectedIndex` 为 0
   - 风险：用户可能丢失之前的选中位置

2. **大数据量渲染**：
   - 当存在大量历史 Turn（如 50+）时，Source 切换栏可能超出终端宽度
   - 当前无水平滚动或截断处理

3. **键盘快捷键冲突**：
   - 使用 `context: 'DiffDialog'` 隔离，但全局快捷键仍可能干扰
   - 如 `ctrl+c` 的处理需要与退出流程协调

### 边界情况

| 场景 | 处理行为 |
|-----|---------|
| `sources.length === 0` | 理论上不会发生（至少包含 Current） |
| `diffData.files.length === 0` | 显示空状态消息 |
| `selectedIndex` 越界 | Effect 自动修正到有效范围 |
| 快速切换 Source | React Compiler 的 memoization 优化渲染性能 |
| 终端宽度变化 | 子组件（DiffFileList, DiffDetailView）响应式调整 |

### 改进建议

1. **用户体验优化**：
   - Source 切换时保持相对选中位置（如选中第 3 个文件，切换后仍尝试选中第 3 个）
   - 添加搜索/过滤功能，支持按文件名快速定位
   - 支持在 Detail 视图直接跳转到下一个/上一个文件的 diff

2. **性能优化**：
   - 对 `sources` 数组使用 `useMemo` 避免重复计算
   - 虚拟化长文件列表（当文件数 > 100 时）
   - 延迟加载历史 Turn 的 diff 数据（当前是一次性计算）

3. **代码结构**：
   - 将 `turnDiffToDiffData` 提取到独立工具文件，便于测试
   - 键盘快捷键配置提取到 keybindings schema，支持用户自定义
   - 空状态消息逻辑可提取为独立函数

4. **可访问性**：
   - 增加 ARIA 标签支持（如 `aria-label` 描述当前选中文件）
   - 支持屏幕阅读器朗读 diff 统计信息

5. **功能扩展**：
   - 支持 Stage/Unstage 操作（集成 `git add`/`git reset`）
   - 支持 Diff 导出（复制到剪贴板或保存到文件）
   - 支持文件对比模式（选择两个 Turn 对比差异）

6. **错误处理**：
   - 增加 Git 命令失败的错误提示
   - 文件读取权限问题的友好提示
   - 网络文件系统（如 SSHFS）的降级处理
