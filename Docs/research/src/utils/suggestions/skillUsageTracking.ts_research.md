# skillUsageTracking.ts 深度研究文档

## 场景与职责

`skillUsageTracking.ts` 是 Claude Code CLI 的**技能使用追踪与评分模块**，负责记录用户对各种技能（prompt 类型命令）的使用情况，并计算用于自动补全排序的使用评分。该模块是智能命令推荐系统的数据基础：

1. **使用记录**：记录技能每次被使用的时间和次数
2. **评分计算**：基于使用频率和时效性计算综合评分
3. **持久化存储**：将使用数据保存到全局配置中
4. **防抖处理**：避免高频使用导致频繁的磁盘写入

该模块直接影响命令自动补全的排序（最近使用命令优先显示），是提升用户体验的关键组件。

## 功能点目的

### 1. 技能使用记录 (`recordSkillUsage`)
- **目的**：记录技能被使用的事件
- **功能**：
  - 更新使用次数（+1）
  - 更新最后使用时间戳
  - 持久化到全局配置
- **防抖**：60 秒防抖，避免频繁写入

### 2. 使用评分计算 (`getSkillUsageScore`)
- **目的**：计算技能的使用评分，用于排序
- **算法**：指数衰减模型
  - 半衰期：7 天（7 天前的使用价值减半）
  - 最小因子：0.1（避免旧技能完全消失）
  - 公式：`usageCount * max(0.5^(days/7), 0.1)`

## 具体技术实现

### 关键数据结构

```typescript
// 全局配置中的技能使用数据（定义在 config.ts）
type GlobalConfig = {
  skillUsage?: Record<string, {
    usageCount: number   // 使用次数
    lastUsedAt: number   // 最后使用时间戳（毫秒）
  }>
  // ... 其他配置
}

// 内存防抖缓存
const SKILL_USAGE_DEBOUNCE_MS = 60_000  // 60 秒
const lastWriteBySkill = new Map<string, number>()  // skillName -> timestamp
```

### 核心算法流程

#### 1. 使用记录流程
```
recordSkillUsage(skillName)
├── 防抖检查
│   ├── 获取该技能上次写入时间
│   ├── 计算时间差
│   └── 差值 < 60 秒 → 直接返回（无操作）
├── 更新内存缓存
│   └── lastWriteBySkill.set(skillName, now)
└── 持久化到配置
    ├── 读取当前配置
    ├── 更新/创建 skillUsage[skillName]
    │   ├── usageCount = (existing?.usageCount ?? 0) + 1
    │   └── lastUsedAt = now
    └── saveGlobalConfig()
```

#### 2. 评分计算流程
```
getSkillUsageScore(skillName)
├── 读取全局配置
├── 获取技能使用数据
│   └── 不存在 → 返回 0
├── 计算时间衰减
│   ├── daysSinceUse = (now - lastUsedAt) / (1000 * 60 * 60 * 24)
│   ├── recencyFactor = 0.5 ^ (daysSinceUse / 7)  // 7 天半衰期
│   └── clampedFactor = max(recencyFactor, 0.1)   // 最小 0.1
└── 返回评分
    └── usageCount * clampedFactor
```

### 评分算法详解

```typescript
// 示例计算：
// 技能 A：使用 10 次，最后使用 1 天前
// score = 10 * max(0.5^(1/7), 0.1) ≈ 10 * 0.906 = 9.06

// 技能 B：使用 5 次，最后使用 7 天前
// score = 5 * max(0.5^(7/7), 0.1) = 5 * 0.5 = 2.5

// 技能 C：使用 100 次，最后使用 30 天前
// score = 100 * max(0.5^(30/7), 0.1) = 100 * 0.1 = 10
```

### 关键代码路径

| 功能 | 函数 | 行号 |
|------|------|------|
| 记录使用 | `recordSkillUsage` | 13-35 |
| 获取评分 | `getSkillUsageScore` | 44-55 |

### 配置持久化

