# dangerousPatterns.ts 研究文档

## 场景与职责

`dangerousPatterns.ts` 是 Claude Code 权限系统的安全模式定义模块，专门用于识别在自动模式（auto mode）下可能绕过分类器安全检查的危险权限规则前缀。

该模块的核心安全目标：

1. **防止代码执行绕过**：识别允许通过解释器（python、node 等）执行任意代码的规则
2. **跨平台一致性**：定义在 Unix 和 Windows 上都存在的代码执行入口点
3. **ANT-ONLY 扩展**：为 Anthropic 内部用户提供额外的危险模式（基于沙箱数据分析）

## 功能点目的

### 1. 跨平台代码执行入口 (CROSS_PLATFORM_CODE_EXEC)

定义在 Unix 和 Windows 上都存在的代码执行入口点，确保两个平台的危险模式列表保持一致：

**解释器**：
- `python`, `python3`, `python2` - Python 解释器
- `node`, `deno`, `tsx` - JavaScript/TypeScript 运行时
- `ruby`, `perl`, `php`, `lua` - 其他脚本语言

**包管理器运行器**：
- `npx`, `bunx` - 包执行器
- `npm run`, `yarn run`, `pnpm run`, `bun run` - 脚本运行

**Shell**：
- `bash`, `sh` - Unix shell（在 Windows 上通过 Git Bash/WSL 可用）

**远程执行**：
- `ssh` - SSH 远程命令执行

### 2. Bash 危险模式 (DANGEROUS_BASH_PATTERNS)

基于 `CROSS_PLATFORM_CODE_EXEC` 扩展的 Bash 特定危险模式：

**额外 Shell**：
- `zsh`, `fish` - 其他 Unix shell
- `eval`, `exec` - 代码求值/执行
- `env` - 环境变量执行
- `xargs` - 参数执行
- `sudo` - 特权执行

**ANT-ONLY 模式**（基于 Anthropic 内部沙箱数据分析）：
- `fa run`, `coo` - 内部工具
- `gh`, `gh api`, `curl`, `wget` - 网络/数据外泄
- `git` - Git 配置操作（可配置任意代码执行）
- `kubectl`, `aws`, `gcloud`, `gsutil` - 云资源操作

## 具体技术实现

### 数据结构

```typescript
export const CROSS_PLATFORM_CODE_EXEC = [
  // 解释器
  'python', 'python3', 'python2', 'node', 'deno', 'tsx',
  'ruby', 'perl', 'php', 'lua',
  // 包运行器
  'npx', 'bunx', 'npm run', 'yarn run', 'pnpm run', 'bun run',
  // Shell
  'bash', 'sh',
  // 远程
  'ssh',
] as const

export const DANGEROUS_BASH_PATTERNS: readonly string[] = [
  ...CROSS_PLATFORM_CODE_EXEC,
  'zsh', 'fish', 'eval', 'exec', 'env', 'xargs', 'sudo',
  ...(process.env.USER_TYPE === 'ant' ? [
    'fa run', 'coo', 'gh', 'gh api', 'curl', 'wget',
    'git', 'kubectl', 'aws', 'gcloud', 'gsutil',
  ] : []),
]
```

### 使用模式

这些模式在 `permissionSetup.ts` 中被 `isDangerousBashPermission()` 函数使用：

```typescript
// 来自 permissionSetup.ts
export function isDangerousBashPermission(
  toolName: string,
  ruleContent: string | undefined,
): boolean {
  // ...
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    const lowerPattern = pattern.toLowerCase()
    
    // 精确匹配
    if (content === lowerPattern) return true
    
    // 前缀语法: "python:*"
    if (content === `${lowerPattern}:*`) return true
    
    // 通配符: "python*"
    if (content === `${lowerPattern}*`) return true
    
    // 空格通配符: "python *"
    if (content === `${lowerPattern} *`) return true
    
    // 选项通配符: "python -*"
    if (content.startsWith(`${lowerPattern} -`) && content.endsWith('*')) return true
  }
  return false
}
```

## 关键代码路径与文件引用

### 导出常量
| 常量 | 类型 | 说明 |
|------|------|------|
| `CROSS_PLATFORM_CODE_EXEC` | `readonly string[]` | 跨平台代码执行入口 |
| `DANGEROUS_BASH_PATTERNS` | `readonly string[]` | Bash 危险模式列表 |

### 被引用位置
- `src/utils/permissions/permissionSetup.ts`：`isDangerousBashPermission()` 函数
- `src/utils/permissions/permissionSetup.ts`：`isDangerousPowerShellPermission()` 函数（共享 `CROSS_PLATFORM_CODE_EXEC`）

### 匹配规则变体

`permissionSetup.ts` 中的匹配器处理多种规则形状：

| 规则形状 | 示例 | 说明 |
|----------|------|------|
| 精确 | `python` | 匹配精确命令 |
| 前缀 | `python:*` | 匹配任何以 python: 开头的命令 |
| 通配符后缀 | `python*` | 匹配 python、python3 等 |
| 空格通配符 | `python *` | 匹配 "python script.py" |
| 选项通配符 | `python -*` | 匹配 "python -c 'code'" |

## 依赖与外部交互

### 上游依赖
该模块**无外部导入**，完全自包含。

### 下游消费者
| 模块 | 用途 |
|------|------|
| `permissionSetup.ts` | 危险权限检测 |

## 风险、边界与改进建议

### 风险点
1. **模式漂移**：Unix 和 Windows 的危险模式列表需要保持同步，防止安全漏洞
2. **ANT-ONLY 依赖**：内部模式基于沙箱数据分析，外部用户无法受益于这些发现
3. **误报/漏报**：过于宽泛的模式可能误报，过于严格可能漏报

### 边界条件
1. **大小写不敏感**：匹配器将规则和模式都转为小写进行比较
2. **空格敏感**：`npm run` 和 `npm` 被视为不同模式
3. **ANT-ONLY 限制**：内部模式仅在 `USER_TYPE === 'ant'` 时包含

### 改进建议
1. **动态更新**：考虑从远程配置加载危险模式，便于快速响应新威胁

```typescript
// 建议：支持远程配置
export async function getDangerousPatterns(): Promise<string[]> {
  const remotePatterns = await getRemoteDangerousPatterns()
  return [...DANGEROUS_BASH_PATTERNS, ...remotePatterns]
}
```

2. **模式分类**：按风险级别分类模式（HIGH/MEDIUM/LOW）

```typescript
export type RiskLevel = 'HIGH' | 'MEDIUM' | 'LOW'

export const DANGEROUS_PATTERNS: Record<RiskLevel, string[]> = {
  HIGH: ['python', 'node', 'bash', 'eval', 'exec'],
  MEDIUM: ['npm run', 'yarn run'],
  LOW: ['ssh'],
}
```

3. **正则表达式支持**：考虑支持更复杂的模式匹配
4. **审计日志**：记录危险规则被拦截的事件
5. **用户教育**：当危险规则被拦截时，向用户解释原因和建议的替代方案
6. **社区共享**：建立机制让社区贡献新的危险模式（经过安全审查）
