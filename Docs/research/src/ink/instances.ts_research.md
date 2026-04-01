# Research: src/ink/instances.ts

## 场景与职责

`instances.ts` 是 Ink 渲染器的全局实例注册表。Ink 采用"按输出流复用实例"的策略：对同一个 `NodeJS.WriteStream`（通常是 `process.stdout`），多次调用 `render()` 或 `rerender()` 必须复用同一个 `Ink` 实例，而不是每次都新建。该模块提供一个以 `WriteStream` 为键、`Ink` 实例为值的 `Map`，供 `root.ts` 的 `renderSync` / `createRoot` 以及 `ink.tsx` 的 `Ink` 类自身读写。

## 功能点目的

1. **实例去重**：避免重复创建 `Ink` 导致的事件监听器泄漏、终端状态混乱（如多次进入 alt-screen、多次注册 stdin data 监听器）。
2. **跨模块共享**：`render.js`（通过 `root.ts`）负责在首次渲染时创建实例并写入 Map；`Ink.unmount()` 负责从 Map 删除自身，实现生命周期的闭环。
3. **按流隔离**：支持同时存在多个 Ink 应用分别写到不同的 stdout（例如主输出 + 辅助日志流），互不干扰。

## 具体技术实现

```ts
const instances = new Map<NodeJS.WriteStream, Ink>()
export default instances
```

- 数据结构：原生 `Map`，键为 `NodeJS.WriteStream` 对象引用，值为 `Ink` 实例。
- 无额外封装：直接导出 Map 实例，调用方使用标准 `get/set/delete`。
- 类型安全：导入 `Ink` 类型仅用于 TypeScript 类型标注，运行时不产生依赖。

## 关键代码路径与文件引用

- **写入路径**：
  - `src/ink/root.ts:90-104` — `renderSync` 中 `getInstance()` 若查不到实例则 `createInstance()` 并 `instances.set(stdout, instance)`。
  - `src/ink/root.ts:150` — `createRoot()` 新建 `Ink` 后同样 `instances.set(stdout, instance)`。
- **读取路径**：
  - `src/ink/root.ts:176-183` — `getInstance()` 优先 `instances.get(stdout)` 复用。
- **删除路径**：
  - `src/ink/ink.tsx` — `Ink.unmount()` 调用 `instances.delete(this.options.stdout)` 清理自身。
  - `src/ink/root.ts:103` — 返回的 `Instance.cleanup` 也暴露 `instances.delete`。

## 依赖与外部交互

- **被 `root.ts` 导入**：作为渲染入口的实例池。
- **被 `ink.tsx` 导入**：`Ink` 类在 unmount 时自我注销。
- **无运行时依赖**：模块本身不 import 任何运行时逻辑，仅 import `type Ink`。

## 风险、边界与改进建议

- **风险**：若调用方传入不同的 but 功能等价的 WriteStream 包装对象（如代理或 mock），Map 以对象引用为键会导致无法命中缓存，产生重复实例。
- **边界**：未做弱引用（WeakMap 不适合以 WriteStream 做键后还需要枚举/删除的场景），进程生命周期内 Map 会持有实例引用；但正常流程下 `unmount()` 会主动清理。
- **改进建议**：
  1. 若需要支持 `stdout` 被替换（如测试框架热替换 `process.stdout`），可考虑增加基于 `fd` 的降级查找。
  2. 可封装为 `get/set/remove` 方法，隐藏 Map 直接操作，便于未来加入调试日志或引用计数。
