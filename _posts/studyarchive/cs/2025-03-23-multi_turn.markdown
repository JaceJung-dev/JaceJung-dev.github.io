---
layout: post
title: "Chatbot Multi-turn(멀티턴), LangChain Memory, Redis"
date: 2025-03-23 22:00:00 +0900
categories:
  - studyarchive
  - cs
notice: |
  ⚠️ **안내:** 이 글은 학습 기록용입니다. 오류나 보완 의견은 댓글로 알려주세요.
---

# Chatbot Multi-turn(멀티턴), LangChain Memory, Redis

## Chatbot Multi-turn(멀티턴)

### 멀티턴이란?

멀티턴(Multi-turn)은 사용자와 챗봇 간의 여러 차례에 걸친 대화를 의미합니다.
싱글턴(Single-turn)이 한 번의 질문과 응답으로 끝나는 것과 달리, 멀티턴 대화에서는 이전 대화의 맥락을 기억하고 활용하여 연속적인 대화가 가능합니다.

**멀티턴이 필요한 이유**

- **문맥 유지**: "그것에 대해 더 알려줘"와 같은 대명사 참조를 이해하려면 이전 대화 내용이 필요
- **대화 흐름**: 복잡한 주제를 여러 번의 질문으로 나누어 점진적으로 탐색 가능
- **개인화**: 사용자가 이전에 언급한 선호도나 정보를 기억하여 맞춤형 응답 제공
- **자연스러운 대화**: 사람 간의 대화처럼 자연스러운 상호작용 구현

**멀티턴 구현의 핵심 요소**

1. **대화 기록 저장**: 이전 메시지들을 어딘가에 저장
2. **컨텍스트 주입**: LLM 호출 시 저장된 대화 기록을 프롬프트에 포함
3. **세션 관리**: 사용자별로 독립적인 대화 기록 유지 (session_id 활용)

### LangChain vs LangGraph에서의 멀티턴

LangChain/LangGraph 프레임워크에서는 각각 다른 방식으로 구현
각각의 방식은 아래와 같은 특징을 가짐:

- **LangChain**
  - `RunnableWithMessageHistory`나 `BaseChatMessageHistory`로 구현 가능
  - '질문-RAG-LLM-응답' 전체 사이클을 거친 후 저장하고 사용. 이전 대화를 재사용할 때도 모든 사이클을 다시 거쳐야 함
- **LangGraph**
  - `LangGraph Persistence`로 구현 가능
  - '질문-RAG-LLM-응답' 각 단계에서 상태(state)를 저장하고, 재사용 시 특정 단계에서 시작하거나 다른 분기로 진행 가능
  - LangGraph의 장점은 이전 대화를 이용하더라도 처음부터 같은 파이프라인을 타는 것이 아니라, 파이프라인 중간이나 다른 파이프라인으로 이동해서 다른 프롬프트, 다른 작업에서 사용 가능

### 1. LangChain Memory 종류

LangChain에서 대화 메모리는 LLM이 사용자와의 이전 상호작용을 기억할 수 있도록 합니다.
챗봇에서는 애플리케이션의 연속성과 문맥을 유지하는 것이 필수적이며, LangChain은 `ConversationChain`을 기반으로 여러 메모리 옵션을 제공합니다.

#### 1-1. ConversationBufferMemory

- 이전 대화의 원본을 저장하고, 전부를 `{history}` 매개변수에 전달
- **장점**: 단순하고 직관적이며, 완전한 대화를 유지
- **단점**: 메모리 사용량이 증가하고, 대화가 길어질수록 처리 속도가 저하

#### 1-2. ConversationBufferWindowMemory

- k값을 정해서 최근 k개의 대화만 저장하고, 이를 `{history}` 매개변수에 전달
- **장점**: k개의 대화만 저장함으로써 메모리 사용 최적화 및 토큰 사용량 감소
- **단점**: 오래된 정보가 소실되며, 적절한 k값 설정이 필요

#### 1-3. ConversationTokenBufferMemory

