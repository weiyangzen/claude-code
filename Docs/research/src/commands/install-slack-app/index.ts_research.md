# 研究文档: src/commands/install-slack-app/index.ts

## 场景与职责

本文件是 `install-slack-app` 命令的**入口定义文件**，负责声明并导出一个本地命令（Local Command）配置对象。该命令允许用户通过 Claude Code CLI 直接安装 Claude Slack 应用。

### 使用场景
- 用户在 Claude Code 交互式会话中输入 `/install-slack-app` 命令
- 该命令仅在用户是 claude.ai 订阅者时可用（通过 `availability: ['claude-ai']` 限制）
- 用于引导用户到 Slack Marketplace 安装 Claude Slack 应用

---

## 功能点目的

### 1. 命令注册与元数据定义
- **命令名称**: `install-slack-app`
- **命令类型**: `local`（本地执行命令，非 prompt 类型）
- **描述**: "Install the Claude Slack app"
- **可用性限制**: 仅对 `claude-ai` 订阅者可见
- **非交互模式支持**: `supportsNonInteractive: false` 表示该命令不支持非交互式执行

### 2. 懒加载实现
- 使用 `load: () => import('./install-slack-app.js')` 实现动态导入
- 延迟加载实际命令实现，减少启动时的模块加载开销
- 符合 CLI 工具的按需加载最佳实践

---

## 具体技术实现

### 数据结构

```typescript
const installSlackApp = {
  type: 'local',
  name: 'install-slack-app',
  description: 'Install the Claude Slack app',
  availability: ['claude-ai'],
  supportsNonInteractive: false,
  load: () => import('./install-slack-app.js'),
} satisfies Command
```

### 关键字段解析

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `'local'` | 命令类型为本地执行命令 |
| `name` | `string` | 命令标识符，用户通过 `/install-slack-app` 调用 |
| `description` | `string` | 命令描述，显示在帮助和自动补全中 |
| `availability` | `CommandAvailability[]` | 限制命令仅对 claude.ai 订阅者可用 |
| `supportsNonInteractive` | `boolean` | 是否支持 `-p` 非交互模式 |
| `load` | `() => Promise<LocalCommandModule>` | 懒加载函数，返回实际命令实现 |

### 类型约束
- 使用 `satisfies Command` 确保对象符合 `Command` 类型定义
- 类型定义位于 `src/types/command.ts`

---

## 关键代码路径与文件引用

### 直接依赖
```
src/commands/install-slack-app/index.ts
├── imports: '../../commands.js' (类型: Command)
└── lazy-loads: './install-slack-app.js' (实际实现)
```

### 调用链
1. **用户输入** `/install-slack-app`
2. **命令解析** → `src/commands.ts` 中的 `getCommands()` 返回命令列表
3. **命令查找** → `findCommand()` 匹配到 `installSlackApp`
4. **可用性检查** → `meetsAvailabilityRequirement()` 检查 `availability: ['claude-ai']`
5. **懒加载执行** → 调用 `load()` 动态导入 `install-slack-app.js`
6. **执行命令** → 调用 `call()` 函数打开浏览器

### 相关文件
| 文件路径 | 作用 |
|----------|------|
| `src/commands.ts` | 命令注册中心，导入并聚合所有命令 |
| `src/types/command.ts` | Command 类型定义 |
| `src/commands/install-slack-app/install-slack-app.ts` | 实际命令实现 |

---

## 依赖与外部交互

### 内部依赖
- **`../../commands.js`**: 仅导入 `Command` 类型，无运行时依赖

### 运行时加载
- **`./install-slack-app.js`**: 动态导入，包含实际的 `call()` 实现

### 命令系统集成
- 通过 `src/commands.ts` 第 31 行导入: `import installSlackApp from './commands/install-slack-app/index.js'`
- 在第 286 行注册到命令列表: `installSlackApp`

---

## 风险、边界与改进建议

### 风险评估

| 风险点 | 等级 | 说明 |
|--------|------|------|
| 可用性限制过于严格 | 低 | 仅 claude.ai 订阅者可用，可能排除部分 Console API 用户 |
| 非交互模式不支持 | 低 | 设计意图明确，该命令需要交互式浏览器操作 |
| 懒加载失败 | 低 | 动态导入失败时会在调用时抛出，上层有错误处理 |

### 边界情况
1. **用户权限**: 非 claude.ai 订阅者看不到此命令
2. **平台限制**: 依赖浏览器打开功能，在无图形界面环境会优雅降级
3. **重复点击**: 实际实现中通过 `slackAppInstallCount` 计数器追踪点击次数

### 改进建议

1. **扩展可用性**（可选）
   ```typescript
   // 考虑对 Console API 用户也开放
   availability: ['claude-ai', 'console']
   ```

2. **添加别名**（可选）
   ```typescript
   aliases: ['slack', 'slack-app']
   ```

3. **国际化支持**（长期）
   - 当前描述为硬编码英文，可考虑 i18n 支持

4. **命令分组**（可选）
   - 考虑添加 `category: 'integrations'` 字段用于帮助文档分组

### 测试建议
- 验证命令在 `isClaudeAISubscriber()` 返回 false 时被正确过滤
- 验证懒加载模块正确导出 `call` 函数
- 验证命令元数据正确显示在帮助输出中
