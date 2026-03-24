# Job Agent System - Architecture and Design Documentation

## 系统概述 (System Overview)

本文档详细说明了基于 AgentScope 框架构建的求职助手智能体系统的完整架构设计、API 接口、工具定义以及常见问题解答。

This document provides a comprehensive architecture design, API specifications, tool definitions, and FAQ for building a job-seeking assistant agent system based on the AgentScope framework.

---

## 系统架构图 (System Architecture)

### 新架构设计理念 (New Architecture Design Philosophy)

本系统采用**任务编排 + 多智能体协作 + 心跳机制**的架构设计，核心特点：

1. **任务编排中心 (Task Orchestrator)**：统一接收请求、任务分解、智能体调度、状态管理
2. **专用智能体池 (Specialized Agent Pools)**：针对不同任务类型的专用智能体池，支持并行处理
3. **状态机驱动 (State Machine Driven)**：清晰的状态边界和转换机制，实现流程平滑过渡
4. **心跳自治 (Heartbeat Autonomy)**：智能体定期/事件驱动自主执行任务，无需用户触发

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Mobile App (iOS/Android)                             │
│                    User Interface Layer                                  │
└────────────────────────┬────────────────────────────────────────────────┘
                         │ HTTPS REST API
                         │ (JWT Authentication)
                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Backend API Server                                  │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              API Gateway & Router                                  │  │
│  │  - User Authentication & Rate Limiting                             │  │
│  └────────────┬──────────────────────────────────────────────────────┘  │
│               │                                                           │
│  ┌────────────▼──────────────────────────────────────────────────────┐  │
│  │              Session & State Manager                               │  │
│  │  - User Session Tracking (user_id → session_id)                   │  │
│  │  - Task State Management (task_id → state + metadata)             │  │
│  │  - State Transition Events                                         │  │
│  └────────────┬──────────────────────────────────────────────────────┘  │
│               │                                                           │
│  ┌────────────▼──────────────────────────────────────────────────────┐  │
│  │         ⭐ Task Orchestrator Agent (核心编排智能体)               │  │
│  │                                                                     │  │
│  │  核心职责：                                                         │  │
│  │  1. 接收用户请求，理解意图                                         │  │
│  │  2. 任务分解与规划 (Task Decomposition)                           │  │
│  │  3. 智能体调度与资源分配                                           │  │
│  │  4. 状态机管理与转换控制                                           │  │
│  │  5. 结果聚合与响应生成                                             │  │
│  │  6. 异常处理与回滚                                                 │  │
│  │                                                                     │  │
│  │  任务编排策略：                                                     │  │
│  │  • 简单任务 → 直接分配单一 Agent                                  │  │
│  │  • 复杂任务 → 分解为多个子任务 → 并行/串行执行                    │  │
│  │  • 批量任务 → Coordinator + Worker Pool 模式                      │  │
│  └────────┬────────────────────────────────────────────────────────┘  │
│           │                                                              │
│           ├──────────────┬──────────────────┬──────────────────┐        │
│           ▼              ▼                  ▼                  ▼        │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐ ┌────────┐  │
│  │ Job Search     │ │ Resume         │ │ HR Comm        │ │ Monitor│  │
│  │ Agent Pool     │ │ Processor Pool │ │ Agent Pool     │ │ Agent  │  │
│  │ (10 agents)    │ │ (20 agents)    │ │ (5 agents)     │ │        │  │
│  └────────────────┘ └────────────────┘ └────────────────┘ └────────┘  │
│                                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              Heartbeat & Scheduler Service                         │  │
│  │                                                                     │  │
│  │  定期任务 (Periodic Tasks):                                        │  │
│  │  ├─ 每日职位推荐 (Daily Job Recommendations)                      │  │
│  │  ├─ 申请状态跟进 (Application Follow-ups)                         │  │
│  │  ├─ 面试提醒 (Interview Reminders)                                │  │
│  │  └─ HR 沟通跟进 (HR Communication Follow-ups)                     │  │
│  │                                                                     │  │
│  │  事件触发 (Event-Driven):                                          │  │
│  │  ├─ 新职位匹配 → 自动推送                                         │  │
│  │  ├─ 申请状态变化 → 通知用户                                       │  │
│  │  ├─ 简历浏览 → 自动打招呼                                         │  │
│  │  └─ 面试邀请 → 自动确认 + 日程安排                               │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              Workflow State Machine (工作流状态机)                 │  │
│  │                                                                     │  │
│  │  IDLE → SEARCHING → MATCHING → APPLIED → HR_COMMUNICATION          │  │
│  │    ↑        ↓           ↓          ↓              ↓                │  │
│  │    └────────────────────────────────────────── COMPLETED           │  │
│  │                                                     ↓                │  │
│  │                                                  ARCHIVED            │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              Shared Services & Infrastructure                      │  │
│  │                                                                     │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐ │  │
│  │  │ Message Queue    │  │ State Store      │  │ Event Bus       │ │  │
│  │  │ (Redis/RabbitMQ) │  │ (MongoDB/Redis)  │  │ (Redis Pub/Sub) │ │  │
│  │  └──────────────────┘  └──────────────────┘  └─────────────────┘ │  │
│  │                                                                     │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐ │  │
│  │  │ Tool Registry    │  │ Memory Store     │  │ Vector Database │ │  │
│  │  │ (Tool Mgmt)      │  │ (Session Data)   │  │ (RAG)           │ │  │
│  │  └──────────────────┘  └──────────────────┘  └─────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              External Services & Integrations                            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌───────────────┐ │
│  │ Multimodal   │ │ Calendar API │ │ Job Board    │ │ Email/SMS     │ │
│  │ LLM          │ │ (Google/     │ │ APIs         │ │ Notification  │ │
│  │ (GPT-4V/     │ │  Apple)      │ │              │ │               │ │
│  │  Qwen-VL)    │ │              │ │              │ │               │ │
│  └──────────────┘ └──────────────┘ └──────────────┘ └───────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 核心设计详解 (Core Design Details)

### 1. 任务编排与智能体协作 (Task Orchestration & Agent Collaboration)

#### 1.1 Task Orchestrator Agent 详细设计

**核心功能模块：**

```python
class TaskOrchestratorAgent:
    """
    任务编排智能体 - 系统的大脑和指挥中心

    职责：
    1. 请求理解与意图识别
    2. 任务分解与规划
    3. 智能体调度与资源分配
    4. 状态机管理
    5. 结果聚合与响应生成
    """

    def __init__(self):
        self.job_search_pool = AgentPool(size=10, agent_type="job_search")
        self.resume_processor_pool = AgentPool(size=20, agent_type="resume_processor")
        self.hr_comm_pool = AgentPool(size=5, agent_type="hr_communication")
        self.state_manager = WorkflowStateManager()
        self.event_bus = EventBus()

    async def process_request(self, user_request: UserRequest) -> Response:
        """处理用户请求的主入口"""

        # Step 1: 理解用户意图
        intent = await self.understand_intent(user_request)

        # Step 2: 创建任务上下文
        task_context = self.create_task_context(user_request, intent)

        # Step 3: 任务分解与规划
        execution_plan = await self.plan_execution(task_context)

        # Step 4: 执行任务编排
        result = await self.execute_plan(execution_plan, task_context)

        # Step 5: 结果聚合与响应
        return await self.generate_response(result, task_context)
```

**任务分类与编排策略：**

| 任务类型 | 复杂度 | 预期时延 | 编排模式 | 智能体配置 |
|---------|--------|---------|---------|-----------|
| 简单查询 | 低 | <2s | 单智能体直接处理 | 1个 Job Search Agent |
| 职位搜索 | 低-中 | 2-5s | 单智能体 + 工具调用 | 1个 Job Search Agent |
| 单简历分析 | 中 | 3-5s | 单智能体处理 | 1个 Resume Processor |
| 批量简历匹配 | **高** | 5-15s | **Coordinator + Worker Pool** | 1个协调器 + N个并行Worker |
| 完整求职流程 | 极高 | 30-60s | **状态机驱动的多阶段流程** | 跨多个Agent Pool |

**批量处理示例：50份简历匹配**

```python
async def batch_resume_matching(
    self,
    task_context: TaskContext,
    job_requirements: dict,
    resume_ids: list[str]
) -> list[MatchResult]:
    """
    批量简历匹配 - 使用 Coordinator + Worker Pool 模式

    性能：50份简历，3-5秒完成（vs 单Agent需要100-250秒）
    """

    # 分批策略：每批10份简历
    batch_size = 10
    batches = [resume_ids[i:i+batch_size]
               for i in range(0, len(resume_ids), batch_size)]

    all_results = []

    for batch in batches:
        # 并行处理一批简历
        tasks = []
        for resume_id in batch:
            # 从池中获取一个 Resume Processor Agent
            agent = await self.resume_processor_pool.acquire()
            task = self.process_single_resume(
                agent,
                resume_id,
                job_requirements,
                task_context
            )
            tasks.append(task)

        # 等待这一批完成
        batch_results = await asyncio.gather(*tasks)
        all_results.extend(batch_results)

        # 归还 Agents 到池
        for agent in agents_used:
            await self.resume_processor_pool.release(agent)

    # 聚合和排序结果
    ranked_results = self.rank_candidates(all_results, job_requirements)

    return ranked_results
```

#### 1.2 各智能体的主要工作内容

**Job Search Agent Pool (职位搜索智能体池)**

```python
主要功能：
├─ 职位搜索与推荐
│  ├─ 根据用户偏好搜索职位
│  ├─ 使用 RAG 增强搜索结果
│  └─ 智能过滤和排序
├─ 职位详情分析
│  ├─ 提取职位要求
│  ├─ 分析公司文化
│  └─ 薪资水平评估
├─ 简历-职位初步匹配
│  ├─ 技能匹配度计算
│  ├─ 经验匹配度评估
│  └─ 生成匹配分数
└─ 职位申请准备
   ├─ 生成求职信
   ├─ 简历定制建议
   └─ 面试准备材料

工具调用：
- search_jobs()
- get_job_details()
- analyze_job_requirements()
- calculate_match_score()
- generate_cover_letter()

输出：
- 匹配的职位列表
- 详细的匹配分析报告
- 申请建议
- 状态转换信号 → MATCHING 或 APPLIED
```

**Resume Processor Agent Pool (简历处理智能体池)**

```python
主要功能：
├─ 简历解析与结构化
│  ├─ 多格式支持 (PDF/DOCX/Image)
│  ├─ OCR 文字识别
│  ├─ 信息提取 (姓名/技能/经验/教育)
│  └─ 结构化存储
├─ 简历分析与评分
│  ├─ 完整性评估
│  ├─ 关键词密度分析
│  ├─ 格式规范检查
│  └─ ATS 友好度评估
├─ 简历优化建议
│  ├─ 针对特定职位优化
│  ├─ 关键词建议
│  ├─ 成就量化建议
│  └─ 格式改进建议
└─ 批量简历对比
   ├─ 多简历并行处理
   ├─ 候选人排序
   └─ 优劣势分析

工具调用：
- parse_resume()
- extract_structured_data()
- analyze_resume()
- optimize_resume_section()
- calculate_ats_score()

输出：
- 结构化的简历数据
- 简历分析报告
- 优化建议
- 匹配评分
```

**HR Communication Agent Pool (HR沟通智能体池)**

```python
主要功能：
├─ 消息起草与发送
│  ├─ 初次联系消息
│  ├─ 跟进提醒
│  ├─ 面试确认
│  └─ 感谢信
├─ 沟通历史管理
│  ├─ 记录所有往来
│  ├─ 关键信息提取
│  ├─ 情感分析
│  └─ 响应时效监控
├─ 面试协调
│  ├─ 日程安排
│  ├─ 会议室/视频会议预定
│  ├─ 提醒通知
│  └─ 重新安排
└─ 自动化跟进
   ├─ 定期状态查询
   ├─ 无回复提醒
   ├─ Offer 谈判辅助
   └─ 入职准备

工具调用：
- send_hr_message()
- schedule_interview()
- create_calendar_event()
- track_communication_status()
- generate_follow_up_message()

输出：
- 已发送的消息记录
- 面试安排确认
- 沟通状态更新
- 状态转换信号 → COMPLETED 或需要用户介入
```

