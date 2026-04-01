# shouldUseSandbox.ts 深度研究文档

## 场景与职责

`shouldUseSandbox.ts` 是 Claude Code CLI 中 BashTool 沙箱系统的核心决策模块，负责决定一个 Bash 命令是否应该在沙箱环境中执行。该模块是安全边界的关键组成部分，确保用户配置的沙箱策略得到正确执行。

### 核心职责
1. **沙箱启用决策**：根据系统配置和用户设置，判断是否应该使用沙箱执行命令
2. **排除命令检查**：检查命令是否匹配用户配置的排除列表（excludedCommands）
3. **动态禁用命令**：支持通过 GrowthBook 动态禁用特定命令（仅对 ant 用户生效）
4. **安全边界协调**：与权限系统协作，确保沙箱和权限检查的一致性

## 功能点目的

### 1. 沙箱使用决策 (`shouldUseSandbox`)

```typescript
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean
```

决策流程：
1. 检查沙箱是否全局启用 (`SandboxManager.isSandboxingEnabled()`)
2. 检查是否显式禁用沙箱且策略允许非沙箱命令
3. 检查命令是否为空
4. 检查命令是否包含排除列表中的命令

### 2. 排除命令检查 (`containsExcludedCommand`)

**重要说明**：`excludedCommands` 是面向用户的便利功能，**不是安全边界**。真正的安全控制是沙箱权限系统（会提示用户确认）。

检查层级：
1. **动态配置检查**（仅 ant 用户）：
   - 通过 GrowthBook 获取 `tengu_sandbox_disabled_commands` 配置
   - 支持禁用命令列表和子字符串匹配
   
2. **用户配置检查**：
   - 从用户设置中读取 `sandbox.excludedCommands`
   - 支持三种匹配模式：前缀匹配、精确匹配、通配符匹配

### 3. 命令解析与匹配

使用 `splitCommand_DEPRECATED` 将复合命令（如 `docker ps && curl evil.com`）拆分为单独子命令，防止通过复合命令绕过排除检查。

匹配前的命令预处理：
- 剥离环境变量前缀（仅限白名单中的变量）
- 剥离安全包装器（timeout, time, nice, nohup）
- 使用不动点迭代处理交错模式（如 `timeout 300 FOO=bar bazel run`）

## 具体技术实现

### 关键数据结构

```typescript
type SandboxInput = {
  command?: string           // 要执行的命令
  dangerouslyDisableSandbox?: boolean  // 是否显式禁用沙箱
}

// 权限规则类型（来自 bashPermissions.ts）
type ShellPermissionRule =
  | { type: 'prefix'; prefix: string }
  | { type: 'exact'; command: string }
  | { type: 'wildcard'; pattern: string }
```

### 关键流程

#### 1. 沙箱决策流程

```
shouldUseSandbox(input)
├── 沙箱全局启用？
│   └── 否 → 返回 false
├── 显式禁用沙箱且策略允许？
│   └── 是 → 返回 false
├── 命令为空？
│   └── 是 → 返回 false
├── 命令在排除列表中？
│   └── 是 → 返回 false
└── 返回 true（使用沙箱）
```

#### 2. 排除命令匹配流程

```
containsExcludedCommand(command)
├── ant 用户检查动态禁用
│   ├── 获取 tengu_sandbox_disabled_commands
│   ├── 检查子字符串匹配
│   └── 检查命令前缀匹配
├── 获取用户配置的 excludedCommands
├── 拆分复合命令
└── 对每个子命令：
    ├── 生成候选命令（剥离环境变量和包装器）
    └── 对每个排除模式：
        ├── 前缀匹配：cmd === prefix 或 cmd.startsWith(prefix + ' ')
        ├── 精确匹配：cmd === rule.command
        └── 通配符匹配：matchWildcardPattern(pattern, cmd)
```

#### 3. 候选命令生成算法

```typescript
// 不动点迭代，处理交错模式
const candidates = [trimmed]
const seen = new Set(candidates)
let startIdx = 0
while (startIdx < candidates.length) {
  const endIdx = candidates.length
  for (let i = startIdx; i < endIdx; i++) {
    const cmd = candidates[i]!
    const envStripped = stripAllLeadingEnvVars(cmd, BINARY_HIJACK_VARS)
    if (!seen.has(envStripped)) {
      candidates.push(envStripped)
      seen.add(envStripped)
    }
    const wrapperStripped = stripSafeWrappers(cmd)
    if (!seen.has(wrapperStripped)) {
      candidates.push(wrapperStripped)
      seen.add(wrapperStripped)
    }
  }
  startIdx = endIdx
}
```

### 依赖函数说明

| 函数 | 来源 | 用途 |
|------|------|------|
| `SandboxManager.isSandboxingEnabled()` | sandbox-adapter.ts | 检查沙箱是否全局启用 |
| `SandboxManager.areUnsandboxedCommandsAllowed()` | sandbox-adapter.ts | 检查策略是否允许非沙箱命令 |
| `splitCommand_DEPRECATED` | commands.ts | 拆分复合命令 |
| `stripAllLeadingEnvVars` | bashPermissions.ts | 剥离环境变量前缀 |
| `stripSafeWrappers` | bashPermissions.ts | 剥离安全包装器 |
| `bashPermissionRule` | bashPermissions.ts | 解析权限规则 |
| `matchWildcardPattern` | bashPermissions.ts | 通配符匹配 |
| `BINARY_HIJACK_VARS` | bashPermissions.ts | 二进制劫持变量正则 |
| `getFeatureValue_CACHED_MAY_BE_STALE` | growthbook.ts | 获取动态配置 |
| `getSettings_DEPRECATED` | settings.ts | 获取用户设置 |

