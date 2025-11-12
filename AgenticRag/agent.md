# AgentExecutor 가이드

## 목차
1. [AgentExecutor 개요](#1-agentexecutor-개요)
2. [핵심 아키텍처](#2-핵심-아키텍처)
3. [주요 속성 상세](#3-주요-속성-상세)
4. [실행 흐름과 제어](#4-실행-흐름과-제어)
5. [오류 처리 전략](#5-오류-처리-전략)
6. [성능 최적화](#6-성능-최적화)
7. [실전 활용 예제](#7-실전-활용-예제)

---

## 1. AgentExecutor 개요

### 1.1 정의
AgentExecutor는 **도구를 사용하는 에이전트를 실행하고 관리하는 핵심 클래스**입니다.

### 1.2 주요 역할
- 에이전트의 실행 루프 관리
- 도구 호출 및 검증
- 실행 시간 및 반복 횟수 제어
- 오류 처리 및 복구

### 1.3 사용 시나리오
- 복잡한 멀티스텝 작업 자동화
- 외부 API 및 도구 통합
- 자율적인 의사결정이 필요한 작업

---

## 2. 핵심 아키텍처

### 2.1 구성 요소

```
┌─────────────────────────────────────────────────────┐
│              AgentExecutor (Chain)                  │
├─────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────┐  │
│  │  Agent (3가지 타입)                           │  │
│  │  - BaseSingleActionAgent                      │  │
│  │  - BaseMultiActionAgent                       │  │
│  │  - RunnableAgent/RunnableMultiActionAgent     │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  Tools (Sequence[BaseTool])                   │  │
│  │  - name_to_tool_map: Dict[str, BaseTool]      │  │
│  │  - color_mapping: Dict[str, str]              │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  Execution Control                            │  │
│  │  - max_iterations: Optional[int] = 15         │  │
│  │  - max_execution_time: Optional[float] = None │  │
│  │  - early_stopping_method: str = "force"       │  │
│  │  - handle_parsing_errors: Union[...]          │  │
│  │  - trim_intermediate_steps: Union[int, Callable]│ │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 2.2 실제 실행 흐름 (_call 메서드)

```python
# AgentExecutor._call() 내부 로직

1. 초기화
   - name_to_tool_map 생성
   - color_mapping 생성
   - intermediate_steps = []
   - iterations = 0
   - time_elapsed = 0.0
   - start_time = time.time()

2. while self._should_continue(iterations, time_elapsed):
   
   ├─ _take_next_step() 호출
   │  │
   │  ├─ _iter_next_step() 이터레이터 실행
   │  │  │
   │  │  ├─ _prepare_intermediate_steps() → trim 적용
   │  │  │
   │  │  ├─ agent.plan() → AgentAction or AgentFinish
   │  │  │  └─ (OutputParserException 발생 시)
   │  │  │     └─ handle_parsing_errors 처리
   │  │  │        └─ ExceptionTool 실행
   │  │  │
   │  │  ├─ AgentFinish면 종료
   │  │  │
   │  │  └─ AgentAction이면 도구 실행
   │  │     └─ _perform_agent_action()
   │  │        ├─ tool in name_to_tool_map?
   │  │        │  ├─ Yes → tool.run()
   │  │        │  └─ No → InvalidTool().run()
   │  │        └─ AgentStep 반환
   │  │
   │  └─ _consume_next_step() → 결과 수집
   │
   ├─ AgentFinish면 → _return() → 종료
   │
   ├─ intermediate_steps.extend(next_step_output)
   │
   ├─ return_direct 도구인 경우 즉시 반환
   │
   ├─ iterations += 1
   └─ time_elapsed = time.time() - start_time

3. 루프 종료 (max_iterations or max_execution_time 도달)
   └─ agent.return_stopped_response()
      ├─ "force" → 상수 문자열 반환
      └─ "generate" → LLM 한 번 더 호출
```

### 2.3 비동기 실행 흐름 (_acall 메서드)

```python
# 동기 실행과 차이점

1. asyncio_timeout 컨텍스트 매니저 사용
   - max_execution_time 초과 시 TimeoutError

2. _atake_next_step() 사용
   - asyncio.gather()로 여러 도구 동시 실행
   - 병렬 처리로 성능 향상

3. TimeoutError 예외 처리
   - early_stopping_method 적용하여 안전하게 종료
```

---

## 3. 주요 속성 상세

### 3.1 agent (필수)
에이전트의 두뇌 역할을 담당합니다.

**역할:**
- 각 단계에서 다음 행동 결정
- 도구 선택 및 입력 파라미터 생성
- 최종 답변 생성

**타입:**
- `AgentType.ZERO_SHOT_REACT_DESCRIPTION`: 도구 설명 기반
- `AgentType.OPENAI_FUNCTIONS`: OpenAI 함수 호출
- `AgentType.STRUCTURED_CHAT`: 구조화된 대화

### 3.2 tools (필수)
에이전트가 사용할 수 있는 도구 목록입니다.

**도구 예시:**
```python
tools = [
    Tool(
        name="Calculator",
        func=calculator_func,
        description="수학 계산을 수행합니다"
    ),
    Tool(
        name="WebSearch",
        func=search_func,
        description="웹에서 정보를 검색합니다"
    )
]
```

### 3.3 return_intermediate_steps
중간 단계 정보 반환 여부를 결정합니다.

**False (기본값):**
```python
{
    "output": "최종 답변"
}
```

**True:**
```python
{
    "output": "최종 답변",
    "intermediate_steps": [
        (AgentAction, "도구 실행 결과"),
        (AgentAction, "도구 실행 결과"),
        ...
    ]
}
```

**활용 시나리오:**
- 디버깅 및 로깅
- 사용자에게 진행 과정 표시
- 에이전트 행동 분석

### 3.4 max_iterations
최대 실행 단계 수를 제한합니다.

**설정 가이드:**
- **간단한 작업**: 3~5
- **중간 복잡도**: 10~15
- **복잡한 작업**: 20~30
- **무제한**: None (권장하지 않음)

**주의사항:**
- 너무 낮으면 작업 미완성 가능
- 너무 높으면 무한 루프 위험 및 비용 증가

### 3.5 max_execution_time
최대 실행 시간을 초 단위로 제한합니다.

**설정 예시:**
```python
max_execution_time=60    # 1분
max_execution_time=300   # 5분
max_execution_time=None  # 무제한
```

**활용:**
- 실시간 응답이 필요한 경우
- 비용 통제
- 시스템 리소스 관리

### 3.6 early_stopping_method
제한 도달 시 종료 방법을 결정합니다.

#### "force" (강제 종료)
```python
early_stopping_method="force"  # 기본값

# 실제 구현 (BaseSingleActionAgent):
def return_stopped_response(self, early_stopping_method, intermediate_steps, **kwargs):
    if early_stopping_method == "force":
        return AgentFinish(
            {"output": "Agent stopped due to iteration limit or time limit."},
            ""
        )
```

**장점:**
- 빠른 종료
- 예측 가능한 동작

**단점:**
- 부분적인 답변도 없음

#### "generate" (답변 생성) - Agent 클래스만 지원
```python
early_stopping_method="generate"

# 실제 구현 (Agent 클래스):
def return_stopped_response(self, early_stopping_method, intermediate_steps, **kwargs):
    if early_stopping_method == "generate":
        # 중간 단계를 스크래치패드에 추가
        thoughts = ""
        for action, observation in intermediate_steps:
            thoughts += action.log
            thoughts += f"\n{self.observation_prefix}{observation}\n{self.llm_prefix}"
        
        # 최종 답변 생성 프롬프트 추가
        thoughts += "\n\nI now need to return a final answer based on the previous steps:"
        
        # LLM에게 최종 답변 요청
        full_output = self.llm_chain.predict(agent_scratchpad=thoughts, stop=self._stop, **kwargs)
        
        # 파싱 시도
        parsed_output = self.output_parser.parse(full_output)
        
        if isinstance(parsed_output, AgentFinish):
            return parsed_output
        else:
            # 파싱 실패 시 전체 출력 반환
            return AgentFinish({"output": full_output}, full_output)
```

**장점:**
- 부분적인 결과라도 제공
- 사용자 경험 향상
- 중간 단계 활용

**단점:**
- 추가 LLM 호출 비용
- Agent 클래스에서만 지원 (RunnableAgent는 미지원)

**⚠️ 주의:**
- `BaseSingleActionAgent`와 `BaseMultiActionAgent`는 "generate" 미지원
- "generate"를 사용하려면 `Agent` 클래스 상속 필요
- 지원하지 않는 에이전트에서 "generate" 사용 시 `ValueError` 발생

### 3.7 handle_parsing_errors
출력 파싱 오류 처리 방법을 설정합니다.

#### 실제 구현 분석

```python
# _iter_next_step() 메서드 내부

try:
    # 1. 중간 단계 준비 (trim 적용)
    intermediate_steps = self._prepare_intermediate_steps(intermediate_steps)
    
    # 2. Agent에게 다음 행동 요청
    output = self._action_agent.plan(
        intermediate_steps,
        callbacks=run_manager.get_child() if run_manager else None,
        **inputs,
    )
    
except OutputParserException as e:
    # 파싱 오류 발생!
    
    # handle_parsing_errors가 bool인 경우
    if isinstance(self.handle_parsing_errors, bool):
        raise_error = not self.handle_parsing_errors
    else:
        raise_error = False
    
    if raise_error:
        # False면 예외 발생
        raise ValueError(
            "An output parsing error occurred. "
            "In order to pass this error back to the agent and have it try "
            "again, pass `handle_parsing_errors=True` to the AgentExecutor. "
            f"This is the error: {str(e)}"
        )
    
    # 오류 메시지 생성
    text = str(e)
    
    if isinstance(self.handle_parsing_errors, bool):
        if e.send_to_llm:
            observation = str(e.observation)
            text = str(e.llm_output)
        else:
            observation = "Invalid or incomplete response"
    elif isinstance(self.handle_parsing_errors, str):
        # 문자열이 지정된 경우
        observation = self.handle_parsing_errors
    elif callable(self.handle_parsing_errors):
        # 함수가 지정된 경우
        observation = self.handle_parsing_errors(e)
    else:
        raise ValueError("Got unexpected type of `handle_parsing_errors`")
    
    # ExceptionTool을 사용하여 오류를 관찰로 변환
    output = AgentAction("_Exception", observation, text)
    
    # 특수 도구 실행
    observation = ExceptionTool().run(
        output.tool_input,
        verbose=self.verbose,
        color=None,
        callbacks=run_manager.get_child() if run_manager else None,
        **tool_run_kwargs,
    )
    
    # AgentStep으로 반환하여 다음 반복에서 재시도
    yield AgentStep(action=output, observation=observation)
    return
```

#### False (오류 발생)
```python
handle_parsing_errors=False  # 기본값

# 파싱 오류 발생 시:
# ValueError 예외 발생 → 실행 중단
```

#### True (자동 재시도)
```python
handle_parsing_errors=True

# 동작:
# 1. OutputParserException 캐치
# 2. 오류 메시지를 observation으로 변환
# 3. ExceptionTool 실행 (오류를 그대로 반환)
# 4. 다음 iteration에서 Agent가 오류를 보고 재시도
```

#### 문자열 (커스텀 메시지)
```python
handle_parsing_errors="출력 형식이 올바르지 않습니다. JSON 형식으로 다시 작성해주세요."

# 파싱 오류 시 이 메시지가 observation으로 전달됨
```

#### 커스텀 함수
```python
def smart_error_handler(error: OutputParserException) -> str:
    """오류 타입에 따라 다른 메시지 반환"""
    error_msg = str(error)
    
    if "JSON" in error_msg:
        return (
            "JSON 파싱 오류가 발생했습니다. "
            "다음 형식으로 응답해주세요: "
            '{"action": "도구명", "action_input": "입력값"}'
        )
    elif "Action" in error_msg:
        return "유효하지 않은 도구입니다. 사용 가능한 도구: " + ", ".join(tool_names)
    else:
        return f"오류: {error_msg}. 다시 시도해주세요."

agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    handle_parsing_errors=smart_error_handler
)
```

#### ExceptionTool 내부 구현
```python
class ExceptionTool(BaseTool):
    """예외를 처리하는 특수 도구"""
    
    name: str = "_Exception"
    description: str = "Exception tool"
    
    def _run(self, query: str, run_manager=None) -> str:
        # 오류 메시지를 그대로 반환
        return query
    
    async def _arun(self, query: str, run_manager=None) -> str:
        return query
```

**핵심 메커니즘:**
1. 파싱 오류 발생
2. `handle_parsing_errors` 설정에 따라 observation 생성
3. `ExceptionTool`이 observation을 그대로 반환
4. 다음 iteration에서 Agent가 오류 메시지를 읽고 수정된 출력 생성

### 3.8 trim_intermediate_steps
메모리 사용량 최적화를 위한 중간 단계 트리밍입니다.

#### 실제 구현
```python
def _prepare_intermediate_steps(
    self, 
    intermediate_steps: list[tuple[AgentAction, str]]
) -> list[tuple[AgentAction, str]]:
    """중간 단계를 준비 (trim 적용)"""
    
    if isinstance(self.trim_intermediate_steps, int) and self.trim_intermediate_steps > 0:
        # 양수 정수: 최근 N개만 유지
        return intermediate_steps[-self.trim_intermediate_steps:]
    
    elif callable(self.trim_intermediate_steps):
        # 함수: 커스텀 로직 적용
        return self.trim_intermediate_steps(intermediate_steps)
    
    else:
        # -1 또는 기타: 모두 유지
        return intermediate_steps
```

#### -1 (트리밍 안 함)
```python
trim_intermediate_steps=-1  # 기본값

# 모든 중간 단계 유지
# 장점: 완전한 컨텍스트
# 단점: 토큰 사용량 증가, 메모리 부담
```

#### 양수 정수 (최근 N개만 유지)
```python
trim_intermediate_steps=5

# 실제 동작:
# intermediate_steps[-5:]  # Python 슬라이싱
# 
# 예: 10개 단계 중 최근 5개만 Agent에게 전달
# [step1, step2, ..., step10] → [step6, step7, step8, step9, step10]
```

#### 커스텀 함수
```python
def custom_trim(steps: list[tuple[AgentAction, str]]) -> list[tuple[AgentAction, str]]:
    """
    중요한 단계와 최근 단계를 결합
    """
    if len(steps) <= 5:
        return steps
    
    # 최근 3개는 무조건 포함
    recent = steps[-3:]
    
    # 이전 단계 중 특정 도구 사용한 것만 포함
    important_tools = {"Calculator", "DatabaseQuery", "WebSearch"}
    important = [
        step for step in steps[:-3]
        if step[0].tool in important_tools
    ]
    
    # 중요한 단계 + 최근 단계
    return important + recent

agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    trim_intermediate_steps=custom_trim
)
```

**호출 시점:**
- `_take_next_step()` → `_iter_next_step()` 시작 시
- 매 iteration마다 `agent.plan()` 호출 전에 실행
- Agent가 받는 `intermediate_steps`는 이미 trim된 상태

**활용 시나리오:**
1. **긴 대화**: 토큰 한도 초과 방지
2. **반복 작업**: 중요한 정보만 유지
3. **비용 절감**: LLM 입력 토큰 감소

---

## 4. 실행 흐름과 제어

### 4.1 invoke 메서드
동기적으로 에이전트를 실행합니다.

```python
from langchain.agents import AgentExecutor, create_react_agent

# 에이전트 생성
agent = create_react_agent(llm, tools, prompt)

# AgentExecutor 초기화
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=15,
    verbose=True
)

# 실행
result = agent_executor.invoke({
    "input": "서울의 현재 날씨를 알려주고, 그에 맞는 옷차림을 추천해줘"
})

print(result["output"])
```

### 4.2 stream 메서드
단계별 결과를 스트리밍으로 받습니다.

#### 실제 구현
```python
def stream(
    self,
    input: Union[dict[str, Any], Any],
    config: Optional[RunnableConfig] = None,
    **kwargs: Any,
) -> Iterator[AddableDict]:
    """스트리밍 실행"""
    
    config = ensure_config(config)
    
    # AgentExecutorIterator 생성
    iterator = AgentExecutorIterator(
        self,
        input,
        config.get("callbacks"),
        tags=config.get("tags"),
        metadata=config.get("metadata"),
        run_name=config.get("run_name"),
        run_id=config.get("run_id"),
        yield_actions=True,  # 액션도 yield
        **kwargs,
    )
    
    # 이터레이터에서 단계별로 yield
    yield from iterator
```

#### 사용 예시
```python
from langchain.agents import AgentExecutor

# 스트리밍 실행
for chunk in agent_executor.stream({"input": "복잡한 작업 수행"}):
    
    # chunk는 AddableDict 타입
    # 가능한 키: "actions", "steps", "messages", "output"
    
    if "actions" in chunk:
        # AgentAction이 결정된 시점
        actions = chunk["actions"]
        for action in actions:
            print(f"🔧 도구 선택: {action.tool}")
            print(f"   입력: {action.tool_input}")
    
    elif "steps" in chunk:
        # 도구 실행이 완료된 시점
        steps = chunk["steps"]
        for step in steps:
            print(f"✅ 결과: {step.observation[:100]}...")
    
    elif "messages" in chunk:
        # 메시지 스트리밍 (RunnableAgent)
        messages = chunk["messages"]
        print(f"💬 메시지: {messages}")
    
    elif "output" in chunk:
        # 최종 출력
        print(f"🎯 최종 답변: {chunk['output']}")
```

#### 비동기 스트리밍
```python
async def stream_async():
    async for chunk in agent_executor.astream({"input": "작업"}):
        if "actions" in chunk:
            print(f"도구: {chunk['actions'][0].tool}")
        elif "output" in chunk:
            print(f"답변: {chunk['output']}")

import asyncio
asyncio.run(stream_async())
```

#### AddableDict 구조
```python
# stream()이 yield하는 각 chunk의 구조

# 1. 액션 결정 시
{
    "actions": [AgentAction(tool="Calculator", tool_input="2+2", log="...")]
}

# 2. 도구 실행 완료 시
{
    "steps": [AgentStep(action=..., observation="4")]
}

# 3. 최종 완료 시
{
    "output": "최종 답변입니다"
}

# 4. RunnableAgent의 경우 메시지 스트리밍
{
    "messages": [AIMessage(content="...")]
}
```

**장점:**
- 실시간 진행 상황 표시
- 사용자 경험 향상 (로딩 표시)
- 조기 중단 가능
- 디버깅 용이

**주의사항:**
- `yield_actions=True`가 기본값
- `AgentExecutorIterator`가 실제 스트리밍 로직 담당
- 각 chunk는 `AddableDict` 타입 (dict + 덧셈 연산 지원)

### 4.3 비동기 실행의 핵심 차이점
대량 처리 또는 비동기 환경에서 활용합니다.

#### 동기 vs 비동기 도구 실행

**동기 (_perform_agent_action):**
```python
def _perform_agent_action(self, name_to_tool_map, color_mapping, agent_action, run_manager):
    """도구를 하나씩 순차 실행"""
    
    if agent_action.tool in name_to_tool_map:
        tool = name_to_tool_map[agent_action.tool]
        
        # 동기 실행 (블로킹)
        observation = tool.run(
            agent_action.tool_input,
            verbose=self.verbose,
            color=color,
            callbacks=run_manager.get_child() if run_manager else None,
            **tool_run_kwargs,
        )
    
    return AgentStep(action=agent_action, observation=observation)
```

**비동기 (_aperform_agent_action + asyncio.gather):**
```python
async def _aiter_next_step(self, ...):
    """비동기 단계 실행"""
    
    # Agent가 여러 액션 반환 가능 (MultiActionAgent)
    actions: list[AgentAction] = output
    
    # 모든 액션을 먼저 yield
    for agent_action in actions:
        yield agent_action
    
    # ⭐ 핵심: asyncio.gather로 병렬 실행
    result = await asyncio.gather(
        *[
            self._aperform_agent_action(
                name_to_tool_map, 
                color_mapping, 
                agent_action, 
                run_manager
            )
            for agent_action in actions
        ],
    )
    
    # 모든 결과를 yield
    for chunk in result:
        yield chunk

async def _aperform_agent_action(self, name_to_tool_map, color_mapping, agent_action, run_manager):
    """단일 도구 비동기 실행"""
    
    if agent_action.tool in name_to_tool_map:
        tool = name_to_tool_map[agent_action.tool]
        
        # 비동기 실행 (non-blocking)
        observation = await tool.arun(
            agent_action.tool_input,
            verbose=self.verbose,
            color=color,
            callbacks=run_manager.get_child() if run_manager else None,
            **tool_run_kwargs,
        )
    
    return AgentStep(action=agent_action, observation=observation)
```

#### 비동기 타임아웃 처리
```python
async def _acall(self, inputs, run_manager=None):
    """비동기 실행 (타임아웃 지원)"""
    
    name_to_tool_map = {tool.name: tool for tool in self.tools}
    color_mapping = get_color_mapping([tool.name for tool in self.tools])
    intermediate_steps = []
    iterations = 0
    time_elapsed = 0.0
    start_time = time.time()
    
    try:
        # ⭐ asyncio_timeout 컨텍스트 매니저
        async with asyncio_timeout(self.max_execution_time):
            
            while self._should_continue(iterations, time_elapsed):
                # 비동기 단계 실행
                next_step_output = await self._atake_next_step(
                    name_to_tool_map,
                    color_mapping,
                    inputs,
                    intermediate_steps,
                    run_manager=run_manager,
                )
                
                if isinstance(next_step_output, AgentFinish):
                    return await self._areturn(
                        next_step_output,
                        intermediate_steps,
                        run_manager=run_manager,
                    )
                
                intermediate_steps.extend(next_step_output)
                iterations += 1
                time_elapsed = time.time() - start_time
            
            # 정상 종료 (반복 제한 도달)
            output = self._action_agent.return_stopped_response(
                self.early_stopping_method, intermediate_steps, **inputs
            )
            return await self._areturn(
                output, intermediate_steps, run_manager=run_manager
            )
    
    except (TimeoutError, asyncio.TimeoutError):
        # ⭐ 타임아웃 예외 처리
        output = self._action_agent.return_stopped_response(
            self.early_stopping_method, intermediate_steps, **inputs
        )
        return await self._areturn(
            output, intermediate_steps, run_manager=run_manager
        )
```

#### 성능 비교

**동기 실행:**
```
도구1 실행 (2초) → 도구2 실행 (3초) → 도구3 실행 (2초)
총 소요 시간: 2 + 3 + 2 = 7초
```

**비동기 실행 (asyncio.gather):**
```
도구1 실행 (2초) ┐
도구2 실행 (3초) ├─ 병렬 실행
도구3 실행 (2초) ┘
총 소요 시간: max(2, 3, 2) = 3초
```

#### 실제 사용 예시
```python
import asyncio

async def run_multiple_agents():
    """여러 에이전트 병렬 실행"""
    
    tasks = [
        agent_executor.ainvoke({"input": "작업1"}),
        agent_executor.ainvoke({"input": "작업2"}),
        agent_executor.ainvoke({"input": "작업3"}),
    ]
    
    # 모든 작업을 병렬로 실행
    results = await asyncio.gather(*tasks)
    return results

# 실행
results = asyncio.run(run_multiple_agents())

# 또는 비동기 스트리밍
async for chunk in agent_executor.astream({"input": "작업"}):
    print(chunk)
```

**핵심 차이점:**
1. **동기**: 도구를 순차적으로 하나씩 실행
2. **비동기**: `asyncio.gather()`로 여러 도구를 동시에 실행
3. **타임아웃**: 비동기만 `asyncio_timeout()` 컨텍스트 매니저 사용
4. **성능**: MultiActionAgent의 경우 비동기가 훨씬 빠름

---

## 5. 오류 처리 전략

### 5.1 파싱 오류 처리 패턴

#### 패턴 1: 자동 재시도
```python
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    handle_parsing_errors=True,  # 자동 재시도
    max_iterations=10
)
```

#### 패턴 2: 커스텀 오류 처리
```python
def smart_error_handler(error):
    error_msg = str(error)
    
    if "JSON" in error_msg:
        return "JSON 형식이 올바르지 않습니다. 다시 생성해주세요."
    elif "Tool" in error_msg:
        return "도구 이름이 올바르지 않습니다. 사용 가능한 도구 목록을 확인하세요."
    else:
        return f"오류: {error_msg}"

agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    handle_parsing_errors=smart_error_handler
)
```

### 5.2 타임아웃 처리
```python
from langchain.callbacks import TimeoutCallback

timeout_callback = TimeoutCallback(timeout=60)

try:
    result = agent_executor.invoke(
        {"input": "작업"},
        config={"callbacks": [timeout_callback]}
    )
except TimeoutError:
    print("작업이 시간 초과되었습니다")
```

### 5.3 반복 제한 처리
```python
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=10,
    early_stopping_method="generate",  # 부분 답변이라도 생성
    return_intermediate_steps=True     # 진행 상황 확인
)

result = agent_executor.invoke({"input": "작업"})

if len(result.get("intermediate_steps", [])) >= 10:
    print("경고: 최대 반복 횟수에 도달했습니다")
    print("중간 단계:", result["intermediate_steps"])
```

---

## 6. 성능 최적화

### 6.1 메모리 최적화

#### 전략 1: 중간 단계 트리밍
```python
def efficient_trim(steps):
    """
    최근 단계와 중요한 단계만 유지
    """
    # 최근 3개 단계
    recent_steps = steps[-3:]
    
    # 중요한 단계 (특정 도구 사용)
    important_steps = [
        step for step in steps[:-3]
        if step[0].tool in ["Calculator", "Database"]
    ]
    
    return important_steps + recent_steps

agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    trim_intermediate_steps=efficient_trim
)
```

#### 전략 2: 토큰 수 기반 트리밍
```python
def token_based_trim(steps, max_tokens=2000):
    """
    토큰 수 제한에 맞춰 트리밍
    """
    from tiktoken import encoding_for_model
    
    enc = encoding_for_model("gpt-4")
    current_tokens = 0
    trimmed_steps = []
    
    # 최신 단계부터 역순으로 추가
    for step in reversed(steps):
        step_tokens = len(enc.encode(str(step)))
        if current_tokens + step_tokens <= max_tokens:
            trimmed_steps.insert(0, step)
            current_tokens += step_tokens
        else:
            break
    
    return trimmed_steps
