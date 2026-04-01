# `src/components/Spinner/teammateSelectHint.ts` 研究

本研究仅基于当前仓库可见的代码、配置类型、hooks、任务实现、调用链与测试文件检索结果完成；未把 `README`、`Docs`、`docs`、其他 Markdown 文档作为研究输入。

## 场景与职责

`teammateSelectHint.ts` 是 teammate 多 agent 协作 UI 中的一个极小的常量模块，职责单一且明确：

1. **定义 teammate 选择的键盘提示文案**：为 `TeammateSpinnerLine` 和 `TeammateSpinnerTree` 提供统一的提示字符串，告诉用户如何通过键盘在 leader 和 teammate 之间导航。

该文件是整个 teammate 选择提示文案的单一事实来源（single source of truth），避免了提示文案在多个组件中硬编码导致的漂移风险。

## 功能点目的

- **文案统一**：确保 `TeammateSpinnerLine`、`TeammateSpinnerTree` 以及任何未来需要显示 teammate 导航提示的组件使用完全相同的文案。
- **维护便捷**：当键盘快捷键变更或需要国际化时，只需修改这一处常量即可全局生效。
- **语义清晰**：常量名 `TEAMMATE_SELECT_HINT` 明确表达了该字符串的用途和上下文。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 源码实现

`src/components/Spinner/teammateSelectHint.ts:1`

```typescript
export const TEAMMATE_SELECT_HINT = 'shift + ↑/↓ to select'
```

### 数据结构

- 导出一个名为 `TEAMMATE_SELECT_HINT` 的字符串常量。
- 值为 `'shift + ↑/↓ to select'`，使用 ASCII 加号、斜杠和 Unicode 上下箭头符号描述键盘组合。

### 使用位置

该常量在以下文件中被导入和使用：

1. **`src/components/Spinner/TeammateSpinnerLine.tsx:16, 132, 222`**
   - 导入：`import { TEAMMATE_SELECT_HINT } from './teammateSelectHint.js'`
   - 使用：在响应式布局中计算提示文本的显示宽度，并在条件满足时渲染 `<Text dimColor> · {TEAMMATE_SELECT_HINT}</Text>`。

2. **`src/components/Spinner/TeammateSpinnerTree.tsx:9, 120, 177`**
   - 导入：`import { TEAMMATE_SELECT_HINT } from './teammateSelectHint.js'`
   - 使用：在 leader 行高亮时渲染 `<Text dimColor> · {TEAMMATE_SELECT_HINT}</Text>`。

### 键盘协议对应关系

`TEAMMATE_SELECT_HINT` 中描述的 `shift + ↑/↓` 与 `src/hooks/useBackgroundTaskNavigation.ts:181-188` 中的实际键盘处理逻辑完全对应：

```typescript
if (e.shift && (e.key === 'up' || e.key === 'down')) {
  e.preventDefault()
  if (teammateCount > 0) {
    stepTeammateSelection(e.key === 'down' ? 1 : -1, setAppState)
  } else if (hasNonTeammateBackgroundTasks) {
    options?.onOpenBackgroundTasks?.()
  }
  return
}
```

这意味着该常量不仅是 UI 文案，也是键盘交互协议的**用户可见契约**。

## 关键代码路径与文件引用

- `src/components/Spinner/teammateSelectHint.ts:1`
  - 常量定义的唯一位置。
- `src/components/Spinner/TeammateSpinnerLine.tsx:16`
  - 导入位置。
- `src/components/Spinner/TeammateSpinnerLine.tsx:132`
  - 计算提示文本显示宽度：`const selectHintText = \` · ${TEAMMATE_SELECT_HINT}\``。
- `src/components/Spinner/TeammateSpinnerLine.tsx:222`
  - 条件渲染提示文本。
- `src/components/Spinner/TeammateSpinnerTree.tsx:9`
  - 导入位置。
- `src/components/Spinner/TeammateSpinnerTree.tsx:120`
  - leader 行高亮时渲染提示。