### 2. Agent 状态边界与平滑过渡机制 (State Boundaries & Smooth Transitions)

#### 2.1 工作流状态机详细设计

```python
class WorkflowState(Enum):
    """工作流状态定义"""
    IDLE = "idle"                           # 空闲状态
    SEARCHING = "searching"                 # 职位搜索中
    MATCHING = "matching"                   # 简历匹配中
    APPLIED = "applied"                     # 已申请职位
    HR_COMMUNICATION = "hr_communication"   # HR沟通中
    INTERVIEW_SCHEDULED = "interview_scheduled"  # 面试已安排
    COMPLETED = "completed"                 # 流程完成
    ARCHIVED = "archived"                   # 已归档
    FAILED = "failed"                       # 流程失败

class StateTransition:
    """状态转换规则"""

    # 状态转换映射表
    TRANSITIONS = {
        WorkflowState.IDLE: [WorkflowState.SEARCHING],
        WorkflowState.SEARCHING: [
            WorkflowState.MATCHING,      # 找到职位 → 开始匹配
            WorkflowState.IDLE,          # 未找到 → 返回空闲
        ],
        WorkflowState.MATCHING: [
            WorkflowState.APPLIED,       # 匹配成功 → 提交申请
            WorkflowState.SEARCHING,     # 匹配不佳 → 继续搜索
        ],
        WorkflowState.APPLIED: [
            WorkflowState.HR_COMMUNICATION,  # 申请后 → 自动打招呼
            WorkflowState.COMPLETED,         # 申请被拒 → 结束
        ],
        WorkflowState.HR_COMMUNICATION: [
            WorkflowState.INTERVIEW_SCHEDULED,  # 获得面试 → 安排面试
            WorkflowState.COMPLETED,            # 沟通结束 → 完成
            WorkflowState.APPLIED,              # 需要补充材料 → 回到申请
        ],
        WorkflowState.INTERVIEW_SCHEDULED: [
            WorkflowState.COMPLETED,     # 面试完成 → 结束
        ],
        WorkflowState.COMPLETED: [
            WorkflowState.ARCHIVED,      # 归档
            WorkflowState.IDLE,          # 重新开始
        ],
    }

    @classmethod
    def can_transition(cls, from_state: WorkflowState,
                      to_state: WorkflowState) -> bool:
        """检查状态转换是否合法"""
        return to_state in cls.TRANSITIONS.get(from_state, [])
```

#### 2.2 关键状态转换：Job Search → HR Communication

**场景：用户申请职位后，自动触发 HR 沟通**

```python
async def handle_application_submitted(
    self,
    task_context: TaskContext,
    application_result: ApplicationResult
):
    """
    处理申请提交成功事件

    关键点：
    1. 检查状态转换合法性
    2. 数据传递与上下文继承
    3. 触发 HR Communication Agent
    4. 设置自动化跟进任务
    """

    # Step 1: 验证状态转换
    current_state = await self.state_manager.get_state(task_context.task_id)
    if not StateTransition.can_transition(
        current_state,
        WorkflowState.HR_COMMUNICATION
    ):
        logger.error(f"Invalid state transition: {current_state} → HR_COMMUNICATION")
        return

    # Step 2: 构建 HR Communication 上下文
    hr_context = HRCommunicationContext(
        task_id=task_context.task_id,
        user_id=task_context.user_id,
        job_id=application_result.job_id,
        company_name=application_result.company_name,
        hr_contact=application_result.hr_contact,
        application_date=datetime.now(),
        # 继承之前的上下文
        user_profile=task_context.user_profile,
        resume_data=task_context.resume_data,
        cover_letter=application_result.cover_letter,
    )

    # Step 3: 更新状态
    await self.state_manager.transition_to(
        task_id=task_context.task_id,
        new_state=WorkflowState.HR_COMMUNICATION,
        metadata={
            "trigger": "application_submitted",
            "application_id": application_result.application_id,
            "transition_time": datetime.now().isoformat(),
        }
    )

    # Step 4: 发送状态转换事件
    await self.event_bus.publish(
        event=StateTransitionEvent(
            task_id=task_context.task_id,
            from_state=current_state,
            to_state=WorkflowState.HR_COMMUNICATION,
            context=hr_context,
        )
    )

    # Step 5: 触发 HR Communication Agent（延迟24小时）
    await self.schedule_hr_greeting(hr_context, delay_hours=24)

    # Step 6: 设置定期跟进任务
    await self.heartbeat_service.register_task(
        task_id=f"hr_followup_{task_context.task_id}",
        task_type="hr_communication_followup",
        schedule="every 3 days",
        max_attempts=5,
        context=hr_context,
    )

    logger.info(
        f"State transition completed: {current_state} → HR_COMMUNICATION"
        f" for task {task_context.task_id}"
    )

async def schedule_hr_greeting(
    self,
    hr_context: HRCommunicationContext,
    delay_hours: int = 24
):
    """
    安排 HR 打招呼任务

    策略：
    - 申请提交后 24 小时自动发送初次联系
    - 使用礼貌、专业的语气
    - 表达对职位的兴趣
    - 附上关键信息
    """

    # 从 HR Communication Agent Pool 获取智能体
    hr_agent = await self.hr_comm_pool.acquire()

    try:
        # 生成初次联系消息
        greeting_message = await hr_agent.generate_greeting_message(
            company_name=hr_context.company_name,
            job_title=hr_context.job_title,
            user_name=hr_context.user_profile.name,
            highlights=hr_context.resume_data.key_achievements[:3],
        )

        # 延迟发送（24小时后）
        await self.scheduler.schedule_task(
            task_name=f"hr_greeting_{hr_context.task_id}",
            execute_at=datetime.now() + timedelta(hours=delay_hours),
            task_func=hr_agent.send_message,
            task_args={
                "recipient": hr_context.hr_contact,
                "subject": f"Following up on {hr_context.job_title} Application",
                "message": greeting_message,
                "context": hr_context,
            }
        )

        logger.info(
            f"HR greeting scheduled for {delay_hours} hours later "
            f"(task: {hr_context.task_id})"
        )

    finally:
        # 归还智能体到池
        await self.hr_comm_pool.release(hr_agent)
```

**状态转换的数据流：**

```
┌─────────────────────────────────────────────────────────────┐
│  Job Search Agent (完成申请)                                │
│                                                              │
│  输出数据包：                                                │
│  ├─ application_id                                          │
│  ├─ job_details (公司、职位、HR联系方式)                    │
│  ├─ application_materials (简历、求职信)                    │
│  └─ application_status: "submitted"                         │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│  Task Orchestrator (状态转换控制器)                         │
│                                                              │
│  1. 验证转换合法性 ✓                                        │
│  2. 构建 HR Communication Context                           │
│     - 继承 Job Search 上下文                                │
│     - 添加 HR 特定信息                                      │
│  3. 更新 State Store                                        │
│  4. 发布状态转换事件                                        │
│  5. 触发 HR Agent 初始化                                    │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│  HR Communication Agent (开始工作)                           │
│                                                              │
│  接收数据：                                                  │
│  ├─ hr_context (完整上下文)                                 │
│  ├─ user_profile (用户档案)                                 │
│  ├─ job_details (职位详情)                                  │
│  └─ application_materials (申请材料)                        │
│                                                              │
│  执行任务：                                                  │
│  1. 24小时后发送初次打招呼                                  │
│  2. 每3天检查申请状态                                       │
│  3. 收到回复 → 及时响应                                     │
│  4. 7天无回复 → 发送礼貌跟进                                │
└─────────────────────────────────────────────────────────────┘
```

### 3. 心跳机制与自动化任务 (Heartbeat Mechanism & Autonomous Tasks)

#### 3.1 Heartbeat Service 设计

**灵感来源：OpenClaw 的心跳机制**

OpenClaw 使用心跳机制让 Agent 定期"苏醒"并执行预定任务，无需外部触发。我们采用类似设计：

```python
class HeartbeatService:
    """
    心跳服务 - 让智能体自主运行

    核心理念：
    1. 定期任务：按固定时间间隔执行（cron-style）
    2. 事件驱动：基于特定事件触发
    3. 条件检查：满足条件时自动执行
    4. 智能调度：根据优先级和负载动态调整
    """

    def __init__(self):
        self.scheduler = AsyncIOScheduler()  # APScheduler
        self.event_bus = EventBus()
        self.task_registry = {}
        self.agent_pools = {}

    async def start(self):
        """启动心跳服务"""
        self.scheduler.start()
        await self.register_default_tasks()
        logger.info("Heartbeat service started")

    async def register_task(
        self,
        task_id: str,
        task_type: str,
        schedule: str | dict,  # "every 1 day" 或 cron 表达式
        agent_type: str,
        task_handler: Callable,
        context: dict,
        condition: Callable = None,  # 可选的执行条件
        max_attempts: int = None,
        priority: int = 5,
    ):
        """
        注册一个心跳任务

        Args:
            task_id: 任务唯一标识
            task_type: 任务类型（用于分类和监控）
            schedule: 调度规则
                - "every 1 day" (每天)
                - "every 3 hours" (每3小时)
                - "0 9 * * *" (每天早上9点 - cron)
            agent_type: 需要使用的智能体类型
            task_handler: 任务执行函数
            context: 任务上下文
            condition: 可选的执行条件函数
            max_attempts: 最大执行次数
            priority: 优先级 (1-10)
        """

        task_config = HeartbeatTask(
            task_id=task_id,
            task_type=task_type,
            schedule=schedule,
            agent_type=agent_type,
            handler=task_handler,
            context=context,
            condition=condition,
            max_attempts=max_attempts,
            priority=priority,
            created_at=datetime.now(),
            attempts=0,
        )

        self.task_registry[task_id] = task_config

        # 根据调度规则添加定时任务
        if isinstance(schedule, str) and schedule.startswith("every"):
            # 解析 "every X hours/days" 格式
            interval = self._parse_interval(schedule)
            self.scheduler.add_job(
                func=self._execute_heartbeat_task,
                trigger='interval',
                **interval,
                args=[task_id],
                id=task_id,
            )
        else:
            # Cron 表达式
            self.scheduler.add_job(
                func=self._execute_heartbeat_task,
                trigger='cron',
                **self._parse_cron(schedule),
                args=[task_id],
                id=task_id,
            )

        logger.info(f"Heartbeat task registered: {task_id} ({schedule})")

    async def _execute_heartbeat_task(self, task_id: str):
        """执行心跳任务"""

        task_config = self.task_registry.get(task_id)
        if not task_config:
            logger.warning(f"Task not found: {task_id}")
            return

        # 检查是否达到最大执行次数
        if (task_config.max_attempts and
            task_config.attempts >= task_config.max_attempts):
            logger.info(f"Task {task_id} reached max attempts, removing")
            await self.unregister_task(task_id)
            return

        # 检查执行条件
        if task_config.condition and not await task_config.condition(task_config.context):
            logger.debug(f"Task {task_id} condition not met, skipping")
            return

        # 获取对应的智能体
        agent_pool = self.agent_pools.get(task_config.agent_type)
        if not agent_pool:
            logger.error(f"Agent pool not found: {task_config.agent_type}")
            return

        agent = await agent_pool.acquire()

        try:
            logger.info(f"Executing heartbeat task: {task_id} (attempt {task_config.attempts + 1})")

            # 执行任务
            result = await task_config.handler(
                agent=agent,
                context=task_config.context
            )

            # 更新执行次数
            task_config.attempts += 1
            task_config.last_executed = datetime.now()
            task_config.last_result = result

            # 发布任务完成事件
            await self.event_bus.publish(
                HeartbeatTaskCompletedEvent(
                    task_id=task_id,
                    task_type=task_config.task_type,
                    result=result,
                    executed_at=datetime.now(),
                )
            )

            logger.info(f"Heartbeat task completed: {task_id}")

        except Exception as e:
            logger.error(f"Heartbeat task failed: {task_id}, error: {e}")
            task_config.failures += 1

            # 发布任务失败事件
            await self.event_bus.publish(
                HeartbeatTaskFailedEvent(
                    task_id=task_id,
                    error=str(e),
                    executed_at=datetime.now(),
                )
            )

        finally:
            # 归还智能体
            await agent_pool.release(agent)
```

#### 3.2 预定义心跳任务