```

### 6.2 실행 시간 최적화

#### 전략 1: 적응형 제한
```python
class AdaptiveAgentExecutor:
    def __init__(self, agent, tools):
        self.agent = agent
        self.tools = tools
        self.avg_iteration_time = []
    
    def invoke(self, input_data):
        start_time = time.time()
        
        # 작업 복잡도 추정
        estimated_iterations = self._estimate_complexity(input_data)
        
        executor = AgentExecutor(
            agent=self.agent,
            tools=self.tools,
            max_iterations=estimated_iterations,
            max_execution_time=estimated_iterations * 10
        )
        
        result = executor.invoke(input_data)
        
        # 평균 시간 업데이트
        elapsed = time.time() - start_time
        self.avg_iteration_time.append(elapsed)
        
        return result
    
    def _estimate_complexity(self, input_data):
        # 간단한 휴리스틱
        query_length = len(input_data["input"])
        
        if query_length < 50:
            return 5
        elif query_length < 150:
            return 10
        else:
            return 15
```

#### 전략 2: 병렬 처리
```python
import asyncio

async def process_multiple_queries(queries):
    """
    여러 쿼리를 병렬로 처리
    """
    tasks = [
        agent_executor.ainvoke({"input": query})
        for query in queries
    ]
    
    results = await asyncio.gather(*tasks)
    return results

