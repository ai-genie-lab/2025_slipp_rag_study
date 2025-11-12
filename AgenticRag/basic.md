

## 📌 목차

### CHAPTER 08: 에이전트 주요 기능
1. 도구(Tools) 개념과 종류
2. 도구 바인딩(Bind Tools)
3. Tool Calling Agent 생성
4. 에이전트 실행과 모니터링
5. Human-in-the-Loop 패턴
6. 메모리와 대화 이력 관리

### CHAPTER 09: 에이전트 활용
1. Agentic RAG 구현
2. 검색 도구 통합
3. 문서 기반 질의응답
4. 멀티 에이전트 시스템
5. 실전 활용 사례

---

## 🎯 CHAPTER 08: 에이전트 주요 기능

### 1️⃣ 도구(Tools) 개념과 종류

#### **도구란?**
- LLM이 외부 세계와 상호작용하기 위한 **인터페이스**
- 에이전트, 체인, LLM이 실행 가능한 함수

#### **빌트인 도구 (Built-in Tools)**

**🔹 Python REPL 도구**
```python
from langchain_experimental.tools import PythonREPLTool

python_tool = PythonREPLTool()
result = python_tool.invoke("print(100 + 200)")
# 출력: 300
```
- Python 코드를 실시간으로 실행
- 계산, 데이터 처리 등에 활용

**🔹 Tavily Search (웹 검색)**
```python
from langchain_community.tools.tavily_search import TavilySearchResults

search = TavilySearchResults(
    max_results=6,
    include_answer=True,
    include_domains=["github.io", "wikidocs.net"]
)
result = search.invoke("LangChain Tools에 대해 알려주세요")
```
- 실시간 웹 검색 기능
- 최신 정보 조회 가능

**🔹 DALL-E 이미지 생성**
```python
from langchain_community.utilities.dalle_image_generator import DallEAPIWrapper

dalle = DallEAPIWrapper(
    model="dall-e-3",
    size="1024x1024",
    quality="standard"
)
image_url = dalle.run("스마트폰을 바라보는 사람들을 풍자한 neo-classicism painting")
```

#### **커스텀 도구 (Custom Tool)**

**@tool 데코레이터 활용**
```python
from langchain.tools import tool

@tool
def add_numbers(a: int, b: int) -> int:
    """두 숫자를 더합니다"""
    return a + b

@tool
def multiply_numbers(a: int, b: int) -> int:
    """두 숫자를 곱합니다"""
    return a * b

# 실행
result = add_numbers.invoke({"a": 3, "b": 4})  # 7
```

**핵심 포인트:**
- Docstring은 **영어**로 작성 (LLM이 도구 선택할 때 참고)
- 타입 힌트 필수
- 명확한 함수명과 설명

---

### 2️⃣ 도구 바인딩(Bind Tools)

#### **bind_tools()의 역할**
LLM 모델에 도구 스키마를 전달하여 도구 호출 가능하게 함

```python
from langchain_openai import ChatOpenAI

# 도구 정의
@tool
def get_word_length(word: str) -> int:
    """단어의 길이를 반환합니다"""
    return len(word)

@tool
def naver_news_crawl(news_url: str) -> str:
    """네이버 뉴스 기사를 크롤링하여 본문을 반환합니다"""
    response = requests.get(news_url)
    # ... 크롤링 로직
    return content

tools = [get_word_length, naver_news_crawl]

# LLM에 도구 바인딩
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
llm_with_tools = llm.bind_tools(tools)
```

#### **도구 호출 결과 확인**
```python
result = llm_with_tools.invoke("What is the length of the word 'teddynote'?")
print(result.tool_calls)
# [{'name': 'get_word_length', 'args': {'word': 'teddynote'}, 'id': '...'}]
```

#### **도구 파싱 및 실행**
```python
from langchain_core.output_parsers.openai_tools import JsonOutputToolsParser

chain = llm_with_tools | JsonOutputToolsParser(tools=tools)
tool_call_results = chain.invoke("What is the length of the word 'teddynote'?")

# 결과: [{'type': 'get_word_length', 'args': {'word': 'teddynote'}}]
```

