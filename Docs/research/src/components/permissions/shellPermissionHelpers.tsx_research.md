# shellPermissionHelpers.tsx 研究文档

## 场景与职责

`shellPermissionHelpers.tsx` 是 Claude Code CLI 权限系统的 **Shell 权限对话框辅助函数模块**。它提供了用于生成 Shell 工具（Bash、PowerShell）权限对话框中"Yes, and apply suggestions"选项标签的辅助函数。

该模块解决了 Shell 权限对话框的**用户体验问题**：
- 根据建议的规则类型（Read 规则、Shell 规则、目录权限）生成人类可读的标签
- 处理多种规则组合的复杂情况
- 提供一致的标签格式和截断逻辑

## 功能点目的

### 1. 智能建议标签生成
根据建议的权限更新生成适当的标签文本：
- **仅 Read 规则**："Yes, allow reading from {path}"
- **仅目录权限**："Yes, and always allow access to {directory}"
- **仅 Shell 命令**："Yes, and don't ask again for {commands}"
- **混合情况**：组合上述格式

### 2. 列表格式化
提供多种列表格式化方式：
- `commandListDisplay`：格式化命令列表（支持 0、1、2、多个）
- `commandListDisplayTruncated`：带截断的列表显示
- `formatPathList`：格式化路径列表

### 3. 命令前缀提取
从权限规则内容中提取命令前缀：
- 支持 `command:*` 格式的规则
- 可选的命令转换（如去除输出重定向）

## 具体技术实现

### 核心数据结构

```typescript
// 主要导出函数
export function generateShellSuggestionsLabel(
  suggestions: PermissionUpdate[],
  shellToolName: string,
  commandTransform?: (command: string) => string
): ReactNode | null;

// 内部使用的辅助函数
function commandListDisplay(commands: string[]): ReactNode;
function commandListDisplayTruncated(commands: string[]): ReactNode;
function formatPathList(paths: string[]): ReactNode;
```

### 关键流程

1. **规则分类**：
   ```typescript
   const allRules = suggestions
     .filter(s => s.type === 'addRules')
     .flatMap(s => s.rules || []);
   
   const readRules = allRules.filter(r => r.toolName === 'Read');
   const shellRules = allRules.filter(r => r.toolName === shellToolName);
   const directories = suggestions
     .filter(s => s.type === 'addDirectories')
     .flatMap(s => s.directories || []);
   ```

2. **路径提取**：
   ```typescript
   // 从 Read 规则提取路径，移除 /** 后缀
   const readPaths = readRules
     .map(r => r.ruleContent?.replace('/**', '') || '')
     .filter(p => p);
   ```

3. **命令提取**：
   ```typescript
   const shellCommands = [...new Set(shellRules.flatMap(rule => {
     if (!rule.ruleContent) return [];
     const command = permissionRuleExtractPrefix(rule.ruleContent) ?? rule.ruleContent;
     return commandTransform ? commandTransform(command) : command;
   }))];
   ```

4. **标签生成逻辑**：
   ```
   仅 Read 规则 → "Yes, allow reading from {path}"
   仅目录权限 → "Yes, and always allow access to {directory}"
   仅 Shell 命令 → "Yes, and don't ask again for {commands} in {cwd}"
   Read+目录（无命令）→ "Yes, and always allow access to {paths}"
   混合（有命令）→ "Yes, and allow access to {paths} and {commands}"
   ```

### 列表格式化实现

```typescript
// 命令列表格式化
function commandListDisplay(commands: string[]): ReactNode {
  switch (commands.length) {
    case 0: return '';
    case 1: return <Text bold>{commands[0]}</Text>;
    case 2: return <Text><Text bold>{commands[0]}</Text> and <Text bold>{commands[1]}</Text></Text>;
    default: return <Text><Text bold>{commands.slice(0, -1).join(', ')}</Text>, and <Text bold>{commands.slice(-1)[0]}</Text></Text>;
  }
}

// 截断版本
function commandListDisplayTruncated(commands: string[]): ReactNode {
  const plainText = commands.join(', ');
  if (plainText.length > 50) {
    return 'similar';  // 太长时显示 "similar"
  }
  return commandListDisplay(commands);
}

// 路径列表格式化
function formatPathList(paths: string[]): ReactNode {
  const names = paths.map(p => basename(p) || p);
  if (names.length === 1) {
    return <Text><Text bold>{names[0]}</Text>{sep}</Text>;
  }
  if (names.length === 2) {
    return <Text><Text bold>{names[0]}</Text>{sep} and <Text bold>{names[1]}</Text>{sep}</Text>;
  }
  return <Text><Text bold>{names[0]}</Text>{sep}, <Text bold>{names[1]}</Text>{sep} and {paths.length - 2} more</Text>;
}
```

### 关键代码路径