**每日职位推荐（Daily Job Recommendations）**

```python
async def daily_job_recommendations_handler(
    agent: JobSearchAgent,
    context: dict
) -> dict:
    """
    每日职位推荐任务

    执行时间：每天早上 9:00
    智能体：Job Search Agent
    """
    user_id = context["user_id"]
    user_preferences = context["user_preferences"]

    # 搜索新发布的职位
    new_jobs = await agent.search_jobs(
        keywords=user_preferences.get("keywords"),
        location=user_preferences.get("location"),
        posted_since="yesterday",
        limit=10,
    )

    if not new_jobs:
        logger.info(f"No new jobs found for user {user_id}")
        return {"status": "no_new_jobs"}

    # 根据用户简历计算匹配度
    user_resume = await get_user_resume(user_id)
    matched_jobs = await agent.match_jobs(new_jobs, user_resume)

    # 过滤出高匹配度的职位（>80%）
    top_matches = [job for job in matched_jobs if job.match_score > 0.8]

    if top_matches:
        # 发送推送通知
        await send_notification(
            user_id=user_id,
            title=f"🎯 发现 {len(top_matches)} 个高匹配职位！",
            message=f"根据您的简历，我们找到了 {len(top_matches)} 个匹配度超过 80% 的新职位",
            data={"jobs": [job.to_dict() for job in top_matches]},
        )

        logger.info(f"Sent {len(top_matches)} job recommendations to user {user_id}")

    return {
        "status": "success",
        "jobs_found": len(new_jobs),
        "top_matches": len(top_matches),
    }

# 注册任务
await heartbeat_service.register_task(
    task_id=f"daily_recommendations_{user_id}",
    task_type="job_recommendations",
    schedule="0 9 * * *",  # 每天早上9点
    agent_type="job_search",
    task_handler=daily_job_recommendations_handler,
    context={
        "user_id": user_id,
        "user_preferences": user_preferences,
    },
)
```

**申请状态跟进（Application Follow-ups）**

```python
async def application_followup_handler(
    agent: HRCommunicationAgent,
    context: dict
) -> dict:
    """
    申请状态跟进任务

    执行时间：每3天检查一次
    智能体：HR Communication Agent
    最大执行次数：5次（15天后停止）
    """
    application_id = context["application_id"]
    task_context = context["task_context"]

    # 检查申请状态
    application_status = await get_application_status(application_id)

    if application_status.state == "completed":
        logger.info(f"Application {application_id} completed, stopping followup")
        return {"status": "completed", "action": "stop_followup"}

    # 检查最近一次沟通时间
    last_communication = await get_last_communication(
        application_id=application_id
    )

    days_since_last_contact = (
        datetime.now() - last_communication.timestamp
    ).days

    # 如果7天无回复，发送礼貌跟进
    if days_since_last_contact >= 7:
        followup_message = await agent.generate_followup_message(
            application=application_status,
            days_elapsed=days_since_last_contact,
            tone="polite_and_professional",
        )

        await agent.send_message(
            recipient=application_status.hr_contact,
            subject=f"Following up - {application_status.job_title} Application",
            message=followup_message,
        )

        logger.info(
            f"Sent followup for application {application_id} "
            f"({days_since_last_contact} days since last contact)"
        )

        return {
            "status": "followup_sent",
            "days_elapsed": days_since_last_contact,
        }

    return {"status": "no_action_needed"}

# 注册任务（申请提交后自动注册）
await heartbeat_service.register_task(
    task_id=f"followup_{application_id}",
    task_type="application_followup",
    schedule="every 3 days",
    agent_type="hr_communication",
    task_handler=application_followup_handler,
    context={
        "application_id": application_id,
        "task_context": task_context,
    },
    max_attempts=5,  # 最多跟进5次（15天）
)
```

**面试提醒（Interview Reminders）**

```python
async def interview_reminder_handler(
    agent: HRCommunicationAgent,
    context: dict
) -> dict:
    """
    面试提醒任务

    执行时间：
    - 面试前24小时
    - 面试前1小时
    智能体：HR Communication Agent
    """
    interview_id = context["interview_id"]
    reminder_type = context["reminder_type"]  # "24h" or "1h"

    interview = await get_interview_details(interview_id)

    if reminder_type == "24h":
        message = f"""
        明天您有一场面试！

        📅 时间：{interview.datetime}
        🏢 公司：{interview.company_name}
        💼 职位：{interview.job_title}
        📍 地点：{interview.location or interview.meeting_link}

        准备建议：
        1. 复习职位描述和公司背景
        2. 准备常见面试问题的回答
        3. 准备2-3个问题问面试官
        4. 检查设备和网络（如果是远程面试）

        祝您面试顺利！🎉
        """
    else:  # 1h
        message = f"""
        ⏰ 提醒：1小时后您有面试！

        📅 时间：{interview.datetime}
        🏢 公司：{interview.company_name}
        💼 职位：{interview.job_title}
        📍 地点：{interview.location or interview.meeting_link}

        请确保已准备就绪！
        """

    await send_notification(
        user_id=interview.user_id,
        title=f"面试提醒 - {interview.company_name}",
        message=message,
        priority="high",
    )

    return {"status": "reminder_sent", "type": reminder_type}

# 面试安排时自动注册提醒任务
async def schedule_interview_reminders(interview: Interview):
    """安排面试提醒"""

    # 24小时提醒
    reminder_24h_time = interview.datetime - timedelta(hours=24)
    await heartbeat_service.register_task(
        task_id=f"interview_reminder_24h_{interview.id}",
        task_type="interview_reminder",
        schedule=reminder_24h_time.strftime("%M %H %d %m *"),  # Cron格式
        agent_type="hr_communication",
        task_handler=interview_reminder_handler,
        context={
            "interview_id": interview.id,
            "reminder_type": "24h",
        },
        max_attempts=1,
    )

    # 1小时提醒
    reminder_1h_time = interview.datetime - timedelta(hours=1)
    await heartbeat_service.register_task(
        task_id=f"interview_reminder_1h_{interview.id}",
        task_type="interview_reminder",
        schedule=reminder_1h_time.strftime("%M %H %d %m *"),
        agent_type="hr_communication",
        task_handler=interview_reminder_handler,
        context={
            "interview_id": interview.id,
            "reminder_type": "1h",
        },
        max_attempts=1,
    )
```

**事件驱动的自动化任务**

```python
class EventDrivenAutomation:
    """事件驱动的自动化任务"""

    def __init__(self, event_bus: EventBus, agent_pools: dict):
        self.event_bus = event_bus
        self.agent_pools = agent_pools
        self.register_event_handlers()

    def register_event_handlers(self):
        """注册事件处理器"""

        # 简历被浏览 → 自动打招呼
        self.event_bus.subscribe(
            event_type="resume_viewed_by_hr",
            handler=self.handle_resume_viewed,
        )

        # 收到面试邀请 → 自动确认并安排
        self.event_bus.subscribe(
            event_type="interview_invitation_received",
            handler=self.handle_interview_invitation,
        )

        # 申请状态变化 → 通知用户
        self.event_bus.subscribe(
            event_type="application_status_changed",
            handler=self.handle_application_status_change,
        )

        # 新职位匹配 → 自动推送
        self.event_bus.subscribe(
            event_type="new_job_matched",
            handler=self.handle_new_job_match,
        )

    async def handle_resume_viewed(self, event: ResumeViewedEvent):
        """处理简历被浏览事件 - 自动打招呼"""

        hr_agent = await self.agent_pools["hr_communication"].acquire()

        try:
            # 生成友好的打招呼消息
            greeting = await hr_agent.generate_greeting_after_view(
                company_name=event.company_name,
                job_title=event.job_title,
                user_profile=event.user_profile,
            )

            # 发送消息
            await hr_agent.send_message(
                recipient=event.hr_contact,
                subject=f"Thank you for viewing my profile",
                message=greeting,
            )

            logger.info(
                f"Sent auto-greeting after resume view: "
                f"{event.user_id} → {event.company_name}"
            )

        finally:
            await self.agent_pools["hr_communication"].release(hr_agent)

    async def handle_interview_invitation(
        self,
        event: InterviewInvitationEvent
    ):
        """处理面试邀请 - 自动确认并安排"""

        hr_agent = await self.agent_pools["hr_communication"].acquire()

        try:
            # 检查用户日历可用性
            available_slots = await check_calendar_availability(
                user_id=event.user_id,
                proposed_times=event.proposed_times,
            )

            if available_slots:
                # 自动确认第一个可用时间
                selected_time = available_slots[0]

                # 发送确认消息
                confirmation = await hr_agent.generate_interview_confirmation(
                    selected_time=selected_time,
                    interview_details=event.interview_details,
                )

                await hr_agent.send_message(
                    recipient=event.hr_contact,
                    subject="Interview Confirmation",
                    message=confirmation,
                )

                # 添加到日历
                await create_calendar_event(
                    user_id=event.user_id,
                    title=f"Interview - {event.company_name}",
                    datetime=selected_time,
                    duration_minutes=event.interview_duration or 60,
                    location=event.meeting_link,
                )

                # 安排提醒任务
                await schedule_interview_reminders(event.interview_id)

                logger.info(
                    f"Auto-confirmed interview: {event.user_id} → "
                    f"{event.company_name} at {selected_time}"
                )
            else:
                # 无可用时间，请求用户介入
                await notify_user_intervention_needed(
                    user_id=event.user_id,
                    reason="interview_scheduling_conflict",
                    details=event.to_dict(),
                )

        finally:
            await self.agent_pools["hr_communication"].release(hr_agent)
```

#### 3.3 心跳机制的优势

**与传统定时任务的对比：**

| 特性 | 传统 Cron/定时任务 | 心跳机制 + 智能体 |
|-----|------------------|------------------|
| 灵活性 | 固定时间执行 | 时间 + 条件 + 事件驱动 |
| 智能性 | 执行固定脚本 | AI智能体自主判断和处理 |
| 上下文理解 | 无 | 完整的任务上下文和历史 |
| 个性化 | 所有用户相同逻辑 | 根据用户情况个性化处理 |
| 错误处理 | 简单重试或失败 | 智能分析并调整策略 |
| 可扩展性 | 手动添加新任务 | 动态注册和管理 |

**心跳机制带来的能力：**

1. **主动性**：智能体不再被动等待用户请求，可以主动发现机会并采取行动
2. **持续性**：即使用户不在线，智能体持续工作（搜索职位、跟进申请、安排面试）
3. **智能性**：根据上下文和历史做出判断（什么时候跟进、如何表达、是否需要用户介入）
4. **个性化**：每个用户有独立的心跳任务，根据个人情况定制
5. **可靠性**：自动重试、错误恢复、状态持久化

---

## 常见问题解答 (FAQ)

### 1. 多用户支持 (Multi-user Support)

**问题：** 服务部署在服务器上，用户通过移动app来使用该agent系统，如何完美支持多用户使用？

**解决方案：**

- **会话管理 (Session Management)：** 为每个用户分配唯一的 `user_id` 和 `session_id`，独立跟踪用户的对话状态
- **数据隔离：** 使用数据库后端（SQLite/MongoDB）为每个用户存储独立的对话历史和上下文
- **Agent 实例管理：**
  - 方案1：为每个会话创建独立的 Agent 实例（适用于低并发）
  - 方案2：使用 Agent 池 + 基于会话的内存隔离（推荐用于高并发）

**实现示例：**

**方案1：为每个会话创建独立 Agent（适用于低并发）**

```python
from agentscope.agent import ReActAgent
from agentscope.memory import InMemoryMemory
from agentscope.tool import Toolkit

# 为每个用户请求创建独立会话
def create_user_session(user_id: str, conversation_type: str):
    session_id = f"{user_id}_{conversation_type}_{timestamp}"

    # 使用数据库支持的内存，确保会话持久化
    memory = SQLiteMemory(
        session_id=session_id,
        db_path=f"./data/sessions/{user_id}.db"
    )

    toolkit = create_toolkit_for_type(conversation_type)

    agent = ReActAgent(
        name=f"{conversation_type}_agent",
        memory=memory,
        toolkit=toolkit,
        model=model_config,
    )

    return agent, session_id
```

**方案2：Agent 池 + 会话内存隔离（推荐用于高并发）**

