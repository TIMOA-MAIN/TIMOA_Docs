# Architecture Decision Records

TIMOA 프로젝트의 주요 아키텍처 결정 기록입니다.
설계부터 구현까지의 기술 선택 근거와 트레이드오프를 기록합니다.

---

## 전체 목록

### 아키텍처 레벨

| 번호 | 제목 | 상태 | 담당 |
|------|------|------|------|
| [0001](./architecture/0001-msa-architecture.md) | MSA 지향 아키텍처 채택 | 승인됨 | 양진형 |
| [0002](./architecture/0002-reverse-proxy-nodejs.md) | NGINX 대신 Node.js로 Reverse Proxy 구현 | 승인됨 | 양진형 |
| [0003](./architecture/0003-shared-db.md) | 단일 공유 DB 유지 결정 | 미완성 | 양진형 |
| [0004](./architecture/0004-protocol-mixed.md) | 서비스별 통신 프로토콜 혼용 결정 | 승인됨 | 양진형 |

### 서비스 전환 레벨

| 번호 | 제목 | 상태 | 담당 |
|------|------|------|------|
| [0005](./service/0005-document-server-migration.md) | 문서 서버 Spring Boot + STOMP → Node.js + Y.js CRDT 전환 | 승인됨 | 양진형 |
| [0006](./service/0006-theia-adoption.md) | 코드 에디터 Monaco + 자체 LSP → Eclipse Theia 전환 | 승인됨 | 정찬호 |
| [0007](./service/0007-notionblock-adoption.md) | 문서 에디터 단순 React → Notionblock.js 전환 | 승인됨 | 양진형 |
| [0008](./service/0008-stomp-removal.md) | STOMP 완전 제거 결정 | 승인됨 | 양진형 |

### 통신 프로토콜 레벨

| 번호 | 제목 | 상태 | 담당 |
|------|------|------|------|
| [0009](./protocol/0009-chat-rest-sse.md) | 채팅에 WebSocket 대신 REST + SSE 채택 | 승인됨 | 정인성 |
| [0010](./protocol/0010-voice-sfu-mediasoup.md) | 음성 통화에 P2P 대신 SFU(MediaSoup) 채택 | 승인됨 | 양진형 |
| [0011](./protocol/0011-crdt-yjs.md) | 실시간 동시 편집에 CRDT(Y.js) 채택 | 승인됨 | 양진형 |
| [0012](./protocol/0012-dashboard-rest-sse.md) | 대시보드 동기화에 REST + SSE 채택 | 승인됨 | 양진형 |

### 데이터 레벨

| 번호 | 제목 | 상태 | 담당 |
|------|------|------|------|
| [0013](./data/0013-file-s3-migration.md) | 파일 저장 Base64 + MS SQL → AWS S3 전환 | 승인됨 | 양진형 |
| [0014](./data/0014-s3-unstructured-data.md) | 비정형 데이터 AWS S3 버킷 분리 결정 | 승인됨 | 양진형 |

### 외부 연동 레벨

| 번호 | 제목 | 상태 | 담당 |
|------|------|------|------|
| [0015](./integration/0015-notion-api-fallback.md) | Notion API 연동 시도 → API Key 제약으로 내부 구현 전환 | 대체됨 | 양진형 |

---

## 상태 정의

| 상태 | 설명 |
|------|------|
| 승인됨 | 결정이 채택되어 현재 시스템에 반영됨 |
| 미완성 | 결정은 됐으나 구현이 완료되지 않음 |
| 대체됨 | 다른 결정이나 방향으로 대체됨 |
| 폐기됨 | 검토 후 채택하지 않기로 결정됨 |
