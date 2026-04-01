# src/utils/billing.ts 深入研究

## 场景与职责

`billing.ts` 负责判断当前用户是否具备查看账单/费用信息的权限。由于 Claude Code 支持多种认证模式（Claude.ai OAuth 订阅者、API Key 用户、未登录用户），且不同组织角色拥有不同权限，因此需要集中封装计费访问控制逻辑。

该模块区分两类计费入口：
- **Console Billing Access**：面向 API Key / OAuth 控制台用户，展示使用成本与配额。
- **Claude.ai Billing Access**：面向 Claude.ai 订阅者（Max/Pro/Team/Enterprise），展示订阅账单。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `hasConsoleBillingAccess()` | 判断当前用户是否有权查看控制台费用信息（受 `DISABLE_COST_WARNINGS`、订阅状态、认证来源、组织/工作区角色约束） |
| `hasClaudeAiBillingAccess()` | 判断 Claude.ai 订阅者是否有权查看账单（Consumer 计划始终允许；Team/Enterprise 需 admin/billing/owner 角色） |
| `setMockBillingAccessOverride(value)` | 为 `/mock-limits` 测试命令提供账单权限的 mock 覆盖 |

## 具体技术实现

### Console Billing Access 判定链
1. 环境变量 `DISABLE_COST_WARNINGS` 为真 → `false`
2. 是 Claude.ai 订阅者 → `false`（订阅者走另一条账单通道）
3. 无认证（既无 token 也无 API key）→ `false`
4. 读取 `globalConfig.oauthAccount.organizationRole` 与 `workspaceRole`
5. 若角色缺失 → `false`（对 grandfathered 用户隐藏，避免未更新角色的旧用户看到错误数据）
6. 最终判定：`orgRole ∈ {admin, billing}` **或** `workspaceRole ∈ {workspace_admin, workspace_billing}`

### Claude.ai Billing Access 判定链
1. mock 覆盖优先（测试场景）
2. 非 Claude.ai 订阅者 → `false`
3. Consumer 计划（`max` / `pro`）→ `true`（个人用户天然拥有账单访问权）
4. Team/Enterprise → 检查 `organizationRole ∈ {admin, billing, owner, primary_owner}`

### Mock 机制
- 模块级变量 `mockBillingAccessOverride: boolean | null = null`
- `setMockBillingAccessOverride` 直接修改模块状态，供 `mockRateLimits.ts` 在测试命令中使用。

## 关键代码路径与文件引用

```
src/costHook.ts
  └── hasConsoleBillingAccess()

src/screens/REPL.tsx
  └── hasConsoleBillingAccess()

src/services/rateLimitMessages.ts
  └── hasClaudeAiBillingAccess()

src/components/messages/RateLimitMessage.tsx
  └── hasClaudeAiBillingAccess()

src/hooks/notifs/useRateLimitWarningNotification.tsx
  └── hasClaudeAiBillingAccess()

src/commands/rate-limit-options/rate-limit-options.tsx
  └── hasClaudeAiBillingAccess()

src/commands/extra-usage/extra-usage-core.ts
  └── hasClaudeAiBillingAccess(), openBrowser()

src/services/mockRateLimits.ts
  └── setMockBillingAccessOverride()
```

### 依赖模块
- `src/utils/auth.ts` — `getAnthropicApiKey`, `getAuthTokenSource`, `getSubscriptionType`, `isClaudeAISubscriber`
- `src/utils/config.ts` — `getGlobalConfig`
- `src/utils/envUtils.ts` — `isEnvTruthy`

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| 全局配置 | `getGlobalConfig()` | 读取 `oauthAccount.organizationRole`、`oauthAccount.workspaceRole` |
| 认证状态 | `src/utils/auth.ts` | 判断订阅类型、API key 存在性、token 来源 |
| 环境变量 | `process.env.DISABLE_COST_WARNINGS` | 全局关闭成本展示 |
| Mock 测试 | `src/services/mockRateLimits.ts` | 通过 `setMockBillingAccessOverride` 注入测试状态 |

## 风险、边界与改进建议

### 风险
1. **角色硬编码无统一常量**：`admin`、`billing`、`workspace_admin` 等字符串直接写在数组字面量中，若后端变更角色命名，需多处同步修改。
2. ** grandfathered 用户静默拒绝**：角色缺失时返回 `false` 的注释说明这是有意为之，但用户侧无提示，可能导致“为什么看不到账单”的支持工单。
3. **模块级 mock 状态污染**：`mockBillingAccessOverride` 是全局变量，若测试未正确清理，可能影响后续测试或其他模块。

### 边界
- `hasConsoleBillingAccess` 对“既是 Max 订阅者又使用 API key”的场景会误判为 `false`（注释已承认），但启动时已有警告提示，故此处简化处理。
- `hasClaudeAiBillingAccess` 的 Consumer 计划与 Team/Enterprise 判定路径完全分离，新增企业计划类型需同时修改两处。

### 改进建议
1. **角色常量化**：将计费相关角色提取为 `BILLING_ROLES` 常量对象，并与后端 schema 对齐（或从生成的类型中导入）。
2. **增加拒绝原因返回**：将函数签名从 `boolean` 改为 `{ access: boolean; reason?: string }`，便于上层展示“缺少角色”或“请联系管理员”等友好提示。
3. **Mock 封装为测试工具**：将 `setMockBillingAccessOverride` 与 `jest.mock` / `vitest` 的依赖注入结合，避免模块级可变状态泄漏到生产代码路径。
4. **统一计费权限模型**：Console 与 Claude.ai 的两套权限逻辑未来若合并（如统一 OAuth 账户体系），可考虑抽象为基于 RBAC 的通用函数。
