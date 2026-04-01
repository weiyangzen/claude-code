# utils.ts 深度研究文档

## 场景与职责

`src/components/TrustDialog/utils.ts` 是 TrustDialog 组件的配套工具模块，负责**检测和报告可能带来安全风险的功能配置**。这些函数被 TrustDialog 调用，用于：

1. **安全扫描**：读取项目设置文件，识别可能执行任意代码的配置
2. **来源追踪**：返回包含风险配置的**文件路径**，帮助用户定位配置来源
3. **遥测支持**：为信任对话框的遥测事件提供检测数据

### 核心职责

该模块不直接执行安全策略，而是**发现与报告**——将检测到的风险配置通知给 TrustDialog，由后者决定如何呈现给用户。

---

## 功能点目的

### 1. Hooks 配置检测 (`getHooksSources`)

检测 `.claude/settings.json` 或 `.claude/settings.local.json` 中是否配置了 hooks。

**检测逻辑**：
```typescript
function hasHooks(settings: SettingsJson | null): boolean {
  if (settings === null || settings.disableAllHooks) return false;
  if (settings.statusLine) return true;      // statusLine 是一种 hook
  if (settings.fileSuggestion) return true;  // fileSuggestion 也是一种 hook
  if (!settings.hooks) return false;
  
  // 检查 hooks 对象中是否有非空数组
  for (const hookConfig of Object.values(settings.hooks)) {
    if (hookConfig.length > 0) return true;
  }
  return false;
}
```

**返回**：包含 hooks 的配置文件路径数组（如 `[".claude/settings.json"]`）

**安全风险**：Hooks 可以执行任意 shell 命令，是代码执行的主要途径之一。

### 2. Bash 权限检测 (`getBashPermissionSources`)

检测权限规则中是否允许 Bash 工具执行。

**检测逻辑**：
```typescript
function hasBashPermission(rules: PermissionRule[]): boolean {
  return rules.some(
    rule =>
      rule.ruleBehavior === 'allow' &&
      (rule.ruleValue.toolName === BASH_TOOL_NAME ||
        rule.ruleValue.toolName.startsWith(BASH_TOOL_NAME + '(')),
  );
}
```

**匹配规则**：
- 精确匹配：`Bash`
- 前缀匹配：`Bash(...)`（如 `Bash(prompt:...)`）

**返回**：包含 Bash 允许规则的配置文件路径数组

### 3. API Key Helper 检测 (`getApiKeyHelperSources`)

检测是否配置了 `apiKeyHelper` 脚本。

**检测逻辑**：
```typescript
function hasApiKeyHelper(settings: SettingsJson | null): boolean {
  return !!settings?.apiKeyHelper;
}
```

**安全风险**：`apiKeyHelper` 是执行外部脚本获取 API 密钥的机制，可能执行任意代码。

### 4. AWS 命令检测 (`getAwsCommandsSources`)

检测是否配置了 AWS 相关命令。

**检测逻辑**：
```typescript
function hasAwsCommands(settings: SettingsJson | null): boolean {
  return !!(settings?.awsAuthRefresh || settings?.awsCredentialExport);
}
```

**检测字段**：
- `awsAuthRefresh`：刷新 AWS 认证的命令
- `awsCredentialExport`：导出 AWS 凭证的命令

### 5. GCP 命令检测 (`getGcpCommandsSources`)

检测是否配置了 GCP 认证刷新命令。

**检测逻辑**：
```typescript
function hasGcpCommands(settings: SettingsJson | null): boolean {
  return !!settings?.gcpAuthRefresh;
}
```

### 6. OTel Headers Helper 检测 (`getOtelHeadersHelperSources`)

检测是否配置了 OpenTelemetry headers helper 脚本。

**检测逻辑**：
```typescript
function hasOtelHeadersHelper(settings: SettingsJson | null): boolean {
  return !!settings?.otelHeadersHelper;
}
```

**安全风险**：该脚本用于动态获取 OTel 导出器的 headers，可能执行网络请求或任意代码。

### 7. 危险环境变量检测 (`getDangerousEnvVarsSources`)

检测是否设置了非安全列表的环境变量。

**检测逻辑**：
```typescript
function hasDangerousEnvVars(settings: SettingsJson | null): boolean {
  if (!settings?.env) return false;
  return Object.keys(settings.env).some(
    key => !SAFE_ENV_VARS.has(key.toUpperCase()),
  );
}
```

