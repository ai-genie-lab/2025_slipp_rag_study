# 예상 질문 & 답변 가이드

## CHAPTER 08 관련 질문

### **Q1. 도구(Tools) 관련**

#### Q: 커스텀 도구를 만들 때 비동기(async) 처리는 어떻게 하나요?
```python
@tool
async def async_web_scraper(url: str) -> str:
    """비동기로 웹사이트를 스크래핑합니다"""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

# Agent에서 자동으로 비동기 처리
```
**답변 포인트:**
- `@tool` 데코레이터는 async 함수 지원
- Agent가 자동으로 `await` 처리
- 여러 도구 동시 호출 시 성능 향상

---

#### Q: 도구 실행 시 타임아웃이나 재시도 로직은 어떻게 구현하나요?
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@tool
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def unstable_api_call(query: str) -> str:
    """불안정한 API를 재시도 로직과 함께 호출합니다"""
    response = requests.get(f"https://api.example.com?q={query}", timeout=5)
    return response.json()
```
**답변 포인트:**
- `tenacity` 라이브러리 활용
- 도구 레벨에서 재시도 구현
- AgentExecutor의 `max_execution_time` 설정 가능

---

#### Q: 도구가 실패했을 때 Agent가 다른 도구로 폴백(fallback)하게 할 수 있나요?
**답변:**
```python
# 프롬프트에 fallback 전략 명시
prompt = ChatPromptTemplate.from_messages([
    ("system", 
     "You are a helpful assistant. "
     "If the `primary_search` tool fails, try using the `backup_search` tool. "
     "If both fail, provide an answer based on your knowledge."),
    # ...
])

# 또는 AgentExecutor의 handle_parsing_errors 활용
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    handle_parsing_errors=True,  # 에러 발생 시 재시도
    max_iterations=5
)
```

---

### **Q2. Bind Tools 관련**

#### Q: bind_tools()와 AgentExecutor의 차이가 정확히 뭔가요?
**답변:**

| 구분 | bind_tools() | AgentExecutor |
|------|-------------|---------------|
| **역할** | 도구 스키마만 LLM에 전달 | 도구 실행 + 반복 루프 |
| **실행** | 수동으로 파싱/실행 필요 | 자동 실행 |
| **반복** | 1회 호출만 | 결과 확인 후 재호출 가능 |
| **제어** | 세밀한 제어 가능 | 자동화된 흐름 |
| **사용 시점** | 커스텀 실행 로직 필요 시 | 일반적인 Agent 구현 |

```python
# bind_tools: 직접 제어
llm_with_tools = llm.bind_tools(tools)
response = llm_with_tools.invoke("질문")
# → 직접 파싱하고 도구 실행해야 함

# AgentExecutor: 자동 실행
agent_executor = AgentExecutor(agent=agent, tools=tools)
response = agent_executor.invoke({"input": "질문"})
# → 필요한 만큼 자동으로 도구 호출
```

---

#### Q: 특정 도구만 선택적으로 바인딩할 수 있나요?
**답변:**
```python
# 조건부 도구 선택
def get_tools_for_user(user_role: str):
    base_tools = [search_tool, calculator_tool]
    
    if user_role == "admin":
        base_tools.extend([delete_tool, update_tool])
    
    return base_tools

tools = get_tools_for_user(current_user.role)
llm_with_tools = llm.bind_tools(tools)
```

---

### **Q3. Agent 실행 관련**

#### Q: Agent의 무한 루프를 방지하려면?
**답변:**
```python
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=10,  # 최대 반복 횟수
    max_execution_time=60,  # 최대 실행 시간(초)
    early_stopping_method="generate"  # 조기 종료 전략
)
```

**early_stopping_method 옵션:**
- `"force"`: 즉시 종료하고 현재까지 결과 반환
- `"generate"`: LLM이 현재까지 정보로 답변 생성

---

#### Q: Agent 중간 단계(intermediate steps)를 로깅하려면?
**답변:**
```python
import logging

# 로깅 설정
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("langchain.agents")

# 또는 커스텀 콜백
from langchain.callbacks.base import BaseCallbackHandler

class CustomLoggingHandler(BaseCallbackHandler):
    def on_tool_start(self, serialized, input_str, **kwargs):
        print(f"🔧 도구 시작: {serialized['name']}")
        print(f"   입력: {input_str}")
    
    def on_tool_end(self, output, **kwargs):
        print(f" 도구 완료: {output}")

agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    callbacks=[CustomLoggingHandler()]
)
```

---

#### Q: 여러 Agent를 병렬로 실행할 수 있나요?
**답변:**
```python
import asyncio

async def run_multiple_agents(queries):
    tasks = [
        agent_executor.ainvoke({"input": query})
        for query in queries
    ]
    results = await asyncio.gather(*tasks)
    return results

# 실행
queries = ["질문1", "질문2", "질문3"]
results = asyncio.run(run_multiple_agents(queries))
```

---

### **Q4. Human-in-the-Loop 관련**

#### Q: 프로덕션 환경에서 Human-in-the-Loop를 어떻게 구현하나요?
**답변:**
```python
# 웹 애플리케이션 예시 (FastAPI)
from fastapi import FastAPI, WebSocket
import asyncio

app = FastAPI()
pending_approvals = {}

@app.websocket("/ws/agent/{session_id}")
async def agent_websocket(websocket: WebSocket, session_id: str):
    await websocket.accept()
    
    for step in agent_executor.iter({"input": query}):
        if output := step.get("intermediate_step"):
            action, value = output[0]
            
            # 클라이언트에게 승인 요청
            await websocket.send_json({
                "type": "approval_request",
                "tool": action.tool,
                "args": action.tool_input
            })
            
            # 승인 대기
            response = await websocket.receive_json()
            if response["approved"] == False:
                break
    
    await websocket.send_json({"type": "final_result", "output": step["output"]})
```

---

#### Q: 특정 도구만 Human-in-the-Loop 적용하려면?
**답변:**
```python
REQUIRE_APPROVAL = ["delete_tool", "send_email_tool", "payment_tool"]

for step in agent_executor.iter({"input": question}):
    if output := step.get("intermediate_step"):
        action, value = output[0]
        
        if action.tool in REQUIRE_APPROVAL:
            # 승인 필요
            _continue = input(f"⚠️ {action.tool} 실행 승인? (y/n): ")
            if _continue.lower() != "y":
                break
        else:
            # 자동 실행
            print(f"✅ {action.tool} 자동 실행됨")
```

---

### **Q5. 메모리 관련**

#### Q: 메모리 용량이 커지면 성능 문제가 있지 않나요?
**답변:**
```python
from langchain.memory import ConversationBufferWindowMemory

# 최근 N개 대화만 유지
memory = ConversationBufferWindowMemory(
    k=5,  # 최근 5개 턴만 기억
    return_messages=True
)

# 또는 토큰 수 기반
from langchain.memory import ConversationTokenBufferMemory

memory = ConversationTokenBufferMemory(
    llm=llm,
    max_token_limit=1000  # 최대 1000 토큰
)

# 요약 기반 메모리
from langchain.memory import ConversationSummaryMemory

memory = ConversationSummaryMemory(
    llm=llm,
    return_messages=True
)
```

---

#### Q: 세션별 메모리를 데이터베이스에 저장하려면?
**답변:**
```python
from langchain_community.chat_message_histories import SQLChatMessageHistory

def get_session_history(session_id: str):
    return SQLChatMessageHistory(
        session_id=session_id,
        connection_string="postgresql://user:pass@localhost/chatdb"
    )

# 또는 Redis
from langchain_community.chat_message_histories import RedisChatMessageHistory

def get_session_history(session_id: str):
    return RedisChatMessageHistory(
        session_id=session_id,
        url="redis://localhost:6379"
    )
```

---

## 📌 CHAPTER 09 관련 질문

### **Q6. Agentic RAG 관련**

#### Q: Agentic RAG와 기존 RAG의 성능 차이는?
**답변:**

| 지표 | 기존 RAG | Agentic RAG |
|------|----------|-------------|
| **정확도** | 70-80% | 85-95% |
| **응답시간** | 1-2초 | 3-8초 (도구 호출 수에 따라) |
| **비용** | 낮음 | 중간~높음 (LLM 호출 증가) |
| **유연성** | 낮음 | 높음 (다양한 소스 활용) |
| **복잡도** | 낮음 | 높음 |

**최적화 전략:**
```python
# 1. 캐싱
from langchain.cache import SQLiteCache
from langchain.globals import set_llm_cache

set_llm_cache(SQLiteCache(database_path=".langchain.db"))

# 2. 스트리밍으로 체감 속도 개선
for chunk in agent_executor.stream({"input": query}):
    print(chunk, end="", flush=True)

