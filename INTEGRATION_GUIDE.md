# ralph-loop + planning-with-files 集成指南

本文档详细说明如何将 ralph-loop 和 planning-with-files 集成到爆文创作系统中。

---

## 工具介绍

### planning-with-files

**作用**: 自动创建和管理任务规划文件

**创建的文件**:
- `task_plan.md`: 任务计划和阶段追踪
- `findings.md`: 记录新发现的规则和经验
- `progress.md`: 记录每次迭代的进度

**调用方式**:
```bash
/planning-with-files --task "任务描述" --phases "阶段1,阶段2,阶段3"
```

### ralph-loop

**作用**: 自动循环执行任务，直到满足完成条件

**核心参数**:
- `--max-iterations`: 最大迭代次数（防止无限循环）
- `--completion-promise`: 完成信号（如 "COMPLETE"）

**调用方式**:
```bash
/ralph-loop "任务描述和步骤" --max-iterations 30 --completion-promise "COMPLETE"
```

**完成条件**: 当输出包含 `<promise>COMPLETE</promise>` 时停止循环

---

## 集成方式

### 方式 1: ralph-loop 直接调用（推荐）

**优势**: 一键启动，全自动执行

**示例**:
```bash
/ralph-loop "为小红书平台创作关于'如何提高工作效率'的内容。

步骤：
1. 调用 planning-with-files 创建任务规划
2. 读取 config/platforms.json 和 knowledge/rules/xiaohongshu/base_rules.json
3. 调用 Writer subagent 生成内容（使用 Task 工具）
4. 调用 Reviewer subagent 评分（使用 Task 工具）
5. 如果评分 < 8.0，根据建议优化并重新生成
6. 每次迭代更新 progress.md
7. 评分达标后输出 <promise>COMPLETE</promise>

平台: xiaohongshu
选题: 如何提高工作效率" \
--max-iterations 30 --completion-promise "COMPLETE"
```

**工作流程**:
```
ralph-loop 启动
    ↓
第1次迭代:
  - 调用 planning-with-files 创建规划文件
  - 读取配置和规则库
  - 调用 Writer subagent 生成内容
  - 调用 Reviewer subagent 评分
  - 评分: 7.5 (< 8.0)
  - 更新 progress.md
    ↓
第2次迭代:
  - 读取上次的评分和建议
  - 调用 Writer subagent 优化内容
  - 调用 Reviewer subagent 评分
  - 评分: 8.2 (>= 8.0)
  - 更新 progress.md
  - 输出 <promise>COMPLETE</promise>
    ↓
ralph-loop 检测到完成信号，停止循环
```

---

### 方式 2: viral-content skill 内部集成

**优势**: 封装在 skill 内部，用户只需调用 skill

**实现**: 在 viral-content/instructions.md 中：

**步骤 1.3**: 调用 planning-with-files
```
Skill(
    skill: "planning-with-files",
    args: "--task '为{platform}平台创作关于{topic}的内容' --phases '生成初版内容,审核评分,根据反馈优化,最终确认'"
)
```

**步骤 5.3**: 输出完成信号
```
输出：<promise>COMPLETE</promise>
```

**用户调用**:
```bash
# 方式 2a: 单次执行
/viral-content --mode create --topic "如何提高工作效率" --platform xiaohongshu

# 方式 2b: 配合 ralph-loop 自动循环
/ralph-loop "调用 /viral-content --mode create --topic '如何提高工作效率' --platform xiaohongshu，如果评分 < 8.0 则继续优化，达标后输出 <promise>COMPLETE</promise>" \
--max-iterations 30 --completion-promise "COMPLETE"
```

---

## 集成检查清单

### planning-with-files 集成

- [x] 在 viral-content/instructions.md 中添加调用代码
- [x] 在 README.md 中添加使用示例
- [x] 创建 INTEGRATION_GUIDE.md 说明文档
- [ ] 实际测试 planning-with-files 调用
- [ ] 验证三个文件（task_plan.md, findings.md, progress.md）正确创建

### ralph-loop 集成

- [x] 在 viral-content/instructions.md 中添加完成信号输出
- [x] 在 README.md 中添加 ralph-loop 使用示例
- [x] 在步骤 5 添加 ralph-loop 使用建议
- [ ] 实际测试 ralph-loop 循环执行
- [ ] 验证完成信号 `<promise>COMPLETE</promise>` 正确触发停止

---

## 实际使用示例

### 示例 1: 完全自动化创作

**场景**: 用户提供选题，系统自动生成高质量内容

**命令**:
```bash
/ralph-loop "为小红书平台创作关于'职场新人如何快速适应新环境'的内容。

步骤：
1. 调用 planning-with-files 创建任务规划
2. 读取配置和规则库
3. 调用 Writer subagent 生成3个版本
4. 调用 Reviewer subagent 评分
5. 选择最高分版本，如果 < 8.0 则优化
6. 每次迭代更新 progress.md
7. 评分达标后输出 <promise>COMPLETE</promise>

平台: xiaohongshu
选题: 职场新人如何快速适应新环境" \
--max-iterations 30 --completion-promise "COMPLETE"
```

**预期结果**:
- 自动创建 task_plan.md, findings.md, progress.md
- 生成 3 个版本的内容
- 自动评分和优化
- 最多迭代 30 次
- 评分达标后自动停止