```typescript
// 路径处理
import { basename, sep } from 'path';

// React 类型
import React, { type ReactNode } from 'react';

// 获取原始工作目录
import { getOriginalCwd } from '../../bootstrap/state.js';

// Ink 组件
import { Text } from '../../ink.js';

// 权限更新类型
import type { PermissionUpdate } from '../../utils/permissions/PermissionUpdateSchema.js';

// 规则前缀提取
import { permissionRuleExtractPrefix } from '../../utils/permissions/shellRuleMatching.js';
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `basename`, `sep` | `path` | 路径处理 |
| `React`, `ReactNode` | `react` | React 类型 |
| `getOriginalCwd` | `../../bootstrap/state.js` | 获取原始工作目录 |
| `Text` | `../../ink.js` | Ink 文本组件 |
| `PermissionUpdate` | `../../utils/permissions/PermissionUpdateSchema.js` | 权限更新类型 |
| `permissionRuleExtractPrefix` | `../../utils/permissions/shellRuleMatching.js` | 提取命令前缀 |

### Shell 规则匹配

`permissionRuleExtractPrefix` 来自 `shellRuleMatching.ts`：

```typescript
export function permissionRuleExtractPrefix(permissionRule: string): string | null {
  const match = permissionRule.match(/^(.+):\*$/);
  return match?.[1] ?? null;
}
```

支持从 `npm:*` 格式的规则中提取前缀 `npm`。

### 被调用方

该模块被以下组件使用：

1. **BashPermissionRequest.tsx** - Bash 权限请求
   ```typescript
   const suggestionsLabel = generateShellSuggestionsLabel(
     toolUseConfirm.permissionResult.suggestions ?? [],
     BashTool.name,
     stripOutputRedirections,  // 去除输出重定向
   );
   ```

2. **PowerShellPermissionRequest.tsx** - PowerShell 权限请求
   ```typescript
   const suggestionsLabel = generateShellSuggestionsLabel(
     toolUseConfirm.permissionResult.suggestions ?? [],
     PowerShellTool.name,
     // 无命令转换
   );
   ```

### 数据流

```
权限检查结果 (PermissionResult.suggestions)
  ↓
BashPermissionRequest / PowerShellPermissionRequest
  ↓
generateShellSuggestionsLabel(suggestions, toolName, transform?)
  ↓
分类规则（Read / Shell / Directories）
  ↓
提取路径和命令
  ↓
根据组合情况生成标签
  ↓
渲染为 ReactNode
```

## 风险、边界与改进建议

### 当前风险

1. **硬编码的截断阈值**：
   - `plainText.length > 50` 是硬编码的
   - 不考虑终端宽度

2. **路径分隔符假设**：
   - 使用 `sep` 假设路径分隔符
   - 在跨平台场景下可能有问题

3. **规则类型硬编码**：
   - `'addRules'`、`'addDirectories'` 是硬编码字符串
   - 如果类型定义改变，需要同步更新

4. **英文文本硬编码**：
   - 所有标签文本都是英文
   - 不支持国际化

### 边界情况

1. **空建议数组**：
   - 如果 `suggestions` 为空，所有过滤结果为空
   - 函数返回 `null`

2. **空规则内容**：
   - `ruleContent` 可能为 undefined
   - 使用 `?.replace('/**', '') || ''` 安全处理

3. **重复命令**：
   - 使用 `new Set` 去重
   - 确保命令列表唯一

4. **长路径列表**：
   - 超过 2 个路径时显示 "and N more"
   - 避免标签过长

### 改进建议

1. **动态截断阈值**：
   ```typescript
   function commandListDisplayTruncated(commands: string[], maxLength: number = 50): ReactNode {
     // 允许传入最大长度
   }
   ```

2. **国际化支持**：
   ```typescript
   const labels = {
     allowReading: t('shell.allowReading'),
     allowAccess: t('shell.allowAccess'),
     dontAskAgain: t('shell.dontAskAgain'),
     // ...
   };
   ```

3. **更智能的路径显示**：
   - 显示相对路径而不是仅 basename
   - 对于深层嵌套路径，使用 `.../parent/dir` 格式

4. **命令分组**：
   - 如果命令有共同前缀，分组显示
   - 如 "npm run (test, build, lint)"

5. **工具提示**：
   - 添加悬停提示显示完整列表
   - 当内容被截断时特别有用

6. **可配置性**：
   - 允许通过设置自定义标签格式
   - 适应不同团队的偏好

7. **测试覆盖**：
   - 添加单元测试覆盖所有组合情况
   - 特别是边界情况（空数组、长列表等）

8. **类型安全**：
   ```typescript
   // 使用 const assertion 确保类型安全
   const SUGGESTION_TYPES = ['addRules', 'addDirectories', ...] as const;
   type SuggestionType = typeof SUGGESTION_TYPES[number];
   ```
