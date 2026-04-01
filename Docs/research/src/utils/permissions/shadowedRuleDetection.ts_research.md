# shadowedRuleDetection.ts 深入研究

## 1. 场景与职责

`shadowedRuleDetection.ts` 是 Claude Code 权限系统的规则冲突检测模块，负责识别和报告"不可达"的权限规则。当用户配置了冲突的权限规则时（例如同时配置了工具级的 ask 规则和特定内容的 allow 规则），该模块能够检测出这些冲突并提供修复建议。

### 核心职责

1. **规则冲突检测**: 检测被其他规则"遮蔽"而无法生效的 allow 规则
2. **阴影类型识别**: 区分被 deny 规则完全阻塞和被 ask 规则拦截两种场景
3. **修复建议生成**: 为用户提供可操作的修复建议
4. **沙箱感知**: 考虑沙箱自动允许功能对规则冲突的影响

### 使用场景

- **设置验证**: 用户添加或修改权限规则时检测潜在冲突
- **诊断工具**: `/doctor` 命令检查权限配置问题
- **设置界面**: 在权限规则列表中高亮显示冲突规则
- **启动检查**: 应用启动时检测并报告配置问题

---

## 2. 功能点目的

### 2.1 不可达规则检测

```typescript
export function detectUnreachableRules(
  context: ToolPermissionContext,
  options: DetectUnreachableRulesOptions,
): UnreachableRule[]
```

**目的**: 扫描所有 allow 规则，找出被其他规则遮蔽而无法生效的规则。

**检测逻辑**:
1. 遍历所有 allow 规则
2. 检查是否被 deny 规则完全阻塞（更严重）
3. 检查是否被 ask 规则拦截（用户总是会被提示）
4. 返回所有不可达规则及其详细信息

### 2.2 Deny 规则遮蔽检测

```typescript
function isAllowRuleShadowedByDenyRule(
  allowRule: PermissionRule,
  denyRules: PermissionRule[],
): ShadowResult
```

**目的**: 检测特定的 allow 规则是否被工具级的 deny 规则完全阻塞。

**场景示例**:
```
Deny: "Bash"          ← 工具级 deny
Allow: "Bash(ls:*)"   ← 这个 allow 永远不会生效
```

**检测条件**:
- allow 规则必须有具体内容（`ruleContent !== undefined`）
- 存在同名的工具级 deny 规则（`ruleContent === undefined`）

### 2.3 Ask 规则遮蔽检测

```typescript
function isAllowRuleShadowedByAskRule(
  allowRule: PermissionRule,
  askRules: PermissionRule[],
  options: DetectUnreachableRulesOptions,
): ShadowResult
```

**目的**: 检测特定的 allow 规则是否被工具级的 ask 规则拦截。

**场景示例**:
```
Ask: "Bash"           ← 工具级 ask
Allow: "Bash(npm:*)"  ← 这个 allow 不会自动生效，用户总被提示
```

**特殊处理 - 沙箱例外**:
当满足以下条件时，Bash 的 ask 规则不遮蔽特定 allow 规则：
- 工具是 Bash
- 沙箱自动允许已启用
- ask 规则来自个人设置（非共享设置）

原因：沙箱中的命令会被自动允许，不受 ask 规则影响。

### 2.4 共享设置源识别

```typescript
export function isSharedSettingSource(source: PermissionRuleSource): boolean
```

**目的**: 区分共享设置（团队可见）和个人设置。

**共享设置**:
- `projectSettings`: 提交到 git，团队共享
- `policySettings`: 企业推送，所有用户
- `command`: 斜杠命令 frontmatter，可能共享

**个人设置**:
- `userSettings`: 用户全局 ~/.claude 设置
- `localSettings`: gitignored 的每项目设置
- `cliArg`: 运行时 CLI 参数
- `session`: 内存中的会话规则
- `flagSettings`: --settings 标志

### 2.5 修复建议生成

```typescript
function generateFixSuggestion(
  shadowType: ShadowType,
  shadowingRule: PermissionRule,
  shadowedRule: PermissionRule,
): string
```

**目的**: 为用户提供可操作的修复建议。

