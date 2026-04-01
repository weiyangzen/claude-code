# Plugin Telemetry 研究文档

## 场景与职责

`pluginTelemetry.ts` 是 Claude Code 的插件系统遥测模块，专门处理插件生命周期事件的数据收集和隐私保护。该模块实现了"双列隐私模式"（twin-column privacy pattern），服务于以下场景：

1. **插件使用分析**：了解哪些插件被使用、使用频率、来源
2. **隐私保护**：对第三方插件名称进行脱敏处理
3. **错误追踪**：监控插件加载和执行失败
4. **企业合规**：支持组织级插件策略分析

### 核心特点

- **双列隐私模式**：每个用户定义名称字段同时发送原始值（PII 标记列）和脱敏值（公开列）
- **哈希聚合键**：`plugin_id_hash` 提供不依赖隐私的插件聚合标识
- **固定盐值**：跨组织可比较的哈希，支持趋势分析
- **错误分类**：将自由格式错误映射到 5 个稳定类别

## 功能点目的

### 1. 双列隐私模式
- **目的**：在保护用户隐私的同时保留分析能力
- **实现**：
  - `_PROTO_plugin_name`：原始名称，路由到 PII 标记的 BQ 列
  - `plugin_name_redacted`：脱敏名称（官方插件显示真实名称，第三方显示 'third-party'）

### 2. 插件 ID 哈希
- **目的**：提供不暴露插件名称的聚合键
- **实现**：SHA256(name@marketplace + 固定盐) 截断 16 字符
- **特点**：
  - 固定盐值（`claude-plugin-telemetry-v1`）跨所有仓库
  - 不旋转，保持趋势连续性
  - 用户可计算已知插件的哈希进行反向匹配

### 3. 插件作用域分类
- **目的**：区分插件来源（官方、企业、用户本地）
- **实现**：4 值枚举（official, default-bundle, org, user-local）

### 4. 启用来源追踪
- **目的**：了解插件如何进入会话（用户安装、组织策略、默认启用）
- **实现**：`EnabledVia` 枚举（user-install, org-policy, default-enable, seed-mount）

### 5. 错误分类
- **目的**：将自由格式错误消息映射到有限类别，便于 dashboard 聚合
- **实现**：正则模式匹配到 5 个类别（network, not-found, permission, validation, unknown）

## 具体技术实现

### 关键数据结构

```typescript
// 固定盐值 - 跨所有仓库和版本保持一致
const PLUGIN_ID_HASH_SALT = 'claude-plugin-telemetry-v1'

// 内置插件市场名称（避免循环依赖）
const BUILTIN_MARKETPLACE_NAME = 'builtin'

// 插件作用域（来源分类）
export type TelemetryPluginScope =
  | 'official'      // Anthropic 官方市场
  | 'org'           // 企业管理员推送
  | 'user-local'    // 用户本地安装
  | 'default-bundle' // 产品内置（@builtin）

// 启用来源
export type EnabledVia =
  | 'user-install'   // 用户显式安装
  | 'org-policy'     // 组织策略推送
  | 'default-enable' // 默认启用（builtin）
  | 'seed-mount'     // 种子目录挂载

// 调用触发方式
export type InvocationTrigger =
  | 'user-slash'      // 用户 / 命令
  | 'claude-proactive' // Claude 主动调用
  | 'nested-skill'    // 嵌套技能调用

// 执行上下文
export type SkillExecutionContext = 'fork' | 'inline' | 'remote'

// 安装来源
export type InstallSource =
  | 'cli-explicit'   // CLI 显式安装
  | 'ui-discover'    // UI 发现页
  | 'ui-suggestion'  // UI 建议
  | 'deep-link'      // 深度链接

// 错误分类
export type PluginCommandErrorCategory =
  | 'network'      // 网络错误
  | 'not-found'    // 资源不存在
  | 'permission'   // 权限错误
  | 'validation'   // 验证错误
  | 'unknown'      // 未知错误
```

