# 项目整体目标 (Project Overview)

## 1. 系统定位
构建一个：**可控、可扩展、可自愈、可审计、可长期运行的智能任务执行平台**

系统本质：**状态机驱动的认知执行系统（State-Driven Cognitive Execution System）**

## 2. 核心设计哲学（5 条铁律）

| 原则 | 说明 | 违反后果 |
|------|------|----------|
| 控制权分离 | 流程控制权永远属于 Execution Engine，而非 LLM | 状态机失控 |
| 认知与控制分离 | 智能体只负责认知，不负责状态跳转 | Agent 越权修改状态 |
| 可追溯性 | 所有执行必须可回放、可追溯 | 无法审计调试 |
| 自愈优先 | 失败是常态，系统必须自愈 | 任务中断无法恢复 |
| 状态隔离 | 状态分层隔离，避免污染 | 数据一致性破坏 |

## 3. 六层架构

```
User Interface Layer
        ↓
Execution Engine (State Machine Core)  ← 系统大脑，独占控制权
        ↓
Cognitive Agent Layer                   ← 5 类 Agent，只读状态
        ↓
Policy & Governance Layer               ← 规则引擎、风险控制、权限系统
        ↓
Tool Runtime Layer                      ← 工具注册、强校验、沙箱
        ↓
State & Memory Infrastructure           ← 三层状态隔离、快照、记忆
```

**依赖方向铁律**：`api → engine → agents → tools → policy → state → memory`

## 4. 核心状态机（13 个状态）

```
INIT → CONTEXT_BUILD → PLAN_GENERATION → PLAN_CHECK → EXECUTION_PREPARE 
     → STEP_EXECUTION → STEP_REVIEW → GLOBAL_REVIEW → COMPLETED
                              ↓              ↓
                          REPLAN         WAIT_HUMAN
                              ↓
                          ROLLBACK → FAILED
```

**状态转移规则**：`Current State + Trigger Event + Guard Condition → Next State`

## 5. 五类认知 Agent

| Agent | 职责 | 输出 | 触发状态 |
|-------|------|------|----------|
| Context Builder | 整理用户输入，构建结构化目标 | 结构化任务定义 | CONTEXT_BUILD |
| Planner | 生成可执行计划 | ExecutionPlan | PLAN_GENERATION |
| Plan Critic | 审核计划完整性 | 评分 + 错误列表 | PLAN_CHECK |
| Step Executor | 生成工具参数，解析结果 | 工具输入/输出 | STEP_EXECUTION |
| Reviewer | 全局监管，拥有否决权 | 信心评分 | STEP_REVIEW/GLOBAL_REVIEW |

## 6. 自愈机制

| 类型 | 触发条件 | 动作 |
|------|----------|------|
| 自动重试 | 工具失败，retryable=true | 指数退避重试 |
| 局部重规划 | 单步骤失败 | 修正当前 Plan |
| 全局重规划 | 计划结构错误 | 恢复快照，重新生成 Plan |

## 7. 三层状态隔离

| 层级 | 可变性 | 用途 | 回滚支持 |
|------|--------|------|----------|
| GlobalState | Immutable | 原始任务、生命周期阶段 | 否（仅 Engine 更新引用） |
| ExecutionContext | Mutable | 当前计划、中间结果 | 是（快照恢复） |
| StepContext | Ephemeral | 当前工具输入输出 | 否（执行后清理） |

## 8. 可观测性要求

所有关键路径必须记录：
- 状态转移日志
- Agent 决策日志
- 工具调用日志
- 监管决策日志
- 执行时间统计

支持完整 Trace 回放。

## 9. 终止条件

任务完成必须同时满足：
- [ ] 所有步骤成功
- [ ] Reviewer 评分 ≥ 阈值
- [ ] 无未解决错误
- [ ] 迭代次数未超限

否则不可进入 COMPLETED 状态。