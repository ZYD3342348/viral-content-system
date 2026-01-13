# 架构说明文档

## 方案演进

### 初始方案（错误）
- 误以为可以创建"子 skill"作为 subagent
- analyzer/writer/reviewer 作为独立的 skill
- viral-content 通过某种方式调用这些 skill

**问题**：
- Skill 不能互相调用
- Skill 不能作为 subagent 使用
- 系统无法协调工作

### 最终方案（正确）：Skill + Subagent 混合架构

**核心思想**：
- Skill 是**指令模板**，不是执行环境
- Subagent 是**执行环境**，通过 Task 工具调用
- Orchestrator 读取 skill 的 instructions，传递给 subagent

## 架构详解

### 1. 角色定位

**Analyzer/Writer/Reviewer Skills**：
- 作用：提供专业的提示词模板
- 位置：`skills/{name}/instructions.md`
- 不能被直接调用（除非用户手动 `/analyzer`）
- 被 Orchestrator 读取并传递给 subagent

**Viral-Content Skill (Orchestrator)**：
- 作用：主控制器，协调所有任务
- 读取其他 skill 的 instructions
- 调用 Task 工具启动 subagent
- 汇总结果并做决策

**General-Purpose Subagent**：
- 作用：实际执行任务的环境
- 通过 Task 工具调用
- 接收完整的提示词（instructions + 上下文）
- 有独立的上下文和工具集

### 2. 工作流程示例

#### Create 模式（选题驱动创作）

**步骤 1: Orchestrator 准备**
```
1. 读取 config/platforms.json
2. 读取 knowledge/rules/xiaohongshu/base_rules.json
3. 读取 skills/writer/instructions.md
```

**步骤 2: 构建提示词**
```
完整提示词 =
    writer/instructions.md 的内容
    + "---"
    + "## 当前任务上下文"
    + "选题: 如何提高工作效率"
    + "平台: xiaohongshu"
    + "平台配置: {...}"
    + "规则库: {...}"
```

**步骤 3: 调用 Subagent**
```python
Task(
    subagent_type="general-purpose",
    prompt=完整提示词,
    description="Generate content for topic"
)
```

**步骤 4: Subagent 执行**
- Writer subagent 在独立环境中执行
- 读取规则库
- 生成 3 个版本
- 保存到 workspace/drafts/
- 返回结果

**步骤 5: Orchestrator 继续**
- 接收 Writer 的结果
- 读取 skills/reviewer/instructions.md
- 构建新的提示词
- 调用 Reviewer subagent
- 根据评分决定是否优化

### 3. 关键优势

**模块化**：
- 每个 skill 的 instructions 独立维护
- 修改 Writer 的提示词不影响其他部分
- 易于测试和调试

**灵活性**：
- 可以复用 skill 的 instructions
- 可以根据需要调整提示词组合
- 支持并行调用多个 subagent

**清晰性**：
- 职责分明：Skill 负责提示词，Subagent 负责执行
- 易于理解和维护
- 符合 Claude Code 的设计理念

### 4. 与其他方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| 纯 Subagent | 简单直接 | 提示词都在一个文件中，难以维护 |
| 纯 Skill | 模块化 | Skill 无法互相调用，无法协调 |
| **Skill + Subagent** | 模块化 + 可协调 | 需要理解两者的区别 |