### 关键流程

#### 1. 插件 ID 哈希生成 (`hashPluginId`)

```
hashPluginId(name, marketplace):
1. 构建键值：
   - 如果有 marketplace："{name}@{marketplace.toLowerCase()}"
   - 否则：name
2. 计算 SHA256：
   hash = SHA256(key + PLUGIN_ID_HASH_SALT)
3. 截断：取前 16 个十六进制字符
4. 返回哈希字符串
```

**设计考量**：
- marketplace 小写确保一致性
- 16 字符截断在 10k 插件规模下碰撞概率可忽略
- 固定盐支持跨组织趋势分析

#### 2. 插件作用域判定 (`getTelemetryPluginScope`)

```
getTelemetryPluginScope(name, marketplace, managedNames):
1. 如果 marketplace === 'builtin'：
   返回 'default-bundle'
2. 如果 isOfficialMarketplaceName(marketplace) 为 true：
   返回 'official'
3. 如果 managedNames 包含 name：
   返回 'org'
4. 默认：
   返回 'user-local'
```

#### 3. 启用来源判定 (`getEnabledVia`)

```
getEnabledVia(plugin, managedNames, seedDirs):
1. 如果 plugin.isBuiltin 为 true：
   返回 'default-enable'
2. 如果 managedNames 包含 plugin.name：
   返回 'org-policy'
3. 如果 plugin.path 以 seedDirs 中任一目录开头：
   返回 'seed-mount'
4. 默认：
   返回 'user-install'
```

**路径匹配注意**：
- 使用 `sep`（平台特定分隔符）
- 确保目录边界（`dir + sep` 避免 `/opt/plugins` 匹配 `/opt/plugins-extra`）

#### 4. 遥测字段构建 (`buildPluginTelemetryFields`)

```
buildPluginTelemetryFields(name, marketplace, managedNames):
1. 计算 scope = getTelemetryPluginScope(...)
2. 判断 isAnthropicControlled：
   - scope === 'official' || scope === 'default-bundle'
3. 返回字段对象：
   {
     plugin_id_hash: hashPluginId(name, marketplace),
     plugin_scope: scope,
     plugin_name_redacted: isAnthropicControlled ? name : 'third-party',
     marketplace_name_redacted: isAnthropicControlled ? marketplace : 'third-party',
     is_official_plugin: isAnthropicControlled
   }
```

#### 5. 会话插件启用事件 (`logPluginsEnabledForSession`)

```
logPluginsEnabledForSession(plugins, managedNames, seedDirs):
对于每个 plugin：
1. 解析 marketplace = parsePluginIdentifier(plugin.repository).marketplace
2. 调用 logEvent('tengu_plugin_enabled_for_session', {
     // PII 标记列（路由到受限 BQ 列）
     _PROTO_plugin_name: plugin.name,
     _PROTO_marketplace_name: marketplace（如有）,
     
     // 标准遥测字段
     ...buildPluginTelemetryFields(plugin.name, marketplace, managedNames),
     
     // 启用来源
     enabled_via: getEnabledVia(plugin, managedNames, seedDirs),
     
     // 插件能力统计
     skill_path_count: (plugin.skillsPath ? 1 : 0) + (plugin.skillsPaths?.length ?? 0),
     command_path_count: (plugin.commandsPath ? 1 : 0) + (plugin.commandsPaths?.length ?? 0),
     has_mcp: plugin.manifest.mcpServers !== undefined,
     has_hooks: plugin.hooksConfig !== undefined,
     
     // 版本信息（如有）
     version: plugin.manifest.version
   })
```

#### 6. 错误分类 (`classifyPluginCommandError`)

