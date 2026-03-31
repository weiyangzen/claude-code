# 文件研究文档: src/commands/sandbox-toggle/index.ts

## 场景与职责

该文件是 Claude Code CLI 中 `/sandbox` 命令的入口定义文件，负责定义沙盒切换命令的元数据和基础配置。它是命令系统的声明层，将用户输入的 `/sandbox` 命令映射到实际的命令处理器。

**核心职责：**
1. 定义命令的显示名称、描述和可见性规则
2. 根据当前沙盒状态动态生成命令描述（包括图标、状态文本）
3. 控制命令在何种条件下对用户可见
4. 声明命令类型和延迟加载的实现模块

## 功能点目的

### 1. 动态命令描述生成
命令的 `description` 是一个 getter，实时反映当前沙盒状态：
- **图标指示**：使用 `figures.warning` 表示依赖缺失，否则使用 `figures.tick`（启用）或 `figures.circle`（禁用）
- **状态文本**：显示 "sandbox enabled"、"sandbox disabled"、"sandbox enabled (auto-allow)" 等状态
- **附加信息**：显示 "fallback allowed"（允许非沙盒回退）和 "(managed)"（策略管理）标记

### 2. 命令可见性控制
通过 `isHidden` getter 控制命令是否对用户可见：
- 当平台不支持沙盒时隐藏（`!SandboxManager.isSupportedPlatform()`）
- 当平台不在启用列表中时隐藏（`!SandboxManager.isPlatformInEnabledList()`）

### 3. 参数提示
`argumentHint: 'exclude "command pattern"'` 提示用户可以使用的子命令参数格式。

### 4. 立即执行标记
`immediate: true` 表示该命令会立即执行，不需要等待停止点（绕过命令队列）。

## 具体技术实现

### 数据结构
```typescript
interface Command {
  name: string                    // 命令名称 'sandbox'
  description: string             // 动态生成的描述文本
  argumentHint?: string           // 参数提示
  isHidden?: boolean              // 是否隐藏
  immediate?: boolean             // 是否立即执行
  type: 'local-jsx'               // 命令类型
  load: () => Promise<...>        // 延迟加载实现
}
```

### 关键流程
1. **命令注册**：该文件导出的命令对象被注册到命令系统中
2. **描述渲染**：当用户输入 `/` 或查看帮助时，调用 `description` getter 获取当前状态
3. **可见性检查**：调用 `isHidden` getter 决定是否显示该命令
4. **执行委托**：通过 `load()` 函数延迟加载 `sandbox-toggle.tsx` 中的实际实现

### 依赖的外部模块
- `figures`: 提供终端图标（✓、○、⚠ 等）
- `../../commands.js`: 提供 `Command` 类型定义
- `../../utils/sandbox/sandbox-adapter.js`: 提供 `SandboxManager` 用于状态查询

## 关键代码路径与文件引用

### 当前文件路径
```
src/commands/sandbox-toggle/index.ts
```

### 直接依赖文件
| 文件路径 | 用途 |
|---------|------|
| `src/commands.js` | `Command` 类型定义 |
| `src/utils/sandbox/sandbox-adapter.ts` | `SandboxManager` 状态管理 |
| `figures` (npm) | 终端图标 |

### 调用链
```
用户输入 /sandbox
    ↓
命令系统查找命令定义 (index.ts)
    ↓
调用 description getter 显示状态
    ↓
执行时调用 load() 加载 sandbox-toggle.tsx
    ↓
渲染 SandboxSettings 组件
```

## 依赖与外部交互

### 运行时依赖
- **SandboxManager**: 提供以下状态查询方法：
  - `isSandboxingEnabled()`: 沙盒是否启用
  - `isAutoAllowBashIfSandboxedEnabled()`: 是否自动允许沙盒内的 bash 命令
  - `areUnsandboxedCommandsAllowed()`: 是否允许非沙盒命令
  - `areSandboxSettingsLockedByPolicy()`: 设置是否被策略锁定
  - `checkDependencies()`: 检查依赖状态
  - `isSupportedPlatform()`: 检查平台支持
  - `isPlatformInEnabledList()`: 检查平台是否在启用列表

### 命令系统集成
- 符合 `Command` 接口规范
- 使用 `type: 'local-jsx'` 表示这是一个本地 JSX 命令
- 使用 `immediate: true` 绕过命令队列

## 风险、边界与改进建议

### 潜在风险
1. **状态同步延迟**: `description` getter 每次被调用时都会实时查询状态，如果在命令执行过程中状态变化，可能导致显示不一致
2. **依赖循环**: 如果 `SandboxManager` 的实现发生变化，可能影响命令的可见性判断

### 边界情况
1. **平台不支持**: 在 Windows 或 WSL1 上命令会被隐藏
2. **依赖缺失**: 即使平台支持，如果缺少 bubblewrap/ripgrep 等依赖，也会显示警告图标
3. **策略锁定**: 当沙盒设置被 policySettings 锁定时，描述会显示 "(managed)"

### 改进建议
1. **缓存优化**: 考虑在状态未变化时缓存描述结果，减少重复计算
2. **更详细的错误提示**: 当依赖缺失时，可以在描述中提供更多安装指导
3. **国际化支持**: 当前描述文本硬编码为英文，可考虑支持多语言

### 相关配置项
- `sandbox.enabled`: 是否启用沙盒
- `sandbox.autoAllowBashIfSandboxed`: 是否自动允许沙盒内的 bash 命令
- `sandbox.allowUnsandboxedCommands`: 是否允许非沙盒命令回退
- `sandbox.enabledPlatforms`: 限制沙盒仅在特定平台启用
