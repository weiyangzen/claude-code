# Research: src/commands/remote-setup/index.ts

## 场景与职责

本文件是 `/web-setup` 命令的入口定义文件，遵循 Claude Code CLI 的命令注册规范。职责包括：

1. **命令元数据定义**：声明命令名称、描述、类型、可用性范围
2. **功能开关控制**：通过 GrowthBook feature flag 和 Policy Limits 控制命令是否启用
3. **懒加载配置**：延迟加载实际的命令实现（`remote-setup.tsx`）

该命令用于帮助用户在 Claude.ai Web 平台（claude.ai/code）上设置 GitHub 集成，实现本地 CLI 与云端 Web 体验的无缝连接。

## 功能点目的

### 1. 命令基础配置

```typescript
{
  type: 'local-jsx',           // 使用 JSX 组件实现的本地命令
  name: 'web-setup',           // 命令名称，用户输入 /web-setup 触发
  description: 'Setup Claude Code on the web...',  // 帮助文本
  availability: ['claude-ai'], // 仅在 Claude.ai OAuth 环境下可用
}
```

**设计决策**：
- `type: 'local-jsx'`：使用 React 组件实现交互式 UI
- `availability: ['claude-ai']`：限制仅 Claude.ai 订阅者可用，Console API key 用户无法看到此命令

### 2. 功能开关 (isEnabled)

**双重检查机制**：

1. **GrowthBook Feature Flag**：`tengu_cobalt_lantern`
   - 用于渐进式发布（gradual rollout）
   - 可在发现问题时快速禁用功能
   - 默认值为 `false`

2. **Policy Limits**：`allow_remote_sessions`
   - 组织级策略控制
   - 企业/团队管理员可通过策略限制禁用远程会话功能
   - 调用 `isPolicyAllowed()` 检查

### 3. 可见性控制 (isHidden)

**动态隐藏逻辑**：
- 当 `allow_remote_sessions` 策略不允许时，命令从帮助和自动补全中隐藏
- 与 `isEnabled` 保持一致，确保用户不会看到无法使用的命令

### 4. 懒加载实现

```typescript
load: () => import('./remote-setup.js')
```

**目的**：
- 减少启动时内存占用
- 仅在用户实际调用命令时加载 `remote-setup.tsx` 及其依赖
- 符合 CLI 模块化设计原则

## 具体技术实现

### 类型定义

```typescript
import type { Command } from '../../commands.js'
```

使用 `satisfies Command` 确保类型安全，同时保留对象字面量的具体类型推断。

### 依赖导入

```typescript
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../../services/analytics/growthbook.js'
import { isPolicyAllowed } from '../../services/policyLimits/index.js'
```

| 依赖 | 用途 |
|------|------|
| `growthbook.ts` | Feature flag 读取（非阻塞缓存版本） |
| `policyLimits/index.ts` | 组织策略限制检查 |

## 关键代码路径与文件引用

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/commands.js` | `Command` 类型定义 |
| `src/services/analytics/growthbook.ts` | Feature flag 系统 |
| `src/services/policyLimits/index.ts` | 策略限制系统 |

### 被调用方

| 文件 | 用途 |
|------|------|
| `src/commands/remote-setup/remote-setup.tsx` | 实际的 UI 实现（懒加载） |

### 命令注册流程

```
index.ts (命令定义)
    ↓
commands.ts (命令注册中心)
    ↓
用户输入 /web-setup
    ↓
load() 触发 → remote-setup.tsx 动态导入
```

## 依赖与外部交互

### 外部系统

1. **GrowthBook**
   - Feature flag：`tengu_cobalt_lantern`
   - 使用 `_CACHED_MAY_BE_STALE` 后缀版本，避免阻塞启动

2. **Policy Limits API**
   - 策略键：`allow_remote_sessions`
   - 检查用户组织是否允许远程会话功能

### 内部依赖图

```
index.ts
├── ../../commands.js (类型)
├── ../../services/analytics/growthbook.js
└── ../../services/policyLimits/index.js
```

## 风险、边界与改进建议

### 功能边界

1. **Feature Flag 缓存**
   - 使用 `getFeatureValue_CACHED_MAY_BE_STALE` 可能读取到过期值
   - 首次启动或 flag 变更后可能需要最多 20 分钟才能感知
   - 权衡：启动性能 vs. 实时性

2. **Policy Limits 异步加载**
   - `isPolicyAllowed()` 读取的是本地缓存的策略限制
   - 初始加载可能未完成，此时默认允许（fail open）

3. **availability 与 isEnabled 的交互**
   - `availability: ['claude-ai']` 先过滤，然后才检查 `isEnabled()`
   - Console API key 用户根本看不到该命令

### 潜在风险

1. **功能开关不一致**
   - GrowthBook flag 和 Policy Limits 是两个独立系统
   - 可能出现 flag 启用但策略禁用，或反之的情况
   - 当前逻辑要求两者同时满足才启用

2. **懒加载失败**
   - 如果 `remote-setup.tsx` 文件损坏或缺失，`load()` 会抛出异常
   - 需要确保构建过程包含该文件

### 改进建议

1. **遥测增强**
   - 添加命令启用/禁用状态的埋点
   - 区分是 flag 禁用还是策略禁用

2. **错误处理**
   - 当前 `isEnabled` 未处理依赖抛异常的情况
   - 建议添加 try-catch，异常时默认禁用

3. **文档完善**
   - 在命令描述中添加更多上下文
   - 说明为什么某些用户看不到此命令

4. **A/B 测试支持**
   - 当前仅支持全局开关
   - 可考虑添加用户分桶逻辑，支持渐进式发布

### 配置建议

对于需要强制禁用此功能的企业用户：

```json
// 组织策略配置
{
  "restrictions": {
    "allow_remote_sessions": {
      "allowed": false
    }
  }
}
```

对于内部测试：

```bash
# 启用 feature flag（仅 ant 用户）
export CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_cobalt_lantern": true}'
```