```
classifyPluginCommandError(error):
1. 提取错误消息：String(error.message ?? error)
2. 按优先级匹配正则：
   - network: ENOTFOUND, ECONNREFUSED, EAI_AGAIN, ETIMEDOUT, ECONNRESET, network, timed out...
   - not-found: 404, not found, does not exist...
   - permission: 401, 403, EACCES, EPERM, permission denied...
   - validation: invalid, malformed, schema, validation, parse error...
   - unknown: 默认
3. 返回分类字符串
```

#### 7. 插件加载错误事件 (`logPluginLoadErrors`)

```
logPluginLoadErrors(errors, managedNames):
对于每个 error：
1. 解析 {name, marketplace} = parsePluginIdentifier(error.source)
2. 确定 pluginName = error.plugin ?? name
3. 调用 logEvent('tengu_plugin_load_failed', {
     error_category: err.type,
     _PROTO_plugin_name: pluginName,
     _PROTO_marketplace_name: marketplace（如有）,
     ...buildPluginTelemetryFields(pluginName, marketplace, managedNames)
   })
```

## 关键代码路径与文件引用

### 导出函数

| 函数名 | 用途 | 调用位置 |
|-------|------|---------|
| `hashPluginId(name, marketplace)` | 生成插件哈希 | buildPluginTelemetryFields, 外部调用 |
| `getTelemetryPluginScope(...)` | 获取插件作用域 | buildPluginTelemetryFields |
| `getEnabledVia(...)` | 获取启用来源 | logPluginsEnabledForSession |
| `buildPluginTelemetryFields(...)` | 构建标准遥测字段 | logPluginsEnabledForSession, logPluginLoadErrors, 外部调用 |
| `buildPluginCommandTelemetryFields(...)` | 为命令调用构建字段 | SkillTool.ts, processSlashCommand.tsx |
| `logPluginsEnabledForSession(...)` | 记录会话启用插件 | 插件初始化 |
| `classifyPluginCommandError(error)` | 分类错误 | 错误处理 |
| `logPluginLoadErrors(errors, ...)` | 记录加载错误 | 插件加载 |

### 依赖文件

| 文件 | 用途 |
|-----|------|
| `src/services/analytics/index.js` | `logEvent()`, `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`, `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` |
| `src/types/plugin.js` | `LoadedPlugin`, `PluginError`, `PluginManifest` 类型 |
| `src/utils/plugins/pluginIdentifier.js` | `isOfficialMarketplaceName()`, `parsePluginIdentifier()` |

### 被调用方

- `src/services/compact/postCompactCleanup.ts` - 可能涉及插件状态
- `src/tools/SkillTool/SkillTool.ts` - 技能调用遥测
- `src/utils/processUserInput/processSlashCommand.tsx` - 斜杠命令遥测
- `src/utils/plugins/pluginInstallationHelpers.ts` - 安装遥测
- `src/services/plugins/pluginCliCommands.ts` - CLI 命令遥测

## 依赖与外部交互

### 常量定义

| 常量 | 值 | 用途 |
|-----|---|------|
| `PLUGIN_ID_HASH_SALT` | `'claude-plugin-telemetry-v1'` | 哈希盐值 |
| `BUILTIN_MARKETPLACE_NAME` | `'builtin'` | 内置插件标识 |

### 外部函数

| 函数 | 来源 | 用途 |
|-----|------|------|
| `isOfficialMarketplaceName()` | `pluginIdentifier.js` | 判断官方市场 |
| `parsePluginIdentifier()` | `pluginIdentifier.js` | 解析插件标识符 |
| `logEvent()` | `analytics/index.js` | 发送遥测事件 |

### 类型标记

- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`：标记非代码/路径数据
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED`：标记 PII 数据（路由到受限列）

## 风险、边界与改进建议

### 风险

1. **隐私泄露风险**
   - `_PROTO_*` 字段需确保正确路由到受限 BQ 列
   - 如果 sink 配置错误，PII 可能进入通用存储
   - 哈希盐固定，理论上可彩虹表攻击（但 16 字符截断增加难度）

