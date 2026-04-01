# SkillsMenu.tsx 深度研究文档

> 目标文件：`src/components/skills/SkillsMenu.tsx`  
> 研究范围：代码、调用链、依赖模块、类型定义、命令注册体系、构建特征  
> 执行器：kimi (k2p5)  
> 生成时间：2026-04-01

---

## 一、场景与职责

`SkillsMenu` 是 Claude Code TUI 中用于展示「可用技能（skills）」列表的 **Ink/React 终端 UI 组件**。当用户在 REPL 中输入 `/skills`（或触发等价的 slash command）时，系统会渲染该组件，弹出一个只读的对话框，列出当前会话中所有已加载、可被模型或用户调用的 skill/command。

### 1.1 核心职责
- **过滤**：从全量 `commands` 中筛选出类型为 `prompt` 且 `loadedFrom` 属于技能来源的命令。
- **分组**：按技能来源（`projectSettings` / `userSettings` / `policySettings` / `plugin` / `mcp`）分组渲染。
- **排序**：组内按命令显示名做字母升序排列。
- **元信息展示**：为每个技能展示预估的 frontmatter token 数；为不同来源展示路径或 MCP server 名等副标题。
- **空态处理**：当没有任何技能时，给出创建指引（`.claude/skills/` 或 `~/.claude/skills/`）。
- **交互退出**：仅支持 `Esc` / `confirm:no` 快捷键关闭对话框（无搜索、无选择执行能力）。

### 1.2 在命令体系中的位置
```
用户输入 /skills
    └── src/commands/skills/index.ts  (注册 local-jsx 命令)
        └── src/commands/skills/skills.tsx  (call 函数)
            └── <SkillsMenu onExit={onDone} commands={context.options.commands} />
                └── src/components/skills/SkillsMenu.tsx  (本文件)
```

---

## 二、功能点目的

| 功能点 | 目的 |
|--------|------|
| **过滤 `commands` 为 `skills`** | 避免把内置 local/local-jsx 命令（如 `/clear`、`/exit`）混入技能列表，保持列表语义纯净。 |
| **按来源分组** | 帮助用户区分「项目级」「用户级」「企业策略级」「插件」「MCP」技能，便于排查来源与优先级。 |
| **展示 `~X tokens`** | 让用户直观感知每个 skill 的 frontmatter 描述长度，辅助判断上下文占用。 |
| **展示来源路径/Server 名** | 对于文件型技能，显示磁盘路径；对于 MCP 技能，显示所属 server，便于定位配置。 |
| **空态提示** | 降低新用户门槛，直接告诉用户去哪里创建第一个 skill。 |
| **仅取消无选择** | 该菜单是「信息展示型」弹窗，不是「选择执行型」列表；用户看完按 Esc 退出即可。 |

---

## 三、具体技术实现

### 3.1 文件形态说明（重要）
`src/components/skills/SkillsMenu.tsx` 的磁盘内容 **并非手写 TypeScript 源码**，而是 **React Compiler 编译后的输出**（带内联 source map）。证据：
- 文件首行：`import { c as _c } from "react/compiler-runtime";`
- 函数体使用 `_c(N)` 作为 memoization cache，所有 JSX 节点被拆分为 `t0`–`tN` 的细粒度条件创建逻辑。
- 文件末尾包含 `//# sourceMappingURL=data:application/json;base64,...`，其 `sources` 字段指向原始的 `SkillsMenu.tsx`。

这意味着：
- 可读性较差，变量名被机器生成（`t0`, `t1`, `$[0]` 等）。
- 任何直接修改本文件的行为，在下次构建时都会被覆盖；真正的可维护入口是编译前的源码（仓库中未保留原始手写 TSX，或原始 TSX 已被编译产物替换）。

### 3.2 类型定义

```ts
// 文件内局部类型（从编译产物还原）
type SkillCommand = CommandBase & PromptCommand;
type SkillSource = SettingSource | 'plugin' | 'mcp';

type Props = {
  onExit: (result?: string, options?: { display?: CommandResultDisplay }) => void;
  commands: Command[];
};
```

### 3.3 过滤逻辑

