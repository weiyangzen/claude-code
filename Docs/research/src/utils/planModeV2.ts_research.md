# planModeV2.ts 研究文档

## 场景与职责

本模块提供 Plan Mode V2 的配置和特性开关功能。核心职责包括：

1. **代理数量配置**：根据订阅类型和环境变量确定探索代理数量
2. **访谈阶段开关**：控制计划模式访谈阶段的启用状态
3. **Pewter Ledger 实验**：控制计划文件结构提示的实验变体

该模块是 Plan Mode V2 功能的配置中心，支持基于用户订阅级别的差异化功能。

## 功能点目的

### 1. `getPlanModeV2AgentCount()` - 主代理数量
- **目的**：确定 Plan Mode V2 可用的主代理数量
- **优先级**：
  1. `CLAUDE_CODE_PLAN_V2_AGENT_COUNT` 环境变量（1-10）
  2. 订阅类型判断：
     - Max 订阅 + `default_claude_max_20x` 速率层级 → 3 个代理
     - Enterprise/Team 订阅 → 3 个代理
     - 其他 → 1 个代理

### 2. `getPlanModeV2ExploreAgentCount()` - 探索代理数量
- **目的**：确定探索阶段的代理数量
- **优先级**：
  1. `CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT` 环境变量（1-10）
  2. 默认：3 个代理

### 3. `isPlanModeInterviewPhaseEnabled()` - 访谈阶段开关
- **目的**：控制计划模式访谈阶段是否启用
- **优先级**：
  1. `USER_TYPE === 'ant'` → 始终启用
  2. `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` 环境变量
  3. GrowthBook flag `tengu_plan_mode_interview_phase`

### 4. `getPewterLedgerVariant()` - Pewter Ledger 实验变体
- **目的**：控制计划文件结构提示的实验
- **背景**：5 阶段计划模式工作流的第 4 阶段 "Final Plan" 提示实验
- **变体**：
  - `null`（对照组）：无特殊指导
  - `'trim'`：轻度压缩指导
  - `'cut'`：中度压缩指导
  - `'cap'`：严格大小限制
- **指标**：
  - 主要：会话级平均成本（Opus 输出价格 5× 输入）
  - 护栏：反馈差评率、请求/会话数、工具错误率

## 具体技术实现

### 关键流程

#### 代理数量计算流程

```
getPlanModeV2AgentCount()
    ↓
检查 CLAUDE_CODE_PLAN_V2_AGENT_COUNT 环境变量
    ↓
有效? → 返回该值
    ↓
获取 subscriptionType 和 rateLimitTier
    ↓
Max + default_claude_max_20x? → 返回 3
Enterprise/Team? → 返回 3
其他 → 返回 1
```

#### Pewter Ledger 变体获取流程

```
getPewterLedgerVariant()
    ↓
从 GrowthBook 获取 'tengu_pewter_ledger' 值
    ↓
值在 ['trim', 'cut', 'cap'] 中?
    是 → 返回该值
    否 → 返回 null
```

### 数据结构

```typescript
// Pewter Ledger 变体类型
export type PewterLedgerVariant = 'trim' | 'cut' | 'cap' | null

// 订阅类型（来自 auth.ts）
type SubscriptionType = 'max' | 'enterprise' | 'team' | 'other'

// 速率限制层级（来自 auth.ts）
type RateLimitTier = 'default_claude_max_20x' | 'other'
```

### 环境变量

| 变量名 | 用途 | 验证 |
|--------|------|------|
| `CLAUDE_CODE_PLAN_V2_AGENT_COUNT` | 主代理数量 | 1-10 的整数 |
| `CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT` | 探索代理数量 | 1-10 的整数 |
| `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` | 访谈阶段开关 | `true`/`false` |
| `USER_TYPE` | 用户类型 | `'ant'` 特殊处理 |

### GrowthBook Flags

| Flag | 用途 | 默认值 |
|------|------|--------|
| `tengu_plan_mode_interview_phase` | 访谈阶段外部用户开关 | `false` |
| `tengu_pewter_ledger` | 计划文件结构实验 | `null` |

### 实验基线数据

```
基线（对照组，14天，N=26.3M）：
- p50: 4,906 字符
- p90: 11,617 字符
- 平均: 6,207 字符
- 82% 使用 Opus 4.6
- 拒绝率随大小单调增长：20%（<2K）→ 50%（20K+）
```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `../services/analytics/growthbook.js` | GrowthBook 功能开关 |
| `./auth.js` | `getRateLimitTier()`, `getSubscriptionType()` |
| `./envUtils.js` | `isEnvDefinedFalsy()`, `isEnvTruthy()` |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/utils/messages.ts` | 获取 Pewter Ledger 变体，生成计划阶段提示 |
| `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts` | 检查访谈阶段开关 |
| `src/tools/EnterPlanModeTool/prompt.ts` | 获取代理数量配置 |
| `src/components/permissions/EnterPlanModePermissionRequest/EnterPlanModePermissionRequest.tsx` | 权限请求 UI |
| `src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx` | 退出权限 UI |
| `src/components/permissions/AskUserQuestionPermissionRequest/AskUserQuestionPermissionRequest.tsx` | 用户问题权限 UI |

## 风险、边界与改进建议

### 已知风险

1. **订阅类型耦合**
   - 风险：订阅类型判断逻辑分散在多处
   - 现状：依赖 `auth.ts` 的函数
   - 潜在问题：订阅类型变更时需同步修改

2. **环境变量验证**
   - 风险：环境变量值可能无效
   - 缓解：范围检查（1-10）
   - 潜在问题：非数字值静默忽略

3. **实验复杂性**
   - Pewter Ledger 实验涉及多个指标
   - 风险：指标间可能存在冲突
   - 现状：有明确的护栏指标

4. **功能开关缓存**
   - 风险：`CACHED_MAY_BE_STALE` 提示可能返回旧值
   - 现状：使用带缓存的 GrowthBook 客户端
   - 影响：配置变更可能有延迟

### 边界情况

| 场景 | 行为 |
|------|------|
| 环境变量非数字 | 忽略，使用默认逻辑 |
| 环境变量超出 1-10 | 忽略，使用默认逻辑 |
| 环境变量负数 | 忽略，使用默认逻辑 |
| GrowthBook 不可用 | 使用硬编码默认值 |
| 订阅类型未知 | 返回 1 个代理（最保守） |
| Pewter Ledger 无效值 | 返回 `null`（对照组） |

### 改进建议

1. **配置集中化**
   - 当前：分散在环境变量和 GrowthBook
   - 建议：统一配置中心，支持动态更新

2. **代理数量策略**
   - 当前：基于订阅类型的硬编码
   - 建议：
     - 基于负载动态调整
     - 用户可配置（在限制范围内）

3. **实验框架**
   - 当前：手动实现实验逻辑
   - 建议：
     - 使用专门的实验框架
     - 自动指标收集和分析

4. **缓存策略优化**
   - 当前：依赖 GrowthBook 缓存
   - 建议：
     - 添加本地缓存层
     - 支持强制刷新

5. **遥测增强**
   - 建议：
     - 记录配置决策路径
     - 记录实验分组
     - A/B 测试效果追踪

6. **错误处理**
   - 当前：静默回退到默认值
   - 建议：
     - 记录配置解析错误
     - 提供诊断信息

7. **文档化**
   - 建议：
     - 配置选项文档
     - 实验设计文档
     - 订阅权益矩阵

8. **测试覆盖**
   - 建议：
     - 各种订阅类型组合测试
     - 环境变量边界测试
     - GrowthBook 故障测试