## 关键代码路径与文件引用

### 核心文件

```
src/tools/BashTool/
├── shouldUseSandbox.ts          # 本文件：沙箱决策逻辑
├── bashPermissions.ts           # 权限规则解析和匹配
├── toolName.ts                  # BASH_TOOL_NAME 常量
└── ...

src/utils/sandbox/
├── sandbox-adapter.ts           # SandboxManager 接口

src/utils/bash/
├── commands.ts                  # 命令拆分工具

src/utils/settings/
├── settings.ts                  # 设置管理

src/services/analytics/
├── growthbook.ts                # 动态配置获取
```

### 调用关系

```
shouldUseSandbox.ts
├── 导入：
│   ├── getFeatureValue_CACHED_MAY_BE_STALE ← growthbook.ts
│   ├── splitCommand_DEPRECATED ← commands.ts
│   ├── SandboxManager ← sandbox-adapter.ts
│   ├── getSettings_DEPRECATED ← settings.ts
│   └── {BINARY_HIJACK_VARS, bashPermissionRule, matchWildcardPattern, stripAllLeadingEnvVars, stripSafeWrappers} ← bashPermissions.ts
│
└── 被调用：
    └── BashTool 执行流程（通过 bashPermissions.ts 间接调用）
```

## 依赖与外部交互

### 1. 沙箱管理器 (SandboxManager)

```typescript
// 来自 sandbox-adapter.ts
interface ISandboxManager {
  isSandboxingEnabled(): boolean
  areUnsandboxedCommandsAllowed(): boolean
  // ... 其他方法
}
```

### 2. GrowthBook 动态配置

仅对 `USER_TYPE === 'ant'` 用户生效：

```typescript
const disabledCommands = getFeatureValue_CACHED_MAY_BE_STALE<{
  commands: string[]    // 完全禁用的命令列表
  substrings: string[]  // 禁用的子字符串列表
}>('tengu_sandbox_disabled_commands', { commands: [], substrings: [] })
```

### 3. 用户设置

```typescript
// settings.json 中的配置
{
  "sandbox": {
    "enabled": true,
    "excludedCommands": ["git:*", "npm run test:*"]
  }
}
```

## 风险、边界与改进建议

### 已知风险

1. **排除列表不是安全边界**
   - 注释明确说明：`excludedCommands` 是用户便利功能，不是安全边界
   - 绕过排除列表不会构成安全漏洞
   - 真正的安全控制是沙箱权限系统的用户提示

2. **动态配置仅对 ant 用户生效**
   - `tengu_sandbox_disabled_commands` 仅当 `USER_TYPE === 'ant'` 时检查
   - 外部用户无法使用此功能

3. **命令解析失败的处理**
   - `splitCommand_DEPRECATED` 可能抛出异常（如畸形 bash 语法）
   - 当前处理：捕获异常，将命令视为未排除
   - 这可能导致某些畸形命令绕过排除检查

### 边界情况

1. **复合命令处理**
   - `docker ps && curl evil.com` 会被拆分为两个子命令分别检查
   - 防止仅因第一个子命令匹配排除模式就放过整个复合命令

2. **环境变量剥离**
   - 仅剥离 `SAFE_ENV_VARS` 白名单中的变量
   - `BINARY_HIJACK_VARS` (LD_*, DYLD_*, PATH) 不会被剥离，防止二进制劫持

3. **包装器命令处理**
   - 支持剥离 `timeout`, `time`, `nice`, `nohup`, `stdbuf`
   - 使用不动点迭代处理交错模式

### 改进建议

1. **增强错误处理**
   ```typescript
   // 当前：解析失败时视为未排除
   // 建议：解析失败时采取更保守的策略
   try {
     subcommands = splitCommand_DEPRECATED(command)
   } catch {
     // 保守策略：要求用户确认
     return true // 或触发权限提示
   }
   ```

2. **统一命令解析**
   - 当前使用 `splitCommand_DEPRECATED`，基于 shell-quote
   - 建议迁移到 tree-sitter AST 解析（已在 `parseForSecurity` 中实现）

3. **配置验证**
   - 添加对 `excludedCommands` 格式的验证
   - 防止用户配置无效的排除模式

4. **审计日志**
   - 记录排除命令匹配决策，用于安全审计
   - 区分用户配置的排除和动态配置的排除

### 测试建议

1. **边界测试**：
   - 畸形 bash 语法
   - 空命令
   - 超长命令
   - 特殊字符和 Unicode

2. **安全测试**：
   - 尝试绕过排除检查的各种方式
   - 环境变量注入
   - 命令注入

3. **集成测试**：
   - 与 SandboxManager 的集成
   - 与权限系统的集成
