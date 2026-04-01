# cyberRiskInstruction.ts 深度研究文档

## 场景与职责

`cyberRiskInstruction.ts` 是 Claude Code CLI 中定义网络安全风险指令的核心安全文件。它明确了 Claude 在处理安全相关请求时的行为边界，区分可接受的防御性安全协助与潜在有害活动。

### 核心使用场景
1. **系统提示词注入**：作为系统提示词的一部分，指导 AI 模型处理安全相关查询
2. **渗透测试边界定义**：明确 Claude 可以协助的授权安全测试范围
3. **CTF 竞赛支持**：为教育性安全挑战提供适当的协助级别
4. **双重用途工具管控**：管理可能被滥用的安全工具（如 C2 框架、凭证测试工具）的使用

### 重要性声明

文件头部包含重要警告：
> **IMPORTANT: DO NOT MODIFY THIS INSTRUCTION WITHOUT SAFEGUARDS TEAM REVIEW**

此指令由 Safeguards 团队拥有，经过精心设计和评估，平衡了安全效用与安全性。未经授权的修改可能影响：
- Claude 如何处理渗透测试和 CTF 请求
- Claude 将协助哪些安全工具和技术
- 防御性与攻击性安全协助之间的边界

---

## 功能点目的

### 指令内容

```
IMPORTANT: Assist with authorized security testing, defensive security, 
CTF challenges, and educational contexts. Refuse requests for destructive 
techniques, DoS attacks, mass targeting, supply chain compromise, or 
detection evasion for malicious purposes. Dual-use security tools 
(C2 frameworks, credential testing, exploit development) require clear 
authorization context: pentesting engagements, CTF competitions, security 
research, or defensive use cases.
```

### 核心原则分解

| 类别 | 允许 | 拒绝 |
|------|------|------|
| **活动类型** | 授权安全测试、防御性安全、CTF 挑战、教育场景 | 破坏性技术、DoS 攻击、大规模目标、供应链破坏、恶意检测规避 |
| **工具使用** | 有明确授权的双重用途工具 | 无上下文或恶意目的的双重用途工具 |
| **授权上下文** | 渗透测试约定、CTF 竞赛、安全研究、防御用例 | 缺乏明确授权说明的请求 |

### 双重用途工具管理

特别指出的需要明确授权的工具类型：
- **C2 框架**（Command and Control）
- **凭证测试工具**
- **漏洞利用开发**

---

## 具体技术实现

### 数据结构

```typescript
// 简单的字符串常量导出
export const CYBER_RISK_INSTRUCTION = `IMPORTANT: Assist with...`
```

### 关键代码路径

#### 1. 系统提示词注入路径

```
构造系统提示词
    ↓
src/constants/prompts.ts
    ↓
导入 CYBER_RISK_INSTRUCTION
    ↓
注入到 getSimpleIntroSection() 中
    ↓
成为系统提示词的核心安全指导
```

**关键代码位置**（`src/constants/prompts.ts`）：
```typescript
function getSimpleIntroSection(outputStyleConfig: OutputStyleConfig | null): string {
  return `
You are an interactive agent that helps users ${outputStyleConfig !== null ? '...' : '...'}

${CYBER_RISK_INSTRUCTION}
IMPORTANT: You must NEVER generate or guess URLs...`
}
```

**关键文件引用**：
- `src/constants/prompts.ts`: 唯一导入和使用此指令的文件

---

## 依赖与外部交互

### 内部依赖

**零依赖**：此文件不导入任何其他模块，仅导出一个字符串常量。

### 被依赖方

| 文件 | 使用的常量 | 用途 |
|------|-----------|------|
| `src/constants/prompts.ts` | `CYBER_RISK_INSTRUCTION` | 系统提示词安全指导 |

### 外部团队交互

| 团队 | 角色 | 联系方式 |
|------|------|---------|
| Safeguards 团队 | 指令所有者 | David Forsythe, Kyla Guru |

---

## 风险、边界与改进建议

### 当前风险

1. **修改审批风险**
   - 文件注释明确要求 Safeguards 团队审批
   - 未经授权的修改可能导致安全策略漏洞

2. **指令注入位置单一**
   - 仅在 `getSimpleIntroSection` 中使用
   - 如果系统提示词有其他构造路径，可能遗漏此指令

3. **上下文理解依赖模型**
   - 指令的效果完全依赖 AI 模型的理解能力
   - 复杂场景下的边界判断可能存在模糊性

4. **本地化缺失**
   - 指令仅提供英文版本
   - 非英语用户可能无法完全理解安全边界

### 边界情况

| 场景 | 预期行为 |
|------|---------|
| 用户请求"测试我的网站安全性" | 允许，属于授权安全测试 |
| 用户请求"帮我黑掉某网站" | 拒绝，缺乏明确授权 |
| CTF 竞赛题目 | 允许，属于教育场景 |
| 请求编写勒索软件 | 拒绝，破坏性技术 |
| 请求解释 SQL 注入原理 | 允许，教育场景 |
| 请求自动化 SQL 注入攻击 | 拒绝，除非有渗透测试授权 |

### 改进建议

1. **多语言支持**
   ```typescript
   // 建议添加本地化支持
   export const CYBER_RISK_INSTRUCTION_I18N = {
     en: `IMPORTANT: Assist with...`,
     zh: `重要：协助授权的安全测试...`,
     // ...
   }
   ```

2. **版本控制增强**
   ```typescript
   // 建议添加版本信息
   export const CYBER_RISK_INSTRUCTION = {
     version: '2024-01-15',
     content: `IMPORTANT: Assist with...`,
     approvedBy: 'Safeguards Team',
     reviewDate: '2024-06-15'
   }
   ```

3. **使用位置审计**
   ```typescript
   // 建议添加使用追踪
   export const CYBER_RISK_INSTRUCTION = new Proxy(`IMPORTANT: Assist with...`, {
     get(target, prop) {
       if (prop === 'toString' || prop === 'valueOf') {
         console.trace('CYBER_RISK_INSTRUCTION accessed')
       }
       return target[prop]
     }
   })
   ```

4. **自动化合规检查**
   - CI/CD 中检测对此文件的修改
   - 自动通知 Safeguards 团队进行审查
   - 阻止未经审批的修改合并

5. **指令效果评估**
   - 定期评估指令在实际对话中的效果
   - 收集边界案例进行迭代优化
   - A/B 测试不同表述的效果

### 修改流程

如需修改此文件，必须遵循：

1. 联系 Safeguards 团队 (David Forsythe, Kyla Guru)
2. 确保对变更进行适当评估
3. 在合并前获得明确批准

**注意**：Claude 不应编辑此文件，除非用户明确要求这样做。
