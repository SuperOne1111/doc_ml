# 关键接口抽象框架（最终修订版）

**版本**：2.0  
**依据**：《多智能体协作系统需求思路详述.md》  
**目标**：构建可控、可扩展、可自愈、可审计、可长期运行的智能任务执行平台。

---

## 一、总体依赖方向（强制规则）

```text
api → engine → agents → tools
              ↓
            policy
              ↓
             state
              ↓
            memory
```

**核心铁律**：

1. **控制权分离**：流程控制权永远属于 **Execution Engine**，而非 LLM。
2. **状态只写**：**Engine** 是唯一允许修改 `GlobalState` 和 `ExecutionContext` 的组件。
3. **Agent 只读**：Agent 只读 GlobalState，不允许直接修改 State，只返回 `AgentOutput`。
4. **工具隔离**：Tool 不可访问 ExecutionEngine、Agent 或 Policy。
5. **依赖倒置**：所有实现必须依赖抽象接口，禁止依赖具体类。

---

## 二、核心类型定义（Pydantic 强类型）

统一数据协议，确保类型安全与可序列化。

```python
from pydantic import BaseModel, Field, ConfigDict
from typing import List, Dict, Any, Optional, Literal
from enum import Enum
from uuid import UUID, uuid4
from abc import ABC, abstractmethod
from datetime import datetime

# ==============================================================================
# 1️⃣ 状态枚举 (严格匹配需求文档状态机)
# ==============================================================================

class LifecycleState(str, Enum):
    """生命周期状态 (Engine 独占控制权)"""
    INIT = "INIT"
    CONTEXT_BUILD = "CONTEXT_BUILD"
    PLAN_GENERATION = "PLAN_GENERATION"
    PLAN_CHECK = "PLAN_CHECK"
    EXECUTION_PREPARE = "EXECUTION_PREPARE"
    STEP_EXECUTION = "STEP_EXECUTION"
    STEP_REVIEW = "STEP_REVIEW"
    GLOBAL_REVIEW = "GLOBAL_REVIEW"
    REPLAN = "REPLAN"
    WAIT_HUMAN = "WAIT_HUMAN"
    ROLLBACK = "ROLLBACK"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"

class StepStatus(str, Enum):
    """步骤执行状态 (支持并行跟踪与自愈)"""
    PENDING = "PENDING"       # 等待执行
    RUNNING = "RUNNING"       # 并行执行中
    COMPLETED = "COMPLETED"   # 执行成功
    FAILED = "FAILED"         # 执行失败
    SKIPPED = "SKIPPED"       # 被跳过

class AgentRole(str, Enum):
    """认知 Agent 角色 (匹配需求文档 5 类 Agent)"""
    CONTEXT_BUILDER = "CONTEXT_BUILDER"
    PLANNER = "PLANNER"
    PLAN_CRITIC = "PLAN_CRITIC"
    STEP_EXECUTOR = "STEP_EXECUTOR"
    REVIEWER = "REVIEWER"

class RiskLevel(str, Enum):
    """风险等级 (用于策略治理)"""
    LOW = "LOW"
    MEDIUM = "MEDIUM"
    HIGH = "HIGH"
    CRITICAL = "CRITICAL"

class PermissionLevel(str, Enum):
    """工具权限等级 (用于权限系统)"""
    PUBLIC = "PUBLIC"         # 所有 Agent 可调用
    INTERNAL = "INTERNAL"     # 仅受信任 Agent 可调用
    ADMIN = "ADMIN"           # 需人工确认或最高权限

class TraceEventType(str, Enum):
    """可观测性事件类型 (支持完整 Trace 回放)"""
    STATE_TRANSITION = "STATE_TRANSITION"
    AGENT_DECISION = "AGENT_DECISION"
    TOOL_CALL_START = "TOOL_CALL_START"
    TOOL_CALL_END = "TOOL_CALL_END"
    POLICY_EVALUATION = "POLICY_EVALUATION"
    SNAPSHOT_CREATED = "SNAPSHOT_CREATED"
    SNAPSHOT_RESTORED = "SNAPSHOT_RESTORED"
    HUMAN_INTERACTION = "HUMAN_INTERACTION"
    ERROR_OCCURRED = "ERROR_OCCURRED"

class MemoryScope(str, Enum):
    """记忆范围 (支持长期记忆与上下文隔离)"""
    EPHEMERAL = "EPHEMERAL"   # 当前步骤临时变量 (Step Context)
    SESSION = "SESSION"       # 当前任务执行上下文 (Execution Context)
    GLOBAL = "GLOBAL"         # 跨任务长期记忆 (Long-term Memory)

# ==============================================================================
# 2️⃣ 计划结构 (支持并行依赖)
# ==============================================================================

class PlanStep(BaseModel):
    id: str
    description: str
    tool_name: str
    input_schema: Dict[str, Any]
    expected_output: Optional[str] = None
    dependencies: List[str] = []  # 依赖步骤 ID 列表 (为空则可并行)
    timeout_ms: Optional[int] = None # 步骤级超时覆盖

class ExecutionPlan(BaseModel):
    goal: str
    steps: List[PlanStep]
    metadata: Dict[str, Any] = {}
    created_at: datetime = Field(default_factory=datetime.now)

# ==============================================================================
# 3️⃣ 状态分层模型 (严格隔离)
# ==============================================================================

class GlobalState(BaseModel):
    """
    Immutable Global State (不可变全局状态)
    仅 Engine 可更新引用，Agent 只读
    """
    model_config = ConfigDict(frozen=True) # 强制不可变

    execution_id: UUID = Field(default_factory=uuid4)
    original_goal: str
    lifecycle_state: LifecycleState
    iteration_count: int = 0
    trace_id: str
    created_at: datetime = Field(default_factory=datetime.now)

class ExecutionContext(BaseModel):
    """
    Execution Context (可快照回滚)
    包含当前计划、中间结果、错误记录
    """
    current_plan: Optional[ExecutionPlan] = None
    active_steps: Dict[str, StepStatus] = {}  # 支持并行状态跟踪
    current_batch_id: Optional[str] = None    # 支持并行批次追踪
    replan_scope: Optional[Literal["LOCAL", "GLOBAL"]] = None # 自愈范围标记
    intermediate_results: Dict[str, Any] = {}
    errors: List[str] = []
    snapshot_id: Optional[str] = None # 当前关联的快照 ID

class StepContext(BaseModel):
    """
    Ephemeral Step Context (临时上下文)
    执行完成后清理，仅包含当前步骤必要信息
    """
    step_id: str 
    tool_input: Optional[Dict[str, Any]] = None
    tool_output: Optional[Any] = None
    validation_flags: Dict[str, bool] = {}
    start_time: Optional[datetime] = None
    end_time: Optional[datetime] = None

# ==============================================================================
# 4️⃣ 错误与决策协议 (支持自愈与治理)
# ==============================================================================

class StructuredError(BaseModel):
    """
    结构化错误 (禁止抛裸异常跨层传播)
    支持自愈逻辑判断
    """
    code: str
    message: str
    severity: Literal["INFO", "WARNING", "CRITICAL"]
    retryable: bool = False           # 指示引擎是否应自动重试
    suggested_action: Optional[Literal["RETRY", "REPLAN", "ROLLBACK", "HALT"]] = None
    metadata: Dict[str, Any] = {}

class PolicyDecision(BaseModel):
    """
    策略决策 (包含风险控制与权限判断)
    """
    allow: bool
    next_state: Optional[LifecycleState] = None
    reason: Optional[str] = None
    risk_level: RiskLevel = RiskLevel.LOW
    require_human_approval: bool = False  # 强制人工介入标记
    blocked_tools: List[str] = []         # 因权限被拦截的工具

class AgentOutput(BaseModel):
    success: bool
    data: Optional[Any] = None
    confidence: float = Field(ge=0.0, le=1.0, default=1.0)
    errors: List[StructuredError] = []
    role: AgentRole # 明确输出所属 Agent 角色

class ToolExecutionResult(BaseModel):
    success: bool
    output: Optional[Any]
    error: Optional[StructuredError] = None
    latency_ms: Optional[int] = None
    side_effect_occurred: bool = False # 记录是否产生副作用

class EngineResult(BaseModel):
    success: bool
    final_output: Optional[Any]
    trace_id: str
    errors: List[StructuredError] = []
    total_latency_ms: Optional[int] = None
```

