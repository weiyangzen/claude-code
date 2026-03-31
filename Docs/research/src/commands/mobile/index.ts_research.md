# 研究文档: src/commands/mobile/index.ts

## 场景与职责

本文件是 `/mobile` slash 命令的入口定义文件（Command Entry Point），职责包括：
1. **命令元数据声明**：定义命令名称、别名、描述、类型等属性
2. **懒加载配置**：通过 `load` 函数实现组件级代码分割，延迟加载实际实现
3. **类型安全**：使用 TypeScript `satisfies` 关键字确保对象符合 `Command` 类型约束

该命令属于 Claude Code CLI 的内置命令系统，用户可通过输入 `/mobile`、`/ios` 或 `/android` 触发，用于在终端内展示 Claude 移动应用的下载二维码。

## 功能点目的

### 1. 命令注册与发现
- **名称**：`mobile`（主命令）
- **别名**：`ios`、`android`（用户可直接输入平台特定别名）
- **描述**："Show QR code to download the Claude mobile app"
- **类型**：`local-jsx`（本地 JSX 组件型命令，渲染交互式 TUI 界面）

### 2. 懒加载优化
- 使用动态导入 `() => import('./mobile.js')` 延迟加载 `mobile.tsx` 的实现
- 避免启动时加载 `qrcode` 等重型依赖，提升 CLI 冷启动速度

## 具体技术实现

### 数据结构
```typescript
const mobile = {
  type: 'local-jsx',           // 命令类型：本地 JSX 组件
  name: 'mobile',              // 主命令名
  aliases: ['ios', 'android'], // 别名数组
  description: '...',          // 用户可见描述
  load: () => import('./mobile.js'), // 懒加载工厂函数
} satisfies Command
```

### 关键流程
1. **命令解析**：用户在 REPL 输入 `/mobile` 或别名
2. **命令查找**：`findCommand()` 在 `commands.ts` 中匹配到本定义
3. **类型分发**：识别为 `local-jsx` 类型，调用 `load()` 获取模块
4. **组件渲染**：执行 `mobile.js` 导出的 `call()` 函数，返回 React 节点
5. **UI 展示**：Ink 渲染 `MobileQRCode` 组件，显示二维码界面

## 关键代码路径与文件引用

### 直接依赖
| 路径 | 用途 |
|------|------|
| `../../commands.js` | 导入 `Command` 类型定义 |
| `./mobile.js` | 懒加载目标，实际 JSX 实现 |

### 调用链
```
用户输入 /mobile
  → REPL 组件 (src/components/REPL.tsx)
  → executeCommand() / handleSlashCommand()
  → commands.ts findCommand()
  → 匹配 mobile 命令定义
  → 调用 load() → import('./mobile.js')
  → mobile.tsx call() 函数
  → MobileQRCode 组件渲染
```

### 注册位置
- **导入点**：`src/commands.ts` 第 34 行 `import mobile from './commands/mobile/index.js'`
- **注册点**：`src/commands.ts` 第 289 行放入 `COMMANDS` 数组
- **远程安全命令**：`src/commands.ts` 第 636 行加入 `REMOTE_SAFE_COMMANDS` 集合，允许在 `--remote` 模式下使用

## 依赖与外部交互

### 类型依赖
- `Command` 类型来自 `src/commands.ts`，包含以下关键字段：
  - `type`: `'local-jsx'` | `'local'` | `'prompt'`
  - `name`: 命令唯一标识
  - `aliases`: 别名数组（可选）
  - `description`: 用户可见描述
  - `load`: 懒加载函数，返回 `Promise<LocalJSXCommandModule>`

### 运行时依赖
- 无直接运行时依赖（所有依赖通过懒加载推迟到 `mobile.tsx`）

## 风险、边界与改进建议

### 当前风险
| 风险点 | 描述 | 等级 |
|--------|------|------|
| **路径硬编码** | `import('./mobile.js')` 假设编译后文件为 `.js` 扩展名，若构建配置变更可能导致模块解析失败 | 低 |
| **无错误边界** | 若 `mobile.js` 加载失败（如文件损坏），错误处理依赖上层调用方 | 低 |

### 边界情况
1. **别名冲突**：`ios`/`android` 作为别名，若未来添加其他平台命令需注意命名空间
2. **类型安全**：`satisfies Command` 在编译期验证，但运行时无法保证 `load()` 返回值的正确性

### 改进建议
1. **添加加载错误处理**：可在 `load` 中包装错误处理逻辑，提供用户友好的降级提示
2. **类型导出**：考虑显式导出 `MobileCommand` 类型，便于测试和外部引用
3. **文档注释**：添加 JSDoc 说明别名用途（如 `ios` 直接显示 iOS 二维码）

---

*文档生成时间：2026-04-01*
*关联文件：mobile.tsx, commands.ts, types/command.ts*
