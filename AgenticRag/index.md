# AGENTIC RAG

## 목차
1. [요약](index.md)
2. [기본](basic.md)
3. [질문](qna.md)
3. [심화](agent.md)
4. [실습](practice.md)

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


- 기본
    - bind tools와 AgentExecutor 차이
    - 도구 호출 에이전트(Tool Calling Agent)
    - AgentExecutor
    - 질문
- 심화
    - AgentExecutor
- 실습 (링크들)
    - 데이터셋 미라지
    - 