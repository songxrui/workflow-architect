---
name: workflow-architect
description: "高级工作流架构师。融合workflow-composer(编排)、dmux-workflows(多agent)、task-capsule-builder(任务胶囊)、heartbeat(心跳监控)、rollback(回滚恢复)。触发词：构建工作流架构/复杂工作流/多agent编排/带监控的工作流/生产级工作流/任务胶囊/回滚机制。不适用场景：简单线性流程(≤3步)→workflow-composer; 单agent任务→dmux-workflows; 一次性脚本→不触发。正例：'帮我设计一个带心跳监控和自动回滚的内容生产工作流'→触发; '把skill-review-master的评测流程封装成可监控的工作流'→触发。反例：'把这两个skill串起来'→不触发→workflow-composer; '只跑一个dmux任务'→不触发→dmux-workflows。"
version: "1.0.0 | created: 2026-06-08 | model: DeepSeek v4 Pro"
---

# Workflow-Architect — 高级工作流架构师

生产级工作流构建器。融合编排、多agent、心跳监控、回滚恢复四大能力。

## 架构层次

```
┌─────────────────────────────────────────┐
│ L4: 监控与恢复层                          │
│ heartbeat(每30min) + rollback(自动回滚)    │
├─────────────────────────────────────────┤
│ L3: 任务胶囊层                            │
│ task-capsule: 输入→执行→验证→输出 原子化   │
├─────────────────────────────────────────┤
│ L2: 多Agent编排层                         │
│ dmux: parallel/sequential/pipeline 模式   │
├─────────────────────────────────────────┤
│ L1: 基础编排层                            │
│ workflow-composer: skill链+handoff+verify │
└─────────────────────────────────────────┘
```

---

## L1: 基础编排（继承 workflow-composer）

每个 workflow 必须包含：
- **Goal**: 一句话定义产出
- **Phases**: 2-7 个阶段，每阶段有：输入/输出/验证
- **Handoff**: 阶段间传递格式
- **Rollback**: 中止条件 + 回滚步骤

---

## L2: 多 Agent 编排（继承 dmux-workflows）

支持三种并行模式：

| 模式 | 适用场景 | 示例 |
|------|---------|------|
| **parallel** | 独立子任务(写集不重叠) | 同时写3篇不同主题文章 |
| **sequential** | 有依赖关系的任务链 | 先评测→再优化→再验证 |
| **pipeline** | 流水线(阶段间传递中间产物) | 信源→结构化→创作→发布 |

---

## L3: 任务胶囊（task-capsule-builder 模式）

每个阶段封装为独立胶囊：

```yaml
capsule:
  id: "<unique-id>"
  skill: "<skill-name>"
  input: "<input spec>"
  expected_output: "<output spec>"
  verification:
    - check: "<condition>"
      on_fail: "<action>"
  timeout_ms: 300000
  retry: 2
  rollback_on_fail: true
```

---

## L4: 监控与恢复

### 心跳监控 (每 30 分钟)

```yaml
heartbeat:
  interval: 1800000  # 30min
  report:
    - current_phase
    - completed_capsules / total_capsules
    - last_error
    - estimated_remaining
    - is_blocked
  alert:
    - on_blocked: "通知用户或尝试自动恢复"
    - on_timeout: "kill并重启当前capsule"
    - on_quality_drop: "暂停并等待人工判断"
```

### 自动回滚

| 触发条件 | 回滚动作 |
|---------|---------|
| 连续 2 次门禁不通过 | 回到上一个通过的 Phase 重做 |
| Capsule 超时 | 重试 2 次后跳过(标记 degraded) |
| 输出质量分 < 阈值 | 回到该 Phase 的输入状态，更换 skill 重试 |
| 磁盘空间不足 | 暂停所有写入，清理临时文件后恢复 |

---

## 工作流设计模板

```markdown
# <Workflow Name>

**Goal**: 一句话

## Architecture
- L1: workflow-composer (基础编排)
- L2: dmux (多agent, parallel/sequential/pipeline)
- L3: task-capsule ×N (原子化执行)
- L4: heartbeat + rollback (监控恢复)

## Capsules

### Capsule 1: <name>
- skill: <skill-name>
- input: <spec>
- output: <spec>
- verification: <conditions>

### Capsule 2: <name>
...

## Execution Plan
1. parallel: Capsule 1 + Capsule 2
2. sequential: Capsule 3 → Capsule 4
3. pipeline: Capsule 5 | Capsule 6

## Monitoring
- heartbeat: 30min
- alert: on_blocked / on_timeout / on_quality_drop

## Rollback
- abort_if: <condition>
- rollback_to: <phase>
```

---

## 内置工作流

### 1. Skill 评测优化循环

```
Capsule 1 (scan) → Capsule 2 (score) → Capsule 3 (optimize) → Capsule 4 (validate)
[parallel: 同时扫描多个skill]              [SkillOpt循环: rollout→edit→validate]
```

### 2. 内容全流程生产

```
Capsule 1 (source) → Capsule 2 (structure) → Capsule 3 (write) → Capsule 4 (humanize) → Capsule 5 (publish)
[sequential pipeline]
```

### 3. 知识库质量审计

```
Capsule 1 (audit_all) → Capsule 2 (flag_low_quality) → Capsule 3 (fix_batch) → Capsule 4 (reaudit)
[parallel scan → sequential fix → pipeline verify]
```

---

## 失败模式与兜底

| 失败模式 | 原因 | 兜底动作 |
|---------|------|---------|
| 工作流卡死在某个 Capsule(>3次重试仍失败) | Capsule输入异常或skill不兼容 | 标记该Capsule为degraded，跳过继续执行，末尾汇总 |
| 并行 Capsule 产出冲突(写同文件) | 写集重叠 | 确保每个 Capsule 有独立输出文件，冲突时后写者标记冲突 |
| 心跳超时无响应(agent挂死) | 底层模型或进程异常 | 外部监控进程 kill + restart，从最近 checkpoint 恢复 |
| 回滚后状态不一致 | 部分 Capsule 已提交但未回滚 | 每个 Capsule 独立 git commit，回滚时 git revert |
| 连续回滚导致无限循环 | 回滚条件过于敏感 | 设置最大回滚次数(3次)，超过则暂停等人工 |

---

## G1-G6 自检

| 门禁 | 要求 | 状态 |
|------|------|------|
| G1 大小 | ≤10KB | ✅ |
| G2 触发层 | ≥5触发词/不适用场景/正反例/边界 | ✅ |
| G3 可执行 | 4层架构 + 3个内置工作流可执行 | ✅ |
| G4 验证 | Capsule 级验证 + Phase 门禁 | ✅ |
| G5 失败兜底 | ≥5失败模式+兜底(含回滚) | ✅ |
| G6 安全 | 无硬编码凭据 | ✅ |