---

## 三、核心抽象接口定义

所有核心组件必须继承以下抽象类，实现多态与插件化。

### 1️⃣ BaseAgent 抽象接口 (认知层)

```python
class BaseAgent(ABC):
    """
    认知 Agent 只负责思考与评估，不做控制。
    对应需求：Context Builder, Planner, Critic, Executor, Reviewer
    """

    @property
    @abstractmethod
    def name(self) -> str:
        pass

    @property
    @abstractmethod
    def role(self) -> AgentRole:
        """明确 Agent 角色，以便 Engine 路由逻辑"""
        pass

    @abstractmethod
    async def run(
        self,
        global_state: GlobalState,
        execution_context: ExecutionContext,
        step_context: Optional[StepContext] = None,
    ) -> AgentOutput:
        """
        执行认知任务。
        约束：只读 global_state，不可修改 state。
        """
        pass

    @abstractmethod
    def validate_output(self, output: AgentOutput) -> bool:
        """本地输出格式校验"""
        pass
```

### 2️⃣ BaseTool 抽象接口 (工具运行层)

```python
class BaseTool(ABC):
    """
    工具层完全独立于 Agent。
    对应需求：工具注册中心声明 (name, version, schema, timeout, permission, side_effect)
    """

    @property
    @abstractmethod
    def name(self) -> str:
        pass

    @property
    @abstractmethod
    def version(self) -> str:
        """工具版本号，用于兼容性管理"""
        pass

    @property
    @abstractmethod
    def input_schema(self) -> Dict[str, Any]:
        pass

    @property
    @abstractmethod
    def output_schema(self) -> Dict[str, Any]:
        pass

    @property
    @abstractmethod
    def timeout_ms(self) -> int:
        """超时控制，防止挂起"""
        pass

    @property
    @abstractmethod
    def permission_level(self) -> PermissionLevel:
        """权限等级，用于 Policy 校验"""
        pass

    @property
    @abstractmethod
    def has_side_effect(self) -> bool:
        """是否产生副作用，用于回滚策略判断"""
        pass

    @abstractmethod
    async def execute(self, input_data: Dict[str, Any]) -> ToolExecutionResult:
        """
        执行工具。
        约束：不可访问 Engine, Agent, Policy。
        必须包含 Schema 验证与异常捕获。
        """
        pass
```

