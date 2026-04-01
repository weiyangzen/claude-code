# permissionSetup.ts 深度研究文档

## 场景与职责

`permissionSetup.ts` 是 Claude Code 权限系统的核心配置与初始化模块，负责权限上下文的构建、模式转换、危险权限检测和自动模式（Auto Mode）的门控管理。该模块在应用启动、模式切换和设置变更时执行关键的权限状态转换。

**核心职责：**
1. **权限上下文初始化**：从 CLI 参数、设置文件构建初始权限上下文
2. **危险权限检测**：识别可能绕过安全分类器的危险规则
3. **模式转换管理**：处理权限模式之间的状态转换（default/auto/plan/acceptEdits/bypassPermissions）
4. **自动模式门控**：控制 Auto Mode 的可用性和进入/退出逻辑
5. **权限规则持久化**：管理规则在内存和磁盘之间的同步

## 功能点目的

### 1. 危险权限检测

#### Bash 危险权限检测
**`isDangerousBashPermission`** - 检测可能绕过分类器的 Bash 规则
- 工具级允许（无内容限制）
- 通配符规则（`*`）
- 解释器前缀（`python:*`, `node:*` 等）
- 解释器通配符（`python*`, `node*` 等）
- 带参数通配符（`python -*` 等）

#### PowerShell 危险权限检测
**`isDangerousPowerShellPermission`** - PowerShell 专用检测
- 除 Bash 通用模式外，额外检测：
  - PowerShell 别名（`iex`, `icm`, `saps` 等）
  - .NET 逃逸（`Add-Type`, `New-Object`）
  - 远程会话（`New-PSSession`, `Enter-PSSession`）
  - 作业启动（`Start-Job`, `Start-ThreadJob`）
  - `.exe` 后缀变体（Windows 二进制名）

#### Agent 危险权限检测
**`isDangerousTaskPermission`** - 子代理权限检测
- 任何 Agent 允许规则都被视为危险
- 原因：会绕过子代理提示的分类器评估

#### 综合检测
**`findDangerousClassifierPermissions`** - 收集所有危险权限
- 扫描磁盘加载的规则
- 扫描 CLI `--allowed-tools` 参数
- 返回结构化信息（规则值、来源、显示文本）

### 2. 过度宽泛权限检测

**`isOverlyBroadBashAllowRule`** / **`isOverlyBroadPowerShellAllowRule`**
- 检测等效于 YOLO 模式的规则（`Bash(*)` / `PowerShell(*)`）
- Ant 内部版本专用，用于警告用户

**`findOverlyBroadBashPermissions`** / **`findOverlyBroadPowerShellPermissions`**
- 收集所有过度宽泛的 shell 权限
- 用于启动时警告

### 3. 危险权限管理

**`removeDangerousPermissions`** - 从上下文移除危险权限
- 按来源分组规则
- 应用权限更新（`applyPermissionUpdate`）
- 支持持久化到设置文件

**`stripDangerousPermissionsForAutoMode`** - Auto Mode 进入准备
- 收集当前所有危险权限
- 从上下文中移除
- 记录被剥离的规则到 `strippedDangerousRules`

**`restoreDangerousPermissions`** - Auto Mode 退出恢复
- 从 `strippedDangerousRules` 恢复规则
- 清除暂存区

### 4. 权限模式转换

**`transitionPermissionMode`** - 中央模式转换处理器
- 处理 Plan Mode 进入/退出附件
- 处理 Auto Mode 激活/停用
- 协调状态转换副作用

**转换矩阵**：
```
From/To    default    auto    plan    acceptEdits    bypassPermissions
default      -       激活    准备     -              -
auto       停用       -      暂存     -              -
plan         -        -       -       -              -
acceptEdits  -        -      准备      -              -
```

### 5. CLI 参数解析

**`parseBaseToolsFromCLI`** - 基础工具集解析
- 支持预设名称（`default`, `none`）
- 支持自定义工具列表

**`parseToolListFromCLI`** - 工具列表解析
- 处理括号内的逗号（不分割）
- 处理空格分隔
- 示例：`"Bash(npm install), Read"` → `['Bash(npm install)', 'Read']`

### 6. 权限上下文初始化