```python
import asyncio
from typing import Dict, List
from queue import Queue
from threading import Lock
from agentscope.agent import ReActAgent
from agentscope.memory import InMemoryMemory
from agentscope.tool import Toolkit
from agentscope.message import Msg

class AgentPool:
    """Agent 池管理器，用于高并发场景下的 Agent 复用"""

    def __init__(self, pool_size: int = 10):
        self.pool_size = pool_size
        self.agent_pools: Dict[str, Queue] = {}  # agent_type -> agent queue
        self.session_memories: Dict[str, InMemoryMemory] = {}  # session_id -> memory
        self.lock = Lock()
        self.model_config = self._create_model_config()

    def _create_model_config(self):
        """创建模型配置"""
        from agentscope.model import DashScopeChatModel
        return DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
            stream=True,
        )

    def _create_agent(self, agent_type: str) -> ReActAgent:
        """创建一个新的 Agent 实例（不绑定特定会话）"""
        toolkit = self._create_toolkit_for_type(agent_type)

        agent = ReActAgent(
            name=f"{agent_type}_agent",
            sys_prompt=self._get_system_prompt(agent_type),
            model=self.model_config,
            memory=None,  # 内存将在运行时动态设置
            toolkit=toolkit,
        )

        return agent

    def _get_system_prompt(self, agent_type: str) -> str:
        """获取不同类型 Agent 的系统提示词"""
        prompts = {
            "job_search": """You are a professional career advisor helping users
            find jobs, optimize resumes, and prepare for interviews.""",
            "hr_communication": """You are a professional communication assistant
            helping users communicate with HR representatives.""",
        }
        return prompts.get(agent_type, "You are a helpful assistant.")

    def _create_toolkit_for_type(self, agent_type: str) -> Toolkit:
        """为不同类型的 Agent 创建工具包"""
        # 根据 agent_type 注册相应的工具
        toolkit = Toolkit()
        # ... 注册工具的逻辑
        return toolkit

    def initialize_pool(self, agent_type: str):
        """初始化指定类型的 Agent 池"""
        if agent_type not in self.agent_pools:
            self.agent_pools[agent_type] = Queue()
            # 预创建指定数量的 Agent 实例
            for _ in range(self.pool_size):
                agent = self._create_agent(agent_type)
                self.agent_pools[agent_type].put(agent)

    def get_or_create_session_memory(self, session_id: str) -> InMemoryMemory:
        """获取或创建会话内存"""
        with self.lock:
            if session_id not in self.session_memories:
                # 创建基于数据库的持久化内存
                self.session_memories[session_id] = SQLiteMemory(
                    session_id=session_id,
                    db_path=f"./data/sessions/{session_id}.db"
                )
            return self.session_memories[session_id]

    def acquire_agent(self, agent_type: str, session_id: str) -> ReActAgent:
        """从池中获取一个 Agent 并绑定会话内存"""
        # 确保池已初始化
        if agent_type not in self.agent_pools:
            self.initialize_pool(agent_type)

        # 从池中获取 Agent（如果池为空会阻塞等待）
        agent = self.agent_pools[agent_type].get()

        # 绑定会话内存
        session_memory = self.get_or_create_session_memory(session_id)
        agent.memory = session_memory

        return agent

    def release_agent(self, agent: ReActAgent, agent_type: str):
        """将 Agent 归还到池中"""
        # 解除内存绑定，防止内存泄漏
        agent.memory = None

        # 归还到池中
        self.agent_pools[agent_type].put(agent)

    async def handle_user_request(
        self,
        user_id: str,
        session_id: str,
        agent_type: str,
        message: str
    ) -> str:
        """处理用户请求（使用 Agent 池）"""
        # 从池中获取 Agent
        agent = self.acquire_agent(agent_type, session_id)

        try:
            # 处理用户消息
            msg = Msg(name="user", content=message, role="user")
            response = await agent(msg)
            result = response.get_text_content()

        finally:
            # 无论成功与否，都要归还 Agent 到池中
            self.release_agent(agent, agent_type)

        return result

    def cleanup_session(self, session_id: str):
        """清理会话内存（当会话结束时调用）"""
        with self.lock:
            if session_id in self.session_memories:
                # 可选：持久化内存到数据库
                memory = self.session_memories[session_id]
                memory.save()  # 假设有 save 方法
                del self.session_memories[session_id]


# 使用示例：高并发场景
async def main():
    # 初始化 Agent 池（只需要初始化一次）
    agent_pool = AgentPool(pool_size=10)

    # 预初始化不同类型的 Agent 池
    agent_pool.initialize_pool("job_search")
    agent_pool.initialize_pool("hr_communication")

    # 模拟多个并发用户请求
    async def process_user_request(user_id: str, message: str):
        session_id = f"session_{user_id}_job_search"
        response = await agent_pool.handle_user_request(
            user_id=user_id,
            session_id=session_id,
            agent_type="job_search",
            message=message
        )
        print(f"User {user_id}: {response}")

    # 并发处理 100 个用户请求
    tasks = [
        process_user_request(f"user_{i}", "帮我找一份Python工程师的工作")
        for i in range(100)
    ]

    # 使用 Agent 池，只需要 10 个 Agent 实例就可以处理 100 个并发请求
    await asyncio.gather(*tasks)

    print("所有请求处理完成！")


if __name__ == "__main__":
    asyncio.run(main())
```

**方案2 的优势：**

1. **资源高效利用：** 10 个 Agent 实例可以处理成百上千的并发会话
2. **快速响应：** 避免了频繁创建/销毁 Agent 的开销
3. **内存隔离：** 每个会话有独立的内存，互不干扰
4. **可扩展性：** 可以根据负载动态调整池大小
5. **故障隔离：** 单个会话的错误不会影响其他会话

**性能对比：**

| 指标 | 方案1（每次创建） | 方案2（Agent池） |
|------|------------------|------------------|
| Agent 初始化时间 | 每次请求 ~500ms | 启动时一次性 |
| 内存占用（100会话） | ~5GB | ~500MB |
| 并发处理能力 | 受限于内存 | 高效复用 |
| 适用场景 | 低并发（<10用户） | 高并发（100+用户） |

### 2. 对话类型区分 (Conversation Separation)

**问题：** 用户自己与大模型的会话如何与HR沟通聊天的会话区分开来，需要给这两种会话单独设置不同的agent吗？

**解决方案：**

**是的，需要使用不同的 Agent** 来处理不同类型的对话：

1. **求职咨询 Agent (Job Search Agent)：**
   - 处理职位推荐、简历优化、面试准备等
   - 系统提示词侧重于专业建议和指导
   - 可以访问所有求职相关工具

2. **HR 沟通 Agent (HR Communication Agent)：**
   - 管理与HR的正式沟通
   - 系统提示词侧重于专业、礼貌的商务沟通
   - 有权访问消息历史和日程安排工具

**实现示例：**

```python
# 求职咨询 Agent
job_search_agent = ReActAgent(
    name="JobSearchAssistant",
    sys_prompt="""You are a professional career advisor helping users
    find jobs, optimize resumes, and prepare for interviews.
    Be supportive, professional, and provide actionable advice.""",
    memory=job_search_memory,
    toolkit=job_search_toolkit,
    model=model,
)

# HR 沟通 Agent
hr_comm_agent = ReActAgent(
    name="HRCommunicator",
    sys_prompt="""You are a professional communication assistant
    helping users communicate with HR representatives.
    Maintain a formal, polite tone and ensure clear communication.""",
    memory=hr_comm_memory,
    toolkit=hr_comm_toolkit,
    model=model,
)

# 根据对话类型路由
async def route_message(user_id: str, conversation_type: str, message: str):
    if conversation_type == "job_search":
        agent = get_or_create_agent(user_id, job_search_agent)
    elif conversation_type == "hr_communication":
        agent = get_or_create_agent(user_id, hr_comm_agent)
    else:
        raise ValueError(f"Unknown conversation type: {conversation_type}")

    response = await agent(Msg("user", message, "user"))
    return response
```

### 3. 日历集成 (Calendar Integration)

**问题：** 日历集成这部分如何应用到安卓和iOS？用shell运行工具可以做到吗？

**解决方案：**

**Shell 工具不适合移动端日历集成。** 推荐方案：

1. **后端 API 集成方式：**
   - 后端服务器集成 Google Calendar API / Apple Calendar API
   - Agent 通过调用后端 API 来创建/查询日历事件
   - 移动端使用原生 Calendar SDK 同步显示

2. **工具实现：**

```python
from agentscope.tool import Toolkit
import requests

async def create_calendar_event(
    user_id: str,
    title: str,
    date: str,
    time: str,
    description: str = ""
) -> dict:
    """Create a calendar event for interview or deadline.

    Args:
        user_id: The user identifier
        title: Event title (e.g., "Interview with TechCorp")
        date: Date in YYYY-MM-DD format
        time: Time in HH:MM format
        description: Additional details

    Returns:
        dict: Calendar event details with event_id
    """
    # Call backend API which handles platform-specific integration
    response = await backend_api_call(
        endpoint="/api/calendar/create",
        data={
            "user_id": user_id,
            "title": title,
            "date": date,
            "time": time,
            "description": description,
        }
    )
    return response.json()

async def query_calendar_events(
    user_id: str,
    start_date: str,
    end_date: str
) -> list:
    """Query calendar events in a date range.

    Args:
        user_id: The user identifier
        start_date: Start date in YYYY-MM-DD format
        end_date: End date in YYYY-MM-DD format

    Returns:
        list: List of calendar events
    """
    response = await backend_api_call(
        endpoint="/api/calendar/query",
        params={
            "user_id": user_id,
            "start_date": start_date,
            "end_date": end_date,
        }
    )
    return response.json()

# 注册到工具包
toolkit = Toolkit()
toolkit.register_tool_function(create_calendar_event)
toolkit.register_tool_function(query_calendar_events)
```

3. **移动端集成：**
   - Android: 使用 Google Calendar API + Calendar Provider
   - iOS: 使用 EventKit framework
   - 后端作为中间层协调不同平台

### 4. 多模态处理 (Multimodal Processing)

**问题：** 多模态处理可以直接用多模态大模型来回复吗？比自己用PIL + transformers要方便？

**解决方案：**

**是的，直接使用多模态 LLM 更加简单方便！**

**优势：**
- 无需自己实现图像预处理和特征提取
- 模型直接理解图像和文本的语义关系
- 更好的上下文理解能力

**实现示例：**

```python
from agentscope.model import OpenAIChatModel
from agentscope.message import Msg
from agentscope.agent import ReActAgent
import base64

# 配置多模态模型
multimodal_model = OpenAIChatModel(
    model_name="gpt-4-vision-preview",
    api_key=os.environ["OPENAI_API_KEY"],
)

# 创建支持多模态的 Agent
resume_analyzer = ReActAgent(
    name="ResumeAnalyzer",
    sys_prompt="""You are an expert resume reviewer.
    Analyze resumes and provide constructive feedback.""",
    model=multimodal_model,
    toolkit=toolkit,
)

# 处理简历图片
async def analyze_resume_image(image_path: str):
    # 读取图片并转换为 base64
    with open(image_path, "rb") as img_file:
        image_data = base64.b64encode(img_file.read()).decode()

    # 创建包含图片的消息
    msg = Msg(
        name="user",
        content=[
            {
                "type": "text",
                "text": "Please analyze this resume and provide feedback on formatting, content, and improvement suggestions."
            },
            {
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/jpeg;base64,{image_data}"
                }
            }
        ],
        role="user"
    )

    # Agent 直接处理多模态输入
    response = await resume_analyzer(msg)
    return response

# 支持的场景
# 1. 简历分析（PDF/图片格式）
# 2. 身份证件验证
# 3. 职位描述截图理解
# 4. 面试穿着建议（用户上传照片）
```

### 5. Browser 工具的必要性 (Browser Tools)

**问题：** 所有的数据可以通过后端接口拿到，就不需要Browser工具了吧？

**解决方案：**

**正确 - 如果有完整的后端 API，不需要 Browser 工具。**

**何时需要 Browser 工具：**
- 需要抓取没有 API 的招聘网站
- 自动填写在线申请表单
- 测试用户界面工作流程

**推荐方案：** 为您的用例创建自定义工具，直接调用后端 API：

