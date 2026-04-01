# PromptInputModeIndicator.tsx 深度研究文档

> **研究对象**: `src/components/PromptInput/PromptInputModeIndicator.tsx`  
> **研究范围**: 源码、调用方、Agent Swarms 颜色系统、主题系统及相关依赖  
> **执行器**: kimi (k2p5)  
> **研究日期**: 2026-04-01

---

## 1. 场景与职责

### 1.1 核心定位

`PromptInputModeIndicator.tsx` 是 Claude Code CLI 中**输入框左侧的模式指示器组件**，负责渲染一个小的提示符（`❯` 或 `!`），向用户传达当前输入所处的模式以及正在查看的 Agent 身份。

### 1.2 应用场景

| 场景 | 说明 |
|------|------|
| **普通对话模式** | 渲染默认的 `❯` 提示符 |
| **Bash 模式** | 渲染 `! ` 前缀，提示用户输入将直接作为 shell 命令执行 |
| **查看 Teammate 模式** | 当用户通过 footer 导航进入某个 teammate 的 transcript 时，显示该 teammate 的 `❯` 提示符并应用其主题色 |
| **Agent Swarms 启用时** | 当前会话如果是 teammate，使用其分配的颜色渲染提示符 |

### 1.3 职责边界

- **最小化展示组件**：只渲染一个提示符，不处理任何输入逻辑
- **颜色状态桥接**：将 Agent Swarms 的颜色系统与 Ink 主题系统桥接起来
- **加载状态反馈**：通过 `dimColor={isLoading}` 在加载时降低提示符亮度

---

## 2. 功能点目的

### 2.1 模式视觉反馈

通过不同的前缀字符让用户一眼识别当前输入模式：

| 模式 | 渲染内容 |
|------|----------|
| `bash` | `<Text color="bashBorder">! </Text>` |
| `prompt`（默认） | `<PromptChar>❯ </PromptChar>` |
| 查看 teammate | `<PromptChar themeColor={viewedTeammateThemeColor}>❯ </PromptChar>` |

### 2.2 Teammate 颜色标识

在 Agent Swarms 模式下，每个 teammate 被分配一种颜色（red/blue/green/yellow/purple/orange/pink/cyan）。该组件：

1. 读取当前 teammate 的颜色（通过 `getTeammateColor()`）
2. 将颜色名映射到主题系统的颜色键（通过 `AGENT_COLOR_TO_THEME_COLOR`）
3. 将主题色应用到 `Text` 组件的 `color` 属性

### 2.3 查看中的 Teammate 高亮

当用户正在查看某个 teammate 的 transcript 时（`viewingAgentName` 存在），提示符使用该 teammate 的颜色，帮助用户明确当前输入将发送给谁。

---

## 3. 具体技术实现

### 3.1 组件 Props

```typescript
type Props = {
  mode: PromptInputMode;
  isLoading: boolean;
  viewingAgentName?: string;
  viewingAgentColor?: AgentColorName;
};
```

### 3.2 颜色映射逻辑

```typescript
function getTeammateThemeColor(): keyof Theme | undefined {
  if (!isAgentSwarmsEnabled()) {
    return undefined;
  }
  const colorName = getTeammateColor();
  if (!colorName) {
    return undefined;
  }
  if (AGENT_COLORS.includes(colorName as AgentColorName)) {
    return AGENT_COLOR_TO_THEME_COLOR[colorName as AgentColorName];
  }
  return undefined;
}
```

### 3.3 PromptChar 子组件

```typescript
type PromptCharProps = {
  isLoading: boolean;
  themeColor?: keyof Theme;
};

function PromptChar({ isLoading, themeColor }: PromptCharProps) {
  const color = themeColor ?? undefined;
  return (
    <Text color={color} dimColor={isLoading}>
      {figures.pointer} 
    </Text>
  );
}
```

> 注：源码中有一个 `false ? "subtle" : undefined` 的死代码分支，实际效果就是 `undefined`。

### 3.4 主渲染逻辑

```typescript
export function PromptInputModeIndicator({
  mode,
  isLoading,
  viewingAgentName,
  viewingAgentColor
}: Props) {
  const teammateColor = getTeammateThemeColor();
  const viewedTeammateThemeColor = viewingAgentColor 
    ? AGENT_COLOR_TO_THEME_COLOR[viewingAgentColor] 
    : undefined;

  return (
    <Box alignItems="flex-start" alignSelf="flex-start" flexWrap="nowrap" justifyContent="flex-start">
      {viewingAgentName 
        ? <PromptChar isLoading={isLoading} themeColor={viewedTeammateThemeColor} />
        : mode === "bash" 
          ? <Text color="bashBorder" dimColor={isLoading}>! </Text>
          : <PromptChar isLoading={isLoading} themeColor={isAgentSwarmsEnabled() ? teammateColor : undefined} />
      }
    </Box>
  );
}
```

### 3.5 React Compiler 编译特征

