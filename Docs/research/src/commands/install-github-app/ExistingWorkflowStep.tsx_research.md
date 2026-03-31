# ExistingWorkflowStep.tsx 深度研究文档

## 场景与职责

`ExistingWorkflowStep.tsx` 是 Claude Code CLI 中 `install-github-app` 命令的决策组件，用于处理当检测到目标仓库已存在 Claude 工作流文件（`.github/workflows/claude.yml`）时的用户交互。该组件提供三种处理策略，让用户决定如何处理冲突。

### 核心职责
1. **检测冲突提示**：告知用户工作流文件已存在
2. **提供处理选项**：三种策略供用户选择
3. **引导查看模板**：提供最新工作流模板的链接
4. **支持取消操作**：允许用户退出而不做任何更改

---

## 功能点目的

### 1. 冲突处理选项
当检测到现有工作流时，提供三种处理方式：

| 选项 | 值 | 说明 |
|-----|---|------|
| Update workflow file with latest version | `update` | 更新现有工作流到最新版本 |
| Skip workflow update (configure secrets only) | `skip` | 仅配置 Secret，保留现有工作流 |
| Exit without making changes | `exit` | 取消操作，不做任何更改 |

### 2. 模板参考链接
提供指向最新工作流模板的链接，方便用户对比：
```
https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml
```

---

## 具体技术实现

### 关键数据结构

```typescript
interface ExistingWorkflowStepProps {
  repoName: string;                     // 仓库名称（owner/repo 格式）
  onSelectAction: (action: 'update' | 'skip' | 'exit') => void;  // 选择回调
}
```

### 关键流程

#### 1. 选项定义
```typescript
const options = [
  {
    label: "Update workflow file with latest version",
    value: "update"
  },
  {
    label: "Skip workflow update (configure secrets only)",
    value: "skip"
  },
  {
    label: "Exit without making changes",
    value: "exit"
  }
];
```

#### 2. 选择处理
```typescript
const handleSelect = (value: string) => {
  onSelectAction(value as 'update' | 'skip' | 'exit');
};

const handleCancel = () => {
  onSelectAction("exit");  // 取消默认执行 exit
};
```

### UI 渲染结构

```jsx
<Box flexDirection="column" borderStyle="round" borderDimColor={true} paddingX={1}>
  {/* 标题和仓库信息 */}
  <Box flexDirection="column" marginBottom={1}>
    <Text bold>Existing Workflow Found</Text>
    <Text dimColor>Repository: {repoName}</Text>
  </Box>
  
  {/* 冲突说明 */}
  <Box flexDirection="column" marginBottom={1}>
    <Text>
      A Claude workflow file already exists at{" "}
      <Text color="claude">.github/workflows/claude.yml</Text>
    </Text>
    <Text dimColor>What would you like to do?</Text>
  </Box>
  
  {/* 选项选择器 */}
  <Box flexDirection="column">
    <Select 
      options={options} 
      onChange={handleSelect} 
      onCancel={handleCancel} 
    />
  </Box>
  
  {/* 模板链接 */}
  <Box marginTop={1}>
    <Text dimColor>
      View the latest workflow template at:{" "}
      <Text color="claude">
        https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml
      </Text>
    </Text>
  </Box>
</Box>
```

---

## 关键代码路径与文件引用

### 内部依赖
| 文件路径 | 用途 |
|---------|------|
| `src/components/CustomSelect/index.js` | Select 选择器组件 |
| `../../ink.js` | Ink 渲染库（Box, Text） |

### 外部调用方
| 文件路径 | 调用场景 |
|---------|---------|
| `install-github-app.tsx` | `state.step === 'check-existing-workflow'` 时渲染 |

### 状态流转
```
用户完成 GitHub App 安装确认
  ↓
检测到 workflowExists = true
  ↓
setState({ step: 'check-existing-workflow' })
  ↓
<ExistingWorkflowStep repoName={state.selectedRepoName} onSelectAction={handleWorkflowAction} />
  ↓ (用户选择)
handleWorkflowAction(action)
  ↓
action === 'exit' → props.onDone('Installation cancelled by user')
action === 'skip' → 进入 api-key 步骤
action === 'update' → 进入 api-key 步骤（后续会覆盖现有工作流）
```

### 父组件处理逻辑
```typescript
const handleWorkflowAction = async (action: 'update' | 'skip' | 'exit') => {
  if (action === 'exit') {
    props.onDone('Installation cancelled by user');
    return;
  }
  
  logEvent('tengu_install_github_app_step_completed', {
    step: 'check-existing-workflow'
  });
  
  setState(prev => ({ ...prev, workflowAction: action }));
  
  if (action === 'skip' || action === 'update') {
    // 继续到 API Key 步骤
    if (existingApiKey) {
      await checkExistingSecret();
    } else {
      setState(prev => ({ ...prev, step: 'api-key' }));
    }
  }
};
```