```python
from agentscope.tool import Toolkit

async def search_jobs(
    keywords: str,
    location: str = "",
    salary_min: int = 0,
    experience_level: str = ""
) -> list:
    """Search for jobs matching criteria.

    Args:
        keywords: Job title or keywords
        location: Job location
        salary_min: Minimum salary
        experience_level: Required experience level

    Returns:
        list: List of matching job postings
    """
    # 直接调用后端 API，而不是使用 Browser 工具
    response = await backend_api_call(
        endpoint="/api/jobs/search",
        params={
            "keywords": keywords,
            "location": location,
            "salary_min": salary_min,
            "experience_level": experience_level,
        }
    )
    return response.json()["jobs"]

async def apply_to_job(
    user_id: str,
    job_id: str,
    cover_letter: str = ""
) -> dict:
    """Submit job application.

    Args:
        user_id: The user identifier
        job_id: The job posting ID
        cover_letter: Optional cover letter

    Returns:
        dict: Application status
    """
    response = await backend_api_call(
        endpoint="/api/jobs/apply",
        data={
            "user_id": user_id,
            "job_id": job_id,
            "cover_letter": cover_letter,
        }
    )
    return response.json()

# 注册工具
toolkit = Toolkit()
toolkit.register_tool_function(search_jobs)
toolkit.register_tool_function(apply_to_job)
```

**优势：**
- 更快、更可靠
- 易于维护和测试
- 更好的错误处理
- 不受网页结构变化影响

### 6. Agent 通信与任务调度 (Agent Communication & Task Distribution)

**问题：** 每个agent之间如何通信？如何做任务分发和调度？

**解决方案：**

AgentScope 提供多种 Agent 通信和协调模式：

**1. MsgHub - 消息中心模式：**

```python
from agentscope.pipeline import MsgHub, sequential_pipeline
from agentscope.message import Msg

async def collaborative_job_search(user_request: str):
    # 创建多个专业 Agent
    resume_agent = ReActAgent(name="ResumeExpert", ...)
    search_agent = ReActAgent(name="JobMatcher", ...)
    interview_agent = ReActAgent(name="InterviewCoach", ...)

    # 使用 MsgHub 进行协作讨论
    async with MsgHub(
        participants=[resume_agent, search_agent, interview_agent],
        announcement=Msg("system", user_request, "system")
    ) as hub:
        # Agent 们可以相互讨论，协作制定求职策略
        await sequential_pipeline([resume_agent, search_agent, interview_agent])

        # 动态管理参与者
        hr_agent = ReActAgent(name="HRCommunicator", ...)
        hub.add(hr_agent)

        # 广播消息给所有 Agent
        await hub.broadcast(Msg("system", "Finalize the job search plan", "system"))
```

**2. Sequential Pipeline - 顺序执行：**

```python
from agentscope.pipeline import sequential_pipeline

async def job_application_workflow(user_id: str, job_id: str):
    # 按顺序执行任务
    agents = [
        resume_analyzer,      # 1. 分析并优化简历
        job_matcher,          # 2. 匹配职位要求
        application_writer,   # 3. 生成申请材料
        submission_agent,     # 4. 提交申请
    ]

    initial_msg = Msg("system", f"Process job application for job {job_id}", "system")
    result = await sequential_pipeline(agents, initial_msg)
    return result
```

**3. Parallel Execution - 并行执行：**

```python
from agentscope.pipeline import parallel_pipeline

async def comprehensive_job_search(keywords: str):
    # 并行搜索多个来源
    search_agents = [
        linkedin_search_agent,
        indeed_search_agent,
        company_website_agent,
    ]

    # 所有 Agent 同时工作
    results = await parallel_pipeline(search_agents, search_msg)

    # 合并结果
    all_jobs = merge_job_results(results)
    return all_jobs
```

**4. 消息传递 - Direct Message Passing：**

```python
async def agent_to_agent_communication():
    # Agent A 处理用户请求
    msg = Msg("user", "Help me prepare for a technical interview", "user")
    response_a = await research_agent(msg)

    # 将 Agent A 的结果传递给 Agent B
    response_b = await interview_coach(response_a)

    # Agent B 的结果传递给 Agent C
    final_response = await practice_agent(response_b)

    return final_response
```

**5. Task Queue - 任务队列模式（推荐用于生产环境）：**

```python
from celery import Celery
import asyncio

# 使用 Celery 进行后台任务调度
celery_app = Celery('job_agent_tasks', broker='redis://localhost:6379/0')

@celery_app.task
async def process_job_application(user_id: str, job_id: str):
    """后台处理求职申请"""
    agent = get_agent_for_user(user_id)
    result = await agent.process_application(job_id)
    notify_user(user_id, result)
    return result

@celery_app.task
async def daily_job_recommendations(user_id: str):
    """每日推送职位推荐"""
    agent = get_job_matcher_agent(user_id)
    recommendations = await agent.find_matching_jobs()
    send_notification(user_id, recommendations)
    return recommendations

# 调度任务
process_job_application.delay(user_id="user123", job_id="job456")
```

---

### 7. Agent 池 + 会话内存隔离方案的通信模式深度分析

**问题：** 如果采用 Agent 池 + 会话内存隔离（方案2）用于高并发场景，应该选择哪种通信和协调模式？各模式的优缺点是什么？

#### 业务特点分析

Job Agent 系统具有以下特点：
- **会话独立性强**：每个用户会话相互独立
- **任务相对简单**：单个请求通常是独立的求职任务（搜索、简历分析等）
- **并发要求高**：需要同时服务多个用户
- **Agent 池复用**：10个 Agent 实例服务上百个会话

#### 各种通信模式的适用性分析

##### **1. 消息传递模式 (Direct Message Passing)** ⭐⭐⭐⭐⭐ 强烈推荐

**适用场景：**
```python
# 用户请求 → Agent 池获取 → 处理 → 归还池
async def handle_user_request(user_id, session_id, message):
    agent = agent_pool.acquire_agent("job_search", session_id)
    try:
        msg = Msg(name="user", content=message, role="user")
        response = await agent(msg)
        return response.get_text_content()
    finally:
        agent_pool.release_agent(agent, "job_search")
```

**优点：**
- ✅ **最适合 Agent 池架构**：简单的获取-使用-归还模式
- ✅ **低延迟**：无需额外的消息路由开销
- ✅ **易于理解和维护**：代码逻辑清晰直观
- ✅ **资源利用率高**：Agent 快速复用
- ✅ **会话隔离完美**：每次绑定独立的 session_memory

**缺点：**
- ❌ 不支持 Agent 间复杂协作（但本场景不需要）

**推荐理由：**
- 业务主要是 **单 Agent 完成单个任务**（搜索职位、分析简历）
- Agent 池模式天然适合这种"获取-处理-归还"流程

---

##### **2. Sequential Pipeline（顺序管道）** ⭐⭐⭐ 适合部分场景

**适用场景：**
```python
# 适合：求职申请工作流（多步骤顺序执行）
async def job_application_workflow(user_id, job_id):
    agents = [
        resume_analyzer,      # 1. 分析简历
        job_matcher,          # 2. 匹配职位
        application_writer,   # 3. 生成申请材料
        submission_agent,     # 4. 提交申请
    ]
    result = await sequential_pipeline(agents, initial_msg)
```

**优点：**
- ✅ 适合多步骤工作流
- ✅ 保证执行顺序
- ✅ 易于追踪进度

**缺点：**
- ❌ **不适合 Agent 池架构**：需要同时占用多个 Agent，降低复用率
- ❌ **延迟累加**：每步顺序执行
- ❌ **资源占用时间长**：工作流期间 Agent 被持续占用

**与 Agent 池的冲突：**
- Agent 池要求快速复用，但 Pipeline 会长时间占用 Agent
- 多步骤任务应该由**单个 Agent 通过工具调用完成**，而非多个 Agent 串行

**改进建议：**
```python
# 更好的做法：单个 Agent + 工具链
async def handle_complex_workflow(user_id, job_id):
    agent = agent_pool.acquire_agent("job_search", session_id)
    try:
        # Agent 内部通过工具调用完成多步骤
        response = await agent(Msg("user", "帮我申请这个职位", "user"))
        # Agent 内部会自动调用：
        # 1. analyze_resume()
        # 2. optimize_resume_section()
        # 3. generate_cover_letter()
        # 4. submit_job_application()
    finally:
        agent_pool.release_agent(agent, "job_search")
```

---

##### **3. Parallel Pipeline（并行管道）** ⭐⭐ 不推荐用于主流程

**适用场景：**
```python
# 并行搜索多个招聘平台
results = await parallel_pipeline(
    [linkedin_agent, indeed_agent, boss_agent],
    search_msg
)
```

**优点：**
- ✅ 可以并行查询多个数据源

**缺点：**
- ❌ **严重消耗 Agent 池资源**：同时占用多个 Agent
- ❌ **复杂度高**：Agent 池管理困难
- ❌ **实际意义不大**：后端 API 本身就是异步的

**更好的替代方案：**
```python
# 单个 Agent 通过异步工具调用实现并行
async def search_multiple_platforms(keywords):
    agent = agent_pool.acquire_agent("job_search", session_id)
    try:
        # Agent 的工具内部使用 asyncio.gather
        response = await agent(Msg("user", f"搜索 {keywords}", "user"))
        # 工具层实现：
        # results = await asyncio.gather(
        #     search_linkedin(keywords),
        #     search_indeed(keywords),
        #     search_boss(keywords)
        # )
    finally:
        agent_pool.release_agent(agent, "job_search")
```

---

##### **4. MsgHub（消息中心）** ⭐ 不适合此场景

**适用场景：**
```python
# 多 Agent 协作讨论
async with MsgHub(participants=[agent1, agent2, agent3]) as hub:
    await sequential_pipeline([agent1, agent2, agent3])
```

**优点：**
- ✅ 支持复杂的多 Agent 协作

**缺点：**
- ❌ **完全不适合 Agent 池**：需要同时占用多个 Agent 长时间讨论
- ❌ **资源浪费严重**：协作期间所有参与 Agent 都被锁定
- ❌ **与会话隔离冲突**：MsgHub 是跨 Agent 通信，破坏独立性
- ❌ **场景不需要**：Job Agent 不需要多个 Agent 讨论决策

---

##### **5. Task Queue（任务队列）+ 消息传递** ⭐⭐⭐⭐⭐ 生产环境最佳

**适用场景：**
```python
# 使用 Celery/RQ 进行后台任务调度
@celery_app.task
async def process_job_application(user_id, session_id, job_id):
    agent = agent_pool.acquire_agent("job_search", session_id)
    try:
        result = await agent.process_application(job_id)
        notify_user(user_id, result)
    finally:
        agent_pool.release_agent(agent, "job_search")

# 调度任务
process_job_application.delay(user_id, session_id, job_id)
```

**优点：**
- ✅ **与 Agent 池完美结合**：每个任务独立获取和归还 Agent
- ✅ **异步处理**：不阻塞用户请求
- ✅ **负载均衡**：自动分发任务到多个 Worker
- ✅ **容错性强**：任务失败可重试
- ✅ **可扩展**：轻松水平扩展 Worker 数量
- ✅ **监控友好**：可以跟踪任务状态

**缺点：**
- ❌ 增加系统复杂度（但值得）
- ❌ 需要额外的 Redis/RabbitMQ

---

#### 推荐架构：Task Queue + 消息传递 + Agent 池

