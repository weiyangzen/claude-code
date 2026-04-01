# outputStyles.ts 深度研究文档

## 场景与职责

`outputStyles.ts` 是 Claude Code CLI 中管理输出样式（Output Styles）系统的核心模块。它定义了不同的 AI 响应风格（如解释性、学习模式），支持用户自定义样式，并允许插件强制应用特定样式。

### 核心使用场景
1. **个性化响应风格**：用户可选择不同的 AI 响应风格（默认、解释性、学习模式）
2. **教育场景支持**：学习模式通过"边做边学"帮助用户理解代码
3. **插件扩展**：插件可注册自定义输出样式并强制应用
4. **项目级配置**：支持通过 `.claude/output-styles/` 目录自定义样式
5. **分层配置**：支持用户级、项目级、托管策略级样式覆盖

---

## 功能点目的

### 1. 输出样式类型定义

```typescript
export type OutputStyleConfig = {
  name: string                    // 样式名称（唯一标识）
  description: string             // 样式描述
  prompt: string                  // 注入到系统提示词的指令
  source: SettingSource | 'built-in' | 'plugin'  // 来源
  keepCodingInstructions?: boolean // 是否保留编码指令
  forceForPlugin?: boolean        // 插件是否强制应用此样式
}

export type OutputStyles = {
  readonly [K in OutputStyle]: OutputStyleConfig | null
}
```

### 2. 内置样式

#### 默认样式 (`default`)
- **值**：`null`（无特殊样式）
- **行为**：标准 Claude Code 响应风格

#### 解释性样式 (`Explanatory`)
- **描述**：Claude 解释其实现选择和代码库模式
- **特点**：
  - 在编写代码前后提供简短的教育性解释
  - 使用 Insight 区块格式化（带星号分隔线）
  - 关注代码库特定的洞察，而非通用编程概念

**Insight 格式示例**：
```
☆ Insight ─────────────────────────────────────
[2-3 关键教育点]
─────────────────────────────────────────────────
```

#### 学习模式 (`Learning`)
- **描述**：Claude 暂停并要求用户编写小段代码进行实践
- **特点**：
  - 协作和鼓励性语气
  - 在生成 20+ 行代码时请求用户贡献 2-10 行
  - 聚焦设计决策、业务逻辑、关键算法
  - 集成 TodoList 跟踪学习进度

**"边做边学"请求格式**：
```
∙ **Learn by Doing**
**Context:** [已构建内容和决策重要性]
**Your Task:** [具体函数/代码段，提及文件和 TODO(human)]
**Guidance:** [权衡和约束]
```

**关键规则**：
- 在代码中添加 `TODO(human)` 标记
- 请求后不再输出任何内容，等待用户实现
- 用户贡献后分享一个洞察

### 3. 样式加载优先级

样式按以下优先级合并（低到高）：
1. **内置样式**（`OUTPUT_STYLE_CONFIG`）
2. **插件样式**
3. **托管策略样式**（`policySettings`）
4. **用户设置样式**（`userSettings`）
5. **项目设置样式**（`projectSettings`）

高优先级样式覆盖低优先级同名样式。

### 4. 插件强制样式

插件可通过 `forceForPlugin: true` 强制应用其输出样式：
- 多个插件强制样式时，仅应用第一个（记录警告日志）
- 通过 `logForDebugging` 记录选择的样式
- 适用于插件需要特定响应格式的场景

---

## 具体技术实现

### 数据结构

```typescript
// 内置样式配置
export const DEFAULT_OUTPUT_STYLE_NAME = 'default'

export const OUTPUT_STYLE_CONFIG: OutputStyles = {
  [DEFAULT_OUTPUT_STYLE_NAME]: null,
  Explanatory: {
    name: 'Explanatory',
    source: 'built-in',
    description: 'Claude explains its implementation choices and codebase patterns',
    keepCodingInstructions: true,
    prompt: `...`,  // 解释性指令
  },
  Learning: {
    name: 'Learning',
    source: 'built-in',
    description: 'Claude pauses and asks you to write small pieces of code for hands-on practice',
    keepCodingInstructions: true,
    prompt: `...`,  // 学习模式指令
  },
}
```

### 关键函数

#### `getAllOutputStyles(cwd: string)`
- **功能**：获取所有可用样式（内置 + 自定义 + 插件）
- **缓存**：使用 `memoize` 缓存结果
- **异步**：读取文件系统和插件样式