---

### 3️⃣ Tool Calling Agent 생성

#### **Agent vs bind_tools**
- **bind_tools**: 단순히 도구 스키마 제공
- **Agent**: 도구 호출 → 결과 확인 → 재호출 등 **자동 실행 루프**

#### **Agent 생성 절차**

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# 1. 프롬프트 정의
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("user", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),
])

# 2. Agent 생성
agent = create_tool_calling_agent(llm, tools, prompt)

# 3. AgentExecutor 생성
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    handle_parsing_errors=True
)
```

#### **Agent 실행**
```python
result = agent_executor.invoke({
    "input": "114.5 + 121.2 + 34.2 + 110.1의 계산 결과는?"
})
print(result["output"])
```

**Agent의 자동 실행 흐름:**
1. 114.5 + 121.2 = 235.7 (add_function 호출)
2. 235.7 + 34.2 = 269.9 (add_function 재호출)
3. 269.9 + 110.1 = 380.0 (add_function 재호출)
4. 최종 답변 생성

---

### 4️⃣ 에이전트 실행과 모니터링

#### **Verbose 모드**
```python
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True  # 중간 단계 출력
)
```

#### **스트리밍 출력**
```python
from langchain_teddynote.messages import AgentStreamParser

agent_stream_parser = AgentStreamParser()

result = agent_executor.stream({"input": "질문"})
for step in result:
    agent_stream_parser.process_agent_steps(step)
```

#### **LangSmith 추적**
```python
from langchain_teddynote import logging

logging.langsmith("CH15-Agent")
```
- 도구 호출 이력
- 실행 시간
- 에러 추적
- 비용 분석

---

### 5️⃣ Human-in-the-Loop 패턴

#### **iter()를 활용한 단계별 실행**

```python
question = "114.5 + 121.2 + 34.2 + 110.1의 계산 결과는?"

for step in agent_executor.iter({"input": question}):
    if output := step.get("intermediate_step"):
        action, value = output[0]
        if action.tool == "add_function":
            print(f"\nTool: {action.tool}, 결과: {value}")
        
        # 사용자에게 계속 진행할지 확인
        _continue = input("계속 진행하시겠습니까? (y/n): ") or "Y"
        if _continue.lower() != "y":
            break

if "output" in step:
    print(step["output"])
```

**활용 시나리오:**
- 중요한 작업 수행 전 확인 필요
- 비용이 많이 드는 API 호출 시
- 민감한 데이터 처리 시
- 학습/디버깅 목적

---

### 6️⃣ 메모리와 대화 이력 관리

#### **RunnableWithMessageHistory**

```python
from langchain_community.chat_message_histories import ChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# 세션 저장소
store = {}

def get_session_history(session_id):
    if session_id not in store:
        store[session_id] = ChatMessageHistory()
    return store[session_id]

# 메모리가 추가된 Agent
agent_with_chat_history = RunnableWithMessageHistory(
    agent_executor,
    get_session_history,
    input_messages_key="input",
    history_messages_key="chat_history"
)
```

#### **대화 이력 활용**
```python
# 첫 번째 질문
response = agent_with_chat_history.stream(
    {"input": "삼성전자가 개발한 AI에 대해 알려줘"},
    config={"configurable": {"session_id": "user123"}}
)

# 후속 질문 (이전 대화 기억)
response = agent_with_chat_history.stream(
    {"input": "이전 답변을 영어로 번역해줘"},
    config={"configurable": {"session_id": "user123"}}
)
```

---

## 🚀 CHAPTER 09: 에이전트 활용

### 1️⃣ Agentic RAG 구현

#### **Agentic RAG란?**
- 기존 RAG: 검색 → 생성 (단방향)
- **Agentic RAG**: Agent가 **필요에 따라** 검색 도구 선택/실행

#### **핵심 아키텍처**
```
질문 입력
    ↓
Agent 판단: 어떤 도구를 사용할까?
    ├─→ PDF 문서 검색 (Retriever)
    ├─→ 웹 검색 (Tavily)
    └─→ 직접 답변
    ↓
