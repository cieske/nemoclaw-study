# 챕터 04: 로컬에서 OpenAI-compatible Server로 실행하기

> 내 컴퓨터에서 Nemoclaw를 OpenAI 호환 서버와 연결해 실행하는 방법을 설명합니다.

---

## 1. 전체 그림 이해하기

먼저 무엇이 어떻게 연결되는지 그림을 보겠습니다:

```
[사용자 (나)]
      |
      | 채팅 또는 API 요청
      v
[Nemoclaw 에이전트]  ← 포트 8080
      |
      | AI 모델에 질문할 때
      v
[OpenAI-compatible 서버]  ← 포트 11434 (Ollama 예시)
      |
      | 실제 AI 모델 추론
      v
[로컬 AI 모델]
(Nemotron, Llama, Mistral 등)
```

핵심: Nemoclaw는 AI가 아닙니다. Nemoclaw는 AI를 **안전하게 감싸는 보안 실행 환경**입니다.
실제 AI 역할은 별도의 모델 서버가 합니다.

---

## 2. OpenAI-compatible Server 옵션들

로컬에서 OpenAI 호환 서버를 실행하는 방법은 여러 가지입니다:

### 옵션 1: Ollama (가장 간단)

```
Ollama = 로컬 AI 모델을 쉽게 실행하는 도구
기본 포트: 11434
API 형식: OpenAI-compatible
```

```bash
# Ollama 설치 후 모델 다운로드
ollama pull llama3.2
ollama pull nemotron-mini

# 서버 실행 (자동으로 포트 11434에서 시작)
ollama serve

# 테스트
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2",
    "messages": [{"role": "user", "content": "안녕?"}]
  }'
```

### 옵션 2: vLLM (NVIDIA GPU 활용)

```
vLLM = NVIDIA GPU에 최적화된 고성능 AI 서버
기본 포트: 8000
API 형식: OpenAI-compatible
```

```bash
# Docker로 vLLM 실행
docker run --gpus all \
  -p 8000:8000 \
  vllm/vllm-openai:latest \
  --model nvidia/Nemotron-Mini-4B-Instruct
```

### 옵션 3: LM Studio (GUI 제공)

GUI가 있어서 비개발자에게 친숙합니다. 설치 후 모델을 선택하고 "Start Server" 버튼을 누르면 됩니다.
기본 포트: `1234`

---

## 3. Nemoclaw 설치

### 사전 요구사항

```
필수:
  - Docker (컨테이너 실행용)
  - Git (소스 다운로드용)
  - 16GB+ RAM 권장
  - (선택) NVIDIA GPU
```

### 설치 과정

```bash
# 1. 소스 코드 다운로드
git clone https://github.com/NVIDIA/NemoClaw.git
cd NemoClaw

# 2. 초기화 스크립트 실행
./scripts/bootstrap.sh

# 3. 설정 파일 생성
cp config.example.yaml config.yaml
```

---

## 4. OpenAI-compatible 서버와 연결하기

### 4-1. 설정 파일 작성

```yaml
# config.yaml

# AI 모델 설정
inference:
  # OpenAI-compatible 서버 주소
  base_url: "http://localhost:11434/v1"  # Ollama 예시
  # base_url: "http://localhost:8000/v1"  # vLLM 예시
  # base_url: "http://localhost:1234/v1"  # LM Studio 예시

  # 사용할 모델 이름
  model: "llama3.2"

  # API 키 (로컬 서버는 보통 필요 없지만 형식 맞추기 위해)
  api_key: "not-needed"

# 게이트웨이 설정
gateway:
  bind: "127.0.0.1:8080"  # 로컬에서만 접근 가능하게
  auth:
    mode: "token"
    token: "my-secret-token-12345"  # 강력한 토큰으로 교체할 것

# 샌드박스 설정
sandbox:
  workspace: "~/nemoclaw-workspace"  # AI가 접근 가능한 폴더
```

### 4-2. 네트워크 정책 설정

```yaml
# network-policy.yaml

network:
  default: deny

  allow:
    # 로컬 AI 서버 접속 허용
    - host: "localhost"
      port: 11434  # Ollama 포트
      methods: ["POST", "GET"]

    # 필요한 외부 서비스만 추가
    # - host: "api.example.com"
    #   port: 443
    #   methods: ["GET", "POST"]
```

