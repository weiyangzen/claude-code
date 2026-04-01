# getNextPermissionMode.ts 研究文档

## 场景与职责

`getNextPermissionMode.ts` 是 Claude Code 权限系统的模式切换逻辑模块，负责处理用户通过 Shift+Tab 快捷键循环切换权限模式时的下一个模式计算。

核心职责：

1. **模式循环计算**：根据当前模式和可用功能，计算下一个应切换到的模式
2. **Ant 用户特殊处理**：Anthropic 内部用户跳过某些模式，直接进入自动模式
3. **功能标志集成**：根据 `TRANSCRIPT_CLASSIFIER` 功能标志决定自动模式是否可用
4. **上下文转换**：在模式切换时执行必要的上下文清理（如剥离危险权限）

## 功能点目的

### 1. 权限模式循环

支持的模式（按循环顺序）：
- `default` → `acceptEdits` → `plan` → `bypassPermissions` → `auto` → `default`

Ant 用户特殊路径（跳过 acceptEdits 和 plan）：
- `default` → `bypassPermissions` → `auto` → `default`

### 2. 自动模式可用性检查

```typescript
function canCycleToAuto(ctx: ToolPermissionContext): boolean {
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const gateEnabled = isAutoModeGateEnabled()
    const can = !!ctx.isAutoModeAvailable && gateEnabled
    // ... 日志记录
    return can
  }
  return false
}
```

检查条件：
- `TRANSCRIPT_CLASSIFIER` 功能标志启用
- 上下文中 `isAutoModeAvailable` 为 true
- GrowthBook gate `isAutoModeGateEnabled()` 返回 true

### 3. 模式切换逻辑

```typescript
export function getNextPermissionMode(
  toolPermissionContext: ToolPermissionContext,
  _teamContext?: { leadAgentId: string },
): PermissionMode {
  switch (toolPermissionContext.mode) {
    case 'default':
      if (process.env.USER_TYPE === 'ant') {
        if (toolPermissionContext.isBypassPermissionsModeAvailable) {
          return 'bypassPermissions'
        }
        if (canCycleToAuto(toolPermissionContext)) {
          return 'auto'
        }
        return 'default'
      }
      return 'acceptEdits'

    case 'acceptEdits':
      return 'plan'

    case 'plan':
      if (toolPermissionContext.isBypassPermissionsModeAvailable) {
        return 'bypassPermissions'
      }
      if (canCycleToAuto(toolPermissionContext)) {
        return 'auto'
      }
      return 'default'

    case 'bypassPermissions':
      if (canCycleToAuto(toolPermissionContext)) {
        return 'auto'
      }
      return 'default'

    case 'dontAsk':
      return 'default'

    default:
      // 包括 auto 模式，总是回退到 default
      return 'default'
  }
}
```

### 4. 上下文转换

```typescript
export function cyclePermissionMode(
  toolPermissionContext: ToolPermissionContext,
  teamContext?: { leadAgentId: string },
): { nextMode: PermissionMode; context: ToolPermissionContext } {
  const nextMode = getNextPermissionMode(toolPermissionContext, teamContext)
  return {
    nextMode,
    context: transitionPermissionMode(
      toolPermissionContext.mode,
      nextMode,
      toolPermissionContext,
    ),
  }
}
```

`transitionPermissionMode`（来自 `permissionSetup.ts`）处理：
- 进入自动模式时剥离危险权限规则
- 退出自动模式时恢复被剥离的规则
- 进入/退出计划模式时的状态更新

## 具体技术实现

### 自动模式门控检查

```typescript
function canCycleToAuto(ctx: ToolPermissionContext): boolean {
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const gateEnabled = isAutoModeGateEnabled()
    const can = !!ctx.isAutoModeAvailable && gateEnabled
    if (!can) {
      logForDebugging(
        `[auto-mode] canCycleToAuto=false: ctx.isAutoModeAvailable=${ctx.isAutoModeAvailable} isAutoModeGateEnabled=${gateEnabled} reason=${getAutoModeUnavailableReason()}`,
      )
    }
    return can
  }
  return false
}
```

关键点：
- 同时检查缓存的 `isAutoModeAvailable` 和实时的 `isAutoModeGateEnabled()`
- 两者可能不同步（如门控配置中途变更）
- 详细的调试日志帮助诊断问题

