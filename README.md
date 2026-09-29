# 쥬얼리 (ZooEarly) — API Gateway

> 원본 팀 레포: https://github.com/sktflyai-passion5/ZooEarly-BE
> 이 레포는 포트폴리오용 사본이다. 커밋 이력은 원본 그대로다.

중도입국 초등 1~2학년 아이의 학교생활 적응을 돕는 음성 대화 앱 **쥬얼리**의 백엔드 게이트웨이.
React Native 앱과 FastAPI 추론 서버(STT / LLM / TTS) 사이의 **릴레이 전용 서버**다.

```
React Native App ──HTTPS/REST──▶ 이 서버 (Gateway) ──HTTP──▶ FastAPI ──▶ OpenAI API
 (UI/시나리오/게임/로컬 상태)      (검증·전달·에러 통일)      (STT/LLM/TTS)
```

이 서버가 하는 일은 세 가지뿐이다: **요청 검증, FastAPI로 전달, 에러 포맷 통일.**
DB 없음, 무상태(stateless). 비즈니스 로직을 여기에 추가하지 않는다.

## 내 역할

**백엔드 게이트웨이 1인 담당** — 설계, 구현, 테스트, 배포, 팀 간 API 계약 문서화까지.

- 앱 ↔ 게이트웨이 API 명세 작성·버전 관리 (v1.7까지, OpenAPI + TS 타입 자동 생성)
- 게이트웨이 ↔ FastAPI 연동 규약 문서화, 실서버 실측으로 불일치 항목 정리
- Docker 이미지화, Azure Container Apps 배포, GitHub Actions CI/CD 구축
- 프론트 연동 테스트용 가짜 추론 서버(mock) 제작
- 브랜치 전략·커밋 컨벤션·PR 템플릿 등 협업 규칙 수립

## 기술 스택

| 분류 | 사용 기술 |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.5 (spring-web, spring-validation), RestClient |
| Build | Gradle 8.14 |
| API 문서 | OpenAPI 3 (springdoc Swagger UI), openapi-typescript |
| Test | JUnit 5, Spring Boot Test (테스트 52개) |
| Infra | Docker (멀티스테이지, alpine JRE), Azure Container Apps, Azure Container Registry |
| CI/CD | GitHub Actions (PR 빌드·테스트, main 머지 시 자동 배포 + 스모크 테스트 + 자동 롤백) |
| 도구 | Python 표준 라이브러리 mock 서버, curl 스모크 테스트 스크립트 |

JPA·DB는 **의도적으로 쓰지 않았다.** 대화 이력·진행 상태는 앱과 FastAPI의 몫이고,
게이트웨이는 무상태로 두어야 수평 확장과 재배포가 단순해진다.

## API

