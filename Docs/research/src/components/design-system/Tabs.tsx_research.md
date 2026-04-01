# Tabs.tsx 深度研究文档

## 场景与职责

Tabs 是 Claude Code 设计系统中功能最复杂的导航组件之一，用于在有限空间内组织多个相关内容面板。它提供了完整的标签页导航功能，包括键盘导航、受控/非受控模式、焦点管理、横幅支持等高级特性，是各种配置界面和复杂对话框的核心导航组件。

**核心职责：**
1. **内容组织**：将相关内容分组到可切换的标签页中
2. **键盘导航**：完整的键盘可访问性支持（左右箭头、Tab键）
3. **焦点管理**：区分标签头焦点和内容焦点，支持双向切换
4. **布局适配**：支持全宽模式、固定内容高度、模态集成
5. **上下文传递**：通过 Context 向子 Tab 传递选中状态和宽度信息

**典型使用场景：**
- 设置界面 (`src/components/Settings/Settings.tsx`, `src/components/Settings/Config.tsx`)
- 沙箱配置 (`src/components/sandbox/SandboxSettings.tsx`, `src/components/sandbox/SandboxOverridesTab.tsx`)
- 权限管理 (`src/components/permissions/rules/WorkspaceTab.tsx`, `src/components/permissions/rules/RecentDenialsTab.tsx`)
- 帮助系统 (`src/components/HelpV2/HelpV2.tsx`, `src/components/HelpV2/Commands.tsx`)
- 统计信息 (`src/components/Stats.tsx`)
- 插件管理 (`src/commands/plugin/PluginSettings.tsx`)

---

## 功能点目的

### 1. 标签页导航
- **目的**：在多个相关内容面板间切换
- **实现**：标签头（Tab Header）+ 内容区（Content Area）
- **视觉**：当前标签高亮显示，支持颜色主题

### 2. 键盘可访问性
- **目的**：支持无鼠标操作
- **按键映射**：
  - `←/→` 或 `Tab/Shift+Tab`：切换标签
  - `↓`：从标签头进入内容区
- **焦点状态**：标签头焦点和内容焦点分离

### 3. 受控与非受控模式
- **非受控模式**：使用 `defaultTab` 指定初始标签，内部管理状态
- **受控模式**：使用 `selectedTab` + `onTabChange` 完全外部控制
- **价值**：适应不同复杂度的使用场景

### 4. 焦点管理高级特性
- **initialHeaderFocused**：控制初始焦点位置
- **navFromContent**：允许从内容区使用 Tab/←/→ 切换标签
- **optIn 机制**：子组件可注册参与焦点管理

### 5. 布局变体
- **useFullWidth**：标签头占满终端宽度
- **contentHeight**：固定内容区高度，防止切换时布局变化
- **hidden**：隐藏标签头（用于特殊场景）

### 6. 模态集成
- **目的**：在 FullscreenLayout 模态中正确工作
- **实现**：检测 `modalScrollRef`，使用 ScrollBox 包裹内容
- **行为**：切换标签时自动重置滚动位置

---

## 具体技术实现

### 关键流程

```
Props 解析 → 标签解析 → 状态初始化 → 键盘绑定 → 渲染
```

**核心逻辑详解：**

1. **标签解析**
   ```typescript
   const tabs = children.map(child => [
     child.props.id ?? child.props.title,  // tab id
     child.props.title                      // display title
   ]);
   ```

2. **状态管理**
   ```typescript
   // 非受控模式
   const [internalSelectedTab, setInternalSelectedTab] = useState(
     defaultTabIndex !== -1 ? defaultTabIndex : 0
   );
   
   // 受控模式
   const isControlled = controlledSelectedTab !== undefined;
   const selectedTabIndex = isControlled 
     ? (controlledTabIndex !== -1 ? controlledTabIndex : 0)
     : internalSelectedTab;
   ```

3. **标签切换处理**
   ```typescript
   const handleTabChange = (offset: number) => {
     const newIndex = (selectedTabIndex + tabs.length + offset) % tabs.length;
     const newTabId = tabs[newIndex]?.[0];
     
     if (isControlled && onTabChange && newTabId) {
       onTabChange(newTabId);
     } else {
       setInternalSelectedTab(newIndex);
     }
     setHeaderFocused(true);
   };
   ```

