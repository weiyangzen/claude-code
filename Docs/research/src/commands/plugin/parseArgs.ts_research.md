# parseArgs.ts 研究文档

## 场景与职责

`parseArgs.ts` 是 Claude Code 插件命令的参数解析模块，负责将用户输入的命令行参数解析为结构化的命令对象。该模块是插件子命令路由系统的核心，支持以下功能：

1. **子命令解析**：识别 `install`, `manage`, `uninstall`, `enable`, `disable`, `validate`, `marketplace` 等子命令
2. **参数提取**：解析插件名称、市场名称、路径等参数
3. **格式识别**：支持多种输入格式（如 `plugin@marketplace`, URL, 本地路径等）
4. **默认值处理**：无参数时返回菜单命令

该模块是纯函数设计，无副作用，便于测试和复用。

## 功能点目的

### 1. 子命令路由
支持以下子命令及其缩写：
- `help`, `--help`, `-h` → 帮助
- `install`, `i` → 安装
- `manage` → 管理
- `uninstall` → 卸载
- `enable` → 启用
- `disable` → 禁用
- `validate` → 验证
- `marketplace`, `market` → 市场操作

### 2. 安装参数解析
支持多种安装目标格式：
- `plugin@marketplace` → 指定插件和市场
- `http://...`, `https://...` → 市场 URL
- `file://...`, `/path/...` → 本地路径
- `plugin-name` → 仅插件名称

### 3. 市场操作解析
支持市场管理子操作：
- `add` → 添加市场
- `remove`, `rm` → 移除市场
- `update` → 更新市场
- `list` → 列出市场

### 4. 路径处理
`validate` 子命令支持包含空格的路径（如 `"/path/to/my dir"`）

## 具体技术实现

### 关键数据结构

```typescript
// 解析结果类型（联合类型）
export type ParsedCommand =
  | { type: 'menu' }
  | { type: 'help' }
  | { type: 'install'; marketplace?: string; plugin?: string }
  | { type: 'manage' }
  | { type: 'uninstall'; plugin?: string }
  | { type: 'enable'; plugin?: string }
  | { type: 'disable'; plugin?: string }
  | { type: 'validate'; path?: string }
  | {
      type: 'marketplace';
      action?: 'add' | 'remove' | 'update' | 'list';
      target?: string;
    };
```

### 核心解析算法

```typescript
export function parsePluginArgs(args?: string): ParsedCommand {
  // 1. 空参数处理
  if (!args) return { type: 'menu' };

  // 2. 分割参数
  const parts = args.trim().split(/\s+/);
  const command = parts[0]?.toLowerCase();

  // 3. 命令分发
  switch (command) {
    case 'help': case '--help': case '-h':
      return { type: 'help' };

    case 'install': case 'i': {
      const target = parts[1];
      if (!target) return { type: 'install' };

      // 检查 plugin@marketplace 格式
      if (target.includes('@')) {
        const [plugin, marketplace] = target.split('@');
        return { type: 'install', plugin, marketplace };
      }

      // 检查是否为市场路径/URL
      const isMarketplace = /* URL 检测逻辑 */;
      if (isMarketplace) {
        return { type: 'install', marketplace: target };
      }

      // 默认作为插件名称
      return { type: 'install', plugin: target };
    }

    // ... 其他命令处理
  }
}
```

### 市场检测逻辑

```typescript
const isMarketplace =
  target.startsWith('http://') ||
  target.startsWith('https://') ||
  target.startsWith('file://') ||
  target.includes('/') ||
  target.includes('\\');
```

### 路径处理

```typescript
case 'validate': {
  // 使用 slice(1).join(' ') 保留路径中的空格
  const target = parts.slice(1).join(' ').trim();
  return { type: 'validate', path: target || undefined };
}
```

## 关键代码路径与文件引用

### 直接依赖

该模块无外部依赖，是纯 TypeScript 逻辑。

### 调用方

| 文件路径 | 用途 |
|---------|------|
| `src/commands/plugin/PluginSettings.tsx` | 初始化视图状态 |
| `src/commands/plugin/plugin.tsx` | 可能间接使用 |

### 使用流程

```
用户输入: /plugin install my-plugin@official
              │
              ▼
PluginSettings.tsx
  ├── parsePluginArgs("install my-plugin@official")
  │       └── { type: 'install', plugin: 'my-plugin', marketplace: 'official' }
  │
  └── getInitialViewState(parsedCommand)
          └── { type: 'browse-marketplace', targetMarketplace: 'official', targetPlugin: 'my-plugin' }
```

## 依赖与外部交互

### 解析结果消费

解析结果在 `PluginSettings.tsx` 中被转换为视图状态：

```typescript
// PluginSettings.tsx
function getInitialViewState(parsedCommand: ParsedCommand): ViewState {
  switch (parsedCommand.type) {
    case 'install':
      if (parsedCommand.marketplace) {
        return {
          type: 'browse-marketplace',
          targetMarketplace: parsedCommand.marketplace,
          targetPlugin: parsedCommand.plugin
        };
      }
      if (parsedCommand.plugin) {
        return {
          type: 'discover-plugins',
          targetPlugin: parsedCommand.plugin
        };
      }
      return { type: 'discover-plugins' };

    case 'manage':
      return { type: 'manage-plugins' };

    case 'uninstall':
      return {
        type: 'manage-plugins',
        targetPlugin: parsedCommand.plugin,
        action: 'uninstall'
      };

    // ... 其他情况
  }
}
```

### 输入示例与输出映射