### 3️⃣ ExecutionEngine 抽象接口 (控制核心)

```python
class BaseExecutionEngine(ABC):
    """
    系统真正的“大脑”。
    对应需求：维护状态机、管理生命周期、控制 Agent 调用、处理异常与回滚。
    """

    @abstractmethod
    async def start(self, user_input: str) -> EngineResult:
        """启动任务，初始化 GlobalState"""
        pass

    @abstractmethod
    async def transition(self) -> None:
        """
        驱动状态机流转。
        规则：Current State + Trigger Event + Guard Condition → Next State
        这是唯一允许修改 GlobalState.lifecycle_state 的地方。
        """
        pass

    @abstractmethod  
    async def submit_human_feedback(self, feedback: str) -> None:
        """处理 WAIT_HUMAN 状态后的用户输入"""
        pass

    @abstractmethod
    async def rollback(self, scope: Literal["LOCAL", "GLOBAL"]) -> None:
        """
        触发回滚。
        调用 SnapshotManager 恢复 ExecutionContext。
        """
        pass

    @abstractmethod
    def register_agent(self, agent: BaseAgent) -> None:
        pass

    @abstractmethod
    def register_tool(self, tool: BaseTool) -> None:
        pass

    @abstractmethod
    def set_policy(self, policy: BasePolicy) -> None:
        """注入策略引擎"""
        pass
```

### 4️⃣ Policy 抽象接口 (治理层)

```python
class BasePolicy(ABC):
    """
    策略与治理层。
    对应需求：规则引擎、风险控制模块、权限系统。
    """

    @abstractmethod
    def evaluate_transition(
        self,
        global_state: GlobalState,
        execution_context: ExecutionContext,
        agent_output: Optional[AgentOutput] = None,
        tool_result: Optional[ToolExecutionResult] = None,
    ) -> PolicyDecision:
        """
        评估状态转移是否允许。
        检查：连续失败次数、数据来源风险、工具权限、置信度阈值。
        """
        pass

    @abstractmethod
    def check_tool_permission(
        self,
        agent_role: AgentRole,
        tool: BaseTool,
    ) -> bool:
        """
        权限系统核心：控制哪些 Agent 可调用哪些工具。
        """
        pass
```

### 5️⃣ Memory 抽象接口 (记忆基础设施)

```python
class BaseMemory(ABC):
    """
    支持持续优化与长期记忆。
    对应需求：成功案例缓存、失败模式库、工具统计系统。
    """

    @abstractmethod
    async def store(self, key: str, value: Any, scope: MemoryScope) -> None:
        """
        存储记忆。
        scope 决定存储位置 (Redis/VectorDB/Local)。
        """
        pass

    @abstractmethod
    async def retrieve(self, key: str, scope: MemoryScope) -> Any:
        pass

    @abstractmethod
    async def search(self, query: str, scope: MemoryScope) -> List[Any]:
        """
        语义搜索，用于 Planner 参考历史成功案例。
        """
        pass

    @abstractmethod
    async def record_failure_pattern(self, pattern: Dict[str, Any]) -> None:
        """专门记录失败模式，用于风险预警"""
        pass
```