```ts
// 对应编译产物中 _temp 函数（原 filter 回调）
function _temp(cmd) {
  return cmd.type === "prompt" && (
    cmd.loadedFrom === "skills" ||
    cmd.loadedFrom === "commands_DEPRECATED" ||
    cmd.loadedFrom === "plugin" ||
    cmd.loadedFrom === "mcp"
  );
}
```

- 只保留 `type === 'prompt'` 的命令。
- `loadedFrom` 限定为四种技能来源：`skills`（新目录格式）、`commands_DEPRECATED`（旧版 /commands/ 目录）、`plugin`（插件提供）、`mcp`（MCP server 提供）。
- **注意**：`bundled`（内置打包技能）和 `managed` 不在过滤条件中，因此不会出现在该菜单里。

### 3.4 分组与排序

```ts
const groups = {
  policySettings: [],
  userSettings: [],
  projectSettings: [],
  localSettings: [],   // 分配了数组但后续未被渲染
  flagSettings: [],    // 同上
  plugin: [],
  mcp: [],
};
```

分组后执行：
```ts
for (const group of Object.values(groups)) {
  group.sort(_temp2);
}
function _temp2(a, b) {
  return getCommandName(a).localeCompare(getCommandName(b));
}
```

渲染顺序硬编码为：
1. `projectSettings`
2. `userSettings`
3. `policySettings`
4. `plugin`
5. `mcp`

`localSettings` 与 `flagSettings` 虽然在 groups 对象中存在，但 **没有被 `renderSkillGroup` 调用**，因此实际上不可见。

### 3.5 副标题生成逻辑 (`getSourceSubtitle`)

| 来源 | 副标题内容 |
|------|-----------|
| `mcp` | 提取所有 skill name 中 `:` 前面的 server 名，去重后逗号拼接。例如 `server1:skillA`、`server1:skillB` → `server1`。 |
| 其他 | 调用 `getSkillsPath(source, 'skills')` 得到路径，再经 `getDisplayPath` 简化为相对路径或 `~` 缩写；若该组包含任何 `loadedFrom === 'commands_DEPRECATED'` 的技能，则额外追加 `commands` 路径。 |

### 3.6 单条技能渲染 (`renderSkill` / `_temp3`)

```tsx
<Box key={`${skill.name}-${skill.source}`}>
  <Text>{getCommandName(skill)}</Text>
  <Text dimColor={true}>
    {pluginName ? ` · ${pluginName}` : ""}
    {" · "}
    {tokenDisplay}{" description tokens"}
  </Text>
</Box>
```

- `getCommandName(skill)` 优先使用 `skill.userFacingName()`，否则 `skill.name`。
- `tokenDisplay` 由 `estimateSkillFrontmatterTokens(skill)` + `formatTokens(...)` 生成，前缀带 `~`。
- Plugin 技能会额外展示 `pluginManifest.name`。

### 3.7 空态渲染

当 `skills.length === 0` 时：
- Dialog 标题：`Skills`，副标题：`No skills found`
- 子节点 1：`<Text dimColor>Create skills in .claude/skills/ or ~/.claude/skills/</Text>`
- 子节点 2：`<ConfigurableShortcutHint action="confirm:no" context="Confirmation" fallback="Esc" description="close" />`

### 3.8 退出行为

```ts
const handleCancel = () => {
  onExit("Skills dialog dismissed", { display: "system" });
};
```

- 点击 `Esc` 或触发 `confirm:no` 键绑定时，调用 `onExit`，返回一条系统级消息（`display: "system"`），随后对话框关闭。

---

## 四、关键代码路径与文件引用

### 4.1 直接依赖（import 链）

| 被导入符号 | 来源文件 | 用途 |
|-----------|---------|------|
| `Command`, `CommandBase`, `CommandResultDisplay`, `getCommandName`, `PromptCommand` | `src/commands.ts` | 类型与名称解析 |
| `Box`, `Text` | `src/ink.ts` | Ink 终端 UI 组件 |
| `estimateSkillFrontmatterTokens`, `getSkillsPath` | `src/skills/loadSkillsDir.ts` | Token 估算与路径获取 |
| `getDisplayPath` | `src/utils/file.ts` | 路径简化（相对路径 / `~`） |
| `formatTokens` | `src/utils/format.ts` | 数字格式化（1.2k 等） |
| `getSettingSourceName`, `SettingSource` | `src/utils/settings/constants.ts` | 来源名称映射 |
| `plural` | `src/utils/stringUtils.ts` | 单复数处理 |
| `ConfigurableShortcutHint` | `src/components/ConfigurableShortcutHint.tsx` | 快捷键提示 |
| `Dialog` | `src/components/design-system/Dialog.tsx` | 对话框容器 |

