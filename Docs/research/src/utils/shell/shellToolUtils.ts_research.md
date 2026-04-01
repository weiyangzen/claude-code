# Research: `src/utils/shell/shellToolUtils.ts`

## 场景与职责

`shellToolUtils.ts` 是 Claude Code **Shell 工具层的统一门控与常量模块**。它不承担具体命令解析或执行逻辑，而是作为整个代码库中判断 "PowerShell 工具是否可用" 以及 "哪些工具属于 Shell 工具" 的**唯一权威来源（single source of truth）**。

其核心职责包括：

1. **定义 Shell 工具名称集合**：通过 `SHELL_TOOL_NAMES` 将 `Bash` 和 `PowerShell` 统一标识为 Shell 类工具，供上下文压缩、统计分类、权限常量、 streamlined 输出等子系统引用。
2. **PowerShell 工具的运行时门控**：`isPowerShellToolEnabled()` 实现了跨模块一致的启用策略，确保工具列表可见性、用户输入路由、Skill frontmatter 路由三端使用同一套判断逻辑。
3. **平台与身份差异化默认策略**：Windows 平台专属；Ant 内部构建默认开启（opt-out），外部构建默认关闭（opt-in）。

该模块的设计意图是**消除决策碎片化**——任何需要判断 "这条命令该不该走 PowerShell" 或 "这个工具算不算 Shell" 的代码，都应引用此处，而不是各自硬编码。

## 功能点目的

### 1. `SHELL_TOOL_NAMES: string[]`

- **目的**：将 `BashTool` 与 `PowerShellTool` 的名称聚合为一个数组，供下游做"Shell 工具"类别的批量判断。
- **设计细节**：
  - 显式从 `toolName.ts` 导入，而非硬编码字符串，避免改名时遗漏。
  - 使用 `string[]` 而非 `readonly string[]`，是为了兼容部分下游 API（如 `Set` 构造、`Array.prototype.includes` 等）的签名要求。

### 2. `isPowerShellToolEnabled(): boolean`

- **目的**：在运行时决定 PowerShell 工具是否可被暴露、路由和执行。
- **平台门控**：
  - 若 `getPlatform() !== 'windows'`，直接返回 `false`。PowerShellTool 的权限引擎使用了 Win32 专属的路径规范化，在非 Windows 平台（包括 WSL/Linux/macOS）上不支持。
- **环境变量门控**：
  - **Ant 用户**（`process.env.USER_TYPE === 'ant'`）：默认开启。仅当 `CLAUDE_CODE_USE_POWERSHELL_TOOL` 被显式定义为 falsy 值（`0/false/no/off`）时才关闭。
  - **外部用户**：默认关闭。仅当 `CLAUDE_CODE_USE_POWERSHELL_TOOL` 被显式定义为 truthy 值（`1/true/yes/on`）时才开启。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

```ts
export const SHELL_TOOL_NAMES: string[] = [
  BASH_TOOL_NAME,      // 'Bash'
  POWERSHELL_TOOL_NAME // 'PowerShell'
]
```

### `isPowerShellToolEnabled` 决策流程

```
getPlatform() === 'windows'?
  ├─ No  → false
  └─ Yes → process.env.USER_TYPE === 'ant'?
             ├─ Yes → !isEnvDefinedFalsy(process.env.CLAUDE_CODE_USE_POWERSHELL_TOOL)
             └─ No  → isEnvTruthy(process.env.CLAUDE_CODE_USE_POWERSHELL_TOOL)
```

### 环境变量判断工具函数

- `isEnvTruthy(value)`：当值为 `1/true/yes/on`（大小写不敏感）时返回 `true`。
- `isEnvDefinedFalsy(value)`：当值已定义且为 `0/false/no/off` 时返回 `true`。
- 两者均对 `undefined`、空字符串、`boolean` 类型做了防御性处理。

## 关键代码路径与文件引用

### 直接依赖（被 import）