### 6️⃣ Snapshot 抽象接口 (自愈基础)

```python
class BaseSnapshotManager(ABC):
    """
    回滚机制核心。
    对应需求：每个关键阶段创建快照，支持多级回滚。
    """

    @abstractmethod
    async def create_snapshot(
        self,
        execution_context: ExecutionContext,
        label: str,
    ) -> str:
        """返回 snapshot_id"""
        pass

    @abstractmethod
    async def restore_snapshot(
        self,
        snapshot_id: str,
    ) -> ExecutionContext:
        pass
```

### 7️⃣ BaseTracer 抽象接口 (可观测性)

```python
class BaseTracer(ABC):
    """
    可观测性系统。
    对应需求：状态转移日志、Agent 决策日志、工具调用日志、支持完整 Trace 回放。
    """

    @abstractmethod
    async def record_event(
        self, 
        event_type: TraceEventType, 
        payload: Dict[str, Any],
        trace_id: str
    ):
        """
        记录标准化事件。
        必须包含时间戳、上下文 ID、事件类型。
        """
        pass

    @abstractmethod
    async def get_trace(self, trace_id: str) -> List[Dict[str, Any]]:
        """支持 Trace 回放"""
        pass
```

---

## 四、注册与依赖注入

```python
class AgentRegistry:
    def __init__(self):
        self._agents: Dict[str, BaseAgent] = {}

    def register(self, agent: BaseAgent):
        if agent.name in self._agents:
            raise ValueError(f"Agent {agent.name} already registered")
        self._agents[agent.name] = agent

    def get_by_role(self, role: AgentRole) -> BaseAgent:
        """通过角色获取 Agent，便于 Engine 调度"""
        for agent in self._agents.values():
            if agent.role == role:
                return agent
        raise ValueError(f"No agent found for role {role}")

    def get(self, name: str) -> BaseAgent:
        return self._agents[name]

# 工具注册同理，需增加权限校验逻辑
class ToolRegistry:
    def __init__(self):
        self._tools: Dict[str, BaseTool] = {}

    def register(self, tool: BaseTool):
        self._tools[tool.name] = tool

    def get(self, name: str) -> BaseTool:
        return self._tools.get(name)
```

---

## 五、扩展能力保证 (架构契约)

此抽象框架保证以下扩展能力，实现类必须遵守：

1. **模型无关性**：可替换 Agent 底层模型（Ollama / OpenAI / Claude），接口不变。
2. **引擎可插拔**：可实现分布式 Engine 或 LangGraph 适配器 (`GraphAdapter`)。
3. **多租户隔离**：`GlobalState.execution_id` 与 `Memory.scope` 支持多租户数据隔离。
4. **热更新策略**：`BasePolicy` 实现可动态加载规则，无需重启 Engine。
5. **自愈闭环**：`StructuredError.suggested_action` 与 `Engine.rollback` 形成自动修复闭环。
6. **审计合规**：`BaseTracer` 确保所有决策、状态变更、工具调用均有日志留痕。

---

## 六、实施检查清单 (Checklist)

在实现具体类时，必须通过以下检查：

- [ ] **状态修改**：是否只有 Engine 在修改 `GlobalState.lifecycle_state`？
- [ ] **工具元数据**：`BaseTool` 实现是否包含了 `timeout`, `permission_level`, `side_effect`？
- [ ] **错误处理**：是否所有异常都捕获并转换为 `StructuredError`？
- [ ] **记忆范围**：`Memory` 调用是否明确指定了 `MemoryScope`？
- [ ] **可观测性**：关键路径是否调用了 `Tracer.record_event` 且使用了标准 `TraceEventType`？
- [ ] **权限控制**：工具执行前是否经过了 `Policy.check_tool_permission`？
- [ ] **并行支持**：`ExecutionContext.active_steps` 是否被正确用于跟踪并行任务状态？
- [ ] **回滚测试**：是否验证了 `SnapshotManager` 能正确恢复 `ExecutionContext`？

---

**备注**：本框架为平台级契约，具体业务逻辑（如 Prompt 模板、具体工具实现、规则引擎细节）应在实现层完成，严禁污染抽象层。