**`initializeToolPermissionContext`** - 主初始化函数
- 解析 CLI 允许/禁止工具列表
- 处理基础工具集（自动禁止不在集合中的工具）
- 检测符号链接工作目录
- 检查 bypassPermissions 可用性（Statsig/设置）
- 加载磁盘规则
- 检测危险/过度宽泛权限
- 应用所有规则到上下文
- 验证并添加额外目录

### 7. 自动模式门控

**`verifyAutoModeGateAccess`** - 异步门控验证
- 检查 GrowthBook 配置（`tengu_auto_mode_config`）
- 检查设置禁用
- 检查模型支持
- 检查 fast mode 断路器
- 返回上下文转换函数（非预计算上下文）

**`isAutoModeGateEnabled`** - 同步门控检查
- 检查断路器状态
- 检查设置禁用
- 检查模型支持

**`getAutoModeUnavailableReason`** - 获取不可用原因
- `settings` - 用户设置禁用
- `circuit-breaker` - 服务端断路
- `model` - 当前模型不支持

### 8. 权限模式确定

**`initialPermissionModeFromCLI`** - 初始模式确定
- 优先级顺序：
  1. `--dangerously-skip-permissions` (bypassPermissions)
  2. `--permission-mode` CLI 参数
  3. 设置中的 `defaultMode`
- 检查 bypassPermissions 禁用（Statsig/设置）
- 检查 Auto Mode 断路器
- CCR 环境限制（仅 acceptEdits/plan）

## 具体技术实现

### 关键数据结构

```typescript
// 危险权限信息
type DangerousPermissionInfo = {
  ruleValue: PermissionRuleValue
  source: PermissionRuleSource
  ruleDisplay: string        // 如 "Bash(*)"
  sourceDisplay: string      // 如 "settings.json" 或 "--allowed-tools"
}

// Auto Mode 门控检查结果
type AutoModeGateCheckResult = {
  updateContext: (ctx: ToolPermissionContext) => ToolPermissionContext
  notification?: string
}

// Auto Mode 不可用原因
type AutoModeUnavailableReason = 'settings' | 'circuit-breaker' | 'model'

// Auto Mode 启用状态
type AutoModeEnabledState = 'enabled' | 'disabled' | 'opt-in'
```

### 危险模式检测算法

```typescript
// Bash 危险模式检查
function isDangerousBashPermission(toolName, ruleContent): boolean {
  if (toolName !== 'Bash') return false
  
  // 1. 工具级允许
  if (!ruleContent) return true
  
  const content = ruleContent.trim().toLowerCase()
  
  // 2. 通配符
  if (content === '*') return true
  
  // 3. 检查 DANGEROUS_BASH_PATTERNS
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    const lowerPattern = pattern.toLowerCase()
    if (content === lowerPattern) return true           // 精确匹配
    if (content === `${lowerPattern}:*`) return true    // 前缀语法
    if (content === `${lowerPattern}*`) return true     // 通配符
    if (content === `${lowerPattern} *`) return true    // 带空格
    if (content.startsWith(`${lowerPattern} -`) && content.endsWith('*')) return true
  }
  return false
}
```

### 模式转换流程

```
输入: fromMode, toMode, context
  ↓
1. 相同模式? → 直接返回
  ↓
2. 处理 Plan Mode 转换:
   - default→plan: 准备 Plan Mode
   - plan→其他: 设置退出标记
  ↓
3. 处理 Auto Mode 转换:
   - 进入 auto: 
     * 检查门控
     * setAutoModeActive(true)
     * stripDangerousPermissionsForAutoMode()
   - 退出 auto:
     * setAutoModeActive(false)
     * setNeedsAutoModeExitAttachment(true)
     * restoreDangerousPermissions()
  ↓
4. 清理 prePlanMode（如适用）
  ↓
返回: 更新后的 context
```

### 初始化流程

```
输入: CLI 参数、设置、额外目录
  ↓
1. 解析工具列表:
   - allowedToolsCli → parsedAllowedToolsCli
   - disallowedToolsCli → parsedDisallowedToolsCli
   - baseToolsCli → 自动禁止不在集合中的工具
  ↓
2. 检测符号链接工作目录
  ↓
3. 检查 bypassPermissions 可用性
  ↓
4. 从磁盘加载所有权限规则
  ↓
5. 检测危险权限:
   - overlyBroadBashPermissions (Ant only)
   - dangerousPermissions (Auto Mode only)
  ↓
6. 构建初始上下文
  ↓
7. 应用磁盘规则到上下文
  ↓
8. 验证并添加额外目录
  ↓
返回: { toolPermissionContext, warnings, dangerousPermissions, overlyBroadBashPermissions }
```

