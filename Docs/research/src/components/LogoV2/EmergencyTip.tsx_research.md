# EmergencyTip.tsx 研究文档

## 场景与职责

`EmergencyTip` 是 Claude Code 终端 UI 中的紧急提示组件，用于向用户显示来自远程配置的重要通知或提示。该组件在以下场景中使用：

1. **启动时重要通知** - 显示来自运营团队的紧急消息（如服务中断、安全公告等）
2. **动态配置推送** - 通过 GrowthBook 远程配置向特定用户群推送提示
3. **一次性提示** - 确保同一提示不会重复显示给用户

组件设计遵循以下原则：
- 非侵入式：仅在有必要内容时显示
- 防重复：记录已显示的提示，避免用户困扰
- 可配置：支持多种颜色主题（dim/warning/error）
- 动态更新：支持远程配置实时更新

## 功能点目的

### 1. 远程配置驱动的提示
- **配置源**: GrowthBook 动态配置 `tengu-top-of-feed-tip`
- **实时性**: 支持紧急情况下快速向用户推送通知
- **目标定位**: 可通过 GrowthBook 的受众定向功能向特定用户群推送

### 2. 防重复显示机制
- **记录机制**: 将已显示的提示内容保存到全局配置
- **比较逻辑**: 仅当新提示与上次显示的不同时才显示
- **持久化**: 使用 `saveGlobalConfig` 确保跨会话记忆

### 3. 多级颜色支持
支持三种颜色级别，用于不同紧急程度：

| 颜色 | 用途 | 场景示例 |
|------|------|----------|
| `dim` | 普通提示 | 功能更新、使用技巧 |
| `warning` | 警告 | 即将弃用的功能、计划维护 |
| `error` | 错误/紧急 | 服务中断、安全漏洞 |

### 4. 缓存优化
- 使用 `useMemo` 缓存提示内容，避免重复读取
- 使用 `getDynamicConfig_CACHED_MAY_BE_STALE` 获取缓存配置
- 平衡实时性和性能

## 具体技术实现

### 关键流程

```
挂载
  ↓
useMemo: getTipOfFeed() → 获取远程配置
  ↓
useMemo: getGlobalConfig().lastShownEmergencyTip → 获取上次显示的提示
  ↓
比较: tip.tip !== lastShownTip → 决定是否显示
  ↓
[shouldShow = true] → useEffect: saveGlobalConfig 记录本次提示
  ↓
渲染: 根据 tip.color 应用对应样式
```

### 数据结构

```typescript
// 提示数据结构
type TipOfFeed = {
  tip: string;                    // 提示文本内容
  color?: 'dim' | 'warning' | 'error';  // 可选的颜色级别
};

// 默认配置
const DEFAULT_TIP: TipOfFeed = {
  tip: '',
  color: 'dim'
};

// 配置常量
const CONFIG_NAME = 'tengu-top-of-feed-tip';
```

### 核心算法

**提示获取**:
```typescript
function getTipOfFeed(): TipOfFeed {
  return getDynamicConfig_CACHED_MAY_BE_STALE<TipOfFeed>(CONFIG_NAME, DEFAULT_TIP);
}
```

**显示决策**:
```typescript
const tip = useMemo(getTipOfFeed, []);
const lastShownTip = useMemo(() => getGlobalConfig().lastShownEmergencyTip, []);
const shouldShow = tip.tip && tip.tip !== lastShownTip;
```

**持久化记录**:
```typescript
useEffect(() => {
  if (shouldShow) {
    saveGlobalConfig(current => {
      if (current.lastShownEmergencyTip === tip.tip) return current;
      return { ...current, lastShownEmergencyTip: tip.tip };
    });
  }
}, [shouldShow, tip.tip]);
```

**颜色应用**:
```typescript
<Text {
  ...tip.color === 'warning' ? { color: 'warning' }
    : tip.color === 'error' ? { color: 'error' }
    : { dimColor: true }
}>
  {tip.tip}
</Text>
```

### 关键代码路径

1. **配置读取** (line 8):
   ```typescript
   const tip = useMemo(getTipOfFeed, []);
   ```
   - 使用 `useMemo` 确保只在挂载时读取一次
   - 使用 `_CACHED_MAY_BE_STALE` 变体避免阻塞

2. **上次记录读取** (line 10):
   ```typescript
   const lastShownTip = useMemo(() => getGlobalConfig().lastShownEmergencyTip, []);
   ```
   - 同样使用 `useMemo` 缓存
   - 注释明确说明："Memoize to prevent re-reads after we save"

3. **显示条件判断** (line 13):
   ```typescript
   const shouldShow = tip.tip && tip.tip !== lastShownTip;
   ```
   - 双重检查：内容非空且与上次不同

4. **持久化保存** (line 16-26):
   ```typescript
   useEffect(() => {
     if (shouldShow) {
       saveGlobalConfig(current => {
         if (current.lastShownEmergencyTip === tip.tip) return current;
         return { ...current, lastShownEmergencyTip: tip.tip };
       });
     }
   }, [shouldShow, tip.tip]);
   ```
   - 使用函数式更新避免竞态条件
   - 重复检查确保不重复写入

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| Box, Text | `src/ink.js` | UI 组件 |
| getDynamicConfig_CACHED_MAY_BE_STALE | `src/services/analytics/growthbook.js` | 远程配置获取 |
| getGlobalConfig, saveGlobalConfig | `src/utils/config.js` | 本地配置读写 |