### 4.2 调用方（谁使用了 SkillsMenu）

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/commands/skills/skills.tsx` | 直接调用 | `call()` 函数创建 `<SkillsMenu onExit={onDone} commands={context.options.commands} />` |
| `src/commands/skills/index.ts` | 命令注册 | 将 `skills` 注册为 `local-jsx` 类型命令，懒加载 `./skills.js`（编译产物） |
| `src/commands.ts` | 命令聚合 | 在 `COMMANDS` 数组中引入 `skills` 命令，使其可被 REPL 解析 |

### 4.3 数据上游（commands 从哪来）

`context.options.commands` 的填充链路：

```
src/commands.ts:getCommands(cwd)
    ├── loadAllCommands(cwd)
    │   ├── getSkills(cwd)
    │   │   ├── getSkillDirCommands(cwd)   ← 文件型技能（/skills/、/commands/）
    │   │   ├── getPluginSkills()          ← 插件技能
    │   │   ├── getBundledSkills()         ← 内置打包技能（不进入 SkillsMenu）
    │   │   └── getBuiltinPluginSkillCommands()
    │   ├── getPluginCommands()
    │   ├── getWorkflowCommands()
    │   └── COMMANDS()                     ← 内置命令（含 skills 命令自身）
    ├── getDynamicSkills()                 ← 运行时动态发现的技能
    └── 过滤 availability / isEnabled
