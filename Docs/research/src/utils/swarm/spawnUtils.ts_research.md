# spawnUtils.ts 深度研究文档

## 场景与职责

`spawnUtils.ts` 是 Claude Code 多代理集群（Agent Swarm）架构中的**创建工具函数模块**，提供跨不同后端（tmux/iTerm2/in-process）共享的 teammate 创建工具函数。

### 核心场景

1. **CLI 标志继承**：确保子 teammate 继承父会话的重要设置（权限模式、模型选择、插件配置）
2. **环境变量转发**：转发关键环境变量到 tmux 创建的 teammate（tmux 可能启动新的登录 shell 而不继承父环境）

### 职责边界

- 不直接创建 teammate，只提供创建所需的辅助函数
- 不管理 teammate 生命周期
- 专注于配置继承和环境准备

---

## 功能点目的

### 1. 获取 Teammate 命令

**函数**：`getTeammateCommand()`

**目的**：获取用于创建 teammate 进程的命令。

**优先级**：
1. `CLAUDE_CODE_TEAMMATE_COMMAND` 环境变量（如果设置）
2. 打包模式：`process.execPath`（当前 Claude 二进制文件）
3. 非打包模式：`process.argv[1]`（脚本路径）

### 2. 构建继承的 CLI 标志

**函数**：`buildInheritedCliFlags(options?)`

**目的**：构建要从当前会话传播到子 teammate 的 CLI 标志。

**继承的标志**：
- `--dangerously-skip-permissions`（权限绕过模式）
- `--permission-mode acceptEdits/auto`（权限模式）
- `--model {model}`（模型覆盖）
- `--settings {path}`（设置文件路径）
- `--plugin-dir {dir}`（内联插件目录）
- `--teammate-mode {mode}`（teammate 模式）
- `--chrome` / `--no-chrome`（Chrome 标志）

**特殊处理**：
- 如果 `planModeRequired` 为 `true`，不继承绕过权限（计划模式优先于安全）

### 3. 构建继承的环境变量

**函数**：`buildInheritedEnvVars()`

**目的**：构建要转发到 tmux 创建的 teammate 的环境变量字符串。

**背景**：tmux 可能启动新的登录 shell，不继承父进程的环境变量。

**转发的变量**：
- 必需：`CLAUDECODE=1`, `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
- API 提供商：`CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `CLAUDE_CODE_USE_FOUNDRY`
- 自定义端点：`ANTHROPIC_BASE_URL`
- 配置目录：`CLAUDE_CONFIG_DIR`
- CCR 标记：`CLAUDE_CODE_REMOTE`, `CLAUDE_CODE_REMOTE_MEMORY_DIR`
- 代理设置：`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY` 等
- 证书：`SSL_CERT_FILE`, `NODE_EXTRA_CA_CERTS` 等

---

## 具体技术实现

### 数据结构

#### 选项类型

```typescript
{
  planModeRequired?: boolean;   // 是否要求计划模式
  permissionMode?: PermissionMode;  // 权限模式
}
```

### 关键流程

#### CLI 标志构建流程

```
buildInheritedCliFlags(options)
├── 检查 planModeRequired
│   └── true: 跳过权限继承
│   └── false: 
│       ├── 检查 permissionMode === 'bypassPermissions' 或 getSessionBypassPermissionsMode()
│       │   └── 添加 '--dangerously-skip-permissions'
│       ├── 检查 permissionMode === 'acceptEdits'
│       │   └── 添加 '--permission-mode acceptEdits'
│       └── 检查 permissionMode === 'auto'
│           └── 添加 '--permission-mode auto'
├── 检查 getMainLoopModelOverride()
│   └── 添加 `--model {quotedModel}`
├── 检查 getFlagSettingsPath()
│   └── 添加 `--settings {quotedPath}`
├── 遍历 getInlinePlugins()
│   └── 为每个插件添加 `--plugin-dir {quotedDir}`
├── 检查 getTeammateModeFromSnapshot()
│   └── 添加 `--teammate-mode {mode}`
└── 检查 getChromeFlagOverride()
    ├── true: 添加 '--chrome'
    └── false: 添加 '--no-chrome'
```

#### 环境变量构建流程

```
buildInheritedEnvVars()
├── 基础变量：
│   ├── 'CLAUDECODE=1'
│   └── 'CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1'
├── 遍历 TEAMMATE_ENV_VARS 数组
│   ├── 检查 process.env[key]
│   ├── 如果存在且非空
│   │   └── 添加 `${key}=${quote([value])}`
└── 返回空格分隔的字符串
```

### 转发的环境变量详解