```python
from celery import Celery
import asyncio
from typing import Dict
from queue import Queue

# 1. Agent 池管理器
class AgentPool:
    def __init__(self, pool_size: int = 10):
        self.pool_size = pool_size
        self.agent_pools: Dict[str, Queue] = {}
        self.session_memories: Dict[str, Memory] = {}

    def acquire_agent(self, agent_type: str, session_id: str):
        """从池中获取 Agent 并绑定会话内存"""
        if agent_type not in self.agent_pools:
            self.initialize_pool(agent_type)

        agent = self.agent_pools[agent_type].get()
        agent.memory = self.get_or_create_session_memory(session_id)
        return agent

    def release_agent(self, agent, agent_type: str):
        """归还 Agent 到池"""
        agent.memory = None  # 解除绑定
        self.agent_pools[agent_type].put(agent)

# 2. Celery 任务队列
celery_app = Celery('job_agent', broker='redis://localhost:6379/0')
agent_pool = AgentPool(pool_size=10)

# 3. 定义异步任务
@celery_app.task
async def handle_user_message(user_id: str, session_id: str,
                              agent_type: str, message: str):
    """处理用户消息（后台任务）"""
    agent = agent_pool.acquire_agent(agent_type, session_id)

    try:
        msg = Msg(name="user", content=message, role="user")
        response = await agent(msg)
        result = response.get_text_content()

        # 通过 WebSocket/SSE 推送结果给用户
        await notify_user(user_id, result)

        return result
    finally:
        agent_pool.release_agent(agent, agent_type)

@celery_app.task
async def daily_job_recommendations(user_id: str):
    """每日职位推荐（定时任务）"""
    session_id = f"daily_rec_{user_id}"
    agent = agent_pool.acquire_agent("job_search", session_id)

    try:
        recommendations = await agent(Msg(
            "system",
            "根据用户偏好推荐今日职位",
            "system"
        ))
        await send_push_notification(user_id, recommendations)
    finally:
        agent_pool.release_agent(agent, "job_search")

# 4. API 层（FastAPI）
from fastapi import FastAPI, WebSocket
app = FastAPI()

@app.post("/api/v1/chat/message")
async def send_message(user_id: str, session_id: str, message: str):
    """同步 API：立即返回任务 ID"""
    task = handle_user_message.delay(
        user_id, session_id, "job_search", message
    )
    return {"task_id": task.id, "status": "processing"}

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
    """WebSocket：实时推送 Agent 响应"""
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        # 触发后台任务
        handle_user_message.delay(user_id, session_id, "job_search", data)
```

---

#### 各模式对比总结

| 模式 | 与 Agent 池兼容性 | 延迟 | 资源利用率 | 适用场景 | 推荐度 |
|------|------------------|------|-----------|---------|--------|
| **消息传递** | ⭐⭐⭐⭐⭐ | 低 | 高 | 单 Agent 任务 | ⭐⭐⭐⭐⭐ |
| **Task Queue + 消息传递** | ⭐⭐⭐⭐⭐ | 中 | 高 | 生产环境 | ⭐⭐⭐⭐⭐ |
| **Sequential Pipeline** | ⭐⭐ | 高 | 低 | 多步骤工作流 | ⭐⭐ |
| **Parallel Pipeline** | ⭐ | 中 | 极低 | 并行查询 | ⭐⭐ |
| **MsgHub** | ❌ | 极高 | 极低 | 多 Agent 协作 | ❌ |

---

#### 最终建议

**对于 Job Agent 系统，推荐使用：**

1. **核心模式：Task Queue + 消息传递 + Agent 池**
   - 用 Celery/RQ 管理异步任务
   - 每个任务从 Agent 池获取 Agent，处理完归还
   - 通过 WebSocket/SSE 实时推送响应

2. **特殊场景：**
   - **简单请求响应**：直接消息传递（同步 API）
   - **复杂工作流**：单个 Agent + 内部工具链（非 Pipeline）
   - **定时任务**：Celery Beat + Agent 池

3. **避免使用：**
   - ❌ MsgHub（不适合 Agent 池）
   - ❌ Parallel Pipeline（浪费资源）
   - ❌ Sequential Pipeline 的多 Agent 模式（改用单 Agent + 工具链）

**关键原则：**
- Agent 池的核心是**快速复用**，任何长时间占用 Agent 的模式都不适合
- 复杂任务应该由**单个 Agent 通过工具调用**完成，而非多个 Agent 协作
- 使用任务队列实现异步处理，提升系统吞吐量

---

## REST API 接口定义 (REST API Specifications)

### 用户认证 API (Authentication APIs)

#### 1. 用户注册
```
POST /api/v1/auth/register
Content-Type: application/json

Request:
{
    "email": "user@example.com",
    "password": "secure_password",
    "phone": "+1234567890",
    "name": "John Doe"
}

Response:
{
    "user_id": "usr_123456",
    "token": "jwt_token_here",
    "expires_at": "2026-03-24T10:00:00Z"
}
```

#### 2. 用户登录
```
POST /api/v1/auth/login
Content-Type: application/json

Request:
{
    "email": "user@example.com",
    "password": "secure_password"
}

Response:
{
    "user_id": "usr_123456",
    "token": "jwt_token_here",
    "refresh_token": "refresh_token_here",
    "expires_at": "2026-03-24T10:00:00Z"
}
```

#### 3. Token 刷新
```
POST /api/v1/auth/refresh
Content-Type: application/json
Authorization: Bearer {refresh_token}

Response:
{
    "token": "new_jwt_token",
    "expires_at": "2026-03-24T12:00:00Z"
}
```

### 会话管理 API (Session Management APIs)

#### 4. 创建会话
```
POST /api/v1/sessions/create
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
    "conversation_type": "job_search",  // 或 "hr_communication"
    "context": {
        "job_preferences": ["software engineer", "remote"]
    }
}

Response:
{
    "session_id": "sess_789012",
    "agent_name": "JobSearchAssistant",
    "created_at": "2026-03-23T10:30:00Z"
}
```

#### 5. 获取会话列表
```
GET /api/v1/sessions?user_id={user_id}&type={conversation_type}
Authorization: Bearer {token}

Response:
{
    "sessions": [
        {
            "session_id": "sess_789012",
            "conversation_type": "job_search",
            "created_at": "2026-03-23T10:30:00Z",
            "last_activity": "2026-03-23T11:00:00Z",
            "message_count": 15
        }
    ],
    "total": 1
}
```

#### 6. 删除会话
```
DELETE /api/v1/sessions/{session_id}
Authorization: Bearer {token}

Response:
{
    "success": true,
    "message": "Session deleted successfully"
}
```

### 对话交互 API (Conversation APIs)

#### 7. 发送消息
```
POST /api/v1/chat/message
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
    "session_id": "sess_789012",
    "message": "帮我找一份软件工程师的工作",
    "attachments": [
        {
            "type": "image",
            "data": "base64_encoded_resume_image",
            "filename": "resume.pdf"
        }
    ]
}

Response:
{
    "message_id": "msg_345678",
    "agent_response": "我会帮您搜索软件工程师职位。首先，让我分析一下您的简历...",
    "actions_taken": [
        {
            "tool": "analyze_resume",
            "status": "completed"
        }
    ],
    "suggestions": [
        "您的简历中可以突出更多项目经验",
        "建议添加技术栈关键词"
    ],
    "timestamp": "2026-03-23T11:05:00Z"
}
```

#### 8. 获取对话历史
```
GET /api/v1/chat/history?session_id={session_id}&limit={limit}&offset={offset}
Authorization: Bearer {token}

Response:
{
    "messages": [
        {
            "message_id": "msg_345678",
            "role": "user",
            "content": "帮我找一份软件工程师的工作",
            "timestamp": "2026-03-23T11:00:00Z"
        },
        {
            "message_id": "msg_345679",
            "role": "assistant",
            "content": "我会帮您搜索软件工程师职位...",
            "timestamp": "2026-03-23T11:05:00Z"
        }
    ],
    "total": 15,
    "has_more": true
}
```

#### 9. 流式响应 (Server-Sent Events)
```
GET /api/v1/chat/stream?session_id={session_id}&message_id={message_id}
Authorization: Bearer {token}

Response (SSE Stream):
data: {"type": "token", "content": "我会"}
data: {"type": "token", "content": "帮您"}
data: {"type": "token", "content": "搜索"}
data: {"type": "tool_call", "tool": "search_jobs", "status": "started"}
data: {"type": "tool_result", "tool": "search_jobs", "status": "completed"}
data: {"type": "done"}
```

### 求职功能 API (Job Search APIs)

#### 10. 搜索职位
```
GET /api/v1/jobs/search?keywords={keywords}&location={location}&salary_min={amount}
Authorization: Bearer {token}

Response:
{
    "jobs": [
        {
            "job_id": "job_111222",
            "title": "Senior Software Engineer",
            "company": "TechCorp",
            "location": "San Francisco, CA",
            "salary_range": "$120k - $180k",
            "posted_date": "2026-03-20",
            "match_score": 0.92,
            "description": "We are looking for...",
            "requirements": ["Python", "AWS", "5+ years experience"]
        }
    ],
    "total": 150,
    "page": 1,
    "per_page": 10
}
```

#### 11. 获取职位详情
```
GET /api/v1/jobs/{job_id}
Authorization: Bearer {token}

Response:
{
    "job_id": "job_111222",
    "title": "Senior Software Engineer",
    "company": "TechCorp",
    "location": "San Francisco, CA",
    "salary_range": "$120k - $180k",
    "employment_type": "Full-time",
    "remote_option": true,
    "posted_date": "2026-03-20",
    "application_deadline": "2026-04-20",
    "description": "Full job description...",
    "requirements": ["Python", "AWS", "5+ years experience"],
    "benefits": ["Health insurance", "401k", "Stock options"],
    "company_info": {
        "size": "1000-5000",
        "industry": "Technology",
        "website": "https://techcorp.com"
    }
}
```

#### 12. 申请职位
```
POST /api/v1/jobs/apply
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
    "job_id": "job_111222",
    "user_id": "usr_123456",
    "resume_id": "resume_001",
    "cover_letter": "Dear Hiring Manager...",
    "additional_info": {
        "available_start_date": "2026-05-01",
        "willing_to_relocate": true
    }
}

Response:
{
    "application_id": "app_999888",
    "status": "submitted",
    "submitted_at": "2026-03-23T11:30:00Z",
    "confirmation_email_sent": true
}
```

#### 13. 获取申请状态
```
GET /api/v1/applications?user_id={user_id}&status={status}
Authorization: Bearer {token}

Response:
{
    "applications": [
        {
            "application_id": "app_999888",
            "job_id": "job_111222",
            "job_title": "Senior Software Engineer",
            "company": "TechCorp",
            "status": "under_review",
            "applied_at": "2026-03-23T11:30:00Z",
            "last_updated": "2026-03-23T14:00:00Z",
            "next_steps": "HR will contact you within 5 business days"
        }
    ],
    "total": 5
}
```

### 简历管理 API (Resume Management APIs)

#### 14. 上传简历
```
POST /api/v1/resumes/upload
Content-Type: multipart/form-data
Authorization: Bearer {token}

Request:
- file: resume.pdf
- user_id: usr_123456

Response:
{
    "resume_id": "resume_001",
    "filename": "resume.pdf",
    "parsed_data": {
        "name": "John Doe",
        "email": "john@example.com",
        "skills": ["Python", "JavaScript", "AWS"],
        "experience_years": 5,
        "education": [...]
    },
    "analysis": {
        "completeness_score": 0.85,
        "suggestions": ["Add more quantifiable achievements"]
    }
}
```

#### 15. 分析简历
```
POST /api/v1/resumes/analyze
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
    "resume_id": "resume_001",
    "target_job_id": "job_111222"
}

Response:
{
    "match_score": 0.87,
    "strengths": [
        "Strong technical skills match",
        "Relevant experience in similar companies"
    ],
    "gaps": [
        "Missing AWS certification mentioned in job requirements",
        "Limited leadership experience"
    ],
    "recommendations": [
        "Highlight your cloud infrastructure projects more prominently",
        "Add metrics to demonstrate impact (e.g., '减少部署时间50%')"
    ],
    "keyword_match": {
        "matched": ["Python", "Docker", "CI/CD"],
        "missing": ["Kubernetes", "Terraform"]
    }
}
```

#### 16. 优化简历
```
POST /api/v1/resumes/optimize
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
    "resume_id": "resume_001",
    "target_job_id": "job_111222",
    "optimization_focus": ["keywords", "formatting", "achievements"]
}

Response:
{
    "optimized_resume_id": "resume_002",
    "changes_made": [
        {
            "section": "work_experience",
            "original": "Worked on backend systems",
            "optimized": "Architected and implemented scalable backend systems serving 1M+ users, reducing API response time by 40%"
        }
    ],
    "download_url": "/api/v1/resumes/download/resume_002"
}
```

### 日历管理 API (Calendar APIs)

