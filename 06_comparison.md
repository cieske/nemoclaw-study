# 챕터 06: OpenClaw vs Nemoclaw 비교

> 두 프로젝트의 차이를 정리하고, Nemoclaw가 무엇을 개선했는지 알아봅니다.

---

## 1. 한눈에 보는 비교

| 항목 | OpenClaw | Nemoclaw |
|------|----------|----------|
| **목적** | 강력한 AI 개인 비서 | OpenClaw + 기업급 보안 |
| **개발사** | 오픈소스 커뮤니티 | NVIDIA |
| **상태** | 프로덕션 사용 가능 | 얼리 프리뷰 (알파) |
| **기본 보안 정책** | 허용(permit) 기본 | 거부(deny) 기본 |
| **샌드박스** | 소프트웨어적 제한 | 커널 수준 Landlock |
| **개인정보 보호** | 수동 설정 필요 | Privacy Router 내장 |
| **행동 검증** | 없음 | Intent Verification |
| **네트워크 제어** | 설정 기반 | 기본 차단 + 화이트리스트 |
| **GPU 최적화** | 없음 | NVIDIA GPU 특화 |
| **Docker 사용** | 선택적 | 필수 (k3s 클러스터) |
| **설치 복잡도** | 낮음 | 높음 |
| **학습 곡선** | 완만 | 가파름 |

---

## 2. 보안 철학의 차이

### OpenClaw의 접근법: "일단 허용, 필요하면 차단"

```
[OpenClaw 방식]

기본 상태: 모든 것 허용 ←─── 시작점

사용자가 명시적으로 차단 규칙 추가
        ↓
차단 규칙이 있는 것만 차단

문제점:
  - 미처 생각하지 못한 경로로 공격 가능
  - 새로운 공격 방법이 나오면 막을 수 없음
  - 보안 설정을 잘 모르면 취약한 상태로 운영
```

### Nemoclaw의 접근법: "일단 차단, 필요하면 허용"

```
[Nemoclaw 방식]

기본 상태: 모든 것 차단 ←─── 시작점

관리자가 명시적으로 허용 규칙 추가
        ↓
허용 규칙이 있는 것만 허용

장점:
  - 예상치 못한 공격 경로가 기본적으로 차단
  - 새로운 공격 방법도 화이트리스트에 없으면 차단
  - 보안 설정을 몰라도 기본적으로 안전한 상태
```

### 현실에서의 차이

```
시나리오: 프롬프트 인젝션으로 AI가 attacker.com에 데이터 전송 시도

OpenClaw:
  1. AI가 attacker.com에 요청 시도
  2. attacker.com이 차단 목록에 없음
  3. 요청 성공 → 데이터 유출 ❌

Nemoclaw:
  1. AI가 attacker.com에 요청 시도
  2. attacker.com이 화이트리스트에 없음
  3. 요청 차단 → 데이터 유출 방지 ✅
  4. 운영자에게 알림: "허가받지 않은 접속 시도"
```

---

## 3. 샌드박스 구현 방식의 차이

### OpenClaw의 샌드박스

```
[OpenClaw 샌드박스]
  소프트웨어 코드로 구현:

  def access_file(path):
      if path in ALLOWED_PATHS:
          return open(path)  # 허용
      else:
          raise PermissionError  # 차단 (소프트웨어적)

  취약점:
    - AI가 버그를 찾아 우회 가능
    - 취약점 발견 시 샌드박스 탈출 가능
    - 코드 수준의 제한은 코드로 우회 가능
```

### Nemoclaw의 샌드박스 (OpenShell + Landlock)

```
[Nemoclaw 샌드박스]
  운영체제 커널이 강제:

  커널: "이 프로세스는 /sandbox만 접근 가능"
  AI: "/etc/passwd 파일 열기 시도"
  커널: "이 경로는 허가되지 않음. 접근 거부."
    ↑ AI 코드 레벨에서 우회 불가

  장점:
    - 어떤 취약점이 있어도 샌드박스 탈출 불가
    - AI 에이전트가 아무리 영리해도 커널 제한 우회 불가
    - 시작 시 고정 → 실행 중 규칙 변경 불가
```

---

## 4. 개인정보 보호의 차이

### OpenClaw

```
클라우드 AI 사용 시:
  사용자 메시지 → [OpenClaw] → 클라우드 AI (그대로 전송)

개인정보 보호:
  - 수동으로 설정해야 함
  - 기본적으로 아무런 필터링 없음
  - 실수로 개인정보가 전송될 수 있음
```

### Nemoclaw

```
클라우드 AI 사용 시:
  사용자 메시지 → [Privacy Router] → PII 제거 → 클라우드 AI

자동화된 개인정보 보호:
  - 이름, 전화번호, 이메일 등 자동 감지 및 제거
  - 민감도에 따라 자동으로 로컬/클라우드 모델 선택
  - 로그에도 PII 자동 마스킹
```

---

## 5. 네트워크 제어의 차이

### OpenClaw 네트워크 설정

```yaml
# OpenClaw: 명시적 차단 방식
network:
  block:
    - "evil.com"
    - "malware.com"
  # 나머지는 모두 허용
```

문제: evil2.com, 새로운 악성 도메인 등은 막을 수 없음

### Nemoclaw 네트워크 설정

```yaml
# Nemoclaw: 화이트리스트 방식
network:
  default: deny
  allow:
    - host: "api.openai.com"
      port: 443
      methods: ["POST"]
    - host: "calendar.google.com"
      port: 443
      methods: ["GET", "POST"]
  # 나머지는 모두 차단
```

장점: 알려지지 않은 악성 도메인도 자동으로 차단됨

