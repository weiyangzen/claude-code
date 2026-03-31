# 研究文档: src/commands/feedback/index.ts

## 场景与职责

本文件是 Claude Code 的 `/feedback` (反馈) 和 `/bug` (Bug 报告) 命令的入口配置模块。它定义了命令的元数据、启用条件和懒加载配置，是命令注册系统的组成部分。

### 使用场景
- 用户通过 `/feedback [描述]` 或 `/bug [描述]` 命令提交产品反馈
- 用户在使用 Claude Code 过程中遇到问题需要报告 Bug
- 内部测试人员或 Anthropic 员工提交反馈

### 核心职责
1. **命令注册**: 向命令系统注册 feedback 命令及其别名 bug
2. **启用控制**: 通过 `isEnabled` 函数控制命令在特定环境下的可用性
3. **懒加载配置**: 指向实际的命令实现文件 `./feedback.js`

---

## 功能点目的

### 1. 命令元数据定义

```typescript
const feedback = {
  aliases: ['bug'],
  type: 'local-jsx',
  name: 'feedback',
  description: `Submit feedback about Claude Code`,
  argumentHint: '[report]',
  // ...
}
```

| 属性 | 值 | 说明 |
|------|-----|------|
| `name` | `'feedback'` | 主命令名，用户输入 `/feedback` 触发 |
| `aliases` | `['bug']` | 别名，用户也可输入 `/bug` 触发同一命令 |
| `type` | `'local-jsx'` | 命令类型，表示这是一个本地 JSX 交互式命令 |
| `description` | 字符串 | 在帮助系统和命令提示中显示 |
| `argumentHint` | `'[report]'` | 参数提示，显示可选的描述参数 |

### 2. 启用条件控制 (isEnabled)

该命令在以下任一条件满足时**被禁用**：

| 条件 | 环境变量 | 说明 |
|------|----------|------|
| Bedrock 模式 | `CLAUDE_CODE_USE_BEDROCK` | 使用 AWS Bedrock API |
| Vertex 模式 | `CLAUDE_CODE_USE_VERTEX` | 使用 Google Vertex AI |
| Foundry 模式 | `CLAUDE_CODE_USE_FOUNDRY` | 使用 Foundry 平台 |
| 显式禁用反馈 | `DISABLE_FEEDBACK_COMMAND` | 手动禁用反馈功能 |
| 显式禁用 Bug | `DISABLE_BUG_COMMAND` | 手动禁用 Bug 报告 |
| 仅基本流量模式 | `isEssentialTrafficOnly()` | 隐私模式，禁用非必要网络流量 |
| Anthropic 内部用户 | `USER_TYPE === 'ant'` | 内部构建不启用 |
| 策略限制 | `!isPolicyAllowed('allow_product_feedback')` | 组织策略禁止反馈 |

### 3. 懒加载配置

```typescript
load: () => import('./feedback.js')
```

- 使用动态导入实现懒加载，减少启动时间
- 仅在用户实际调用命令时加载 `feedback.tsx` 中的实现

---

## 具体技术实现

### 类型定义

文件使用 TypeScript 的 `satisfies Command` 确保类型安全：

```typescript
import type { Command } from '../../commands.js'
// ...
} satisfies Command
```

这保证了对象结构符合 `Command` 类型定义的要求。

### 依赖的工具函数

| 函数 | 来源 | 用途 |
|------|------|------|
| `isPolicyAllowed` | `../../services/policyLimits/index.js` | 检查组织策略是否允许产品反馈 |
| `isEnvTruthy` | `../../utils/envUtils.js` | 检查环境变量是否为真值 |
| `isEssentialTrafficOnly` | `../../utils/privacyLevel.js` | 检查是否处于仅基本流量模式 |

### isEnabled 逻辑详解

```typescript
isEnabled: () =>
  !(
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    isEnvTruthy(process.env.DISABLE_FEEDBACK_COMMAND) ||
    isEnvTruthy(process.env.DISABLE_BUG_COMMAND) ||
    isEssentialTrafficOnly() ||
    process.env.USER_TYPE === 'ant' ||
    !isPolicyAllowed('allow_product_feedback')
  )
```

逻辑采用"否定之否定"模式：
- 内部使用 `||` 连接所有禁用条件
- 外部使用 `!` 取反，表示"没有任何禁用条件时启用"

---

## 关键代码路径与文件引用

### 本文件位置
```
src/commands/feedback/index.ts
```

### 直接依赖
| 文件 | 用途 |
|------|------|
| `../../commands.js` | 导入 `Command` 类型定义 |
| `../../services/policyLimits/index.js` | 导入策略限制检查 |
| `../../utils/envUtils.js` | 导入环境变量工具 |
| `../../utils/privacyLevel.js` | 导入隐私级别检查 |

### 被调用方
| 文件 | 用途 |
|------|------|
| `../../commands.ts` | 导入并注册 feedback 命令到命令列表 |

### 懒加载目标
| 文件 | 用途 |
|------|------|
| `./feedback.js` (编译后) / `./feedback.tsx` (源码) | 实际命令实现 |

---

## 依赖与外部交互

### 外部服务依赖

1. **Policy Limits API** (通过 `isPolicyAllowed`)
   - 检查组织级别的功能限制
   - 策略键: `'allow_product_feedback'`

2. **环境变量系统**
   - 读取多个 `process.env` 变量决定启用状态

### 类型系统交互

```
index.ts → commands.js (Command 类型)
         → services/policyLimits (isPolicyAllowed)
         → utils/envUtils (isEnvTruthy)
         → utils/privacyLevel (isEssentialTrafficOnly)
```

---

## 风险、边界与改进建议

### 潜在风险

1. **策略检查时机**
   - `isPolicyAllowed` 在模块加载时可能尚未完成初始化
   - 如果策略限制 API 尚未加载，可能返回默认允许状态

2. **环境变量竞争条件**
   - 依赖 `process.env` 在模块加载时的状态
   - 如果环境变量在运行时动态修改，可能产生不一致行为

3. **USER_TYPE 硬编码**
   - `'ant'` 字符串硬编码，如果内部构建标识变化需要同步修改

### 边界情况

1. **所有条件都不满足**
   - 命令正常启用，用户可以使用 `/feedback` 和 `/bug`

2. **多个禁用条件同时满足**
   - 任一条件满足即禁用，逻辑正确

3. **策略 API 失败**
   - `isPolicyAllowed` 在 API 失败时通常返回 `true` (fail open)
   - 但 `essential-traffic-only` 模式下对 `allow_product_feedback` 会 fail closed

### 改进建议

1. **缓存策略检查结果**
   ```typescript
   // 当前每次调用都重新检查
   // 可考虑在策略更新时触发重新评估
   ```

2. **更细粒度的控制**
   - 当前 `bug` 和 `feedback` 是同一命令的别名
   - 可考虑分离为两个独立的启用控制

3. **日志记录**
   - 当前静默禁用，用户不知道命令为何不可用
   - 建议添加调试日志说明禁用原因

4. **类型安全增强**
   - `USER_TYPE` 可使用常量枚举替代字符串字面量

### 测试建议

1. 测试各环境变量组合下的启用状态
2. 测试策略 API 失败时的降级行为
3. 测试动态导入失败时的错误处理