---

## 5. 실행 및 테스트

### Nemoclaw 시작

```bash
# 1. Docker 네트워크 확인
docker network ls

# 2. Nemoclaw 시작
./scripts/start.sh

# 또는 Docker Compose로 시작
docker compose up -d
```

### 실행 확인

```bash
# 게이트웨이가 실행 중인지 확인
curl http://localhost:8080/health \
  -H "Authorization: Bearer my-secret-token-12345"

# 기대 응답: {"status": "ok"}
```

### 간단한 테스트

```bash
# AI에게 질문하기
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer my-secret-token-12345" \
  -d '{
    "messages": [{"role": "user", "content": "안녕하세요!"}]
  }'
```

---

## 6. 포트 구성 한눈에 보기

```
[내 컴퓨터]
  │
  ├── 포트 8080  ← Nemoclaw 게이트웨이
  │               (외부에서 AI에게 요청할 때 사용)
  │
  ├── 포트 11434 ← Ollama (로컬 AI 모델 서버)
  │               (Nemoclaw가 AI 모델에 질문할 때 사용)
  │               (외부에서 직접 접근 불필요 → 방화벽으로 차단)
  │
  └── 포트 18789 ← OpenShell 게이트웨이
                  (보안 정책 관리 UI)
                  (반드시 로컬에서만 접근)
```

### 중요: 각 포트의 노출 범위

| 포트 | 서비스 | 권장 노출 범위 |
|------|--------|----------------|
| 8080 | Nemoclaw 게이트웨이 | localhost만 (원격 접속 필요 시 VPN/Tailscale 사용) |
| 11434 | Ollama | localhost만 |
| 18789 | OpenShell | localhost만 |

---

## 7. 자주 발생하는 오류와 해결 방법

### 오류 1: 포트가 이미 사용 중

```
Error: Port 8080 is already in use
```

해결:
```bash
# 어떤 프로세스가 포트를 사용하는지 확인
lsof -i :8080
# 또는
ss -tulpn | grep 8080

# config.yaml에서 다른 포트로 변경
gateway:
  bind: "127.0.0.1:9090"  # 8080 대신 9090 사용
```

### 오류 2: AI 모델 서버 연결 실패

```
Error: Connection refused to http://localhost:11434
```

해결:
```bash
# Ollama가 실행 중인지 확인
ollama list

# Ollama 서버 시작
ollama serve
```

### 오류 3: 인증 실패

```
Error: 401 Unauthorized
```

해결:
```bash
# 토큰이 config.yaml의 설정과 일치하는지 확인
# Authorization 헤더 형식 확인: "Bearer 토큰값"
curl -H "Authorization: Bearer my-secret-token-12345" ...
```

---

## 8. Inference Profile (추론 프로파일)

Nemoclaw는 상황에 따라 다른 모델을 사용하도록 프로파일을 설정할 수 있습니다.

```yaml
# inference-profiles.yaml

profiles:
  # 일반 작업용 (빠른 로컬 모델)
  default:
    base_url: "http://localhost:11434/v1"
    model: "llama3.2"

  # 복잡한 추론 (느리지만 정확한 모델)
  reasoning:
    base_url: "http://localhost:11434/v1"
    model: "nemotron-mini"

  # 개인정보 없는 일반 질문 (클라우드 모델)
  cloud:
    base_url: "https://api.openai.com/v1"
    model: "gpt-4o"
    api_key: "${OPENAI_API_KEY}"
```

Privacy Router는 이 프로파일 설정을 보고 **자동으로** 어떤 모델에 요청을 보낼지 결정합니다.

---

## 9. 로컬 실행 시 보안 체크리스트

```
[ ] 게이트웨이가 127.0.0.1에만 바인딩되어 있는가?
[ ] 강력한 인증 토큰이 설정되어 있는가?
[ ] 로컬 AI 서버(Ollama 등)가 외부에 노출되지 않는가?
[ ] 네트워크 정책이 default deny로 설정되어 있는가?
[ ] 필요한 외부 호스트만 화이트리스트에 있는가?
[ ] OpenShell 게이트웨이가 외부에 노출되지 않는가?
```

---

다음 챕터: [05 - 보안 심층 분석](./05_security.md)
