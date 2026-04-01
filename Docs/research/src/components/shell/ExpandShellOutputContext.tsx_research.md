# ExpandShellOutputContext.tsx 研究文档

## 场景与职责

`ExpandShellOutputContext.tsx` 是一个极轻量的 React Context 模块，职责单一：为 shell 输出提供“自动展开”信号。当用户发送最新的 bash 命令（`!` 命令）后，其对应的输出消息需要默认展开显示（不截断），而历史消息则保持截断以节省屏幕空间。该模块通过布尔型 Context 将这一状态透传给下游的 `OutputLine` 等渲染组件，使它们能够根据是否处于“最新输出”上下文来决定是否执行内容截断。

## 功能点目的

1. **自动展开最新 bash 输出**  
   在消息列表中，只有 `latestBashOutputUUID === message.uuid` 的最新用户 bash 输出消息会被 `ExpandShellOutputProvider` 包裹，从而让其子树内的所有 `OutputLine` 默认展示完整内容。

2. **避免嵌套截断逻辑与消息层耦合**  
   截断决策属于渲染层细节，不应由 `Message.tsx` 直接通过 props 层层传递。Context 模式让 `OutputLine` 自行感知环境，保持接口简洁。

3. **与现有模式对齐**  
   代码注释明确说明其遵循与 `MessageResponseContext`、`SubAgentContext` 相同的布尔 Context 设计模式：Provider 固定传 `true`，子组件通过 `useContext` 读取。

## 具体技术实现

### 关键流程

```
Message.tsx (判断 isLatestBashOutput)
    │
    ├─ true  → <ExpandShellOutputProvider>{content}</ExpandShellOutputProvider>
    │
    └─ false → {content} (无 Provider)
                           │
                           ▼
              OutputLine.tsx 调用 useExpandShellOutput()
                           │
              ├─ 在 Provider 内 → true → 不截断
              └─ 不在 Provider 内 → false → 可能截断
```

### 数据结构

- `ExpandShellOutputContext: React.Context<boolean>`  
  默认值 `false`，表示“不在自动展开区域内”。

- `ExpandShellOutputProvider(props: { children: React.ReactNode })`  
  直接返回 `<ExpandShellOutputContext.Provider value={true}>{children}</...>`。

- `useExpandShellOutput(): boolean`  
  对 `useContext(ExpandShellOutputContext)` 的薄封装。

### 编译产物特征

文件已被 React Compiler（`react/compiler-runtime`）编译，产物中使用了 `_c(2)` 等 memo cache 辅助函数，但源码逻辑极其简单，仅涉及 children 的透传。

## 关键代码路径与文件引用

| 路径 | 角色 |
|------|------|
| `src/components/shell/ExpandShellOutputContext.tsx` | 本文件：定义 Context、Provider、Hook |
| `src/components/Message.tsx:31` | 导入 `ExpandShellOutputProvider` |
| `src/components/Message.tsx:191-229` | 计算 `isLatestBashOutput`，条件包裹 Provider |
| `src/components/shell/OutputLine.tsx:11` | 导入 `useExpandShellOutput` |
| `src/components/shell/OutputLine.tsx:59` | 调用 Hook，决定 `shouldShowFull` |

## 依赖与外部交互

- **React**：仅依赖 `React.createContext` 与 `useContext`。
- **无第三方依赖**。
- **上游调用方**：`Message.tsx` 是唯一 Provider 挂载点。
- **下游消费方**：`OutputLine.tsx` 是唯一消费方（通过 `useExpandShellOutput`）。

## 风险、边界与改进建议

1. **Context 覆盖范围较粗**  
   Provider 包裹的是整条用户消息（`content`），如果消息内包含多个文本块或混合内容，所有子组件都会收到 `expand=true`。目前这符合设计预期，因为 bash 输出消息通常只包含单一文本块。

2. **无卸载/重置逻辑**  
   Provider 只是静态传 `true`，不存在状态变化导致的重渲染问题，性能开销可忽略。

3. **扩展性**  
   若未来需要“仅展开 stdout 而不展开 stderr”等更细粒度控制，当前布尔 Context 将不够用，需要升级为对象 Context 或增加额外 props。但就当前需求而言，保持简单是合理选择。

4. **测试覆盖**  
   在仓库中未检索到针对该 Context 的单元测试。由于其逻辑极为简单（纯 React 模式），风险较低，但如需回归保护，可补充一个测试：验证 Provider 内 `useExpandShellOutput()` 返回 `true`，外部返回 `false`。