---

## 6. 행동 검증의 차이

### OpenClaw

```
사용자: "~/Documents 폴더 정리해줘"
AI: 파일 50개 중 30개를 삭제
→ 사용자가 모르는 사이에 파일 삭제됨
```

### Nemoclaw

```
사용자: "~/Documents 폴더 정리해줘"
AI: 파일 삭제 시도
↓
[Intent Verification 개입]
↓
운영자 알림: "AI가 다음 파일들을 삭제하려 합니다:
             - important_doc.pdf
             - project_notes.txt
             - ... (28개 더)
             허용하시겠습니까? [Y/N/상세보기]"
↓
운영자 결정 후 실행
```

---

## 7. 실제 사용 시나리오별 비교

### 시나리오 1: 개인 생산성 도구

```
목적: 이메일 관리, 일정 관리, 파일 정리

OpenClaw:
  ✅ 빠른 설치, 간단한 설정
  ✅ 다양한 채팅 앱 연동
  ⚠️  보안 설정을 수동으로 해야 함

Nemoclaw:
  ✅ 더 강력한 보안
  ✅ 자동 개인정보 보호
  ❌ 설치 복잡
  ❌ 아직 얼리 프리뷰
```

### 시나리오 2: 기업 내부 AI 에이전트

```
목적: 내부 문서 처리, 고객 데이터 접근

OpenClaw:
  ❌ 기본 보안이 기업 요구사항에 부족
  ❌ 데이터 유출 위험
  ❌ 컴플라이언스 충족 어려움

Nemoclaw:
  ✅ 기업급 보안
  ✅ 개인정보 자동 보호 (GDPR 등 대응)
  ✅ 감사 로그 자동 생성
  ✅ 행동 검증으로 실수 방지
```

### 시나리오 3: 연구/개발 환경

```
목적: 새로운 AI 에이전트 기능 개발/테스트

OpenClaw:
  ✅ 유연한 설정
  ✅ 빠른 프로토타이핑

Nemoclaw:
  ⚠️  보안 제약이 개발을 불편하게 할 수 있음
  ✅ 보안 취약점 없는 프로토타입 가능
```

---

## 8. OpenAI-compatible API 지원 비교

| 기능 | OpenClaw | Nemoclaw |
|------|----------|----------|
| OpenAI API 호환 | ✅ | ✅ |
| 로컬 모델 서버 연결 | ✅ | ✅ |
| 여러 모델 프로파일 | 제한적 | ✅ |
| 자동 모델 라우팅 | ❌ | ✅ (Privacy Router) |
| PII 제거 후 클라우드 전송 | ❌ | ✅ |
| 응답 엔드포인트 폴백 | ❌ | ✅ (/responses → /chat/completions) |

---

## 9. Nemoclaw가 개선한 것들 요약

```
1. 보안 기본값 변경
   Before (OpenClaw): 허용이 기본 → 사용자가 차단 설정
   After (Nemoclaw):  차단이 기본 → 관리자가 허용 설정

2. 샌드박스 강화
   Before: 소프트웨어 레벨 제한 (우회 가능)
   After:  커널 레벨 Landlock (우회 불가)

3. 개인정보 자동 보호
   Before: 수동 설정 필요
   After:  Privacy Router로 자동 PII 제거

4. 행동 검증 추가
   Before: AI가 즉시 실행
   After:  위험한 행동은 운영자 승인 필요

5. 네트워크 정책 세분화
   Before: 호스트 수준 차단 목록
   After:  호스트+포트+HTTP메서드+프로세스 단위 제어

6. 감사 로그 강화
   Before: 기본 로깅
   After:  컴플라이언스용 상세 감사 로그
```

---

## 10. 언제 무엇을 선택해야 하나?

```
OpenClaw를 선택하세요:
  - 지금 당장 안정적인 도구가 필요할 때
  - 개인 사용 목적
  - 보안 설정을 직접 세밀하게 제어하고 싶을 때
  - 다양한 채팅 앱 연동이 필요할 때

Nemoclaw를 선택하세요:
  - 기업/팀 환경에서 사용할 때
  - 개인정보/민감 데이터를 다룰 때
  - 보안이 최우선일 때
  - NVIDIA GPU가 있어 로컬 AI를 최대 활용하고 싶을 때
  - 얼리 프리뷰 단계를 감수할 수 있을 때

둘 다 공부하는 것이 좋은 이유:
  - OpenClaw를 이해하면 Nemoclaw의 개선점이 더 잘 보임
  - Nemoclaw의 보안 철학은 OpenClaw에도 적용 가능
  - AI 에이전트 보안의 전반적인 이해에 도움
```

---

## 마무리: 학습 로드맵

```
1단계: 기초 다지기
  [01_basic_concepts.md] → Docker, Port, Gateway, Sandbox 이해

2단계: OpenClaw 이해
  [02_openclaw.md] → AI 에이전트의 기본 동작 원리

3단계: Nemoclaw 이해
  [03_nemoclaw.md] → 보안 강화 아키텍처 이해

4단계: 직접 실행해보기
  [04_local_setup.md] → 로컬 환경에서 실행

5단계: 보안 심화
  [05_security.md] → 취약점과 대응 방법 깊이 이해

6단계: 비교 분석 (이 챕터)
  [06_comparison.md] → 전체 맥락에서 두 프로젝트 비교
```

---

> 이 자료는 2026년 3월 기준으로 작성되었습니다.
> Nemoclaw는 빠르게 발전 중이므로 최신 공식 문서([https://docs.nvidia.com/nemoclaw/](https://docs.nvidia.com/nemoclaw/))를 함께 참고하세요.
