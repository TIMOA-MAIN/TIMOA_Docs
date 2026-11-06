# 목표 아키텍처 (To-Be)

## 개요

현재 아키텍처([as-is.md](./as-is.md))의 주요 제약사항—단일 공유 DB, Redis 부재, CDN 미적용, 컨테이너화 미적용—을 해소하는 목표 아키텍처를 기술합니다.

관련 결정: [0001 MSA 아키텍처 채택](../decisions/architecture/0001-msa-architecture.md) · [0003 단일 공유 DB 유지 결정 (임시)](../decisions/architecture/0003-shared-db.md)

---

## 목표 구성도

```
Desktop Client (Electron + React + TypeScript)
        │
        │ HTTP · WebSocket · WebRTC
        ▼
┌───────────────────────────────────────────────────────────┐
│                     Proxy Server                          │
│            (Node.js 또는 AWS ALB로 교체)                  │
└──┬──────────┬──────────┬──────────┬──────────┬────────────┘
   │          │          │          │          │
   ▼          ▼          ▼          ▼          ▼
┌──────┐  ┌──────┐  ┌──────┐  ┌──────────────────────┐  ┌──────┐
│ Chat │  │Dash- │  │User  │  │   Code / Document    │  │Voice │
│Server│  │board │  │Server│  │      Server          │  │Server│
│      │  │      │  │      │  │  (Node.js + Y.js)    │  │      │
└──┬───┘  └──┬───┘  └──┬───┘  └──────────┬───────────┘  └──┬───┘
   │          │          │                │                   │
   ▼          ▼          ▼                ▼                   ▼
┌──────┐  ┌──────┐  ┌──────┐  ┌────────────────┐    ┌──────────┐
│Mongo │  │MSSQL │  │MSSQL │  │Redis (세션·diff)│    │Redis     │
│DB    │  │(독립)│  │(독립)│  └───────┬────────┘    │(Room 상태)│
│(채팅)│  │      │  │      │          │              └──────────┘
└──────┘  └──────┘  └──────┘    ┌─────▼──────┐
                                 │  MS SQL    │
                                 │(문서 영속화)│
                                 └────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Redis Event Bus                          │
│         서비스 간 이벤트 통신 (Pub/Sub)                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              AWS S3  +  CloudFront CDN                      │
│       chat/ · profiles/ · docs/  ← 파일 캐싱·배포           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│          Kubernetes (컨테이너 오케스트레이션)                 │
│   각 서비스 Docker 컨테이너 · 독립 배포 · 오토스케일링        │
└─────────────────────────────────────────────────────────────┘
```

---

## 개선 항목

### 1. DB 분리 (서비스별 독립 DB)

현재 모든 서비스가 단일 MS SQL을 공유하는 구조에서 서비스별 최적 DB로 분리합니다.

| 서비스 | 현재 | 목표 | 이유 |
|--------|------|------|------|
| Chat Server | MS SQL (공유) | MongoDB | 메시지 R/W 빈번, 스키마 유연성 |
| Dashboard Server | MS SQL (공유) | MS SQL (독립 인스턴스) | 정형 데이터 유지, 장애 격리 |
| User Server | MS SQL (공유) | MS SQL (독립 인스턴스) | 보안 격리, 독립 배포 |
| Document Server | MS SQL (공유) | Redis(세션) + MS SQL(영속화) | 휘발성 CRDT 세션은 Redis, 저장본은 SQL |
| Voice Server | 메모리 | Redis | Room·Transport 상태 서버 재시작 후 복원 |

**분리 순서 (우선순위):**
1. Chat Server → MongoDB (가장 높은 R/W 빈도)
2. Document/Code Server → Redis 세션 도입
3. Dashboard Server → 독립 MS SQL 인스턴스
4. User Server → 독립 MS SQL 인스턴스
5. Voice Server → Redis Room 상태

---

### 2. Redis 기반 서비스 간 이벤트 통신

현재 서비스 간 통신이 공유 DB 직접 조회로 이루어지는 구조에서, Redis Pub/Sub를 이벤트 버스로 도입합니다.

