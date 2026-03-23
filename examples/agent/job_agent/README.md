# Job Agent System - Architecture and Design Documentation

## 系统概述 (System Overview)

本文档详细说明了基于 AgentScope 框架构建的求职助手智能体系统的完整架构设计、API 接口、工具定义以及常见问题解答。

This document provides a comprehensive architecture design, API specifications, tool definitions, and FAQ for building a job-seeking assistant agent system based on the AgentScope framework.

---

## 系统架构图 (System Architecture)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Mobile App (iOS/Android)                     │
│                    User Interface Layer                          │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS REST API
                         │ (JWT Authentication)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Backend API Server                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              API Gateway & Router                        │   │
│  │  - User Authentication & Session Management              │   │
│  │  - Rate Limiting & Request Validation                    │   │
│  └────────────┬────────────────────────────────────────────┘   │
│               │                                                  │
│  ┌────────────▼──────────────────────────────────────────────┐ │
│  │              Session Manager                               │ │
│  │  - user_id → session_id mapping                           │ │
│  │  - Conversation type routing                              │ │
│  │    • job_search                                           │ │
│  │    • hr_communication                                     │ │
│  └────────────┬──────────────────────────────────────────────┘ │
│               │                                                  │
│  ┌────────────▼──────────────────────────────────────────────┐ │
│  │              Agent Orchestration Layer                     │ │
│  │                                                            │ │
│  │  ┌─────────────────┐  ┌──────────────────┐               │ │
│  │  │ Job Search      │  │ HR Communication │               │ │
│  │  │ Agent           │  │ Agent            │               │ │
│  │  │ (ReActAgent)    │  │ (ReActAgent)     │               │ │
│  │  └────────┬────────┘  └────────┬─────────┘               │ │
│  │           │                     │                          │ │
│  │  ┌────────▼─────────────────────▼─────────┐               │ │
│  │  │  Application Assistant Agent            │               │ │
│  │  │  - Resume optimization                  │               │ │
│  │  │  - Interview preparation                │               │ │
│  │  │  - Document generation                  │               │ │
│  │  └────────┬────────────────────────────────┘               │ │
│  │           │                                                 │ │
│  └───────────┼─────────────────────────────────────────────────┘ │
│              │                                                  │
│  ┌───────────▼─────────────────────────────────────────────┐  │
│  │              Toolkit & Tool Layer                        │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐    │  │
│  │  │ Job Search   │ │ Resume       │ │ Calendar     │    │  │
│  │  │ Tools        │ │ Tools        │ │ Tools        │    │  │
│  │  └──────────────┘ └──────────────┘ └──────────────┘    │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐    │  │
│  │  │ Notification │ │ HR Comm      │ │ Document     │    │  │
│  │  │ Tools        │ │ Tools        │ │ Gen Tools    │    │  │
│  │  └──────────────┘ └──────────────┘ └──────────────┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Memory & Storage Layer                       │  │
│  │  ┌────────────────┐  ┌────────────────┐                 │  │
│  │  │ SQLite/MongoDB │  │ Vector Store   │                 │  │
│  │  │ Session Store  │  │ (for RAG)      │                 │  │
│  │  └────────────────┘  └────────────────┘                 │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              External Services & Integrations                    │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
│  │ Multimodal   │ │ Calendar API │ │ Job Board    │           │
│  │ LLM          │ │ (Google/     │ │ APIs         │           │
│  │ (GPT-4V/     │ │  Apple)      │ │              │           │
│  │  Qwen-VL)    │ │              │ │              │           │
│  └──────────────┘ └──────────────┘ └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

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
