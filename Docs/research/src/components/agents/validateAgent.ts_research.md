# validateAgent.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心场景
`validateAgent.ts` 是 Claude Code **Agent 配置验证** 的核心模块，负责验证用户创建的 Agent 配置是否合法、合理。

主要应用场景：
1. **Agent 创建向导 - 类型步骤** (`TypeStep.tsx`): 验证 Agent 标识符
2. **Agent 创建向导 - 确认步骤** (`ConfirmStep.tsx`): 完整验证 Agent 配置
3. **Agent 编辑器**: 验证编辑后的配置

### 1.2 核心职责
- **标识符验证**: 检查 `agentType` 格式（长度、字符、唯一性）
- **描述验证**: 检查 `whenToUse` 长度和合理性
- **工具验证**: 检查工具列表有效性
- **系统提示词验证**: 检查 `systemPrompt` 长度

---

## 2. 功能点目的

### 2.1 验证规则

#### Agent 标识符规则
| 规则 | 限制 |
|------|------|
| 必填 | 不能为空 |
| 格式 | 必须以字母数字开头和结尾，中间可含连字符 |
| 正则 | `/^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$/` |
| 最小长度 | 3 个字符 |
| 最大长度 | 50 个字符 |
| 唯一性 | 不能与现有 Agent 重复（跨来源） |

#### 描述（whenToUse）规则
| 规则 | 限制 |
|------|------|
| 必填 | 必须提供 |
| 建议长度 | 至少 10 个字符（警告） |
| 最大警告 | 超过 5000 字符警告 |

#### 工具规则
| 规则 | 限制 |
|------|------|
| 类型 | 必须是数组 |
| 通配符 | `undefined` 或 `['*']` 表示所有工具（警告） |
| 空数组 | 警告：Agent 能力受限 |
| 有效性 | 所有工具名必须在可用工具列表中 |

#### 系统提示词规则
| 规则 | 限制 |
|------|------|
| 必填 | 必须提供 |
| 最小长度 | 20 个字符 |
| 最大警告 | 超过 10000 字符警告 |

### 2.2 验证结果分级

- **Error**: 阻止保存的严重问题
- **Warning**: 提示用户但允许保存的潜在问题

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
export type AgentValidationResult = {
  isValid: boolean;
  errors: string[];
  warnings: string[];
};
```

### 3.2 关键流程

#### 标识符验证流程
```typescript
export function validateAgentType(agentType: string): string | null {
  // 1. 必填检查
  if (!agentType) {
    return 'Agent type is required';
  }
  
  // 2. 格式检查（正则）
  if (!/^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$/.test(agentType)) {
    return 'Agent type must start and end with alphanumeric characters...';
  }
  
  // 3. 长度检查
  if (agentType.length < 3) {
    return 'Agent type must be at least 3 characters long';
  }
  if (agentType.length > 50) {
    return 'Agent type must be less than 50 characters';
  }
  
  return null;  // 验证通过
}
```

#### 完整验证流程
```typescript
export function validateAgent(
  agent: Omit<CustomAgentDefinition, 'location'>,
  availableTools: Tools,
  existingAgents: AgentDefinition[],
): AgentValidationResult {
  const errors: string[] = [];
  const warnings: string[] = [];
  
  // 1. 验证 agentType
  if (!agent.agentType) {
    errors.push('Agent type is required');
  } else {
    const typeError = validateAgentType(agent.agentType);
    if (typeError) errors.push(typeError);
    
    // 检查重复（排除自身，用于编辑场景）
    const duplicate = existingAgents.find(
      a => a.agentType === agent.agentType && a.source !== agent.source,
    );
    if (duplicate) {
      errors.push(`Agent type "${agent.agentType}" already exists in ${...}`);
    }
  }
  
  // 2. 验证 whenToUse
  if (!agent.whenToUse) {
    errors.push('Description (description) is required');
  } else if (agent.whenToUse.length < 10) {
    warnings.push('Description should be more descriptive...');
  } else if (agent.whenToUse.length > 5000) {
    warnings.push('Description is very long...');
  }
  
  // 3. 验证 tools
  if (agent.tools !== undefined && !Array.isArray(agent.tools)) {
    errors.push('Tools must be an array');
  } else {
    if (agent.tools === undefined) {
      warnings.push('Agent has access to all tools');
    } else if (agent.tools.length === 0) {
      warnings.push('No tools selected...');
    }
    
    // 检查无效工具
    const resolvedTools = resolveAgentTools(agent, availableTools, false);
    if (resolvedTools.invalidTools.length > 0) {
      errors.push(`Invalid tools: ${resolvedTools.invalidTools.join(', ')}`);
    }
  }
  
  // 4. 验证 systemPrompt
  const systemPrompt = agent.getSystemPrompt();
  if (!systemPrompt) {
    errors.push('System prompt is required');
  } else if (systemPrompt.length < 20) {
    errors.push('System prompt is too short...');
  } else if (systemPrompt.length > 10000) {
    warnings.push('System prompt is very long...');
  }
  
  return { isValid: errors.length === 0, errors, warnings };
}
```

### 3.3 工具解析

```typescript
// 使用 resolveAgentTools 验证工具有效性
const resolvedTools = resolveAgentTools(agent, availableTools, false);