앱에 노출하는 경로는 `/api/v1/ai/*`로 고정하고, FastAPI 쪽 경로는 `application.yml`로 주입한다.

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/api/v1/ai/chat` | 음성 대화 (STT→LLM→TTS 통합) |
| POST | `/api/v1/ai/stt` | 음성 → 텍스트 |
| POST | `/api/v1/ai/tts` | 텍스트 → 음성 |
| POST | `/api/v1/ai/feedback` | 표현 교정 피드백 |
| POST | `/api/v1/ai/pronunciation` | 발음 채점 |
| GET | `/api/v1/ai/pronunciation/sentences` | 발음 연습 문장 목록 |
| POST | `/api/v1/ai/story` | 하루 플레이 기록 → 동화 생성 |

성공 응답은 FastAPI가 만든 JSON을 **가공 없이** 그대로 전달하고, 실패는 게이트웨이가 공통 포맷으로 통일한다.

```json
{ "success": false, "error": { "code": "AI_TIMEOUT", "message": "...", "field": null } }
```

## 문제 해결 과정

### 1. 장애 상황에서 명세와 다른 에러 코드가 나가던 문제

**상황** — 실제 서버를 띄워 14개 장애 케이스를 검증해보니 3가지가 명세와 달랐다.

| 증상 | 원인 | 해결 |
|---|---|---|
| FastAPI 타임아웃인데 `504` 대신 `500` | RestClient가 응답 body를 읽다 타임아웃되면 `ResourceAccessException`이 아닌 일반 `RestClientException`으로 감싼다. 예외 타입으로만 판별해서 놓쳤다 | 예외 타입 대신 **원인 사슬(cause chain)에서 `SocketTimeoutException`을 탐색** |
| 10MB 초과 오디오에 `AUDIO_TOO_LARGE` 대신 `413` | multipart 한도가 명세 상한과 같은 10MB라 컨트롤러 진입 전에 잘림. 서비스의 용량 검증은 **도달 불가 코드**였다 | 컨테이너 한도를 12MB로 올려 게이트웨이가 먼저 판정하고, 컨테이너 한도는 무제한 업로드 방어선으로만 사용 |
| 잘못된 메서드·Content-Type이 `500` | catch-all 핸들러가 클라이언트 실수까지 삼킴. 500은 "서버 고장" 신호라 앱이 불필요하게 재시도하고 장애 알람도 오염된다 | 클라이언트 실수는 `400 INVALID_PARAMETER`로 분리 |

**결과** — 기존 테스트가 `contextLoads` 하나뿐이라 CI가 이런 문제를 잡을 수 없었다. 에러 계약 회귀 테스트 16개를 추가했다.

### 2. 오디오를 통째로 메모리에 올리던 문제 (OOM)

**상황** — `MultipartFile.getBytes()`가 업로드 파일 전체를 힙에 복사하고 있었다.

**해결** — `getResource()`로 바꿔 **스트림 그대로 FastAPI에 릴레이**했다. 코드도 19줄 줄었다.
FastAPI로 나가는 요청(파트 이름·파일명·Content-Type·길이)이 바뀌지 않았는지 캡처로 확인했다.

**측정** (힙 64MB, 10MB 오디오 8개 동시 요청, 백엔드 4초 지연, 2회 반복)

| | 성공 | 실패 | OutOfMemoryError |
|---|---|---|---|
| 변경 전 | 2 | 6 | 12건 |
| 변경 후 | 6~7 | 1~2 | **0건** |

### 3. 타임아웃을 감이 아니라 실측으로 정하기

**상황** — FastAPI 실서버를 붙이자 STT 요청이 **항상** `504`로 끝났다. 한도는 15초인데 실제로는 35~70초가 걸렸다.

**과정**

1. 처음에는 STT만 90초로 분리했다. 전부 늘리면 정말 서버가 죽었을 때 아이를 90초씩 기다리게 하기 때문이다
2. 재측정해보니 예열된 상태는 6~8초였고, 43초는 **콜드 스타트**(FastAPI `min-replicas=0`)였다
3. 발표 중 한동안 조용하다가 누르는 첫 요청이 정확히 콜드 스타트라, 이를 넘기는 **45초**로 확정했다

동화 생성도 같은 방식으로 정했다. 실측은 76 / 89 / 126초였다. 한도를 **일부러 최대값보다 짧은 120초**로 두었다.
아이를 3분 가까이 기다리게 하느니 끊고 "괜찮아, 다시 해볼까?"로 넘기는 편이 낫다고 판단했다.
근본 원인(장면 4개 순차 생성)은 FastAPI 쪽에 병렬화 개선안으로 전달했다.

### 4. 404가 CORS 에러·서버 고장으로 둔갑하던 문제

**상황 1** — FastAPI 담당자가 접두사 없이 `/ai/stt`를 호출하고 "모든 경로가 500이니 게이트웨이가 죽었다"고 판단했다.
매핑 없는 경로(`NoResourceFoundException`)가 catch-all 핸들러로 흘러 `500`이 되고 있었다.
→ **`404`로 응답하도록 수정.** 잘못된 주소는 부르는 쪽 문제지 서버 고장이 아니다.

**상황 2** — 프론트가 잘못된 경로를 부르자 브라우저 개발자 도구에는 404가 아니라 **"CORS 에러"만** 보였다.
CORS 매핑이 없는 경로의 에러 응답에는 헤더가 붙지 않아, 브라우저가 응답 자체를 차단한 것이다.
→ 허용 오리진은 그대로 두고, **모든 경로에 CORS 헤더가 붙도록** 수정해 진짜 상태 코드가 보이게 했다.

### 5. FastAPI와의 명세 불일치 — "문서만 보고 판단하지 않기"

처음에는 FastAPI 명세 문서만 대조하고 "요청 쪽은 이미 맞다"고 적었다. **실서버에 실제로 호출해보니 틀렸다.**
`audio` → `audio_file`, `language` → `language_code` 등 요청 필드명이 전부 `422`였다.

역할을 구조적으로 나눴다.
- **요청 쪽**은 게이트웨이가 맞출 수 있다 → FastAPI 경로를 설정(`inference.path.*`)으로 빼서 코드 수정 없이 대응
- **응답 쪽**은 게이트웨이가 body를 파싱하지 않으므로 **FastAPI만 고칠 수 있다** → before/after 대조표로 정리해 전달

### 6. 배포본에서만 `/docs`가 빈 화면

로컬에서는 멀쩡했는데 배포본에서 Swagger UI가 빈 화면이었다.
`.dockerignore`가 `docs/`를 통째로 제외해 **이미지 안에 명세 파일이 없었고**, 리소스 복사가 조용히 아무것도 하지 않았다.
→ `docs/` 제외는 유지하고 명세 파일 하나만 예외로 포함했다.

### 7. 안전한 CI/CD 설계

`main` 머지 시 **테스트 → 빌드 → ACR 푸시 → 배포 → 스모크 테스트 → 실패 시 자동 롤백**.

- 테스트 잡과 배포 잡을 분리했다. 레포 코드를 실행하는 잡에 Azure 토큰을 두지 않기 위해서다
- 이미지는 태그가 아닌 **digest로 고정**해 배포했다. 태그는 덮어쓸 수 있어 롤백 기준이 될 수 없다
- 서드파티 Action은 **커밋 SHA로 고정**했다
- Azure 인증은 비밀키 없이 **OIDC**로 했다
- Docker 런타임 베이스는 실측으로 골랐다. jre-jammy 460MB와 jre-alpine 342MB의 기동·응답을 모두 확인한 뒤 alpine을 택했다

## 배운 점

- **측정하지 않은 값은 추측이다.** 타임아웃, 필드명, 메모리 문제 모두 문서나 감으로 정한 값은 틀렸고, 실측한 뒤에야 맞는 답이 나왔다. 수치와 근거를 커밋 메시지와 설정 주석에 남기는 습관이 생겼다
- **에러 코드는 사람의 판단을 좌우한다.** 404를 500으로 내리자 다른 팀원이 "서버가 죽었다"고 오진했고, CORS 헤더 누락은 주소 오타를 설정 문제로 보이게 했다. 상태 코드가 정확해야 디버깅 시간이 줄어든다
- **"하지 않을 것"을 정하는 것도 설계다.** DB·비즈니스 로직·응답 가공을 게이트웨이에서 뺐다. 덕분에 FastAPI 스키마가 여러 번 바뀌어도 게이트웨이는 설정 변경만으로 대응할 수 있었다
- **팀 간 계약은 문서로 고정해야 한다.** 3개 파트(앱·게이트웨이·FastAPI)가 동시에 움직였다. 이때 OpenAPI 명세, 자동 생성 타입, 변경 이력, 연동 대조표가 대화보다 확실한 합의 수단이었다
- **사용자를 기준으로 기술 결정을 내린다.** 사용자가 초등 1~2학년이라는 점이 타임아웃 길이, 에러 메시지 톤, 음성 원본 비저장·로그 본문 미기록 같은 결정의 근거가 됐다

## 실행

```bash
# FastAPI 주소는 환경변수로 주입 (기본값 http://localhost:8000)
export INFERENCE_BASE_URL=http://localhost:8000

