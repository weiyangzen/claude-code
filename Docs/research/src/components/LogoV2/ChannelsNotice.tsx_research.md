# ChannelsNotice.tsx 研究文档

## 场景与职责

`ChannelsNotice` 是 Claude Code 的 MCP (Model Context Protocol) Channels 功能的 UI 通知组件。该组件在以下场景中使用：

1. **启动时状态展示** - 当用户使用 `--channels` 或 `--dangerously-load-development-channels` 标志启动时显示通道状态
2. **配置验证反馈** - 显示通道配置的问题（未安装的插件、未配置的服务器等）
3. **权限状态提示** - 显示组织策略阻止、认证缺失等状态

组件设计遵循以下原则：
- 条件渲染：仅在 `feature('KAIROS') || feature('KAIROS_CHANNELS')` 启用时通过 `require()` 动态加载
- 用户友好：清晰解释为什么通道功能不可用
- 安全导向：明确区分开发通道和生产通道的风险

## 功能点目的

### 1. 多状态通道通知
组件处理以下状态：

| 状态 | 触发条件 | 显示内容 |
|------|----------|----------|
| 隐藏 | `channels.length === 0` | 不渲染 |
| 禁用 | `!isChannelsEnabled()` | 通道功能被全局禁用 |
| 未认证 | `!getClaudeAIOAuthTokens()?.accessToken` | 需要 claude.ai 认证 |
| 策略阻止 | `managed && policy?.channelsEnabled !== true` | 组织策略未启用通道 |
| 正常 | 以上都不满足 | 显示监听的通道列表和警告 |

### 2. 配置验证与问题提示
- **未匹配的插件**: 检查插件是否已安装
- **未匹配的服务器**: 检查 MCP 服务器是否已配置
- **白名单检查**: 验证插件是否在允许列表中

### 3. 动态标志名称
根据配置自动选择显示的标志名称：
- `--channels`: 仅使用开发通道
- `--dangerously-load-development-channels`: 混合使用开发和生产通道

## 具体技术实现

### 关键流程

```
挂载 → useState 初始化 → _temp() 收集状态
            ↓
    ┌───────┼───────┬───────────┬───────────┐
    ↓       ↓       ↓           ↓           ↓
  无通道  禁用    未认证      策略阻止      正常
    ↓       ↓       ↓           ↓           ↓
  返回    错误    错误提示    错误+详情    警告+列表
  null    提示    + /login    +未匹配项    +实验性警告
```

### 数据结构

```typescript
// 通道条目类型 (来自 bootstrap/state.js)
type ChannelEntry =
  | { kind: 'plugin'; name: string; marketplace: string; dev?: boolean }
  | { kind: 'server'; name: string; dev?: boolean };

// 内部状态类型
interface ChannelState {
  channels: ChannelEntry[];      // 允许的通道列表
  disabled: boolean;             // 功能是否被全局禁用
  noAuth: boolean;               // 是否缺少认证
  policyBlocked: boolean;        // 是否被组织策略阻止
  list: string;                  // 格式化的通道列表字符串
  unmatched: Unmatched[];        // 未匹配的通道项及原因
}

type Unmatched = {
  entry: ChannelEntry;
  why: string;
};
```

### 核心算法

**_temp() - 状态收集函数**:
```typescript
function _temp(): ChannelState {
  const ch = getAllowedChannels();
  if (ch.length === 0) {
    return { channels: ch, disabled: false, noAuth: false, 
             policyBlocked: false, list: "", unmatched: [] };
  }
  
  const l = ch.map(formatEntry).join(", ");
  const sub = getSubscriptionType();
  const managed = sub === "team" || sub === "enterprise";
  const policy = getSettingsForSource("policySettings");
  const allowlist = getEffectiveChannelAllowlist(sub, policy?.allowedChannelPlugins);
  
  return {
    channels: ch,
    disabled: !isChannelsEnabled(),
    noAuth: !getClaudeAIOAuthTokens()?.accessToken,
    policyBlocked: managed && policy?.channelsEnabled !== true,
    list: l,
    unmatched: findUnmatched(ch, allowlist)
  };
}
```

