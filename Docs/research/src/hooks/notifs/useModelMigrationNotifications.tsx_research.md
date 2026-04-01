# useModelMigrationNotifications.tsx 深度研究

## 场景与职责

`useModelMigrationNotifications` 是一个 React Hook，用于在模型迁移完成后向用户显示一次性通知。当 Claude Code 自动将用户的模型设置从旧版本迁移到新版本时（如 Sonnet 4.5 → 4.6），此 Hook 会检测迁移时间戳并显示相应的通知。

### 核心场景
1. **模型版本升级通知**：当用户的默认模型自动升级时通知用户
2. **遗留模型重映射通知**：当用户的旧版模型设置被映射到新版时通知用户
3. **迁移后即时提示**：仅在迁移发生后的 3 秒内显示通知（即当前启动时）
4. **可扩展架构**：支持添加未来的模型迁移通知

## 功能点目的

### 1. 迁移检测机制
- 检查全局配置中的迁移时间戳字段
- 如果时间戳在最近 3 秒内，认为迁移刚刚发生
- 每个迁移定义独立的检测逻辑和时间戳字段

### 2. 支持的迁移类型
当前支持两种模型迁移：

#### Sonnet 4.5 → 4.6 迁移
- 时间戳字段：`sonnet45To46MigrationTimestamp`
- 通知内容："Model updated to Sonnet 4.6"
- 颜色：suggestion（建议/提示色）
- 优先级：high
- 超时：3 秒

#### Opus Pro → 默认/Opus 4.6 迁移
- 时间戳字段：`legacyOpusMigrationTimestamp` 或 `opusProMigrationTimestamp`
- 区分遗留重映射和普通迁移
- 遗留重映射显示额外的环境变量提示
- 颜色：suggestion
- 优先级：high
- 超时：3 秒（普通）或 8 秒（遗留，因为包含更多信息）

### 3. 启动时一次性执行
- 使用 `useStartupNotification` 确保只在启动时检查
- 避免会话期间的重复检查

### 4. 可扩展设计
- 使用 `MIGRATIONS` 数组定义所有迁移
- 新迁移只需添加新的数组项
- 每个迁移独立定义检测逻辑和通知内容

## 具体技术实现

### 关键数据结构

```typescript
// 通知类型
interface Notification {
  key: string
  text: string
  color: 'suggestion' | 'warning' | 'error' | 'text'
  priority: 'low' | 'medium' | 'high' | 'immediate'
  timeoutMs?: number
}

// 全局配置中的迁移时间戳
type GlobalConfig = {
  sonnet45To46MigrationTimestamp?: number
  legacyOpusMigrationTimestamp?: number
  opusProMigrationTimestamp?: number
}

// 迁移函数类型
type Migration = (config: GlobalConfig) => Notification | undefined
```

### 核心流程

```
useStartupNotification 初始化
    ↓
获取全局配置 (getGlobalConfig)
    ↓
遍历 MIGRATIONS 数组
    对每个迁移函数：
        - 传入配置
        - 如果返回通知对象，添加到结果数组
    ↓
如果有通知，返回通知数组
否则返回 null
```

### 关键代码路径

