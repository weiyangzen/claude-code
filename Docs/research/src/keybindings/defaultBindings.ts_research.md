# defaultBindings.ts 研究文档

## 场景与职责

`defaultBindings.ts` 是 Claude Code 键盘快捷键系统的配置中心，定义了所有默认的键盘绑定。它是用户自定义绑定的基础，用户配置会在此基础上进行覆盖和扩展。

**核心职责：**
1. **定义默认绑定**：为所有支持的 UI 上下文定义标准键盘快捷键
2. **平台适配**：根据操作系统（Windows/macOS/Linux）调整特定快捷键
3. **功能开关集成**：根据 GrowthBook 功能标志（feature flags）条件性启用绑定
4. **终端兼容性处理**：处理不同终端模拟器的兼容性问题

**架构位置：**
```
defaultBindings.ts (本文件)
    ↓ 被 loadUserBindings.ts 导入
loadUserBindings.ts
    ↓ 合并用户绑定
KeybindingSetup
    ↓ 通过 Context 分发
各组件使用
```

---

## 功能点目的

### 1. 平台特定键定义

**图片粘贴快捷键：**
```typescript
const IMAGE_PASTE_KEY = getPlatform() === 'windows' ? 'alt+v' : 'ctrl+v'
```
- Windows: `alt+v`（`ctrl+v` 被系统粘贴占用）
- 其他平台: `ctrl+v`

**VT 模式支持检测：**
```typescript
const SUPPORTS_TERMINAL_VT_MODE =
  getPlatform() !== 'windows' ||
  (isRunningWithBun()
    ? satisfies(process.versions.bun, '>=1.2.23')
    : satisfies(process.versions.node, '>=22.17.0 <23.0.0 || >=24.2.0'))
```
- Windows Terminal 需要 VT 模式才能正确处理修饰键组合
- Node.js 在 24.2.0 / 22.17.0 启用 VT 模式
- Bun 在 1.2.23 启用 VT 模式

**模式循环快捷键：**
```typescript
const MODE_CYCLE_KEY = SUPPORTS_TERMINAL_VT_MODE ? 'shift+tab' : 'meta+m'
```
- 支持 VT 模式: `shift+tab`
- 不支持 VT 模式: `meta+m`（Windows 旧终端回退）

### 2. 默认绑定结构

**绑定块格式：**
```typescript
{
  context: 'ContextName',  // UI 上下文名称
  bindings: {
    'key-combination': 'action:name',  // 快捷键 → 动作
    ...
  }
}
```

**支持的上下文（18个）：**
1. **Global** - 全局生效
2. **Chat** - 聊天输入框聚焦时
3. **Autocomplete** - 自动补全菜单可见时
4. **Settings** - 设置菜单打开时
5. **Confirmation** - 确认/权限对话框显示时
6. **Tabs** - 标签导航激活时
7. **Transcript** - 查看转录时
8. **HistorySearch** - 搜索命令历史时（ctrl+r）
9. **Task** - 任务/代理前台运行时
10. **ThemePicker** - 主题选择器打开时
11. **Scroll** - 滚动视图
12. **Help** - 帮助覆盖层打开时
13. **Attachments** - 选择对话框中导航图片附件时
14. **Footer** - 页脚指示器聚焦时
15. **MessageSelector** - 消息选择器（回退）打开时
16. **DiffDialog** - diff 对话框打开时
17. **ModelPicker** - 模型选择器打开时
18. **Select** - 选择/列表组件聚焦时
19. **Plugin** - 插件对话框打开时

### 3. 功能标志控制

**KAIROS / KAIROS_BRIEF：**
```typescript
...(feature('KAIROS') || feature('KAIROS_BRIEF')
  ? { 'ctrl+shift+b': 'app:toggleBrief' as const }
  : {})
```