| 文件 | 导入符号 | 作用 |
|------|---------|------|
| `src/tools/BashTool/toolName.ts` | `BASH_TOOL_NAME` | 获取 Bash 工具的规范名称 `'Bash'` |
| `src/tools/PowerShellTool/toolName.ts` | `POWERSHELL_TOOL_NAME` | 获取 PowerShell 工具的规范名称 `'PowerShell'` |
| `src/utils/envUtils.ts` | `isEnvDefinedFalsy`, `isEnvTruthy` | 环境变量的标准化布尔判断 |
| `src/utils/platform.ts` | `getPlatform` | 获取当前运行平台（memoized） |

### 下游调用方

| 调用方文件 | 调用符号 | 场景 |
|-----------|---------|------|
| `src/tools.ts` | `isPowerShellToolEnabled` | `getPowerShellTool()` 懒加载门控：若未启用则返回 `null`，避免加载 ~300KB 的 PowerShell 模块 |
| `src/constants/tools.ts` | `SHELL_TOOL_NAMES` | `ASYNC_AGENT_ALLOWED_TOOLS` 常量集合中展开 Shell 工具名称 |
| `src/utils/processUserInput/processBashCommand.tsx` | `isPowerShellToolEnabled` | `!` 输入框的 Shell 路由：结合 `resolveDefaultShell()` 决定使用 BashTool 还是 PowerShellTool |
| `src/utils/promptShellExecution.ts` | `isPowerShellToolEnabled` | Skill markdown frontmatter 中 `shell: powershell` 的路由门控 |
| `src/services/compact/microCompact.ts` | `SHELL_TOOL_NAMES` | `COMPACTABLE_TOOLS` 集合：Shell 工具的结果可被 microcompact 清除 |
| `src/services/compact/apiMicrocompact.ts` | `SHELL_TOOL_NAMES` | `TOOLS_CLEARABLE_RESULTS`：API 原生上下文管理策略中的可清除结果工具 |
| `src/utils/stats.ts` | `SHELL_TOOL_NAMES` | `extractShotCountFromMessages` 中识别 Shell 工具调用以提取 shot count |
| `src/utils/streamlinedTransform.ts` | `SHELL_TOOL_NAMES` | `COMMAND_TOOLS` 分类：streamlined 输出模式中将 Shell 工具归类为 "ran N commands" |

### 调用链示例

**工具列表装配链：**
```
src/tools.ts:getAllBaseTools()
  └── getPowerShellTool()
        └── isPowerShellToolEnabled()
              └── getPlatform() + env checks
```

**用户输入路由链：**
```
src/utils/processUserInput/processBashCommand.tsx:processBashCommand()
  └── isPowerShellToolEnabled() && resolveDefaultShell() === 'powershell'
        └── 选择 PowerShellTool.call() 或 BashTool.call()
```

**Skill Shell 执行链：**
```
src/utils/promptShellExecution.ts:executeShellCommandsInPrompt()
  └── shell === 'powershell' && isPowerShellToolEnabled()
        └── getPowerShellTool() / BashTool
```

## 依赖与外部交互

### 平台检测

`getPlatform()` 来自 `src/utils/platform.ts`，实现逻辑：
- `process.platform === 'darwin'` → `macos`
- `process.platform === 'win32'` → `windows`
- `process.platform === 'linux'` → 读取 `/proc/version` 判断是否为 `wsl`，否则 `linux`
- 其他 → `unknown`

该函数被 `memoize` 缓存，进程生命周期内只计算一次。

### 环境变量规范

- `CLAUDE_CODE_USE_POWERSHELL_TOOL`：用户显式控制 PowerShell 工具的开关。
- `USER_TYPE`：构建时注入（`ant` 或 `external`），决定默认策略方向。

### 与 PowerShellTool 的懒加载配合

`src/tools.ts` 中不直接静态 import `PowerShellTool`，而是通过 getter 延迟 `require`：

```ts
const getPowerShellTool = () => {
  if (!isPowerShellToolEnabled()) return null
  return require('./tools/PowerShellTool/PowerShellTool.js').PowerShellTool
}
```

这确保了：
- 非 Windows 用户完全不加载 PowerShell 相关代码（节省 ~300KB）。
- 外部未 opt-in 用户同样不加载。
- `shellToolUtils.ts` 本身只做门控判断，不引入 PowerShell 模块的重量级依赖。

