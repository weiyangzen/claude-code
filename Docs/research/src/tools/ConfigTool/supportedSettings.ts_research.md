# supportedSettings.ts 深度研究文档

## 1. 场景与职责

supportedSettings.ts 是 ConfigTool 的设置注册中心，负责定义所有支持的配置项及其元数据。该文件是 ConfigTool 的核心数据层，提供了配置项的声明式定义和查询接口。

## 2. 功能点目的

### 2.1 设置注册表（SUPPORTED_SETTINGS）
集中定义所有可配置项，包括：
- 设置键名（如 `theme`, `model`, `verbose`）
- 存储源（全局配置或项目设置）
- 值类型（布尔或字符串）
- 描述信息
- 可选的有效值列表或生成函数
- 可选的 AppState 同步键
- 可选的验证和格式化函数

### 2.2 设置查询接口
提供一组工具函数用于查询设置元数据：
- `isSupported(key)`: 检查设置是否受支持
- `getConfig(key)`: 获取设置的完整配置
- `getAllKeys()`: 获取所有设置键名
- `getOptionsForSetting(key)`: 获取设置的有效选项
- `getPath(key)`: 获取设置的存储路径

## 3. 具体技术实现

### 3.1 核心类型定义

```typescript
type SyncableAppStateKey = 'verbose' | 'mainLoopModel' | 'thinkingEnabled'

type SettingConfig = {
  source: 'global' | 'settings'      // 配置存储位置
  type: 'boolean' | 'string'         // 值类型
  description: string                // 用户可见描述
  path?: string[]                    // 嵌套路径（如 ['permissions', 'defaultMode']）
  options?: readonly string[]        // 固定选项列表
  getOptions?: () => string[]        // 动态选项生成
  appStateKey?: SyncableAppStateKey  // AppState 同步键
  validateOnWrite?: (v) => Promise<{valid, error?}> // 异步验证
  formatOnRead?: (v) => unknown      // 读取格式化
}
```

### 3.2 设置注册表

```typescript
export const SUPPORTED_SETTINGS: Record<string, SettingConfig> = {
  // 全局设置
  theme: {
    source: 'global',
    type: 'string',
    description: 'Color theme for the UI',
    options: feature('AUTO_THEME') ? THEME_SETTINGS : THEME_NAMES,
  },
  editorMode: {
    source: 'global',
    type: 'string',
    description: 'Key binding mode',
    options: EDITOR_MODES,
  },
  verbose: {
    source: 'global',
    type: 'boolean',
    description: 'Show detailed debug output',
    appStateKey: 'verbose',
  },
  
  // 项目设置
  model: {
    source: 'settings',
    type: 'string',
    description: 'Override the default model',
    appStateKey: 'mainLoopModel',
    getOptions: () => {
      try {
        return getModelOptions()
          .filter(o => o.value !== null)
          .map(o => o.value as string)
      } catch {
        return ['sonnet', 'opus', 'haiku']
      }
    },
    validateOnWrite: v => validateModel(String(v)),
    formatOnRead: v => (v === null ? 'default' : v),
  },
  
  // 条件编译设置（功能开关控制）
  ...(feature('VOICE_MODE')
    ? {
        voiceEnabled: {
          source: 'settings',
          type: 'boolean',
          description: 'Enable voice dictation (hold-to-talk)',
        },
      }
    : {}),
  
  ...(feature('BRIDGE_MODE')
    ? {
        remoteControlAtStartup: {
          source: 'global',
          type: 'boolean',
          description: 'Enable Remote Control for all sessions',
          formatOnRead: () => getRemoteControlAtStartup(),
        },
      }
    : {}),
  
  // Ant 内部用户专属设置
  ...(process.env.USER_TYPE === 'ant'
    ? {
        classifierPermissionsEnabled: {
          source: 'settings',
          type: 'boolean',
          description: 'Enable AI-based classification for Bash permission rules',
        },
      }
    : {}),
}
```

### 3.3 查询函数实现

```typescript
export function isSupported(key: string): boolean {
  return key in SUPPORTED_SETTINGS
}

export function getConfig(key: string): SettingConfig | undefined {
  return SUPPORTED_SETTINGS[key]
}

export function getAllKeys(): string[] {
  return Object.keys(SUPPORTED_SETTINGS)
}

export function getOptionsForSetting(key: string): string[] | undefined {
  const config = SUPPORTED_SETTINGS[key]
  if (!config) return undefined
  if (config.options) return [...config.options]
  if (config.getOptions) return config.getOptions()
  return undefined
}

export function getPath(key: string): string[] {
  const config = SUPPORTED_SETTINGS[key]
  return config?.path ?? key.split('.')
}
```

## 4. 关键代码路径与文件引用

### 4.1 本文件导出

| 导出 | 类型 | 用途 |
|------|------|------|
| `SUPPORTED_SETTINGS` | 常量对象 | 设置注册表 |
| `isSupported` | 函数 | 检查设置是否受支持 |
| `getConfig` | 函数 | 获取设置配置 |
| `getAllKeys` | 函数 | 获取所有设置键 |
| `getOptionsForSetting` | 函数 | 获取设置选项 |
| `getPath` | 函数 | 获取设置路径 |

