# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 Agent Swarm 模块的核心常量定义文件，集中管理 Swarm 功能中使用的各种标识符、环境变量名和工具函数。它为整个 Swarm 系统提供统一的命名约定和配置接口。

### 核心职责
1. **定义核心标识符**: 团队领导、会话名称、窗口名称等关键标识
2. **环境变量定义**: 定义用于控制队友行为的 CLAUDE_CODE_* 环境变量
3. **工具命令常量**: tmux 等外部工具的命令常量
4. **辅助函数**: 提供动态生成标识符的函数（如 socket 名称）

## 功能点目的

### 1. 核心角色标识符
定义 Swarm 架构中的关键角色和会话名称：
- `TEAM_LEAD_NAME = 'team-lead'`: 团队领导（主 Agent）的标识名称
- `SWARM_SESSION_NAME = 'claude-swarm'`: Swarm 会话的名称
- `SWARM_VIEW_WINDOW_NAME = 'swarm-view'`: Swarm 视图窗口名称
- `HIDDEN_SESSION_NAME = 'claude-hidden'`: 隐藏会话名称（用于隐藏面板）

### 2. 外部工具常量
- `TMUX_COMMAND = 'tmux'`: tmux 命令名称

### 3. Socket 名称生成
```typescript
export function getSwarmSocketName(): string {
  return `claude-swarm-${process.pid}`
}
```
为外部 Swarm 会话生成唯一的 socket 名称，使用 PID 确保多 Claude 实例不冲突。

### 4. 环境变量定义

#### `TEAMMATE_COMMAND_ENV_VAR = 'CLAUDE_CODE_TEAMMATE_COMMAND'`
- 用于覆盖生成队友实例的命令
- 默认使用 `process.execPath`（当前 Claude 二进制文件）
- 允许为不同环境或测试场景自定义

#### `TEAMMATE_COLOR_ENV_VAR = 'CLAUDE_CODE_AGENT_COLOR'`
- 在生成的队友上设置，指示其分配的颜色
- 用于彩色输出和面板识别

#### `PLAN_MODE_REQUIRED_ENV_VAR = 'CLAUDE_CODE_PLAN_MODE_REQUIRED'`
- 当设置为 'true' 时，队友必须在实施前进入计划模式并获得批准
- 用于严格控制队友的执行流程

## 具体技术实现

### 代码结构
```typescript
// 核心标识符常量
export const TEAM_LEAD_NAME = 'team-lead'
export const SWARM_SESSION_NAME = 'claude-swarm'
export const SWARM_VIEW_WINDOW_NAME = 'swarm-view'
export const TMUX_COMMAND = 'tmux'
export const HIDDEN_SESSION_NAME = 'claude-hidden'

// 动态标识符生成
export function getSwarmSocketName(): string {
  return `claude-swarm-${process.pid}`
}

// 环境变量名常量
export const TEAMMATE_COMMAND_ENV_VAR = 'CLAUDE_CODE_TEAMMATE_COMMAND'
export const TEAMMATE_COLOR_ENV_VAR = 'CLAUDE_CODE_AGENT_COLOR'
export const PLAN_MODE_REQUIRED_ENV_VAR = 'CLAUDE_CODE_PLAN_MODE_REQUIRED'
```

### Socket 名称设计
`getSwarmSocketName()` 函数的设计考虑：
1. **隔离性**: 使用单独的 socket 将 Swarm 操作与用户的 tmux 会话隔离
2. **唯一性**: 包含 PID 确保多个 Claude 实例不会冲突
3. **可预测性**: 基于进程 ID 生成，便于调试和日志追踪

## 关键代码路径与文件引用

### 本文件导出
| 导出项 | 类型 | 说明 |
|-------|------|------|
| `TEAM_LEAD_NAME` | string | 团队领导标识 |
| `SWARM_SESSION_NAME` | string | Swarm 会话名 |
| `SWARM_VIEW_WINDOW_NAME` | string | Swarm 视图窗口名 |
| `TMUX_COMMAND` | string | tmux 命令 |
| `HIDDEN_SESSION_NAME` | string | 隐藏会话名 |
| `getSwarmSocketName` | function | Socket 名称生成器 |
| `TEAMMATE_COMMAND_ENV_VAR` | string | 队友命令环境变量名 |
| `TEAMMATE_COLOR_ENV_VAR` | string | 队友颜色环境变量名 |
| `PLAN_MODE_REQUIRED_ENV_VAR` | string | 计划模式环境变量名 |