| 用户输入 | 解析结果 | 视图状态 |
|---------|---------|---------|
| `/plugin` | `{ type: 'menu' }` | `{ type: 'discover-plugins' }` |
| `/plugin help` | `{ type: 'help' }` | 显示帮助 |
| `/plugin install` | `{ type: 'install' }` | 发现插件界面 |
| `/plugin install foo` | `{ type: 'install', plugin: 'foo' }` | 发现插件，预选 foo |
| `/plugin install foo@bar` | `{ type: 'install', plugin: 'foo', marketplace: 'bar' }` | 浏览 bar 市场，预选 foo |
| `/plugin install https://...` | `{ type: 'install', marketplace: 'https://...' }` | 浏览该 URL 市场 |
| `/plugin manage` | `{ type: 'manage' }` | 管理插件界面 |
| `/plugin uninstall foo` | `{ type: 'uninstall', plugin: 'foo' }` | 管理界面，预选卸载 foo |
| `/plugin enable foo` | `{ type: 'enable', plugin: 'foo' }` | 管理界面，预选启用 foo |
| `/plugin disable foo` | `{ type: 'disable', plugin: 'foo' }` | 管理界面，预选禁用 foo |
| `/plugin validate /path/to` | `{ type: 'validate', path: '/path/to' }` | 验证界面 |
| `/plugin marketplace add url` | `{ type: 'marketplace', action: 'add', target: 'url' }` | 添加市场界面 |
| `/plugin marketplace list` | `{ type: 'marketplace', action: 'list' }` | 市场列表 |

## 风险、边界与改进建议

### 潜在风险

1. **歧义解析**
   - `plugin@marketplace` 格式使用简单 `split('@')`
   - 如果插件名包含 `@`（虽然不推荐），会错误解析
   - **示例**：`my@plugin@market` → `plugin: 'my'`, `marketplace: 'plugin@market'`

2. **路径检测误判**
   - 使用 `includes('/')` 检测路径，可能误判
   - **示例**：`my-plugin/name` 会被误判为路径而非插件名
   - 实际影响较小，因为有效插件名不应包含 `/`

3. **大小写敏感**
   - 命令转小写处理，但参数保持原样
   - 市场名称大小写敏感可能导致问题

### 边界情况

| 场景 | 当前行为 | 备注 |
|-----|---------|------|
| `args = undefined` | `{ type: 'menu' }` | 默认到菜单 |
| `args = ''` | `{ type: 'menu' }` | 空字符串处理 |
| `args = '   '` | `{ type: 'menu' }` | 空白字符处理后为空 |
| `args = 'unknown'` | `{ type: 'menu' }` | 未知命令默认到菜单 |
| `args = 'install @market'` | `{ type: 'install', plugin: '', marketplace: 'market' }` | 空插件名 |
| `args = 'install plugin@'` | `{ type: 'install', plugin: 'plugin', marketplace: '' }` | 空市场名 |
| `args = 'validate path with spaces'` | `{ type: 'validate', path: 'path with spaces' }` | 正确处理空格 |

### 改进建议

1. **更严格的插件名验证**
   ```typescript
   // 建议：验证插件名格式
   if (plugin && !/^[a-z0-9-]+$/.test(plugin)) {
     console.warn(`Plugin name "${plugin}" may contain invalid characters`);
   }
   ```

2. **支持引号参数**
   ```typescript
   // 建议：支持引号包裹的参数
   // /plugin install "my plugin"@market
   const parts = args.match(/[^\s"']+|"([^"]*)"|'([^']*)'/g) || [];
   ```

3. **错误提示增强**
   ```typescript
   // 建议：对未知命令给出提示
   default:
     console.log(`Unknown command: ${command}. Use /plugin help for usage.`);
     return { type: 'menu' };
   ```

4. **参数验证**
   ```typescript
   // 建议：验证必填参数
   case 'uninstall':
     if (!parts[1]) {
       return { type: 'error', message: 'Plugin name required for uninstall' };
     }
     return { type: 'uninstall', plugin: parts[1] };
   ```

5. **类型安全增强**
   ```typescript
   // 建议：使用更精确的类型
   type SubCommand = 'install' | 'manage' | 'uninstall' | ...;
   const VALID_COMMANDS: SubCommand[] = ['install', 'manage', ...];
   ```

6. **文档生成**
   ```typescript
   // 建议：从代码生成帮助文档
   export const COMMAND_DOCS = {
     install: {
       syntax: '/plugin install [plugin[@marketplace]|marketplace]',
       examples: ['/plugin install foo', '/plugin install foo@bar']
     }
   } as const;
   ```

7. **模糊匹配**
   ```typescript
   // 建议：支持命令缩写
   case 'i':
   case 'inst':
   case 'install':
     return { type: 'install', ... };
   ```

### 测试建议

1. **单元测试矩阵**：
   ```typescript
   describe('parsePluginArgs', () => {
     test.each([
       [undefined, { type: 'menu' }],
       ['help', { type: 'help' }],
       ['install foo', { type: 'install', plugin: 'foo' }],
       ['install foo@bar', { type: 'install', plugin: 'foo', marketplace: 'bar' }],
       // ... 更多用例
     ])('parsePluginArgs(%p) = %p', (input, expected) => {
       expect(parsePluginArgs(input)).toEqual(expected);
     });
   });
   ```

2. **边界测试**：
   - 空字符串、空白字符、特殊字符
   - 超长输入
   - Unicode 字符

3. **模糊测试**：
   - 随机字符串输入
   - 验证不抛出异常

4. **集成测试**：
   - 验证与 `PluginSettings.tsx` 的集成
   - 验证视图状态正确性
