# 研究文档: src/commands/keybindings/index.ts

## 场景与职责

本文件是 `/keybindings` 命令的入口模块（Entry Point），负责定义和导出 keybindings 命令的元数据。该命令允许用户打开或创建键盘快捷键配置文件（`~/.claude/keybindings.json`）。

**核心职责：**
1. 定义命令的元数据（名称、描述、类型、启用条件等）
2. 实现懒加载（lazy loading）机制，延迟加载实际的命令实现
3. 通过 `isEnabled` 函数控制命令的可用性（基于 GrowthBook feature flag）

**产品定位：**
- 这是一个**本地命令**（`type: 'local'`），直接在终端执行，不涉及 AI 模型交互
- 命令属于**远程安全命令**（`REMOTE_SAFE_COMMANDS`），可在远程模式下使用
- 当前功能处于 **Preview 阶段**，通过 feature flag 控制是否对外启用

---

## 功能点目的

### 1. 命令定义与导出

```typescript
const keybindings = {
  name: 'keybindings',
  description: 'Open or create your keybindings configuration file',
  isEnabled: () => isKeybindingCustomizationEnabled(),
  supportsNonInteractive: false,
  type: 'local',
  load: () => import('./keybindings.js'),
} satisfies Command
```

**关键属性说明：**

| 属性 | 值 | 说明 |
|------|-----|------|
| `name` | `'keybindings'` | 命令名称，用户输入 `/keybindings` 触发 |
| `description` | 见代码 | 在帮助文档和自动补全中显示 |
| `isEnabled` | 函数 | 动态检查功能是否启用（基于 GrowthBook） |
| `supportsNonInteractive` | `false` | 不支持非交互模式（需要 TUI） |
| `type` | `'local'` | 本地执行命令，不发送到模型 |
| `load` | 函数 | 懒加载实现模块 `./keybindings.js` |

### 2. 功能开关控制

命令通过 `isKeybindingCustomizationEnabled()` 函数控制可用性：

```typescript
// 来自 src/keybindings/loadUserBindings.ts
export function isKeybindingCustomizationEnabled(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_keybinding_customization_release',
    false,
  )
}
```

- Feature flag: `tengu_keybinding_customization_release`
- 默认值为 `false`，即默认不启用
- 使用缓存值（可能不是最新），用于快速判断

---

## 具体技术实现

### 懒加载机制

```typescript
load: () => import('./keybindings.js')
```

**实现细节：**
1. 使用动态 `import()` 实现懒加载
2. 返回 Promise，解析为包含 `call` 函数的模块
3. 只有在用户实际执行 `/keybindings` 命令时才加载 `./keybindings.js`
4. 减少启动时的内存占用和初始化时间

### 类型安全

使用 `satisfies Command` 确保对象符合 `Command` 类型定义：

```typescript
import type { Command } from '../../commands.js'
```

`Command` 类型定义在 `src/types/command.ts`，包含：
- `CommandBase`: 基础属性（name, description, isEnabled, etc.）
- `LocalCommand`: 本地命令特有属性（type, supportsNonInteractive, load）

---

## 关键代码路径与文件引用

### 依赖关系图

```
src/commands/keybindings/index.ts
    │
    ├── imports ──────────────────────────────┐
    │   ├── type { Command } from '../../commands.js'
    │   └── isKeybindingCustomizationEnabled 
    │       from '../../keybindings/loadUserBindings.js'
    │
    ├── lazy loads ───────────────────────────┤
    │   └── './keybindings.js' (实际实现)
    │
    └── used by ──────────────────────────────┤
        ├── src/commands.ts (命令注册)
        └── REMOTE_SAFE_COMMANDS (远程模式白名单)
```

### 关键文件引用

| 文件路径 | 用途 |
|----------|------|
| `src/commands.ts` | 导入并注册到命令列表（第27行导入，第284行加入 COMMANDS） |
| `src/keybindings/loadUserBindings.ts` | 提供功能开关检查函数 |
| `src/commands/keybindings/keybindings.ts` | 实际命令实现（懒加载） |
| `src/types/command.ts` | Command 类型定义 |

### 在命令系统中的位置

```typescript
// src/commands.ts
import keybindings from './commands/keybindings/index.js'

// 加入内置命令列表
const COMMANDS = memoize((): Command[] => [
  // ... 其他命令
  keybindings,  // 第284行
  // ... 其他命令
])

// 远程安全命令集合
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  // ...
  keybindings,  // 第633行
  // ...
])
```

---

## 依赖与外部交互

### 外部依赖

| 模块 | 导出内容 | 用途 |
|------|----------|------|
| `../../commands.js` | `Command` 类型 | 类型定义 |
| `../../keybindings/loadUserBindings.js` | `isKeybindingCustomizationEnabled` | 功能开关检查 |

### 被调用方

| 调用方 | 调用方式 | 用途 |
|--------|----------|------|
| `src/commands.ts` | `import keybindings from './commands/keybindings/index.js'` | 注册命令 |
| `getCommands()` | 遍历 COMMANDS 数组 | 获取可用命令列表 |
| `isBridgeSafeCommand()` | 检查 REMOTE_SAFE_COMMANDS | 判断远程模式是否可用 |

---

## 风险、边界与改进建议

### 潜在风险

1. **Feature Flag 缓存不一致**
   - 使用 `getFeatureValue_CACHED_MAY_BE_STALE` 可能导致命令可见性与实际功能状态不一致
   - 用户可能在命令可见但功能未完全初始化时执行命令

2. **懒加载失败**
   - 如果 `./keybindings.js` 文件损坏或缺失，动态导入会失败
   - 需要在调用方处理加载错误

3. **非交互模式限制**
   - `supportsNonInteractive: false` 意味着在 CI/CD 等场景下无法使用
   - 可能需要考虑提供非交互模式的支持

### 边界条件

1. **功能未启用时的行为**
   - 当 `isKeybindingCustomizationEnabled()` 返回 `false` 时，命令在帮助和自动补全中不可见
   - 这是预期行为，用于控制 Preview 功能的曝光

2. **远程模式支持**
   - 命令在 `REMOTE_SAFE_COMMANDS` 中，支持远程控制模式
   - 但打开编辑器的行为在远程客户端可能有不同的表现

### 改进建议

1. **错误处理增强**
   ```typescript
   // 建议添加加载错误处理
   load: async () => {
     try {
       return await import('./keybindings.js')
     } catch (error) {
       return {
         call: async () => ({
           type: 'text' as const,
           value: 'Failed to load keybindings module. Please try again.'
         })
       }
     }
   }
   ```

2. **非交互模式支持**
   - 考虑支持 `--create` 或 `--open` 参数，允许非交互式创建/打开配置文件
   - 将 `supportsNonInteractive` 设为 `true` 并提供相应实现

3. **功能开关实时更新**
   - 考虑监听 feature flag 变化，动态启用/禁用命令
   - 避免需要重启应用才能看到命令

4. **文档完善**
   - 在命令描述中添加更多上下文，说明配置文件的位置和格式
   - 提供指向官方文档的链接

---

## 附录：相关代码片段

### Command 类型定义（简化）

```typescript
// src/types/command.ts
export type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>
}

export type CommandBase = {
  name: string
  description: string
  isEnabled?: () => boolean
  // ... 其他属性
}

export type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

### 功能开关实现

```typescript
// src/keybindings/loadUserBindings.ts (第41-46行)
export function isKeybindingCustomizationEnabled(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_keybinding_customization_release',
    false,
  )
}
```