2. **哈希碰撞**
   - 16 字符截断在 10k 插件规模下概率低，但非零
   - 碰撞导致不同插件被统计为同一插件

3. **分类准确性**
   - 错误分类基于正则匹配，可能误判
   - 新类型错误可能被归类为 'unknown'

4. **国际化支持**
   - 错误分类正则基于英文错误消息
   - 非英文系统可能无法正确分类

### 边界情况

1. **空 marketplace**
   - `buildPluginTelemetryFields` 支持 undefined marketplace
   - 哈希计算时省略 marketplace 部分

2. **路径遍历攻击**
   - `getEnabledVia` 中的路径检查使用 `startsWith`
   - 已添加分隔符检查防止部分匹配

3. **PluginError 变体**
   - 不同错误类型可能有不同字段（plugin vs pluginId）
   - `logPluginLoadErrors` 使用 fallback 逻辑处理

4. **大小写敏感**
   - marketplace 在哈希时转为小写
   - 插件名称保持原样（enabledPlugins 键大小写敏感）

### 改进建议

1. **隐私增强**
   - 定期轮换哈希盐（权衡趋势分析能力）
   - 添加哈希 pepper（服务器端秘密）
   - 实现差分隐私（添加噪声）

2. **分类改进**
   - 添加更多错误模式（基于实际数据）
   - 支持本地化错误消息分类
   - 使用机器学习分类（离线训练）

3. **可观测性**
   - 添加哈希碰撞检测和告警
   - 记录分类准确率统计
   - 监控 PII 字段路由正确性

4. **功能扩展**
   - 支持插件版本趋势分析
   - 添加插件依赖关系追踪
   - 支持插件性能指标（加载时间、内存使用）

5. **代码改进**
   - 添加单元测试覆盖边界情况
   - 使用类型守卫替代类型断言
   - 添加 JSDoc 文档

6. **配置灵活性**
   - 支持按组织配置盐值
   - 支持自定义错误分类规则
   - 支持遥测字段白名单/黑名单

7. **合规增强**
   - 支持 GDPR 数据删除请求
   - 实现遥测数据导出功能
   - 添加审计日志记录遥测发送

### 代码示例改进

```typescript
// 改进：添加哈希碰撞检测
const HASH_COLLISION_CHECK = new Map<string, string>()

export function hashPluginId(name: string, marketplace?: string): string {
  const key = marketplace ? `${name}@${marketplace.toLowerCase()}` : name
  const hash = createHash('sha256')
    .update(key + PLUGIN_ID_HASH_SALT)
    .digest('hex')
    .slice(0, 16)
  
  // 碰撞检测（开发/测试环境）
  if (process.env.NODE_ENV === 'development') {
    const existing = HASH_COLLISION_CHECK.get(hash)
    if (existing && existing !== key) {
      console.warn(`[PluginTelemetry] Hash collision detected: ${hash} for ${key} and ${existing}`)
    }
    HASH_COLLISION_CHECK.set(hash, key)
  }
  
  return hash
}

// 改进：支持本地化错误分类
const ERROR_PATTERNS: Record<PluginCommandErrorCategory, RegExp[]> = {
  network: [
    /ENOTFOUND|ECONNREFUSED|EAI_AGAIN|ETIMEDOUT|ECONNRESET/i,
    /network|connection|resolve|refused|timed out/i,
    // 添加其他语言模式
    /连接|超时|拒绝|网络/i, // 中文
  ],
  // ... 其他分类
}

export function classifyPluginCommandError(
  error: unknown,
  locale?: string
): PluginCommandErrorCategory {
  const msg = String((error as { message?: unknown })?.message ?? error)
  
  for (const [category, patterns] of Object.entries(ERROR_PATTERNS)) {
    if (patterns.some(p => p.test(msg))) {
      return category as PluginCommandErrorCategory
    }
  }
  
  return 'unknown'
}
```
