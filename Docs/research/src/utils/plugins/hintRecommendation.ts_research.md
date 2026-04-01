# hintRecommendation.ts 深度研究文档

## 场景与职责

`hintRecommendation.ts` 实现 Claude Code 的插件提示推荐系统，是 `lspRecommendation.ts` 的配套模块。当 CLI/SDK 向 stderr 输出 `<claude-code-hint />` 标签时（由 Bash/PowerShell 工具检测），触发插件安装推荐。

核心场景：
1. **CLI 工具集成**：工具检测到特定环境时提示安装对应插件
2. **SDK 场景识别**：如检测到 AWS 环境提示安装 AWS 插件
3. **一次性提示**：每插件每会话仅提示一次，避免骚扰

## 功能点目的

### 1. 提示预存储门控 (`maybeRecordPluginHint`)
- **触发**：Shell 工具检测到 `type="plugin"` 的 hint 标签时同步调用
- **同步设计**：Shell 工具不应为剥离 stderr 行而等待市场查询
- **过滤条件**：
  - 功能开关 `tengu_lapis_finch` 关闭 → 丢弃
  - 本会话已显示对话框 → 丢弃
  - 用户禁用提示 (`claudeCodeHints.disabled`) → 丢弃
  - 已显示插件列表达上限 (MAX_SHOWN_PLUGINS=100) → 丢弃
  - 插件 ID 格式无效（非 `name@marketplace`）→ 丢弃
  - 非官方市场（v1 硬编码限制）→ 丢弃
  - 插件已安装 → 丢弃
  - 插件被策略阻止 → 丢弃
  - 本会话已尝试查找该插件 → 丢弃（防重复）

### 2. 提示解析 (`resolvePluginHint`)
- **异步执行**：Hook 侧执行市场查询（预存储门控跳过的操作）
- **缓存检查**：若插件不在市场缓存中，丢弃提示
- **遥测**：记录 `tengu_plugin_hint_detected` 事件，标记 PII

### 3. 提示状态管理
- **`markHintPluginShown`**：记录插件已提示（无论用户响应）
- **`disableHintRecommendations`**：用户选择"不再显示"时禁用
- **存储位置**：`GlobalConfig.claudeCodeHints`
  ```typescript
  {
    disabled?: boolean
    plugin?: string[] // 已提示的插件 ID 列表
  }
  ```

## 具体技术实现

### 核心常量与类型
```typescript
const MAX_SHOWN_PLUGINS = 100 // 限制配置增长

export type PluginHintRecommendation = {
  pluginId: string
  pluginName: string
  marketplaceName: string
  pluginDescription?: string
  sourceCommand: string // 触发提示的命令
}
```

### 预存储门控实现
```typescript
export function maybeRecordPluginHint(hint: ClaudeCodeHint): void {
  // 功能开关检查
  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_lapis_finch', false)) return
  
  // 会话级去重
  if (hasShownHintThisSession()) return
  
  const state = getGlobalConfig().claudeCodeHints
  if (state?.disabled) return
  
  // 配置增长上限
  const shown = state?.plugin ?? []
  if (shown.length >= MAX_SHOWN_PLUGINS) return
  
  const pluginId = hint.value
  const { name, marketplace } = parsePluginIdentifier(pluginId)
  
  // 格式验证
  if (!name || !marketplace) return
  
  // 官方市场限制（v1 硬编码）
  if (!isOfficialMarketplaceName(marketplace)) return
  
  // 已安装检查
  if (isPluginInstalled(pluginId)) return
  
  // 策略阻止检查
  if (isPluginBlockedByPolicy(pluginId)) return
  
  // 会话级查找去重
  if (triedThisSession.has(pluginId)) return
  triedThisSession.add(pluginId)
  
  // 通过所有门控，存储待处理提示
  setPendingHint(hint)
}
```

### 解析流程
```typescript
export async function resolvePluginHint(hint: ClaudeCodeHint): Promise<PluginHintRecommendation | null> {
  const pluginId = hint.value
  const { name, marketplace } = parsePluginIdentifier(pluginId)
  
  // 异步市场查询
  const pluginData = await getPluginById(pluginId)
  
  // 遥测上报（PII 标记）
  logEvent('tengu_plugin_hint_detected', {
    _PROTO_plugin_name: name as PII_TAGGED,
    _PROTO_marketplace_name: marketplace as PII_TAGGED,
    result: pluginData ? 'passed' : 'not_in_cache'
  })
  
  if (!pluginData) {
    logForDebugging(`[hintRecommendation] ${pluginId} not found in marketplace cache`)
    return null
  }
  
  return {
    pluginId,
    pluginName: pluginData.entry.name,
    marketplaceName: marketplace ?? '',
    pluginDescription: pluginData.entry.description,
    sourceCommand: hint.sourceCommand
  }
}
```

