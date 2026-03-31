# CreatingStep.tsx 深度研究文档

## 场景与职责

`CreatingStep.tsx` 是 Claude Code CLI 中 `install-github-app` 命令的状态展示组件，用于在 GitHub Actions 工作流创建过程中显示实时进度。该组件以步骤列表的形式展示当前执行状态，让用户了解安装进度和已完成的操作。

### 核心职责
1. **展示安装进度**：以步骤列表形式显示当前执行阶段
2. **动态步骤生成**：根据配置（是否跳过工作流、Secret 状态等）动态生成步骤列表
3. **状态可视化**：通过颜色和对勾符号区分已完成、进行中、待执行状态

---

## 功能点目的

### 1. 进度步骤展示
根据配置动态生成步骤列表：

**标准模式（skipWorkflow = false）**：
1. Getting repository information
2. Creating branch
3. Creating workflow file(s)
4. Setting up {secretName} secret / Using existing API key secret
5. Opening pull request page

**跳过工作流模式（skipWorkflow = true）**：
1. Getting repository information
2. Setting up {secretName} secret / Using existing API key secret

### 2. 状态可视化
- **已完成（completed）**：绿色 ✓ 前缀
- **进行中（in-progress）**：黄色，带 … 后缀
- **待执行（pending）**：默认颜色，无特殊标记

### 3. 智能文案适配
- 根据 `selectedWorkflows.length` 决定单复数："Creating workflow file" vs "Creating workflow files"
- 根据 `secretExists` 和 `useExistingSecret` 决定 Secret 相关文案

---

## 具体技术实现

### 关键数据结构

```typescript
interface CreatingStepProps {
  currentWorkflowInstallStep: number;   // 当前步骤索引（0-based）
  secretExists: boolean;                // 是否已存在 Secret
  useExistingSecret: boolean;           // 是否使用现有 Secret
  secretName: string;                   // Secret 名称
  skipWorkflow?: boolean;               // 是否跳过工作流创建
  selectedWorkflows: Workflow[];        // 选中的工作流列表
}

// Workflow 类型（来自 types.js）
type Workflow = 'claude' | 'claude-review';
```

### 关键流程

#### 1. 步骤列表生成
```typescript
const progressSteps = skipWorkflow
  ? [
      'Getting repository information',
      secretExists && useExistingSecret
        ? 'Using existing API key secret'
        : `Setting up ${secretName} secret`,
    ]
  : [
      'Getting repository information',
      'Creating branch',
      selectedWorkflows.length > 1
        ? 'Creating workflow files'
        : 'Creating workflow file',
      secretExists && useExistingSecret
        ? 'Using existing API key secret'
        : `Setting up ${secretName} secret`,
      'Opening pull request page',
    ];
```

#### 2. 状态计算逻辑
```typescript
progressSteps.map((stepText, index) => {
  let status = "pending";
  
  if (index < currentWorkflowInstallStep) {
    status = "completed";
  } else if (index === currentWorkflowInstallStep) {
    status = "in-progress";
  }
  
  return (
    <Box key={index}>
      <Text color={
        status === "completed" ? "success" : 
        status === "in-progress" ? "warning" : 
        undefined
      }>
        {status === "completed" ? "✓ " : ""}
        {stepText}
        {status === "in-progress" ? "…" : ""}
      </Text>
    </Box>
  );
});
```

### UI 渲染结构

```jsx
<>
  <Box flexDirection="column" borderStyle="round" paddingX={1}>
    {/* 标题 */}
    <Box flexDirection="column" marginBottom={1}>
      <Text bold>Install GitHub App</Text>
      <Text dimColor>Create GitHub Actions workflow</Text>
    </Box>
    
    {/* 步骤列表 */}
    {progressSteps.map((stepText, index) => {
      const status = 
        index < currentWorkflowInstallStep ? "completed" :
        index === currentWorkflowInstallStep ? "in-progress" : "pending";
      
      return (
        <Box key={index}>
          <Text color={
            status === "completed" ? "success" :
            status === "in-progress" ? "warning" : undefined
          }>
            {status === "completed" && "✓ "}
            {stepText}
            {status === "in-progress" && "…"}
          </Text>
        </Box>
      );
    })}
  </Box>
</>
```

---

## 关键代码路径与文件引用

### 内部依赖
| 文件路径 | 用途 |
|---------|------|
| `../../ink.js` | Ink 渲染库（Box, Text） |
| `./types.js` | Workflow 类型定义 |

### 外部调用方
| 文件路径 | 调用场景 |
|---------|---------|
| `install-github-app.tsx` | `state.step === 'creating'` 时渲染 |

### 状态流转与步骤更新
```
用户确认开始创建
  ↓
setState({ step: 'creating', currentWorkflowInstallStep: 0 })
  ↓
runSetupGitHubActions(
  repoName,
  apiKeyOrOAuthToken,
  secretName,
  updateProgress,  // ← 回调函数
  skipWorkflow,
  selectedWorkflows,
  authType
)
  ↓ (在 setupGitHubActions.ts 中)
// Step 0: 获取仓库信息
updateProgress() → setState(s => ({ currentWorkflowInstallStep: 1 }))
  ↓
// Step 1: 创建分支（如果不跳过）
updateProgress() → setState(s => ({ currentWorkflowInstallStep: 2 }))
  ↓
// Step 2: 创建工作流文件（如果不跳过）
updateProgress() → setState(s => ({ currentWorkflowInstallStep: 3 }))
  ↓
// Step 3: 设置 Secret
updateProgress() → setState(s => ({ currentWorkflowInstallStep: 4 }))
  ↓
// Step 4: 打开 PR 页面（如果不跳过）
updateProgress() → setState(s => ({ currentWorkflowInstallStep: 5 }))
```

