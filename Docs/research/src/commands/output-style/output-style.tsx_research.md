# 研究文档: src/commands/output-style/output-style.tsx

## 场景与职责

### 文件定位
本文件是 `/output-style` 命令的**实现模块**，包含命令的实际执行逻辑。由于该命令已弃用，当前实现仅返回一条迁移提示消息。

### 技术特点
- 采用 `local-jsx` 命令类型的标准实现模式
- 使用 TypeScript/TSX 语法（尽管当前实现无 JSX 元素）
- 通过回调函数 `onDone` 与命令系统通信

### 业务价值
1. **用户体验连续性**: 老用户输入熟悉命令时获得明确指引
2. **功能迁移透明**: 清晰告知替代方案（`/config` 或设置文件）
3. **零副作用**: 不修改任何状态，纯信息展示

---

## 功能点目的

### 核心功能
当用户执行 `/output-style` 命令时：
1. 显示弃用警告消息
2. 告知用户使用 `/config` 命令修改输出风格
3. 提示用户也可通过设置文件修改
4. 说明更改将在下次会话生效

### 消息内容解析
```
'/output-style has been deprecated. 
 Use /config to change your output style, 
 or set it in your settings file. 
 Changes take effect on the next session.'
```

| 信息片段 | 含义 |
|----------|------|
| `has been deprecated` | 明确声明命令已弃用 |
| `Use /config` | 引导至新入口 |
| `or set it in your settings file` | 提供替代配置方式 |
| `next session` | 设置生效时机说明 |

---

## 具体技术实现

### 1. 函数签名

```typescript
import type { LocalJSXCommandOnDone } from '../../types/command.js';

export async function call(onDone: LocalJSXCommandOnDone): Promise<undefined> {
  onDone('/output-style has been deprecated. Use /config to change your output style, or set it in your settings file. Changes take effect on the next session.', {
    display: 'system'
  });
}
```

### 2. 参数详解

**`onDone: LocalJSXCommandOnDone`**
```typescript
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
```

**调用方式**:
```typescript
onDone(message, { display: 'system' })
```

### 3. 显示模式说明

`display: 'system'` 表示：
- 消息以系统消息形式展示
- 使用暗淡/灰色样式（与系统提示一致）
- 不触发模型查询（`shouldQuery` 默认为 false）
- 适合展示状态变更、配置提示等信息

### 4. 返回值

函数返回 `Promise<undefined>`：
- 符合 `LocalJSXCommandCall` 类型签名
- 无需返回 React 元素（纯文本消息）
- 异步函数支持未来可能的异步操作扩展

---

## 关键代码路径与文件引用

### 直接依赖
| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `../../types/command.js` | `LocalJSXCommandOnDone` 类型 | 回调函数类型定义 |

### 类型定义来源
```typescript
// src/types/command.ts (Line 117-126)
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: CommandResultDisplay
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  },
) => void

// src/types/command.ts (Line 107)
export type CommandResultDisplay = 'skip' | 'system' | 'user'
```

### 被依赖方
| 文件 | 引用方式 |
|------|----------|
| `src/commands/output-style/index.ts` | `load: () => import('./output-style.js')` |

### 调用链
```
用户输入 /output-style
    ↓
src/commands.ts 路由到 outputStyle 命令
    ↓
调用 load() 动态导入本文件
    ↓
执行 call(onDone, context, args)
    ↓
onDone(message, { display: 'system' })
    ↓
REPL 渲染系统消息
```

---

## 依赖与外部交互

### 类型系统依赖
```typescript
// 来自 src/types/command.ts
export type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>
```

### 与输出风格系统的关联
虽然本命令已弃用，但"输出风格"功能本身仍然存在：

```
output-style.tsx (弃用命令)
    ↓
引导用户至 /config
    ↓
Config.tsx 打开 OutputStylePicker
    ↓
用户选择新风格
    ↓
保存到 localSettings
    ↓
下次会话生效
```

### 相关配置系统
| 模块 | 职责 |
|------|------|
| `src/constants/outputStyles.ts` | 内置风格定义、风格合并逻辑 |
| `src/components/OutputStylePicker.tsx` | 风格选择 UI 组件 |
| `src/components/Settings/Config.tsx` | 配置界面集成 |
| `src/utils/settings/settings.ts` | 设置读写 |

---

## 风险、边界与改进建议

### 当前风险

1. **功能残留**
   - 风险: 弃用代码长期存在增加维护负担
   - 现状: 7 行代码，无复杂逻辑
   - 评估: 风险极低，可接受

2. **消息国际化**
   - 风险: 硬编码英文消息，不支持多语言
   - 现状: 其他弃用命令采用相同策略
   - 建议: 若系统支持 i18n，应统一处理

3. **上下文参数未使用**
   - 风险: 函数签名未接收 `context` 和 `args` 参数
   - 现状: 对于纯提示功能无需这些参数
   - 说明: 符合 `LocalJSXCommandCall` 的最小实现

### 边界情况

1. **空参数调用**
   - 用户输入 `/output-style` 不带参数
   - 正常显示弃用提示

2. **带参数调用**
   - 用户输入 `/output-style some-arg`
   - 参数被忽略，仍显示相同提示
   - 无错误处理需求

3. **并发调用**
   - 无状态操作，可安全并发执行
   - 多次调用产生多条相同消息

### 改进建议

1. **添加 JSDoc 注释**
   ```typescript
   /**
    * @deprecated This command is deprecated. Use /config instead.
    * Displays a deprecation notice directing users to the config panel.
    */
   export async function call(onDone: LocalJSXCommandOnDone): Promise<undefined> {
     // ...
   }
   ```

2. **考虑参数提示**
   ```typescript
   export async function call(onDone: LocalJSXCommandOnDone, _context: LocalJSXCommandContext, args: string): Promise<undefined> {
     if (args.trim()) {
       onDone('Note: /output-style no longer accepts arguments. ' + DEPRECATION_MESSAGE, { display: 'system' });
       return;
     }
     // ...
   }
   ```

3. **未来移除计划**
   - 评估该命令的调用频率
   - 在版本更新日志中宣布移除计划
   - 最终删除整个 `output-style` 目录

---

## 附录: 输出风格系统完整架构

### 配置存储
```typescript
// ~/.claude/settings.json
{
  "outputStyle": "Explanatory"  // 或 "Learning", "default" 等
}
```

### 风格加载优先级
```
1. built-in (内置: default, Explanatory, Learning)
2. plugin (插件定义)
3. userSettings (用户设置目录)
4. projectSettings (项目 .claude/output-styles/)
5. policySettings (策略/托管设置)
```

### 风格应用流程
```
会话启动
    ↓
buildSystemPrompt()
    ↓
getOutputStyleConfig()
    ↓
获取所有可用风格
    ↓
根据设置选择当前风格
    ↓
将风格 prompt 注入系统提示词
    ↓
模型生成响应时应用风格
```

### 自定义风格示例
```markdown
---
name: "Concise"
description: "Minimal responses"
keep-coding-instructions: true
---

Provide extremely concise responses. Focus only on code changes.
```

文件位置：`~/.claude/output-styles/concise.md` 或 `.claude/output-styles/concise.md`
