# ApproveApiKey.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`ApproveApiKey` 是一个**安全确认对话框组件**，用于处理环境变量中检测到自定义 API Key 的场景。当系统发现用户设置了 `ANTHROPIC_API_KEY` 环境变量时，显示此对话框让用户确认是否使用该 Key。

### 1.2 使用场景
- **安全启动检查**：启动时检测环境变量中的 API Key
- **用户确认流程**：防止恶意项目通过 `.env` 文件注入 API Key
- **信任建立**：用户批准后，该 Key 的截断形式会被记录到已批准列表

### 1.3 安全背景
```
场景：恶意项目包含 .env 文件设置 ANTHROPIC_API_KEY
风险：用户可能在不知情的情况下使用攻击者的 API Key
防护：首次检测到 Key 时显示确认对话框
```

---

## 2. 功能点目的

### 2.1 核心功能

| 功能点 | 目的 |
|--------|------|
| API Key 显示 | 显示截断的 API Key (`sk-ant-...{suffix}`) |
| 用户确认 | 提供 Yes/No 选项让用户选择 |
| 配置持久化 | 将用户选择保存到全局配置 |
| 默认安全 | 默认选择 "No"（推荐），防止误操作 |

### 2.2 Props 接口

```typescript
type Props = {
  customApiKeyTruncated: string;  // API Key 截断后缀（用于显示和记录）
  onDone(approved: boolean): void; // 完成回调，true=批准, false=拒绝
}
```

### 2.3 用户流程

```
检测到 ANTHROPIC_API_KEY
        │
        ▼
┌─────────────────────┐
│ 显示 ApproveApiKey  │
│ 对话框              │
└─────────────────────┘
        │
    ┌───┴───┐
    ▼       ▼
  Yes      No (默认)
    │       │
    ▼       ▼
保存到     保存到
approved   rejected
列表       列表
    │       │
    └───┬───┘
        ▼
   onDone(布尔值)
```

---

## 3. 具体技术实现

### 3.1 状态管理

```typescript
// 用户选择处理
function onChange(value: 'yes' | 'no') {
  switch (value) {
    case 'yes':
      saveGlobalConfig(current => ({
        ...current,
        customApiKeyResponses: {
          ...current.customApiKeyResponses,
          approved: [
            ...(current.customApiKeyResponses?.approved ?? []),
            customApiKeyTruncated  // 添加当前 Key 到已批准列表
          ]
        }
      }))
      onDone(true)
      break
    case 'no':
      saveGlobalConfig(current => ({
        ...current,
        customApiKeyResponses: {
          ...current.customApiKeyResponses,
          rejected: [
            ...(current.customApiKeyResponses?.rejected ?? []),
            customApiKeyTruncated  // 添加当前 Key 到已拒绝列表
          ]
        }
      }))
      onDone(false)
  }
}
```

### 3.2 配置数据结构

```typescript
// GlobalConfig 中的相关字段
type GlobalConfig = {
  customApiKeyResponses?: {
    approved?: string[];   // 已批准的 Key 截断后缀列表
    rejected?: string[];   // 已拒绝的 Key 截断后缀列表
  }
  // ... 其他配置
}
```

### 3.3 UI 渲染

```tsx
<Dialog 
  title="Detected a custom API key in your environment" 
  color="warning" 
  onCancel={() => onChange("no")}
>
  {/* API Key 显示 */}
  <Text>
    <Text bold>ANTHROPIC_API_KEY</Text>
    <Text>: sk-ant-...{customApiKeyTruncated}</Text>
  </Text>
  
  {/* 问题文本 */}
  <Text>Do you want to use this API key?</Text>
  
  {/* 选择器 */}
  <Select 
    defaultValue="no"
    defaultFocusValue="no"
    options={[
      { label: "Yes", value: "yes" },
      { label: "No (recommended)", value: "no" }  // 推荐选项加粗
    ]}
    onChange={...}
    onCancel={() => onChange("no")}
  />
</Dialog>
```

### 3.4 关键依赖

**saveGlobalConfig** (`src/utils/config.ts`):
```typescript
export function saveGlobalConfig(
  updater: (currentConfig: GlobalConfig) => GlobalConfig
): void
```
- 使用文件锁防止并发写入冲突
- 包含重入保护避免递归
- 回退机制处理写入失败