- 대화의 개수가 아닌 토큰 길이를 기준으로 저장된 대화 내용을 정리(flush)
- **장점**: 토큰 길이를 기준으로 함으로써 LLM 입력의 토큰 제한을 넘지 않도록 조절 가능
- **단점**: `ConversationBufferWindowMemory`보다 긴 대화에서 좋은 성능을 보이지만 관리가 복잡

#### 1-4. ConversationSummaryMemory

- 대화의 기록을 요약해서 `{history}` 매개변수에 전달하는 방식
- **장점**: 과거 대화를 요약해서 정리하므로 LLM 입력 토큰 제한을 넘지 않음. 중요한 정보를 계속 유지 가능
- **단점**: 요약 품질이 LLM에 영향을 받으며, 요약 과정에서 디테일한 정보가 사라질 수 있음

#### 1-5. ConversationSummaryBufferMemory (선택)

- 과거의 대화를 기록하다가, 특정 토큰 수를 넘기면 요약해서 저장하는 방식
- **장점**: 효율적인 메모리 관리와 중요한 정보 보존 가능
- **단점**: 복잡한 관리가 필요하고, 요약 과정에서 LLM API 사용 비용이 발생

### 2. LangGraph Persistence

![langgraph_persistence](/img/cs/langgraph_persistence.avif)

- LangGraph는 LangChain의 상위 개념으로, 복잡한 에이전트 워크플로우를 그래프 구조로 표현
- LangGraph의 Persistence 기능은 그래프의 각 노드 실행 후 상태(state)를 자동으로 저장하여 멀티턴 대화를 구현

#### 2-1. LangGraph의 상태(State) 개념

LangGraph에서 상태는 그래프 전체에서 공유되는 데이터 구조입니다. 대화 기록, 중간 결과, 사용자 컨텍스트 등을 포함할 수 있습니다.

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]  # 대화 기록
    context: str  # RAG 검색 결과 등 추가 컨텍스트
```

- `add_messages`: 메시지 리스트를 자동으로 병합하는 reducer 함수
- 새 메시지가 추가될 때 기존 메시지와 합쳐져서 대화 기록 유지

#### 2-2. Checkpointer를 통한 상태 저장

![langgraph_memory](/img/cs/langgraph_memory_store.avif)

Checkpointer는 그래프 실행 중 상태를 저장하고 복원하는 역할을 합니다.

**MemorySaver (개발/테스트용)**

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

**PostgresSaver (프로덕션 환경)**

```python
from langgraph.checkpoint.postgres import PostgresSaver

with PostgresSaver.from_conn_string("postgresql://...") as memory:
    graph = graph_builder.compile(checkpointer=memory)
```

#### 2-3. Thread를 통한 세션 관리

LangGraph에서는 `thread_id`를 통해 사용자별 대화 세션을 구분합니다.

```python
config = {"configurable": {"thread_id": "user_123_session_1"}}

# 첫 번째 대화
response1 = graph.invoke(
    {"messages": [{"role": "user", "content": "Python이 뭐야?"}]},
    config=config
)

# 두 번째 대화 - 같은 thread_id로 이전 대화 컨텍스트 유지
response2 = graph.invoke(
    {"messages": [{"role": "user", "content": "그걸로 뭘 만들 수 있어?"}]},
    config=config
)
```

- 같은 `thread_id`를 사용하면 이전 대화 상태가 자동으로 로드됨
- 다른 `thread_id`를 사용하면 새로운 대화 세션으로 시작

#### 2-4. LangGraph 멀티턴 전체 흐름 예시

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI

# 1. 상태 정의
class State(TypedDict):
    messages: Annotated[list, add_messages]

# 2. 노드 함수 정의
def chatbot(state: State):
    llm = ChatOpenAI(model="gpt-4")
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# 3. 그래프 구성
graph_builder = StateGraph(State)
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)

# 4. Checkpointer 연결 및 컴파일
memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)

# 5. 대화 실행
config = {"configurable": {"thread_id": "session_001"}}
graph.invoke({"messages": [("user", "안녕하세요")]}, config=config)
graph.invoke({"messages": [("user", "제 이름은 철수입니다")]}, config=config)
graph.invoke({"messages": [("user", "제 이름이 뭐였죠?")]}, config=config)
# → "철수"라고 답변 (이전 대화 기억)
```