源码经过 React Compiler 编译：
- `PromptChar` 使用 `_c(3)` 缓存
- `PromptInputModeIndicator` 使用 `_c(6)` 缓存
- `getTeammateThemeColor()` 的结果通过 `Symbol.for("react.memo_cache_sentinel")` 模式在首次渲染时计算并缓存

---

## 4. 关键代码路径与文件引用

### 4.1 组件入口

| 路径 | 说明 |
|------|------|
| `src/components/PromptInput/PromptInputModeIndicator.tsx` | 主组件文件（编译后输出） |
| `src/components/PromptInput/PromptInput.tsx` | 直接调用方，传入 `mode`、`isLoading`、`viewingAgentName`、`viewingAgentColor` |
| `src/components/PromptInput/PromptInputFooterSuggestions.tsx` | 同目录其他组件（无直接调用关系） |

### 4.2 核心依赖

| 路径 | 说明 |
|------|------|
| `figures` | npm 包，提供跨平台 Unicode 符号（`pointer` = `❯`） |
| `src/ink.js` | `Box`、`Text` 组件 |
| `src/tools/AgentTool/agentColorManager.ts` | `AGENT_COLOR_TO_THEME_COLOR`、`AGENT_COLORS`、`AgentColorName` |
| `src/types/textInputTypes.ts` | `PromptInputMode` 类型 |
| `src/utils/teammate.ts` | `getTeammateColor()` |
| `src/utils/theme.ts` | `Theme` 类型 |
| `src/utils/agentSwarmsEnabled.ts` | `isAgentSwarmsEnabled()` |

### 4.3 颜色系统链路

```
PromptInputModeIndicator.tsx
  → getTeammateColor() [src/utils/teammate.ts]
    → getTeammateContext() [src/utils/teammateContext.ts] (in-process)
    → dynamicTeamContext (tmux teammates)
  → AGENT_COLOR_TO_THEME_COLOR [src/tools/AgentTool/agentColorManager.ts]
    → Theme color key [src/utils/theme.ts]
  → Ink <Text color={...} />
```

---

## 5. 依赖与外部交互

### 5.1 Agent Swarms 启用检查

```typescript
isAgentSwarmsEnabled()
```

该函数是 Agent Swarms 功能的总开关：
- Ant 构建：始终返回 `true`
- 外部构建：需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 环境变量或 `--agent-teams` 标志，且 GrowthBook `tengu_amber_flint` 开关开启

### 5.2 Teammate 颜色来源

 teammate 颜色可能来自两个来源：
1. **In-process teammates**：通过 `AsyncLocalStorage` 存储在 `teammateContext.ts` 中
2. **Tmux teammates**：通过 CLI 参数传入，存储在模块级 `dynamicTeamContext` 变量中

### 5.3 调用方数据准备

在 `PromptInput.tsx` 中，查看中的 teammate 颜色经过严格校验：

```typescript
const viewingAgentColor = viewedTeammate?.identity.color 
  && AGENT_COLORS.includes(viewedTeammate.identity.color as AgentColorName) 
  ? viewedTeammate.identity.color as AgentColorName 
  : undefined;
```

这确保了传入 `PromptInputModeIndicator` 的颜色一定是有效的 `AgentColorName`。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 |
|------|------|
| **死代码分支** | `const color = teammateColor ?? (false ? "subtle" : undefined);` 中 `"subtle"` 分支永远不会执行 |
| **注释与实现不一致** | `PromptCharProps` 的注释说 `themeColor` 参数是为了避免 "teammate" 字符串出现在外部构建中，但实际代码中并无此用途 |
| **颜色回退不明确** | 当 `themeColor` 为 `undefined` 时，Ink `Text` 组件使用默认前景色，但用户可能期望有一个明确的颜色回退 |
| **无测试覆盖** | 未找到针对该组件的单元测试 |

### 6.2 边界情况

- **Agent Swarms 未启用**：`getTeammateThemeColor()` 直接返回 `undefined`，提示符使用默认颜色
- **Teammate 无颜色**：返回 `undefined`，与未启用时表现一致
- **无效颜色名**：`AGENT_COLORS.includes()` 检查失败，返回 `undefined`
- **同时满足 `viewingAgentName` 和 `mode === "bash"`**：`viewingAgentName` 优先级更高，显示 teammate 的 `❯` 而非 `!`

### 6.3 改进建议

1. **清理死代码**：移除 `false ? "subtle" : undefined` 分支，或替换为有意义的颜色回退逻辑
2. **更新注释**：`PromptCharProps` 的注释与实际实现不符，应修正或删除
3. **增加颜色回退**：当 teammate 颜色未设置时，可考虑使用一个柔和的默认主题色（如 `subtle`）来区分普通模式
4. **增加单元测试**：测试不同 `mode`、`viewingAgentName`、`viewingAgentColor` 组合下的渲染输出
5. **考虑提取为纯函数**：`getTeammateThemeColor()` 依赖全局状态，测试时可能需要 mock；可考虑将颜色解析逻辑提取为接收参数的纯函数
