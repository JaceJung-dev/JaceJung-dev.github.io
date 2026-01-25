---
layout: post
title: "WebSocket, Gateway Interface, Channel Layer"
date: 2025-03-21 22:00:00 +0900
categories:
  - studyarchive
  - cs
notice: |
  ⚠️ **안내:** 이 글은 학습 기록용입니다. 오류나 보완 의견은 댓글로 알려주세요.
---

# WebSocket, Gateway Interface, Channel Layer

## 1. WebSocket

### 1-1. HTTP (HyperText Transfer Protocol)

- HTTP는 웹에서 가장 기본이 되는 애플리케이션 계층 프로토콜
- 브라우저(클라이언트)와 서버가 요청(Request) -> 응답(Response) 형태로 통신하도록 설계되어 있음
  - 클라이언트가 먼저 요청을 보낸다.
  - 서버는 그 요청에 대한 응답을 반환한다.
  - 연결은 요청 단위로 짧게 끝난다.
- HTTP의 장단점:

| 구분 | 내용                                                                                        |
| ---- | ------------------------------------------------------------------------------------------- |
| 강점 | 문서·리소스 조회, REST API 등 **요청이 왔을 때만 처리하면 되는 구조**에 적합                |
| 약점 | 서버가 먼저 데이터를 보내야 하거나 **연속적인 대화·실시간 스트리밍**이 필요한 경우에 부적합 |

### 1-2. 실시간을 HTTP로 억지로 만들면: Polling

![polling](/img/cs/polling.gif)

_출처: https://yong-nyong.tistory.com/90_

- 클라이언트의 요청 없이 서버가 먼저 응답을 보낼 수 없기 때문에 클라이언트가 주기적으로 서버에 요청을 보내는 방식
  - 클라이언츠가 일정 주기(예: 1초마다)로 서버에 요청을 보냄
  - 서버는 "새로운 데이터가 있든 없든" 응답을 반환
- 서버 리소스 낭비 및 네트워크 비용 증가
  - 데이터가 없을 때도 요청이 계속 발생 -> 불필요한 트래픽
  - 요청은 곧 비용(서버 처리/네트워크/로드밸런서/로그)
- long polling 등 보완 기법이 존재하지만 구조적 한계는 유지
  - 클라이언트가 요청을 보냄
  - 서버는 **바로 응답하지 않고 대기**
  - 새로운 이벤트가 생기면 그 때 응답 (일정시간동안 이벤트가 없는 경우 타임아웃으로 응답)
  - 클라이언트는 응답을 받자마자 다시 요청을 보냄
- long polling은 연결을 오래 잡고 있어야 하므로 동시 접속자 수가 많으면 서버/프록시 자원의 부담이 증가

## 1-3. WebSocket: 실시간 위한 프로토콜

![websocket](/img/cs/websocket.gif)

_출처: https://yong-nyong.tistory.com/90_

- **WebSocket은 프로토콜의 한 종류**로 HTTP와 마찬가지로 네트워크 상에서 통신 규칙을 정의
- 실시간을 위해 설계된 프로토콜
  - HTTP처럼 요청-응답을 반복하는 대신 **한 번 연결을 수립(Handshake) 한 뒤, 연결을 끊지 않고 데이터를 주고 받는 방식
    즉, **한 번 연결되면 연결을 끊지 않고 데이터를 계속 주고받을 수 있다.\*\*

#### WebSocket이 제공하는 통신 특성

- **지속 연결(Persistent Connection)**: 연결을 매번 새로 만드는 대신 “대화 채널”을 유지
- **양방향(Full-Duplex)**: 클라이언트도 서버도 서로 먼저 메시지를 보낼 수 있음
- **서버 Push가 기본값**: 이벤트가 생기면 서버가 즉시 전달 가능
- **실시간 UX에 적합한 메시지 모델**: “요청/응답”이 아니라 “메시지/이벤트” 중심

#### Handshake (HTTP Upgrade)

WebSocket은 처음부터 완전히 새로운 네트워크가 아니라, 웹 생태계(브라우저, 프록시, 방화벽)를 통과하기 위해 **HTTP 업그레이드 방식** 을 사용한다.

- 클라이언트는 HTTP 요청을 통해 WebSocket 업그레이드 시도
- 요청 헤더에 다음 정보를 포함
  - `Upgrade: websocket`
  - `Connection: Upgrade`
- 서버가 이를 수락하면 `HTTP 101 Switching Protocols` 응답 반환
- 이후 HTTP → WebSocket 프로토콜로 전환

👉 이 과정을 **Handshake**라고 함

## 2. Gateway Interface

### 2-1. Server Gateway Interface(SGI)란?

![wsgi](/img/cs/wsgi.png)

- 웹 서버와 웹 애플리케이션이 **어떻게 통신하는지 정의한 표준 규약**
- HTML, CSS, JS, 이미지, 동영상 같이 동적 파일(static files)은 웹서버(Nginx)에서 처리
- 애플리케이션의 데이터베이스 쿼리, API처리, Websocket 통과 같은 동적 요청(Dynamic Requests)는 WSGI/ASGI 서버에서 처리

### 2-2. WSGI vs ASGI

| 구분             | WGI                          | ASGI                                  |
| ---------------- | ---------------------------- | ------------------------------------- |
| 정의             | Web Server Gateway Interface | Asynchronous Server Gateway Interface |
| Django 실행 방식 | 전통적인 실행 방식           | 차세대 실행 인터페이스                |
| 처리 모델        | 동기(Synchronous)            | 비동기(Asynchronous)                  |
| 지원 프로토콜    | HTTP 전용                    | HTTP, WebSocket, 기타 비동기 프로토콜 |
| 연결 처리 방식   | 요청당 처리, 블로킹 구조     | 이벤트 루프 기반 다중 연결 처리       |
| WebSocket 지원   | ❌ 불가                      | ✅ 가능                               |
| 실시간 서비스    | ❌ 부적합                    | ✅ 적합                               |
| 확장성           | 요청 증가 시 제한적          | 대규모·실시간 서비스에 유리           |
| 대표 사용 사례   | REST API, 서버 렌더링        | 실시간 채팅, 스트리밍, 알림 시스템    |

## 3. Channel Layer

### 3-1. Channel Layer란?

![channel_layer](/img/cs/channel_layer.png)

- Channel Layer는 **실시간 애플리케이션에서 서버 내부에 고립된 연결들을 논리적으로 연결하기 위한 메시지 전달 추상화 계층**
- 실제 클라이언트는 서버랑 일대일로 통신하고, 서버랑 통신한 내용은 다른 WebSocket들이 알 수 없음
  - 서버 A에 연결된 클라이언트는 서버 B의 클라이언트와 직접 통신 불가
- 즉, 각각 따로 동작하는 WebSocket 연결들이 메시지를 주고받을 수 있게 해주는 공용 메시지 통로 같은 것
  - WebSocket의 연결 자체를 관리하는 것은 아님
  - 메시지를 **어디로, 누구에게 전달할지**에 대한 규칙만 정의
    - 구현 기술은 아니고 개념적 계층(인터페이스)임

### 3-2. 메시지 브로커

- Channel Layer가 요구하는 기능을 실제로 수행하는 존재
- 메시지를 중간에서 받아서 서버 간 전달
- Redis를 메시지 브로커로 사용할 수도 있음