#### 17. 创建日历事件
```
POST /api/v1/calendar/events
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
    "user_id": "usr_123456",
    "title": "Interview with TechCorp",
    "description": "Technical interview for Senior SWE position",
    "date": "2026-03-25",
    "time": "14:00",
    "duration_minutes": 60,
    "location": "Zoom Meeting",
    "meeting_link": "https://zoom.us/j/123456789",
    "reminders": [
        {"minutes_before": 60, "method": "notification"},
        {"minutes_before": 1440, "method": "email"}
    ]
}

Response:
{
    "event_id": "evt_555666",
    "created": true,
    "calendar_provider": "google",
    "event_url": "https://calendar.google.com/event?eid=..."
}
```

#### 18. 查询日历事件
```
GET /api/v1/calendar/events?user_id={user_id}&start_date={date}&end_date={date}
Authorization: Bearer {token}

Response:
{
    "events": [
        {
            "event_id": "evt_555666",
            "title": "Interview with TechCorp",
            "date": "2026-03-25",
            "time": "14:00",
            "duration_minutes": 60,
            "status": "confirmed",
            "related_job_id": "job_111222"
        }
    ],
    "total": 3
}
```

#### 19. 更新/取消事件
```
PATCH /api/v1/calendar/events/{event_id}
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
    "status": "cancelled",
    "cancellation_reason": "Interview rescheduled"
}

Response:
{
    "event_id": "evt_555666",
    "status": "cancelled",
    "updated_at": "2026-03-24T10:00:00Z"
}
```

### HR 沟通 API (HR Communication APIs)

#### 20. 发送消息给 HR
```
POST /api/v1/hr/messages
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
    "user_id": "usr_123456",
    "job_id": "job_111222",
    "recipient_hr_id": "hr_777888",
    "subject": "Question about the position",
    "message": "I would like to know more about...",
    "priority": "normal"
}

Response:
{
    "message_id": "hrmsg_123123",
    "status": "sent",
    "sent_at": "2026-03-23T12:00:00Z",
    "expected_response_time": "24-48 hours"
}
```

#### 21. 获取 HR 消息历史
```
GET /api/v1/hr/messages?user_id={user_id}&job_id={job_id}
Authorization: Bearer {token}

Response:
{
    "messages": [
        {
            "message_id": "hrmsg_123123",
            "from": "user",
            "to": "hr",
            "subject": "Question about the position",
            "message": "I would like to know more about...",
            "timestamp": "2026-03-23T12:00:00Z",
            "read": true
        },
        {
            "message_id": "hrmsg_123124",
            "from": "hr",
            "to": "user",
            "subject": "Re: Question about the position",
            "message": "Thank you for your interest...",
            "timestamp": "2026-03-24T09:00:00Z",
            "read": false
        }
    ]
}
```

### 通知 API (Notification APIs)

#### 22. 获取通知
```
GET /api/v1/notifications?user_id={user_id}&unread_only={boolean}
Authorization: Bearer {token}

Response:
{
    "notifications": [
        {
            "notification_id": "notif_444555",
            "type": "application_status_update",
            "title": "Application Status Update",
            "message": "Your application to TechCorp has been reviewed",
            "related_entity": {
                "type": "application",
                "id": "app_999888"
            },
            "priority": "high",
            "read": false,
            "created_at": "2026-03-23T14:00:00Z"
        }
    ],
    "unread_count": 3
}
```

#### 23. 标记通知已读
```
POST /api/v1/notifications/{notification_id}/read
Authorization: Bearer {token}

Response:
{
    "success": true,
    "notification_id": "notif_444555",
    "read_at": "2026-03-23T15:00:00Z"
}
```

### 用户配置 API (User Profile APIs)

#### 24. 获取用户资料
```
GET /api/v1/users/{user_id}/profile
Authorization: Bearer {token}

Response:
{
    "user_id": "usr_123456",
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "+1234567890",
    "job_preferences": {
        "desired_roles": ["Software Engineer", "DevOps Engineer"],
        "locations": ["San Francisco", "Remote"],
        "salary_min": 120000,
        "employment_type": ["full-time", "contract"],
        "remote_preference": "remote_only"
    },
    "skills": ["Python", "AWS", "Docker"],
    "experience_years": 5,
    "current_resume_id": "resume_001"
}
```

#### 25. 更新用户资料
```
PATCH /api/v1/users/{user_id}/profile
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
    "job_preferences": {
        "desired_roles": ["Senior Software Engineer"],
        "salary_min": 150000
    }
}

Response:
{
    "success": true,
    "updated_fields": ["job_preferences"],
    "updated_at": "2026-03-23T16:00:00Z"
}
```

---

## AgentScope 工具定义 (Tool Specifications)

### 1. 求职搜索工具 (Job Search Tools)

#### search_jobs
```python
async def search_jobs(
    keywords: str,
    location: str = "",
    salary_min: int = 0,
    salary_max: int = 0,
    experience_level: str = "",
    employment_type: str = "",
    remote_only: bool = False,
    company_size: str = "",
    limit: int = 10
) -> list[dict]:
    """Search for jobs matching the specified criteria.

    Args:
        keywords: Job title or keywords to search for
        location: Preferred job location (e.g., "San Francisco, CA")
        salary_min: Minimum salary requirement
        salary_max: Maximum salary range
        experience_level: Experience level (entry, mid, senior, lead)
        employment_type: Type of employment (full-time, part-time, contract)
        remote_only: Whether to only show remote positions
        company_size: Preferred company size (startup, medium, large)
        limit: Maximum number of results to return

    Returns:
        list: List of job postings with details
    """
    pass
```

#### get_job_details
```python
async def get_job_details(job_id: str) -> dict:
    """Get detailed information about a specific job posting.

    Args:
        job_id: The unique identifier for the job posting

    Returns:
        dict: Complete job details including description, requirements, benefits
    """
    pass
```

#### get_company_info
```python
async def get_company_info(company_name: str) -> dict:
    """Fetch information about a company.

    Args:
        company_name: Name of the company

    Returns:
        dict: Company information including size, industry, culture, reviews
    """
    pass
```

#### get_salary_insights
```python
async def get_salary_insights(
    job_title: str,
    location: str,
    experience_years: int
) -> dict:
    """Get salary insights for a specific role.

    Args:
        job_title: The job title to research
        location: Location for salary comparison
        experience_years: Years of experience

    Returns:
        dict: Salary ranges, percentiles, and market trends
    """
    pass
```

### 2. 简历处理工具 (Resume Tools)

#### analyze_resume
```python
async def analyze_resume(
    resume_id: str,
    target_job_id: str = ""
) -> dict:
    """Analyze a resume and provide feedback.

    Args:
        resume_id: The ID of the resume to analyze
        target_job_id: Optional job ID to compare against

    Returns:
        dict: Analysis results with scores, strengths, gaps, recommendations
    """
    pass
```

#### optimize_resume_section
```python
async def optimize_resume_section(
    resume_id: str,
    section: str,
    target_keywords: list[str]
) -> dict:
    """Optimize a specific section of the resume.

    Args:
        resume_id: The ID of the resume
        section: Section to optimize (summary, experience, skills, education)
        target_keywords: Keywords to incorporate

    Returns:
        dict: Optimized content with before/after comparison
    """
    pass
```

#### generate_cover_letter
```python
async def generate_cover_letter(
    user_id: str,
    job_id: str,
    tone: str = "professional",
    highlights: list[str] = []
) -> str:
    """Generate a personalized cover letter for a job application.

    Args:
        user_id: The user's ID
        job_id: The job posting ID
        tone: Desired tone (professional, enthusiastic, casual)
        highlights: Specific achievements to highlight

    Returns:
        str: Generated cover letter content
    """
    pass
```

#### parse_resume_from_file
```python
async def parse_resume_from_file(
    file_path: str,
    file_type: str = "pdf"
) -> dict:
    """Parse resume content from a file (PDF/DOCX/Image).

    Args:
        file_path: Path to the resume file
        file_type: Type of file (pdf, docx, image)

    Returns:
        dict: Parsed resume data (contact info, experience, skills, education)
    """
    pass
```

### 3. 申请管理工具 (Application Tools)

#### submit_job_application
```python
async def submit_job_application(
    user_id: str,
    job_id: str,
    resume_id: str,
    cover_letter: str = "",
    additional_info: dict = {}
) -> dict:
    """Submit a job application.

    Args:
        user_id: The user's ID
        job_id: The job posting ID
        resume_id: The resume to use
        cover_letter: Optional cover letter
        additional_info: Additional information (start date, relocate, etc.)

    Returns:
        dict: Application confirmation with application_id and status
    """
    pass
```

#### track_application_status
```python
async def track_application_status(
    application_id: str
) -> dict:
    """Get the current status of a job application.

    Args:
        application_id: The application ID

    Returns:
        dict: Current status, timeline, and next steps
    """
    pass
```

#### withdraw_application
```python
async def withdraw_application(
    application_id: str,
    reason: str = ""
) -> dict:
    """Withdraw a job application.

    Args:
        application_id: The application ID
        reason: Optional reason for withdrawal

    Returns:
        dict: Confirmation of withdrawal
    """
    pass
```

### 4. 日历工具 (Calendar Tools)

#### create_calendar_event
```python
async def create_calendar_event(
    user_id: str,
    title: str,
    date: str,
    time: str,
    duration_minutes: int = 60,
    description: str = "",
    location: str = "",
    meeting_link: str = "",
    reminders: list[dict] = []
) -> dict:
    """Create a calendar event for interviews or deadlines.

    Args:
        user_id: The user's ID
        title: Event title
        date: Date in YYYY-MM-DD format
        time: Time in HH:MM format
        duration_minutes: Duration of the event
        description: Event description
        location: Location or "Virtual"
        meeting_link: Video meeting URL
        reminders: List of reminder configurations

    Returns:
        dict: Created event details with event_id
    """
    pass
```

#### query_calendar_events
```python
async def query_calendar_events(
    user_id: str,
    start_date: str,
    end_date: str,
    event_type: str = ""
) -> list[dict]:
    """Query calendar events in a date range.

    Args:
        user_id: The user's ID
        start_date: Start date in YYYY-MM-DD format
        end_date: End date in YYYY-MM-DD format
        event_type: Filter by event type (interview, deadline, meeting)

    Returns:
        list: List of calendar events
    """
    pass
```

#### update_calendar_event
```python
async def update_calendar_event(
    event_id: str,
    updates: dict
) -> dict:
    """Update an existing calendar event.

    Args:
        event_id: The event ID
        updates: Dictionary of fields to update

    Returns:
        dict: Updated event details
    """
    pass
```

#### delete_calendar_event
```python
async def delete_calendar_event(
    event_id: str
) -> dict:
    """Delete a calendar event.

    Args:
        event_id: The event ID

    Returns:
        dict: Confirmation of deletion
    """
    pass
```

### 5. HR 沟通工具 (HR Communication Tools)

#### send_hr_message
```python
async def send_hr_message(
    user_id: str,
    job_id: str,
    recipient_hr_id: str,
    subject: str,
    message: str,
    priority: str = "normal"
) -> dict:
    """Send a message to HR representative.

    Args:
        user_id: The user's ID
        job_id: Related job ID
        recipient_hr_id: HR representative's ID
        subject: Message subject
        message: Message content
        priority: Priority level (low, normal, high)

    Returns:
        dict: Message confirmation with message_id
    """
    pass
```

#### get_hr_messages
```python
async def get_hr_messages(
    user_id: str,
    job_id: str = "",
    unread_only: bool = False
) -> list[dict]:
    """Retrieve HR communication messages.

    Args:
        user_id: The user's ID
        job_id: Filter by specific job (optional)
        unread_only: Only return unread messages

    Returns:
        list: List of messages with HR
    """
    pass
```

#### schedule_hr_call
```python
async def schedule_hr_call(
    user_id: str,
    hr_id: str,
    preferred_dates: list[str],
    preferred_times: list[str],
    topic: str
) -> dict:
    """Request to schedule a call with HR.

    Args:
        user_id: The user's ID
        hr_id: HR representative's ID
        preferred_dates: List of preferred dates
        preferred_times: List of preferred time slots
        topic: Topic of discussion

    Returns:
        dict: Scheduling request confirmation
    """
    pass
```

### 6. 面试准备工具 (Interview Preparation Tools)