| 变量 | 用途 | 备注 |
|------|------|------|
| `CLAUDE_CODE_USE_BEDROCK` | 使用 AWS Bedrock API | 避免 teammate 默认使用 firstParty |
| `CLAUDE_CODE_USE_VERTEX` | 使用 Google Vertex API | 同上 |
| `CLAUDE_CODE_USE_FOUNDRY` | 使用 Azure Foundry API | 同上 |
| `ANTHROPIC_BASE_URL` | 自定义 API 端点 | 转发到 teammate |
| `CLAUDE_CONFIG_DIR` | 配置目录覆盖 | 保持配置一致性 |
| `CLAUDE_CODE_REMOTE` | CCR 标记 | 用于 CCR 感知代码路径 |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | CCR 内存目录 | 防止在临时 CCR 文件系统上禁用内存 |
| `HTTPS_PROXY` / `https_proxy` | HTTPS 代理 | 转发父进程的 MITM 中继 |
| `HTTP_PROXY` / `http_proxy` | HTTP 代理 | 同上 |
| `NO_PROXY` / `no_proxy` | 代理排除 | 同上 |
| `SSL_CERT_FILE` | SSL 证书 | 同上 |
| `NODE_EXTRA_CA_CERTS` | Node.js CA 证书 | 同上 |
| `REQUESTS_CA_BUNDLE` | Python requests CA | 同上 |
| `CURL_CA_BUNDLE` | curl CA | 同上 |

---

## 关键代码路径与文件引用

### 核心导出

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `getTeammateCommand()` | Function | 获取 teammate 命令路径 |
| `buildInheritedCliFlags()` | Function | 构建继承的 CLI 标志 |
| `buildInheritedEnvVars()` | Function | 构建继承的环境变量 |

### 调用方文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `src/tools/shared/spawnMultiAgent.ts` | `buildInheritedEnvVars` | 构建 tmux teammate 的环境变量 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/bootstrap/state.ts` | `getChromeFlagOverride`, `getFlagSettingsPath`, `getInlinePlugins`, `getMainLoopModelOverride`, `getSessionBypassPermissionsMode` |
| `src/utils/bash/shellQuote.ts` | `quote` 函数用于安全引用参数 |
| `src/utils/bundledMode.ts` | `isInBundledMode` 检测打包模式 |
| `src/utils/permissions/PermissionMode.ts` | `PermissionMode` 类型 |
| `src/utils/swarm/backends/teammateModeSnapshot.ts` | `getTeammateModeFromSnapshot` |
| `src/utils/swarm/constants.ts` | `TEAMMATE_COMMAND_ENV_VAR` |

---

## 依赖与外部交互

### 模块依赖图

```
spawnUtils.ts
├── bootstrap/state.ts       # 会话状态获取
├── bash/shellQuote.ts       # 参数引用
├── bundledMode.ts           # 打包模式检测
├── permissions/PermissionMode.ts  # 权限模式类型
├── swarm/backends/teammateModeSnapshot.ts  # teammate 模式
└── swarm/constants.ts       # 常量定义
```

### 与 spawnMultiAgent.ts 的交互

```typescript
// spawnMultiAgent.ts
import { buildInheritedEnvVars } from '../../utils/swarm/spawnUtils.js';

// 在构建 tmux 命令时使用
const envStr = buildInheritedEnvVars();
const spawnCommand = `cd ${quote([workingDir])} && env ${envStr} ${quote([binaryPath])} ${teammateArgs}${flagsStr}`;

await sendCommandToPane(paneId, spawnCommand, !insideTmux);
```

### 与 constants.ts 的交互

```typescript
// constants.ts
export const TEAMMATE_COMMAND_ENV_VAR = 'CLAUDE_CODE_TEAMMATE_COMMAND';
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 环境变量遗漏

**风险**：`TEAMMATE_ENV_VARS` 列表可能遗漏新添加的重要环境变量。

**当前列表**：17 个变量

**改进建议**：
```typescript
// 添加自动化测试验证所有 CLAUDE_CODE_* 变量都被考虑
const ALL_CLAUDE_VARS = Object.keys(process.env).filter(k => 
  k.startsWith('CLAUDE_CODE_')
);

// 在测试中发现未列出的变量
const unlistedVars = ALL_CLAUDE_VARS.filter(v => 
  !TEAMMATE_ENV_VARS.includes(v as typeof TEAMMATE_ENV_VARS[number])
);
if (unlistedVars.length > 0) {
  console.warn(`Unlisted CLAUDE_CODE_* env vars: ${unlistedVars.join(', ')}`);
}
```

#### 2. 敏感信息泄漏

**风险**：转发的环境变量可能包含敏感信息（如代理凭据）。

**当前保护**：使用 `quote()` 函数进行 shell 引用

**改进建议**：
```typescript
// 添加敏感变量检测和警告
const SENSITIVE_PATTERNS = [/password/i, /secret/i, /token/i, /key/i];