**安全列表** (`src/utils/managedEnvConstants.ts` 中的 `SAFE_ENV_VARS`)：
- 包含 80+ 个被认为安全的环境变量名
- 如：`ANTHROPIC_MODEL`, `AWS_PROFILE`, `BASH_MAX_TIMEOUT_MS` 等
- **危险变量**（不在列表中）：`HTTP_PROXY`, `ANTHROPIC_BASE_URL`, `NODE_TLS_REJECT_UNAUTHORIZED` 等

### 8. 列表格式化工具 (`formatListWithAnd`)

辅助函数，用于格式化检测结果的展示文本。

**功能**：
- 支持限制显示数量（如只显示前 3 个，其余显示为 "and N more"）
- 正确使用英文连接词（", " 和 " and "）

**示例**：
```typescript
formatListWithAnd(['A', 'B', 'C'], 2);  // "A, B, and 1 more"
formatListWithAnd(['A', 'B'], 0);       // "A and B" (limit=0 表示不限制)
```

---

## 具体技术实现

### 配置来源优先级

所有检测函数都检查两个配置来源：

```typescript
const sources: string[] = [];

// 1. 项目级配置 (.claude/settings.json)
const projectSettings = getSettingsForSource('projectSettings');
if (hasXxx(projectSettings)) {
  sources.push('.claude/settings.json');
}

// 2. 本地配置 (.claude/settings.local.json)
const localSettings = getSettingsForSource('localSettings');
if (hasXxx(localSettings)) {
  sources.push('.claude/settings.local.json');
}

return sources;
```

**设计决策**：
- 只检查项目级和本地配置，不检查用户全局配置 (`~/.claude/settings.json`)
- 原因：全局配置影响所有项目，不应阻止用户进入特定项目

### 依赖注入模式

函数依赖通过模块导入实现：

```typescript
import { getSettingsForSource } from 'src/utils/settings/settings.js';
import { BASH_TOOL_NAME } from '../../tools/BashTool/toolName.js';
import { SAFE_ENV_VARS } from '../../utils/managedEnvConstants.js';
import { getPermissionRulesForSource } from '../../utils/permissions/permissionsLoader.js';
```

### 类型安全

使用 TypeScript 类型确保配置结构正确：

```typescript
import type { PermissionRule } from 'src/utils/permissions/PermissionRule.js'
import type { SettingsJson } from 'src/utils/settings/types.js'
```

---

## 关键代码路径与文件引用

### 核心文件

| 文件 | 作用 |
|------|------|
| `src/components/TrustDialog/utils.ts` | 本模块，安全功能检测实现 |
| `src/utils/settings/settings.ts` | 配置读取 (`getSettingsForSource`) |
| `src/utils/settings/types.ts` | 配置类型定义 (`SettingsJson`) |
| `src/utils/permissions/permissionsLoader.ts` | 权限规则加载 |
| `src/utils/permissions/PermissionRule.ts` | 权限规则类型 |
| `src/utils/managedEnvConstants.ts` | 安全环境变量白名单 |
| `src/tools/BashTool/toolName.ts` | Bash 工具名称常量 |

### 调用关系

```
TrustDialog.tsx
  ├── getHooksSources()
  ├── getBashPermissionSources()
  ├── getApiKeyHelperSources()
  ├── getAwsCommandsSources()
  ├── getGcpCommandsSources()
  ├── getOtelHeadersHelperSources()
  └── getDangerousEnvVarsSources()
      └── utils.ts 中的检测函数
          ├── getSettingsForSource('projectSettings')
          ├── getSettingsForSource('localSettings')
          ├── getPermissionRulesForSource('projectSettings')
          ├── getPermissionRulesForSource('localSettings')
          └── SAFE_ENV_VARS (常量)
```

### 配置键名映射

| 检测功能 | SettingsJson 字段 | 配置文件 |
|----------|-------------------|----------|
| Hooks | `hooks`, `statusLine`, `fileSuggestion`, `disableAllHooks` | `.claude/settings.json` |
| Bash 权限 | `permissions.allow[]` | `.claude/settings.json` |
| API Key Helper | `apiKeyHelper` | `.claude/settings.json` |
| AWS 命令 | `awsAuthRefresh`, `awsCredentialExport` | `.claude/settings.json` |
| GCP 命令 | `gcpAuthRefresh` | `.claude/settings.json` |
| OTel Helper | `otelHeadersHelper` | `.claude/settings.json` |
| 环境变量 | `env` | `.claude/settings.json` |

---

## 依赖与外部交互

### 外部依赖

