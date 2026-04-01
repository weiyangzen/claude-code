# standaloneAgent.ts 深度研究

## 场景与职责

`standaloneAgent.ts` 提供**独立 Agent（非 Swarm 团队成员）的上下文访问工具**。当会话属于 Swarm 团队时，这些函数返回 undefined，让 Swarm 上下文优先。

**核心职责：**
1. 获取独立 Agent 的名称（如果设置且非团队成员）
2. 与 Swarm 团队检测集成

**应用场景：**
- 显示 Agent 标识
- 会话恢复时的 Agent 上下文恢复
- 区分独立 Agent 和团队成员

---

## 功能点目的

### 获取独立 Agent 名称
```typescript
export function getStandaloneAgentName(appState: AppState): string | undefined
```

**逻辑：**
1. 检查是否在团队中（通过 `getTeamName()`）
2. 如果在团队中，返回 undefined（让 Swarm 上下文优先）
3. 否则返回 `appState.standaloneAgentContext?.name`

---

## 具体技术实现

### 实现代码
```typescript
export function getStandaloneAgentName(appState: AppState): string | undefined {
  // If in a team (swarm), don't return standalone name
  if (getTeamName()) {
    return undefined
  }
  return appState.standaloneAgentContext?.name
}
```

**设计要点：**
- 使用 `getTeamName()` 与 `isTeammate()` 保持一致
- 简单的优先级规则：Swarm > Standalone

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `getStandaloneAgentName` | 获取独立 Agent 名称 |

### 依赖模块
| 模块 | 用途 |
|------|------|
| `../state/AppState.js` | `AppState` 类型 |
| `./teammate.js` | `getTeamName` |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/screens/ResumeConversation.tsx` | 会话恢复 |
| `src/screens/REPL.tsx` | REPL |
| `src/state/AppStateStore.ts` | 状态存储 |
| `src/commands/color/color.ts` | 颜色命令 |
| `src/components/PromptInput/useSwarmBanner.ts` | Swarm Banner |
| `src/utils/sessionRestore.ts` | 会话恢复 |
| `src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx` | 权限请求 |
| `src/commands/rename/rename.ts` | 重命名命令 |
| `src/commands/clear/conversation.ts` | 清除对话 |

---

## 依赖与外部交互

### 外部依赖
- 无外部依赖

### 内部依赖
| 模块 | 用途 |
|------|------|
| `AppState.js` | 状态类型 |
| `teammate.js` | 团队检测 |

---

## 风险、边界与改进建议

### 已知风险

1. **依赖全局状态**
   - `getTeamName()` 可能依赖全局状态
   - 测试时需要正确设置环境

2. **功能单一**
   - 目前仅支持名称获取
   - 颜色等其他独立 Agent 属性需要额外函数

### 边界情况

| 场景 | 处理 |
|------|------|
| appState 为 null | 类型系统确保传入有效 AppState |
| standaloneAgentContext 未设置 | 返回 undefined |
| 团队名称为空字符串 | `getTeamName()` 返回 falsy，视为非团队 |

### 改进建议

1. **功能扩展**
   - 添加 `getStandaloneAgentColor` 函数
   - 添加 `getStandaloneAgentContext` 返回完整上下文

2. **统一接口**
   - 与 Swarm 上下文统一为 `getAgentDisplayInfo`
   - 自动处理优先级逻辑

3. **类型安全**
   - 添加运行时验证确保 AppState 结构
   - 使用 branded types 区分不同上下文

4. **测试覆盖**
   - 添加单元测试覆盖各种组合
   - 测试团队切换场景
