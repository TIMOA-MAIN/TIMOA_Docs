# 전체 시스템 구성도

## 개요

TIMOA는 MSA 지향 아키텍처를 채택한 실시간 협업 플랫폼입니다. 각 기능 도메인이 독립 서비스로 분리되어 있으며, 서비스별 특성에 맞는 프로토콜과 기술 스택을 선택했습니다.

관련 결정: [0001 MSA 아키텍처 채택](../decisions/architecture/0001-msa-architecture.md) · [0004 프로토콜 혼용](../decisions/architecture/0004-protocol-mixed.md)

---

## 시스템 구성도

```mermaid
flowchart LR
  subgraph CLIENT["🖥️ Desktop Client · Electron + React"]
    direction TB
    C_AUTH["<b>Auth</b><br/>로그인 · 회원가입 · 인증"]
    C_SRV["<b>Server</b><br/>팀 서버 생성/수정/삭제 · 멤버 · 초대 코드"]
    C_DASH["<b>Dashboard Channel</b><br/>WBS · Kanban"]
    C_DOC["<b>Document Channel</b><br/>블록 편집 · PDF 내보내기"]
    C_CHAT["<b>Chat Channel</b><br/>채팅 · 파일 업/다운로드 · 멘션"]
    C_VOICE["<b>Voice Channel</b><br/>mediasoup-client"]
    C_CODE["<b>Code-Editor Channel</b><br/>프로젝트/파일 CRUD · 휴지통 · 실시간 동기화"]
    C_IDE["<b>TIMOA IDE</b> · Eclipse Theia<br/>컴파일 · 인텔리센스 · 다이어그램 플러그인"]
  end

  subgraph PROXY["🔀 Proxy Server · api.timoa.app"]
    direction TB
    P_REST["<b>API Gateway</b><br/>/proxy/{service}"]
    P_SSE["<b>SSE Routing</b><br/>/proxy/sse-{service}"]
    P_WS["<b>WebSocket Gateway</b><br/>Upgrade 중계"]
    P_LB["헬스체크 5s · Round-robin LB<br/>/proxy/user 공개 · 나머지 인증 경로"]
  end

  subgraph SERVER["⚙️ Services"]
    direction TB
    S_USER["<b>User Server</b> · Spring Boot<br/>회원 · 팀 서버 · 멤버 · JWT 발급"]
    S_DASH["<b>Dashboard Server</b> · Spring Boot<br/>WBS · Kanban · Excel 내보내기"]
    S_CHAT["<b>Chat Server</b> · Spring Boot<br/>메시지 · 멘션 · 첨부파일"]
    S_DOC["<b>Document Server</b> · Node.js<br/>문서 채널 · 블록 CRDT (Y.js)"]
    S_CODE["<b>Code Server</b> · Node.js<br/>프로젝트/파일 · 휴지통 · CRDT (Y.js)"]
    S_VOICE["<b>Voice Server</b> · MediaSoup SFU<br/>팀 서버별 Router · 참여자 상태"]
    S_DIAG["<b>Diagram Server</b><br/>코드 → 클래스 다이어그램 생성"]
  end

  subgraph DATA["🗄️ Data"]
    direction TB
    DB[("<b>MS SQL</b> · 공유 DB<br/>사용자 · 팀 서버/멤버 · 채팅<br/>대시보드 · 문서 채널/블록 · 코드 프로젝트/파일")]
    S3[("<b>AWS S3</b><br/>프로필 이미지 · 채팅 첨부 · 대용량 파일")]
  end

  %% Client → Proxy
  CLIENT -->|REST| P_REST
  CLIENT -.->|SSE| P_SSE
  CLIENT ==>|WebSocket| P_WS
  C_CODE -.->|IPC 실행| C_IDE
  C_IDE -->|Plugin · /proxy/diagram| P_REST

  %% Proxy → Services
  P_REST --> S_USER & S_DASH & S_CHAT & S_DOC & S_CODE & S_DIAG
  P_SSE -.-> S_USER & S_DASH & S_CHAT & S_DOC & S_CODE
  P_WS ==>|"Y.js · /collaboration/:room"| S_CODE
  P_WS ==>|"Y.js · /document/..."| S_DOC
  P_WS ==>|"Socket.IO · /voice"| S_VOICE
  C_VOICE <-->|"WebRTC · UDP/TCP 10000-10100"| S_VOICE

  %% Services → Data
  S_USER & S_DASH & S_CHAT & S_DOC & S_CODE & S_VOICE --> DB
  S_USER & S_CHAT --> S3

  %% 0-1 REST/SSE, 2 WS, 3 IPC, 4 Plugin
  linkStyle 0 stroke:#e75480,stroke-width:2px
  linkStyle 1 stroke:#2e9e5b,stroke-width:2px
  linkStyle 2 stroke:#2f6fdf,stroke-width:3px
  linkStyle 3 stroke:#888,stroke-width:1px
  linkStyle 4 stroke:#8e44ad,stroke-width:2px
  %% 5-10 REST, 11-15 SSE, 16-18 WS, 19 WebRTC
  linkStyle 5,6,7,8,9,10 stroke:#e75480,stroke-width:2px
  linkStyle 11,12,13,14,15 stroke:#2e9e5b,stroke-width:2px
  linkStyle 16,17,18 stroke:#2f6fdf,stroke-width:3px
  linkStyle 19 stroke:#2f6fdf,stroke-width:2px,stroke-dasharray:4
  %% 20-25 SQL, 26-27 S3
  linkStyle 20,21,22,23,24,25 stroke:#e67e22,stroke-width:2px
  linkStyle 26,27 stroke:#c0392b,stroke-width:2px

  classDef planned stroke-dasharray:5 5
  class S_DIAG planned
```

