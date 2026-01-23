# Multi-Agent Design Patterns

> **Source**: [Developer's Guide to Multi-Agent Patterns in ADK](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)
> **Author**: Shubham Saboo, Senior AI Product Manager at Google

Multi-Agent Systems (MAS)는 AI의 마이크로서비스 아키텍처입니다. 단일 에이전트가 너무 많은 책임을 지면 복잡도가 증가하면서 신뢰성이 저하됩니다.

---

## Pattern Overview

| # | Pattern | 별칭 | ADK Agent | Use Case |
|---|---------|------|-----------|----------|
| 1 | Sequential Pipeline | Assembly Line | `SequentialAgent` | 단계별 처리 |
| 2 | Coordinator/Dispatcher | Concierge | `Agent` + sub_agents | 의도 기반 라우팅 |
| 3 | Parallel Fan-Out/Gather | Octopus | `ParallelAgent` | 병렬 처리 |
| 4 | Hierarchical Decomposition | Russian Doll | `AgentTool` | 태스크 분할 |
| 5 | Generator/Critic | Editor's Desk | `LoopAgent` | 생성-검증 |
| 6 | Iterative Refinement | Sculptor | `LoopAgent` | 품질 개선 |
| 7 | Human-in-the-Loop | Safety Net | Custom Tool | 인간 승인 |
| 8 | Composite Patterns | Mix-and-Match | Mixed | 패턴 조합 |

---

## 1. Sequential Pipeline (Assembly Line)

선형적이고 결정론적인 워크플로우. 에이전트가 순차적으로 작업을 전달합니다.

### Use Case
- PDF 처리: Parser → Extractor → Summarizer
- 데이터 파이프라인
- ETL 작업

### Implementation

```python
from google.adk.agents import SequentialAgent, Agent

parser = Agent(
    name="parser",
    model="gemini-2.5-flash",
    instruction="Parse the PDF and extract raw text.",
    output_key="parsed_text",  # session.state에 저장
)

extractor = Agent(
    name="extractor",
    model="gemini-2.5-flash",
    instruction="Extract key information from {parsed_text}.",
    output_key="extracted_data",
)

summarizer = Agent(
    name="summarizer",
    model="gemini-2.5-flash",
    instruction="Summarize {extracted_data} in 3 bullet points.",
    output_key="summary",
)

pipeline = SequentialAgent(
    name="pdf_pipeline",
    sub_agents=[parser, extractor, summarizer],
)
```

### Key Points
- `output_key`를 사용하여 `session.state`에 결과 저장
- 다음 에이전트가 이전 결과에 접근 가능
- 명확한 키 이름 사용 권장

---

## 2. Coordinator/Dispatcher (Concierge)

중앙 에이전트가 사용자 의도를 분석하고 적절한 전문 에이전트로 라우팅합니다.

### Use Case
- 고객 서비스 시스템
- 멀티 도메인 챗봇
- 헬프데스크

### Implementation

```python
from google.adk.agents import Agent

billing_agent = Agent(
    name="billing_agent",
    model="gemini-2.5-flash",
    description="Handles billing inquiries, payment issues, refunds.",  # 라우팅 힌트
    instruction="Help with billing and payment questions.",
)

tech_support = Agent(
    name="tech_support",
    model="gemini-2.5-flash",
    description="Handles technical issues, bugs, troubleshooting.",
    instruction="Provide technical support and troubleshooting.",
)

coordinator = Agent(
    name="coordinator",
    model="gemini-2.5-flash",
    instruction="""Analyze user intent and delegate to the appropriate specialist.
    - Billing questions → billing_agent
    - Technical issues → tech_support
    """,
    sub_agents=[billing_agent, tech_support],  # AutoFlow로 자동 라우팅
)
```

### Key Points
- `description`이 LLM 라우팅 결정의 핵심
- ADK의 AutoFlow 메커니즘 활용
- 명확한 역할 분리 필요

---

## 3. Parallel Fan-Out/Gather (Octopus)

여러 독립적인 에이전트가 동시에 실행되어 지연 시간을 줄이거나 다양한 관점을 얻습니다.

### Use Case
- 코드 리뷰 (보안, 스타일, 성능 동시 검사)
- 다중 소스 검색
- 앙상블 분석

### Implementation

```python
from google.adk.agents import ParallelAgent, Agent

security_auditor = Agent(
    name="security_auditor",
    model="gemini-2.5-flash",
    instruction="Analyze code for security vulnerabilities.",
    output_key="security_report",
)

style_enforcer = Agent(
    name="style_enforcer",
    model="gemini-2.5-flash",
    instruction="Check code style and formatting.",
    output_key="style_report",
)

performance_analyst = Agent(
    name="performance_analyst",
    model="gemini-2.5-flash",
    instruction="Identify performance bottlenecks.",
    output_key="performance_report",
)

synthesizer = Agent(
    name="synthesizer",
    model="gemini-2.5-flash",
    instruction="""Combine all reports:
    - Security: {security_report}
    - Style: {style_report}
    - Performance: {performance_report}
    Provide unified recommendations.""",
)

code_reviewer = SequentialAgent(
    name="code_reviewer",
    sub_agents=[
        ParallelAgent(
            name="parallel_analysis",
            sub_agents=[security_auditor, style_enforcer, performance_analyst],
        ),
        synthesizer,
    ],
)
```

### Key Points
- 독립적인 태스크에 적합
- 결과 통합을 위한 Synthesizer 에이전트 필요
- `output_key`로 각 에이전트 결과 구분

---

## 4. Hierarchical Decomposition (Russian Doll)

복잡한 태스크를 부모 에이전트가 서브태스크로 분할하여 자식 에이전트에게 위임합니다.

### Use Case
- 복잡한 프로젝트 관리
- 연구 보고서 작성
- 대규모 데이터 분석

### Implementation

```python
from google.adk.agents import Agent
from google.adk.tools import AgentTool

# 서브 에이전트를 AgentTool로 래핑
research_agent = Agent(
    name="research_agent",
    model="gemini-2.5-flash",
    instruction="Conduct research on the given topic.",
)

research_tool = AgentTool(agent=research_agent)

analysis_agent = Agent(
    name="analysis_agent",
    model="gemini-2.5-flash",
    instruction="Analyze the research findings.",
)

analysis_tool = AgentTool(agent=analysis_agent)

# 부모 에이전트
project_manager = Agent(
    name="project_manager",
    model="gemini-2.5-flash",
    instruction="""You manage a research project.
    1. Use research_tool to gather information
    2. Use analysis_tool to analyze findings
    3. Synthesize results into final report
    """,
    tools=[research_tool, analysis_tool],  # 함수처럼 호출
)
```

### Key Points
- `AgentTool`로 서브 에이전트를 도구처럼 사용
- 라우팅과 달리 부모가 부분 작업 위임 후 결과 대기
- 복잡한 조건부 로직에 적합

---

## 5. Generator/Critic (Editor's Desk)

콘텐츠 생성과 검증을 분리하여 조건부 루프로 실행합니다.

### Use Case
- 콘텐츠 품질 보증
- 코드 생성 및 검증
- 문서 작성

### Implementation

```python
from google.adk.agents import LoopAgent, Agent

generator = Agent(
    name="generator",
    model="gemini-2.5-flash",
    instruction="""Generate content based on requirements.
    If feedback is provided, improve based on it: {feedback}
    """,
    output_key="draft",
)

critic = Agent(
    name="critic",
    model="gemini-2.5-flash",
    instruction="""Evaluate the draft: {draft}

    If quality is acceptable:
    - Respond with: APPROVED

    If improvements needed:
    - Provide specific feedback
    """,
    output_key="feedback",
)

editor = LoopAgent(
    name="editor",
    sub_agents=[generator, critic],
    max_iterations=3,
    # exit_condition: "APPROVED"가 응답에 포함되면 종료
)
```

### Key Points
- `LoopAgent`로 반복 실행
- 품질 게이트로 종료 조건 설정
- Pass/Fail 판정에 초점

---

## 6. Iterative Refinement (Sculptor)

점진적 품질 개선을 위한 순환 프로세스입니다. Generator/Critic과 달리 Pass/Fail이 아닌 지속적 개선에 초점을 둡니다.

### Use Case
- 에세이/보고서 작성
- 디자인 개선
- 최적화 작업

### Implementation

```python
from google.adk.agents import LoopAgent, Agent

draft_writer = Agent(
    name="draft_writer",
    model="gemini-2.5-flash",
    instruction="Create initial draft or improve based on feedback.",
    output_key="current_draft",
)

critique_agent = Agent(
    name="critique_agent",
    model="gemini-2.5-flash",
    instruction="""Review {current_draft} and suggest improvements.
    Focus on: clarity, structure, engagement.
    If excellent (score > 9/10), respond: FINALIZED
    """,
    output_key="critique",
)

refinement_agent = Agent(
    name="refinement_agent",
    model="gemini-2.5-flash",
    instruction="""Polish {current_draft} based on {critique}.
    Make targeted improvements while preserving good elements.
    """,
    output_key="refined_draft",
)

sculptor = LoopAgent(
    name="sculptor",
    sub_agents=[draft_writer, critique_agent, refinement_agent],
    max_iterations=5,
)
```

### Key Points
- 품질 임계값 도달 시 조기 종료 가능
- Generator/Critic보다 더 세분화된 개선
- 정성적 품질 향상에 초점

---

## 7. Human-in-the-Loop (Safety Net)

에이전트가 기본 작업을 처리하고, 고위험 결정에 인간 승인을 요청합니다.

### Use Case
- 금융 거래 승인
- 프로덕션 배포
- 민감한 데이터 처리

### Implementation

```python
from google.adk.agents import Agent
from google.adk.tools import FunctionTool

def request_human_approval(action: str, details: str) -> str:
    """Request human approval for high-stakes actions.

    Args:
        action: The action requiring approval
        details: Details about the action

    Returns:
        Approval status from human
    """
    # 실제 구현: 슬랙 알림, 이메일, UI 프롬프트 등
    print(f"APPROVAL REQUIRED: {action}")
    print(f"Details: {details}")

    # 실행 일시 중지 및 인간 개입 대기
    response = input("Approve? (yes/no): ")
    return "APPROVED" if response.lower() == "yes" else "REJECTED"

approval_tool = FunctionTool(func=request_human_approval)

financial_agent = Agent(
    name="financial_agent",
    model="gemini-2.5-flash",
    instruction="""Process financial transactions.
    For transactions over $10,000, use request_human_approval tool.
    Never proceed without approval for large amounts.
    """,
    tools=[approval_tool],
)
```

### Key Points
- 되돌릴 수 없는 작업에 필수
- 커스텀 도구로 실행 중단 구현
- 알림 시스템 통합 가능

---

## 8. Composite Patterns (Mix-and-Match)

실제 시스템에서는 여러 패턴을 조합합니다.

### Example: 고객 지원 시스템

```
User Request
    │
    ▼
┌─────────────────┐
│   Coordinator   │  ← Pattern 2: Dispatcher
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│Billing│ │ Tech  │
│ Agent │ │Support│
└───┬───┘ └───┬───┘
    │         │
    ▼         ▼
┌─────────────────┐
│ Parallel Search │  ← Pattern 3: Fan-Out
│  (Docs + FAQ)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Generator/Critic│  ← Pattern 5: Quality Gate
│  (Tone Check)   │
└────────┬────────┘
         │
         ▼
    Final Response
```

### Implementation Sketch

```python
from google.adk.agents import Agent, SequentialAgent, ParallelAgent, LoopAgent

# 병렬 검색
doc_search = Agent(name="doc_search", ...)
faq_search = Agent(name="faq_search", ...)

parallel_search = ParallelAgent(
    name="parallel_search",
    sub_agents=[doc_search, faq_search],
)

# 응답 생성 및 톤 체크
response_generator = Agent(name="response_generator", ...)
tone_checker = Agent(name="tone_checker", ...)

quality_gate = LoopAgent(
    name="quality_gate",
    sub_agents=[response_generator, tone_checker],
    max_iterations=2,
)

# 전문 에이전트 파이프라인
billing_pipeline = SequentialAgent(
    name="billing_pipeline",
    sub_agents=[parallel_search, quality_gate],
)

tech_pipeline = SequentialAgent(
    name="tech_pipeline",
    sub_agents=[parallel_search, quality_gate],
)

# 코디네이터
support_system = Agent(
    name="support_system",
    model="gemini-2.5-flash",
    instruction="Route to appropriate specialist.",
    sub_agents=[billing_pipeline, tech_pipeline],
)
```

---

## Implementation Tips

### 1. State Management

```python
# 명확한 output_key 사용
agent = Agent(
    name="analyzer",
    output_key="analysis_result_v2",  # 설명적인 이름
)

# 다운스트림 에이전트에서 접근
next_agent = Agent(
    instruction="Based on {analysis_result_v2}, ...",
)
```

### 2. Clear Descriptions

```python
# description은 LLM 라우팅의 핵심
billing_agent = Agent(
    name="billing_agent",
    description="""Handles ALL billing inquiries including:
    - Payment issues and refunds
    - Subscription management
    - Invoice questions
    - Pricing inquiries
    NOT for: technical support, account security
    """,
)
```

### 3. Incremental Complexity

1. **Sequential Chain**부터 시작
2. 충분히 디버깅
3. 점진적으로 복잡도 추가

---

## Pattern Selection Guide

| 요구사항 | 추천 패턴 |
|----------|-----------|
| 순차적 데이터 처리 | Sequential Pipeline |
| 의도 기반 라우팅 | Coordinator/Dispatcher |
| 독립적 병렬 작업 | Parallel Fan-Out |
| 복잡한 태스크 분할 | Hierarchical Decomposition |
| 품질 검증 필요 | Generator/Critic |
| 지속적 개선 | Iterative Refinement |
| 고위험 결정 | Human-in-the-Loop |
| 복합 요구사항 | Composite Patterns |

---

## Official Resources

| Resource | URL |
|----------|-----|
| Original Article | https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/ |
| ADK Documentation | https://google.github.io/adk-docs/ |
| ADK Samples | https://github.com/google/adk-samples |