**QUICK_SEARCH：**
```typescript
...(feature('QUICK_SEARCH')
  ? {
      'ctrl+shift+f': 'app:globalSearch' as const,
      'cmd+shift+f': 'app:globalSearch' as const,
      'ctrl+shift+p': 'app:quickOpen' as const,
      'cmd+shift+p': 'app:quickOpen' as const,
    }
  : {})
```

**TERMINAL_PANEL：**
```typescript
...(feature('TERMINAL_PANEL') ? { 'meta+j': 'app:toggleTerminal' } : {})
```

**MESSAGE_ACTIONS：**
```typescript
...(feature('MESSAGE_ACTIONS')
  ? { 'shift+up': 'chat:messageActions' as const }
  : {})
```

**VOICE_MODE：**
```typescript
...(feature('VOICE_MODE') ? { space: 'voice:pushToTalk' } : {})
```

### 4. 特殊绑定说明

**不可重新绑定的快捷键：**
```typescript
'ctrl+c': 'app:interrupt'  // 中断/退出（硬编码）
'ctrl+d': 'app:exit'        // 退出（硬编码）
```
- 定义在这里以便 resolver 能找到它们
- 用户在 `reservedShortcuts.ts` 中尝试覆盖会收到错误

**双绑定支持：**
```typescript
'ctrl+_': 'chat:undo',       // 传统终端（发送 \x1f 控制字符）
'ctrl+shift+-': 'chat:undo', // Kitty 协议（发送带修饰符的物理键）
```

**命令绑定：**
```typescript
'ctrl+x ctrl+e': 'chat:externalEditor'  // readline 原生编辑并执行命令绑定
'ctrl+g': 'chat:externalEditor'         // 替代绑定
```

---

## 具体技术实现

### 1. 平台检测逻辑

```typescript
import { getPlatform } from '../utils/platform.js'
import { isRunningWithBun } from '../utils/bundledMode.js'
import { satisfies } from 'src/utils/semver.js'
```

**平台值：**
- `'windows'` - Windows（包括 WSL 检测）
- `'macos'` - macOS
- `'linux'` - Linux
- `'wsl'` - Windows Subsystem for Linux
- `'unknown'` - 未知平台

### 2. 功能标志检测

```typescript
import { feature } from 'bun:bundle'
```

**使用方式：**
- `feature('FLAG_NAME')` 返回布尔值
- 在编译时评估，未启用的功能代码会被 tree-shake

### 3. 绑定合并机制

**在 loadUserBindings.ts 中：**
```typescript
const defaultBindings = getDefaultParsedBindings()
const userParsed = parseBindings(userBlocks)
// 用户绑定在后，覆盖默认绑定
const mergedBindings = [...defaultBindings, ...userParsed]
```

**覆盖规则：**
- 相同上下文 + 相同键组合，用户绑定获胜
- `null` 值可以解绑默认快捷键

### 4. 类型安全

**使用 `as const` 断言：**
```typescript
'ctrl+shift+b': 'app:toggleBrief' as const
```

**原因：**
- 确保动作名称被推断为字面量类型
- 支持类型检查和自动补全

---

## 关键代码路径与文件引用

### 核心常量

| 常量 | 定义 | 用途 |
|------|------|------|
| `IMAGE_PASTE_KEY` | 平台相关 | 图片粘贴快捷键 |
| `SUPPORTS_TERMINAL_VT_MODE` | 版本检测 | VT 模式支持判断 |
| `MODE_CYCLE_KEY` | 条件选择 | 模式循环快捷键 |

### 依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `bun:bundle` | `feature` | 功能标志检测 |
| `src/utils/semver.js` | `satisfies` | 版本比较 |
| `../utils/bundledMode.js` | `isRunningWithBun` | 运行时检测 |
| `../utils/platform.js` | `getPlatform` | 平台检测 |
| `./types.js` | `KeybindingBlock` | 类型定义 |

### 导出符号

```typescript
export { DEFAULT_BINDINGS }  // 默认绑定数组
```

### 绑定统计