# 사용
queries = ["질문1", "질문2", "질문3"]
results = asyncio.run(process_multiple_queries(queries))
```

### 6.3 비용 최적화

#### 전략 1: 도구 사용 최소화
```python
from langchain.agents import AgentExecutor

class CostAwareExecutor(AgentExecutor):
    def __init__(self, *args, max_tool_calls=5, **kwargs):
        super().__init__(*args, **kwargs)
        self.max_tool_calls = max_tool_calls
        self.tool_call_count = 0
    
    def _take_next_step(self, *args, **kwargs):
        if self.tool_call_count >= self.max_tool_calls:
            # 도구 호출 제한 도달
            return AgentFinish(
                return_values={"output": "도구 사용 제한에 도달했습니다"},
                log=""
            )
        
        result = super()._take_next_step(*args, **kwargs)
        self.tool_call_count += 1
        return result
```

#### 전략 2: 캐싱
```python
from functools import lru_cache

@lru_cache(maxsize=100)
def cached_tool_call(tool_name, tool_input):
    """
    도구 호출 결과를 캐싱
    """
    tool = tool_map[tool_name]
    return tool.run(tool_input)

# 도구에 캐싱 적용
cached_tools = [
    Tool(
        name=tool.name,
        func=lambda x, t=tool: cached_tool_call(t.name, x),
        description=tool.description
    )
    for tool in tools
]
```

---

## 7. 실전 활용 예제

### 7.1 기본 예제: 리서치 에이전트
```python
from langchain.agents import create_react_agent, AgentExecutor
from langchain_openai import ChatOpenAI
from langchain.tools import Tool

