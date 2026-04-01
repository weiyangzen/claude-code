# managedPlugins.ts 深度研究文档

## 场景与职责

`managedPlugins.ts` 是 Claude Code 插件系统中负责**管理组织策略控制的插件**的简单模块。它从企业/组织策略设置中提取被锁定的插件名称，用于在插件管理 UI 中显示锁定状态，防止用户修改组织强制配置的插件。

### 核心职责
1. **策略插件提取**：从 `policySettings.enabledPlugins` 中提取组织管理的插件名称
2. **锁定状态标识**：识别哪些插件被组织策略锁定（用户无法禁用/卸载）
3. **策略存在检测**：判断是否存在有效的组织策略配置

### 使用场景
- 企业环境：IT 管理员通过远程管理设置强制启用某些插件
- 锁定显示：在 `/plugins` UI 中显示锁定图标，提示用户无法修改
- 权限控制：阻止用户对锁定插件执行禁用/卸载操作

---

## 功能点目的

### 1. 管理插件名称获取（`getManagedPluginNames`）
- **目的**：获取被组织策略锁定的插件名称集合
- **输入**：`policySettings.enabledPlugins` 配置
- **输出**：`Set<string>` 或 `null`
- **返回 null 的情况**：
  - `enabledPlugins` 不存在（无策略配置）
  - 没有有效的 `plugin@marketplace` 格式条目
  - 处理后集合为空

### 2. 插件标识符解析
- **支持的格式**：`pluginName@marketplaceName`
- **提取逻辑**：
  1. 检查值为布尔类型（`true` 或 `false`）
  2. 检查键包含 `@` 符号
  3. 分割字符串，取 `@` 前部分作为插件名称
- **排除的格式**：
  - 遗留的 `owner/repo` 数组格式
  - 非布尔值条目
  - 不包含 `@` 的键

---

## 具体技术实现

### 关键数据类型

```typescript
// 函数签名
function getManagedPluginNames(): Set<string> | null

// 输入数据类型（来自 settings）
type EnabledPlugins = Record<string, boolean | string[]>
// 示例：
// {
//   "my-plugin@official-marketplace": true,    // ✓ 被包含
//   "another@custom-marketplace": false,       // ✓ 被包含（false 也被锁定）
//   "owner/repo": ["plugin1", "plugin2"],      // ✗ 被排除（遗留格式）
//   "invalid-entry": "some-value"              // ✗ 被排除（非布尔值）
// }
```

### 核心函数实现

```typescript
export function getManagedPluginNames(): Set<string> | null {
  // 从 policySettings 获取 enabledPlugins
  const enabledPlugins = getSettingsForSource('policySettings')?.enabledPlugins
  
  // 无策略配置，返回 null（最常见情况）
  if (!enabledPlugins) {
    return null
  }
  
  const names = new Set<string>()
  
  // 遍历所有条目
  for (const [pluginId, value] of Object.entries(enabledPlugins)) {
    // 筛选条件：
    // 1. 值必须是布尔类型（true OR false）
    // 2. 键必须包含 @（plugin@marketplace 格式）
    if (typeof value !== 'boolean' || !pluginId.includes('@')) {
      continue
    }
    
    // 提取插件名称（@ 前的部分）
    const name = pluginId.split('@')[0]
    if (name) {
      names.add(name)
    }
  }
  
  // 返回集合，如果为空则返回 null
  return names.size > 0 ? names : null
}
```

### 执行流程

```
输入: 无（从全局设置获取）
  ↓
获取 policySettings.enabledPlugins
  ├─ 不存在: 返回 null
  └─ 存在: 继续处理
  ↓
初始化空 Set
  ↓
遍历 enabledPlugins 条目
  ├─ 值不是布尔类型: 跳过
  ├─ 键不包含 @: 跳过
  └─ 通过筛选: 提取名称并添加到 Set
  ↓
返回 Set（非空）或 null（空）
```

---

## 关键代码路径与文件引用

### 入口函数
| 函数 | 行号 | 说明 |
|------|------|------|
| `getManagedPluginNames` | 9-27 | 获取被策略管理的插件名称集合 |

### 依赖的文件与模块

```typescript
// 核心依赖
import { getSettingsForSource } from '../settings/settings.js'

// 类型依赖（推断）
import type { SettingSource } from '../settings/constants.js'
```

### 调用方
- `src/utils/plugins/pluginLoader.ts`：加载插件时检查是否被策略管理
- `src/commands/plugin/ManagePlugins.tsx`：插件管理 UI，显示锁定状态
- `src/hooks/useManagePlugins.ts`：管理插件状态的 Hook

### 策略设置来源
- **Source**：`'policySettings'`
- **配置项**：`enabledPlugins`
- **设置方式**：远程管理配置（企业/组织策略）

---

## 依赖与外部交互

