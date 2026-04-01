# GenerateStep.tsx 研究文档

## 场景与职责

`GenerateStep.tsx` 是 Agent 创建向导中的一个关键步骤组件，提供 AI 辅助的 Agent 生成功能。用户可以通过自然语言描述他们想要的 Agent 功能，系统会调用 Claude API 自动生成完整的 Agent 配置（名称、描述、系统提示词）。

该组件在以下场景中使用：
- 用户在 MethodStep 选择 "Generate with Claude (recommended)" 后
- 用户希望通过 AI 快速生成 Agent 配置时
- 在 LocationStep 之后，TypeStep/PromptStep/DescriptionStep 之前（生成后会跳过这些步骤）

## 功能点目的

1. **AI 生成 Agent**: 根据用户描述自动生成 Agent 配置
2. **生成状态管理**: 管理生成过程中的加载状态、错误状态
3. **取消生成**: 支持在生成过程中取消操作
4. **外部编辑器支持**: 支持在外部编辑器中编辑生成提示
5. **导航控制**: 支持返回上一步、进入下一步
6. **错误处理**: 处理生成过程中的各种错误情况

## 具体技术实现

### 关键流程

1. **组件初始化**:
   ```typescript
   const { updateWizardData, goBack, goToStep, wizardData } = useWizard<AgentWizardData>();
   const [prompt, setPrompt] = useState(wizardData.generationPrompt || '');
   const [isGenerating, setIsGenerating] = useState(false);
   const [error, setError] = useState<string | null>(null);
   const [cursorOffset, setCursorOffset] = useState(prompt.length);
   const model = useMainLoopModel();
   const abortControllerRef = useRef<AbortController | null>(null);
   ```

2. **取消生成处理** (`handleCancelGeneration`):
   ```typescript
   const handleCancelGeneration = useCallback(() => {
     if (abortControllerRef.current) {
       abortControllerRef.current.abort();
       abortControllerRef.current = null;
       setIsGenerating(false);
       setError('Generation cancelled');
     }
   }, []);
   ```
   - 中止正在进行的 API 请求
   - 清理 abort controller
   - 更新状态显示取消信息

3. **键盘快捷键绑定**:
   - 生成中：Esc 取消生成
     ```typescript
     useKeybinding('confirm:no', handleCancelGeneration, {
       context: 'Settings',
       isActive: isGenerating
     });
     ```
   - 未生成：Esc 返回上一步
     ```typescript
     useKeybinding('confirm:no', handleGoBack, {
       context: 'Settings',
       isActive: !isGenerating
     });
     ```
   - Ctrl+G：在外部编辑器中编辑
     ```typescript
     useKeybinding('chat:externalEditor', handleExternalEditor, {
       context: 'Chat',
       isActive: !isGenerating
     });
     ```

4. **返回处理** (`handleGoBack`):
   ```typescript
   const handleGoBack = useCallback(() => {
     updateWizardData({
       generationPrompt: '',
       agentType: '',
       systemPrompt: '',
       whenToUse: '',
       generatedAgent: undefined,
       wasGenerated: false
     });
     setPrompt('');
     setError(null);
     goBack();
   }, [updateWizardData, goBack]);
   ```
   - 清空所有生成相关的向导数据
   - 重置本地状态
   - 返回上一步

5. **生成处理** (`handleGenerate`):
   ```typescript
   const handleGenerate = async (): Promise<void> => {
     const trimmedPrompt = prompt.trim();
     if (!trimmedPrompt) {
       setError('Please describe what the agent should do');
       return;
     }
     
     setError(null);
     setIsGenerating(true);
     updateWizardData({ generationPrompt: trimmedPrompt, isGenerating: true });
     
     const controller = createAbortController();
     abortControllerRef.current = controller;
     
     try {
       const generated = await generateAgent(trimmedPrompt, model, [], controller.signal);
       updateWizardData({
         agentType: generated.identifier,
         whenToUse: generated.whenToUse,
         systemPrompt: generated.systemPrompt,
         generatedAgent: generated,
         isGenerating: false,
         wasGenerated: true
       });
       goToStep(6); // 跳转到 ToolsStep
     } catch (err) {
       if (err instanceof APIUserAbortError) {
         // 用户取消，不显示错误
       } else if (err instanceof Error && !err.message.includes('No assistant message found')) {
         setError(err.message || 'Failed to generate agent');
       }
       updateWizardData({ isGenerating: false });
     } finally {
       setIsGenerating(false);
       abortControllerRef.current = null;
     }
   };
   ```