결과 통합 및 답변 생성
```

---

### 2️⃣ 검색 도구 통합

#### **Tavily Search 도구**
```python
from langchain_community.tools.tavily_search import TavilySearchResults

search = TavilySearchResults(k=6)
result = search.invoke("2024년 프로야구 플레이오프 진출팀은?")
```

**특징:**
- 실시간 웹 정보
- 신뢰도 높은 검색 결과
- API 키 필요 (https://app.tavily.com)

---

### 3️⃣ 문서 기반 질의응답

#### **Retriever 도구 생성**

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings
from langchain.document_loaders import PyPDFLoader
from langchain.tools.retriever import create_retriever_tool

# 1. PDF 로드
loader = PyPDFLoader("data/SPRI_AI_Brief_2023년12월호_F.pdf")

# 2. 문서 분할
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=100
)
split_docs = loader.load_and_split(text_splitter)

# 3. Vector Store 생성
vector = FAISS.from_documents(split_docs, OpenAIEmbeddings())
retriever = vector.as_retriever()

# 4. Retriever를 도구로 변환
retriever_tool = create_retriever_tool(
    retriever,
    name="pdf_search",
    description="use this tool to search information from the PDF document"
)
```

#### **도구 목록 구성**
```python
tools = [search, retriever_tool]  # 웹 검색 + 문서 검색
```

#### **Agentic RAG Agent 생성**
```python
prompt = ChatPromptTemplate.from_messages([
    ("system", 
     "You are a helpful assistant. "
     "Make sure to use the `pdf_search` tool for searching information from the PDF document. "
     "If you can't find the information from the PDF document, use the `search` tool for searching information from the web."),
    ("placeholder", "{chat_history}"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=False)
```

#### **실행 예시**

```python
# 문서에서 검색
result = agent_executor.invoke({
    "input": "삼성전자가 개발한 생성형 AI 정보를 문서에서 찾아줘"
})
# → pdf_search 도구 사용

# 웹에서 검색
result = agent_executor.invoke({
    "input": "2024년 프로야구 플레이오프 진출팀 5개를 알려줘"
})
# → search 도구 사용
```

**Agent의 지능적 판단:**
- 문서 관련 질문 → `pdf_search` 사용
- 최신 정보 질문 → `search` 사용
- 문서에 없으면 → 웹 검색으로 전환

---

### 4️⃣ 멀티 에이전트 시스템

#### **DialogueAgent 구현**

```python
from langchain.schema import SystemMessage, HumanMessage
from langchain_openai import ChatOpenAI

class DialogueAgent:
    def __init__(self, name: str, system_message: SystemMessage, model: ChatOpenAI):
        self.name = name
        self.system_message = system_message
        self.model = model
        self.message_history = ["Here is the conversation so far."]
    
    def send(self) -> str:
        """메시지 전송"""
        message = self.model([
            self.system_message,
            HumanMessage(content="\n".join([f"{self.name}: "] + self.message_history))
        ])
        return message.content
    
    def receive(self, name: str, message: str):
        """메시지 수신"""
        self.message_history.append(f"{name}: {message}")
```

#### **DialogueSimulator 구현**

```python
class DialogueSimulator:
    def __init__(self, agents: List[DialogueAgent], selection_function):
        self.agents = agents
        self._step = 0
        self.select_next_speaker = selection_function
    
    def inject(self, name: str, message: str):
        """대화 시작"""
        for agent in self.agents:
            agent.receive(name, message)
    
    def step(self) -> tuple[str, str]:
        """한 단계 진행"""
        speaker_idx = self.select_next_speaker(self._step, self.agents)
        speaker = self.agents[speaker_idx]
        message = speaker.send()
        
        for agent in self.agents:
            agent.receive(speaker.name, message)
        
        self._step += 1
        return speaker.name, message
```

#### **토론 Agent 시나리오**