**建议格式**:
- Deny 遮蔽: `Remove the "Bash" deny rule from {source}, or remove the specific allow rule from {source}`
- Ask 遮蔽: `Remove the "Bash" ask rule from {source}, or remove the specific allow rule from {source}`

---

## 3. 具体技术实现

### 3.1 数据结构

**不可达规则**:
```typescript
export type UnreachableRule = {
  rule: PermissionRule           // 被遮蔽的规则
  reason: string                 // 人类可读的说明
  shadowedBy: PermissionRule     // 遮蔽它的规则
  shadowType: ShadowType         // 'ask' | 'deny'
  fix: string                    // 修复建议
}
```

**阴影结果**（判别联合类型）:
```typescript
type ShadowResult =
  | { shadowed: false }
  | { shadowed: true; shadowedBy: PermissionRule; shadowType: ShadowType }
```

**检测选项**:
```typescript
export type DetectUnreachableRulesOptions = {
  sandboxAutoAllowEnabled: boolean  // 沙箱自动允许是否启用
}
```

### 3.2 检测算法

**主检测流程**:
```typescript
export function detectUnreachableRules(
  context: ToolPermissionContext,
  options: DetectUnreachableRulesOptions,
): UnreachableRule[] {
  const unreachable: UnreachableRule[] = []
  
  // 获取所有规则
  const allowRules = getAllowRules(context)
  const askRules = getAskRules(context)
  const denyRules = getDenyRules(context)

  // 检查每个 allow 规则
  for (const allowRule of allowRules) {
    // 1. 先检查 deny 遮蔽（更严重）
    const denyResult = isAllowRuleShadowedByDenyRule(allowRule, denyRules)
    if (denyResult.shadowed) {
      unreachable.push({
        rule: allowRule,
        reason: `Blocked by "${denyResult.shadowedBy.ruleValue.toolName}" deny rule (from ${shadowSource})`,
        shadowedBy: denyResult.shadowedBy,
        shadowType: 'deny',
        fix: generateFixSuggestion('deny', denyResult.shadowedBy, allowRule),
      })
      continue  // 跳过 ask 检查
    }

    // 2. 检查 ask 遮蔽
    const askResult = isAllowRuleShadowedByAskRule(allowRule, askRules, options)
    if (askResult.shadowed) {
      unreachable.push({
        rule: allowRule,
        reason: `Shadowed by "${askResult.shadowedBy.ruleValue.toolName}" ask rule (from ${shadowSource})`,
        shadowedBy: askResult.shadowedBy,
        shadowType: 'ask',
        fix: generateFixSuggestion('ask', askResult.shadowedBy, allowRule),
      })
    }
  }

  return unreachable
}
```

### 3.3 Deny 遮蔽检测逻辑

```typescript
function isAllowRuleShadowedByDenyRule(
  allowRule: PermissionRule,
  denyRules: PermissionRule[],
): ShadowResult {
  const { toolName, ruleContent } = allowRule.ruleValue

  // 只有有具体内容的 allow 规则才会被遮蔽
  // 工具级 allow 规则与 deny 规则冲突但不是"被遮蔽"
  if (ruleContent === undefined) {
    return { shadowed: false }
  }

  // 查找同名的工具级 deny 规则
  const shadowingDenyRule = denyRules.find(
    denyRule =>
      denyRule.ruleValue.toolName === toolName &&
      denyRule.ruleValue.ruleContent === undefined,
  )

  if (!shadowingDenyRule) {
    return { shadowed: false }
  }

  return { shadowed: true, shadowedBy: shadowingDenyRule, shadowType: 'deny' }
}
```

### 3.4 Ask 遮蔽检测逻辑（含沙箱例外）

