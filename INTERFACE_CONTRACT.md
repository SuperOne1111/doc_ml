# 接口契约 (Interface Contract)

## 1. 核心抽象类清单

所有实现类必须继承以下抽象类：

| 抽象类 | 位置 | 用途 |
|--------|------|------|
| BaseAgent | src/core/interfaces.py | 5 类认知 Agent 基类 |
| BaseTool | src/core/interfaces.py | 所有工具基类 |
| BaseExecutionEngine | src/core/interfaces.py | 执行引擎基类 |
| BasePolicy | src/core/interfaces.py | 策略引擎基类 |
| BaseMemory | src/core/interfaces.py | 记忆系统基类 |
| BaseSnapshotManager | src/core/interfaces.py | 快照管理基类 |
| BaseTracer | src/core/interfaces.py | 追踪系统基类 |

## 2. 核心数据模型

### GlobalState（不可变）
```python
class GlobalState(BaseModel):
    model_config = ConfigDict(frozen=True)  # ⚠️ 强制不可变
    execution_id: UUID
    original_goal: str
    lifecycle_state: LifecycleState
    iteration_count: int
    trace_id: str
    created_at: datetime
```

### ExecutionContext（可快照）
```python
class ExecutionContext(BaseModel):
    current_plan: Optional[ExecutionPlan]
    active_steps: Dict[str, StepStatus]  # 支持并行
    current_batch_id: Optional[str]
    replan_scope: Optional[Literal["LOCAL", "GLOBAL"]]
    intermediate_results: Dict[str, Any]
    errors: List[str]
    snapshot_id: Optional[str]
```

### StepContext（临时）
```python
class StepContext(BaseModel):
    step_id: str
    tool_input: Optional[Dict[str, Any]]
    tool_output: Optional[Any]
    validation_flags: Dict[str, bool]
    start_time: Optional[datetime]
    end_time: Optional[datetime]
```

## 3. 核心协议

### AgentOutput（Agent 统一输出）
```python
class AgentOutput(BaseModel):
    success: bool
    data: Optional[Any]
    confidence: float  # 0.0-1.0，用于 Reviewer 决策
    errors: List[StructuredError]
    role: AgentRole
```

### StructuredError（统一错误）
```python
class StructuredError(BaseModel):
    code: str
    message: str
    severity: Literal["INFO", "WARNING", "CRITICAL"]
    retryable: bool
    suggested_action: Optional[Literal["RETRY", "HALT"]]
    metadata: Dict[str, Any]
```

### PolicyDecision（策略决策）
```python
class PolicyDecision(BaseModel):
    allow: bool
    next_state: Optional[LifecycleState]
    reason: Optional[str]
    risk_level: RiskLevel
    require_human_approval: bool  # 强制人工介入
    blocked_tools: List[str]
    
    def validate_consistency(self) -> bool:
        """
        验证决策一致性
        当 allow=False 时，next_state 应为 None 或明确指向错误处理状态
        """
        if not self.allow:
            # 如果不允许继续，则不能有正常的下一步状态
            from src.core.types import LifecycleState
            if self.next_state is not None and self.next_state not in [
                LifecycleState.FAILED, 
                LifecycleState.ROLLBACK, 
                LifecycleState.WAIT_HUMAN,
                LifecycleState.REPLAN
            ]:
                return False
        return True
```

## 4. 核心枚举

### LifecycleState（13 个状态）
```python
INIT, CONTEXT_BUILD, PLAN_GENERATION, PLAN_CHECK, EXECUTION_PREPARE,
STEP_EXECUTION, STEP_REVIEW, GLOBAL_REVIEW, REPLAN, WAIT_HUMAN,
ROLLBACK, COMPLETED, FAILED
```

### AgentRole（5 类 Agent）
```python
CONTEXT_BUILDER, PLANNER, PLAN_CRITIC, STEP_EXECUTOR, REVIEWER
```