```

---

## 五、依赖与外部交互

### 5.1 与技能加载系统的交互

`SkillsMenu` 本身 **只读展示**，不直接调用任何 I/O 或技能加载函数。它依赖父组件传入的 `commands` 数组，而该数组由 `src/commands.ts` 中的 `getCommands()` 异步组装。因此：
- 技能加载失败（如目录权限问题）不会导致 `SkillsMenu` 崩溃；上游会 catch 错误并返回空数组。
- 动态技能（`getDynamicSkills`）在文件操作后可能新增，用户再次执行 `/skills` 即可看到更新后的列表。

### 5.2 与键绑定系统的交互

`SkillsMenu` 通过 `Dialog` 组件间接使用键绑定：
- `Dialog` 内部调用 `useKeybinding("confirm:no", onCancel, ...)`。
- `SkillsMenu` 在空态和非空态都嵌入了 `<ConfigurableShortcutHint action="confirm:no" ... fallback="Esc" description="close" />`，向用户提示关闭方式。

### 5.3 与 Token 估算服务的交互

`estimateSkillFrontmatterTokens`（`src/skills/loadSkillsDir.ts`）实现：

```ts
export function estimateSkillFrontmatterTokens(skill: Command): number {
  const frontmatterText = [skill.name, skill.description, skill.whenToUse]
    .filter(Boolean)
    .join(' ')
  return roughTokenCountEstimation(frontmatterText)
}
```

- 仅估算 `name` + `description` + `whenToUse` 的 token 数，不包含完整 markdown 内容（内容只在实际调用时才加载）。
- `roughTokenCountEstimation` 来自 `src/services/tokenEstimation.js`。

---

## 六、风险、边界与改进建议

### 6.1 风险

1. **编译产物直接入仓，维护性极差**
   - 当前 `SkillsMenu.tsx` 是 React Compiler 的输出，充斥着 `_c(35)`、`$[0]`、`t14` 等机器生成代码。
   - 风险：人工直接修改极易引入 memo cache 不一致的 bug；且下次构建会被覆盖。
   - 建议：确认构建流程，确保源码修改后重新编译并替换该文件，或恢复手写 TSX 源码。

2. **`localSettings` 与 `flagSettings` 分组被声明但未被渲染**
   - `groups` 对象包含这两个 key，但 `renderSkillGroup` 从未对它们调用，导致如果未来有技能来源被标记为 `localSettings` 或 `flagSettings`，会在过滤后静默丢失。
   - 建议：要么在渲染顺序中补全，要么从 `groups` 中移除以减少误导。

3. **`bundled` 技能被过滤排除**
   - `getBundledSkills()` 加载的技能 `loadedFrom` 为 `bundled`，但 `_temp` 过滤条件未包含它，因此用户无法通过 `/skills` 查看内置打包技能。
   - 建议：确认这是产品意图还是遗漏；若意图是「只展示用户可编辑/配置的技能」，应在代码中补充注释说明。

4. **无单元测试覆盖**
   - 全局搜索未找到任何针对 `SkillsMenu` 或 `skills.tsx` 的测试文件。
   - 风险：UI 布局调整、分组逻辑变更、空态文案修改都缺乏自动化回归保障。

5. **MCP 技能名称解析的脆弱性**
   - `getSourceSubtitle` 对 `mcp` 来源使用 `s.name.indexOf(':')` 来提取 server 名。
   - 边界：若 skill name 不含 `:` 或 `:` 在首位，会返回 `null` 并被过滤，导致该 skill 的 server 信息不展示，但不会报错。

6. **`commands_DEPRECATED` 路径拼接的潜在问题**
   - `getSourceSubtitle` 中，当 `hasCommandsSkills` 为 true 时，会同时展示 `skills` 和 `commands` 路径。
   - 若 `source` 为 `mcp` 或 `plugin`，`getSkillsPath` 返回固定字符串（如 `'plugin'`），此时 `getDisplayPath('plugin')` 会原样返回 `'plugin'`，副标题可能显得无意义。

### 6.2 边界行为

| 场景 | 行为 |
|------|------|
| `commands` 为空数组 | 渲染空态，提示创建路径 |
| `commands` 包含非 `prompt` 类型 | 被过滤，不展示 |
| `commands` 包含 `loadedFrom === 'bundled'` | 被过滤，不展示 |
| 组内无元素 | `renderSkillGroup` 返回 `null`，不渲染该组标题 |
| 多个来源有技能 | 按固定顺序堆叠展示，组间 `gap={1}` |
| 用户按 Esc | `handleCancel` → `onExit` → 系统消息 "Skills dialog dismissed" |

### 6.3 改进建议

1. **源码与编译产物分离**
   - 将手写 TSX 保留在 `src/components/skills/SkillsMenu.tsx`，编译产物输出到 `dist/` 或 `build/`，避免机器代码污染源码树。

2. **补全分组或清理死代码**
   - 若 `localSettings` / `flagSettings` 确实不应出现，从 `groups` 初始化中删除；若应出现，补到渲染顺序中。

3. **增加测试**
   - 至少为以下场景补充快照或单元测试：
     - 空态渲染
     - 多来源分组与排序
     - MCP server 名提取
     - Token 估算展示格式化

4. **统一副标题逻辑**
   - 对 `plugin` / `mcp` 来源，避免调用 `getSkillsPath` 获取无意义的固定字符串；可在 `getSourceSubtitle` 开头对这两种来源做短路处理。

5. **类型安全增强**
   - `SkillSource` 当前是 `SettingSource | 'plugin' | 'mcp'`，但 `SettingSource` 包含 `localSettings`、`flagSettings` 等实际上未被渲染的值。建议拆分为「可展示来源」与「完整来源」两个类型，利用 TS 编译期排除非法值。

---

## 附录：关键代码片段索引

- **过滤函数**：`src/components/skills/SkillsMenu.tsx` L234-236 (`_temp`)
- **分组初始化**：`src/components/skills/SkillsMenu.tsx` L64-72
- **排序回调**：`src/components/skills/SkillsMenu.tsx` L231-233 (`_temp2`)
- **单条渲染**：`src/components/skills/SkillsMenu.tsx` L225-229 (`_temp3`)
- **来源标题**：`src/components/skills/SkillsMenu.tsx` L24-32 (`getSourceTitle`)
- **来源副标题**：`src/components/skills/SkillsMenu.tsx` L33-46 (`getSourceSubtitle`)
- **空态渲染**：`src/components/skills/SkillsMenu.tsx` L101-125
- **主渲染出口**：`src/components/skills/SkillsMenu.tsx` L47-223 (`SkillsMenu`)