```python
# Agent 생성
agent_찬성 = DialogueAgent(
    name="찬성측",
    system_message=SystemMessage(content="당신은 AI 규제 찬성 입장입니다."),
    model=ChatOpenAI(model="gpt-4o")
)

agent_반대 = DialogueAgent(
    name="반대측",
    system_message=SystemMessage(content="당신은 AI 규제 반대 입장입니다."),
    model=ChatOpenAI(model="gpt-4o")
)

# 시뮬레이션
simulator = DialogueSimulator(
    agents=[agent_찬성, agent_반대],
    selection_function=lambda step, agents: step % len(agents)
)

simulator.inject("사회자", "AI 규제에 대한 토론을 시작합니다.")

# 5턴 진행
for i in range(5):
    name, message = simulator.step()
    print(f"{name}: {message}\n")
```

**활용 분야:**
- 의사결정 지원 (다양한 관점 분석)
- 교육/학습 (토론 시뮬레이션)
- 아이디어 브레인스토밍
- 코드 리뷰 (여러 전문가 관점)

---

### 5️⃣ 실전 활용 사례

#### **사례 1: 기술 문서 QA 챗봇**
```python
# 내부 기술 문서 + Stack Overflow 검색
tools = [
    create_retriever_tool(internal_docs_retriever, "internal_docs", "회사 내부 기술 문서 검색"),
    create_retriever_tool(api_docs_retriever, "api_docs", "API 문서 검색"),
    TavilySearchResults(k=3, include_domains=["stackoverflow.com"])
]
```

#### **사례 2: 고객 지원 Agent**
```python
tools = [
    create_retriever_tool(faq_retriever, "faq", "FAQ 검색"),
    create_retriever_tool(manual_retriever, "manual", "사용자 매뉴얼 검색"),
    send_email_tool,  # 티켓 생성
    check_order_status_tool  # 주문 상태 조회
]
```

#### **사례 3: 데이터 분석 Agent**
```python
tools = [
    PythonREPLTool(),  # 데이터 분석 코드 실행
    sql_query_tool,  # DB 조회
    plot_generation_tool,  # 차트 생성
    TavilySearchResults(k=3)  # 외부 데이터 검색
]
```

---

## 💡 핵심 정리

### CHAPTER 08 요약
1. **도구**: LLM이 외부와 상호작용하는 인터페이스
2. **바인딩**: `bind_tools()`로 LLM에 도구 스키마 전달
3. **Agent**: 도구를 자동으로 선택하고 실행하는 시스템
4. **Human-in-the-Loop**: 중간 단계에서 사용자 확인
5. **메모리**: 대화 이력 관리로 컨텍스트 유지

### CHAPTER 09 요약
1. **Agentic RAG**: Agent가 필요에 따라 검색 도구 선택
2. **검색 통합**: Tavily Search로 실시간 정보 접근
3. **문서 QA**: Retriever 도구로 문서 기반 답변
4. **멀티 Agent**: 여러 Agent 간 협업/토론
5. **실전 활용**: 다양한 도메인에 적용 가능

---

## 🔧 실습 권장 사항

1. **시작**: [01-Tools.ipynb](cci:7://file:///D:/project/2025_slipp_rag_study/15-Agent/01-Tools.ipynb:0:0-0:0) → [02-Bind-Tools.ipynb](cci:7://file:///D:/project/2025_slipp_rag_study/15-Agent/02-Bind-Tools.ipynb:0:0-0:0) 순서로 실습
2. **핵심**: [06-Agentic-RAG.ipynb](cci:7://file:///D:/project/2025_slipp_rag_study/15-Agent/06-Agentic-RAG.ipynb:0:0-0:0) 완전 이해 (템플릿 코드 활용)
3. **심화**: [10-Two-Agent-Debate-With-Tools.ipynb](cci:7://file:///D:/project/2025_slipp_rag_study/15-Agent/10-Two-Agent-Debate-With-Tools.ipynb:0:0-0:0) 멀티 Agent 구현
4. **디버깅**: LangSmith로 Agent 행동 추적
5. **최적화**: 도구 description을 명확히 작성 (Agent 판단 정확도 향상)

---

## 📚 참고 자료

- LangChain 공식 문서: https://python.langchain.com/docs/modules/agents/
- Tavily API: https://app.tavily.com/
- LangSmith: https://smith.langchain.com/