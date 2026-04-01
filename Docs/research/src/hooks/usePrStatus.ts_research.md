# Research: src/hooks/usePrStatus.ts

## 场景与职责

`usePrStatus` 是一个用于在 REPL 底部状态栏显示**当前分支关联的 GitHub Pull Request 状态**的 Hook。它通过周期性调用 `gh pr view` 获取 PR 编号、URL 和评审状态（approved / changes_requested / pending / draft 等），并将这些信息展示在 `PromptInputFooterLeftSide` 组件的 `PrBadge` 中。

该 Hook 的设计充分考虑了性能与资源占用：
- 仅在用户活跃时轮询（60 秒间隔）。
- 当用户 60 分钟无操作时自动停止轮询，不保留任何定时器。
- 若单次 `gh` 调用超过 4 秒，则永久禁用该会话的轮询，避免慢网络/大仓库导致的持续卡顿。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **周期性轮询** | 每 `POLL_INTERVAL_MS`（60,000ms）调用 `fetchPrStatus()`，保持 PR 状态相对实时。 |
| **空闲自动停止** | `IDLE_STOP_MS`（60 分钟）无交互后停止轮询，避免后台无意义地执行 `gh` 子进程。 |
| **慢查询熔断** | `SLOW_GH_THRESHOLD_MS`（4,000ms）若单次 `gh` 超时，则永久禁用（`disabledRef.current = true`），防止慢网络环境持续拖累性能。 |
| **turn 边界对齐** | effect 重跑时（`isLoading` 变化代表一轮对话开始/结束），根据 `lastFetchRef` 计算剩余时间，确保 turn 边界不会导致 `gh` 被频繁触发。 |
| **状态去重** | 仅当 PR 编号或评审状态发生变化时才调用 `setPrStatus`，减少不必要的重渲染。 |

## 具体技术实现

### 轮询控制逻辑

```ts
useEffect(() => {
  if (!enabled) return
  if (disabledRef.current) return

  let cancelled = false
  let lastSeenInteractionTime = -1
  let lastActivityTimestamp = Date.now()

  async function poll() {
    if (cancelled) return

    const currentInteractionTime = getLastInteractionTime()
    if (lastSeenInteractionTime !== currentInteractionTime) {
      lastSeenInteractionTime = currentInteractionTime
      lastActivityTimestamp = Date.now()
    } else if (Date.now() - lastActivityTimestamp >= IDLE_STOP_MS) {
      return // 停止轮询
    }

    const start = Date.now()
    const result = await fetchPrStatus()
    if (cancelled) return
    lastFetchRef.current = start

    setPrStatus(prev => {
      const newNumber = result?.number ?? null
      const newReviewState = result?.reviewState ?? null
      if (prev.number === newNumber && prev.reviewState === newReviewState) {
        return prev
      }
      return { number: newNumber, url: result?.url ?? null, reviewState: newReviewState, lastUpdated: Date.now() }
    })

    if (Date.now() - start > SLOW_GH_THRESHOLD_MS) {
      disabledRef.current = true
      return
    }

    if (!cancelled) {
      timeoutRef.current = setTimeout(poll, POLL_INTERVAL_MS)
    }
  }

  const elapsed = Date.now() - lastFetchRef.current
  if (elapsed >= POLL_INTERVAL_MS) {
    void poll()
  } else {
    timeoutRef.current = setTimeout(poll, POLL_INTERVAL_MS - elapsed)
  }

  return () => { cancelled = true; clearTimeout(timeoutRef.current) }
}, [isLoading, enabled])
```

### 关键设计细节

- **`isLoading` 作为 effect 依赖**：对话轮次开始/结束时 `isLoading` 会翻转，这会重启 effect。设计意图是利用 turn 边界作为自然的"用户仍在活跃"信号，同时避免在对话进行中也触发 `gh` 调用。
- **`lastFetchRef` 对齐机制**：effect 重启时计算距离上次成功抓取过去了多久。若已超过 60s 则立即 poll，否则等待剩余时间。这保证了一个 turn 内不会多次 spawn `gh`。
- **refs 的使用**：`disabledRef`、`lastFetchRef`、`timeoutRef` 均为 refs，避免它们的变化触发额外的重渲染。

