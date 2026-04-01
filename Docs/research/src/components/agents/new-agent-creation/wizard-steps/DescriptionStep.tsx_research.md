# DescriptionStep.tsx 研究文档

## 场景与职责

`DescriptionStep.tsx` 是 Agent 创建向导中的一个步骤组件，负责收集 Agent 的使用场景描述（`whenToUse`）。这个描述告诉 Claude 在什么情况下应该调用这个 Agent，是 Agent 定义中关键的元数据字段。

该组件在以下场景中使用：
- 用户在创建 Agent 流程中，完成名称和系统提示词输入后
- 需要为 Agent 提供使用场景描述时
- 在 PromptStep 之后，ToolsStep 之前

## 功能点目的

1. **描述输入**: 提供文本输入界面让用户输入 Agent 的使用场景描述
2. **外部编辑器支持**: 支持通过快捷键在外部编辑器中编辑描述
3. **输入验证**: 验证描述不能为空
4. **导航控制**: 支持 Enter 确认、Esc 返回上一步
5. **光标管理**: 维护光标位置状态

## 具体技术实现

### 关键流程

1. **组件初始化**:
   ```typescript
   const { goNext, goBack, updateWizardData, wizardData } = useWizard();
   const [whenToUse, setWhenToUse] = useState(wizardData.whenToUse || "");
   const [cursorOffset, setCursorOffset] = useState(whenToUse.length);
   const [error, setError] = useState(null);
   ```
   - 从向导上下文获取导航和数据操作方法
   - 初始化状态：描述文本、光标位置、错误信息
   - 如果向导数据中已有描述，恢复之前输入的值

2. **键盘快捷键绑定**:
   - `confirm:no` (Esc): 返回上一步
     ```typescript
     useKeybinding("confirm:no", goBack, { context: "Settings" });
     ```
   - `chat:externalEditor` (Ctrl+G): 在外部编辑器中编辑
     ```typescript
     useKeybinding("chat:externalEditor", handleExternalEditor, { context: "Chat" });
     ```

3. **外部编辑器处理** (`handleExternalEditor`):
   ```typescript
   const handleExternalEditor = useCallback(async () => {
     const result = await editPromptInEditor(whenToUse);
     if (result.content !== null) {
       setWhenToUse(result.content);
       setCursorOffset(result.content.length);
     }
   }, [whenToUse]);
   ```
   - 调用 `editPromptInEditor` 打开系统默认编辑器
   - 如果编辑成功，更新描述文本和光标位置
   - 将光标移动到文本末尾

4. **提交处理** (`handleSubmit`):
   ```typescript
   const handleSubmit = useCallback((value: string) => {
     const trimmedValue = value.trim();
     if (!trimmedValue) {
       setError("Description is required");
       return;
     }
     setError(null);
     updateWizardData({ whenToUse: trimmedValue });
     goNext();
   }, [goNext, updateWizardData]);
   ```
   - 去除首尾空白字符
   - 验证不能为空
   - 更新向导数据并进入下一步

5. **UI 渲染**:
   - 使用 `WizardDialogLayout` 作为布局容器
   - 显示标题："Description (tell Claude when to use this agent)"
   - 显示提示文本："When should Claude use this agent?"
   - 使用 `TextInput` 组件处理文本输入
   - 显示错误信息（如果有）
   - 底部显示快捷键提示

### 数据结构

**组件状态**:
```typescript
type State = {
  whenToUse: string;        // 描述文本
  cursorOffset: number;     // 光标位置
  error: string | null;     // 错误信息
}
```

**向导数据字段**:
```typescript
type AgentWizardData = {
  whenToUse?: string;       // 使用场景描述
  // ... 其他字段
}
```

**TextInput 属性**:
```typescript
{
  value: string;                    // 当前值
  onChange: (value: string) => void; // 变更回调
  onSubmit: (value: string) => void; // 提交回调
  placeholder: string;              // 占位符文本
  columns: number;                  // 输入框宽度（80列）
  cursorOffset: number;             // 光标偏移量
  onChangeCursorOffset: (offset: number) => void;  // 光标变更回调
  focus: boolean;                   // 是否自动聚焦
  showCursor: boolean;              // 是否显示光标
}
```

### 依赖与外部交互

**导入依赖**:
- `react`: React 核心库（`useState`, `useCallback`）
- `ink.js`: 终端 UI 库（`Box`, `Text`）
- `useKeybinding`: 键盘快捷键绑定钩子
- `editPromptInEditor`: 外部编辑器工具
- `ConfigurableShortcutHint`: 可配置快捷键提示
- `Byline`, `KeyboardShortcutHint`: 设计系统组件
- `TextInput`: 文本输入组件
- `useWizard`: 向导上下文钩子
- `WizardDialogLayout`: 向导布局组件

**外部交互**:
- 通过 `useWizard` 与向导系统交互
- 通过 `editPromptInEditor` 与外部编辑器交互
- 通过 `TextInput` 处理用户输入

## 关键代码路径与文件引用

### 当前文件
- `/home/sansha/Github/claude-code-instructkr/src/components/agents/new-agent-creation/wizard-steps/DescriptionStep.tsx`