```typescript
// 配置系统
import { getSettingsForSource } from 'src/utils/settings/settings.js';
import type { SettingsJson } from 'src/utils/settings/types.js';

// 权限系统
import { getPermissionRulesForSource } from '../../utils/permissions/permissionsLoader.js';
import type { PermissionRule } from 'src/utils/permissions/PermissionRule.js';

// 常量
import { BASH_TOOL_NAME } from '../../tools/BashTool/toolName.js';
import { SAFE_ENV_VARS } from '../../utils/managedEnvConstants.js';
```

### 配置系统交互

`getSettingsForSource` 支持多种配置来源：
- `'userSettings'`：`~/.claude/settings.json`
- `'projectSettings'`：`.claude/settings.json`
- `'localSettings'`：`.claude/settings.local.json`
- `'policySettings'`：托管设置（企业策略）

本模块只使用 `projectSettings` 和 `localSettings`，**有意忽略** `userSettings` 和 `policySettings`。

**原因**：
1. **User Settings**：全局配置不应阻止用户进入特定项目
2. **Policy Settings**：企业策略是管理员强制执行的，不是用户可控的"风险"

---

## 风险、边界与改进建议

### 已知限制

1. **不检测实际执行的 hooks**
   - 仅检测配置存在性，不验证 hooks 是否真的会被触发
   - 例如：`disableAllHooks: true` 时，即使有 hooks 配置也不会执行

2. **不分析脚本内容**
   - `apiKeyHelper`、`awsAuthRefresh` 等只检测路径存在
   - 不验证脚本内容是否真正危险

3. **环境变量白名单维护负担**
   - `SAFE_ENV_VARS` 需要随新功能更新
   - 遗漏的变量可能导致误报或漏报

4. **权限规则解析简化**
   - 仅检测 `allow` 行为的 Bash 规则
   - 不处理复杂的权限逻辑（如 `ask` 行为实际上也会执行）

### 边界情况

| 场景 | 行为 |
|------|------|
| 配置文件不存在 | `getSettingsForSource` 返回 `null`，检测返回 `false` |
| 配置文件 JSON 语法错误 | 解析失败，检测返回 `false` |
| 配置值为空字符串 | 视为 falsy，检测返回 `false` |
| 环境变量名大小写 | 统一转为大写后比较 |
| Bash 规则为 `ask` 行为 | 不视为"允许"，但用户仍可能在提示后执行 |

### 改进建议

1. **增强检测粒度**

   ```typescript
   // 建议：返回更详细的检测信息
   export interface SecurityScanResult {
     hasFeature: boolean;
     source: string;
     details?: {
       hookCount?: number;
       hookTypes?: string[];
       scriptPaths?: string[];
       envVarNames?: string[];
     };
   }
   ```

2. **验证脚本存在性**

   ```typescript
   // 建议：检查配置的脚本路径是否真实存在
   async function validateScriptExists(scriptPath: string): Promise<boolean>;
   ```

3. **分析脚本内容风险**

   ```typescript
   // 建议：简单的静态分析检测危险模式
   function analyzeScriptRisk(scriptPath: string): 'low' | 'medium' | 'high';
   ```

4. **支持忽略特定检测**

   ```typescript
   // 建议：允许用户在配置中标记某些功能为"已知安全"
   // .claude/settings.json
   {
     "trustDialog": {
       "ignore": ["apiKeyHelper", "awsAuthRefresh"]
     }
   }
   ```

5. **缓存检测结果**

   当前每次打开信任对话框都重新读取配置，可考虑添加缓存：

   ```typescript
   const cache = new Map<string, { result: string[]; mtime: number }>();
   ```

6. **统一的安全评分**

   ```typescript
   // 建议：综合所有检测给出风险评分
   export function calculateSecurityRisk(): {
     score: number;  // 0-100
     level: 'low' | 'medium' | 'high' | 'critical';
     findings: SecurityFinding[];
   };
   ```

### 测试建议

建议添加以下单元测试：

1. **各检测函数的边界测试**
   - 空配置、null、undefined 处理
   - 部分匹配、完全匹配
   - 大小写敏感性（环境变量）

2. **多来源配置合并**
   - projectSettings 和 localSettings 同时存在
   - 相同功能在不同来源的配置

3. **与实际配置系统的集成测试**
   - 验证能正确读取真实的 `.claude/settings.json`
   - 验证文件修改后检测更新

4. **性能测试**
   - 大配置文件的解析性能
   - 频繁调用的缓存效果