### 数据结构

```ts
export type PrStatusState = {
  number: number | null
  url: string | null
  reviewState: PrReviewState | null
  lastUpdated: number
}

const INITIAL_STATE: PrStatusState = {
  number: null,
  url: null,
  reviewState: null,
  lastUpdated: 0,
}
```

## 关键代码路径与文件引用

| 文件 | 作用 |
|------|------|
| `src/hooks/usePrStatus.ts` | 本 Hook：轮询控制、状态管理、熔断逻辑。 |
| `src/components/PromptInput/PromptInputFooterLeftSide.tsx` | 调用方：在底部状态栏根据 `prStatus` 渲染 `PrBadge`。 |
| `src/utils/ghPrStatus.ts` | `fetchPrStatus` 实现：调用 `gh pr view --json number,url,reviewDecision,isDraft,headRefName,state` 并解析。 |
| `src/bootstrap/state.js` | `getLastInteractionTime()` 来源：记录最后一次用户交互时间戳。 |
| `src/components/PrBadge.tsx` | UI 组件：将 `PrReviewState` 渲染为带颜色和图标的徽章。 |

## 依赖与外部交互

### 运行时依赖
- **React**：`useEffect`、`useRef`、`useState`。
- **GitHub CLI (`gh`)**：`fetchPrStatus` 依赖本地安装的 `gh` 命令行工具。
- **子进程执行**：`execFileNoThrow`（在 `ghPrStatus.ts` 中）以 5 秒超时执行 `gh`。

### 与调用方的契约
- `usePrStatus(isLoading, enabled)`：
  - `isLoading: boolean`：驱动 effect 重跑，对齐轮询时机。
  - `enabled: boolean`（默认 true）：总开关，传 `false` 可完全跳过轮询（满足 Rules of Hooks 的无条件调用要求）。
- 返回 `PrStatusState`，消费方（`PromptInputFooterLeftSide`）根据 `number !== null && reviewState !== null` 决定是否渲染徽章。

## 风险、边界与改进建议

### 风险与边界
1. **`gh` 未安装或不在 PATH**：`fetchPrStatus` 在 `ghPrStatus.ts` 中通过 `execFileNoThrow` 执行，若 `gh` 不存在则返回 `null`，Hook 会每 60s 尝试一次但始终无结果。虽然不会崩溃，但属于无意义开销。
2. **非 GitHub 仓库**：`gh pr view` 会失败，同样返回 `null`，行为与上一条一致。
3. **默认分支过滤**：`ghPrStatus.ts` 中显式过滤了默认分支（main/master）以及 headRefName 为默认分支的 PR，避免显示已合并/无关的 PR。这在某些工作流（如 PR 从 main 到 release 分支）下可能遗漏有效 PR。
4. **merged/closed PR 被过滤**：`gh pr view` 可能返回分支最近关联的已关闭 PR，Hook 主动过滤掉 `MERGED` 和 `CLOSED` 状态，只显示开放中的 PR。
5. **慢查询熔断是 session-scoped**：`disabledRef` 是组件生命周期内的 ref，页面刷新或 REPL 重挂载后会重置，不会持久化到磁盘。

### 改进建议
1. **缓存 `gh` 可用性检查结果**：在会话内首次检测到 `gh` 不存在后，可设置一个更长的重试间隔（如 5 分钟），避免持续无意义轮询。
2. **支持 GitLab / 其他 Git 托管平台**：当前硬编码依赖 `gh` CLI，未来若支持其他平台，需要抽象 `PrProvider` 接口。
3. **持久化慢查询熔断状态**：将 `disabledRef` 的状态存入 `AppState` 或全局配置，避免 REPL 重挂载后再次触发慢查询。
4. **增加 PR 状态变更的音效/震动提示**：对于长时间运行的会话，PR 从 `pending` 变为 `approved` 时可增加 subtle 的终端 bell 或通知，提升用户感知。