### 数据结构

**组件状态**:
```typescript
type State = {
  prompt: string;              // 生成提示文本
  isGenerating: boolean;       // 是否正在生成
  error: string | null;        // 错误信息
  cursorOffset: number;        // 光标位置
}
```

**向导数据字段**:
```typescript
type AgentWizardData = {
  generationPrompt?: string;   // 生成提示
  agentType?: string;          // 生成的 Agent 名称
  whenToUse?: string;          // 生成的使用描述
  systemPrompt?: string;       // 生成的系统提示词
  generatedAgent?: GeneratedAgent;  // 完整的生成结果
  wasGenerated?: boolean;      // 是否通过 AI 生成
  isGenerating?: boolean;      // 是否正在生成
}
```

**GeneratedAgent 类型** (来自 generateAgent.ts):
```typescript
type GeneratedAgent = {
  identifier: string;          // Agent 名称
  whenToUse: string;           // 使用场景描述
  systemPrompt: string;        // 系统提示词
}
```

### UI 状态

1. **生成中状态**:
   - 显示旋转加载指示器（Spinner）
   - 显示 "Generating agent from description..."
   - 底部显示 "Esc to cancel"

2. **输入状态**:
   - 显示文本输入框
   - 显示错误信息（如果有）
   - 底部显示快捷键提示（Enter 提交，Ctrl+G 外部编辑器，Esc 返回）

### 依赖与外部交互

**导入依赖**:
- `@anthropic-ai/sdk`: Claude API SDK（`APIUserAbortError`）
- `react`: React 核心库
- `useMainLoopModel`: 获取当前使用的模型
- `ink.js`: 终端 UI 库
- `useKeybinding`: 键盘快捷键绑定
- `createAbortController`: 创建中止控制器
- `editPromptInEditor`: 外部编辑器工具
- `ConfigurableShortcutHint`, `Byline`, `Spinner`: UI 组件
- `TextInput`: 文本输入组件
- `useWizard`: 向导上下文
- `WizardDialogLayout`: 向导布局
- `generateAgent`: Agent 生成服务

