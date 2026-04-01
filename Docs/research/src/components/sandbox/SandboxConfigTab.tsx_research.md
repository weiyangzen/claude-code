# SandboxConfigTab.tsx 深度研究文档

## 1. 场景与职责

**SandboxConfigTab** 是 Claude Code CLI 的 `/sandbox` 命令界面中的一个子组件，负责展示当前沙箱配置的详细信息。它是沙箱设置界面的"Config"标签页内容，向用户展示沙箱的实时配置状态，包括：

- 沙箱是否启用
- 依赖检查警告
- 排除命令列表
- 文件系统读写限制
- 网络访问限制
- Unix Socket 允许列表
- Linux Glob 模式警告

### 使用场景

1. **用户执行 `/sandbox` 命令后**：用户可以通过 Config 标签页查看当前沙箱的详细配置
2. **故障排查时**：当沙箱行为不符合预期时，用户可以检查实际生效的配置
3. **配置验证时**：用户修改设置后，可以通过此界面确认配置已正确应用

---

## 2. 功能点目的

### 2.1 沙箱状态展示

| 功能 | 目的 |
|------|------|
| `isEnabled` 检查 | 显示沙箱是否实际启用（受平台支持、依赖可用性、设置影响） |
| 依赖警告展示 | 当 seccomp 等可选依赖缺失时给出警告 |
| 未启用提示 | 当沙箱未启用时，明确告知用户状态 |

### 2.2 配置详情展示

| 配置项 | 数据来源 | 展示内容 |
|--------|----------|----------|
| Excluded Commands | `SandboxManager.getExcludedCommands()` | 不经过沙箱执行的命令列表 |
| Filesystem Read Restrictions | `SandboxManager.getFsReadConfig()` | 禁止读取的路径、允许例外 |
| Filesystem Write Restrictions | `SandboxManager.getFsWriteConfig()` | 允许写入的路径、禁止例外 |
| Network Restrictions | `SandboxManager.getNetworkRestrictionConfig()` | 允许/禁止访问的主机 |
| Unix Sockets | `SandboxManager.getAllowUnixSockets()` | 允许的 Unix Socket 路径 |
| Glob Pattern Warnings | `SandboxManager.getLinuxGlobPatternWarnings()` | Linux 上不完全支持的 Glob 模式 |

---

## 3. 具体技术实现

### 3.1 组件架构

```tsx
export function SandboxConfigTab(): React.ReactNode {
  const isEnabled = SandboxManager.isSandboxingEnabled()
  
  // 依赖检查警告
  const depCheck = SandboxManager.checkDependencies()
  const warningsNote = depCheck.warnings.length > 0 ? (...) : null
  
  if (!isEnabled) {
    return <沙箱未启用提示>{warningsNote}</沙箱未启用提示>
  }
  
  // 获取各项配置
  const fsReadConfig = SandboxManager.getFsReadConfig()
  const fsWriteConfig = SandboxManager.getFsWriteConfig()
  const networkConfig = SandboxManager.getNetworkRestrictionConfig()
  const allowUnixSockets = SandboxManager.getAllowUnixSockets()
  const excludedCommands = SandboxManager.getExcludedCommands()
  const globPatternWarnings = SandboxManager.getLinuxGlobPatternWarnings()
  
  return <完整配置展示界面 />
}
```

### 3.2 关键数据结构

**FsReadRestrictionConfig**（来自 @anthropic-ai/sandbox-runtime）：
```typescript
{
  denyOnly: string[]      // 禁止读取的路径
  allowWithinDeny: string[]  // 在禁止区域内的允许例外
}
```

**FsWriteRestrictionConfig**：
```typescript
{
  allowOnly: string[]     // 只允许写入这些路径
  denyWithinAllow: string[]  // 在允许区域内的禁止例外
}
```

**NetworkRestrictionConfig**：
```typescript
{
  allowedHosts?: string[]  // 允许访问的主机
  deniedHosts?: string[]   // 禁止访问的主机
}
```

### 3.3 React Compiler 优化

组件使用了 React Compiler（通过 `_c` 函数），自动进行记忆化优化：

