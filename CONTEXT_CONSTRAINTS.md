# 设计约束 (Context Constraints)

## ⚠️ 核心铁律（违反即架构错误）

### 1. 控制权分离
```
✅ 正确：Engine.transition() 修改 lifecycle_state
❌ 错误：Agent.run() 直接修改 state
```

**检查清单**：
- [ ] 是否只有 Engine 在修改 `GlobalState.lifecycle_state`？
- [ ] Agent 是否只返回 `AgentOutput` 而不触碰 State？
- [ ] Tool 是否无法访问 Engine、Agent、Policy？

### 2. 状态只写
```
✅ 正确：Engine 创建新的 GlobalState 实例（Pydantic frozen）
❌ 错误：直接修改 GlobalState 字段
```

**检查清单**：
- [ ] GlobalState 是否设置了 `frozen=True`？
- [ ] 状态更新是否通过创建新实例实现？

### 3. Agent 只读
```
✅ 正确：Agent 读取 global_state.original_goal
❌ 错误：Agent 尝试修改 global_state.lifecycle_state
```

**检查清单**：
- [ ] Agent.run() 参数中 global_state 是否只读？
- [ ] Agent 输出是否为 AgentOutput 而非 State？

### 4. 工具隔离
```
✅ 正确：Tool.execute() 只处理输入输出
❌ 错误：Tool 调用 Engine.transition()
❌ 错误：Tool 建议 ROLLBACK 或 REPLAN（应由 Engine 决定）
```

**检查清单**：
- [ ] Tool 是否无法导入 Engine、Agent、Policy 模块？
- [ ] Tool 执行前是否经过 Policy.check_tool_permission？
- [ ] Tool 的 suggested_action 是否仅限 RETRY 或 HALT？

### 5. 错误处理
```
✅ 正确：捕获异常，转换为 StructuredError
❌ 错误：抛裸异常跨层传播
```

**检查清单**：
- [ ] 所有异常是否转换为 StructuredError？
- [ ] StructuredError 是否包含 suggested_action？

### 6. 并发安全
```
✅ 正确：ExecutionContext 修改通过 safe_update 方法
❌ 错误：直接修改 ExecutionContext 字段
```

**检查清单**：
- [ ] ExecutionContext 修改是否都通过 safe_update 方法？
- [ ] 并行执行中 ExecutionContext 是否存在竞态条件？
- [ ] Snapshot 恢复后 ExecutionContext.snapshot_id 是否被清空？

## 📋 实现检查清单（每次代码生成后自查）

| 检查项 | 验证方法 | 通过标准 |
|--------|----------|----------|
| 状态修改 | 代码审查 | 只有 Engine.transition() 修改 lifecycle_state |
| 工具元数据 | 检查 BaseTool 实现 | 包含 timeout, permission_level, side_effect |
| 错误处理 | 代码审查 | 所有 try/except 转换为 StructuredError |
| 记忆范围 | 检查 Memory 调用 | 明确指定 MemoryScope (EPHEMERAL/SESSION/GLOBAL) |
| 可观测性 | 检查关键路径 | 调用 Tracer.record_event 且使用标准 TraceEventType |
| 权限控制 | 检查工具调用前 | 经过 Policy.check_tool_permission |
| 并行支持 | 检查 ExecutionContext | active_steps 正确跟踪并行状态 |
| 回滚测试 | 运行测试用例 | SnapshotManager 能正确恢复 ExecutionContext |

## 🚫 禁止行为清单

1. ❌ 禁止 Agent 直接修改任何 State
2. ❌ 禁止 Tool 访问 Engine/Agent/Policy
3. ❌ 禁止抛裸异常跨层传播
4. ❌ 禁止私自新增 LifecycleState
5. ❌ 禁止绕过 Policy 进行状态转移
6. ❌ 禁止在 StepContext 中存储长期数据
7. ❌ 禁止跳过 Tracer 记录关键事件
8. ❌ 禁止在 COMPLETED 前跳过 Reviewer 评审

## ✅ 强制行为清单

1. ✅ 所有状态转移必须通过 Engine.transition()
2. ✅ 所有工具执行前必须经过权限校验
3. ✅ 所有关键阶段必须创建快照
4. ✅ 所有错误必须封装为 StructuredError
5. ✅ 所有关键路径必须调用 Tracer.record_event
6. ✅ 所有 Memory 操作必须指定 MemoryScope
7. ✅ 所有 Agent 输出必须包含 confidence 评分
8. ✅ 所有 Tool 必须声明 input_schema 和 output_schema