// 返回结果包含：
{
  hasWildcard: boolean;      // 是否使用通配符
  validTools: string[];      // 有效工具名
  invalidTools: string[];    // 无效工具名（用于错误提示）
  resolvedTools: Tools;      // 解析后的工具对象
  allowedAgentTypes?: string[];  // 允许的子 Agent 类型
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 调用方

| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `TypeStep.tsx` | `validateAgentType()` | 验证标识符输入 |
| `ConfirmStep.tsx` | `validateAgent()` | 完整配置验证 |

### 4.2 依赖文件

| 文件 | 用途 |
|------|------|
| `src/tools/AgentTool/agentToolUtils.ts` | `resolveAgentTools()` - 工具解析验证 |
| `src/tools/AgentTool/loadAgentsDir.ts` | `AgentDefinition`, `CustomAgentDefinition` 类型 |
| `src/Tool.ts` | `Tools` 类型 |
| `./utils.ts` | `getAgentSourceDisplayName()` - 来源显示名 |

---

## 5. 依赖与外部交互

### 5.1 依赖关系图

```
validateAgent.ts
├── agentToolUtils.ts (resolveAgentTools)
│   ├── Tool.ts (toolMatchesName)
│   └── constants/tools.ts (各种工具集合)
├── loadAgentsDir.ts (AgentDefinition, CustomAgentDefinition)
├── Tool.ts (Tools 类型)
└── utils.ts (getAgentSourceDisplayName)
    └── settings/constants.ts (getSettingSourceName)
```

### 5.2 验证流程图

```
validateAgent()
├── validateAgentType()
│   ├── 必填检查
│   ├── 正则匹配 /^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$/
│   └── 长度检查 (3-50)
├── 重复检查
│   └── existingAgents.find(...)
├── whenToUse 验证
│   ├── 必填检查
│   └── 长度检查 (警告: <10, >5000)
├── tools 验证
│   ├── 类型检查 (必须是数组)
│   ├── 通配符/空数组警告
│   └── resolveAgentTools() - 有效性检查
└── systemPrompt 验证
    ├── 必填检查
    └── 长度检查 (错误: <20, 警告: >10000)
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险1: 正则表达式限制
```typescript
/^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$/
```
**问题**: 
- 不允许下划线 `_`
- 不允许国际化域名（中文、日文等）
- 单字符标识符不可能满足（需要首尾都是字母数字，中间可有连字符）

**示例**:
- `a` - 无效（需要至少2字符满足首尾）
- `a-b` - 有效
- `test_runner` - 无效（含下划线）

#### 风险2: 并发重复检查
重复检查基于传入的 `existingAgents` 数组，如果在验证和保存之间有新 Agent 创建，可能导致重复。

#### 风险3: 工具解析的性能
`resolveAgentTools` 可能涉及大量工具遍历，对于大型项目可能成为性能瓶颈。

### 6.2 边界情况

| 场景 | 处理行为 |
|------|---------|
| `agentType` 为 `"a"` | 长度检查失败（需要 ≥3） |
| `agentType` 为 `"ab"` | 正则检查失败（无法匹配） |
| `agentType` 为 `"-abc"` | 正则检查失败（不以字母数字开头） |
| `tools` 为 `['*', 'FileReadTool']` | 不是通配符语义，按具体工具处理 |
| `systemPrompt` 为 19 个字符 | 错误：太短 |
| `systemPrompt` 为 20 个字符 | 通过 |
| 编辑时保持 `agentType` 不变 | 通过 `source !== agent.source` 排除自身检查 |

### 6.3 改进建议

#### 建议1: 更灵活的标识符规则
```typescript
// 支持下划线、Unicode 字符
const AGENT_TYPE_REGEX = /^[\p{L}\p{N}][\p{L}\p{N}_-]*[\p{L}\p{N}]$/u;

// 或使用更宽松的规则
function validateAgentType(agentType: string): string | null {
  // 使用 Unicode 属性转义支持国际化
  if (!/[\p{L}\p{N}]/u.test(agentType[0])) {
    return 'Agent type must start with a letter or number';
  }
  // ...
}
```

#### 建议2: 异步唯一性检查
```typescript
export async function validateAgentAsync(
  agent: ...,
  availableTools: Tools,
  existingAgents: AgentDefinition[],
  checkRemote: () => Promise<boolean>,  // 远程唯一性检查
): Promise<AgentValidationResult> {
  const result = validateAgent(agent, availableTools, existingAgents);
  
  // 额外检查远程/数据库
  if (result.isValid && await checkRemote()) {
    result.errors.push('Agent type already exists in remote registry');
    result.isValid = false;
  }
  
  return result;
}
```

#### 建议3: 更详细的验证结果
```typescript
export type ValidationIssue = {
  field: string;
  code: string;
  message: string;
  severity: 'error' | 'warning';
  suggestion?: string;  // 修复建议
};

export type AgentValidationResult = {
  isValid: boolean;
  issues: ValidationIssue[];
};

// 使用示例
{
  field: 'agentType',
  code: 'TOO_SHORT',
  message: 'Agent type must be at least 3 characters',
  severity: 'error',
  suggestion: 'Try "my-agent" instead of "my"'
}
```

#### 建议4: 可配置的验证规则
```typescript
export type ValidationRules = {
  agentType: {
    minLength: number;
    maxLength: number;
    pattern: RegExp;
    reservedNames: string[];  // 保留名
  };
  whenToUse: {
    minLength: number;
    maxLength: number;
  };
  systemPrompt: {
    minLength: number;
    maxLength: number;
  };
};

export const DEFAULT_RULES: ValidationRules = {
  agentType: { minLength: 3, maxLength: 50, pattern: /.../, reservedNames: ['help', 'exit'] },
  // ...
};

export function validateAgent(
  agent: ...,
  availableTools: Tools,
  existingAgents: AgentDefinition[],
  rules: ValidationRules = DEFAULT_RULES,
): AgentValidationResult {
  // 使用可配置的规则
}
```

#### 建议5: 验证缓存
对于重复验证相同 Agent 的场景，添加缓存：
```typescript
import memoize from 'lodash-es/memoize.js';

const getValidationCacheKey = (agent: ..., tools: Tools) => 
  `${agent.agentType}:${agent.whenToUse}:${JSON.stringify(agent.tools)}`;

export const validateAgentCached = memoize(
  validateAgent,
  getValidationCacheKey
);
```

#### 建议6: 实时验证支持
```typescript
export function createAgentValidator(
  availableTools: Tools,
  existingAgents: AgentDefinition[]
) {
  return {
    validateField<K extends keyof CustomAgentDefinition>(
      field: K,
      value: CustomAgentDefinition[K]
    ): ValidationIssue[] {
      // 只验证单个字段，用于实时反馈
    },
    
    validateAll(agent: CustomAgentDefinition): AgentValidationResult {
      return validateAgent(agent, availableTools, existingAgents);
    }
  };
}
```

### 6.4 测试建议

| 测试场景 | 验证点 |
|---------|--------|
| 边界长度值 | 2字符（失败）、3字符（通过）、50字符（通过）、51字符（失败） |
| 特殊字符 | 下划线、空格、Unicode、emoji |
| 并发重复 | 两个用户同时创建同名 Agent |
| 工具不存在 | 正确识别并报告无效工具 |
| 编辑场景 | 不报告自身为重复 |
| 性能 | 大量工具（>1000）时的验证速度 |