export function buildInheritedEnvVars(): string {
  const envVars = ['CLAUDECODE=1', 'CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1'];
  
  for (const key of TEAMMATE_ENV_VARS) {
    const value = process.env[key];
    if (value !== undefined && value !== '') {
      // 检测敏感信息
      if (SENSITIVE_PATTERNS.some(p => p.test(key))) {
        logForDebugging(`[spawnUtils] Forwarding sensitive env var: ${key}`);
      }
      envVars.push(`${key}=${quote([value])}`);
    }
  }
  
  return envVars.join(' ');
}
```

#### 3. 标志冲突

**风险**：`buildInheritedCliFlags` 和 teammate 特定的标志可能冲突。

**当前处理**：在 `spawnMultiAgent.ts` 中手动移除冲突的标志

```typescript
// spawnMultiAgent.ts 中的处理
if (model) {
  // Remove any inherited --model flag first
  inheritedFlags = inheritedFlags
    .split(' ')
    .filter((flag, i, arr) => flag !== '--model' && arr[i - 1] !== '--model')
    .join(' ');
  // Add the teammate's model
  inheritedFlags = inheritedFlags
    ? `${inheritedFlags} --model ${quote([model])}`
    : `--model ${quote([model])}`;
}
```

**改进建议**：
```typescript
// 在 spawnUtils.ts 中提供更智能的标志合并
export function mergeCliFlags(
  inheritedFlags: string,
  overrides: Record<string, string | undefined>
): string {
  const flagMap = new Map<string, string>();
  
  // 解析继承的标志
  const parts = inheritedFlags.split(' ');
  for (let i = 0; i < parts.length; i++) {
    if (parts[i].startsWith('--')) {
      const key = parts[i];
      const value = parts[i + 1] && !parts[i + 1].startsWith('--') ? parts[++i] : '';
      flagMap.set(key, value);
    }
  }
  
  // 应用覆盖
  for (const [key, value] of Object.entries(overrides)) {
    if (value === undefined) {
      flagMap.delete(key);
    } else {
      flagMap.set(key, value);
    }
  }
  
  // 重建标志字符串
  return Array.from(flagMap.entries())
    .map(([k, v]) => v ? `${k} ${quote([v])}` : k)
    .join(' ');
}
```

### 边界条件

| 场景 | 行为 |
|------|------|
| `TEAMMATE_COMMAND_ENV_VAR` 设置为空字符串 | 视为未设置，使用默认逻辑 |
| 所有权限模式都不匹配 | 不添加权限相关标志 |
| 模型覆盖为空字符串 | 不添加 `--model` 标志 |
| 设置路径为空 | 不添加 `--settings` 标志 |
| 无内联插件 | 不添加 `--plugin-dir` 标志 |
| 环境变量值为空字符串 | 视为未设置，不转发 |
| 环境变量包含特殊字符 | `quote()` 函数进行转义 |

### 改进建议

#### 1. 配置化环境变量列表

```typescript
// 允许用户配置额外的环境变量
export function buildInheritedEnvVars(
  extraVars: string[] = []
): string {
  const allVars = [...TEAMMATE_ENV_VARS, ...extraVars];
  // ...
}
```

#### 2. 环境变量验证

```typescript
// 验证转发的环境变量值
export function validateEnvVar(key: string, value: string): boolean {
  // 检查值长度
  if (value.length > 10000) {
    logError(new Error(`Env var ${key} value too long`));
    return false;
  }
  
  // 检查是否包含 null 字节
  if (value.includes('\0')) {
    logError(new Error(`Env var ${key} contains null bytes`));
    return false;
  }
  
  return true;
}
```

#### 3. 标志冲突检测

```typescript
// 检测并报告标志冲突
export function detectFlagConflicts(
  inheritedFlags: string,
  newFlags: string
): string[] {
  const inheritedSet = new Set(parseFlags(inheritedFlags).map(f => f.name));
  const newSet = parseFlags(newFlags);
  
  return newSet
    .filter(f => inheritedSet.has(f.name))
    .map(f => f.name);
}
```

#### 4. 文档生成

```typescript
// 自动生成转发变量的文档
export function generateEnvVarDocs(): string {
  return TEAMMATE_ENV_VARS.map(varName => {
    const description = ENV_VAR_DESCRIPTIONS[varName];
    return `- \`${varName}\`: ${description || 'No description'}`;
  }).join('\n');
}
```

### 测试建议

1. **单元测试**：
   - 验证每个 CLI 标志正确继承
   - 验证 `planModeRequired` 阻止权限继承
   - 验证环境变量正确引用

2. **集成测试**：
   - 验证转发的环境变量在 tmux 中可用
   - 验证 CLI 标志在 teammate 进程中生效

3. **安全测试**：
   - 测试特殊字符的引用
   - 测试长值的处理
   - 测试敏感信息的日志记录