- **Global 上下文**：9-13 个绑定（取决于功能标志）
- **Chat 上下文**：16-20 个绑定
- **其他上下文**：平均 5-10 个绑定
- **总计**：约 100+ 个默认绑定

---

## 依赖与外部交互

### 1. 上游依赖（输入）

**平台信息：**
- `getPlatform()` - 检测当前操作系统
- `process.versions.bun` / `process.versions.node` - 运行时版本

**功能标志：**
- `feature('KAIROS')` - KAIROS 功能
- `feature('KAIROS_BRIEF')` - KAIROS 简报功能
- `feature('QUICK_SEARCH')` - 快速搜索
- `feature('TERMINAL_PANEL')` - 终端面板
- `feature('MESSAGE_ACTIONS')` - 消息操作
- `feature('VOICE_MODE')` - 语音模式

### 2. 下游消费（输出）

**被 loadUserBindings.ts 消费：**
```typescript
import { DEFAULT_BINDINGS } from './defaultBindings.js'
const defaultBindings = parseBindings(DEFAULT_BINDINGS)
```

**被 template.ts 消费：**
```typescript
import { DEFAULT_BINDINGS } from './defaultBindings.js'
// 生成用户配置模板
```

**被 validate.ts 消费（间接）：**
- 验证用户绑定时参考默认绑定

### 3. 与 resolver 的交互

- 绑定通过 `parseBindings()` 转换为 `ParsedBinding[]`
- resolver 使用这些解析后的绑定进行按键匹配

---

## 风险、边界与改进建议

### 1. 已知风险

**平台检测复杂性：**
- VT 模式检测涉及多个运行时版本比较
- 如果未来有更多运行时（如 Deno），需要更新检测逻辑

**功能标志编译时评估：**
- `feature()` 在编译时评估，代码分支会被静态分析
- 如果功能标志系统变化，可能影响 tree-shaking

**硬编码快捷键：**
- `ctrl+c` 和 `ctrl+d` 虽然定义在这里，但实际处理是硬编码的
- 可能导致用户困惑（为什么定义了但不能重新绑定）

### 2. 边界情况

**WSL 处理：**
- WSL 被检测为独立平台 `'wsl'`
- 但 `getPlatform() === 'windows'` 为 false，所以使用 Linux 绑定

**未知平台：**
- `getPlatform()` 返回 `'unknown'` 时使用 Linux 绑定

**功能标志组合：**
- 多个功能标志可能同时启用，绑定数量增加
- 需要确保没有冲突的快捷键

### 3. 改进建议

**代码组织：**
```typescript
// 建议：将大型绑定块提取为独立对象
const CHAT_BINDINGS_BASE = { ... }
const CHAT_BINDINGS_WITH_VOICE = { ... }
// 然后根据功能标志合并
```

**文档生成：**
- 当前绑定文档是手动的
- 建议从代码自动生成帮助文档

**冲突检测：**
- 添加构建时检查，确保默认绑定没有冲突
- 特别是功能标志组合时的冲突

**平台测试：**
- 添加自动化测试，验证各平台的绑定正确性
- 特别是 Windows VT 模式回退

### 4. 维护建议

**添加新绑定时：**
1. 确定所属上下文
2. 检查是否与现有绑定冲突
3. 考虑平台差异（特别是 Windows）
4. 如果涉及功能标志，使用条件展开
5. 在 `schema.ts` 中添加对应动作到 `KEYBINDING_ACTIONS`

**修改现有绑定时：**
1. 检查 `reservedShortcuts.ts` 中的保留快捷键
2. 更新相关文档
3. 考虑向后兼容性

### 5. 动作命名规范

**命名空间前缀：**
- `app:` - 应用级动作
- `chat:` - 聊天输入动作
- `history:` - 历史导航
- `autocomplete:` - 自动补全
- `confirm:` - 确认对话框
- `select:` - 选择组件
- `settings:` - 设置面板
- `voice:` - 语音功能
- `plugin:` - 插件对话框

**一致性检查：**
- 建议添加 lint 规则，确保新动作符合命名规范