#### generate_interview_questions
```python
async def generate_interview_questions(
    job_id: str,
    question_type: str = "all",
    difficulty: str = "mixed"
) -> list[dict]:
    """Generate potential interview questions for a job.

    Args:
        job_id: The job posting ID
        question_type: Type of questions (technical, behavioral, all)
        difficulty: Difficulty level (easy, medium, hard, mixed)

    Returns:
        list: List of interview questions with suggested answers
    """
    pass
```

#### provide_interview_feedback
```python
async def provide_interview_feedback(
    user_id: str,
    question: str,
    user_answer: str
) -> dict:
    """Provide feedback on interview answer.

    Args:
        user_id: The user's ID
        question: The interview question
        user_answer: User's answer to evaluate

    Returns:
        dict: Feedback with score, strengths, improvements
    """
    pass
```

#### get_company_interview_insights
```python
async def get_company_interview_insights(
    company_name: str
) -> dict:
    """Get insights about a company's interview process.

    Args:
        company_name: Name of the company

    Returns:
        dict: Interview process details, common questions, tips
    """
    pass
```

### 7. 通知工具 (Notification Tools)

#### send_notification
```python
async def send_notification(
    user_id: str,
    title: str,
    message: str,
    notification_type: str,
    priority: str = "normal",
    related_entity: dict = {}
) -> dict:
    """Send a notification to the user.

    Args:
        user_id: The user's ID
        title: Notification title
        message: Notification message
        notification_type: Type (application_update, interview_reminder, etc.)
        priority: Priority level (low, normal, high)
        related_entity: Related object (job_id, application_id, etc.)

    Returns:
        dict: Notification confirmation
    """
    pass
```

#### get_notifications
```python
async def get_notifications(
    user_id: str,
    unread_only: bool = False,
    notification_type: str = ""
) -> list[dict]:
    """Retrieve user notifications.

    Args:
        user_id: The user's ID
        unread_only: Only return unread notifications
        notification_type: Filter by notification type

    Returns:
        list: List of notifications
    """
    pass
```

### 8. 文档生成工具 (Document Generation Tools)

#### generate_resignation_letter
```python
async def generate_resignation_letter(
    user_id: str,
    company_name: str,
    last_working_day: str,
    reason: str = "career_growth",
    tone: str = "professional"
) -> str:
    """Generate a resignation letter.

    Args:
        user_id: The user's ID
        company_name: Current company name
        last_working_day: Last day of work in YYYY-MM-DD format
        reason: Reason for resignation
        tone: Desired tone

    Returns:
        str: Generated resignation letter
    """
    pass
```

#### generate_thank_you_email
```python
async def generate_thank_you_email(
    user_id: str,
    interviewer_name: str,
    company_name: str,
    interview_date: str,
    key_points: list[str] = []
) -> str:
    """Generate a post-interview thank you email.

    Args:
        user_id: The user's ID
        interviewer_name: Name of the interviewer
        company_name: Company name
        interview_date: Date of interview
        key_points: Key discussion points to mention

    Returns:
        str: Generated thank you email
    """
    pass
```

### 9. 技能评估工具 (Skill Assessment Tools)

#### assess_skill_match
```python
async def assess_skill_match(
    user_skills: list[str],
    job_requirements: list[str]
) -> dict:
    """Assess how well user skills match job requirements.

    Args:
        user_skills: List of user's skills
        job_requirements: List of required skills

    Returns:
        dict: Match analysis with matched, missing, and optional skills
    """
    pass
```

#### recommend_skill_development
```python
async def recommend_skill_development(
    user_id: str,
    target_role: str
) -> list[dict]:
    """Recommend skills to develop for career goals.

    Args:
        user_id: The user's ID
        target_role: Target job role

    Returns:
        list: Recommended skills with learning resources
    """
    pass
```

### 10. 数据分析工具 (Analytics Tools)

#### get_application_statistics
```python
async def get_application_statistics(
    user_id: str,
    time_range: str = "30d"
) -> dict:
    """Get statistics about user's job applications.

    Args:
        user_id: The user's ID
        time_range: Time range for stats (7d, 30d, 90d, all)

    Returns:
        dict: Statistics including total applications, response rate, etc.
    """
    pass
```

#### analyze_job_market_trends
```python
async def analyze_job_market_trends(
    job_title: str,
    location: str = "",
    time_range: str = "90d"
) -> dict:
    """Analyze job market trends for a specific role.

    Args:
        job_title: Job title to analyze
        location: Location filter
        time_range: Time period to analyze

    Returns:
        dict: Market trends, demand, salary trends, popular skills
    """
    pass
```

---

## 实现示例 (Implementation Example)

### 完整的 Job Search Agent 实现

```python
import os
import asyncio
from agentscope.agent import ReActAgent
from agentscope.model import DashScopeChatModel
from agentscope.formatter import DashScopeChatFormatter
from agentscope.memory import InMemoryMemory
from agentscope.tool import Toolkit
from agentscope.message import Msg

# 导入所有工具函数
from tools.job_search import (
    search_jobs,
    get_job_details,
    get_company_info,
    get_salary_insights,
)
from tools.resume import (
    analyze_resume,
    optimize_resume_section,
    generate_cover_letter,
)
from tools.application import (
    submit_job_application,
    track_application_status,
)
from tools.calendar import (
    create_calendar_event,
    query_calendar_events,
)
from tools.interview import (
    generate_interview_questions,
    provide_interview_feedback,
)

class JobAgentSystem:
    """Job Agent System for multi-user job search assistance."""

    def __init__(self):
        self.model = DashScopeChatModel(
            model_name="qwen-max",
            api_key=os.environ["DASHSCOPE_API_KEY"],
            stream=True,
        )
        self.sessions = {}  # user_id -> agent mapping

    def create_job_search_toolkit(self) -> Toolkit:
        """Create toolkit for job search agent."""
        toolkit = Toolkit()

        # Register job search tools
        toolkit.register_tool_function(search_jobs)
        toolkit.register_tool_function(get_job_details)
        toolkit.register_tool_function(get_company_info)
        toolkit.register_tool_function(get_salary_insights)

        # Register resume tools
        toolkit.register_tool_function(analyze_resume)
        toolkit.register_tool_function(optimize_resume_section)
        toolkit.register_tool_function(generate_cover_letter)

        # Register application tools
        toolkit.register_tool_function(submit_job_application)
        toolkit.register_tool_function(track_application_status)

        # Register calendar tools
        toolkit.register_tool_function(create_calendar_event)
        toolkit.register_tool_function(query_calendar_events)

        # Register interview tools
        toolkit.register_tool_function(generate_interview_questions)
        toolkit.register_tool_function(provide_interview_feedback)

        return toolkit

    def create_hr_communication_toolkit(self) -> Toolkit:
        """Create toolkit for HR communication agent."""
        toolkit = Toolkit()

        # Import HR communication tools
        from tools.hr_comm import (
            send_hr_message,
            get_hr_messages,
            schedule_hr_call,
        )

        toolkit.register_tool_function(send_hr_message)
        toolkit.register_tool_function(get_hr_messages)
        toolkit.register_tool_function(schedule_hr_call)
        toolkit.register_tool_function(create_calendar_event)
        toolkit.register_tool_function(query_calendar_events)

        return toolkit

    def create_job_search_agent(self, user_id: str, session_id: str) -> ReActAgent:
        """Create a job search agent for a user."""
        memory = InMemoryMemory()
        toolkit = self.create_job_search_toolkit()

        agent = ReActAgent(
            name="JobSearchAssistant",
            sys_prompt="""You are a professional career advisor and job search assistant.

Your responsibilities:
- Help users find suitable job opportunities based on their skills and preferences
- Analyze and optimize resumes for better results
- Provide interview preparation guidance
- Track job applications and deadlines
- Offer career development advice

Be supportive, professional, and provide actionable, specific advice.
When searching for jobs, always consider the user's preferences and qualifications.
Use tools effectively to provide accurate information.""",
            model=self.model,
            memory=memory,
            formatter=DashScopeChatFormatter(),
            toolkit=toolkit,
        )

        return agent

    def create_hr_communication_agent(self, user_id: str, session_id: str) -> ReActAgent:
        """Create an HR communication agent for a user."""
        memory = InMemoryMemory()
        toolkit = self.create_hr_communication_toolkit()

        agent = ReActAgent(
            name="HRCommunicator",
            sys_prompt="""You are a professional communication assistant specializing in HR interactions.

Your responsibilities:
- Help users craft professional messages to HR representatives
- Schedule and manage interviews and calls
- Track communication history with recruiters
- Ensure all communication maintains a professional, courteous tone

Always be formal, polite, and clear in your communication.
Proofread all messages before sending.
Help users make a positive impression on potential employers.""",
            model=self.model,
            memory=memory,
            formatter=DashScopeChatFormatter(),
            toolkit=toolkit,
        )

        return agent

    async def handle_user_message(
        self,
        user_id: str,
        conversation_type: str,
        message: str,
        session_id: str = None
    ) -> str:
        """Handle a message from a user.

        Args:
            user_id: The user's unique identifier
            conversation_type: Type of conversation (job_search or hr_communication)
            message: The user's message
            session_id: Optional session ID for continuing conversation

        Returns:
            str: Agent's response
        """
        # Create or retrieve agent for this session
        session_key = f"{user_id}_{conversation_type}_{session_id or 'default'}"

        if session_key not in self.sessions:
            if conversation_type == "job_search":
                agent = self.create_job_search_agent(user_id, session_id)
            elif conversation_type == "hr_communication":
                agent = self.create_hr_communication_agent(user_id, session_id)
            else:
                raise ValueError(f"Unknown conversation type: {conversation_type}")

            self.sessions[session_key] = agent

        agent = self.sessions[session_key]

        # Process message
        msg = Msg(name="user", content=message, role="user")
        response = await agent(msg)

        return response.get_text_content()


# Example usage
async def main():
    system = JobAgentSystem()

    # User 1: Job search
    response1 = await system.handle_user_message(
        user_id="user_123",
        conversation_type="job_search",
        message="帮我找一份 Python 后端开发的工作，地点在北京或上海，薪资不低于 30k"
    )
    print(f"Agent: {response1}")

    # User 2: Different session
    response2 = await system.handle_user_message(
        user_id="user_456",
        conversation_type="job_search",
        message="请帮我分析一下我的简历"
    )
    print(f"Agent: {response2}")

    # User 1: HR communication
    response3 = await system.handle_user_message(
        user_id="user_123",
        conversation_type="hr_communication",
        message="我想给腾讯的 HR 发一封邮件询问面试结果"
    )
    print(f"Agent: {response3}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 部署建议 (Deployment Recommendations)

### 1. 本地开发环境
```bash
# 安装依赖
pip install agentscope

# 运行开发服务器
python app.py
```

### 2. 生产环境部署

**使用 Docker:**
```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "app:app"]
```

**使用 Kubernetes:**
- 参考 AgentScope 官方文档的 K8s 部署指南
- 配置 HPA 自动扩缩容
- 使用 Redis/MongoDB 作为共享会话存储

### 3. 性能优化

- **Agent 池管理:** 预创建 Agent 实例池，减少启动时间
- **缓存策略:** 缓存常见查询结果（职位列表、公司信息）
- **异步处理:** 使用任务队列处理耗时操作
- **负载均衡:** 使用 Nginx/ALB 分发请求

---

## 安全考虑 (Security Considerations)

1. **用户认证:** 使用 JWT token 进行身份验证
2. **数据加密:** 敏感数据（简历、个人信息）需加密存储
3. **API 限流:** 防止滥用和 DDoS 攻击
4. **输入验证:** 严格验证所有用户输入
5. **权限控制:** 确保用户只能访问自己的数据
6. **审计日志:** 记录所有关键操作

---

## 总结 (Summary)

本文档提供了基于 AgentScope 构建求职助手智能体系统的完整指南，包括：

- ✅ 系统架构设计
- ✅ 6 个常见问题的详细解答
- ✅ 25+ REST API 接口定义
- ✅ 40+ AgentScope 工具定义
- ✅ 完整的实现示例代码

该架构支持：
- 多用户并发使用
- 独立的会话管理
- 不同对话类型的区分
- 移动端日历集成
- 多模态内容处理
- 灵活的 Agent 通信机制

建议根据实际业务需求，选择性实现所需功能。