4. **键盘绑定**
   ```typescript
   // 标签头焦点时的导航
   useKeybindings({
     "tabs:next": () => handleTabChange(1),
     "tabs:previous": () => handleTabChange(-1)
   }, { context: "Tabs", isActive: !hidden && !disableNavigation && headerFocused });
   
   // 内容区焦点时的导航（opt-in）
   useKeybindings({
     "tabs:next": () => { handleTabChange(1); setHeaderFocused(true); },
     "tabs:previous": () => { handleTabChange(-1); setHeaderFocused(true); }
   }, { context: "Tabs", isActive: navFromContent && !headerFocused && optedIn && !hidden && !disableNavigation });
   ```

5. **焦点管理**
   ```typescript
   // 从标签头进入内容区
   const handleKeyDown = (e: KeyboardEvent) => {
     if (!headerFocused || !optedIn || hidden) return;
     if (e.key === "down") {
       e.preventDefault();
       setHeaderFocused(false);
     }
   };
   ```

### 数据结构

**TabsProps 接口：**
```typescript
type TabsProps = {
  children: Array<React.ReactElement<TabProps>>;  // Tab 子组件
  title?: string;                                  // 标签组标题
  color?: keyof Theme;                             // 主题颜色
  defaultTab?: string;                             // 默认选中标签（非受控）
  hidden?: boolean;                                // 隐藏标签头
  useFullWidth?: boolean;                          // 占满宽度
  selectedTab?: string;                            // 受控模式当前标签
  onTabChange?: (tabId: string) => void;           // 受控模式回调
  banner?: React.ReactNode;                        // 横幅内容
  disableNavigation?: boolean;                     // 禁用键盘导航
  initialHeaderFocused?: boolean;                  // 初始焦点在标签头
  contentHeight?: number;                          // 固定内容高度
  navFromContent?: boolean;                        // 允许从内容区导航
}
```

**TabProps 接口：**
```typescript
type TabProps = {
  title: string;        // 标签标题（必填）
  id?: string;          // 标签标识（可选，默认使用 title）
  children: React.ReactNode;  // 标签内容
}
```

**TabsContextValue：**
```typescript
type TabsContextValue = {
  selectedTab: string | undefined;  // 当前选中标签 ID
  width: number | undefined;        // 内容区宽度
  headerFocused: boolean;           // 标签头是否有焦点
  focusHeader: () => void;          // 聚焦标签头
  blurHeader: () => void;           // 移除标签头焦点
  registerOptIn: () => () => void;  // 注册焦点参与
}
```

### 布局结构

```
TabsContext.Provider
└── Box (column, tabIndex=0, autoFocus)
    ├── Box (row, gap=1)           // 标签头行
    │   ├── Text (bold, color)     // 标题（可选）
    │   ├── Text[]                 // 标签按钮
    │   │   ├── backgroundColor    // 当前标签高亮
    │   │   ├── color              // 文字颜色
    │   │   └── inverse            // 非彩色终端的反色
    │   └── Text                   // 填充空格（useFullWidth）
    ├── banner                     // 横幅（可选）
    └── Box/ScrollBox              // 内容区
        └── children               // Tab 组件
```

### 标签渲染逻辑

```typescript
tabs.map(([id, title], i) => {
  const isCurrent = selectedTabIndex === i;
  const hasColorCursor = color && isCurrent && headerFocused;
  
  return (
    <Text 
      key={id}
      backgroundColor={hasColorCursor ? color : undefined}
      color={hasColorCursor ? "inverseText" : undefined}
      inverse={isCurrent && !hasColorCursor}
      bold={isCurrent}
    >
      {" "}{title}{" "}
    </Text>
  );
});
```

---

## 关键代码路径与文件引用

### 当前文件
- **路径**：`src/components/design-system/Tabs.tsx`
- **大小**：约 41KB（含 source map）
- **代码行数**：约 340 行

### 核心代码段

