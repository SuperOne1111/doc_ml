# 多智能体协作系统 - 完整执行计划

## 阶段 0：项目初始化 ✅ 已完成

| 序号 | 任务 | 文件 | 状态 |
|------|------|------|------|
| 0.1 | 核心类型定义 | src/core/types.py | ✅ |
| 0.2 | 核心数据模型 | src/core/models.py | ✅ |
| 0.3 | 错误与决策协议 | src/core/protocols.py | ✅ |
| 0.4 | 抽象接口定义 | src/core/interfaces.py | ✅ |
| 0.5 | Agent 注册中心 | src/registry/agent_registry.py | ✅ |
| 0.6 | Tool 注册中心 | src/registry/tool_registry.py | ✅ |
| 0.7 | 模块__init__.py | 所有模块 | ✅ |
| 0.8 | 初始化验证测试 | tests/test_initialization.py | ✅ |

---

## 阶段 1：基础设施层（Infrastructure Layer）

| 序号 | 任务 | 文件 | 优先级 | 预计耗时 |
|------|------|------|--------|----------|
| 1.1 | Tracer 抽象基类 | src/infrastructure/tracer/base_tracer.py | P0 | 1h |
| 1.2 | Tracer 控制台实现 | src/infrastructure/tracer/console_tracer.py | P0 | 1h |
| 1.3 | Tracer 单元测试 | tests/infrastructure/test_tracer.py | P0 | 1h |
| 1.4 | Memory 抽象基类 | src/infrastructure/memory/base_memory.py | P0 | 1h |
| 1.5 | Memory 本地实现 | src/infrastructure/memory/local_memory.py | P1 | 2h |
| 1.6 | Memory 单元测试 | tests/infrastructure/test_memory.py | P0 | 1h |
| 1.7 | Snapshot 抽象基类 | src/infrastructure/snapshot/base_snapshot.py | P0 | 1h |
| 1.8 | Snapshot JSON 实现 | src/infrastructure/snapshot/json_snapshot.py | P0 | 2h |
| 1.9 | Snapshot 单元测试 | tests/infrastructure/test_snapshot.py | P0 | 1h |

**阶段验收标准**：
- [ ] 所有基础设施组件通过 mypy 类型检查
- [ ] 所有单元测试通过（pytest）
- [ ] Tracer 能记录并回放完整 Trace
- [ ] Memory 支持三种 Scope 存储
- [ ] Snapshot 能正确创建和恢复 ExecutionContext
- [ ] Snapshot.create_snapshot 实现深拷贝，防止状态污染
- [ ] Tool 的 suggested_action 仅限 RETRY 或 HALT（不允许建议 ROLLBACK/REPLAN）
- [ ] PolicyDecision.validate_consistency 验证 allow=False 时 next_state 的合法性
- [ ] ExecutionContext 并发访问安全（锁机制或不可变副本）
- [ ] 恢复快照后 ExecutionContext.snapshot_id 被正确清空
- [ ] 恢复快照后 ExecutionContext.current_batch_id 被正确清空

---

## 阶段 2：执行引擎层（Execution Engine Layer）

| 序号 | 任务 | 文件 | 优先级 | 预计耗时 |
|------|------|------|--------|----------|
| 2.1 | Engine 骨架实现 | src/engine/execution_engine.py | P0 | 3h |
| 2.2 | 状态机规则定义 | src/engine/state_machine.py | P0 | 2h |
| 2.3 | 并行批次管理 | src/engine/batch_manager.py | P1 | 2h |
| 2.4 | Engine 单元测试 | tests/unit/test_engine.py | P0 | 2h |
| 2.5 | 状态流转集成测试 | tests/integration/test_workflow.py | P0 | 2h |

**阶段验收标准**：
- [ ] Engine 能正确初始化 GlobalState
- [ ] 状态转移严格遵循 13 个 LifecycleState
- [ ] 只有 Engine.transition() 能修改 lifecycle_state
- [ ] 支持 INIT → COMPLETED 硬编码流程跑通
- [ ] ExecutionContext 并发访问安全（锁机制或不可变副本）

---

## 阶段 3：治理层（Policy & Governance Layer）

| 序号 | 任务 | 文件 | 优先级 | 预计耗时 |
|------|------|------|--------|----------|
| 3.1 | Policy 抽象实现 | src/policy/base_policy.py | P0 | 2h |
| 3.2 | 状态转移规则引擎 | src/policy/rules.py | P0 | 2h |
| 3.3 | 风险控制模块 | src/policy/risk_control.py | P1 | 2h |
| 3.4 | 权限系统 | src/policy/permission.py | P0 | 2h |
| 3.5 | Policy 单元测试 | tests/unit/test_policy.py | P0 | 1h |

**阶段验收标准**：
- [ ] Policy.evaluate_transition() 能正确评估状态转移
- [ ] Policy.check_tool_permission() 能拦截未授权调用
- [ ] 支持风险等级评估和人工介入触发

---

## 阶段 4：认知 Agent 层（Cognitive Agent Layer）

