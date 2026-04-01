# Research: src/ink/measure-element.ts

## 场景与职责

`measure-element.ts` 暴露一个用于测量 `<Box>` 元素尺寸的辅助函数。Ink 的组件层（React）允许用户通过 `ref` 获取底层 DOM 节点，并调用 `measureElement(ref)` 获取该节点在 Yoga 布局完成后的计算宽高。这在需要基于实际渲染尺寸做动态调整（如自适应弹窗、tooltip 定位）时非常有用。

## 功能点目的

1. **组件层 API 支撑**：为 `useMeasureElement` hook 或直接的 `measureElement` 调用提供底层实现。
2. **Yoga 结果透传**：将 Yoga 布局引擎计算后的宽高以简单对象形式返回，屏蔽 `yogaNode` 的 C++ / WASM 接口细节。
3. **安全降级**：若节点尚未完成布局（`yogaNode` 不存在），返回 `{ width: 0, height: 0 }`，避免调用方处理 `undefined`。

## 具体技术实现

```ts
type Output = {
  width: number
  height: number
}

const measureElement = (node: DOMElement): Output => ({
  width: node.yogaNode?.getComputedWidth() ?? 0,
  height: node.yogaNode?.getComputedHeight() ?? 0,
})

export default measureElement
```

- 参数 `node` 必须是 `DOMElement`（即 `ink-box` 等宿主节点），不能是文本节点。
- 使用可选链 `?.` 访问 Yoga 的计算属性，空值合并 `?? 0` 保证返回值始终为数字。

## 关键代码路径与文件引用

- **导出位置**：`src/ink/measure-element.ts` 默认导出 `measureElement`。
- **调用方**：
  - 在 Ink 的公共 API 中通过 `src/ink/measure-element.js` 重新导出（或直接在组件层使用）。
  - 用户代码中常见的调用链：`useRef<DOMElement>() -> measureElement(ref.current)`。
- **类型依赖**：`src/ink/dom.js` — `DOMElement` 类型定义。

## 依赖与外部交互

- 仅依赖 `./dom.js` 中的 `DOMElement` 类型。
- 无运行时外部依赖，无状态，无副作用。

## 风险、边界与改进建议

- **风险**：调用方若在 Yoga 布局尚未计算（如首次渲染的同步阶段）调用 `measureElement`，会得到 `{0,0}`，容易误判为元素不存在。文档或 hook 层应提示用户确保在布局完成后（如 `useLayoutEffect`）读取。
- **边界**：
  - 仅测量 `Box`（`ink-box`）等拥有 `yogaNode` 的节点；对纯 `Text` 节点调用会返回 `0,0`（因为 `TextNode` 不是 `DOMElement`）。
  - 返回的是"计算尺寸"，不包含边框（border）的外扩；若用户设置了 `borderStyle`，视觉尺寸会比返回值大 2（上下/左右各 1）。
- **改进建议**：
  1. 可考虑扩展返回对象，增加 `x`、`y`（相对于父节点的偏移），让调用方获得完整布局信息。
  2. 对 `Text` 节点可返回基于 `stringWidth` / 行数的测量结果，提升 API 一致性。
  3. 增加开发环境断言：若传入 `TextNode` 则抛出明确错误，帮助用户快速定位问题。