### 被引用情况

通过 Grep 搜索，这些常量被以下文件引用：

#### `TEAM_LEAD_NAME` 引用
- `src/utils/swarm/inProcessRunner.ts`: 用于向领导发送消息
- `src/utils/teammateMailbox.ts`: 用于识别领导消息
- `src/utils/swarm/backends/TmuxBackend.ts`: 用于会话管理

#### `SWARM_SESSION_NAME` 引用
- `src/utils/swarm/backends/TmuxBackend.ts`: 创建 Swarm 会话
- `src/utils/swarm/backends/ITermBackend.ts`: iTerm2 会话管理

#### `getSwarmSocketName` 引用
- `src/utils/swarm/backends/TmuxBackend.ts`: 生成 tmux socket 路径

#### 环境变量引用
- `src/utils/swarm/spawnInProcess.ts`: 读取环境变量配置
- `src/utils/teammate.ts`: 获取队友颜色等属性

## 依赖与外部交互

### 运行时依赖
- `process.pid`: Node.js 进程 ID，用于生成唯一 socket 名称

### 外部系统交互
1. **tmux**: 通过 `TMUX_COMMAND` 和 socket 名称与 tmux 交互
2. **iTerm2**: 通过 `it2` CLI 与会话名称交互
3. **子进程**: 通过环境变量向生成的队友传递配置

### 环境变量使用场景

| 环境变量 | 设置者 | 读取者 | 用途 |
|---------|-------|-------|------|
| `CLAUDE_CODE_TEAMMATE_COMMAND` | 用户/系统 | `spawnInProcess.ts` | 自定义队友启动命令 |
| `CLAUDE_CODE_AGENT_COLOR` | Swarm 系统 | 队友进程 | UI 颜色识别 |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | 用户/配置 | 队友进程 | 强制计划模式 |

## 风险、边界与改进建议

### 风险点

1. **硬编码标识符冲突**
   - `team-lead`、`claude-swarm` 等名称是硬编码的
   - 如果用户系统中有同名 tmux 会话，可能产生冲突
   - **缓解**: `getSwarmSocketName()` 使用 PID 确保唯一性

2. **环境变量命名空间**
   - 使用 `CLAUDE_CODE_*` 前缀减少与其他应用冲突的可能性
   - 但仍可能与其他 Claude 相关工具冲突

3. **Socket 文件残留**
   - 如果 Claude 异常退出，socket 文件可能残留
   - 需要清理机制或超时处理

### 边界情况

1. **PID 重用**
   - 系统 PID 可能在 Claude 重启后被重用
   - 可能导致短暂的 socket 名称冲突（概率极低）

2. **长会话名称**
   - tmux 对会话名称长度有限制（通常 255 字符）
   - 当前设计 `claude-swarm-${PID}` 通常不会超限

3. **多用户系统**
   - 在多用户共享系统中，不同用户的 socket 可能位于不同目录
   - 需要确保权限隔离

### 改进建议

1. **可配置标识符**
   ```typescript
   // 建议添加配置选项
   export function getSwarmSessionName(customPrefix?: string): string {
     return `${customPrefix || 'claude'}-swarm`
   }
   ```

2. **Socket 清理机制**
   - 在启动时检查并清理过期的 socket 文件
   - 使用文件锁或时间戳验证 socket 有效性

3. **环境变量验证**
   - 添加环境变量值的有效性验证
   - 例如验证 `CLAUDE_CODE_AGENT_COLOR` 是有效的颜色名称

4. **常量文档生成**
   - 自动生成常量使用文档
   - 便于开发者了解各常量的用途和约束

5. **版本化标识符**
   - 考虑在会话名称中加入版本信息
   - 便于处理不同版本 Claude 之间的兼容性问题
   ```typescript
   export const SWARM_SESSION_NAME = `claude-swarm-v${CLAUDE_VERSION}`
   ```