**findUnmatched() - 配置验证**:
```typescript
function findUnmatched(
  entries: readonly ChannelEntry[], 
  allowlist: ReturnType<typeof getEffectiveChannelAllowlist>
): Unmatched[] {
  // 1. 收集所有作用域的已配置服务器
  const scopes = ['enterprise', 'user', 'project', 'local'] as const;
  const configured = new Set<string>();
  for (const scope of scopes) {
    for (const name of Object.keys(getMcpConfigsByScope(scope).servers)) {
      configured.add(name);
    }
  }
  
  // 2. 收集已安装插件
  const installedPluginIds = new Set(
    Object.keys(loadInstalledPluginsV2().plugins)
  );
  
  // 3. 验证每个条目
  const out: Unmatched[] = [];
  for (const entry of entries) {
    if (entry.kind === 'server') {
      if (!configured.has(entry.name)) {
        out.push({ entry, why: 'no MCP server configured with that name' });
      }
      if (!entry.dev) {
        out.push({ entry, why: 'server: entries need --dangerously-load-development-channels' });
      }
    } else {
      // plugin kind
      if (!installedPluginIds.has(`${entry.name}@${entry.marketplace}`)) {
        out.push({ entry, why: 'plugin not installed' });
      }
      if (!entry.dev && !allowed.some(e => e.plugin === entry.name && e.marketplace === entry.marketplace)) {
        out.push({ entry, why: 'not on the approved channels allowlist' });
      }
    }
  }
  return out;
}
```

### 关键代码路径

1. **动态加载保护** (line 2-6):
   ```typescript
   // Conditionally require()'d in LogoV2.tsx behind feature('KAIROS') ||
   // feature('KAIROS_CHANNELS'). No feature() guard here — the whole file
   // tree-shakes via the require pattern when both flags are false
   ```

2. **标志名称选择** (line 33):
   ```typescript
   const flag = getHasDevChannels() && hasNonDev 
     ? "Channels" 
     : getHasDevChannels() 
       ? "--dangerously-load-development-channels" 
       : "--channels";
   ```

3. **策略阻止详情显示** (line 88-126):
   - 显示阻止原因
   - 列出未匹配的通道项
   - 提供管理员配置指导

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| ChannelEntry, getAllowedChannels, getHasDevChannels | `../../bootstrap/state.js` | 通道条目类型和状态获取 |
| Box, Text | `../../ink.js` | UI 组件 |
| isChannelsEnabled | `../../services/mcp/channelAllowlist.js` | 通道功能开关检查 |
| getEffectiveChannelAllowlist | `../../services/mcp/channelNotification.js` | 有效白名单获取 |
| getMcpConfigsByScope | `../../services/mcp/config.js` | MCP 配置获取 |
| getClaudeAIOAuthTokens, getSubscriptionType | `../../utils/auth.js` | 认证和订阅状态 |
| loadInstalledPluginsV2 | `../../utils/plugins/installedPluginsManager.js` | 已安装插件列表 |
| getSettingsForSource | `../../utils/settings/settings.js` | 策略设置读取 |

### 依赖详情

**通道功能开关** (`src/services/mcp/channelAllowlist.ts`):
- `isChannelsEnabled()`: 全局功能开关，来自 GrowthBook `tengu_harbor`
- `getChannelAllowlist()`: 批准的插件白名单

**通道通知系统** (`src/services/mcp/channelNotification.ts`):
- `getEffectiveChannelAllowlist()`: 获取有效的白名单（组织级别或 GrowthBook）
- `gateChannelServer()`: 服务器通道的门控逻辑

**MCP 配置** (`src/services/mcp/config.ts`):
- `getMcpConfigsByScope()`: 按作用域获取 MCP 服务器配置
- 支持的作用域: 'enterprise', 'user', 'project', 'local'

