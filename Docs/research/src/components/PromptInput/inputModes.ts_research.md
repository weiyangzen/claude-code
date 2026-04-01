# inputModes.ts 研究文档

## 场景与职责

PromptInput 的模式字符解析器，负责用户输入与内部 `PromptInputMode` / `HistoryMode` 的双向转换。当用户在空输入框键入 `!` 时识别为 `bash` 模式；历史记录导航也依赖它判断记录属于 `bash` 还是 `prompt`。

## 功能点目的

- 为 `bash` 模式自动加上 `!` 前缀
- 从输入字符串反推当前模式
- 去掉模式字符后返回纯文本内容
- 判断单个字符是否为模式切换字符

## 具体技术实现

### 关键流程

四个纯函数：

| 函数 | 逻辑 |
|------|------|
| `prependModeCharacterToInput(input, mode)` | `mode === 'bash'` 时返回 `!${input}`，否则原样 |
| `getModeFromInput(input)` | 以 `!` 开头返回 `bash`，否则 `prompt` |
| `getValueFromInput(input)` | 先取 mode，`prompt` 原样返回，`bash` 去掉首字符 |
| `isInputModeCharacter(input)` | 仅当 `input === '!'` 返回 true |

### 数据结构

- `HistoryMode = PromptInputMode`
- `PromptInputMode` 来自 `src/types/textInputTypes.ts`：`bash` | `prompt` | `orphaned-permission` | `task-notification`

### 协议/命令

无外部 IO、无 React 依赖、无状态管理。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/inputModes.ts` | 本文件 |
| `src/components/PromptInput/PromptInput.tsx` | 首字符插入检测、模式切换、历史搜索解析 |
| `src/hooks/useArrowKeyHistory.tsx` | 过滤历史记录模式 |
| `src/hooks/useTypeahead.tsx` | 引用 |
| `src/hooks/useHistorySearch.ts` | 引用 |
| `src/hooks/useTextInput.ts` | 引用 |
| `src/screens/REPL.tsx` | 引用 `prependModeCharacterToInput` |

## 依赖与外部交互

完全纯函数，是 PromptInput 与历史系统之间的契约层。

## 风险、边界与改进建议

1. **扩展性瓶颈**：模式字符硬编码为 `!`，若未来增加更多模式，需同步修改多处逻辑。建议提取为常量字典 `{ '!': 'bash' }`
2. **前缀冲突**：`getModeFromInput` 仅检测 `startsWith('!')`，`!!something` 会被识别为 `bash`，`getValueFromInput` 只去掉第一个 `!`。当前设计可接受
3. **历史过滤局限**：若历史记录以 `!` 开头但实为普通 prompt，会被错误归类到 `bash` 模式。这是模式字符方案的固有局限
4. **测试建议**：覆盖空字符串、仅 `!`、`!command`、普通文本四个边界