## 关键代码路径与文件引用

### 核心依赖

| 导入路径 | 用途 |
|---------|------|
| `bun:bundle` | 条件编译（TRANSCRIPT_CLASSIFIER, TEMPLATES） |
| `../../bootstrap/state.js` | 原始工作目录、模式转换状态 |
| `../../Tool.js` | `ToolPermissionContext` 类型 |
| `../settings/settings.js` | 设置读写 |
| `./permissionsLoader.js` | 从磁盘加载规则 |
| `./PermissionUpdate.js` | 应用权限更新 |
| `./permissionRuleParser.js` | 规则解析/序列化 |
| `./dangerousPatterns.js` | 危险模式常量 |
| `../../services/analytics/growthbook.js` | Statsig/GrowthBook 门控 |

### 被调用方

- **应用启动**: `initializeToolPermissionContext` 构建初始权限状态
- **模式切换 UI**: `transitionPermissionMode` 处理用户模式切换
- **设置变更**: `transitionPlanAutoMode` 响应设置更新
- **初始化检查**: `verifyAutoModeGateAccess` 验证 Auto Mode 可用性

### 调用时序

```
应用启动
  ↓
initializeToolPermissionContext
  ↓
用户切换模式 (Shift+Tab / CLI)
  ↓
transitionPermissionMode
  ↓
verifyAutoModeGateAccess (异步)
  ↓
applyPermissionUpdate (持久化)
```

## 依赖与外部交互

### 外部服务

1. **GrowthBook/Statsig**:
   - `tengu_auto_mode_config` - Auto Mode 配置
   - `tengu_disable_bypass_permissions_mode` - 绕过权限禁用
   - `tengu_iron_gate_closed` - 分类器故障关闭策略

2. **设置系统**:
   - 用户设置（`~/.claude/settings.json`）
   - 项目设置（`.claude/settings.json`）
   - 本地设置（`.claude/settings.local.json`）

3. **模型系统**:
   - `modelSupportsAutoMode` - 检查当前模型是否支持 Auto Mode

### 状态管理

- **全局状态**: `autoModeStateModule`（条件编译）
  - `setAutoModeActive()`
  - `isAutoModeActive()`
  - `setAutoModeCircuitBroken()`
  - `isAutoModeCircuitBroken()`
  - `getAutoModeFlagCli()`

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**:
   - `verifyAutoModeGateAccess` 是异步的，可能被模式切换抢占
   - 缓解：返回转换函数而非预计算上下文

2. **状态不一致**:
   - `autoModeStateModule` 和 `toolPermissionContext.mode` 可能不同步
   - 缓解：`transitionPermissionMode` 集中管理状态转换

3. **权限绕过**:
   - 危险规则检测可能遗漏新的代码执行向量
   - 缓解：定期更新 `DANGEROUS_BASH_PATTERNS`

4. **断路器延迟**:
   - GrowthBook 缓存可能导致断路器生效延迟
   - 缓解：同步检查 + 异步验证双重机制

### 边界条件

1. **空规则列表**: 正常处理，无危险权限
2. **无效工具名**: 解析时标准化，可能产生意外行为
3. **循环目录链接**: 符号链接检测有限，依赖路径解析
4. **并发设置变更**: 文件系统竞争条件，依赖原子写入

### 改进建议

1. **状态机重构**:
   - 当前模式转换逻辑分散，建议使用显式状态机
   - 定义所有有效转换和副作用

2. **测试覆盖**:
   - 添加模式转换矩阵的完整测试
   - 模拟 GrowthBook 响应测试门控逻辑

3. **性能优化**:
   - `findDangerousClassifierPermissions` 在大量规则时可能慢
   - 考虑缓存危险规则检测结果

4. **可观测性**:
   - 添加模式转换事件日志
   - 记录门控决策原因

5. **用户体验**:
   - 危险权限警告提供更具体的修复建议
   - 区分临时和持久化规则的风险等级

6. **安全增强**:
   - 添加规则变更审计日志
   - 敏感操作（如 bypassPermissions）需要额外确认

7. **代码简化**:
   - 文件较长（1500+ 行），建议按功能拆分为多个模块
   - 如：模式转换、门控管理、危险检测分离
