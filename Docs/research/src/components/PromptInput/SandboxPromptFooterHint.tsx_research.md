# SandboxPromptFooterHint.tsx 研究文档

## 场景与职责

PromptInput footer 动态提示组件，在 sandbox 启用且发生权限违规时，展示最近被拦截的操作数量，并提示查看详情快捷键与 `/sandbox` 禁用命令。

## 功能点目的

- 实时展示新增 sandbox 违规数
- 5 秒后自动消隐
- 引导用户查看详情或禁用沙箱

## 具体技术实现

### 关键流程

1. `useEffect` 检查 `SandboxManager.isSandboxingEnabled()`，若启用则订阅 `SandboxViolationStore`
2. 用 `lastCount` 记录上一次总数，回调中 `currentCount - lastCount` 得新增量
3. 新增量 > 0 时 `setRecentViolationCount(newViolations)` 并启动 5s 定时器，到时归零
4. Cleanup：`unsubscribe()` + `clearTimeout()`
5. 渲染：`recentViolationCount > 0` 时显示 `⧈ Sandbox blocked {count} operation(s) · {detailsShortcut} for details · /sandbox to disable`

### 数据结构

- `recentViolationCount: number` — 最近 5s 内新增违规数
- `timerRef: React.MutableRefObject<NodeJS.Timeout | null>` — 消隐定时器
- `detailsShortcut: string` — `useShortcutDisplay("app:toggleTranscript", "Global", "ctrl+o")`

### 协议/命令

- `@anthropic-ai/sandbox-runtime` 的 `SandboxViolationStore`（`getTotalCount` / `subscribe`）
- `src/utils/sandbox/sandbox-adapter.ts` 的 `SandboxManager`
- `src/keybindings/useShortcutDisplay.ts`

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/SandboxPromptFooterHint.tsx` | 本组件 |
| `src/components/PromptInput/Notifications.tsx` | 调用方，footer 通知流末尾渲染 |
| `src/utils/sandbox/sandbox-adapter.ts` | SandboxManager 适配层 |
| `src/keybindings/useShortcutDisplay.ts` | 快捷键显示 |

## 依赖与外部交互

- 源码经 React Compiler 编译，使用 `_c` 缓存数组
- 使用 `ink.js` 的 `<Box>`、`<Text>` 渲染终端 UI

## 风险、边界与改进建议

1. **计数覆盖**：5s 内连续多批违规，后一批重置定时器，用户可能只看到最后一批数量
2. **低对比度风险**：仅使用 `inactive` 色，部分终端不够醒目，可考虑违规突增时用 `warning` 色
3. **测试覆盖**：建议补充 store 订阅模拟、5s 消隐、sandbox 未启用返回 null 的单元测试