---

## 依赖与外部交互

### React 依赖
- React Compiler: 使用 `_c` 缓存优化
- 无状态 hooks（纯展示组件）

### Ink 生态
- **Box**: 布局容器，带边框样式
- **Text**: 文本渲染，支持颜色：
  - `success`: 已完成步骤（绿色）
  - `warning`: 进行中步骤（黄色）
  - 默认：待执行步骤

### 父组件交互
该组件为纯展示型，仅接收 props：
- `currentWorkflowInstallStep`: 控制高亮显示的步骤
- 其他 props 用于动态生成步骤文案

### setupGitHubActions.ts 交互
| 步骤 | 操作 | updateProgress 调用时机 |
|-----|------|------------------------|
| 0 | 检查仓库存在 | 初始状态 |
| 1 | 获取默认分支 | 完成仓库检查后 |
| 2 | 创建分支 | 获取 SHA 后（如果不跳过） |
| 3 | 创建工作流文件 | 创建分支后（如果不跳过） |
| 4 | 设置 Secret | 工作流创建后（或分支创建后如果跳过工作流） |
| 5 | 打开 PR 页面 | Secret 设置后（如果不跳过） |

---

## 风险、边界与改进建议

### 潜在风险

1. **步骤索引越界**
   - `currentWorkflowInstallStep` 可能超过 `progressSteps.length`
   - 当前代码无越界保护，可能导致无步骤高亮
   - 建议：添加边界检查

2. **步骤数量不一致**
   - `skipWorkflow` 和 `secretExists/useExistingSecret` 组合影响步骤数量
   - 父组件需要确保 `updateProgress()` 调用次数与步骤数匹配
   - 风险：调用次数不匹配导致进度显示异常

3. **Workflow 类型扩展**
   - 当前仅支持 'claude' 和 'claude-review'
   - 新增 Workflow 类型需要同步更新步骤生成逻辑

### 边界情况

1. **所有步骤已完成**
   - `currentWorkflowInstallStep >= progressSteps.length`
   - 所有步骤显示为完成状态，无"进行中"步骤

2. **跳过工作流且使用现有 Secret**
   - 最少步骤场景：仅 2 步
   - "Getting repository information" → "Using existing API key secret"

3. **多工作流选择**
   - `selectedWorkflows = ['claude', 'claude-review']`
   - 文案自动切换为复数形式 "Creating workflow files"

4. **Secret 名称长度**
   - 极长的 `secretName` 可能导致步骤文案换行
   - 影响视觉美观但不影响功能

### 改进建议

1. **添加步骤完成百分比**
   ```typescript
   const progressPercent = Math.min(
     100,
     Math.round((currentWorkflowInstallStep / progressSteps.length) * 100)
   );
   
   <Text dimColor>{progressPercent}% complete</Text>
   ```

2. **添加预计时间提示**
   ```typescript
   // 基于历史数据或固定估算
   const estimatedSeconds = progressSteps.length * 3;
   <Text dimColor>Estimated time: ~{estimatedSeconds} seconds</Text>
   ```

3. **错误状态显示**
   ```typescript
   interface CreatingStepProps {
     errorStep?: number;  // 发生错误的步骤索引
     errorMessage?: string;
   }
   
   // 在步骤旁边显示错误图标
   {index === errorStep && <Text color="error">✗ {errorMessage}</Text>}
   ```

4. **动画效果**
   ```typescript
   import { Spinner } from '../../components/Spinner.js';
   
   {status === "in-progress" && (
     <Box>
       <Spinner />
       <Text color="warning">{stepText}…</Text>
     </Box>
   )}
   ```

5. **详细日志展开**
   ```typescript
   const [showDetails, setShowDetails] = useState(false);
   
   // 允许用户查看详细日志
   <Text dimColor onPress={() => setShowDetails(!showDetails)}>
     Press 'l' to view logs
   </Text>
   ```

6. **步骤计数保护**
   ```typescript
   // 确保 currentWorkflowInstallStep 在有效范围内
   const safeStepIndex = Math.min(
     currentWorkflowInstallStep,
     progressSteps.length - 1
   );
   ```

---

## 总结

`CreatingStep.tsx` 是一个简洁高效的进度展示组件，通过动态步骤生成和状态颜色编码，为用户提供了清晰的安装进度反馈。组件设计考虑了多种配置组合（跳过工作流、使用现有 Secret 等），确保在各种场景下都能正确显示。

主要关注点在于父组件 `setupGitHubActions.ts` 中的步骤更新调用必须与组件的步骤生成逻辑保持同步。任何步骤数量或顺序的变更都需要在两个文件中同步更新，这是维护时需要特别注意的地方。