**外部交互**:
- 调用 `generateAgent` 与 Claude API 交互
- 通过 `useWizard` 与向导系统交互
- 通过 `editPromptInEditor` 与外部编辑器交互

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/agents/new-agent-creation/wizard-steps/GenerateStep.tsx`

### 直接依赖文件
- `/home/sansha/Github/claude-code-instructkr/src/components/agents/new-agent-creation/types.js` - AgentWizardData 类型
- `/home/sansha/Github/claude-code-instructkr/src/components/agents/generateAgent.ts` - Agent 生成逻辑
- `/home/sansha/Github/claude-code-instructkr/src/components/wizard/index.js` - useWizard 钩子
- `/home/sansha/Github/claude-code-instructkr/src/components/wizard/WizardDialogLayout.tsx` - 向导布局
- `/home/sansha/Github/claude-code-instructkr/src/components/TextInput.tsx` - 文本输入组件
- `/home/sansha/Github/claude-code-instructkr/src/components/Spinner.tsx` - 加载指示器
- `/home/sansha/Github/claude-code-instructkr/src/hooks/useMainLoopModel.ts` - 模型获取钩子
- `/home/sansha/Github/claude-code-instructkr/src/utils/abortController.ts` - 中止控制器工具
- `/home/sansha/Github/claude-code-instructkr/src/utils/promptEditor.ts` - 外部编辑器工具

### 调用方文件
- `/home/sansha/Github/claude-code-instructkr/src/components/agents/new-agent-creation/CreateAgentWizard.tsx` - 创建 Agent 向导主组件

### 生成服务
- `/home/sansha/Github/claude-code-instructkr/src/components/agents/generateAgent.ts`:
  - `generateAgent`: 调用 Claude API 生成 Agent 配置
  - 包含详细的系统提示词指导 AI 如何生成 Agent

## 风险、边界与改进建议

### 潜在风险

1. **API 调用失败风险**:
   - 网络问题、API 限流、服务不可用等都可能导致生成失败
   - 当前有基本的错误处理，但可能需要更详细的错误分类
   - 建议：添加重试机制、网络状态检测

2. **生成内容质量问题**:
   - AI 生成的 Agent 配置可能不符合用户期望
   - 用户可能需要多次尝试才能得到满意的结果
   - 建议：添加"重新生成"选项、预览功能

3. **长时间生成**:
   - 复杂描述的生成可能需要较长时间
   - 用户可能在等待期间失去耐心
   - 建议：添加生成进度指示、预计时间显示

4. **并发问题**:
   - 如果用户快速多次触发生成，可能产生竞态条件
   - 当前有 `isGenerating` 状态，但依赖 React 的批量更新
   - 建议：使用更严格的并发控制

5. **内存泄漏**:
   - `abortControllerRef` 需要在组件卸载时清理
   - 当前在 `finally` 中清理，但如果组件在请求期间卸载可能有问题
   - 建议：添加 `useEffect` 的清理函数

### 边界情况

1. **空提示处理**:
   - 验证逻辑检查 `!trimmedPrompt`
   - 显示错误 "Please describe what the agent should do"

2. **用户取消**:
   - 正确处理 `APIUserAbortError`
   - 不显示错误信息（用户主动取消不是错误）

3. **特定错误过滤**:
   - 过滤掉 "No assistant message found" 错误
   - 这可能是 API 的已知问题

4. **生成后跳转**:
   - 成功生成后跳转到步骤 6（ToolsStep）
   - 跳过了 TypeStep、PromptStep、DescriptionStep

5. **返回清理**:
   - 返回时清空所有生成相关数据
   - 确保状态一致性

### 改进建议

1. **添加重试机制**:
   ```typescript
   const [retryCount, setRetryCount] = useState(0);
   const MAX_RETRIES = 3;
   
   // 在 catch 块中
   if (retryCount < MAX_RETRIES && isRetryableError(err)) {
     setRetryCount(c => c + 1);
     handleGenerate();
     return;
   }
   ```

2. **添加生成预览**:
   - 在保存前显示生成的配置预览
   - 允许用户编辑或重新生成
   ```typescript
   const [showPreview, setShowPreview] = useState(false);
   // 显示 agentType, whenToUse, systemPrompt 预览
   ```

3. **添加重新生成选项**:
   ```typescript
   const handleRegenerate = useCallback(() => {
     setError(null);
     handleGenerate();
   }, [handleGenerate]);
   ```

4. **改进错误分类**:
   ```typescript
   function classifyError(err: unknown): 'network' | 'api' | 'cancelled' | 'unknown' {
     if (err instanceof APIUserAbortError) return 'cancelled';
     if (err instanceof Error && err.message.includes('network')) return 'network';
     // ... 更多分类
   }
   ```

5. **添加生成历史**:
   - 保存多次生成的结果
   - 允许用户选择最满意的版本

6. **优化加载体验**:
   - 添加生成进度条（如果 API 支持流式响应）
   - 显示预计剩余时间
   - 添加有趣的等待提示

7. **组件卸载清理**:
   ```typescript
   useEffect(() => {
     return () => {
       if (abortControllerRef.current) {
         abortControllerRef.current.abort();
       }
     };
   }, []);
   ```

8. **添加提示模板**:
   - 提供常用 Agent 类型的模板
   - 帮助用户写出更好的生成提示
   ```typescript
   const TEMPLATES = [
     { name: 'Code Reviewer', template: 'Create a code reviewer agent that...' },
     { name: 'Test Writer', template: 'Create a test writing agent that...' },
   ];
   ```

9. **提示质量检查**:
   - 在提交前检查提示质量
   - 如果提示太短或太模糊，给出建议
   ```typescript
   function checkPromptQuality(prompt: string): string | null {
     if (prompt.length < 20) return 'Please provide a more detailed description';
     // ... 更多检查
   }
   ```

10. **支持流式生成**:
    - 如果 API 支持，使用流式响应
    - 实时显示生成进度

11. **添加撤销生成**:
    - 允许用户撤销生成结果
    - 返回到手动输入模式

12. **代码结构优化**:
    - 将生成逻辑提取到自定义 hook
    ```typescript
    function useAgentGeneration() {
      // ... 生成逻辑
      return { generate, isGenerating, error, result };
    }
    ```

13. **测试覆盖**:
    - 测试生成成功流程
    - 测试取消流程
    - 测试各种错误情况
    - 测试边界条件（空提示、超长提示等）

14. **国际化支持**:
    - 错误信息和 UI 文本支持多语言
    - 生成提示也可以考虑支持多语言

15. **性能优化**:
    - 使用 `useMemo` 缓存不需要重复计算的值
    - 优化重新渲染性能