| 序号 | 任务 | 文件 | 优先级 | 预计耗时 |
|------|------|------|--------|----------|
| 4.1 | Agent 通用骨架 | src/agents/base_agent.py | P0 | 1h |
| 4.2 | ContextBuilder Agent | src/agents/context_builder.py | P0 | 2h |
| 4.3 | Planner Agent | src/agents/planner.py | P0 | 3h |
| 4.4 | PlanCritic Agent | src/agents/plan_critic.py | P0 | 2h |
| 4.5 | StepExecutor Agent | src/agents/step_executor.py | P0 | 3h |
| 4.6 | Reviewer Agent | src/agents/reviewer.py | P0 | 2h |
| 4.7 | Agent 单元测试 | tests/unit/test_agents.py | P0 | 2h |

**阶段验收标准**：
- [ ] 5 类 Agent 全部继承 BaseAgent
- [ ] Agent 只读 GlobalState，不修改 State
- [ ] 所有 Agent 返回 AgentOutput 格式
- [ ] Reviewer 拥有否决权逻辑实现
- [ ] Agent.validate_output 验证 output.data 不包含生命周期状态转移相关字段

---

## 阶段 5：工具运行层（Tool Runtime Layer）

| 序号 | 任务 | 文件 | 优先级 | 预计耗时 |
|------|------|------|--------|----------|
| 5.1 | Tool 通用骨架 | src/tools/base_tool.py | P0 | 1h |
| 5.2 | Tool 注册中心 | src/tools/registry.py | P0 | 1h |
| 5.3 | Schema 验证器 | src/tools/validator.py | P0 | 1h |
| 5.4 | Search Tool | src/tools/builtin/search_tool.py | P1 | 2h |
| 5.5 | Calculator Tool | src/tools/builtin/calculator_tool.py | P1 | 1h |
| 5.6 | Tool 单元测试 | tests/unit/test_tools.py | P0 | 1h |

**阶段验收标准**：
- [ ] 所有 Tool 继承 BaseTool
- [ ] Tool 执行前经过 Schema 验证
- [ ] Tool 执行前经过 Policy 权限校验
- [ ] 错误封装为 StructuredError
- [ ] Tool 的 suggested_action 仅限 RETRY 或 HALT（不允许建议 ROLLBACK/REPLAN）

---

## 阶段 6：集成与自愈测试（Integration & Self-Healing）

| 序号 | 任务 | 文件 | 优先级 | 预计耗时 |
|------|------|------|--------|----------|
| 6.1 | 主入口文件 | src/main.py | P0 | 2h |
| 6.2 | 完整工作流测试 | tests/integration/test_workflow.py | P0 | 3h |
| 6.3 | 自愈机制测试 | tests/integration/test_self_healing.py | P0 | 3h |
| 6.4 | 回滚机制测试 | tests/integration/test_rollback.py | P0 | 2h |
| 6.5 | 人工介入测试 | tests/integration/test_human_in_loop.py | P1 | 2h |

**阶段验收标准**：
- [ ] 完整工作流 INIT → COMPLETED 跑通
- [ ] 计划评审失败能触发 REPLAN
- [ ] 严重错误能触发 ROLLBACK 并恢复快照
- [ ] 低置信度能触发 WAIT_HUMAN
- [ ] 完整 Trace 可回放

---

## 阶段 7：配置与部署（Configuration & Deployment）

| 序号 | 任务 | 文件 | 优先级 | 预计耗时 |
|------|------|------|--------|----------|
| 7.1 | 默认配置 | configs/default.yaml | P1 | 1h |
| 7.2 | 策略规则配置 | configs/policies.yaml | P1 | 1h |
| 7.3 | 权限配置 | configs/permissions.yaml | P1 | 1h |
| 7.4 | README 文档 | README.md | P1 | 2h |
| 7.5 | 依赖清单 | requirements.txt | P0 | 0.5h |

**阶段验收标准**：
- [ ] 配置支持热更新
- [ ] README 包含完整使用说明
- [ ] 一键启动脚本可用

---

## 总体里程碑

| 里程碑 | 完成阶段 | 预计总耗时 | 验收标准 |
|--------|----------|------------|----------|
| M1：核心层完成 | 阶段 0 | 4h | 所有接口和模型定义完成 |
| M2：基础设施完成 | 阶段 1 | 12h | Tracer/Memory/Snapshot可用 |
| M3：引擎可用 | 阶段 2 | 11h | 状态机能跑通硬编码流程 |
| M4：治理层完成 | 阶段 3 | 9h | 策略和权限系统可用 |
| M5：Agent 完成 | 阶段 4 | 15h | 5 类 Agent 全部实现 |
| M6：工具层完成 | 阶段 5 | 7h | 内置工具可用 |
| M7：集成测试完成 | 阶段 6 | 12h | 自愈机制验证通过 |
| M8：发布就绪 | 阶段 7 | 5.5h | 文档和配置完整 |

**预计总耗时**：约 75 小时（单人开发）