### TraceEventType（可观测性事件）
```python
STATE_TRANSITION, AGENT_DECISION, TOOL_CALL_START, TOOL_CALL_END,
POLICY_EVALUATION, SNAPSHOT_CREATED, SNAPSHOT_RESTORED,
HUMAN_INTERACTION, ERROR_OCCURRED
```

### MemoryScope（记忆范围）
```python
EPHEMERAL, SESSION, GLOBAL
```

## 5. 接口实现样例

### BaseAgent 实现样例
```python
from src.core.interfaces import BaseAgent
from src.core.types import AgentRole
from src.core.models import GlobalState, ExecutionContext, StepContext
from src.core.protocols import AgentOutput

class ContextBuilderAgent(BaseAgent):
    @property
    def name(self) -> str:
        return "context_builder"
    
    @property
    def role(self) -> AgentRole:
        return AgentRole.CONTEXT_BUILDER
    
    async def run(
        self,
        global_state: GlobalState,
        execution_context: ExecutionContext,
        step_context: Optional[StepContext] = None,
    ) -> AgentOutput:
        # ⚠️ 只读 global_state，不修改
        # ⚠️ 返回 AgentOutput，不直接触发状态转移
        return AgentOutput(
            success=True,
            data={"goal": "...", "constraints": []},
            confidence=0.95,
            errors=[],
            role=self.role
        )
    
    def validate_output(self, output: AgentOutput) -> bool:
        return output.success and output.data is not None
```

### BaseTool 实现样例
```python
from src.core.interfaces import BaseTool
from src.core.types import PermissionLevel
from src.core.protocols import ToolExecutionResult, StructuredError

class SearchTool(BaseTool):
    @property
    def name(self) -> str:
        return "search"
    
    @property
    def version(self) -> str:
        return "1.0.0"
    
    @property
    def input_schema(self) -> Dict[str, Any]:
        return {"type": "object", "properties": {"query": {"type": "string"}}}
    
    @property
    def output_schema(self) -> Dict[str, Any]:
        return {"type": "object", "properties": {"results": {"type": "array"}}}
    
    @property
    def timeout_ms(self) -> int:
        return 5000
    
    @property
    def permission_level(self) -> PermissionLevel:
        return PermissionLevel.PUBLIC
    
    @property
    def has_side_effect(self) -> bool:
        return False
    
    async def execute(self, input_data: Dict[str, Any]) -> ToolExecutionResult:
        try:
            # ⚠️ 不可访问 Engine, Agent, Policy
            # ⚠️ 必须包含 Schema 验证与异常捕获
            result = {"results": [...]}
            return ToolExecutionResult(success=True, output=result)
        except Exception as e:
            return ToolExecutionResult(
                success=False,
                error=StructuredError(
                    code="TOOL_EXEC_FAILED",
                    message=str(e),
                    severity="CRITICAL",
                    retryable=True,
                    suggested_action="RETRY"
                )
            )
```

## 6. 依赖注入规范

### Agent 注册
```python
from src.registry.agent_registry import AgentRegistry

registry = AgentRegistry()
registry.register(ContextBuilderAgent())
agent = registry.get_by_role(AgentRole.CONTEXT_BUILDER)
```

### Tool 注册
```python
from src.registry.tool_registry import ToolRegistry

registry = ToolRegistry()
registry.register(SearchTool())
tool = registry.get("search")
```

## 7. 类型检查要求

所有代码必须通过 `mypy` 检查：
```bash
mypy src/
```

**常见类型错误**：
- ❌ 将 `LifecycleState` 字符串直接赋值（应使用枚举）
- ❌ 将 `ExecutionContext` 传给只接受 `GlobalState` 的参数
- ❌ 忘记 `Optional` 字段的 None 检查
- ❌ Tool 的 `suggested_action` 包含 `REPLAN` 或 `ROLLBACK`（应由 Engine 决定）