#### 2-5. LangChain Memory vs LangGraph Persistence

| 구분                  | LangChain Memory         | LangGraph Persistence                     |
| --------------------- | ------------------------ | ----------------------------------------- |
| **저장 단위**         | 전체 대화 사이클 완료 후 | 각 노드 실행 후 (단계별)                  |
| **재시작 위치**       | 항상 처음부터            | 특정 체크포인트에서 재개 가능             |
| **분기 처리**         | 단일 경로                | 조건부 분기, 다중 경로 지원               |
| **Human-in-the-loop** | 제한적                   | `interrupt_before/after`로 중간 개입 가능 |
| **적합한 사용 사례**  | 단순 Q&A 챗봇            | 복잡한 에이전트, 워크플로우               |

### 3. 저장소 선택: Redis

#### 3-1. Redis란?

Redis(Remote Dictionary Server)는 메모리 기반의 오픈소스 NoSQL 데이터베이스
Key-Value 형태로 데이터를 저장하며, 모든 데이터를 메모리에 저장하기 때문에 읽기/쓰기 속도가 빠름

**주요 특징**

- **In-Memory 데이터 저장**: 모든 데이터를 RAM에 저장하여 디스크 I/O 없이 빠른 응답 속도 제공
- **다양한 자료구조 지원**: String, List, Set, Hash, Sorted Set 등 다양한 자료구조를 기본 제공
- **싱글 스레드 기반**: 단일 스레드로 동작하여 Race Condition 없이 원자적(Atomic) 연산 보장
- **데이터 영속성 옵션**: RDB(스냅샷), AOF(Append Only File) 방식으로 디스크에 데이터 백업 가능
- **Pub/Sub 메시징**: 발행/구독 패턴을 통한 실시간 메시지 브로커 기능 제공

**주요 용도**

- **캐싱(Caching)**: DB 쿼리 결과, API 응답 등을 캐싱하여 서버 부하 감소
- **세션 저장소**: 분산 환경에서 사용자 세션 정보를 중앙에서 관리
- **실시간 데이터 처리**: 채팅, 알림, 실시간 순위표 등 빠른 응답이 필요한 서비스
- **메시지 큐**: 비동기 작업 처리를 위한 작업 큐로 활용

**Redis 데이터베이스 구조**

- Redis는 16개의 논리적 데이터베이스를 제공 (0번~15번, 기본값: 0번)
- 각 데이터베이스는 독립적인 key-value 저장소로 동작

#### 3-2. 대화 저장소 비교: In-Memory vs Redis vs RDBMS

| 구분            | In-Memory           | Redis                    | RDBMS                   |
| --------------- | ------------------- | ------------------------ | ----------------------- |
| **영속성**      | 서버 재시작 시 소실 | Redis 재시작 전까지 유지 | 완전한 영속성           |
| **속도**        | 빠름                | 빠름                     | 상대적으로 느림         |
| **확장성**      | 단일 인스턴스       | 다중 인스턴스 공유 가능  | 다중 인스턴스 공유 가능 |
| **적합한 용도** | 개발/테스트         | 실시간 채팅              | 장기 보관/분석          |

- `RedisChatMessageHistory`는 Redis를 재시작하기 전까지 대화 데이터를 영구적으로 저장할 수 있어, 사용자가 접속을 끊었다가 다시 접속해도 이전 대화를 유지 가능
- Redis 기반 메시지 기록을 사용하면 다중 인스턴스 환경에서도 일관된 대화 내역을 공유할 수 있어 확장성이 뛰어남
- RDBMS는 장기 보관이 필요한 데이터(사용자별 대화 분석, 로그 저장)에는 적합하지만, 실시간 메시지 처리에는 부적합할 수 있음
