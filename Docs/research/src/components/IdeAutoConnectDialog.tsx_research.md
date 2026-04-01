# IdeAutoConnectDialog.tsx 研究文档

## 场景与职责

IdeAutoConnectDialog 是 Claude Code CLI 的 IDE 自动连接配置对话框组件。它用于引导用户配置是否自动连接到检测到的 IDE（如 VS Code、Cursor、JetBrains 系列等），并在非支持的终端中提供相关配置选项。

**核心职责：**
- 首次使用时询问用户是否启用 IDE 自动连接
- 提供禁用自动连接的确认对话框
- 管理相关配置项的持久化
- 根据环境条件智能决定是否显示对话框

## 功能点目的

### 1. 自动连接启用对话框（IdeAutoConnectDialog）
- **触发时机**：首次在非支持终端中启动且未配置时
- **询问内容**：是否启用 IDE 自动连接功能
- **选项**：Yes / No
- **默认选项**：Yes
- **配置保存**：
  - `autoConnectIde`: 用户选择（true/false）
  - `hasIdeAutoConnectDialogBeenShown`: true（标记已显示）

### 2. 自动连接禁用对话框（IdeDisableAutoConnectDialog）
- **触发时机**：用户尝试在已启用自动连接的情况下禁用
- **询问内容**：是否确认禁用自动连接
- **选项**：No / Yes（No 在前，防止误操作）
- **默认选项**：No
- **配置保存**：
  - `autoConnectIde`: false（如果选择 Yes）

### 3. 显示条件控制
- **启用对话框显示条件**：
  - 不在支持的终端中（`!isSupportedTerminal()`）
  - 未启用自动连接（`autoConnectIde !== true`）
  - 未显示过对话框（`hasIdeAutoConnectDialogBeenShown !== true`）

- **禁用对话框显示条件**：
  - 不在支持的终端中（`!isSupportedTerminal()`）
  - 已启用自动连接（`autoConnectIde === true`）

## 具体技术实现

### 关键数据结构

```typescript
// 启用对话框属性
type IdeAutoConnectDialogProps = {
  onComplete: () => void;  // 完成回调
};

// 禁用对话框属性
type IdeDisableAutoConnectDialogProps = {
  onComplete: (disableAutoConnect: boolean) => void;  // 完成回调，参数表示是否已禁用
};

// 选择选项结构
type Option = {
  label: string;   // 显示标签
  value: string;   // 选项值（"yes" / "no"）
};
```

### 核心流程

#### 1. 启用对话框流程

```
用户选择 → handleSelect(value)
       → autoConnect = value === "yes"
       → saveGlobalConfig({
             autoConnectIde: autoConnect,
             hasIdeAutoConnectDialogBeenShown: true
         })
       → onComplete()
```

**代码实现：**
```typescript
const handleSelect = async (value: string) => {
  const autoConnect = value === "yes";
  saveGlobalConfig(current => ({
    ...current,
    autoConnectIde: autoConnect,
    hasIdeAutoConnectDialogBeenShown: true
  }));
  onComplete();
};
```

#### 2. 禁用对话框流程

```
用户选择 → handleSelect(value)
       → disableAutoConnect = value === "yes"
       → 如果 disableAutoConnect:
           saveGlobalConfig({ autoConnectIde: false })
       → onComplete(disableAutoConnect)
```

**代码实现：**
```typescript
const handleSelect = (value: string) => {
  const disableAutoConnect = value === "yes";
  if (disableAutoConnect) {
    saveGlobalConfig(current => ({
      ...current,
      autoConnectIde: false
    }));
  }
  onComplete(disableAutoConnect);
};
```

#### 3. 显示条件检查

**启用对话框：**
```typescript
export function shouldShowAutoConnectDialog(): boolean {
  const config = getGlobalConfig();
  return !isSupportedTerminal() && 
         config.autoConnectIde !== true && 
         config.hasIdeAutoConnectDialogBeenShown !== true;
}
```

**禁用对话框：**
```typescript
export function shouldShowDisableAutoConnectDialog(): boolean {
  const config = getGlobalConfig();
  return !isSupportedTerminal() && 
         config.autoConnectIde === true;
}
```

### UI 组件结构

#### IdeAutoConnectDialog

```tsx
<Dialog 
  title="Do you wish to enable auto-connect to IDE?" 
  color="ide" 
  onCancel={onComplete}
>
  <Select 
    options={[
      { label: "Yes", value: "yes" },
      { label: "No", value: "no" }
    ]} 
    onChange={handleSelect} 
    defaultValue="yes" 
  />
  <Text dimColor={true}>
    You can also configure this in /config or with the --ide flag
  </Text>
</Dialog>
```

