# 0005. 문서 서버 Java Spring Boot + STOMP → Node.js + Y.js CRDT 전환

## 상태
승인됨

## 맥락
초기 문서 서버는 Java Spring Boot + STOMP over WebSocket으로 구현되었다. STOMP는 WebSocket의 상위 프로토콜로 구독/발행 구조를 제공하며, Spring Boot 환경에서 HTTP Upgrade 처리가 간편하다는 이점이 있었다.

그러나 문서 기능의 고도화 요구사항이 생겼다.
- 여러 사용자가 동시에 같은 문서를 편집할 때 충돌 없이 병합되어야 함
- 커서 위치 실시간 공유
- 네트워크 단절 후 재연결 시 문서 상태 자동 복원

STOMP 기반 구조에서 동시 편집 충돌을 직접 해결하려면 OT(Operational Transformation) 알고리즘을 직접 구현해야 했는데, 이는 개발 복잡도가 매우 높았다.

## 고려한 옵션

### 옵션 A — Spring Boot + STOMP 유지 + OT 직접 구현
- 장점: 기존 서버 유지, 언어 통일성
- 단점: OT 알고리즘 직접 구현 복잡도 매우 높음, 유지보수 어려움, 검증되지 않은 충돌 해결 로직

### 옵션 B — Spring Boot + STOMP + Yjs (하이브리드)
- 장점: 기존 Spring Boot 유지
- 단점: Y.js는 Node.js 생태계에 최적화되어 있어 Java 환경에서 통합이 복잡하고 버전 호환성 문제 발생 가능

### 옵션 C — Node.js + Y.js CRDT (선택)
- 장점: Y.js가 Node.js 생태계 표준, y-websocket 라이브러리로 WebSocket 연결 자동 처리, CRDT 알고리즘이 충돌 해결을 자동화, React 클라이언트와 버전 호환성 높음
- 단점: 서버 언어 변경(Java → Node.js), 기존 Spring Boot 코드 재작성 필요

## 결정
문서 서버를 Node.js + Y.js CRDT로 전환한다. STOMP는 문서 서버에서 완전히 제거한다.

Y.js 핵심 구성은 다음과 같다.
- `Y.Doc`: 문서 상태를 관리하는 CRDT 객체
- `y-websocket`: WebSocket 연결 및 동기화 자동 처리
- `Y.Text`: 텍스트 편집을 위한 CRDT 타입
- `Y.Map`: 키-값 저장을 위한 CRDT 타입

## 근거
CRDT(Conflict-free Replicated Data Type)는 분산 환경에서 충돌 없는 병합을 알고리즘 수준에서 보장한다. OT를 직접 구현하는 것보다 검증된 라이브러리를 사용하는 것이 안정성과 개발 속도 면에서 모두 우월했다. Y.js는 Google Docs, Notion 등에서 사용하는 검증된 CRDT 구현체다.

Node.js로 전환한 이유는 Y.js가 Node.js 생태계에 최적화되어 있고, Java 환경에서의 통합은 불필요한 복잡성을 유발하기 때문이다.

## 트레이드오프
- 얻은 것: 충돌 없는 실시간 동시 편집, 커서 위치 공유, 자동 상태 복원
- 감수한 것: Java → Node.js 서버 재작성, 팀의 Node.js 학습 비용

## 결과
- 문서 서버가 Node.js + Y.js CRDT 기반으로 완전 전환
- STOMP 의존성 문서 서버에서 제거
- 실시간 동시 편집 기능 구현 완료
- 관련 결정: [0008](./0008-stomp-removal.md), [0011](../protocol/0011-crdt-yjs.md)
