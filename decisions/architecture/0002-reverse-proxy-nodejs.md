# 0002. NGINX 대신 Node.js(http-proxy-middleware)로 Reverse Proxy 구현

## 상태
승인됨

## 맥락
MSA 지향 구조([0001](./0001-msa-architecture.md))를 채택함에 따라 클라이언트의 모든 요청을 각 서비스로 적절히 분배하는 Reverse Proxy가 필요했다.

Reverse Proxy는 다음 역할을 수행해야 했다.
- HTTP 요청 라우팅 및 로드밸런싱
- WebSocket(Y.js, MediaSoup) 연결 중계 (HTTP Upgrade 헤더 처리)
- 서비스 헬스체크 및 장애 서버 자동 제외
- 동적 라우팅 테이블 관리

특히 WebSocket 중계가 핵심 요구사항이었는데, CRDT Server와 Voice Server가 WebSocket 기반으로 동작하기 때문이다.

## 고려한 옵션

### 옵션 A — NGINX
- 장점: C 기반 고성능, 낮은 메모리 사용, 검증된 프로덕션 안정성
- 단점: 설정 파일(nginx.conf) 기반이라 런타임 동적 라우팅 불가, 헬스체크 로직 커스터마이징 어려움, WebSocket 중계 시 별도 설정 필요(`proxy_http_version 1.1`, `Upgrade`, `Connection` 헤더 수동 처리)

### 옵션 B — Node.js + http-proxy-middleware (선택)
- 장점: 런타임 동적 라우팅 테이블 갱신 가능, 헬스체크 로직 자유롭게 커스터마이징, `ws: true` 옵션으로 WebSocket Upgrade 자동 처리, 기존 Node.js 서버들과 동일한 언어·생태계 공유로 유지보수 편의성
- 단점: NGINX 대비 고트래픽 처리 성능 낮음, Node.js 싱글 스레드 특성상 연결 수 증가 시 이벤트 루프 부하

### 옵션 C — AWS ALB (Application Load Balancer)
- 장점: 관리형 서비스, 고가용성, WebSocket 지원
- 단점: 예산 제약으로 도입 불가, 온프레미스 환경에서 활용 어려움

## 결정
Node.js + http-proxy-middleware로 Reverse Proxy를 직접 구현한다.

구현된 기능은 다음과 같다.
- HTTP / WebSocket 통합 중계
- 서비스별 헬스체크 (주기적 엔드포인트 호출)
- 헬스체크 결과 기반 동적 라우팅 테이블 갱신
- 장애 서버 자동 제외 및 복구 시 재편입

## 근거
헬스체크 결과에 따라 런타임에 라우팅 테이블을 갱신해야 하는 요구사항이 NGINX의 정적 설정 방식으로는 구현이 어려웠다. Node.js로 구현하면 헬스체크 로직과 라우팅 로직을 하나의 코드베이스에서 관리할 수 있고, 팀의 주력 언어인 JavaScript로 통일되어 유지보수가 용이하다.

## 트레이드오프
고트래픽 환경에서 NGINX 대비 처리량이 낮을 수 있다. 트래픽이 증가할 경우 Node.js `cluster` 모듈로 멀티 프로세스화하거나, AWS ALB로 교체하는 것을 고려한다.

## 결과
- HTTP와 WebSocket을 단일 Proxy에서 통합 처리
- 헬스체크 기반 자동 장애 대응 구현
- 예산 제약 하에서 LB 기능 확보
- 관련 결정: [0001](./0001-msa-architecture.md), [0010](../protocol/0010-voice-sfu-mediasoup.md), [0011](../protocol/0011-crdt-yjs.md)