---

### 示例 2: 分析爆文并更新规则库

**场景**: 用户提供爆文，系统分析并提炼规则

**命令**:
```bash
/ralph-loop "分析以下小红书爆文并更新规则库：

[爆文内容]
标题：🔥7天瘦10斤！这个方法太绝了
正文：姐妹们！我用这个方法7天瘦了10斤...
[完整内容]

步骤：
1. 调用 Analyzer subagent 分析爆文
2. 提炼新规则
3. 读取现有规则库 knowledge/rules/xiaohongshu/base_rules.json
4. 合并新规则（避免重复）
5. 保存更新后的规则库
6. 更新 findings.md 记录新规则
7. 输出 <promise>COMPLETE</promise>

平台: xiaohongshu" \
--max-iterations 10 --completion-promise "COMPLETE"
```

**预期结果**:
- 分析爆文的标题、结构、互动设计
- 提炼 3-5 条新规则
- 更新规则库（权重、示例）
- 记录到 findings.md

---

### 示例 3: 批量创作多个选题

**场景**: 用户提供多个选题，系统批量生成内容

**命令**:
```bash
/ralph-loop "为小红书平台批量创作以下选题的内容：

选题列表：
1. 如何提高工作效率
2. 职场新人必备技能
3. 时间管理的5个技巧

步骤：
1. 调用 planning-with-files 创建任务规划
2. 对每个选题：
   a. 调用 Writer subagent 生成内容
   b. 调用 Reviewer subagent 评分
   c. 如果评分 < 8.0，优化一次
   d. 保存到 workspace/drafts/
3. 更新 progress.md 记录所有选题的进度
4. 全部完成后输出 <promise>COMPLETE</promise>

平台: xiaohongshu" \
--max-iterations 50 --completion-promise "COMPLETE"
```

**预期结果**:
- 生成 3 篇内容
- 每篇都经过评分和优化
- 所有内容保存到 workspace/drafts/
- progress.md 记录完整进度

---

## 调试技巧

### 1. 检查 planning-with-files 是否正常工作

**测试命令**:
```bash
/planning-with-files --task "测试任务" --phases "阶段1,阶段2,阶段3"
```

**预期结果**:
- 创建 task_plan.md
- 创建 findings.md
- 创建 progress.md

**如果失败**: 在 viral-content/instructions.md 中使用手动创建的备用方案

---

### 2. 检查 ralph-loop 是否正确识别完成信号

**测试命令**:
```bash
/ralph-loop "输出 <promise>COMPLETE</promise>" --max-iterations 5 --completion-promise "COMPLETE"
```

**预期结果**:
- 第1次迭代后立即停止
- 不会执行第2次迭代

**如果失败**: 检查完成信号格式是否正确（必须是 `<promise>COMPLETE</promise>`）

---

### 3. 检查 progress.md 是否正确更新

**测试步骤**:
1. 运行一次完整流程
2. 检查 progress.md 是否存在
3. 检查是否记录了每次迭代的信息

**预期内容**:
```markdown
## 2026-01-12 23:30
- 完成选题：如何提高工作效率
- 平台：xiaohongshu
- 最终评分：8.4
- 迭代次数：2
- 生成文件：workspace/drafts/xiaohongshu_工作效率_20260112.md
```

---

## 常见问题

### Q1: planning-with-files 调用失败怎么办？

**A**: viral-content/instructions.md 中已包含备用方案，会手动创建 task_plan.md。不影响核心功能。

---

### Q2: ralph-loop 无限循环怎么办？

**A**: 设置了 `--max-iterations 30`，最多执行30次后自动停止。可以根据需要调整这个数字。

---

### Q3: 如何查看每次迭代的详细信息？

**A**: 查看 progress.md 文件，记录了每次迭代的评分、建议、生成的文件等信息。

---

### Q4: 评分一直达不到 8.0 怎么办？

**A**:
1. 检查规则库是否完善
2. 检查 Reviewer 的评分标准是否合理
3. 降低评分阈值（如改为 7.5）
4. 增加最大迭代次数

---

### Q5: 如何停止正在运行的 ralph-loop？

**A**:
- 方式1: 手动输出 `<promise>COMPLETE</promise>`
- 方式2: 等待达到最大迭代次数
- 方式3: 使用 `/cancel-ralph` 命令（如果可用）

---

## 总结

### 集成优势

1. **planning-with-files**:
   - ✅ 自动创建任务规划文件
   - ✅ 记录每次迭代的进度
   - ✅ 防止上下文丢失
   - ✅ 便于追踪和回顾

2. **ralph-loop**:
   - ✅ 自动循环执行，无需手动催促
   - ✅ 设置评分阈值，自动迭代直到达标
   - ✅ 最大迭代次数保护，防止无限循环
   - ✅ 完成信号机制，精确控制停止时机

3. **组合使用**:
   - ✅ 完全自动化的内容创作流程
   - ✅ 持续学习和优化
   - ✅ 完整的进度追踪和记录
   - ✅ 可重复、可追溯的工作流程

### 下一步

1. 在新的 Claude Code 会话中测试完整的集成流程
2. 验证 planning-with-files 和 ralph-loop 的实际效果
3. 根据测试结果优化提示词和参数
4. 完善错误处理和边界情况

---

**文档版本**: 1.0
**最后更新**: 2026-01-12