**Context 定义：**
```javascript
const TabsContext = createContext<TabsContextValue>({
  selectedTab: undefined,
  width: undefined,
  headerFocused: false,
  focusHeader: () => {},
  blurHeader: () => {},
  registerOptIn: () => () => {}
});
```

**Tab 组件：**
```javascript
export function Tab(t0) {
  const $ = _c(4);
  const { title, id, children } = t0;
  const { selectedTab, width } = useContext(TabsContext);
  const insideModal = useIsInsideModal();
  
  // 非当前标签不渲染
  if (selectedTab !== (id ?? title)) {
    return null;
  }
  
  const flexShrink = insideModal ? 0 : undefined;
  // ... 渲染逻辑
  return <Box width={width} flexShrink={flexShrink}>{children}</Box>;
}
```

**useTabHeaderFocus Hook：**
```javascript
export function useTabHeaderFocus() {
  const $ = _c(6);
  const { headerFocused, focusHeader, blurHeader, registerOptIn } = useContext(TabsContext);
  
  // 注册 opt-in
  useEffect(registerOptIn, [registerOptIn]);
  
  return { headerFocused, focusHeader, blurHeader };
}
```

### 调用方文件

| 文件路径 | 使用场景 | 特性使用 |
|---------|---------|---------|
| `src/components/Settings/Settings.tsx` | 设置主界面 | 多标签配置 |
| `src/components/Settings/Config.tsx` | 配置详情 | 受控/非受控 |
| `src/components/sandbox/SandboxSettings.tsx` | 沙箱设置 | 主题颜色 |
| `src/components/sandbox/SandboxOverridesTab.tsx` | 覆盖设置 | 嵌套标签 |
| `src/components/permissions/rules/WorkspaceTab.tsx` | 工作区权限 | 内容高度 |
| `src/components/permissions/rules/RecentDenialsTab.tsx` | 拒绝记录 | 宽度计算 |
| `src/components/HelpV2/HelpV2.tsx` | 帮助系统 | 全宽模式 |
| `src/components/HelpV2/Commands.tsx` | 命令帮助 | 键盘导航 |
| `src/components/Stats.tsx` | 统计信息 | 横幅支持 |
| `src/commands/plugin/PluginSettings.tsx` | 插件设置 | 模态集成 |
| `src/screens/REPL.tsx` | 主界面 | 标签切换 |
| `src/components/TagTabs.tsx` | 标签页 | 自定义实现 |
| `src/components/LogSelector.tsx` | 日志选择 | 宽度传递 |

### 依赖文件

**1. useKeybindings**
- **路径**：`src/keybindings/useKeybinding.ts`
- **功能**：键盘事件绑定
- **动作**：`tabs:next`, `tabs:previous`

**2. useTerminalSize**
- **路径**：`src/hooks/useTerminalSize.ts`
- **功能**：获取终端尺寸
- **用途**：计算全宽模式下的填充空格

**3. ScrollBox**
- **路径**：`src/ink/components/ScrollBox.tsx`
- **功能**：可滚动容器
- **用途**：模态模式下包裹内容

**4. modalContext**
- **路径**：`src/context/modalContext.tsx`
- **功能**：模态上下文检测
- **用途**：检测是否在模态中，调整渲染

**5. stringWidth**
- **路径**：`src/ink/stringWidth.ts`
- **功能**：计算字符串显示宽度
- **用途**：计算标签宽度，全宽布局

---

## 依赖与外部交互

### 直接依赖

```typescript
import React, { createContext, useCallback, useContext, useEffect, useState } from 'react';
import { useIsInsideModal, useModalScrollRef } from '../../context/modalContext.js';
import { useTerminalSize } from '../../hooks/useTerminalSize.js';
import ScrollBox from '../../ink/components/ScrollBox.js';
import type { KeyboardEvent } from '../../ink/events/keyboard-event.js';
import { stringWidth } from '../../ink/stringWidth.js';
import { Box, Text } from '../../ink.js';
import { useKeybindings } from '../../keybindings/useKeybinding.js';
import type { Theme } from '../../utils/theme.js';
```

### 键盘绑定

