# 챕터 08: 관련 논문 정리

> OpenClaw/Nemoclaw의 보안을 다룬 학술 논문들을 비전공자도 이해할 수 있도록 정리합니다.
> 모두 2026년 초에 발표된 논문들로, OpenClaw의 급격한 확산과 함께 등장한 보안 연구들입니다.

---

## 논문들의 전체 맥락

2026년 초, OpenClaw가 GitHub 20만 스타를 돌파하며 폭발적으로 확산되자,
전 세계 보안 연구자들이 동시다발적으로 취약점 분석 논문을 발표했습니다.

```
2026년 2월~3월: 최소 10편 이상의 OpenClaw 보안 논문 발표
   ↓
공통된 발견: OpenClaw의 기본 방어율은 평균 17%에 불과
   ↓
각 팀마다 다른 각도로 문제를 분석하고 대안을 제시
```

이 챕터는 그 중 대표적인 10편을 정리합니다.

---

## 논문 1

**제목:** Don't Let the Claw Grip Your Hand: A Security Analysis and Defense Framework for OpenClaw
*(클로가 손을 잡게 두지 마라: OpenClaw 보안 분석 및 방어 프레임워크)*

**arXiv:** [2603.10387](https://arxiv.org/abs/2603.10387) | **발표:** 2026년 3월 11일

**저자:** Zhengyang Shan, Jiayun Xin, Yue Zhang, Minghui Xu

---

### 핵심 발견: 기본 방어율 17%

이 논문의 가장 충격적인 결과입니다.
연구팀은 OpenClaw에 47가지 공격 시나리오를 실행했고,
**OpenClaw가 스스로 막은 것은 평균 17%에 불과**했습니다.

```
47가지 공격 시도
     ↓
기본 OpenClaw가 막은 것: 8개 (17%)
기본 OpenClaw가 뚫린 것: 39개 (83%)
```

### 분류한 6가지 공격 유형

| 유형 | 설명 | 예시 |
|------|------|------|
| 인코딩 우회 | 명령어를 Base64 등으로 숨겨서 전달 | `ZWNobyBoZWxsbw==` |
| 샌드박스 경계 위반 | 허가된 영역 밖 파일 접근 시도 | `/etc/passwd` 읽기 |
| 간접 프롬프트 인젝션 | 외부 콘텐츠에 숨겨진 명령 | 웹페이지 내 숨겨진 지시 |
| 공급망 오염 | 악성 스킬/플러그인 주입 | 악성 ClawHub 스킬 |
| 리소스 고갈 | CPU/메모리 소모 공격 | 무한 루프 실행 |
| 권한 상승 | 더 높은 권한 획득 시도 | 시스템 파일 수정 |

### 제안한 방어책: HITL (Human-in-the-Loop)

```
35개 탐지 규칙 + 의미론적 판단기 + 인간 승인 의무화

결과:
  HITL 없음: 방어율 17%
  HITL 있음: 방어율 19%~92% (공격 유형에 따라)
  → 인간이 개입하는 것이 핵심
```

**오픈소스:** https://github.com/S2yyyy/OpenClaw-Analysis

---

## 논문 2

**제목:** From Assistant to Double Agent: Formalizing and Benchmarking Attacks on OpenClaw for Personalized Local AI Agent
*(비서에서 이중간첩으로: 개인화 로컬 AI 에이전트 OpenClaw에 대한 공격 형식화 및 벤치마킹)*

**arXiv:** [2602.08412](https://arxiv.org/abs/2602.08412) | **발표:** 2026년 2월 9일

**저자:** Yuhang Wang 외 8명 (Xidian University)

---

### 핵심 아이디어: "개인화된 에이전트"의 고유 위험

기존 보안 연구는 "AI가 특정 작업을 하다가 공격받는" 상황을 분석했습니다.
이 논문은 **"나의 개인 정보를 알고, 나 대신 행동하는 AI"**가 공격받으면 어떻게 되는지에 집중했습니다.

```
일반 AI 에이전트:
  작업 요청 → 처리 → 완료
  (개인 맥락 없음)

개인화 AI 에이전트 (OpenClaw):
  내 이메일 접근 가능
  내 캘린더 접근 가능
  내 파일 접근 가능
  내 채팅 기록 알고 있음
  → 탈취되면 피해 규모가 완전히 다름
```

### PASB: 최초의 개인화 에이전트 보안 벤치마크

연구팀이 만든 **PASB(Personalized Agent Security Bench)**의 특징:

```
기존 벤치마크의 문제:
  → 인공적인 시나리오, 단순 작업, 한 번의 상호작용

PASB의 개선:
  → 실제 사용 시나리오 (이메일, 일정, 파일 관리)
  → 허니 토큰(함정 정보)과 기밀 파일 포함
  → 장기 상호작용 (여러 세션에 걸친 공격)
  → 실제 OpenClaw를 블랙박스로 평가
```

### 핵심 발견

```
1. 공격이 누적된다
   → 한 세션에서 심은 공격이 다음 세션에서 발동
   → 장기 상호작용일수록 피해 증가

2. 취약점이 여러 단계에 걸쳐 발생
   → 프롬프트 처리 단계
   → 도구 호출 단계
   → 외부 콘텐츠 접근 단계
   → 메모리 관련 동작 단계

3. 프롬프트 레벨 보호만으로는 부족
   → 실제 배포 환경의 위험을 합성 벤치마크로 측정 불가
```

---

## 논문 3

**제목:** A Trajectory-Based Safety Audit of Clawdbot (OpenClaw)
*(Clawdbot(OpenClaw)의 궤적 기반 안전성 감사)*

**arXiv:** [2602.14364](https://arxiv.org/abs/2602.14364) | **발표:** 2026년 2월 16일

**저자:** Tianyu Chen, Dongrui Liu, Xia Hu, Jingyi Yu, Wenjie Wang

---

### 핵심 아이디어: "궤적(Trajectory)" 기반 평가

기존 방법: "AI가 이 질문에 맞는 답을 했는가?" (결과만 평가)

이 논문의 방법: "AI가 어떤 경로로 이 결과에 도달했는가?" (과정 전체 평가)

```
궤적(Trajectory) = 메시지 + 행동 + 도구 호출 + 도구 결과의 전체 흐름

예시:
  사용자 요청: "이 파일 삭제해줘"

  궤적 분석:
  메시지 수신 → 파일 경로 확인 → 파일 목록 조회 →
  사용자 확인 요청 → 파일 삭제 실행 → 완료 보고

  이 과정의 어느 단계에서 안전하지 않은 동작이 있었는가?
```

### 평가 방법

```
34개 테스트 케이스
  + 자동 판정기 (AgentDoG-Qwen3-4B)
  + 인간 검토자

6가지 위험 차원:
  1. 신뢰성
  2. 의도 해석
  3. 범위 준수
  4. 정보 보안
  5. 행동 안전성
  6. 복구 가능성
```

### 핵심 발견

```
비균일한 안전 프로파일:
  → 명확한 작업: 대체로 안전
  → 모호한 의도, 열린 목표: 자주 실패

가장 위험한 상황:
  → 의도 오해 (사용자가 원하지 않는 행동 실행)
  → 프롬프트 인젝션 취약성
  → 연쇄 오류 (작은 실수가 큰 피해로 확대)

OpenClaw의 광범위한 권한 때문에
사소한 오해가 심각한 결과로 이어질 수 있음
```

---

## 논문 4

**제목:** Bounds and Identification of Joint Probabilities...
*(인과추론 분야 논문 — OpenClaw와 무관)*

**arXiv:** [2602.18762](https://arxiv.org/abs/2602.18762)

> **⚠️ 주의:** 이 논문은 통계/인과추론 분야의 논문으로 OpenClaw와 관계없습니다.
> 원래 의도한 논문은 **arXiv:2603.18762** (ClawTrap)일 가능성이 높습니다.

---

### 원래 의도된 논문 (arXiv:2603.18762)

**제목:** ClawTrap: A MITM-Based Red-Teaming Framework for Real-World OpenClaw Security Evaluation
*(ClawTrap: 실제 OpenClaw 보안 평가를 위한 MITM 기반 레드팀 프레임워크)*

**저자:** Haochen Zhao (싱가포르국립대), Shaoyang Cui (칭화대) | **발표:** 2026년 3월 19일

### 핵심 아이디어: 실제 네트워크 공격 환경 재현

기존 보안 평가: 정적 환경에서 텍스트 수준 공격만 테스트

ClawTrap: **실시간 네트워크 트래픽을 중간에서 가로채고 조작**

```
[OpenClaw 에이전트]
        ↕
[mitmdump 중간자 엔진] ← ClawTrap의 핵심
        ↕
[외부 웹사이트/API]
```

### ClawTrap이 할 수 있는 것

```
1. Static HTML Replacement: 웹페이지 내용 전체 교체
   → AI가 읽는 웹페이지를 조작된 내용으로 대체

2. Iframe Popup Injection: 팝업 형태로 악성 내용 삽입
   → 실제 사이트는 정상이지만 AI에게는 악성 내용 주입

3. Dynamic Content Modification: 실시간 내용 변경
   → AI가 데이터를 수집하는 동안 데이터 조작
```

### 핵심 발견

```
모델 간 차이가 크다:
  약한 모델: 조작된 내용을 그대로 신뢰 → 위험한 행동
  강한 모델: 이상한 점을 인식하고 안전한 대안 선택

→ 네트워크 레벨 공격은 정적 테스트로 발견 불가
→ 실제 네트워크 환경에서만 나타나는 취약점 존재
```

---

## 논문 5

**제목:** Uncovering Security Threats and Architecting Defenses in Autonomous Agents: A Case Study of OpenClaw
*(자율 에이전트의 보안 위협 발굴 및 방어 설계: OpenClaw 사례 연구)*

**arXiv:** [2603.12644](https://arxiv.org/abs/2603.12644) | **발표:** 2026년 3월 13일

**저자:** Zonghao Ying 외 10명

---

### 핵심 아이디어: 3층 위험 분류 체계

기존 보안 연구는 취약점을 나열하는 방식이었습니다.
이 논문은 **체계적인 분류 체계**를 제안합니다.

```
Layer 1: AI 인지 (AI Cognitive)
  → AI가 "잘못 생각"하는 문제
  → 예: 잘못된 의도 해석, 잘못된 맥락 판단

Layer 2: 소프트웨어 실행 (Software Execution)
  → 코드/도구가 "잘못 실행"되는 문제
  → 예: 권한 없는 파일 접근, 명령 주입

Layer 3: 정보 시스템 (Information System)
  → 데이터/통신이 "잘못 처리"되는 문제
  → 예: 데이터 유출, 공급망 오염
```

### FASA: 전 생애주기 보안 아키텍처

**Full-Lifecycle Agent Security Architecture**

```
기존 방어: "입력을 필터링하자" (단일 지점 방어)

FASA의 접근:
  에이전트의 모든 생애주기에서 동시에 방어
  → Zero-trust 실행 원칙
  → 동적 의도 검증
  → 추론-행동 상관관계 분석
  (AI가 생각하는 것과 실제로 하는 것을 비교)
```

### Project ClawGuard

FASA를 실제로 구현한 엔지니어링 프로젝트입니다.
이 논문과 함께 코드와 데이터셋이 공개되었습니다.

---