```typescript
// 使用 saveGlobalConfig 进行原子更新
saveGlobalConfig(current => {
  const existing = current.skillUsage?.[skillName]
  return {
    ...current,
    skillUsage: {
      ...current.skillUsage,
      [skillName]: {
        usageCount: (existing?.usageCount ?? 0) + 1,
        lastUsedAt: now,
      },
    },
  }
})
```

## 依赖与外部交互

### 直接依赖模块

```typescript
import { getGlobalConfig, saveGlobalConfig } from '../config.js'  // 全局配置
```

### 依赖详解

1. **config.ts** (内部模块)
   - `getGlobalConfig()`: 读取全局配置（内存缓存）
   - `saveGlobalConfig()`: 原子更新全局配置
   - 配置存储在 `~/.claude.json`
   - 包含 `skillUsage` 字段记录技能使用数据

### 调用方

- `commandSuggestions.ts`: 
  - `generateCommandSuggestions`: 获取最近使用命令列表
  - Fuse 结果排序时作为平局决胜因素
- 命令执行逻辑（在命令被成功执行后调用 `recordSkillUsage`）

### 配置结构

```json
{
  "skillUsage": {
    "git-commit": {
      "usageCount": 15,
      "lastUsedAt": 1712345678901
    },
    "code-review": {
      "usageCount": 3,
      "lastUsedAt": 1712000000000
    }
  }
}
```

## 风险、边界与改进建议

### 已知风险

1. **防抖粒度较粗**
   - 60 秒防抖意味着短时间内多次使用只记录一次
   - **影响**：使用频率统计可能偏低
   - **权衡**：减少磁盘 I/O，提升性能

2. **配置膨胀**
   - 长期使用会积累大量技能使用记录
   - **当前处理**：无清理机制
   - **风险**：`~/.claude.json` 文件变大

3. **时间精度问题**
   - 仅记录最后使用时间，无使用历史
   - **影响**：无法分析使用模式（如每天使用频率）

4. **跨设备同步**
   - 配置存储在本地，不同设备间不共享
   - **影响**：换设备后技能推荐需要重新学习

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| 首次使用技能 | 创建新记录，usageCount = 1 |
| 防抖期间使用 | 忽略，不更新记录 |
| 配置读取失败 | 依赖 config.ts 的错误处理 |
| 配置写入失败 | 依赖 config.ts 的错误处理 |
| 负时间差（时钟回拨） | 可能产生异常评分（未处理） |
| 技能名称为空 | 正常处理（但无实际意义） |

### 改进建议

1. **算法优化**
   - 考虑使用更复杂的衰减模型（如指数加权移动平均 EWMA）
   - 添加使用场景权重（如成功执行 vs 仅查看）
   - 区分主动使用和被动调用（模型调用 vs 用户输入）

2. **数据清理**
   - 定期清理长期未使用的技能记录（如 1 年）
   - 限制记录数量（如最多 1000 个技能）
   - 提供用户手动清除的选项

3. **功能扩展**
   - 记录更多维度（使用时长、成功率等）
   - 支持技能使用的时间序列分析
   - 添加技能推荐（基于使用模式预测）

4. **持久化优化**
   - 批量写入（收集多个更新后一次性写入）
   - 使用独立的数据库/文件存储使用数据
   - 支持云同步（可选）

5. **可配置性**
   - 允许用户调整半衰期
   - 可配置的防抖时间
   - 开关技能追踪功能

6. **隐私考虑**
   - 敏感技能名称的哈希处理
   - 使用数据的本地加密
   - 提供不追踪选项

### 测试要点

- 防抖逻辑的正确性（60 秒内多次调用）
- 评分计算的准确性（各种时间差场景）
- 配置读写的错误处理
- 时钟回拨场景的处理
- 大量技能记录的内存和性能表现
- 并发调用的安全性

### 相关代码参考

- `config.ts`: 全局配置的完整实现
- `commandSuggestions.ts`: 使用评分的消费者
- `types/command.ts`: Command 类型定义