### 模式循环状态机

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   default   │────→│ acceptEdits │────→│    plan     │
└──────┬──────┘     └─────────────┘     └──────┬──────┘
       │                                        │
       │    Ant 用户: 跳过 acceptEdits/plan      │
       │    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━     │
       │         default ──→ bypass             │
       │                                        │
       │    无 bypass: default ──→ auto         │
       │                                        │
       └────────────────────────────────────────┘
                          │
                    ┌─────┴─────┐
                    │  bypass   │
                    └─────┬─────┘
                          │
                    ┌─────┴─────┐
                    │   auto    │────→ default (循环)
                    └───────────┘
```

## 关键代码路径与文件引用

### 导入依赖
- `bun:bundle`：`feature()` 功能标志
- `../../Tool.js`：`ToolPermissionContext`
- `../debug.js`：`logForDebugging`
- `./PermissionMode.js`：`PermissionMode` 类型
- `./permissionSetup.js`：`transitionPermissionMode`、`isAutoModeGateEnabled` 等

### 导出函数
| 函数 | 用途 |
|------|------|
| `getNextPermissionMode` | 计算下一个权限模式 |
| `cyclePermissionMode` | 执行模式切换，返回新模式和上下文 |

### 被引用位置
- `src/components/PromptInput/PromptInput.tsx`：Shift+Tab 模式切换
- `src/hooks/toolPermission/handlers/interactiveHandler.ts`：权限处理
- `src/screens/REPL.tsx`：REPL 界面模式切换

## 依赖与外部交互

### 上游依赖
| 模块 | 用途 |
|------|------|
| `bun:bundle` | 功能标志检查 |
| `Tool.js` | 权限上下文类型 |
| `debug.js` | 调试日志 |
| `PermissionMode.js` | 权限模式类型 |
| `permissionSetup.js` | 模式转换逻辑、门控检查 |

### 下游消费者
| 模块 | 用途 |
|------|------|
| `PromptInput.tsx` | Shift+Tab 模式切换 |
| `interactiveHandler.ts` | 交互式权限处理 |
| `REPL.tsx` | REPL 模式切换 |

## 风险、边界与改进建议

### 风险点
1. **模式转换复杂性**：`transitionPermissionMode` 在 `permissionSetup.ts` 中实现，跨文件依赖增加了理解难度
2. **Ant 用户特殊逻辑**：`USER_TYPE === 'ant'` 的特殊路径增加了代码复杂度，需要确保测试覆盖
3. **门控检查竞态**：`isAutoModeGateEnabled()` 是实时检查，可能与缓存的 `isAutoModeAvailable` 不同步

### 边界条件
1. **团队上下文**：`teamContext` 参数当前未使用（以 `_` 前缀标记），为未来团队权限预留
2. **dontAsk 模式**：`dontAsk` 模式不在 UI 循环中，但函数处理了这种情况
3. **auto 模式回退**：`default` case 处理所有其他模式（包括 auto），总是回退到 default

### 改进建议
1. **统一模式定义**：将模式循环顺序定义为配置，而非硬编码在 switch 语句中

```typescript
// 建议：配置化模式循环
const MODE_CYCLE: Record<PermissionMode, PermissionMode[]> = {
  default: ['acceptEdits', 'bypassPermissions', 'auto'],
  acceptEdits: ['plan'],
  plan: ['bypassPermissions', 'auto', 'default'],
  bypassPermissions: ['auto', 'default'],
  auto: ['default'],
  dontAsk: ['default'],
}

// Ant 用户覆盖
const ANT_MODE_CYCLE: Record<PermissionMode, PermissionMode[]> = {
  default: ['bypassPermissions', 'auto'],
  // ...
}
```

2. **团队上下文实现**：完成 `teamContext` 参数的实现，支持团队级别的权限模式

3. **可视化模式循环**：提供 API 返回完整的模式循环路径，供 UI 显示

```typescript
export function getModeCyclePath(
  startMode: PermissionMode,
  toolPermissionContext: ToolPermissionContext,
): PermissionMode[] {
  const path: PermissionMode[] = []
  let currentMode = startMode
  do {
    currentMode = getNextPermissionMode(toolPermissionContext, currentMode)
    path.push(currentMode)
  } while (currentMode !== startMode)
  return path
}
```

4. **模式切换确认**：对于高风险模式切换（如进入 bypassPermissions），考虑添加确认步骤

5. **模式历史**：记录用户的模式切换历史，便于审计和理解用户行为

6. **测试覆盖**：增加单元测试覆盖所有模式转换路径，特别是 Ant 用户路径