## 风险、边界与改进建议

### 风险

1. **平台判断的单一性**
   `getPlatform()` 对 WSL 返回 `'wsl'` 而非 `'windows'`。由于 `isPowerShellToolEnabled()` 严格判断 `getPlatform() !== 'windows'`，WSL 用户即使安装了 `pwsh` 也无法通过 PowerShellTool 执行命令。这是设计意图（WSL 中 Bash 已足够），但文档化不足可能导致用户困惑。

2. **环境变量大小写敏感问题**
   `isEnvTruthy` 和 `isEnvDefinedFalsy` 会先将值 `toLowerCase().trim()` 再比较，因此 `TRUE`、`True`、`  true  ` 均可识别。但某些 Windows 用户可能习惯 `Claude_Code_Use_Powershell_Tool=1`（大小写混合），这在逻辑上没问题，但环境变量名本身在 *nix 上是区分大小写的，设置时需注意。

3. **`SHELL_TOOL_NAMES` 的易变性**
   数组类型为 `string[]` 而非 `readonly string[]`，理论上下游可能意外 `push` 新元素导致全局状态污染。目前未发现此类调用，但类型层面未做不可变保证。

4. **Ant/External 策略的硬编码**
   默认开启/关闭策略直接写死在函数中，若未来需要针对其他用户类型（如 `enterprise`、`beta`）做差异化策略，需要修改本文件。

### 边界

- **仅做启用判断，不做能力检测**：`isPowerShellToolEnabled()` 不检查系统中是否实际安装了 `pwsh.exe` 或 `powershell.exe`。实际的可执行性检测在 `PowerShellTool` 内部完成（如 `resolveDefaultShell.ts` 或 `shouldUseSandbox.ts`）。
- **不处理运行时切换**：虽然 `getPlatform()` 被 memoize，但 `isPowerShellToolEnabled()` 本身未被 memoize。这意味着每次调用都会重新读取 `process.env`，在极少数测试场景（如测试框架动态修改 `process.env`）中行为正确，但在热路径中略有重复计算开销。
- **与 `resolveDefaultShell()` 的协同**：`processBashCommand.tsx` 中需要同时满足 `isPowerShellToolEnabled() && resolveDefaultShell() === 'powershell'` 才会路由到 PowerShell。仅开启门控但默认 Shell 仍为 bash 时，用户输入的 `!` 命令仍走 BashTool。

### 改进建议

1. **类型安全：将 `SHELL_TOOL_NAMES` 标记为 readonly**
   ```ts
   export const SHELL_TOOL_NAMES: readonly string[] = [...]
   ```
   若下游 API 兼容，应提升不可变性保证，防止意外修改。

2. **缓存 `isPowerShellToolEnabled` 结果**
   该函数在 `tools.ts`、`processBashCommand.tsx`、`promptShellExecution.ts` 等多处被调用，且其依赖的 `getPlatform()` 和 `process.env` 在单次请求内不会变化。可用模块级变量缓存结果，减少重复平台判断和 env 解析。

3. **策略配置外部化**
   将 Ant/External 的默认策略提取到配置对象或 feature flag 中：
   ```ts
   const DEFAULTS = {
     ant: true,
     external: false,
   }
   ```
   便于未来扩展新用户类型时无需修改核心门控逻辑。

4. **增加测试覆盖**
   仓库中未找到针对 `shellToolUtils.ts` 的单元测试。建议补充：
   - 各平台（windows/mac/linux/wsl）下的返回行为；
   - `USER_TYPE=ant` 时 env 各种取值（undefined、`0`、`1`、`true`、`false`）的边界；
   - `USER_TYPE=external` 时的对称边界；
   - `SHELL_TOOL_NAMES` 的内容稳定性断言。

5. **文档化 WSL 不支持 PowerShellTool 的设计决策**
   在设置面板或 CLI help 中明确告知 WSL 用户：即使设置了 `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`，由于平台限制，PowerShellTool 仍不可用。减少支持工单和用户困惑。