#### `getOutputStyleConfig()`
- **功能**：获取当前应用的输出样式配置
- **逻辑**：
  1. 检查插件强制样式
  2. 从设置读取用户选择的样式
  3. 返回对应配置或 `null`

#### `clearAllOutputStylesCache()`
- **功能**：清除样式缓存
- **用途**：配置变更后刷新

#### `hasCustomOutputStyle()`
- **功能**：检查用户是否选择了非默认样式
- **用途**：UI 显示和遥测

### 关键代码路径

#### 1. 系统提示词构造路径

```
构造系统提示词
    ↓
src/constants/prompts.ts 的 getSystemPrompt()
    ↓
调用 getOutputStyleConfig()
    ↓
获取当前样式配置
    ↓
调用 getOutputStyleSection() 生成样式指令
    ↓
注入到系统提示词
```

**关键代码**（`src/constants/prompts.ts`）：
```typescript
function getOutputStyleSection(outputStyleConfig: OutputStyleConfig | null): string | null {
  if (outputStyleConfig === null) return null
  
  return `# Output Style: ${outputStyleConfig.name}
${outputStyleConfig.prompt}`
}
```

**关键文件引用**：
- `src/constants/prompts.ts`: 系统提示词生成

#### 2. 自定义样式加载路径

```
获取所有样式
    ↓
getAllOutputStyles(cwd)
    ↓
调用 getOutputStyleDirStyles(cwd)  // src/outputStyles/loadOutputStylesDir.ts
    ↓
读取 .claude/output-styles/ 目录
    ↓
解析样式配置文件
    ↓
调用 loadPluginOutputStyles()  // src/utils/plugins/loadPluginOutputStyles.ts
    ↓
加载插件注册的样式
    ↓
按优先级合并
```

**关键文件引用**：
- `src/outputStyles/loadOutputStylesDir.ts`: 自定义样式目录加载
- `src/utils/plugins/loadPluginOutputStyles.ts`: 插件样式加载

#### 3. 设置界面路径

```
用户打开设置
    ↓
src/components/Settings/Config.tsx
    ↓
显示可用样式列表
    ↓
用户选择样式
    ↓
src/components/OutputStylePicker.tsx
    ↓
预览样式效果
```

**关键文件引用**：
- `src/components/Settings/Config.tsx`: 设置界面
- `src/components/OutputStylePicker.tsx`: 样式选择器

#### 4. 状态栏显示路径

```
渲染状态栏
    ↓
src/components/StatusLine.tsx
    ↓
检查 hasCustomOutputStyle()
    ↓
[有自定义样式] 显示样式指示器
```

**关键文件引用**：
- `src/components/StatusLine.tsx`: 状态栏

#### 5. CLI 打印路径

```
CLI 输出
    ↓
src/cli/print.ts
    ↓
根据输出样式调整格式
```

**关键文件引用**：
- `src/cli/print.ts`: CLI 输出

#### 6. 消息系统路径

```
处理消息
    ↓
src/utils/messages.ts
    ↓
考虑输出样式调整消息格式
    ↓
src/utils/messages/systemInit.ts
    ↓
系统初始化时应用样式
```

**关键文件引用**：
- `src/utils/messages.ts`: 消息工具
- `src/utils/messages/systemInit.ts`: 消息系统初始化

---

## 依赖与外部交互

### 内部依赖

| 导入 | 用途 |
|------|------|
| `figures` | 符号图标（如星号） |
| `lodash-es/memoize.js` | 样式缓存 |
| `../outputStyles/loadOutputStylesDir.js` | 加载自定义样式目录 |
| `../utils/config.js` 的 `OutputStyle` | 样式类型 |
| `../utils/cwd.js` 的 `getCwd` | 获取当前工作目录 |
| `../utils/debug.js` 的 `logForDebugging` | 调试日志 |
| `../utils/plugins/loadPluginOutputStyles.js` | 加载插件样式 |
| `../utils/settings/constants.js` 的 `SettingSource` | 设置来源类型 |
| `../utils/settings/settings.js` 的 `getSettings_DEPRECATED` | 获取用户设置 |

### 被依赖方