**schema.ts 定义：**
```typescript
export const KEYBINDING_ACTIONS = [
  // Tabs 导航
  'tabs:next',
  'tabs:previous',
  // ...
] as const;

export const KEYBINDING_CONTEXTS = [
  'Tabs',
  // ...
] as const;
```

**默认绑定（defaultBindings.ts）：**
```typescript
{
  context: 'Tabs',
  bindings: {
    'tab': 'tabs:next',
    'shift+tab': 'tabs:previous',
    'right': 'tabs:next',
    'left': 'tabs:previous',
  }
}
```

---

## 风险、边界与改进建议

### 潜在风险

1. **复杂度风险**
   - 组件功能丰富导致代码复杂（340+ 行）
   - 受控/非受控逻辑交织，容易出错
   - 焦点管理逻辑复杂，可能有边缘情况

2. **性能风险**
   - 每次切换标签重新渲染所有 Tab 子组件
   - `stringWidth` 计算在大量标签时可能有性能影响
   - ScrollBox 重新挂载（key={selectedTabIndex}）可能丢失状态

3. **键盘冲突**
   - `navFromContent` 可能与子组件的键盘处理冲突
   - Tab 键在表单中的标准行为被覆盖

4. **宽度计算**
   - `useFullWidth` 依赖终端宽度，resize 时可能闪烁
   - 中文字符宽度计算可能有误差

### 边界情况

| 场景 | 行为 | 建议 |
|------|------|------|
| children 为空数组 | 渲染空标签头 | 应添加警告或空状态 |
| 重复的 id/title | 后出现的优先 | 应添加唯一性检查 |
| selectedTab 不存在 | 默认选中第一个 | 符合预期 |
| contentHeight 小于内容 | 内容被裁剪 | 调用方应确保足够高度 |
| 同时设置 defaultTab 和 selectedTab | 受控模式优先 | 文档应明确说明 |

### 改进建议

1. **代码拆分**
   ```typescript
   // 将复杂逻辑拆分为自定义 hooks
   function useTabState(props: TabsProps) { }
   function useTabKeyboard(props: TabsProps) { }
   function useTabFocus(props: TabsProps) { }
   ```

2. **性能优化**
   ```typescript
   // 使用 React.memo 缓存 Tab 组件
   export const Tab = React.memo(function Tab(props) {
     // ...
   });
   
   // 延迟渲染非活动标签
   const [renderedTabs, setRenderedTabs] = useState(new Set([selectedTab]));
   useEffect(() => {
     setRenderedTabs(prev => new Set([...prev, selectedTab]));
   }, [selectedTab]);
   ```

3. **添加标签懒加载**
   ```typescript
   type TabProps = {
     // ...
     lazy?: boolean;  // 首次激活时才渲染内容
   }
   ```

4. **支持拖拽排序**
   ```typescript
   type TabsProps = {
     // ...
     draggable?: boolean;
     onReorder?: (newOrder: string[]) => void;
   }
   ```

5. **添加标签关闭**
   ```typescript
   type TabProps = {
     // ...
     closable?: boolean;
     onClose?: () => void;
   }
   ```

6. **改进可访问性**
   ```typescript
   // 添加 ARIA 属性
   <Box role="tablist" aria-label={title}>
     <Text role="tab" aria-selected={isCurrent} aria-controls={panelId}>
   </Box>
   <Box role="tabpanel" id={panelId} aria-labelledby={tabId}>
   ```

7. **错误边界**
   ```typescript
   // 包装 Tab 内容，防止单个标签错误影响整体
   <ErrorBoundary fallback={<TabError />}>
     {children}
   </ErrorBoundary>
   ```

### 测试建议

1. **单元测试**
   - 受控/非受控模式切换
   - 键盘导航所有路径
   - 焦点管理状态转换
   - 宽度计算准确性

2. **集成测试**
   - 与 ScrollBox 的集成
   - 在模态中的行为
   - 与 useKeybindings 的协调

3. **可访问性测试**
   - 屏幕阅读器导航
   - 键盘-only 操作
   - 焦点可见性

4. **性能测试**
   - 大量标签的渲染性能
   - 快速切换的响应性
   - 内存占用分析
