# outputLimits.ts 研究文档

## 场景与职责

`outputLimits.ts` 是 Claude Code CLI 中 **Bash/PowerShell 工具输出长度限制** 的集中配置点。它定义了默认输出上限、硬顶上限，并提供一个读取环境变量 `BASH_MAX_OUTPUT_LENGTH` 的入口函数 `getMaxOutputLength()`。该模块被 `BashTool/utils.ts`、`PowerShellTool/prompt.ts`、`TaskOutput.ts` 等多个下游模块引用，用于决定何时截断 shell 输出。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `BASH_MAX_OUTPUT_UPPER_LIMIT` | 硬顶上限：150_000 字符，防止用户或模型通过环境变量设置过大的输出导致上下文爆炸。 |
| `BASH_MAX_OUTPUT_DEFAULT` | 默认值：30_000 字符，平衡信息完整性与上下文窗口占用。 |
| `getMaxOutputLength()` | 读取 `process.env.BASH_MAX_OUTPUT_LENGTH`，做整数解析、下限校验、上限封顶，返回最终生效值。 |

## 具体技术实现

### 1. 环境变量读取与校验流程

```ts
export function getMaxOutputLength(): number {
  const result = validateBoundedIntEnvVar(
    'BASH_MAX_OUTPUT_LENGTH',
    process.env.BASH_MAX_OUTPUT_LENGTH,
    BASH_MAX_OUTPUT_DEFAULT,    // 30_000
    BASH_MAX_OUTPUT_UPPER_LIMIT, // 150_000
  )
  return result.effective
}
```

- 委托 `src/utils/envValidation.ts` 的 `validateBoundedIntEnvVar` 完成校验。
- 校验规则：
  - 未设置 → 返回默认值（status: 'valid'）。
  - 解析失败或 ≤ 0 → 返回默认值（status: 'invalid'），并打印 debug 日志。
  - 超过上限 → 返回上限值（status: 'capped'），并打印 debug 日志。
  - 正常正整数且在范围内 → 返回该值（status: 'valid'）。

### 2. 常量设计

```ts
export const BASH_MAX_OUTPUT_UPPER_LIMIT = 150_000
export const BASH_MAX_OUTPUT_DEFAULT = 30_000
```

- 使用数字分隔符 `_` 提升可读性。
- 命名带 `BASH_` 前缀是历史遗留，实际同时服务于 BashTool 与 PowerShellTool。

## 关键代码路径与文件引用

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/utils/shell/outputLimits.ts` | 本文件 | 输出限制常量与读取函数。 |
| `src/utils/envValidation.ts` | 被调用 | `validateBoundedIntEnvVar` 提供通用环境变量整型校验。 |
| `src/tools/BashTool/utils.ts` | 调用方 | `formatOutput()` 使用 `getMaxOutputLength()` 截断 stdout。 |
| `src/tools/PowerShellTool/prompt.ts` | 调用方 | 在 prompt 中告知模型输出超过该值会被截断。 |
| `src/utils/task/TaskOutput.ts` | 调用方 | 任务输出缓冲区使用该限制决定是否 spill 到磁盘或截断。 |
| `src/utils/managedEnvConstants.ts` | 可能引用 | 若存在托管环境常量映射，可能间接关联。 |

## 依赖与外部交互

- **Node.js 内置**: `process.env`。
- **内部模块**: `../envValidation.js`。
- **无外部网络或进程交互**: 纯本地常量与配置读取。

## 风险、边界与改进建议

### 风险

1. **命名误导**: 常量前缀为 `BASH_`，但同样被 PowerShell 工具使用，新开发者可能误以为仅作用于 Bash。
2. **单一维度限制**: 仅限制字符长度，不限制行数或字节数；在包含大量宽字符（如 CJK 或 emoji）时，实际 token 消耗可能超出预期。
3. **环境变量全局生效**: `BASH_MAX_OUTPUT_LENGTH` 一旦设置，影响所有后续 shell 工具调用，无法按工具或按会话动态调整。

### 边界

- 下限为 1（通过 `parsed <= 0` 判断），不允许 0 或负值。
- 上限 150_000 是写死的编译期常量，无法通过环境变量本身突破。
- 仅做字符串到整数的 `parseInt`，不支持科学计数法或单位后缀（如 `30k`）。

### 改进建议

1. **重命名去 Bash 化**: 将常量与函数重命名为 `SHELL_MAX_OUTPUT_*` / `getShellMaxOutputLength()`，并在代码中保留一段时期的别名兼容。
2. **增加按工具覆盖能力**: 允许在 `ExecOptions` 或工具输入中传入 `maxOutputLength`，优先级高于环境变量，方便特定大输出场景（如 `cat` 大日志）临时放宽。
3. **双指标限制**: 除字符长度外，增加行数上限（如 2000 行），防止单行极长输出（如 minified JS）在截断前已撑爆上下文。
4. **增强日志可观测性**: 在 `capped` 或 `invalid` 时输出 `warn` 级别日志（当前仅 `logForDebugging`），帮助用户排查为何输出被意外截断。