```typescript
// 迁移定义数组
const MIGRATIONS: Migration[] = [
  // Sonnet 4.5 → 4.6 (pro/max/team premium)
  c => {
    if (!recent(c.sonnet45To46MigrationTimestamp)) return
    return {
      key: 'sonnet-46-update',
      text: 'Model updated to Sonnet 4.6',
      color: 'suggestion',
      priority: 'high',
      timeoutMs: 3000
    }
  },
  
  // Opus Pro → default, or pinned 4.0/4.1 → opus alias
  c => {
    const isLegacyRemap = Boolean(c.legacyOpusMigrationTimestamp)
    const ts = c.legacyOpusMigrationTimestamp ?? c.opusProMigrationTimestamp
    if (!recent(ts)) return
    return {
      key: 'opus-pro-update',
      text: isLegacyRemap 
        ? 'Model updated to Opus 4.6 · Set CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP=1 to opt out'
        : 'Model updated to Opus 4.6',
      color: 'suggestion',
      priority: 'high',
      timeoutMs: isLegacyRemap ? 8000 : 3000
    }
  }
]

// 主 Hook
export function useModelMigrationNotifications() {
  useStartupNotification(() => {
    const config = getGlobalConfig()
    const notifs: Notification[] = []
    
    for (const migration of MIGRATIONS) {
      const notif = migration(config)
      if (notif) {
        notifs.push(notif)
      }
    }
    
    return notifs.length > 0 ? notifs : null
  })
}

// 时间戳检查函数
function recent(ts: number | undefined): boolean {
  return ts !== undefined && Date.now() - ts < 3000
}
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `Notification` | `src/context/notifications.js` | 通知类型 |
| `GlobalConfig`, `getGlobalConfig` | `src/utils/config.js` | 全局配置类型和读取 |
| `useStartupNotification` | `./useStartupNotification.js` | 启动通知基类 |

### 依赖模块详解

#### 1. useStartupNotification (src/hooks/notifs/useStartupNotification.ts)
提供启动时一次性通知的基础设施：
- 远程模式检查
- 单次执行保证（useRef）
- 异步计算支持
- 错误处理

#### 2. 全局配置 (src/utils/config.ts)
存储迁移时间戳：
```typescript
type GlobalConfig = {
  // ... 其他配置
  sonnet45To46MigrationTimestamp?: number
  legacyOpusMigrationTimestamp?: number
  opusProMigrationTimestamp?: number
}
```

时间戳由迁移逻辑在模型变更时写入，表示迁移发生的时间。

#### 3. 迁移执行逻辑（其他文件）
虽然不在此文件中，但迁移时间戳由以下迁移模块写入：
- `src/migrations/migrateSonnet45ToSonnet46.ts`
- `src/migrations/migrateOpusToOpus1m.ts`
- `src/migrations/migrateLegacyOpusToCurrent.ts`

## 风险、边界与改进建议

### 潜在风险

1. **时间戳精度问题**
   - 使用 `Date.now() - ts < 3000` 判断迁移是否"最近"
   - 如果应用启动较慢（超过 3 秒），可能错过通知
   - 建议：增加更长的窗口或持久化"已显示"状态

2. **多迁移同时触发**
   - 如果多个迁移同时发生，会显示多个通知
   - 可能给用户造成信息过载

3. **时区问题**
   - `Date.now()` 使用本地时间
   - 如果迁移时间戳由服务器生成，可能存在时区差异

4. **配置读取失败**
   - 如果 `getGlobalConfig()` 失败，整个 Hook 会失败
   - 由 `useStartupNotification` 捕获，但用户看不到通知

### 边界情况

1. **快速重启**
   - 如果用户在 3 秒内重启应用，会再次看到通知
   - 这是预期行为（提醒用户迁移发生）

2. **时间戳为 0 或负数**
   - `recent()` 函数会正确处理（返回 false）

3. **未定义的迁移**
   - 如果添加新的迁移类型但用户配置中没有对应时间戳
   - 迁移函数返回 undefined，不产生通知

4. **空迁移数组**
   - 如果 `MIGRATIONS` 为空数组
   - 返回 null，不产生通知

### 改进建议

1. **延长检测窗口**
   ```typescript
   // 当前 3 秒可能太短
   const MIGRATION_WINDOW_MS = 10000 // 10 秒
   function recent(ts: number | undefined): boolean {
     return ts !== undefined && Date.now() - ts < MIGRATION_WINDOW_MS
   }
   ```

2. **添加已显示追踪**
   ```typescript
   // 使用 sessionStorage 或全局状态追踪已显示的迁移
   const shownMigrations = useRef<Set<string>>(new Set())
   
   // 在显示前检查
   if (notif && !shownMigrations.current.has(notif.key)) {
     shownMigrations.current.add(notif.key)
     notifs.push(notif)
   }
   ```

3. **添加分析事件**
   ```typescript
   if (notifs.length > 0) {
     logEvent('tengu_model_migration_notified', {
       migrations: notifs.map(n => n.key)
     })
   }
   ```

4. **支持更多元数据**
   ```typescript
   // 考虑在通知中包含更多信息
   interface MigrationNotification extends Notification {
     oldModel?: string
     newModel?: string
     migrationType: 'upgrade' | 'remap' | 'deprecation'
   }
   ```

5. **考虑使用配置标志而非时间戳**
   ```typescript
   // 替代方案：使用布尔标志
   type GlobalConfig = {
     hasShownSonnet46Migration?: boolean
   }
   ```

6. **添加迁移详情链接**
   ```typescript
   // 在通知中添加链接到文档
   text: 'Model updated to Sonnet 4.6 · See /help model-migration'
   ```

### 相关文件引用

- **实现文件**: `src/hooks/notifs/useModelMigrationNotifications.tsx`
- **启动通知基类**: `src/hooks/notifs/useStartupNotification.ts`
- **全局配置**: `src/utils/config.ts`
- **Sonnet 4.5→4.6 迁移**: `src/migrations/migrateSonnet45ToSonnet46.ts`
- **Opus 迁移**: `src/migrations/migrateOpusToOpus1m.ts`
- **遗留 Opus 迁移**: `src/migrations/migrateLegacyOpusToCurrent.ts`