### 依赖详情

**`getDynamicConfig_CACHED_MAY_BE_STALE`** (`src/services/analytics/growthbook.ts`):
- 从 GrowthBook 获取动态配置
- 使用缓存优先策略，非阻塞
- 支持类型安全的默认值

**`getGlobalConfig` / `saveGlobalConfig`** (`src/utils/config.ts`):
- 读写 `~/.claude.json` 配置文件
- 自动处理并发写入和文件锁定
- 支持函数式更新模式

**全局配置扩展** (`src/utils/config.ts` line 456):
```typescript
lastShownEmergencyTip?: string;  // 存储上次显示的紧急提示
```

### 调用关系

```
LogoV2.tsx / Feed.tsx
    ↓
EmergencyTip.tsx
    ├─→ src/ink.js (Box, Text)
    ├─→ src/services/analytics/growthbook.js (远程配置)
    └─→ src/utils/config.js (本地配置)
```

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**: 多个组件实例同时尝试更新 `lastShownEmergencyTip`
   - 缓解: `saveGlobalConfig` 使用文件锁和函数式更新
   - 残余风险: 极端并发下仍可能丢失更新

2. **配置传播延迟**: GrowthBook 缓存可能导致紧急提示延迟显示
   - 缓解: 使用 `_CACHED_MAY_BE_STALE` 快速获取缓存值
   - 残余风险: 最紧急情况仍可能有 5 分钟延迟

3. **空内容显示**: 如果配置返回 `{ tip: "" }`，组件正确返回 null
   - 缓解: `shouldShow` 检查 `tip.tip` 真值

4. **过长内容**: 没有长度限制，极端长的提示可能破坏布局
   - 缓解: 依赖运营团队配置时的审核
   - 改进: 可添加最大长度截断

### 边界情况

| 场景 | 行为 |
|------|------|
| tip.tip = "" | 返回 null，不渲染 |
| tip.tip = lastShownTip | 返回 null，不重复显示 |
| tip.color = undefined | 使用 dimColor（默认） |
| tip.color = 无效值 | 使用 dimColor（默认） |
| 配置读取失败 | 使用 DEFAULT_TIP（空内容），不显示 |
| 配置保存失败 | 提示仍显示，但下次可能重复 |

### 改进建议

1. **内容长度限制**: 添加最大长度检查，防止超长提示破坏布局
   ```typescript
   const MAX_TIP_LENGTH = 200;
   const shouldShow = tip.tip && 
                      tip.tip !== lastShownTip && 
                      tip.tip.length <= MAX_TIP_LENGTH;
   ```

2. **过期机制**: 添加提示过期时间，避免显示过期的紧急通知
   ```typescript
   type TipOfFeed = {
     tip: string;
     color?: 'dim' | 'warning' | 'error';
     expiresAt?: number;  // 新增：过期时间戳
   };
   
   const shouldShow = tip.tip && 
                      tip.tip !== lastShownTip &&
                      (!tip.expiresAt || Date.now() < tip.expiresAt);
   ```

3. **显示次数限制**: 某些提示可能需要限制显示次数而非仅一次
   ```typescript
   // 配置扩展
   type TipOfFeed = {
     tip: string;
     maxShows?: number;  // 最大显示次数
   };
   
   // 全局配置扩展
   tipShowCount?: Record<string, number>;
   ```

4. **操作按钮**: 对于某些提示，可添加操作按钮（如 "了解更多"、"立即更新"）
   ```typescript
   type TipOfFeed = {
     tip: string;
     action?: {
       label: string;
       command: string;  // 如 "/changelog"
     };
   };
   ```

5. **本地化支持**: 当前仅支持英文，可考虑添加多语言支持
   ```typescript
   type TipOfFeed = {
     tip: string;
     tip_zh?: string;
     tip_ja?: string;
     // ...
   };
   ```

6. **测试覆盖**: 建议添加:
   - 不同颜色级别的渲染测试
   - 防重复逻辑测试
   - 配置读取失败回退测试

### 安全考虑

1. **XSS 防护**: 提示内容直接渲染为文本，不解释 HTML/JSX
   - 当前实现是安全的，因为使用 Ink 的 Text 组件
   - 如果未来支持格式化，需要转义处理

2. **配置注入**: GrowthBook 配置被篡改可能导致恶意提示
   - 缓解: 限制提示内容长度
   - 缓解: 仅显示文本，不支持链接或命令

3. **隐私**: 提示内容可能包含用户敏感信息
   - 建议: 运营团队配置时避免包含用户特定信息
   - 建议: 使用 GrowthBook 的受众定向而非内容模板

### 相关文件引用

- 主实现: `src/components/LogoV2/EmergencyTip.tsx`
- 调用方: `src/components/LogoV2/LogoV2.tsx` (推测)
- 远程配置: `src/services/analytics/growthbook.ts`
- 本地配置: `src/utils/config.ts`