./gradlew bootRun
```

빌드·테스트:

```bash
./gradlew build      # 컴파일 + 테스트
./gradlew test       # 테스트만
```

동작 확인:

```bash
# 400 INVALID_PARAMETER가 §1.2 포맷으로 오면 정상
curl -X POST http://localhost:8080/api/v1/ai/tts \
  -H "Content-Type: application/json" -d '{}'
```

### FastAPI 없이 전 구간 돌려보기

진짜 추론 서버가 없으면 모든 요청이 `502 AI_SERVER_ERROR`로 떨어져 정상 화면을 볼 수 없다.
`tools/mock-inference/`의 가짜 추론 서버를 8000번에 띄우면 명세대로 생긴 200 응답이 온다.

```bash
python tools/mock-inference/mock_server.py    # 터미널 1 (표준 라이브러리만 씀)
./gradlew bootRun                              # 터미널 2
```

에러 화면 테스트용으로 강제 에러 토큰(`__slow__` `__stt_fail__` 등)도 지원한다.
자세한 건 [`tools/mock-inference/README.md`](tools/mock-inference/README.md).

> 게이트웨이 코드가 아니다. `src/`와 무관한 개발용 도구이고, 진짜 FastAPI가 뜨면 폴더째 지워도 된다.

## 패키지 구조

```
src/main/java/com/zooearly/
├── ZooEarlyApplication.java
├── common/
│   ├── config/     CorsConfig, RequestLoggingFilter(접근 로그)
│   ├── response/   ApiResponse(§1.2 래퍼), ErrorCode(§1.3)
│   └── exception/  GlobalExceptionHandler — 모든 에러를 공통 포맷으로
└── ai/
    ├── AiController.java      7개 엔드포인트
    ├── AiRelayService.java    검증 + 전달 (여기에 로직을 늘리지 말 것)
    └── client/InferenceClient.java  FastAPI 호출, 타임아웃(§0.4), 에러 변환
