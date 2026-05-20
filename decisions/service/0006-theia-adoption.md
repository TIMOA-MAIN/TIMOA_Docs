# 0006. 코드 에디터 Monaco Editor + 자체 LSP → Eclipse Theia 전환

## 상태
승인됨

## 맥락
초기 코드 에디터는 Monaco Editor(VS Code의 에디터 엔진)를 직접 React에 통합하고, 인텔리센스를 위해 Eclipse LSP(Java)와 Clangd(C/C++)를 별도로 WebSocket으로 연결하는 방식으로 구현되었다.

이 구조의 한계는 다음과 같았다.
- 지원 언어를 추가할 때마다 별도 LSP 서버를 구현해야 함
- LSP 연결, 워크스페이스 생성, compile_commands.json 생성 등 언어별 설정 코드가 복잡함
- VS Code Extension 생태계를 활용할 수 없어 기능 확장이 어려움
- 유지보수 비용이 높고 언어 확장에 제약이 큼

## 고려한 옵션

### 옵션 A — Monaco Editor + 자체 LSP 유지
- 장점: 기존 구현 유지, 완전한 제어권
- 단점: 언어 추가 시마다 LSP 서버 직접 구현 필요, 유지보수 비용 지속 증가

### 옵션 B — Eclipse Theia 도입 (선택)
- 장점: VS Code Extension 생태계 그대로 활용, 언어별 LSP 내장 처리, VS Code와 유사한 UX 제공, 언어 제약 없음
- 단점: 프레임워크 의존성 증가, 커스터마이징 범위 제한, 초기 학습 비용

### 옵션 C — VS Code Server (code-server)
- 장점: 완전한 VS Code 환경
- 단점: 협업 플랫폼과의 통합 어려움, 라이선스 제약

## 결정
Eclipse Theia 프레임워크를 도입하여 코드 에디터를 전환한다.

## 근거
Theia는 VS Code와 동일한 Extension API를 지원하므로 기존 VS Code Extension을 그대로 활용할 수 있다. 언어별 LSP를 직접 구현하는 대신 Extension으로 처리하면 언어 확장이 대폭 단순해진다. VS Code와 유사한 UX를 제공하므로 사용자 진입장벽도 낮아진다.

## 트레이드오프
- 얻은 것: 언어 제약 없는 에디터, VS Code Extension 생태계, 유지보수 비용 감소
- 감수한 것: 프레임워크 의존성, 세부 커스터마이징 범위 제한

## 결과
- 자체 LSP 서버(Eclipse LSP, Clangd) 의존성 제거
- Java, C, C++ 외 다양한 언어 지원 가능
- VS Code와 유사한 개발 환경 제공
- 관련 결정: [0005](./0005-document-server-migration.md), [0008](./0008-stomp-removal.md)
