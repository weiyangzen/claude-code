# useMaybeTruncateInput.ts 研究文档

## 场景与职责

React hook，确保 PromptInput 的截断逻辑对同一个输入值只执行一次，防止重复截断导致输入不断缩短、ID 不断递增。输入清空（如提交后）时重置标志位，为下一次大粘贴做准备。

## 功能点目的

- 一次性截断，避免重渲染时重复执行
- 截断后光标移到新输入末尾
- 同步新 `pastedContents` 到父组件
- 输入清空后重置标志位

## 具体技术实现

### 关键流程

1. `useState(false)` 维护 `hasAppliedTruncationToInput`
2. **截断 effect**：
   - 依赖：`input`、`hasAppliedTruncationToInput`、`pastedContents`、各 setter
   - `hasAppliedTruncationToInput === true` 或 `input.length <= 10000` 时直接 return
   - 调用 `maybeTruncateInput` 得到 `newInput` 和 `newPastedContents`
   - 依次调用 `onInputChange(newInput)`、`setCursorOffset(newInput.length)`、`setPastedContents(newPastedContents)`
   - 设置标志位为 true
3. **重置 effect**：依赖 `[input]`，`input === ''` 时重置标志位为 false

### 数据结构

```ts
type Props = {
  input: string
  pastedContents: Record<number, PastedContent>
  onInputChange: (input: string) => void
  setCursorOffset: (offset: number) => void
  setPastedContents: (contents: Record<number, PastedContent>) => void
}
```

### 协议/命令

- `maybeTruncateInput` → `src/components/PromptInput/inputPaste.ts`

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/useMaybeTruncateInput.ts` | 本 hook |
| `src/components/PromptInput/PromptInput.tsx` | 唯一调用方 |
| `src/components/PromptInput/inputPaste.ts` | 截断逻辑 |
| `src/utils/config.ts` | `PastedContent` 类型 |

## 依赖与外部交互

完全依赖父组件 props，与 `PromptInput.tsx` 的 `trackAndSetInput` 形成受控更新闭环。

## 风险、边界与改进建议

1. **阈值魔法数字重复**：hook 中硬编码 `10000`，与 `inputPaste.ts` 的 `TRUNCATION_THRESHOLD` 耦合。建议从 `inputPaste.ts` 导出常量并引用
2. **光标强制末尾**：截断后光标移到末尾，若用户需要中间编辑可能显得突兀，但在 10K+ 字符场景下可接受
3. **pastedContents 引用稳定性**：effect 依赖数组包含 `pastedContents` 对象，但 `hasAppliedTruncationToInput` 为 true 后会阻断再次执行，不会无限循环
4. **测试建议**：验证大文本触发截断、同一输入不重复截断、清空后再次输入大文本可重新截断