**Dialog** (`src/components/design-system/Dialog.js`):
- 提供统一的对话框样式
- 支持标题、边框颜色、取消回调
- 内置键盘快捷键处理

**Select** (`src/components/CustomSelect/index.js`):
- 自定义选择器组件
- 支持键盘导航
- 可配置默认选项

---

## 4. 关键代码路径与文件引用

### 4.1 文件位置
```
src/components/ApproveApiKey.tsx
```

### 4.2 依赖图

```
ApproveApiKey.tsx
├── react (React 核心)
├── ../ink.js (Text 组件)
├── ../utils/config.js
│   └── saveGlobalConfig
├── ./CustomSelect/index.js
│   └── Select 组件
└── ./design-system/Dialog.js
    └── Dialog 组件
```

### 4.3 配置存储路径

```
~/.claude.json (全局配置文件)
└── customApiKeyResponses
    ├── approved: ["abc123", "def456", ...]
    └── rejected: ["xyz789", ...]
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| React | 'react' | 组件运行时 |
| Text | '../ink.js' | 文本渲染 |
| saveGlobalConfig | '../utils/config.js' | 配置持久化 |
| Select | './CustomSelect/index.js' | 选择器组件 |
| Dialog | './design-system/Dialog.js' | 对话框容器 |

### 5.2 数据流

```
用户交互
    │
    ▼
Select onChange
    │
    ├── 'yes' → saveGlobalConfig (approved)
    │             └── ~/.claude.json
    │
    └── 'no'  → saveGlobalConfig (rejected)
                  └── ~/.claude.json
    │
    ▼
onDone(approved: boolean)
    │
    ▼
父组件处理后续逻辑
```

### 5.3 配置验证流程

```
启动时
    │
    ▼
检测 ANTHROPIC_API_KEY
    │
    ▼
获取 Key 截断后缀
    │
    ▼
检查 ~/.claude.json
    │
    ├── 在 approved 列表中 → 直接使用
    ├── 在 rejected 列表中 → 拒绝使用
    └── 都不在 → 显示 ApproveApiKey 对话框
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 严重程度 |
|------|------|----------|
| 截断碰撞 | 不同 Key 可能有相同截断后缀 | 中 |
| 配置损坏 | ~/.claude.json 损坏导致配置丢失 | 中 |
| 社会工程 | 攻击者可能使用看似合法的 Key 名称 | 低 |
| 误批准 | 用户可能误点击 "Yes" | 低（有默认保护） |

### 6.2 边界情况

1. **空截断后缀**：`customApiKeyTruncated` 为空字符串时的处理
2. **重复批准**：同一 Key 多次批准会添加重复条目
3. **配置并发**：多进程同时修改配置的竞争条件
4. **取消操作**：用户按 Esc 触发 `onCancel`，默认拒绝

### 6.3 改进建议

1. **安全性增强**：
   ```typescript
   // 添加 Key 指纹验证
   const keyFingerprint = hashApiKey(apiKey)
   // 存储指纹而非截断后缀
   ```

2. **用户体验**：
   - 显示 Key 的创建时间（如果可从 Key 解码）
   - 添加 "记住我的选择，不再询问" 选项
   - 提供查看已批准/已拒绝 Key 列表的管理界面

3. **代码质量**：
   - 提取配置更新逻辑为独立函数
   - 添加单元测试覆盖各种用户选择场景
   - 使用更严格的类型定义避免字符串字面量错误

4. **可观察性**：
   ```typescript
   // 添加分析事件
   logEvent('tengu_api_key_approval', {
     approved: boolean,
     keySuffix: customApiKeyTruncated
   })
   ```

5. **配置管理**：
   - 定期清理过期的 rejected 条目
   - 添加配置迁移逻辑处理旧格式
   - 支持按项目单独配置（而非全局）

### 6.4 相关配置项

```typescript
// ~/.claude.json
{
  "customApiKeyResponses": {
    "approved": ["a1b2c3", "d4e5f6"],
    "rejected": ["x9y8z7"]
  }
}
```
