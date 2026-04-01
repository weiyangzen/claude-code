# Voice Command Index Module Research Document

## 场景与职责

`src/commands/voice/index.ts` 是 `/voice` 命令的入口模块，采用 Claude Code 标准的命令注册模式。该模块负责：

1. **命令元数据定义**：声明命令名称、描述、类型、可用性等基础属性
2. **特性开关控制**：通过 `isEnabled` 和 `isHidden` 属性控制命令的可见性和可用性
3. **懒加载实现**：延迟加载实际命令实现 (`voice.ts`)，避免启动时加载不必要的依赖
4. **权限范围声明**：限制命令仅对 Claude.ai 订阅者可用

该模块是语音功能的第一道闸门，确保只有符合条件的用户才能看到和使用 `/voice` 命令。

## 功能点目的

### 1. 命令元数据结构

```typescript
const voice = {
  type: 'local',           // 本地执行命令，不发送到模型
  name: 'voice',           // 命令标识符
  description: 'Toggle voice mode',  // 用户可见描述
  availability: ['claude-ai'],       // 仅 Claude.ai 用户可用
  isEnabled: () => isVoiceGrowthBookEnabled(),  // GrowthBook 特性开关
  get isHidden() {         // 动态隐藏控制
    return !isVoiceModeEnabled()
  },
  supportsNonInteractive: false,     // 不支持非交互模式
  load: () => import('./voice.js'),  // 懒加载实现
} satisfies Command
```

### 2. 双重开关机制

| 属性 | 作用 | 检查内容 |
|------|------|----------|
| `isEnabled` | 决定是否注册命令 | GrowthBook 特性标志 `tengu_amber_quartz_disabled` |
| `isHidden` | 决定是否显示在帮助/补全中 | 完整的语音模式启用状态（Auth + GrowthBook） |

这种设计允许：
- 特性被禁用时（kill-switch），命令完全不注册
- 用户未登录时，命令注册但隐藏（避免未登录用户看到不可用的功能）

## 具体技术实现

### 依赖导入

```typescript
import type { Command } from '../../commands.js'
import {
  isVoiceGrowthBookEnabled,
  isVoiceModeEnabled,
} from '../../voice/voiceModeEnabled.js'
```

- `Command` 类型：来自 `src/commands.ts`，定义命令的标准接口
- `isVoiceGrowthBookEnabled`：仅检查 GrowthBook kill-switch 标志
- `isVoiceModeEnabled`：完整检查（Auth + GrowthBook）

### 懒加载模式

```typescript
load: () => import('./voice.js')
```

使用动态导入实现懒加载：
- 启动时不加载 `voice.ts` 及其依赖（包括原生音频模块）
- 用户首次执行 `/voice` 时才加载实际实现
- 避免未使用语音功能的用户支付启动成本

## 关键代码路径与文件引用

### 调用关系

```
用户输入 /voice
    ↓
src/commands.ts: getCommands() 
    ↓
meetsAvailabilityRequirement(cmd) - 检查 availability: ['claude-ai']
    ↓
isCommandEnabled(cmd) - 调用 isEnabled()
    ↓
命令匹配成功，执行 cmd.load()
    ↓
import('./voice.js') - 懒加载
    ↓
执行 voice.ts 中的 call 函数
```

### 相关文件

| 文件路径 | 作用 |
|----------|------|
| `src/commands.ts` | 命令注册中心，加载所有命令 |
| `src/voice/voiceModeEnabled.ts` | 语音模式启用状态检查 |
| `src/commands/voice/voice.ts` | 实际命令实现 |
| `src/types/command.ts` | Command 类型定义 |

## 依赖与外部交互

### 内部依赖

1. **`src/commands.js`**
   - 提供 `Command` 类型定义
   - 通过 `feature('VOICE_MODE')` 条件导入本模块

2. **`src/voice/voiceModeEnabled.ts`**
   - `isVoiceGrowthBookEnabled()`: 检查 GrowthBook kill-switch
   - `isVoiceModeEnabled()`: 综合检查 OAuth + GrowthBook

### 外部服务

- **GrowthBook**: 通过 `isVoiceGrowthBookEnabled()` 检查特性标志
  - 标志名: `tengu_amber_quartz_disabled`
  - 作用: 紧急禁用语音功能（kill-switch）

## 风险、边界与改进建议

### 风险点

1. **条件导入依赖**
   ```typescript
   // src/commands.ts:80-82
   const voiceCommand = feature('VOICE_MODE')
     ? require('./commands/voice/index.js').default
     : null
   ```
   - `feature('VOICE_MODE')` 必须在编译时确定
   - 如果构建配置错误，模块可能被意外排除

2. **availability 检查时机**
   - `meetsAvailabilityRequirement()` 在每次 `getCommands()` 时执行
   - 用户登录状态变更后（如执行 `/login`），命令会自动显示

3. **isHidden 的 getter 模式**
   - 每次访问都重新计算，可能产生多次函数调用
   - 虽然开销小，但在大量命令列表渲染时需注意

### 边界情况

| 场景 | 行为 |
|------|------|
| 用户未登录 | 命令隐藏（isHidden=true），但已注册 |
| GrowthBook 禁用 | 命令不注册（isEnabled=false） |
| 非 Claude.ai 用户 | 命令不可用（availability 不匹配） |
| 构建时 VOICE_MODE 关闭 | 整个模块被排除，节省包体积 |

### 改进建议

1. **添加调试日志**
   ```typescript
   // 可考虑在 isEnabled/isHidden 中添加调试输出
   isEnabled: () => {
     const enabled = isVoiceGrowthBookEnabled()
     logForDebugging(`[voice] command enabled: ${enabled}`)
     return enabled
   }
   ```

2. **统一检查逻辑**
   - 当前 `isEnabled` 和 `isHidden` 使用不同检查函数
   - 考虑统一为 `isVoiceModeEnabled()`，简化理解成本

3. **文档化特性标志**
   - 在代码注释中明确列出所有相关的 GrowthBook 标志
   - 便于运维人员快速定位配置问题

4. **类型安全增强**
   ```typescript
   // 可考虑使用更精确的类型
   availability: ['claude-ai'] as const satisfies CommandAvailability[]
   ```