```typescript
function isAllowRuleShadowedByAskRule(
  allowRule: PermissionRule,
  askRules: PermissionRule[],
  options: DetectUnreachableRulesOptions,
): ShadowResult {
  const { toolName, ruleContent } = allowRule.ruleValue

  // 只有有具体内容的 allow 规则才会被遮蔽
  if (ruleContent === undefined) {
    return { shadowed: false }
  }

  // 查找同名的工具级 ask 规则
  const shadowingAskRule = askRules.find(
    askRule =>
      askRule.ruleValue.toolName === toolName &&
      askRule.ruleValue.ruleContent === undefined,
  )

  if (!shadowingAskRule) {
    return { shadowed: false }
  }

  // 沙箱例外：Bash + 沙箱启用 + 个人设置中的 ask 规则
  if (toolName === BASH_TOOL_NAME && options.sandboxAutoAllowEnabled) {
    if (!isSharedSettingSource(shadowingAskRule.source)) {
      return { shadowed: false }
    }
    // 共享设置的 ask 规则仍然报告（其他团队成员可能没有沙箱）
  }

  return { shadowed: true, shadowedBy: shadowingAskRule, shadowType: 'ask' }
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 导出类型和函数

| 名称 | 行号 | 类型 | 描述 |
|------|------|------|------|
| `ShadowType` | 14 | type | 阴影类型：'ask' \| 'deny' |
| `UnreachableRule` | 19-25 | type | 不可达规则描述 |
| `DetectUnreachableRulesOptions` | 30-37 | type | 检测选项 |
| `isSharedSettingSource` | 61-67 | function | 检查是否为共享设置源 |
| `detectUnreachableRules` | 193-234 | function | 主检测函数 |

### 4.2 内部函数

| 函数 | 行号 | 描述 |
|------|------|------|
| `formatSource` | 72-74 | 格式化规则源显示 |
| `generateFixSuggestion` | 79-92 | 生成修复建议 |
| `isAllowRuleShadowedByAskRule` | 111-147 | Ask 遮蔽检测 |
| `isAllowRuleShadowedByDenyRule` | 160-184 | Deny 遮蔽检测 |

### 4.3 依赖关系

```
src/utils/permissions/shadowedRuleDetection.ts
├── 导入:
│   ├── src/Tool.ts (ToolPermissionContext)
│   ├── src/tools/BashTool/toolName.ts (BASH_TOOL_NAME)
│   ├── src/utils/permissions/PermissionRule.ts (PermissionRule, PermissionRuleSource)
│   └── src/utils/permissions/permissions.ts (getAllowRules, getAskRules, getDenyRules, permissionRuleSourceDisplayString)
│
├── 被导入:
│   ├── src/utils/doctorContextWarnings.ts (医生上下文警告)
│   └── src/components/permissions/rules/PermissionRuleList.tsx (权限规则列表 UI)
```

---

## 5. 依赖与外部交互

### 5.1 权限系统依赖

| 函数 | 来源 | 用途 |
|------|------|------|
| `getAllowRules` | `permissions.ts` | 获取所有 allow 规则 |
| `getAskRules` | `permissions.ts` | 获取所有 ask 规则 |
| `getDenyRules` | `permissions.ts` | 获取所有 deny 规则 |
| `permissionRuleSourceDisplayString` | `permissions.ts` | 格式化规则源显示 |

### 5.2 类型依赖

| 类型 | 来源 | 用途 |
|------|------|------|
| `ToolPermissionContext` | `Tool.ts` | 权限上下文 |
| `PermissionRule` | `PermissionRule.ts` | 规则类型 |
| `PermissionRuleSource` | `PermissionRule.ts` | 规则源类型 |
| `BASH_TOOL_NAME` | `toolName.ts` | Bash 工具名称常量 |

---

## 6. 风险、边界与改进建议

### 6.1 安全风险

| 风险 | 描述 | 评估 |
|------|------|------|
| 检测绕过 | 复杂的规则组合可能未被检测到 | 低风险 - 只影响用户体验，不影响安全 |
| 沙箱例外滥用 | 沙箱例外可能被误解为安全保证 | 低风险 - 代码注释已明确说明限制 |

### 6.2 边界情况

1. **多级遮蔽**: 只检测直接的工具级遮蔽，不检测多级间接遮蔽
   ```
   Deny: "Bash"
   Ask: "Bash(ls:*)"   ← 这个 ask 也被 deny 阻塞，但不会被检测
   Allow: "Bash(ls -la:*)"  ← 只检测到这个被 ask 遮蔽
   ```

2. **MCP 工具**: 当前实现主要针对 Bash 等内置工具，MCP 工具的规则遮蔽检测可能不完整

3. **通配符规则**: 不检测通配符规则之间的遮蔽（如 `Bash(*:*)` 和 `Bash(npm:*)`）

4. **Agent 工具**: `Agent(agentType)` 语法的规则遮蔽未在此模块处理

### 6.3 已知限制

1. **只检测 Allow 规则**: 不检测 Deny 规则之间的冲突或 Ask 规则之间的冲突
2. **工具级规则**: 只检测工具级规则对内容级规则的遮蔽
3. **静态分析**: 只在配置时检测，不运行时验证

### 6.4 改进建议

#### 6.4.1 功能增强

1. **多级遮蔽检测**: 递归检测多级规则遮蔽
   ```typescript
   function detectMultiLevelShadowing(
     rule: PermissionRule,
     context: ToolPermissionContext,
     depth: number = 0,
   ): ShadowChain[]
   ```

2. **通配符冲突检测**: 检测通配符规则之间的包含关系
   ```typescript
   function detectWildcardConflicts(
     rules: PermissionRule[],
   ): WildcardConflict[]
   ```

3. **规则建议**: 不仅检测问题，还提供优化建议
   ```typescript
   function suggestRuleOptimizations(
     context: ToolPermissionContext,
   ): RuleOptimization[]
   ```

#### 6.4.2 用户体验

1. **严重程度分级**: 区分 deny 遮蔽（严重）和 ask 遮蔽（提示）
   ```typescript
   type Severity = 'error' | 'warning' | 'info'
   interface UnreachableRuleWithSeverity extends UnreachableRule {
     severity: Severity
   }
   ```

2. **一键修复**: 提供自动修复功能
   ```typescript
   export function autoFixUnreachableRules(
     unreachable: UnreachableRule[],
   ): PermissionUpdate[]
   ```

3. **可视化**: 在设置界面显示规则依赖图

#### 6.4.3 代码质量

1. **单元测试**: 添加更多边界情况的测试
   ```typescript
   describe('shadowedRuleDetection', () => {
     it('should detect deny shadowing', () => { ... })
     it('should respect sandbox exception', () => { ... })
     it('should handle shared vs personal settings', () => { ... })
   })
   ```

2. **类型安全**: 使用更严格的类型
   ```typescript
   // 使用 branded types 区分不同类型的规则
   type ToolWideRule = PermissionRule & { __brand: 'toolWide' }
   type ContentSpecificRule = PermissionRule & { __brand: 'contentSpecific' }
   ```

3. **性能优化**: 对于大量规则，使用索引加速查找
   ```typescript
   const rulesByToolName = new Map<string, PermissionRule[]>()
   ```

#### 6.4.4 集成改进

1. **实时检测**: 在设置变更时实时检测并报告
2. **CI/CD 集成**: 提供 API 供 CI/CD 检查权限配置
3. **导入验证**: 在导入权限规则时验证冲突

---

## 7. 总结

`shadowedRuleDetection.ts` 是 Claude Code 权限系统的诊断模块，负责检测和报告权限规则之间的冲突。它通过识别被遮蔽的 allow 规则，帮助用户优化权限配置，避免"以为配置了自动允许但实际上总被提示"的困惑。

关键设计亮点：
- **沙箱感知**: 考虑沙箱自动允许功能对规则冲突的影响
- **共享设置区分**: 区分个人设置和团队共享设置，避免误报
- **修复建议**: 不仅检测问题，还提供可操作的修复建议
- **严重程度区分**: 区分 deny 遮蔽（完全阻塞）和 ask 遮蔽（提示拦截）

主要限制：
- 只检测直接的工具级规则对内容级规则的遮蔽
- 不检测多级间接遮蔽
- 不检测通配符规则之间的冲突

该模块虽然代码量小，但在提升用户体验和帮助用户理解权限系统行为方面起着重要作用。通过清晰的错误消息和修复建议，它降低了权限系统的使用门槛。