**认证系统** (`src/utils/auth.ts`):
- `getClaudeAIOAuthTokens()`: 获取 OAuth 令牌
- `getSubscriptionType()`: 获取订阅类型（team/enterprise/pro/max 等）

**插件管理** (`src/utils/plugins/installedPluginsManager.ts`):
- `loadInstalledPluginsV2()`: 加载已安装插件的 V2 格式数据

### 调用关系

```
LogoV2.tsx (条件 require)
    ↓
ChannelsNotice.tsx
    ├─→ bootstrap/state.js (通道状态)
    ├─→ services/mcp/channelAllowlist.js (功能开关)
    ├─→ services/mcp/channelNotification.js (白名单)
    ├─→ services/mcp/config.js (MCP 配置)
    ├─→ utils/auth.js (认证)
    ├─→ utils/plugins/installedPluginsManager.js (插件)
    └─→ utils/settings/settings.js (策略设置)
```

## 风险、边界与改进建议

### 已知风险

1. **冷缓存问题**: GrowthBook 缓存可能为空，导致所有插件显示白名单警告
   - 缓解: 这是已接受的权衡，见代码注释 "GrowthBook _CACHED_MAY_BE_STALE"

2. **作用域遍历性能**: `findUnmatched` 遍历所有 4 个作用域的 MCP 配置
   - 缓解: 项目作用域的目录遍历是主要开销，但通常在可接受范围内
   - 改进: 可考虑缓存结果

3. **重复警告**: 一个插件条目可能同时显示 "未安装" 和 "不在白名单" 两个警告
   - 缓解: 这是设计意图，独立 if 语句确保用户看到所有问题

### 边界情况

| 场景 | 行为 |
|------|------|
| 无通道配置 | 组件返回 null，不渲染 |
| 混合开发和生产通道 | 显示 "Channels" 标志名称 |
| 仅开发通道 | 显示 "--dangerously-load-development-channels" |
| 团队/企业组织未启用策略 | 显示策略阻止消息，提示管理员配置 |
| API Key 认证 | 显示需要 claude.ai 认证的消息 |

### 改进建议

1. **缓存优化**: `findUnmatched` 的结果可以在会话期间缓存，避免重复计算
   ```typescript
   // 可能的优化
   const unmatchedCache = useMemo(() => findUnmatched(ch, allowlist), [ch, allowlist]);
   ```

2. **国际化支持**: 当前所有提示文本都是硬编码的英文，可考虑添加 i18n 支持

3. **更详细的配置指导**: 对于策略阻止的情况，可以提供更详细的配置示例:
   ```json
   // 建议添加的示例
   {
     "channelsEnabled": true,
     "allowedChannelPlugins": [
       { "marketplace": "anthropic", "plugin": "slack" }
     ]
   }
   ```

4. **交互式修复**: 对于未安装插件的情况，可以提供一键安装按钮（如果 CLI 支持）

5. **测试覆盖**: 建议添加:
   - 各种状态组合的单元测试
   - `findUnmatched` 逻辑的边界情况测试
   - 不同订阅类型的行为测试

### 安全考虑

1. **开发通道风险**: 组件明确区分 `--channels` 和 `--dangerously-load-development-channels`，提醒用户开发通道的安全风险

2. **提示注入警告**: 正常状态下显示 "Experimental · inbound messages will be pushed into this session, this carries prompt injection risks"

3. **组织策略控制**: 团队/企业用户必须通过组织策略显式启用通道功能

### 相关文件引用

- 主实现: `src/components/LogoV2/ChannelsNotice.tsx`
- 调用方: `src/components/LogoV2/LogoV2.tsx`
- 通道白名单: `src/services/mcp/channelAllowlist.ts`
- 通道通知: `src/services/mcp/channelNotification.ts`
- MCP 配置: `src/services/mcp/config.ts`
- 全局状态: `src/bootstrap/state.ts`
