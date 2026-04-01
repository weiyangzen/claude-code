# useShowFastIconHint.ts 研究文档

## 场景与职责

控制 PromptInput 中 Fast 模式图标旁提示文本的显示逻辑。首次满足条件时弹出 `/fast` 提示，持续 5 秒后自动消失，且整个会话只展示一次。

## 功能点目的

- 提示用户可通过 `/fast` 切换快速模式
- 会话级去重，只显示一次
- 5 秒后自动隐藏
- 仅在 `showFastIcon` 为 true 时触发

## 具体技术实现

### 关键流程

1. 模块级变量 `let hasShownThisSession = false`
2. `useState(false)` 维护 `showHint`
3. `useEffect` 依赖 `[showFastIcon]`：
   - `hasShownThisSession` 或 `showFastIcon` 为 false 时直接 return
   - 设置 `hasShownThisSession = true`，`setShowHint(true)`
   - 启动 `setTimeout(..., 5000, false)`
   - Cleanup：`clearTimeout` + `setShowHint(false)`

### 数据结构

- `hasShownThisSession: boolean` — 模块级会话唯一标志
- `showHint: boolean` — 组件级渲染开关
- `HINT_DISPLAY_DURATION_MS = 5000`

### 协议/命令

无外部依赖，纯 UI 行为 hook。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/useShowFastIconHint.ts` | 本 hook |
| `src/components/PromptInput/PromptInput.tsx` | 调用方，传入 `showFastIcon` |

## 依赖与外部交互

仅依赖 React `useEffect` / `useState`。`showFastIcon` 的生成逻辑在 `PromptInput.tsx` 中，涉及 fast 模式可用性判断。

## 风险、边界与改进建议

1. **无磁盘持久化**：每次启动新进程都会再显示一次。若需限制总次数，可借鉴 voice hint 存入 `getGlobalConfig()`
2. **组件卸载不重置**：`hasShownThisSession` 是模块级变量，卸载再挂载不会重新显示
3. **快速切换 showFastIcon**：若 false→true→false→true 在 5s 内发生，由于 `hasShownThisSession` 已变 true，后续 true 不会触发。符合"一次性"语义
4. **测试建议**：验证首次 true 触发显示、5s 后隐藏、重新挂载不再次显示、false 时清理定时器