#### IdeDisableAutoConnectDialog

```tsx
<Dialog 
  title="Do you wish to disable auto-connect to IDE?" 
  subtitle="You can also configure this in /config"
  onCancel={handleCancel}  // 取消时调用 onComplete(false)
  color="ide"
>
  <Select 
    options={[
      { label: "No", value: "no" },   // No 在前，防止误操作
      { label: "Yes", value: "yes" }
    ]} 
    onChange={handleSelect} 
    defaultValue="no" 
  />
</Dialog>
```

## 关键代码路径与文件引用

### 当前文件
- `/src/components/IdeAutoConnectDialog.tsx` - 主组件实现

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `/src/components/CustomSelect/index.ts` | Select 组件导出 |
| `/src/components/design-system/Dialog.tsx` | 对话框基础组件 |
| `/src/ink.ts` | Ink 组件（Text） |
| `/src/utils/config.ts` | 配置管理（getGlobalConfig, saveGlobalConfig） |
| `/src/utils/ide.ts` | IDE 工具函数（isSupportedTerminal） |

### 调用方

该组件由应用启动流程或配置系统调用：
- 应用启动时检查 `shouldShowAutoConnectDialog()`
- 用户通过 `/config` 修改设置时
- 用户使用 `--ide` 标志时

## 依赖与外部交互

### 配置系统

**相关配置项：**
```typescript
type GlobalConfig = {
  autoConnectIde?: boolean;                    // 是否启用自动连接
  hasIdeAutoConnectDialogBeenShown?: boolean;  // 对话框是否已显示
  // ... 其他配置
};
```

**配置保存：**
```typescript
// 保存到 ~/.claude.json
saveGlobalConfig(updater: (current: GlobalConfig) => GlobalConfig): void;
```

### 终端检测

```typescript
// 来自 utils/ide.ts
export const isSupportedTerminal = memoize(() => {
  return (
    isSupportedVSCodeTerminal() ||
    isSupportedJetBrainsTerminal() ||
    Boolean(process.env.FORCE_CODE_TERMINAL)
  );
});
```

支持的终端：
- VS Code 内置终端
- JetBrains IDE 内置终端
- 强制启用（`FORCE_CODE_TERMINAL` 环境变量）

### IDE 自动连接流程

```
应用启动
    ↓
检查是否在支持终端中
    ↓
否 → 检查 shouldShowAutoConnectDialog()
    ↓
是 → 显示 IdeAutoConnectDialog
    ↓
用户选择
    ↓
保存配置 → 继续启动流程
```

当 `autoConnectIde = true` 时：
```
启动流程
    ↓
调用 findAvailableIDE() 检测可用 IDE
    ↓
如果找到 IDE → 自动建立连接
    ↓
启用 IDE 相关功能（如 diff 集成）
```

## 风险、边界与改进建议

### 已知风险

1. **配置竞态**
   - 风险：多实例同时修改配置可能导致数据丢失
   - 缓解：`saveGlobalConfig` 使用文件锁机制

2. **用户困惑**
   - 风险：用户不理解"自动连接 IDE"的含义
   - 缓解：对话框提供额外说明文本，指向 `/config` 和 `--ide` 选项

3. **误操作禁用**
   - 风险：用户可能误操作禁用自动连接
   - 缓解：禁用对话框将 "No" 设为默认选项和首位

4. **环境变化**
   - 风险：用户从非支持终端切换到支持终端后配置可能不适用
   - 缓解：每次启动都重新检测环境

### 边界情况

| 场景 | 行为 |
|------|------|
| 在支持终端中 | 不显示任何对话框 |
| 已显示过启用对话框且选择 No | 不再显示 |
| 已启用自动连接 | 显示禁用对话框 |
| 用户取消禁用对话框 | 保持启用状态 |
| 配置读取失败 | 使用默认值（false） |

### 改进建议

1. **更多上下文信息**
   - 当前：简单询问是否启用
   - 建议：显示检测到的 IDE 信息和连接好处

2. **条件引导**
   - 当前：所有非支持终端都询问
   - 建议：只在检测到 IDE 可用时询问

3. **配置迁移**
   - 建议：支持从旧版本配置平滑迁移

4. **批量配置**
   - 建议：支持企业/团队级别的默认配置

5. **诊断信息**
   - 建议：在对话框中显示 IDE 检测状态

6. **撤销操作**
   - 当前：禁用后需要手动重新启用
   - 建议：提供快速重新启用的入口

7. **IDE 选择**
   - 当前：自动连接第一个检测到的 IDE
   - 建议：如果检测到多个 IDE，让用户选择