# 3. 더 빠른 모델 사용 (추론 단계)
fast_llm = ChatOpenAI(model="gpt-3.5-turbo")
agent = create_tool_calling_agent(fast_llm, tools, prompt)
```

---

#### Q: 여러 문서를 검색할 때 어떤 문서를 선택할지 Agent가 판단하나요?
**답변:**
```python
# 각 문서를 별도 도구로 등록
tools = [
    create_retriever_tool(
        tech_docs_retriever,
        name="tech_docs",
        description="Search technical documentation for API references and code examples"
    ),
    create_retriever_tool(
        business_docs_retriever,
        name="business_docs",
        description="Search business documents for policies, procedures, and guidelines"
    ),
    create_retriever_tool(
        faq_retriever,
        name="faq",
        description="Search FAQ for common questions and quick answers"
    ),
]

# Agent가 description을 보고 적절한 도구 선택
# → description을 명확하고 구체적으로 작성하는 것이 핵심!
```

---

#### Q: Retriever의 검색 품질을 개선하려면?
**답변:**
```python
# 1. Hybrid Search (키워드 + 벡터)
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25_retriever = BM25Retriever.from_documents(documents)
faiss_retriever = vector_store.as_retriever()

ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, faiss_retriever],
    weights=[0.5, 0.5]
)

# 2. Re-ranking
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CohereRerank

compressor = CohereRerank()
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=retriever
)

# 3. Query Transformation
from langchain.retrievers.multi_query import MultiQueryRetriever

retriever = MultiQueryRetriever.from_llm(
    retriever=vector_store.as_retriever(),
    llm=llm
)
```

---

### **Q7. 검색 도구 통합 관련**

#### Q: Tavily 외에 다른 검색 엔진을 사용할 수 있나요?
**답변:**
```python
# Google Search
from langchain_community.utilities import GoogleSearchAPIWrapper
from langchain.tools import Tool

search = GoogleSearchAPIWrapper()
google_tool = Tool(
    name="google_search",
    description="Search Google for recent results",
    func=search.run
)

# Bing Search
from langchain_community.utilities import BingSearchAPIWrapper

bing_search = BingSearchAPIWrapper()
bing_tool = Tool(
    name="bing_search",
    description="Search Bing for information",
    func=bing_search.run
)

# DuckDuckGo (무료, API 키 불필요)
from langchain_community.tools import DuckDuckGoSearchRun

ddg_search = DuckDuckGoSearchRun()
```

---

#### Q: 검색 결과를 필터링하거나 후처리하려면?
**답변:**
```python
@tool
def filtered_search(query: str) -> str:
    """특정 도메인만 검색하고 결과를 요약합니다"""
    # 1. 검색
    search = TavilySearchResults(
        k=10,
        include_domains=["github.com", "stackoverflow.com", "docs.python.org"]
    )
    results = search.invoke(query)
    
    # 2. 필터링 (예: 날짜, 점수 등)
    filtered = [r for r in results if r.get("score", 0) > 0.7]
    
    # 3. 후처리 (예: 중복 제거, 요약)
    seen_urls = set()
    unique_results = []
    for r in filtered:
        if r["url"] not in seen_urls:
            unique_results.append(r)
            seen_urls.add(r["url"])
    
    return "\n\n".join([f"{r['title']}: {r['content']}" for r in unique_results[:3]])
```

---

### **Q8. 멀티 에이전트 관련**

#### Q: Agent 간 통신을 비동기로 처리할 수 있나요?
**답변:**
```python
import asyncio
from typing import List

class AsyncDialogueAgent:
    async def send(self) -> str:
        message = await self.model.ainvoke([
            self.system_message,
            HumanMessage(content="\n".join([self.prefix] + self.message_history))
        ])
        return message.content

class AsyncDialogueSimulator:
    async def step(self) -> tuple[str, str]:
        speaker_idx = self.select_next_speaker(self._step, self.agents)
        speaker = self.agents[speaker_idx]
        
        # 비동기 메시지 전송
        message = await speaker.send()
        
        # 모든 Agent에게 동시에 전달
        await asyncio.gather(*[
            agent.receive_async(speaker.name, message)
            for agent in self.agents
        ])
        
        self._step += 1
        return speaker.name, message
