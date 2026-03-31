# 研究文档: src/commands/skills/skills.tsx

## 场景与职责

`src/commands/skills/skills.tsx` 是 `/skills` 命令的实际实现文件，负责渲染交互式技能列表界面。当用户执行 `/skills` 命令时，该模块导出的 `call` 函数被调用，返回一个 React 元素，由 Ink 渲染为终端 UI。

核心职责：
- 提供命令入口函数 `call`，符合 `LocalJSXCommandCall` 类型签名
- 渲染 `SkillsMenu` 组件展示所有可用技能
- 处理命令完成回调，支持正常退出和结果传递

## 功能点目的

1. **命令执行入口**: 实现 `LocalJSXCommandCall` 接口，作为 `/skills` 命令的执行体
2. **组件渲染**: 渲染 `SkillsMenu` 组件，展示分类的技能列表
3. **上下文传递**: 将命令上下文中的可用命令列表传递给 SkillsMenu
4. **生命周期管理**: 通过 `onDone` 回调通知命令完成

## 具体技术实现

### 关键函数签名

```typescript
export async function call(
  onDone: LocalJSXCommandOnDone,
  context: LocalJSXCommandContext,
): Promise<React.ReactNode>
```

### 参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| `onDone` | `LocalJSXCommandOnDone` | 命令完成回调函数，用于通知系统命令执行结束 |
| `context` | `LocalJSXCommandContext` | 命令执行上下文，包含各种运行时信息和选项 |

### 返回值

返回 `Promise<React.ReactNode>`，即渲染的 React 元素：
```jsx
<SkillsMenu onExit={onDone} commands={context.options.commands} />
```

### 关键类型定义（来自 src/types/command.ts）

```typescript
// 命令完成回调类型
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: CommandResultDisplay  // 'skip' | 'system' | 'user'
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  },
) => void

// 命令上下文类型
export type LocalJSXCommandContext = ToolUseContext & {
  canUseTool?: CanUseToolFn
  setMessages: (updater: (prev: Message[]) => Message[]) => void
  options: {
    dynamicMcpConfig?: Record<string, ScopedMcpServerConfig>
    ideInstallationStatus: IDEExtensionInstallationStatus | null
    theme: ThemeName
    commands: Command[]  // 关键：传递给 SkillsMenu 的命令列表
  }
  onChangeAPIKey: () => void
  onChangeDynamicMcpConfig?: (...)
  onInstallIDEExtension?: (ide: IdeType) => void
  resume?: (...)
}
```

### 渲染的组件

**SkillsMenu** (`src/components/skills/SkillsMenu.tsx`):
- 接收 `onExit` 和 `commands` 两个 props
- `onExit`: 用户关闭技能菜单时调用，触发 `onDone` 回调
- `commands`: 当前可用的所有命令列表，用于渲染技能分类展示

## 关键代码路径与文件引用

### 当前文件
- `src/commands/skills/skills.tsx` - 命令实现

### 直接依赖
| 文件 | 用途 |
|------|------|
|`react`|React 核心库，用于 JSX 渲染|
|`../../commands.js`|`LocalJSXCommandContext` 类型定义|
|`../../components/skills/SkillsMenu.js`|技能菜单 UI 组件|
|`../../types/command.js`|`LocalJSXCommandOnDone` 类型定义|

### 调用链
```
用户输入 /skills
  → commands.ts 路由
  → index.ts 的 load() 导入本文件
  → call() 函数执行
  → SkillsMenu 组件渲染
  → 用户浏览/关闭菜单
  → onExit/onDone 回调执行
  → 命令结束，返回 REPL
```

### SkillsMenu 组件关键逻辑

SkillsMenu 组件（`src/components/skills/SkillsMenu.tsx`）的主要功能：

1. **技能过滤**: 从 `commands` 中筛选出类型为 `prompt` 且来源为技能相关的命令
   ```typescript
   const skills = commands.filter(cmd => 
     cmd.type === "prompt" && 
     (cmd.loadedFrom === "skills" || 
      cmd.loadedFrom === "commands_DEPRECATED" || 
      cmd.loadedFrom === "plugin" || 
      cmd.loadedFrom === "mcp")
   )
   ```

2. **分组展示**: 按来源分组展示技能
   - `projectSettings` - 项目级技能 (.claude/skills/)
   - `userSettings` - 用户级技能 (~/.claude/skills/)
   - `policySettings` - 策略级技能
   - `plugin` - 插件技能
   - `mcp` - MCP 技能

3. **技能信息展示**: 每个技能显示名称、来源插件名、预估 token 数

## 依赖与外部交互

### 运行时依赖
| 模块 | 说明 |
|------|------|
|React|UI 渲染框架|
|Ink|React 终端渲染器（通过 components 导入）|

### 数据流
```
commands.ts (命令中心)
  ↓ 提供 commands 数组
skills.tsx (本文件)
  ↓ 传递 commands prop
SkillsMenu.tsx (UI 组件)
  ↓ 过滤、分组、渲染
终端 UI 展示
```

### 命令来源
`context.options.commands` 来自 `commands.ts` 的 `getCommands(cwd)` 函数，包含：
- 内置命令（built-in commands）
- 技能目录命令（skill dir commands）
- 插件技能（plugin skills）
- 捆绑技能（bundled skills）
- 内置插件技能（builtin plugin skills）
- 动态技能（dynamic skills）

## 风险、边界与改进建议

### 风险点
1. **空命令列表**: 若 `context.options.commands` 为空或 undefined，SkillsMenu 会显示空状态
2. **类型安全**: 依赖 TypeScript 类型确保 context 结构正确，运行时无校验
3. **异步加载**: 技能列表在命令执行时确定，新添加的技能需重启或刷新才能显示

### 边界情况
1. **无技能时**: SkillsMenu 显示 "No skills found" 和创建提示
2. **大量技能**: 组件使用 React Compiler 优化渲染性能（文件头有 `react/compiler-runtime` 导入）
3. **取消操作**: 用户按 Esc 取消时，onDone 被调用并传递 "Skills dialog dismissed" 消息

### 改进建议
1. **添加加载状态**: 若 commands 加载较慢，可添加加载指示
2. **错误边界**: 添加 Error Boundary 捕获 SkillsMenu 渲染错误
3. **实时更新**: 考虑监听技能变化事件，支持动态刷新
4. **搜索过滤**: SkillsMenu 已支持，但可考虑添加更高级的筛选功能
5. **缓存优化**: 对于大型技能列表，可考虑虚拟滚动优化性能

### 相关配置
- 技能目录位置: `.claude/skills/` (项目级), `~/.claude/skills/` (用户级)
- 技能文件格式: `skill-name/SKILL.md`
- 条件技能: 支持 `paths` frontmatter 控制技能在特定文件操作时才显示

---

*文档生成时间: 2026-04-01*
*研究范围: 代码实现、组件架构、命令系统、技能加载机制*