# LLM 초기화
llm = ChatOpenAI(model="gpt-4", temperature=0)

# 도구 정의
def search_web(query):
    # 웹 검색 로직
    return f"'{query}'에 대한 검색 결과"

def analyze_data(data):
    # 데이터 분석 로직
    return f"분석 결과: {data}"

tools = [
    Tool(
        name="WebSearch",
        func=search_web,
        description="웹에서 정보를 검색합니다"
    ),
    Tool(
        name="DataAnalyzer",
        func=analyze_data,
        description="데이터를 분석합니다"
    )
]

# 에이전트 생성
agent = create_react_agent(llm, tools, prompt)

# AgentExecutor 설정
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=10,
    max_execution_time=120,
    early_stopping_method="generate",
    handle_parsing_errors=True,
    return_intermediate_steps=True,
    verbose=True
)

# 실행
result = agent_executor.invoke({
    "input": "2024년 AI 트렌드를 조사하고 분석해줘"
})

print("최종 답변:", result["output"])
print("실행 단계:", len(result["intermediate_steps"]))
```

### 7.2 고급 예제: 스트리밍 + 진행률 표시
```python
import sys

def stream_with_progress(query):
    """
    스트리밍으로 실행하면서 진행률 표시
    """
    step_count = 0
    
    print(f"작업 시작: {query}")
    print("-" * 50)
    
    for step in agent_executor.stream({"input": query}):
        if "actions" in step:
            step_count += 1
            action = step["actions"][0]
            print(f"\n[단계 {step_count}] 도구 사용: {action.tool}")
            print(f"  입력: {action.tool_input}")
            sys.stdout.flush()
            
        elif "steps" in step:
            observation = step["steps"][-1][1]
            print(f"  결과: {observation[:100]}...")
            sys.stdout.flush()
            
        elif "output" in step:
            print("\n" + "=" * 50)
            print("최종 답변:")
            print(step["output"])
            print("=" * 50)

