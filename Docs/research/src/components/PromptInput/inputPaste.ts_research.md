# inputPaste.ts 研究文档

## 场景与职责

处理超大文本粘贴进输入框时的截断与引用替换。当粘贴文本超过 10,000 字符时，将超长部分折叠为占位符 `[...Truncated text #N +X lines...]`，并把被截断内容存入 `pastedContents`，在提交时由 `expandPastedTextRefs` 恢复。

## 功能点目的

- 超过阈值时只保留开头和结尾各约 500 字符
- 生成带 ID 和行数提示的占位符
- 将被截断的中间部分以 `PastedContent(type: 'text')` 写入记录
- 自动分配递增 ID，避免与已有引用冲突

## 具体技术实现

### 关键流程

1. **`maybeTruncateMessageForInput(text, nextPasteId)`**
   - `text.length <= 10000` 时原样返回，`placeholderContent` 为空
   - 否则取前后各 500 字符，中间部分作为 `placeholderContent`
   - 调用 `getPastedTextRefNumLines` 计算被截断行数
   - 生成占位符：`[...Truncated text #${id} +${lines} lines...]`

2. **`maybeTruncateInput(input, pastedContents)`**
   - 从 `pastedContents` 取最大 ID +1 作为 `nextPasteId`
   - 调用上述函数，若 `placeholderContent` 为空则返回原样
   - 否则返回截断输入和扩展后的 `pastedContents`

### 数据结构

```ts
const TRUNCATION_THRESHOLD = 10000
const PREVIEW_LENGTH = 1000

type TruncatedMessage = { truncatedText: string; placeholderContent: string }
```

- `PastedContent` 来自 `src/utils/config.ts`

### 协议/命令

- `getPastedTextRefNumLines` → `src/history.ts`
- 占位符格式与 `src/history.ts` 的 `parseReferences` 正则兼容，确保提交时可展开

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/inputPaste.ts` | 本文件 |
| `src/components/PromptInput/useMaybeTruncateInput.ts` | 调用 `maybeTruncateInput` |
| `src/components/PromptInput/PromptInput.tsx` | 调用 hook 并管理 `pastedContents` |
| `src/history.ts` | 提供行数计算、引用解析与展开 |
| `src/utils/config.ts` | 提供 `PastedContent` 类型 |

## 依赖与外部交互

纯工具函数，与 `history.ts` 的引用解析系统深度耦合，占位符格式必须与其正则一致。

## 风险、边界与改进建议

1. **阈值硬编码**：`10000` 和 `1000` 未考虑终端宽度或性能差异，建议根据终端高度动态调整或暴露为隐藏设置
2. **行数计算是物理换行符**：`getPastedTextRefNumLines` 计算 `\r\n|\r|\n`，而非 ink 渲染后的折行行数。终端很窄时可能产生"行数对不上"的困惑
3. **ID 分配竞态**：`Math.max(...existingIds) + 1` 理论上在单渲染周期内安全（有 `useMaybeTruncateInput` 保护），但非 React 调用方需注意
4. **测试建议**：覆盖 10000/10001 边界、ID 分配、占位符可被 `parseReferences` 正确解析