### 直接依赖文件
- `/home/sansha/Github/claude-code-instructkr/src/components/agents/new-agent-creation/types.js` - AgentWizardData 类型
- `/home/sansha/Github/claude-code-instructkr/src/components/wizard/index.js` - useWizard 钩子
- `/home/sansha/Github/claude-code-instructkr/src/components/wizard/WizardDialogLayout.tsx` - 向导布局
- `/home/sansha/Github/claude-code-instructkr/src/components/TextInput.tsx` - 文本输入组件
- `/home/sansha/Github/claude-code-instructkr/src/components/ConfigurableShortcutHint.tsx` - 快捷键提示
- `/home/sansha/Github/claude-code-instructkr/src/utils/promptEditor.ts` - 外部编辑器工具

### 调用方文件
- `/home/sansha/Github/claude-code-instructkr/src/components/agents/new-agent-creation/CreateAgentWizard.tsx` - 创建 Agent 向导主组件

### 向导步骤顺序
根据 `CreateAgentWizard.tsx`，步骤顺序为：
1. LocationStep - 选择存储位置
2. MethodStep - 选择创建方式
3. GenerateStep - AI 生成（可选）
4. TypeStep - 输入 Agent 名称
5. PromptStep - 输入系统提示词
6. **DescriptionStep** - 输入使用场景描述 ← 当前步骤
7. ToolsStep - 选择工具
8. ModelStep - 选择模型
9. ColorStep - 选择颜色
10. MemoryStep - 选择内存（可选）
11. ConfirmStepWrapper - 确认和保存

## 风险、边界与改进建议

### 潜在风险

1. **输入长度风险**:
   - 当前没有对描述长度进行限制
   - 过长的描述可能影响性能或存储
   - 建议：添加长度限制（如 5000 字符）

2. **特殊字符处理**:
   - 描述中可能包含特殊字符或控制字符
   - 可能影响 YAML frontmatter 的解析
   - 建议：添加字符过滤或转义处理

3. **光标位置同步问题**:
   - 光标状态由组件和 TextInput 共同管理
   - 如果两者不同步，可能导致光标位置异常
   - 建议：确保光标更新逻辑的一致性

4. **外部编辑器错误处理**:
   - `editPromptInEditor` 可能失败
   - 当前代码静默忽略错误（`result.content !== null`）
   - 建议：添加错误提示

### 边界情况

1. **空输入处理**:
   - 验证逻辑检查 `!trimmedValue`
   - 显示错误信息 "Description is required"
   - 阻止进入下一步

2. **空白字符处理**:
   - 使用 `trim()` 去除首尾空白
   - 纯空白输入会被视为空

3. **数据恢复**:
   - 从 `wizardData.whenToUse` 恢复之前输入的值
   - 支持向导步骤间的数据持久化

4. **光标位置初始化**:
   - 初始光标位置设为文本长度
   - 确保光标在文本末尾

### 改进建议

1. **添加长度限制**:
   ```typescript
   const MAX_DESCRIPTION_LENGTH = 5000;
   
   const handleSubmit = useCallback((value: string) => {
     const trimmedValue = value.trim();
     if (!trimmedValue) {
       setError("Description is required");
       return;
     }
     if (trimmedValue.length > MAX_DESCRIPTION_LENGTH) {
       setError(`Description must be less than ${MAX_DESCRIPTION_LENGTH} characters`);
       return;
     }
     // ... 后续逻辑
   }, []);
   ```

2. **添加字符计数器**:
   ```typescript
   <Text dimColor>{whenToUse.length}/5000</Text>
   ```

3. **改进外部编辑器错误处理**:
   ```typescript
   const handleExternalEditor = useCallback(async () => {
     const result = await editPromptInEditor(whenToUse);
     if (result.error) {
       setError(`Failed to open editor: ${result.error}`);
       return;
     }
     if (result.content !== null) {
       setWhenToUse(result.content);
       setCursorOffset(result.content.length);
     }
   }, [whenToUse]);
   ```

4. **添加描述模板/示例**:
   - 提供常用描述模板
   - 帮助用户理解应该写什么
   ```typescript
   const EXAMPLES = [
     "Use this agent when you need to review code for security vulnerabilities",
     "Use this agent when you want to generate unit tests for your code",
     "Use this agent when you need help with database schema design"
   ];
   ```

5. **支持多行输入**:
   - 当前 `TextInput` 可能支持多行
   - 明确启用多行模式以支持长描述
   ```typescript
   <TextInput multiline maxVisibleLines={10} ... />
   ```

6. **实时验证**:
   - 在用户输入时实时验证
   - 而不是仅在提交时验证
   ```typescript
   useEffect(() => {
     if (whenToUse.trim()) {
       setError(null);
     }
   }, [whenToUse]);
   ```

7. **添加撤销/重做支持**:
   - 支持 Ctrl+Z 撤销输入
   - 支持 Ctrl+Y 重做输入

8. **代码优化**:
   - 提取验证逻辑为独立函数
   - 便于测试和复用
   ```typescript
   function validateDescription(value: string): string | null {
     const trimmed = value.trim();
     if (!trimmed) return "Description is required";
     if (trimmed.length > MAX_LENGTH) return `Description too long`;
     return null;
   }
   ```

9. **可访问性改进**:
   - 添加输入框标签
   - 确保屏幕阅读器可以正确识别

10. **测试覆盖**:
    - 测试空输入验证
    - 测试长输入处理
    - 测试外部编辑器集成
    - 测试数据恢复逻辑

11. **国际化支持**:
    - 错误信息和 UI 文本支持多语言
    - 占位符文本也支持国际化