# 사용
stream_with_progress("복잡한 분석 작업 수행")
```

### 7.3 프로덕션 예제: 오류 처리 + 로깅
```python
import logging
from datetime import datetime

# 로깅 설정
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

class ProductionAgentExecutor:
    def __init__(self, agent, tools):
        self.agent_executor = AgentExecutor(
            agent=agent,
            tools=tools,
            max_iterations=15,
            max_execution_time=180,
            early_stopping_method="generate",
            handle_parsing_errors=self._custom_error_handler,
            return_intermediate_steps=True,
            verbose=False  # 프로덕션에서는 False
        )
    
    def _custom_error_handler(self, error):
        """커스텀 오류 처리"""
        logger.error(f"파싱 오류 발생: {error}")
        return "오류가 발생했습니다. 다시 시도해주세요."
    
    def execute(self, query, user_id=None):
        """실행 + 로깅 + 모니터링"""
        start_time = datetime.now()
        
        logger.info(f"작업 시작 - User: {user_id}, Query: {query}")
        
        try:
            result = self.agent_executor.invoke({"input": query})
            
            elapsed = (datetime.now() - start_time).total_seconds()
            steps = len(result.get("intermediate_steps", []))
            
            logger.info(
                f"작업 완료 - User: {user_id}, "
                f"시간: {elapsed:.2f}초, 단계: {steps}"
            )
            
            # 메트릭 수집
            self._record_metrics(user_id, elapsed, steps, success=True)
            
            return {
                "success": True,
                "output": result["output"],
                "execution_time": elapsed,
                "steps": steps
            }
            
        except Exception as e:
            elapsed = (datetime.now() - start_time).total_seconds()
            logger.error(f"작업 실패 - User: {user_id}, Error: {e}")
            
            self._record_metrics(user_id, elapsed, 0, success=False)
            
            return {
                "success": False,
                "error": str(e),
                "execution_time": elapsed
            }
    
    def _record_metrics(self, user_id, time, steps, success):
        """메트릭 기록 (모니터링 시스템에 전송)"""
        metrics = {
            "user_id": user_id,
            "execution_time": time,
            "steps": steps,
            "success": success,
            "timestamp": datetime.now().isoformat()
        }
        # 실제 구현: Prometheus, DataDog 등에 전송
        logger.info(f"메트릭: {metrics}")