---

## 依赖与外部交互

### React 依赖
- React Compiler: 使用 `_c` 缓存优化
- 无本地状态（纯展示 + 选择组件）

### Ink 生态
- **Box**: 布局容器，带边框和 `borderDimColor` 属性
- **Text**: 文本渲染，支持：
  - `bold`: 标题加粗
  - `dimColor`: 次要信息
  - `color="claude"`: 链接高亮

### CustomSelect 组件
- **Select**: 自定义选择器组件
  - `options`: 选项数组（label, value）
  - `onChange`: 选择变更回调
  - `onCancel`: 取消回调（通常绑定到 Escape 键）

### 父组件交互
| 回调 | 触发条件 | 用途 |
|-----|---------|------|
| `onSelectAction('update')` | 选择"更新工作流" | 覆盖现有工作流文件 |
| `onSelectAction('skip')` | 选择"跳过工作流" | 仅配置 Secret |
| `onSelectAction('exit')` | 选择"退出"或按 Escape | 取消安装 |

---

## 风险、边界与改进建议

### 潜在风险

1. **工作流差异对比缺失**
   - 组件仅告知工作流已存在，不展示差异对比
   - 用户无法判断是否需要更新
   - 风险：用户可能盲目选择 update 导致配置丢失

2. **Select 组件依赖**
   - 依赖外部 `CustomSelect` 组件
   - 如果该组件行为变更，可能影响用户体验

3. **硬编码文件路径**
   - `.github/workflows/claude.yml` 硬编码在组件中
   - 如果工作流路径变更，需要同步修改多处

### 边界情况

1. **仓库名称格式**
   - `repoName` 期望 `owner/repo` 格式
   - 如果传入其他格式，显示可能不美观

2. **Cancel 操作**
   - 按 Escape 键触发 `handleCancel`
   - 等同于选择 "Exit without making changes"

3. **多工作流文件**
   - 当前仅检测 `claude.yml`
   - 如果存在 `claude-code-review.yml` 等其他工作流，不提示

4. **网络链接可访问性**
   - 模板链接以纯文本显示
   - 终端用户无法直接点击

### 改进建议

1. **添加差异预览**
   ```typescript
   // 获取现有工作流内容和最新模板对比
   interface ExistingWorkflowStepProps {
     repoName: string;
     currentWorkflowContent?: string;
     latestWorkflowContent?: string;
     onSelectAction: (action: 'update' | 'skip' | 'exit') => void;
   }
   
   // 提供 'v' 键查看差异
   <Text dimColor>Press 'v' to view workflow differences</Text>
   ```

2. **工作流版本信息**
   ```typescript
   // 显示现有工作流的版本或创建时间
   <Text dimColor>Existing workflow: v1.2.3 (created 2024-01-15)</Text>
   <Text dimColor>Latest version: v2.0.0</Text>
   ```

3. **备份提示**
   ```typescript
   // 选择 update 前提示备份
   {selectedAction === 'update' && (
     <Text color="warning">
       ⚠️  This will overwrite your existing workflow. 
       Consider backing up your current file first.
     </Text>
   )}
   ```

4. **链接可交互**
   ```typescript
   import { Link } from '../../ink.js';
   
   <Link url="https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml">
     <Text color="claude" underline>View latest template</Text>
   </Link>
   ```

5. **工作流路径配置化**
   ```typescript
   // 从常量导入而非硬编码
   import { WORKFLOW_FILE_PATH } from '../../constants/github-app.js';
   
   <Text color="claude">{WORKFLOW_FILE_PATH}</Text>
   ```

6. **批量工作流检测**
   ```typescript
   // 检测多个工作流文件
   interface ExistingWorkflowStepProps {
     existingWorkflows: string[];  // ['claude.yml', 'claude-code-review.yml']
     // ...
   }
   ```

7. **智能推荐**
   ```typescript
   // 基于工作流内容分析推荐操作
   const getRecommendedAction = (currentContent: string): 'update' | 'skip' => {
     const currentVersion = extractVersion(currentContent);
     const latestVersion = getLatestVersion();
     return currentVersion < latestVersion ? 'update' : 'skip';
   };
   ```

---

## 总结

`ExistingWorkflowStep.tsx` 是一个关键的分支决策组件，在安装流程中处理工作流冲突场景。通过提供更新、跳过、退出三种策略，给予用户充分的控制权。组件设计简洁，主要依赖 `CustomSelect` 组件提供选择交互。

主要关注点在于帮助用户做出明智的决策，当前实现通过提供模板链接来辅助决策，但仍有改进空间（如差异对比、版本信息等）。该组件在整体安装流程中起到了保护用户现有配置不被意外覆盖的重要作用。