| 선 | 프로토콜 | 사용처 |
|---|---|---|
| 🩷 실선 | **REST** | 모든 CRUD 요청 (`/proxy/{service}`), 파일 업로드 (`/chat-upload`, `/user-upload`) |
| 💚 점선 | **SSE** | 서버 → 클라이언트 변경 알림 (User · Dashboard · Chat · Document · Code) |
| 💙 굵은 선 | **WebSocket** | Y.js CRDT 동기화 (Code · Document), Socket.IO 시그널링 (Voice) |
| 💙 점선 | **WebRTC** | 음성 미디어. Proxy를 거치지 않고 SFU로 직접 전송 |
| 🧡 | **SQL** | 단일 MS SQL 공유 DB ([ADR 0003](https://github.com/TIMOA-MAIN/TIMOA_Docs/blob/main/decisions/architecture/0003-shared-db.md)) |
| ❤️ | **S3** | User(프로필 이미지 · 파일), Chat(첨부파일) |
| 💜 | **Plugin** | TIMOA IDE 플러그인 → Diagram Server (`/proxy/diagram`) |

> Diagram Server는 Proxy에 라우팅(`/diagram`)만 등록되어 있고 이 조직에 레포가 없어 점선 테두리로 표시했습니다.

---

## 서비스 상세

### Proxy Server
| 항목 | 내용 |
|------|------|
| 언어 | Node.js |
| 라이브러리 | http-proxy-middleware |
| 역할 | 클라이언트 요청 라우팅, 로드밸런싱, 헬스체크, WebSocket Upgrade 중계 |
| 프로토콜 | HTTP (라우팅) + WebSocket Upgrade 중계 |
| 특이사항 | 런타임 동적 라우팅 테이블 갱신, 장애 서버 자동 제외·복구 시 재편입 |

> NGINX 대신 Node.js를 선택한 근거 → [0002](../decisions/architecture/0002-reverse-proxy-nodejs.md)

---

### Code Server
| 항목 | 내용 |
|------|------|
| 언어 | Node.js |
| 프레임워크 | Eclipse Theia |
| 실시간 동기화 | Y.js CRDT + y-websocket |
| 프로토콜 | WebSocket |
| 역할 | 실시간 코드 동시 편집, 인텔리센스(VS Code Extension), 코드 실행 |

> Eclipse Theia 채택 근거 → [0006](../decisions/service/0006-theia-adoption.md)  
> CRDT 채택 근거 → [0011](../protocol/0011-crdt-yjs.md)

---

### Document Server (CRDT)
| 항목 | 내용 |
|------|------|
| 언어 | Node.js |
| 라이브러리 | Y.js · y-websocket · Notionblock.js |
| 프로토콜 | WebSocket |
| 역할 | 블록 기반 문서 동시 편집, 커서 위치 공유, 문서 상태 영속화 |

> Spring Boot → Node.js 전환 근거 → [0005](../decisions/service/0005-document-server-migration.md)  
> Notionblock.js 채택 근거 → [0007](../decisions/service/0007-notionblock-adoption.md)

---

### Voice Server
| 항목 | 내용 |
|------|------|
| 언어 | Node.js |
| 라이브러리 | MediaSoup (C++ 코어) |
| 프로토콜 | WebRTC + WebSocket (시그널링) |
| 역할 | SFU 기반 다중 사용자 음성 통화 |
| 코덱 | OPUS 48kHz |
| 구조 | Transport · Producer · Consumer · Router · Room |

> SFU(MediaSoup) 채택 근거 → [0010](../protocol/0010-voice-sfu-mediasoup.md)

---

### Chat Server
| 항목 | 내용 |
|------|------|
| 언어 | Java |
| 프레임워크 | Spring Boot |
| 프로토콜 | REST (메시지 전송) + SSE (메시지 수신) |
| 역할 | 채널 채팅, 파일·이미지 첨부 |
| 파일 스토리지 | AWS S3 |

> REST + SSE 채택 근거 → [0009](../protocol/0009-chat-rest-sse.md)  
> S3 전환 근거 → [0013](../data/0013-file-s3-migration.md)

---

### Dashboard Server
| 항목 | 내용 |
|------|------|
| 언어 | Java |
| 프레임워크 | Spring Boot · JPA (Cascade) |
| 프로토콜 | REST (CRUD) + SSE (변경 이벤트 전파) |
| 역할 | WBS · 칸반보드 섹션·Row·카드 관리 및 실시간 동기화 |

> REST + SSE 채택 근거 → [0012](../protocol/0012-dashboard-rest-sse.md)

---

### User Server
| 항목 | 내용 |
|------|------|
| 언어 | Java |
| 프레임워크 | Spring Boot |
| 프로토콜 | REST |
| 역할 | 사용자 인증·계정 관리 |

---

## 프로토콜 요약

| 서비스 | 클라이언트 → 서버 | 서버 → 클라이언트 |
|--------|-------------------|-------------------|
| Code Server | WebSocket (Y.js) | WebSocket (Y.js CRDT diff) |
| Document Server | WebSocket (Y.js) | WebSocket (Y.js CRDT diff) |
| Voice Server | WebRTC + WebSocket(시그널링) | WebRTC(미디어) + WebSocket(시그널링) |
| Chat Server | REST POST | SSE push |
| Dashboard Server | REST (CRUD) | SSE push |
| User Server | REST | REST |

---

## 데이터 저장소

| 저장소 | 용도 | 대상 서비스 |
|--------|------|------------|
| MS SQL (공유) | 사용자·메시지·프로젝트·문서 정형 데이터 | Chat · Dashboard · User · Document |
| AWS S3 | 채팅 첨부파일 · 프로필 이미지 · 문서 이미지 | Chat · Document |

> S3 버킷 구조 → [0014](../data/0014-s3-unstructured-data.md)

---

## 현재 한계 및 개선 방향

현재 아키텍처의 주요 한계점과 개선 방향은 다음 두 문서를 참조하세요.

- 현재 구현 상태 → [`as-is.md`](./as-is.md)
- 목표 아키텍처 → [`to-be.md`](./to-be.md)