# 사용
executor = ProductionAgentExecutor(agent, tools)
result = executor.execute(
    "고객 데이터 분석 및 리포트 생성",
    user_id="user123"
)

if result["success"]:
    print(result["output"])
else:
    print(f"오류: {result['error']}")
```

---

## 핵심 요약

### AgentExecutor 설정 체크리스트

✅ **필수 설정**
- [ ] agent: 적절한 에이전트 타입 선택
- [ ] tools: 필요한 도구 목록 정의

✅ **실행 제어**
- [ ] max_iterations: 작업 복잡도에 맞게 설정 (10-15 권장)
- [ ] max_execution_time: 타임아웃 설정 (60-180초 권장)
- [ ] early_stopping_method: "generate" 권장

✅ **오류 처리**
- [ ] handle_parsing_errors: True 또는 커스텀 함수

✅ **최적화**
- [ ] trim_intermediate_steps: 긴 작업에서 메모리 관리
- [ ] return_intermediate_steps: 디버깅/모니터링 시에만 True

✅ **프로덕션 고려사항**
- [ ] 로깅 및 모니터링 구현
- [ ] 오류 처리 및 재시도 로직
- [ ] 비용 및 성능 최적화
- [ ] 사용자 피드백 수집

### 성능 최적화 팁

1. **메모리**: `trim_intermediate_steps`로 토큰 사용량 제어
2. **속도**: 작업 복잡도에 맞는 `max_iterations` 설정
3. **비용**: 도구 호출 횟수 모니터링 및 캐싱 활용
4. **신뢰성**: `early_stopping_method="generate"`로 부분 답변 제공
5. **모니터링**: `return_intermediate_steps=True`로 디버깅

---

## 참고 자료

- LangChain 공식 문서: https://python.langchain.com/docs/modules/agents/
- AgentExecutor API: https://api.python.langchain.com/en/latest/agents/langchain.agents.agent.AgentExecutor.html
- 에이전트 타입 가이드: https://python.langchain.com/docs/modules/agents/agent_types/