### 与 settings/settings.ts 的交互
- **调用**：`getSettingsForSource('policySettings')` 获取策略设置
- **数据**：返回 `policySettings` 作用域的设置对象
- **字段访问**：`?.enabledPlugins` 可选链访问

### 与 settings/constants.ts 的交互
- **类型**：`SettingSource` 包含 `'policySettings'`

### 与 pluginLoader.ts 的交互
- **消费**：`getManagedPluginNames()` 结果用于标记插件的锁定状态
- **影响**：锁定插件不能被用户禁用或卸载

---

## 风险、边界与改进建议

### 已知风险

1. **格式限制严格**
   - 风险：只接受 `plugin@marketplace` 布尔格式，遗留格式被排除
   - 影响：使用旧版配置的企业可能无法正确识别管理插件
   - 代码注释："Legacy owner/repo array form is not."

2. **名称冲突**
   - 风险：不同市场的同名插件会被视为同一插件
   - 示例：`my-plugin@marketplace-a` 和 `my-plugin@marketplace-b` 都映射为 `my-plugin`
   - 影响：锁定状态可能错误地应用到多个插件

3. **布尔值 false 也被锁定**
   - 行为：`enabledPlugins` 中值为 `false` 的条目也被视为管理插件
   - 原因：组织明确声明了该插件，即使是禁用状态也由组织控制
   - 潜在困惑：用户可能不理解为什么禁用的插件也被锁定

4. **空名称处理**
   - 风险：`pluginId.split('@')[0]` 可能返回空字符串（如 `@marketplace`）
   - 缓解：`if (name)` 检查过滤空字符串

### 边界情况

1. **无策略配置**
   - 行为：返回 `null`（最常见情况，表示无组织策略）

2. **空 enabledPlugins**
   - 行为：返回 `null`

3. **只有遗留格式**
   - 行为：返回 `null`（所有条目被跳过）

4. **混合格式**
   - 行为：只提取有效格式，忽略其他

5. **重复插件名称**
   - 行为：`Set` 自动去重

6. **特殊字符插件名**
   - 行为：`split('@')[0]` 提取，不验证名称有效性

### 改进建议

1. **支持完整插件 ID**
   - 当前：只返回插件名称，丢失市场信息
   - 建议：返回 `Set<{ name: string; marketplace: string }>` 或完整 `pluginId`
   - 好处：避免同名不同市场插件的冲突

2. **遗留格式支持**
   - 当前：完全跳过 `owner/repo` 数组格式
   - 建议：
     - 添加转换逻辑，将遗留格式映射为新格式
     - 或记录警告，提示管理员更新配置

3. **更详细的锁定信息**
   - 当前：只返回名称集合
   - 建议：返回 `Map<string, { locked: boolean; enabled: boolean; source: string }>`
   - 好处：UI 可以显示更详细的锁定原因

4. **缓存结果**
   - 当前：每次调用都重新计算
   - 建议：添加简单的记忆化（settings 不经常变化）
   ```typescript
   let cachedNames: Set<string> | null | undefined
   let cachedSettingsHash: string
   
   export function getManagedPluginNames(): Set<string> | null {
     const settings = getSettingsForSource('policySettings')
     const hash = JSON.stringify(settings?.enabledPlugins)
     if (hash === cachedSettingsHash) return cachedNames
     // ... 重新计算
   }
   ```

5. **验证和日志**
   - 当前：静默跳过无效条目
   - 建议：
     - 添加 debug 日志，记录被跳过的条目
     - 验证插件名称格式（如不允许空格）

6. **区分锁定类型**
   - 当前：`true` 和 `false` 都被锁定
   - 建议：
     - `true`：强制启用且锁定
     - `false`：强制禁用且锁定
     - 返回更详细的类型，让 UI 显示不同图标

7. **批量查询接口**
   - 当前：返回所有管理插件，调用方自行检查
   - 建议：添加 `isPluginManaged(pluginId: string): boolean` 便捷函数
   ```typescript
   export function isPluginManaged(pluginId: string): boolean {
     const names = getManagedPluginNames()
     if (!names) return false
     const name = pluginId.split('@')[0]
     return names.has(name)
   }
   ```

### 测试要点

1. **正常情况**：验证正确提取 `plugin@marketplace` 格式
2. **布尔值 false**：验证 `false` 值也被包含
3. **遗留格式**：验证 `owner/repo` 数组被排除
4. **空值处理**：验证 `null`、`undefined`、空对象返回 `null`
5. **边界格式**：验证 `@marketplace`（空名称）被排除
6. **重复名称**：验证 Set 去重行为
7. **混合输入**：验证复杂配置的正确处理

### 代码简洁性评价

**优点**：
- 代码简洁，职责单一
- 早期返回优化（`if (!enabledPlugins) return null`）
- 使用 `Set` 自动去重

**潜在改进**：
- 注释可以更详细说明设计决策
- 考虑提取常量（如 `@` 分隔符）
- 添加 JSDoc 类型注释提高 IDE 支持
