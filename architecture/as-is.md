# 현재 구현된 아키텍처 (As-Is)

## 구현 완료 현황

현재 TIMOA는 7개 서비스로 구성된 MSA 지향 아키텍처로 운영되고 있습니다. 예산·인프라 제약으로 인해 완전한 MSA 원칙에 미달하는 부분이 존재하며, 이를 명시적으로 기록합니다.

---

## 현재 구성도

```
Desktop Client (Electron + React + TypeScript)
        │
        │ HTTP · WebSocket · WebRTC
        ▼
┌───────────────────────────────────┐
│          Proxy Server             │
│  (Node.js + http-proxy-middleware)│
│  - 헬스체크 기반 동적 라우팅        │
│  - WebSocket Upgrade 중계         │
│  - 장애 서버 자동 제외             │
└──┬────────┬────────┬──────────────┘
   │        │        │
   ▼        ▼        ▼
┌──────┐ ┌──────┐ ┌──────┐
│ Chat │ │Dash- │ │User  │     ◄─── REST + SSE (Java Spring Boot)
│Server│ │board │ │Server│
└──┬───┘ └──┬───┘ └──┬───┘
   │         │         │
   └────┬────┘         │
        ▼              ▼
   ┌─────────────────────┐
   │      MS SQL         │  ◄─── 단일 공유 DB (모든 Java 서비스 공유)
   │  (정형 데이터 전체)  │
   └─────────────────────┘

   ┌──────────────────────────────┐
   │        Code Server           │  ◄─── WebSocket (Node.js)
   │  Eclipse Theia + Y.js CRDT  │
   └──────────────┬───────────────┘
                  │
   ┌──────────────────────────────┐
   │      Document Server         │  ◄─── WebSocket (Node.js)
   │  Notionblock.js + Y.js CRDT │
   └──────────────┬───────────────┘
                  │
                  ▼
           ┌───────────┐
           │  MS SQL   │  ◄─── 문서 상태 영속화 (공유 DB)
           └───────────┘

   ┌──────────────────────────────┐
   │        Voice Server          │  ◄─── WebRTC + WebSocket (Node.js)
   │      MediaSoup SFU           │
   └──────────────────────────────┘

   ┌──────────────────────────────┐
   │          AWS S3              │
   │  chat/ · profiles/ · docs/  │  ◄─── 비정형 데이터 (파일, 이미지)
   └──────────────────────────────┘
```

---

## 서비스별 구현 상태

### ✅ Proxy Server (완성)
- Node.js + `http-proxy-middleware`로 직접 구현
- 주기적 헬스체크 엔드포인트 호출 → 동적 라우팅 테이블 갱신
- HTTP 및 WebSocket Upgrade 통합 중계
- 장애 서버 자동 제외, 복구 시 재편입

### ✅ Code Server (완성)
- Eclipse Theia 기반 IDE (VS Code Extension API 호환)
- Y.js CRDT + y-websocket으로 실시간 코드 동시 편집
- Monaco Editor 기반 에디터 (Theia 내장)
- Java · C · C++ 및 VS Code Extension으로 추가 언어 지원 가능
- 초기 Monaco Editor + 자체 LSP 구조에서 전환 ([0006](../decisions/service/0006-theia-adoption.md))

### ✅ Document Server (완성)
- Node.js + Y.js CRDT
- Notionblock.js 기반 블록 에디터 (슬래시 커맨드, 드래그 앤 드롭)
- 커서 위치 실시간 공유
- 네트워크 단절 후 자동 상태 복원
- 초기 Spring Boot + STOMP 구조에서 전환 ([0005](../decisions/service/0005-document-server-migration.md))

### ✅ Voice Server (완성)
- Node.js + MediaSoup (C++ 코어 SFU)
- Transport · Producer · Consumer · Router · Room 구조
- OPUS 48kHz 고음질 오디오
- 방화벽·기업 네트워크 환경 안정적 동작

### ✅ Chat Server (완성)
- Java Spring Boot, REST POST(전송) + SSE(수신)
- 채널 기반 메시지 관리
- AWS S3 연동 파일·이미지 첨부

### ✅ Dashboard Server (완성)
- Java Spring Boot + JPA, REST(CRUD) + SSE(이벤트 전파)
- WBS · 칸반보드 섹션·Row·카드 관리
- 섹션 삭제 시 하위 Row·Task JPA Cascade 자동 삭제 + SSE 전파

### ✅ User Server (완성)
- Java Spring Boot, REST API
- 사용자 인증 · 계정 관리

---

## 현재 아키텍처의 제약사항

### 1. 단일 공유 DB (MSA 원칙 미준수)

현재 모든 서비스가 단일 MS SQL 인스턴스를 공유합니다.

```
Chat Server  ──┐
Dashboard   ───┤
User Server ───┤──► MS SQL (단일 인스턴스)
Document    ───┘
```

**영향:**
- 단일 장애점: DB 장애 시 전체 서비스 영향
- 서비스 간 스키마 결합 (독립 배포 어려움)
- DB 수준 확장 불가

**근거:** 예산·온프레미스 인프라 제약 → [0003](../decisions/architecture/0003-shared-db.md)

---

### 2. 서비스 간 통신 — 직접 DB 조회

현재 서비스 간 데이터 공유가 필요할 때 공유 DB를 직접 조회합니다. Redis 기반 이벤트 버스가 없어 서비스 간 결합도가 높습니다.

---

### 3. 파일 접근 — S3 직접 접근 (CDN 미적용)

현재 클라이언트가 S3 URL에 직접 접근합니다. CloudFront CDN이 적용되지 않아 파일 전송 지연이 발생할 수 있습니다.

---

### 4. 컨테이너화 미적용

서비스별 컨테이너(Docker)화 및 오케스트레이션(Kubernetes)이 적용되지 않아 배포·확장이 수동으로 이루어집니다.

---

### 5. STOMP 완전 제거 완료 ✅

초기 채팅·문서 서버에서 사용하던 STOMP가 완전히 제거되었습니다. 현재 통신 스택은 REST / SSE / WebSocket(Y.js) / WebRTC(MediaSoup)로 정리되었습니다. → [0008](../decisions/service/0008-stomp-removal.md)

---

## 현재 통신 프로토콜 현황

| 서비스 | 프로토콜 | 상태 |
|--------|----------|------|
| Code Server | WebSocket (Y.js CRDT) | ✅ 운영 중 |
| Document Server | WebSocket (Y.js CRDT) | ✅ 운영 중 |
| Voice Server | WebRTC + WebSocket(시그널링) | ✅ 운영 중 |
| Chat Server | REST + SSE | ✅ 운영 중 |
| Dashboard Server | REST + SSE | ✅ 운영 중 |
| User Server | REST | ✅ 운영 중 |
| Proxy Server | HTTP + WebSocket Upgrade 중계 | ✅ 운영 중 |

---

## 개선 방향

현재 아키텍처의 제약사항을 해소하는 목표 아키텍처는 [`to-be.md`](./to-be.md)를 참조하세요.