**활용 시나리오:**
- 채팅 메시지 → 알림 서비스에 이벤트 전파
- 사용자 상태(온라인/오프라인) 변경 → 전체 서비스 동기화
- Dashboard 변경 이벤트 → 연관 서비스 캐시 무효화

```
Chat Server ──publish──► Redis (채널: chat.message)
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
             Notification          Dashboard
               Service               Service
            (구독: chat.message)  (구독: chat.message)
```

---

### 3. AWS CloudFront CDN 연동

현재 클라이언트가 S3 URL에 직접 접근하는 구조에서, CloudFront를 S3 앞단에 배치합니다.

```
현재: Client ──────────────────────► AWS S3

목표: Client ──► CloudFront (CDN) ──► AWS S3
                  (엣지 캐싱)
```

**기대 효과:**
- 파일 전송 지연 감소 (엣지 캐싱)
- S3 직접 노출 제거 (보안 개선)
- 대역폭 비용 절감

---

### 4. 컨테이너화 (Docker + Kubernetes)

각 서비스를 Docker 컨테이너로 패키징하고 Kubernetes로 오케스트레이션합니다.

```
┌─────────────────────────────────────────────┐
│              Kubernetes Cluster              │
│                                             │
│  ┌────────┐ ┌────────┐ ┌────────┐          │
│  │ Chat   │ │ Doc    │ │ Voice  │  ...      │
│  │Pod × N │ │Pod × N │ │Pod × N │          │
│  └────────┘ └────────┘ └────────┘          │
│                                             │
│  - 독립 배포 (서비스별 롤링 업데이트)         │
│  - 오토스케일링 (HPA)                       │
│  - 헬스체크 기반 자동 재시작                  │
└─────────────────────────────────────────────┘
```

**컨테이너화 순서:**
1. Node.js 서비스 (Proxy, Document, Voice, Code) — 의존성 단순
2. Java 서비스 (Chat, Dashboard, User) — JVM 튜닝 필요
3. DB 마이그레이션 완료 후 Kubernetes로 전환

---

### 5. Proxy Server 개선 (AWS ALB 전환 검토)

현재 Node.js로 직접 구현한 Reverse Proxy를 고트래픽 대응을 위해 AWS ALB(Application Load Balancer)로 전환을 검토합니다.

| | Node.js Proxy (현재) | AWS ALB (목표) |
|---|---|---|
| 처리량 | Node.js 이벤트 루프 한계 | 관리형 서비스, 고가용성 |
| WebSocket | ✅ 지원 | ✅ 지원 |
| 동적 라우팅 | ✅ 런타임 갱신 | 타겟 그룹으로 대체 |
| 헬스체크 | 직접 구현 | 내장 헬스체크 |
| 비용 | 서버 비용 | ALB 사용 비용 |

---

## 개선 로드맵

```
Phase 1 (단기)
├── Redis 도입 (Document/Voice 세션 관리)
└── CloudFront CDN S3 앞단 연결

Phase 2 (중기)
├── Chat Server → MongoDB 전환
├── Redis Event Bus 도입 (서비스 간 이벤트)
└── 각 서비스 Dockerfile 작성 및 컨테이너화

Phase 3 (장기)
├── 나머지 서비스 독립 DB 분리
├── Kubernetes 클러스터 전환
└── Proxy Server → AWS ALB 교체 검토
```

---

## 목표 아키텍처 달성 시 기대 효과

| 항목 | 현재 | 목표 |
|------|------|------|
| DB 장애 격리 | 단일 장애점 | 서비스별 독립 |
| 서비스 간 결합 | 공유 DB 직접 조회 | Redis Pub/Sub 이벤트 |
| 파일 전송 | S3 직접 | CloudFront 캐싱 |
| 배포 | 수동 | Kubernetes 롤링 업데이트 |
| 확장성 | 수동 스케일링 | HPA 오토스케일링 |
| MSA 완성도 | MSA 지향 | 완전한 MSA |
