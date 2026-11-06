# Code Server

## 개요

Code Server는 여러 사용자가 동시에 같은 코드를 편집할 수 있는 실시간 협업 코드 에디터 서비스입니다. VS Code와 동일한 개발 경험을 웹 기반으로 제공하면서, Y.js CRDT를 통해 충돌 없는 동시 편집을 구현합니다.

---

## 기술 스택

| 항목 | 기술 |
|------|------|
| 런타임 | Node.js |
| IDE 프레임워크 | Eclipse Theia |
| 에디터 엔진 | Monaco Editor (Theia 내장) |
| 실시간 동기화 | Y.js CRDT + y-websocket |
| 프로토콜 | WebSocket |
| 언어 지원 | Java · C · C++ (VS Code Extension으로 확장 가능) |

---

## 아키텍처

```
Desktop Client
      │
      │ WebSocket (Y.js 동기화)
      │ WebSocket (Theia LSP 통신)
      ▼
┌────────────────────────────────────────────────┐
│                 Code Server                    │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │          Eclipse Theia Backend           │  │
│  │  - VS Code Extension API 호환            │  │
│  │  - 언어별 LSP 내장 처리                   │  │
│  │  - 파일 시스템 접근                        │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │          Y.js CRDT 동기화 레이어          │  │
│  │  - Y.Doc: 문서 상태 CRDT 객체            │  │
│  │  - Y.Text: 텍스트 편집 CRDT 타입         │  │
│  │  - y-websocket: WebSocket 연결 처리       │  │
│  │  - 커서 위치 Awareness 공유              │  │
│  └──────────────────────────────────────────┘  │
│                                                │
└────────────────────────────────────────────────┘
      │
      ▼
   MS SQL (편집 내역 영속화)
```

---

## 핵심 기능

### 1. 실시간 동시 편집 (CRDT)

Y.js CRDT를 사용하여 여러 사용자가 동시에 같은 파일을 편집할 때 충돌 없이 병합합니다.

- **충돌 없는 병합** — CRDT 알고리즘이 수학적으로 보장
- **커서 위치 공유** — Awareness 프로토콜로 각 사용자의 커서 실시간 표시
- **자동 복원** — 네트워크 단절 후 재연결 시 문서 상태 자동 동기화

```
사용자 A: "Hello |" (커서: 6번째)
사용자 B: "He|llo" (커서: 2번째)
        ↓ 동시 편집
사용자 A: "Hello !" 입력
사용자 B: "yHello !" (y 삽입)

Y.js CRDT 병합 결과:
"yHello !"  ← 충돌 없이 자동 병합
```

CRDT 채택 근거 → [0011 CRDT(Y.js) 채택](../protocol/0011-crdt-yjs.md)

---

### 2. Eclipse Theia IDE

VS Code Extension API를 완전히 지원하는 Eclipse Theia 프레임워크를 기반으로 합니다.

**지원 기능:**
- Monaco Editor 기반 코드 편집 (VS Code와 동일한 에디터 엔진)
- VS Code Extension으로 언어 지원 추가 가능 (추가 LSP 서버 직접 구현 불필요)
- 인텔리센스 (자동 완성, 타입 힌트, 오류 표시)
- 구문 하이라이팅, 코드 접기
- VS Code와 유사한 단축키 및 UX

**지원 언어 (기본):**
| 언어 | LSP | 비고 |
|------|-----|------|
| Java | Eclipse JDT Language Server | 내장 |
| C | Clangd | Extension |
| C++ | Clangd | Extension |
| JavaScript / TypeScript | tsserver | Extension |

Theia 채택 근거 → [0006 Eclipse Theia 전환](../decisions/service/0006-theia-adoption.md)

---

### 3. WebSocket 기반 연결

Y.js의 `y-websocket` 라이브러리를 통해 WebSocket 연결을 처리합니다.

```
클라이언트 A ──WS──► Code Server ──WS──► 클라이언트 B
                        │
                   Y.js CRDT
                   diff 계산 및 전파
```

Proxy Server가 WebSocket Upgrade 헤더를 중계하여 클라이언트가 Code Server에 WebSocket으로 연결됩니다. → [0002 Reverse Proxy Node.js 구현](../decisions/architecture/0002-reverse-proxy-nodejs.md)

---

## 초기 설계 대비 변경 이력

| 항목 | 초기 설계 | 현재 구현 |
|------|-----------|-----------|
| IDE 프레임워크 | Monaco Editor (직접 통합) | Eclipse Theia |
| 인텔리센스 | Eclipse LSP (Java) + Clangd (C/C++) 직접 구현 | Theia Extension API |
| 언어 추가 | 별도 LSP 서버 구현 필요 | VS Code Extension |
| 동시 편집 | — (미구현) | Y.js CRDT |

> 전환 근거 → [0006 Eclipse Theia 전환](../decisions/service/0006-theia-adoption.md)

---

## Y.js 핵심 구성

```javascript
// 문서 상태 관리
const ydoc = new Y.Doc()

// 텍스트 편집용 CRDT 타입
const ytext = ydoc.getText('content')

// WebSocket 기반 실시간 동기화
const provider = new WebsocketProvider(
  'ws://code-server',
  roomName,  // 채널 ID (파일 경로)
  ydoc
)

// 커서 위치 공유 (Awareness)
provider.awareness.setLocalStateField('user', {
  name: userName,
  color: userColor,
  cursor: cursorPosition
})

// Monaco Editor 연동
const monacoBinding = new MonacoBinding(
  ytext,
  monacoEditor.getModel(),
  new Set([monacoEditor]),
  provider.awareness
)
```

---

## 연관 결정

| ADR | 내용 |
|-----|------|
| [0001](../decisions/architecture/0001-msa-architecture.md) | MSA 지향 아키텍처 — Code Server 독립 서비스 분리 |
| [0004](../decisions/architecture/0004-protocol-mixed.md) | WebSocket 프로토콜 선택 |
| [0006](../decisions/service/0006-theia-adoption.md) | Monaco + 자체 LSP → Eclipse Theia 전환 |
| [0011](../protocol/0011-crdt-yjs.md) | 실시간 동시 편집에 CRDT(Y.js) 채택 |
