# prompt.ts 深度研究文档

## 1. 场景与职责

prompt.ts 负责生成 ConfigTool 的动态提示文档（prompt），该文档向 AI 模型描述 ConfigTool 的功能、可用设置列表和使用示例。提示文档是动态的，因为它需要根据当前系统状态（如可用模型、功能开关）生成不同的内容。

## 2. 功能点目的

### 2.1 工具描述（DESCRIPTION）
提供工具的简短描述：`Get or set Claude Code configuration settings.`

### 2.2 动态提示生成（generatePrompt）
根据当前系统配置生成完整的工具提示文档，包括：
- 全局设置列表（存储在 `~/.claude.json`）
- 项目设置列表（存储在 `settings.json`）
- 模型选择说明
- 使用示例

### 2.3 设置分类展示
将设置按存储源分类：
- **Global Settings**: 全局配置，影响所有项目
- **Project Settings**: 项目级配置，仅影响当前项目

## 3. 具体技术实现

### 3.1 依赖导入

```typescript
import { feature } from 'bun:bundle'
import { getModelOptions } from '../../utils/model/modelOptions.js'
import { isVoiceGrowthBookEnabled } from '../../voice/voiceModeEnabled.js'
import {
  getOptionsForSetting,
  SUPPORTED_SETTINGS,
} from './supportedSettings.js'
```

### 3.2 核心函数 generatePrompt

```typescript
export function generatePrompt(): string {
  const globalSettings: string[] = []
  const projectSettings: string[] = []

  for (const [key, config] of Object.entries(SUPPORTED_SETTINGS)) {
    // 跳过 model - 单独处理
    if (key === 'model') continue
    
    // 语音设置受 GrowthBook 开关控制
    if (
      feature('VOICE_MODE') &&
      key === 'voiceEnabled' &&
      !isVoiceGrowthBookEnabled()
    ) continue

    const options = getOptionsForSetting(key)
    let line = `- ${key}`

    if (options) {
      line += `: ${options.map(o => `"${o}"`).join(', ')}`
    } else if (config.type === 'boolean') {
      line += `: true/false`
    }

    line += ` - ${config.description}`

    if (config.source === 'global') {
      globalSettings.push(line)
    } else {
      projectSettings.push(line)
    }
  }

  const modelSection = generateModelSection()

  return `Get or set Claude Code configuration settings.

  View or change Claude Code settings...

## Usage
- **Get current value:** Omit the "value" parameter
- **Set new value:** Include the "value" parameter

## Configurable settings list
...
`
}
```

### 3.3 模型部分生成（generateModelSection）

```typescript
function generateModelSection(): string {
  try {
    const options = getModelOptions()
    const lines = options.map(o => {
      const value = o.value === null ? 'null/"default"' : `"${o.value}"`
      return `  - ${value}: ${o.descriptionForModel ?? o.description}`
    })
    return `## Model
- model - Override the default model. Available options:
${lines.join('\n')}`
  } catch {
    return `## Model
- model - Override the default model (sonnet, opus, haiku, best, or full model ID)`
  }
}
```

### 3.4 设置行格式

每个设置项格式化为：
```
- {key}: "option1", "option2" - {description}
```

或对于布尔值：
```
- {key}: true/false - {description}
```

## 4. 关键代码路径与文件引用

### 4.1 本文件导出

| 导出 | 类型 | 用途 |
|------|------|------|
| `DESCRIPTION` | 字符串常量 | 工具简短描述 |
| `generatePrompt` | 函数 | 生成完整提示文档 |

### 4.2 依赖文件

| 文件 | 导入内容 | 用途 |
|------|----------|------|
| `bun:bundle` | `feature` | 功能开关检查 |
| `../../utils/model/modelOptions.ts` | `getModelOptions` | 获取可用模型列表 |
| `../../voice/voiceModeEnabled.ts` | `isVoiceGrowthBookEnabled` | 语音功能开关检查 |
| `./supportedSettings.ts` | `getOptionsForSetting`, `SUPPORTED_SETTINGS` | 设置元数据 |

### 4.3 被引用位置

在 `ConfigTool.ts` 中：
```typescript
import { DESCRIPTION, generatePrompt } from './prompt.js'

export const ConfigTool = buildTool({
  name: CONFIG_TOOL_NAME,
  async description() {
    return DESCRIPTION
  },
  async prompt() {
    return generatePrompt()
  },
  // ...
})
```

## 5. 依赖与外部交互

### 5.1 功能开关集成

使用 `feature('VOICE_MODE')` 检查语音功能是否启用：
- 仅在 VOICE_MODE 功能开启时显示 `voiceEnabled` 设置
- 同时检查 `isVoiceGrowthBookEnabled()` 运行时开关

### 5.2 模型系统集成

通过 `getModelOptions()` 获取当前用户可用的模型列表：
- 根据用户订阅类型（Ant/Max/Pro/PAYG）返回不同选项
- 包含模型别名（sonnet/opus/haiku）和完整模型 ID
- 提供模型描述（包括定价信息）

### 5.3 设置注册表交互

遍历 `SUPPORTED_SETTINGS` 对象：
- 动态收集所有支持的设置
- 根据 `config.source` 分类到全局/项目设置
- 调用 `getOptionsForSetting` 获取每个设置的有效选项

## 6. 风险、边界与改进建议

### 6.1 已知风险

1. **模型选项获取失败**：`generateModelSection` 使用 try-catch，但失败时回退信息可能不够详细
2. **设置列表过长**：如果支持设置过多，可能导致提示文档超出模型上下文限制
3. **动态内容不一致**：`generatePrompt` 每次调用可能返回不同内容（如模型选项变化）

### 6.2 边界情况

1. **空设置列表**：如果所有设置都被功能开关过滤，可能产生空列表
2. **模型选项为空**：`getModelOptions` 返回空数组时的处理
3. **特殊字符转义**：设置描述中的特殊字符可能影响 Markdown 格式

### 6.3 改进建议

1. **缓存机制**：`generatePrompt` 结果可缓存，避免重复生成
2. **分页显示**：设置过多时可分页或分组显示
3. **搜索功能**：添加设置搜索/过滤功能
4. **默认值显示**：在提示中显示每个设置的当前默认值

```typescript
// 建议：添加默认值显示
function formatSettingLine(key: string, config: SettingConfig): string {
  const options = getOptionsForSetting(key)
  const currentValue = getCurrentValue(key) // 新增：获取当前值
  let line = `- ${key}`
  
  if (options) {
    line += `: ${options.map(o => o === currentValue ? `**"${o}"**` : `"${o}"`).join(', ')}`
  }
  line += ` - ${config.description}`
  if (currentValue !== undefined) {
    line += ` (current: ${JSON.stringify(currentValue)})`
  }
  return line
}
```

5. **国际化支持**：将硬编码的英文描述提取到翻译文件

### 6.4 测试要点

1. **功能开关组合**：测试不同功能开关组合下的提示生成
2. **模型选项变化**：模拟 `getModelOptions` 返回不同结果
3. **设置变更**：验证新增/删除设置后的提示更新
