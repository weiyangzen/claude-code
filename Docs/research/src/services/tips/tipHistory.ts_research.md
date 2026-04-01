# tipHistory.ts 研究文档

## 场景与职责

`tipHistory.ts` 是 Claude Code 提示系统（Tips System）的基础模块，负责管理提示的展示历史记录。它提供了两个核心功能：

1. **记录提示展示**：当某个提示被展示给用户时，记录该提示的 ID 和当前会话计数
2. **查询冷却状态**：计算自上次展示某个提示以来经过了多少个会话，用于控制提示的展示频率

该模块是提示系统的"记忆中枢"，确保用户不会在短时间内重复看到相同的提示，同时保证提示能够按合理的间隔循环展示。

## 功能点目的

### 1. `recordTipShown(tipId: string): void`

**目的**：在提示被展示给用户时，将提示 ID 和当前 `numStartups` 值记录到全局配置中。

**工作机制**：
- 读取当前全局配置中的 `numStartups`（会话启动计数）
- 使用 `saveGlobalConfig` 将提示 ID 和当前 `numStartups` 存入 `tipsHistory` 对象
- 如果该提示已经在当前会话中记录过（`history[tipId] === numStartups`），则跳过写入以避免重复

**数据结构**：
```typescript
// 存储在 ~/.claude.json 中的格式
tipsHistory: {
  [tipId: string]: number  // key 是提示 ID，value 是展示时的 numStartups
}
```

### 2. `getSessionsSinceLastShown(tipId: string): number`

**目的**：计算自上次展示指定提示以来经过了多少个会话。

**返回值**：
- `Infinity`：如果该提示从未展示过（新用户或新提示）
- `number`：自上次展示以来经过的会话数（当前 `numStartups` - 上次记录的 `numStartups`）

**使用场景**：
- 在 `tipRegistry.ts` 的 `getRelevantTips` 函数中，用于筛选满足 `cooldownSessions` 条件的提示
- 在 `tipScheduler.ts` 的 `selectTipWithLongestTimeSinceShown` 函数中，用于选择最久未展示的提示

## 具体技术实现

### 关键流程

```
用户看到提示 → recordTipShown(tipId) → 写入 tipsHistory[tipId] = numStartups
                                    ↓
下次选择提示 → getSessionsSinceLastShown(tipId) → 返回 numStartups - tipsHistory[tipId]
```

### 数据结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `tipsHistory` | `Record<string, number>` | 存储在 `GlobalConfig` 中的提示历史 |
| `tipId` | `string` | 提示的唯一标识符，如 `"new-user-warmup"` |
| `numStartups` | `number` | 用户启动 Claude Code 的总次数 |

### 核心算法

**防重复写入优化**：
```typescript
if (history[tipId] === numStartups) return c  // 同一会话中已记录，跳过
```

这个检查避免了在同一会话中多次展示同一提示时产生不必要的配置写入。

## 关键代码路径与文件引用

### 导出函数

| 函数 | 导出类型 | 被引用文件 |
|------|----------|------------|
| `recordTipShown` | 命名导出 | `tipScheduler.ts` |
| `getSessionsSinceLastShown` | 命名导出 | `tipRegistry.ts`, `tipScheduler.ts` |

### 依赖关系

```
tipHistory.ts
├── 导入: getGlobalConfig, saveGlobalConfig from '../../utils/config.js'
├── 被 tipScheduler.ts 导入
├── 被 tipRegistry.ts 导入（仅 getSessionsSinceLastShown）
└── 被 REPL.tsx 间接使用（通过 tipScheduler）
```

### 配置项位置

- **配置文件**：`~/.claude.json`
- **配置键**：`tipsHistory`（对象类型）
- **默认值**：`{}`（空对象）

## 依赖与外部交互

### 依赖模块

| 模块路径 | 导入内容 | 用途 |
|----------|----------|------|
| `../../utils/config.js` | `getGlobalConfig`, `saveGlobalConfig` | 读取和写入全局配置 |

### 与配置系统的交互

`tipHistory.ts` 是 `GlobalConfig` 的消费者和生产者：

1. **读取**：通过 `getGlobalConfig()` 获取当前 `numStartups` 和 `tipsHistory`
2. **写入**：通过 `saveGlobalConfig()` 更新 `tipsHistory`

### 与会话计数的关系

`numStartups` 在 `main.tsx` 中每次启动时递增：

```typescript
// main.tsx ~3044
saveGlobalConfig(current => ({
  ...current,
  numStartups: (current.numStartups ?? 0) + 1
}));
```

## 风险、边界与改进建议

### 潜在风险

1. **配置写入频率**：虽然做了防重复检查，但如果提示展示逻辑有 bug，可能导致频繁的配置写入
2. **历史记录无限增长**：`tipsHistory` 对象会随着展示过的提示数量增长，但通常提示数量有限（几十个），影响可忽略
3. **并发写入**：`saveGlobalConfig` 内部有锁机制，但极端情况下仍可能存在竞态条件

### 边界情况

| 场景 | 行为 |
|------|------|
| 提示从未展示过 | `getSessionsSinceLastShown` 返回 `Infinity` |
| 同一会话中重复展示 | `recordTipShown` 检测到相同 `numStartups`，跳过写入 |
| `tipsHistory` 未定义 | 使用空对象 `{}` 作为默认值 |
| 配置损坏/重置 | 历史记录丢失，提示会重新从冷却期开始计算 |

### 改进建议

1. **历史记录清理**：考虑添加机制清理长期未展示的提示历史，防止配置文件膨胀
   ```typescript
   // 建议：只保留最近 N 个会话的提示历史
   const MAX_HISTORY_AGE = 100;  // 保留最近 100 个会话
   ```

2. **更细粒度的时间控制**：当前使用会话数（`numStartups`）作为冷却单位，可考虑支持基于实际时间的冷却（如"7天内不重复"）

3. **按用户/项目隔离**：当前提示历史是全局的，可考虑支持项目级别的提示历史

4. **添加调试日志**：在开发和测试环境中，可以添加日志记录提示的展示和筛选过程

### 测试注意事项

- 测试 `numStartups` 递增时历史记录的正确性
- 测试同一会话内多次调用 `recordTipShown` 的防重复行为
- 测试 `tipsHistory` 为 `undefined` 时的容错处理