### 4.2 依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `bun:bundle` | `feature` | 功能开关检查 |
| `../../utils/config.ts` | `getRemoteControlAtStartup` | 远程控制默认值 |
| `../../utils/configConstants.ts` | `EDITOR_MODES`, `NOTIFICATION_CHANNELS`, `TEAMMATE_MODES` | 固定选项列表 |
| `../../utils/model/modelOptions.ts` | `getModelOptions` | 模型选项生成 |
| `../../utils/model/validateModel.ts` | `validateModel` | 模型验证 |
| `../../utils/theme.ts` | `THEME_NAMES`, `THEME_SETTINGS` | 主题选项 |

### 4.3 被引用位置

| 文件 | 用途 |
|------|------|
| `ConfigTool.ts` | 设置验证、路径解析、选项检查 |
| `prompt.ts` | 生成设置列表提示 |

## 5. 依赖与外部交互

### 5.1 功能开关集成

使用条件展开运算符根据功能开关动态添加设置：

```typescript
// VOICE_MODE 功能开启时添加 voiceEnabled 设置
...(feature('VOICE_MODE') ? { voiceEnabled: {...} } : {})

// BRIDGE_MODE 功能开启时添加 remoteControlAtStartup 设置
...(feature('BRIDGE_MODE') ? { remoteControlAtStartup: {...} } : {})

// KAIROS 或 KAIROS_PUSH_NOTIFICATION 功能开启时添加通知设置
...(feature('KAIROS') || feature('KAIROS_PUSH_NOTIFICATION')
  ? { taskCompleteNotifEnabled: {...}, ... }
  : {})
```

### 5.2 用户类型区分

通过 `process.env.USER_TYPE` 区分内部（Ant）和外部用户：

```typescript
...(process.env.USER_TYPE === 'ant'
  ? { classifierPermissionsEnabled: {...} }
  : {})
```

### 5.3 模型系统集成

`model` 设置具有最复杂的配置：
- **动态选项**：通过 `getModelOptions()` 获取用户可用的模型列表
- **异步验证**：通过 `validateModel()` 在设置时验证模型可用性
- **格式化显示**：将 `null` 值显示为 `'default'`
- **AppState 同步**：变更同步到 `mainLoopModel`

### 5.4 主题系统集成

`theme` 设置根据 `AUTO_THEME` 功能开关提供不同选项：
- `AUTO_THEME` 开启：`['auto', 'dark', 'light', ...]`
- `AUTO_THEME` 关闭：`['dark', 'light', ...]`

## 6. 风险、边界与改进建议

### 6.1 已知风险

1. **运行时依赖**：`getOptions` 和 `validateOnWrite` 是运行时函数，可能在配置查询时抛出异常
2. **功能开关扩散**：条件编译导致设置定义分散，难以一览全貌
3. **路径解析歧义**：`getPath` 默认使用 `key.split('.')`，可能与显式 `path` 配置冲突

### 6.2 边界情况

1. **嵌套路径**：`permissions.defaultMode` 需要正确解析为 `['permissions', 'defaultMode']`
2. **空选项**：`getOptionsForSetting` 可能返回 `undefined`（无限制）、空数组（无可用选项）或有值数组
3. **验证失败**：`validateOnWrite` 可能返回 `{valid: false, error: '...'}`，需要正确处理

### 6.3 改进建议

1. **类型安全增强**：为设置键添加字面量类型：

```typescript
// 建议改进
type SettingKey = 
  | 'theme' 
  | 'editorMode' 
  | 'verbose' 
  | 'model' 
  | 'voiceEnabled'
  | ...

export function getConfig<K extends SettingKey>(key: K): SettingConfig<K>
```

2. **分组管理**：按功能模块分组设置，便于管理：

```typescript
const UI_SETTINGS = { theme: {...}, editorMode: {...} }
const MODEL_SETTINGS = { model: {...}, alwaysThinkingEnabled: {...} }
const PERMISSION_SETTINGS = { 'permissions.defaultMode': {...} }

export const SUPPORTED_SETTINGS = {
  ...UI_SETTINGS,
  ...MODEL_SETTINGS,
  ...PERMISSION_SETTINGS,
  // 条件编译设置
  ...(feature('VOICE_MODE') ? VOICE_SETTINGS : {}),
}
```

3. **默认值声明**：在 `SettingConfig` 中添加默认值字段：

```typescript
type SettingConfig<T = unknown> = {
  // ...
  defaultValue: T
  // ...
}
```

4. **验证器组合**：支持多个验证器组合：

```typescript
validateOnWrite: [
  v => validateType(v, 'boolean'),
  v => validateRange(v, 0, 100),
  v => validateCustom(v),
]
```

5. **文档自动生成**：从 `SUPPORTED_SETTINGS` 自动生成用户文档

### 6.4 测试要点

1. **功能开关组合**：测试所有功能开关组合下的设置可用性
2. **选项生成**：验证 `getOptions` 在各种条件下的返回值
3. **路径解析**：测试带点号的键名和显式 `path` 配置
4. **验证函数**：模拟验证成功/失败场景
