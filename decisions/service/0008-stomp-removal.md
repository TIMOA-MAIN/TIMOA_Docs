# 0008. STOMP 완전 제거 결정

## 상태
승인됨

## 맥락
초기 TIMOA는 채팅과 문서 서버에서 STOMP(Simple Text Oriented Messaging Protocol)를 사용했다. STOMP는 WebSocket의 상위 프로토콜로 구독/발행 구조를 제공하고, Spring Boot 환경에서 HTTP Upgrade 처리가 간편하다는 이점이 있었다.

고도화 과정에서 각 서비스의 통신 방식이 변경되면서 STOMP의 사용처가 점차 줄어들었다.

- 문서 서버: Spring Boot + STOMP → Node.js + Y.js CRDT로 전환 ([0005](./0005-document-server-migration.md))
- 채팅 서버: STOMP → REST + SSE로 전환 ([0009](../protocol/0009-chat-rest-sse.md))
- 코드 에디터: Theia 도입으로 자체 LSP 제거 ([0006](./0006-theia-adoption.md))

WebSocket Upgrade가 필요한 서비스는 Y.js(y-websocket)와 MediaSoup이 각자의 라이브러리에서 직접 처리하게 되어, STOMP가 해결하던 문제가 다른 방식으로 대체되었다.

## 고려한 옵션

### 옵션 A — STOMP 일부 유지
- 장점: 기존 코드 일부 재활용
- 단점: 사용처가 없는 의존성 유지, 불필요한 복잡성

### 옵션 B — STOMP 완전 제거 (선택)
- 장점: 의존성 단순화, 시스템 이해 용이, 불필요한 Spring Boot STOMP 설정 제거
- 단점: 기존 STOMP 관련 코드 전면 재작성 필요

## 결정
STOMP를 시스템에서 완전히 제거한다.

현재 WebSocket Upgrade 처리는 다음 두 곳에서만 사용된다.
- Y.js (y-websocket): CRDT 문서 동기화
- MediaSoup: SFU 기반 음성 통화

두 경우 모두 각 라이브러리가 WebSocket Upgrade를 자체적으로 처리하므로 STOMP가 불필요하다.

## 근거
STOMP를 도입했던 이유는 WebSocket Upgrade의 편의성이었지만, 고도화 과정에서 각 서비스가 더 적합한 방식으로 교체되면서 STOMP의 존재 이유가 사라졌다. 사용하지 않는 프로토콜 레이어를 유지하는 것은 불필요한 복잡성을 유발한다.

## 트레이드오프
- 얻은 것: 의존성 단순화, 시스템 통신 구조 명확화
- 감수한 것: 기존 STOMP 기반 코드 재작성 비용

## 결과
- Spring Boot STOMP 설정 및 관련 코드 완전 제거
- 시스템 통신 스택이 REST, SSE, WebSocket(Y.js), WebRTC로 정리됨
- 관련 결정: [0005](./0005-document-server-migration.md), [0009](../protocol/0009-chat-rest-sse.md), [0011](../protocol/0011-crdt-yjs.md)