## 关键代码路径与文件引用

### 内部依赖
```
hintRecommendation.ts
  ├─ services/analytics/growthbook.ts: getFeatureValue_CACHED_MAY_BE_STALE
  ├─ services/analytics/index.ts: logEvent
  ├─ claudeCodeHints.ts:
  │   ├─ ClaudeCodeHint
  │   ├─ hasShownHintThisSession
  │   └─ setPendingHint
  ├─ config.ts: getGlobalConfig, saveGlobalConfig
  ├─ debug.ts: logForDebugging
  ├─ installedPluginsManager.ts: isPluginInstalled
  ├─ marketplaceManager.ts: getPluginById
  ├─ pluginIdentifier.ts: parsePluginIdentifier, isOfficialMarketplaceName
  └─ pluginPolicy.ts: isPluginBlockedByPolicy
```

### 外部调用方
| 调用方 | 调用函数 | 用途 |
|--------|----------|------|
| `bash.ts` / `powershell.ts` | `maybeRecordPluginHint` | 检测到 hint 标签时同步调用 |
| `hooks.ts` | `resolvePluginHint` | 异步解析待处理提示 |
| `UI 组件` | `markHintPluginShown` | 提示显示后记录 |
| `UI 组件` | `disableHintRecommendations` | 用户禁用提示 |

### 调用链
```
Bash/PowerShell Tool
  └─ 检测到 <claude-code-hint type="plugin" value="plugin@marketplace" />
      └─ maybeRecordPluginHint(hint) [同步]
          └─ 通过门控 → setPendingHint(hint)
              └─ Hook 侧
                  └─ resolvePluginHint(hint) [异步]
                      ├─ getPluginById() 市场查询
                      ├─ logEvent() 遥测
                      └─ 返回 PluginHintRecommendation
                          └─ UI 显示安装提示
                              ├─ 用户响应 → markHintPluginShown()
                              └─ 用户禁用 → disableHintRecommendations()
```

## 依赖与外部交互

### Hint 标签格式
```xml
<claude-code-hint type="plugin" value="aws@claude-plugins-official" />
```

### 配置存储
- **位置**：`~/.claude/config.json` 中的 `claudeCodeHints`
- **结构**：
  ```json
  {
    "claudeCodeHints": {
      "disabled": false,
      "plugin": ["aws@claude-plugins-official", "docker@claude-plugins-official"]
    }
  }
  ```

### PII 处理
- 插件名和市场名标记为 `_PROTO_*`（PII 标签）
- 仅进入受限访问的 BigQuery 列

## 风险、边界与改进建议

### 已知风险

1. **v1 官方市场硬编码限制**
   - 风险：仅支持官方市场插件提示，限制第三方生态
   - 现状：`isOfficialMarketplaceName(marketplace)` 检查
   - 计划：v2 可能放宽

2. **配置无界增长**
   - 风险：长期用户 `plugin` 数组持续增长
   - 缓解：`MAX_SHOWN_PLUGINS = 100` 硬上限
   - 局限：达到上限后停止提示，可能错过新推荐

3. **市场缓存依赖**
   - 风险：缓存未命中时提示被静默丢弃
   - 现状：记录调试日志，但用户无感知
   - 建议：考虑后台刷新市场缓存

4. **会话级去重内存泄漏**
   - 风险：`triedThisSession` Set 持续增长
   - 现状：受 `MAX_SHOWN_PLUGINS` 限制，实际风险低

### 边界条件

| 场景 | 行为 |
|------|------|
| hint.value 格式无效 | 静默丢弃 |
| 市场缓存未命中 | 记录日志，返回 null |
| 用户已禁用提示 | 所有后续提示丢弃 |
| 达到 MAX_SHOWN_PLUGINS | 停止添加新提示 |
| 同一会话重复 hint | 首次后丢弃 |
| 插件安装后卸载 | 仍在 shown 列表，不再提示 |

### 改进建议

1. **第三方市场支持**
   - 移除 `isOfficialMarketplaceName` 检查，或添加可信市场白名单

2. **智能上限管理**
   - 当前：硬截断
   - 建议：LRU 淘汰或时间衰减（如 90 天未见的插件移出列表）

3. **提示优先级**
   - 当前：FIFO
   - 建议：基于安装量、用户项目上下文排序

4. **离线模式处理**
   - 当前：缓存未命中即丢弃
   - 建议：队列化，网络恢复后重试

5. **A/B 测试支持**
   - 当前：单一功能开关
   - 建议：支持提示文案/时机的实验