```

---

#### Q: 멀티 Agent에서 합의(Consensus)를 도출하려면?
**답변:**
```python
class ConsensusDialogueSimulator(DialogueSimulator):
    def __init__(self, agents, threshold=0.7):
        super().__init__(agents, None)
        self.threshold = threshold
    
    def check_consensus(self) -> bool:
        """Agent들의 의견 일치도 확인"""
        # 각 Agent에게 현재 입장 점수 요청 (1-10)
        scores = []
        for agent in self.agents:
            response = agent.model.invoke([
                SystemMessage(content="Rate your agreement with the current consensus from 1-10"),
                HumanMessage(content="\n".join(agent.message_history))
            ])
            score = int(response.content.strip())
            scores.append(score)
        
        # 평균 점수가 임계값 이상이면 합의
        avg_score = sum(scores) / len(scores)
        return avg_score >= (self.threshold * 10)
    
    def run(self, topic: str, max_rounds: int = 10):
        self.inject("Moderator", topic)
        
        for round in range(max_rounds):
            name, message = self.step()
            print(f"Round {round+1} - {name}: {message}\n")
            
            if self.check_consensus():
                print("✅ 합의 도달!")
                break
```

---

#### Q: Agent 간 역할을 동적으로 할당할 수 있나요?
**답변:**
```python
class DynamicRoleAgent(DialogueAgent):
    def __init__(self, name: str, base_system_message: str, model: ChatOpenAI):
        self.base_system_message = base_system_message
        self.current_role = None
        super().__init__(name, SystemMessage(content=base_system_message), model)
    
    def assign_role(self, role: str, role_description: str):
        """동적으로 역할 변경"""
        self.current_role = role
        self.system_message = SystemMessage(
            content=f"{self.base_system_message}\n\nCurrent Role: {role}\n{role_description}"
        )
        print(f"🎭 {self.name}의 역할이 '{role}'로 변경되었습니다.")

# 사용 예시
agent = DynamicRoleAgent("Agent1", "You are a helpful assistant", llm)

# 초기: 분석가 역할
agent.assign_role("Analyst", "Analyze data and provide insights")

# 나중에: 리뷰어 역할로 변경
agent.assign_role("Reviewer", "Review and critique the analysis")
```

---

### **Q9. 실전 활용 관련**

#### Q: 프로덕션 환경에서 에러 처리는 어떻게 하나요?
**답변:**
```python
from langchain.callbacks.base import BaseCallbackHandler

class ErrorHandlingCallback(BaseCallbackHandler):
    def on_tool_error(self, error: Exception, **kwargs):
        logger.error(f"도구 실행 오류: {error}")
        # Slack, 이메일 등으로 알림
        send_alert(f"Agent tool error: {error}")
    
    def on_chain_error(self, error: Exception, **kwargs):
        logger.error(f"체인 실행 오류: {error}")
        # 모니터링 시스템에 기록
        monitoring.record_error(error)

try:
    result = agent_executor.invoke(
        {"input": query},
        config={"callbacks": [ErrorHandlingCallback()]}
    )
except Exception as e:
    # Fallback 응답
    result = {
        "output": "죄송합니다. 일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요.",
        "error": str(e)
    }
    logger.exception("Agent execution failed")
```

---

#### Q: Agent의 비용을 추적하고 제한하려면?
**답변:**
```python
from langchain.callbacks import get_openai_callback

# 비용 추적
with get_openai_callback() as cb:
    result = agent_executor.invoke({"input": query})
    print(f"토큰 사용: {cb.total_tokens}")
    print(f"비용: ${cb.total_cost:.4f}")

# 비용 제한
class CostLimitCallback(BaseCallbackHandler):
    def __init__(self, max_cost: float):
        self.max_cost = max_cost
        self.current_cost = 0.0
    
    def on_llm_end(self, response, **kwargs):
        # 비용 계산 (대략적)
        tokens = response.llm_output.get("token_usage", {})
        cost = (tokens.get("total_tokens", 0) / 1000) * 0.002  # gpt-3.5 기준
        self.current_cost += cost
        
        if self.current_cost > self.max_cost:
            raise Exception(f"비용 한도 초과: ${self.current_cost:.4f}")

agent_executor.invoke(
    {"input": query},
    config={"callbacks": [CostLimitCallback(max_cost=0.50)]}
)
```

---

#### Q: Agent 응답 속도를 개선하려면?
**답변:**
```python
# 1. 스트리밍 활성화
for chunk in agent_executor.stream({"input": query}):
    print(chunk, end="", flush=True)

# 2. 더 빠른 모델 사용
llm = ChatOpenAI(model="gpt-3.5-turbo")  # gpt-4 대신

# 3. 병렬 도구 실행
from langchain.agents import AgentType

agent = initialize_agent(
    tools=tools,
    llm=llm,
    agent=AgentType.OPENAI_MULTI_FUNCTIONS,  # 병렬 실행 지원
)

# 4. 캐싱
from langchain.cache import InMemoryCache
set_llm_cache(InMemoryCache())