- `src/hooks/useBackgroundTaskNavigation.ts:181-188`
  - 实际处理 `Shift+↑/↓` 键盘事件的逻辑。

## 依赖与外部交互

### 直接依赖

该文件没有任何运行时依赖，也不依赖任何类型定义。它是一个纯粹的字符串常量模块。

### 外部交互

- **消费者**：`TeammateSpinnerLine.tsx`、`TeammateSpinnerTree.tsx`。
- **键盘协议实现方**：`useBackgroundTaskNavigation.ts`。
- **全局 keybinding 系统**：`src/hooks/useGlobalKeybindings.tsx:198-205` 注册了 `app:toggleTeammatePreview` action，但 teammate 选择的物理按键绑定直接由 `useBackgroundTaskNavigation` 处理，不经过全局 keybinding 系统。

## 风险、边界与改进建议

### 1. 文案与真实键盘协议的一致性风险

`TEAMMATE_SELECT_HINT` 是一个纯文本常量，没有任何机制确保它与 `useBackgroundTaskNavigation.ts` 中的实际按键处理逻辑保持一致。如果未来键盘导航逻辑改为 `Ctrl+↑/↓` 或 `Tab` 切换，但开发者忘记同步修改这个常量，用户会看到错误的提示文案。

**建议**：
- 在 `useBackgroundTaskNavigation.ts` 附近添加显式注释，提醒修改键盘导航键时必须同步更新 `teammateSelectHint.ts`。
- 考虑将按键描述生成逻辑与实际的 keybinding 配置关联起来，例如从 keybinding 配置对象中动态生成提示字符串，而不是硬编码。

### 2. 国际化（i18n）缺失

当前常量是纯英文硬编码字符串，没有使用任何国际化框架或翻译函数。如果产品需要支持多语言，这个提示文案会成为漏网之鱼。

**建议**：
- 当项目引入 i18n 时，将该常量迁移到翻译键体系中，例如 `t('spinner.teammateSelectHint')`。
- 在此之前，可以在文件顶部添加 `// TODO(i18n): migrate to translation keys` 标记。

### 3. 符号名称的语义范围

常量名 `TEAMMATE_SELECT_HINT` 很好，但如果未来需要为 teammate 树添加更多提示（如 "enter to view"、"enter to collapse"、"f to view transcript"），这些提示目前分散在组件的 JSX 中硬编码。`teammateSelectHint.ts` 只覆盖了其中一个提示。

**建议**：
- 评估是否将 teammate 树的所有提示文案统一收敛到一个 `teammateHints.ts` 模块中，包括：
  - `TEAMMATE_SELECT_HINT`
  - `TEAMMATE_VIEW_HINT`（"enter to view"）
  - `TEAMMATE_COLLAPSE_HINT`（"enter to collapse"）
  - `TEAMMATE_KILL_HINT`（"k to kill"，如果未来需要显示）
- 这样可以进一步降低文案漂移风险，并便于统一国际化。

### 4. 箭头符号的终端兼容性

常量中使用了 Unicode 箭头 `↑` 和 `↓`。虽然现代终端普遍支持这些字符，但在某些老旧终端、特定字体或远程 SSH 环境中，这些符号可能出现显示异常或宽度计算错误。

**建议**：
- 确认 `stringWidth` 函数对 `↑` 和 `↓` 的宽度计算是否正确（应为 2 个显示列，因为它们是 East Asian Wide 字符）。
- 如果终端兼容性成为问题，可考虑回退到 ASCII 表示形式，如 `'shift + up/down to select'`。

### 5. 文件粒度的极端精简

该文件只有 1 行有效代码，64 字节。虽然单一职责原则支持这种粒度，但也有人认为过小的文件会增加项目文件数量和管理开销。

**建议**：
- 如果项目风格倾向于合并小常量文件，可以考虑将其并入一个更广泛的 `spinnerConstants.ts` 或 `teammateConstants.ts` 中。
- 如果项目风格倾向于保持单一职责，则当前状态是合理的，无需改动。