```tsx
const $ = _c(3)  // 3 个记忆化槽位

// 使用 Symbol.for("react.memo_cache_sentinel") 作为缓存标记
if ($[0] === Symbol.for("react.memo_cache_sentinel")) {
  // 首次渲染，计算并缓存
  $[0] = computedValue
} else {
  // 后续渲染，使用缓存
  computedValue = $[0]
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 直接依赖

| 导入 | 路径 | 用途 |
|------|------|------|
| React | 'react' | JSX 运行时 |
| Box, Text | '../../ink.js' | Ink 终端 UI 组件 |
| SandboxManager | '../../utils/sandbox/sandbox-adapter.js' | 沙箱管理器 |
| shouldAllowManagedSandboxDomainsOnly | '../../utils/sandbox/sandbox-adapter.js' | 托管域名限制检查 |

### 4.2 调用链

```
SandboxConfigTab.tsx
  ↓ 调用
SandboxManager (sandbox-adapter.ts)
  ↓ 委托
BaseSandboxManager (@anthropic-ai/sandbox-runtime)
  ↓ 系统调用
macOS: seatbelt / Linux: bubblewrap + seccomp
```

### 4.3 配置读取流程

```
SandboxManager.getFsReadConfig()
  ↓
BaseSandboxManager.getFsReadConfig() (来自 sandbox-runtime)
  ↓
读取运行时配置（由 convertToSandboxRuntimeConfig 生成）
  ↓
从 settings.json 解析的权限规则
```

---

## 5. 依赖与外部交互

### 5.1 外部包依赖

| 包名 | 用途 |
|------|------|
| @anthropic-ai/sandbox-runtime | 底层沙箱运行时，提供 BaseSandboxManager |
| react | React 框架 |
| ink | 终端渲染框架（通过 ../../ink.js） |

### 5.2 内部模块依赖

```
src/components/sandbox/SandboxConfigTab.tsx
├── src/ink.ts                          # Ink 渲染层封装
├── src/utils/sandbox/sandbox-adapter.ts # 沙箱适配器
│   ├── @anthropic-ai/sandbox-runtime   # 外部运行时
│   ├── src/utils/settings/settings.ts  # 设置管理
│   ├── src/utils/platform.ts           # 平台检测
│   └── ...
└── src/components/design-system/       # 设计系统组件（间接）
```

### 5.3 设置系统集成

配置数据最终来源于 `settings.json` 文件，通过以下路径传递：

1. **用户设置**: `~/.claude/settings.json`
2. **项目设置**: `.claude/settings.json`
3. **本地设置**: `.claude/settings.local.json`
4. **策略设置**: 托管设置（managed-settings.json）
5. **标志设置**: CLI 参数传入

设置优先级（高到低）：`policySettings` > `flagSettings` > `localSettings` > `projectSettings` > `userSettings`

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 配置不同步 | 显示的配置可能与实际运行时配置不一致 | 每次打开标签页时重新读取 |
| 敏感信息泄露 | 配置中可能包含敏感路径 | 仅显示当前用户有权限查看的配置 |
| Linux Glob 限制 | bubblewrap 不完全支持复杂 Glob 模式 | 显示警告提示用户 |

### 6.2 边界情况

1. **平台不支持**: 在 Windows 或不支持的平台上，`isSandboxingEnabled()` 返回 false，显示简化界面
2. **依赖缺失**: 当 ripgrep、bubblewrap 等依赖缺失时，显示警告但不阻止界面展示
3. **配置锁定**: 当设置被 policySettings 锁定时，Config 标签页只读展示

### 6.3 改进建议

1. **配置刷新机制**: 当前配置是打开标签页时的一次性快照，建议添加手动刷新按钮
2. **配置编辑**: 可考虑在此界面直接编辑某些配置（如 excludedCommands）
3. **配置验证**: 增加配置冲突检测和提示（如 allow 和 deny 规则冲突）
4. **搜索过滤**: 当配置项较多时，添加搜索/过滤功能
5. **配置导出**: 允许用户将当前生效配置导出为 settings.json 片段

### 6.4 测试要点

- 沙箱启用/禁用状态的正确显示
- 各平台（macOS/Linux/WSL）下的配置展示
- 依赖缺失时的警告显示
- 空配置（无 excludedCommands、无限制规则）的展示
- 长列表的渲染性能

---

## 附录：相关文件索引

| 文件 | 描述 |
|------|------|
| `src/components/sandbox/SandboxSettings.tsx` | 沙箱设置主界面，包含 Tabs 容器 |
| `src/components/sandbox/SandboxDependenciesTab.tsx` | 依赖检查标签页 |
| `src/components/sandbox/SandboxOverridesTab.tsx` | 覆盖设置标签页 |
| `src/utils/sandbox/sandbox-adapter.ts` | 沙箱适配器，核心逻辑 |
| `src/entrypoints/sandboxTypes.ts` | 沙箱配置类型定义 |
| `src/utils/settings/types.ts` | 设置系统类型定义 |