# 5. Embedding 캐싱
from langchain_community.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore

store = LocalFileStore("./cache/")
cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    OpenAIEmbeddings(),
    store,
    namespace="openai_embeddings"
)
```

---

## 🔥 고급 질문

### **Q10. LangGraph vs AgentExecutor 차이는?**
**답변:**

| 기능 | AgentExecutor | LangGraph |
|------|---------------|-----------|
| **제어 흐름** | 선형적 | 그래프 기반 (조건부 분기) |
| **복잡도** | 단순 | 복잡한 워크플로우 가능 |
| **상태 관리** | 제한적 | 명시적 상태 관리 |
| **디버깅** | 어려움 | 시각화 가능 |
| **사용 시점** | 일반적인 Agent | 복잡한 워크플로우 |

```python
# LangGraph 예시
from langgraph.graph import StateGraph

workflow = StateGraph()
workflow.add_node("search", search_node)
workflow.add_node("analyze", analyze_node)
workflow.add_node("respond", respond_node)

workflow.add_edge("search", "analyze")
workflow.add_conditional_edges(
    "analyze",
    should_continue,
    {
        "continue": "search",
        "end": "respond"
    }
)
```

---

### **Q11. Agent의 hallucination을 방지하려면?**
**답변:**
```python
# 1. 프롬프트에 명시
prompt = ChatPromptTemplate.from_messages([
    ("system", 
     "You are a helpful assistant. "
     "IMPORTANT: Only use information from the tools. "
     "If you cannot find the information using tools, say 'I don't have enough information' instead of making up answers."),
    # ...
])

# 2. 검증 단계 추가
@tool
def verified_search(query: str) -> str:
    """검증된 정보만 반환하는 검색"""
    results = search_tool.invoke(query)
    
    # 출처 확인
    if not results or len(results) == 0:
        return "No verified information found"
    
    # 신뢰도 점수 확인
    verified_results = [r for r in results if r.get("score", 0) > 0.8]
    
    return "\n".join([f"[{r['url']}] {r['content']}" for r in verified_results])

# 3. 후처리 검증
def validate_response(response: str, sources: list) -> bool:
    """응답이 출처와 일치하는지 검증"""
    # LLM을 사용한 사실 검증
    validator = ChatOpenAI(model="gpt-4", temperature=0)
    validation_result = validator.invoke([
        SystemMessage(content="Verify if the response is supported by the sources"),
        HumanMessage(content=f"Response: {response}\n\nSources: {sources}")
    ])
    return "verified" in validation_result.content.lower()
```

---

### **Q12. 보안 관련 고려사항은?**
**답변:**
```python
# 1. 도구 실행 권한 제어
class SecureToolWrapper:
    def __init__(self, tool, allowed_users: set):
        self.tool = tool
        self.allowed_users = allowed_users
    
    def invoke(self, input_data, user_id: str):
        if user_id not in self.allowed_users:
            raise PermissionError(f"User {user_id} not authorized")
        return self.tool.invoke(input_data)

# 2. 입력 검증
@tool
def secure_database_query(query: str) -> str:
    """안전한 데이터베이스 쿼리"""
    # SQL Injection 방지
    if any(keyword in query.lower() for keyword in ["drop", "delete", "truncate"]):
        raise ValueError("Dangerous SQL keyword detected")
    
    # Parameterized query 사용
    return execute_safe_query(query)

# 3. 출력 필터링
def filter_sensitive_data(response: str) -> str:
    """민감한 정보 마스킹"""
    import re
    # 이메일 마스킹
    response = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', 
                      '***@***.***', response)
    # 전화번호 마스킹
    response = re.sub(r'\d{3}-\d{4}-\d{4}', '***-****-****', response)
    return response
```

---

## 💬 실무 팁

### 디버깅 체크리스트
```python
# Agent가 잘 안될 때 확인할 것들
checklist = {
    "1. 도구 description": "영어로, 명확하고 구체적으로 작성했는가?",
    "2. 프롬프트": "도구 사용 방법을 명시했는가?",
    "3. 모델": "tool calling을 지원하는 모델인가? (gpt-4, gpt-3.5-turbo 등)",
    "4. 타입 힌트": "도구 함수에 타입 힌트가 있는가?",
    "5. 에러 처리": "handle_parsing_errors=True 설정했는가?",
    "6. 반복 제한": "max_iterations가 충분한가?",
    "7. verbose": "디버깅을 위해 verbose=True로 설정했는가?",
}
```