```

## 협업 규칙

### 브랜치 전략

상시 유지하는 브랜치는 `main`과 `dev` 둘뿐이다. 나머지는 전부 임시다.

| 브랜치 | 역할 | 규칙 |
|---|---|---|
| `main` | 시연·제출 가능한 안정판 | 직접 푸시 금지. `dev`에서만 PR로 들어온다 |
| `dev` | **기본 브랜치**, 개발 통합 지점 | 직접 푸시 금지. `feature/*`에서 PR로 들어온다 |
| `feature/작업내용` | 작업용 임시 | `dev`에서 따고 `dev`로 PR. **머지되면 즉시 삭제** |

```bash
git switch dev && git pull            # 항상 dev에서 시작
git switch -c feature/tts-endpoint
# ... 작업 & 커밋 ...
git push -u origin feature/tts-endpoint
# GitHub에서 PR 생성 (base: dev) → CI 통과 → 머지 → 브랜치 삭제
```

발표·배포 시점에만 `dev` → `main` PR을 올린다.
`main`을 비워두지 않는다 — 항상 "그 시점에 돌아가는 버전"이어야 안전판이 된다.

### 커밋 컨벤션

`타입: 설명` 형식. 타입은 아래 중 하나.

| 타입 | 용도 | 예시 |
|---|---|---|
| `feat` | 기능 추가 | `feat: tts 엔드포인트 릴레이 구현` |
| `fix` | 버그 수정 | `fix: history 검증에서 빈 배열 허용` |
| `refactor` | 동작 변화 없는 정리 | `refactor: 검증 헬퍼 분리` |
| `docs` | 문서만 변경 | `docs: README 실행법 추가` |
| `test` | 테스트만 변경 | `test: chat 검증 실패 케이스 추가` |
| `chore` | 빌드·설정 | `chore: gradle 8.14로 업그레이드` |

### PR 규칙

- PR은 작게 — 엔드포인트 하나, 수정 하나 단위로.
- 템플릿(`.github/PULL_REQUEST_TEMPLATE.md`)의 체크리스트를 채운다.
- 최소 1명 리뷰 후 머지.

## 주의사항

- **API 키·비밀값을 커밋하지 않는다.** `application.yml`에는 `${환경변수}` 참조만 두고, 실제 값은 로컬 환경변수나 `application-local.yml`(.gitignore 됨)에 둔다.
- FastAPI 응답 body를 게이트웨이에서 파싱·가공하지 않는다. 가공하는 순간 FastAPI 스키마가 바뀔 때마다 이 서버도 재배포해야 한다.
- 에러 응답에 "틀렸어요"류 문구를 넣지 않는다 — 아이 화면 폴백 정책은 앱 담당.