| 文件 | 使用的常量/函数 | 用途 |
|------|----------------|------|
| `src/constants/prompts.ts` | `getOutputStyleConfig`, `OutputStyleConfig` | 系统提示词 |
| `src/outputStyles/loadOutputStylesDir.ts` | `OutputStyleConfig` | 样式加载 |
| `src/cli/print.ts` | `getOutputStyleConfig` | CLI 输出 |
| `src/components/Settings/Config.tsx` | `getAllOutputStyles` | 设置界面 |
| `src/components/OutputStylePicker.tsx` | `getAllOutputStyles`, `OutputStyleConfig` | 样式选择器 |
| `src/components/StatusLine.tsx` | `hasCustomOutputStyle` | 状态栏 |
| `src/utils/messages/systemInit.ts` | `OutputStyleConfig` | 消息初始化 |
| `src/utils/messages.ts` | `OutputStyleConfig` | 消息处理 |
| `src/utils/plugins/schemas.ts` | `OutputStyleConfig` | 插件 schema |
| `src/utils/plugins/loadPluginOutputStyles.ts` | `OutputStyleConfig` | 插件样式加载 |
| `src/utils/plugins/cacheUtils.ts` | `OutputStyleConfig` | 插件缓存 |
| `src/utils/promptCategory.ts` | `OutputStyleConfig` | 提示词分类 |

---

## 风险、边界与改进建议

### 当前风险

1. **提示词注入风险**
   - 自定义样式和插件样式的 `prompt` 直接注入系统提示词
   - 恶意样式可能尝试提示词注入攻击

2. **缓存失效复杂性**
   - `getAllOutputStyles` 使用 memoize 缓存
   - 文件系统变更后需要手动清除缓存
   - 插件动态加载可能导致缓存不一致

3. **样式冲突**
   - 多个插件强制样式时仅应用第一个
   - 用户可能不理解为什么选择的样式未生效

4. **弃用 API 使用**
   - 使用 `getSettings_DEPRECATED` 获取设置
   - 需要迁移到新的设置系统

### 边界情况

| 场景 | 行为 |
|------|------|
| 同名样式 | 高优先级覆盖低优先级 |
| 插件强制样式冲突 | 应用第一个，记录警告 |
| 样式文件解析失败 | 可能抛出异常或静默跳过 |
| 缓存清除后并发访问 | 可能多次读取文件系统 |
| 空 prompt | 注入空内容，可能影响提示词结构 |

### 改进建议

1. **样式验证**
   ```typescript
   // 建议添加样式内容验证
   export function validateOutputStyle(style: OutputStyleConfig): void {
     // 检查 prompt 长度
     if (style.prompt.length > MAX_PROMPT_LENGTH) {
       throw new Error(`Prompt too long for style ${style.name}`)
     }
     
     // 检查禁止关键词
     if (containsForbiddenPatterns(style.prompt)) {
       throw new Error(`Prompt contains forbidden patterns`)
     }
   }
   ```

2. **样式预览**
   ```typescript
   // 建议添加样式预览功能
   export function previewOutputStyle(style: OutputStyleConfig): string {
     // 返回应用样式后的示例响应
     return generatePreview(style.prompt)
   }
   ```

3. **样式版本控制**
   ```typescript
   // 建议添加版本信息
   export type OutputStyleConfig = {
     // ... 现有字段
     version: string
     minCliVersion: string
     deprecated?: boolean
   }
   ```

4. **异步缓存刷新**
   ```typescript
   // 建议添加文件系统监听
   export function watchOutputStyles(cwd: string): void {
     const watcher = fs.watch('.claude/output-styles/', () => {
       clearAllOutputStylesCache()
     })
   }
   ```

5. **样式继承**
   ```typescript
   // 建议支持样式继承
   export type OutputStyleConfig = {
     // ... 现有字段
     extends?: string  // 父样式名称
   }
   
   // 合并父样式和子样式
   export function resolveOutputStyle(style: OutputStyleConfig): OutputStyleConfig {
     if (!style.extends) return style
     const parent = findStyle(style.extends)
     return mergeStyles(parent, style)
   }
   ```

6. **国际化支持**
   ```typescript
   // 建议支持多语言样式
   export type OutputStyleConfig = {
     // ... 现有字段
     i18n: Record<string, { description: string; prompt: string }>
   }
   ```

### 与提示词系统的关系

```
outputStyles.ts (样式定义)
    ↓ 被使用
constants/prompts.ts (系统提示词)
    ↓ 注入
系统提示词中的 Output Style 区块
    ↓ 影响
AI 模型响应风格
```

输出样式系统通过修改系统提示词来影响 AI 行为，是一种声明式的行为控制机制。
