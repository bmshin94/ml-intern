# ML Intern 분석 & 활용 가이드 (한국어)

> 이 문서는 `ml-intern` 프로젝트를 처음 접한 사용자를 위해
> 저장소 코드를 직접 분석해서 정리한 한국어 가이드입니다.

## 📌 저장소 주소

| 구분 | 주소 |
|---|---|
| **원본 (Hugging Face 공식)** | https://github.com/huggingface/ml-intern |
| **현재 작업 저장소 (내 복사본)** | https://github.com/bmshin94/ml-intern |
| **공식 데모 (웹)** | https://smolagents-ml-intern.hf.space/ |
| **라이선스** | Apache License 2.0 (상업적 이용 가능) |

---

# 1. 이게 뭐하는 프로젝트야?

## 한 줄 요약

> **"AI 모델을 만들어주는 AI 알바생"**

사용자가 `"라마 모델을 내 데이터셋으로 파인튜닝해줘"` 라고 말만 하면,
에이전트가 아래 과정을 **혼자 다** 수행합니다.

```
1. 관련 논문 검색 & 방법론 정독
2. 최신 HF 라이브러리 문서 확인
3. GitHub에서 동작하는 예제 코드 수집
4. 학습 스크립트 작성
5. 클라우드 GPU 빌려서 학습 실행
6. 완성된 모델을 Hugging Face Hub에 업로드
```

## 비유로 이해하기

식당에 비유하면:

> "김치찌개 만들어줘" 라고 말하면
> → 레시피 찾아보고 → 장 보고 → 주방 빌려서 요리하고 → 완성품을 갖다주는 알바생

여기서 "김치찌개" 자리에 들어가는 게 **AI 모델**입니다.

## 만든 사람

Hugging Face 공식 팀 (smolagents 팀)
- Aksel Joonas Reedi, Henri Bonamy, Yoan Di Cosmo, Leandro von Werra, Lewis Tunstall

## 규모

| 항목 | 수치 |
|---|---|
| 파이썬 코드 | 약 25,000 줄 |
| 전체 파일 | 약 200개 |
| 유닛 테스트 | 50개 이상 |

장난감 프로젝트가 아니라 **프로덕션급** 코드베이스입니다.

---

# 2. 폴더 구조 분석

## 최상위 구조

| 폴더 | 크기 | 역할 |
|---|---|---|
| `agent/` | 1.1MB | 🧠 **에이전트 두뇌 — 사실상 본체** |
| `frontend/` | 744KB | 💻 React + Vite 채팅 UI |
| `backend/` | 228KB | 🔌 FastAPI 웹서버 |
| `tests/` | 552KB | ✅ 유닛/통합 테스트 |
| `scripts/` | 120KB | 🛠️ KPI 집계, 백로그 우선순위 등 유틸 |
| `configs/` | - | ⚙️ 기본 모델·MCP 서버 설정 JSON |

## `agent/` 내부 (가장 중요)

```
agent/
├── core/                    ← 에이전트 엔진
│   ├── agent_loop.py            # 핵심 루프 (최대 300 iteration)
│   ├── session.py               # 세션 상태 관리
│   ├── tools.py                 # ToolRouter (도구 라우팅)
│   ├── doom_loop.py             # ⭐ 무한 삽질 감지 & 교정 프롬프트 주입
│   ├── approval_policy.py       # ⭐ 위험 작업 승인 게이트
│   ├── cost_estimation.py       # ⭐ GPU 비용 사전 예측
│   ├── yolo_budget.py           # ⭐ 자동승인 모드 예산 상한
│   ├── usage_thresholds.py      # 사용량 임계치 알림
│   ├── context_manager 연동     # 대화 자동 압축 (170k 토큰 기준)
│   ├── model_switcher.py        # 런타임 모델 교체
│   ├── session_persistence.py   # 세션 저장/복원
│   └── redact.py                # 민감정보 마스킹
├── tools/                   ← 에이전트의 "손발"
├── prompts/                 ← 시스템 프롬프트 (v1, v2, v3)
├── context_manager/         ← 대화 히스토리 관리
├── messaging/               ← Slack 알림 게이트웨이
└── utils/                   ← 터미널 렌더링 등
```

---

# 3. 에이전트가 가진 도구 목록

`agent/core/tools.py` 에서 실제로 확인한 목록입니다.

## 📚 리서치 계열

| 도구 | 설명 |
|---|---|
| `research` | **서브 에이전트 소환** — 논문 인용 그래프를 병렬로 추적 |
| `hf_papers` | 논문 검색 / 읽기 / 인용 그래프 / 데이터셋 추출 |
| `explore_hf_docs` | HF 공식 문서 탐색 |
| `fetch_hf_docs` | HF 문서 원문 가져오기 |
| `web_search` | 웹 검색 |
| `github_find_examples` | GitHub에서 동작하는 예제 코드 검색 |
| `github_read_file` | GitHub 파일 읽기 (.ipynb 포함) |
| `github_list_repos` | 저장소 목록 조회 |

## 🗂️ 데이터 / 모델 계열

| 도구 | 설명 |
|---|---|
| `hf_inspect_dataset` | 데이터셋 컬럼 구조 확인 (KeyError 방지) |
| `hf_repo_files` | HF Hub 파일 업로드/관리 |
| `hf_repo_git` | HF Hub Git 조작 |

## ⚡ 실행 계열 (핵심)

| 도구 | 설명 |
|---|---|
| `sandbox_create` | HF Space를 GPU 샌드박스로 생성 |
| `hf_jobs` | 실제 GPU 학습 잡을 클라우드에 제출 |
| `bash` / `read` / `write` / `edit` | 로컬 파일 조작 (로컬 모드일 때) |

## 📢 기타

| 도구 | 설명 |
|---|---|
| `plan_tool` | 작업 계획 수립 및 진행상황 추적 |
| `notify` | Slack 알림 발송 |

---

# 4. 시스템 프롬프트의 인상적인 부분

`agent/prompts/system_prompt_v3.yaml` 에서 발췌한 내용입니다.

## "네 지식은 낡았다"고 명시

> *"You do not know current APIs for TRL, Transformers, PEFT, Trackio...*
> *Your internal knowledge WILL produce wrong imports, wrong argument names."*

AI가 아는 척하다 틀리는 것을 막기 위해, **무조건 최신 문서를 먼저 읽으라고** 강제합니다.

## AI가 저지를 실수를 미리 명시

| 실수 | 내용 | 대응 |
|---|---|---|
| **HALLUCINATED IMPORTS** | 존재하지 않는 모듈 import | 실제 예제 코드 먼저 읽기 |
| **WRONG TRAINER ARGUMENTS** | 없는 설정값 전달 | 실제 문서 fetch |
| **WRONG DATASET FORMAT** | 컬럼명 추측 → KeyError | `hf_inspect_dataset` 호출 |
| **DEFAULT TIMEOUT KILLS JOBS** | 30분 기본 타임아웃으로 학습 중단 | 최소 2시간 설정 |
| **LOST MODELS** | `push_to_hub=True` 누락 → 결과 영구 소실 | 학습 설정에 반드시 포함 |
| **BATCH FAILURES** | 잡 여러 개 동시 제출 → 같은 버그로 전부 실패 | 1개 먼저 검증 후 확장 |
| **SILENT DATASET SUBSTITUTION** | 데이터셋 실패 시 몰래 다른 걸로 대체 | 사용자에게 알리고 물어보기 |
| **FLASH-ATTN 컴파일** | 컴파일 실패 빈번 | HF Hub 커널 사용, T4 회피 |

이 항목들은 전부 **실제 운영에서 겪은 사고**가 축적된 결과물입니다.

---

# 5. 설치 및 사용법

## 5-1. 준비물

| 준비물 | 필요 이유 |
|---|---|
| **Python 3.12** | `.python-version` 기준 |
| **uv** | 파이썬 패키지 매니저 |
| **HF 토큰** | 필수 (AI 모델 호출) |
| **GitHub 토큰** | 선택 (예제 코드 검색용) |

### uv 설치

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

## 5-2. 토큰 발급

### Hugging Face 토큰 (필수)

1. https://huggingface.co 로그인
2. Settings → Access Tokens → New token
3. **권한 설정 (중요!)**
   - ✅ `Make calls to Inference Providers` ← **없으면 아예 동작 안 함**
   - ✅ `Write access to contents/settings of all repos` (모델 업로드용)

### GitHub 토큰 (선택)

1. GitHub → Settings → Developer settings → Personal access tokens (Fine-grained)
2. 읽기 권한만으로 충분

## 5-3. 설치

```bash
cd ~/ml-intern

uv sync                 # 의존성 설치
uv tool install -e .    # 전역 명령어 등록

ml-intern --help        # 설치 확인
```

## 5-4. `.env` 파일

프로젝트 루트에 생성:

```bash
HF_TOKEN=hf_xxxxxxxxxxxxx
GITHUB_TOKEN=github_pat_xxxxxxxxxxxxx
```

> `.env` 없이 실행해도 CLI가 첫 실행 시 토큰 입력을 요청합니다.

## 5-5. 실행

### 대화형 모드 (권장)

```bash
ml-intern
```

### 헤드리스 모드 (자동 승인 — 주의!)

```bash
ml-intern "llama 모델을 내 데이터셋으로 파인튜닝해줘"
```

⚠️ 헤드리스 모드는 **승인 없이 GPU를 사용**합니다. 익숙해지기 전에는 사용하지 마세요.

### CLI 옵션 (argparse 실측)

| 옵션 | 설명 |
|---|---|
| `prompt` (위치인자) | 헤드리스 모드로 실행할 프롬프트 |
| `--model`, `-m` | 사용할 모델 지정 |
| `--max-iterations` | 턴당 최대 LLM 요청 수 (기본 50, `-1`은 무제한) |
| `--no-stream` | 토큰 스트리밍 비활성화 |
| `--sandbox-tools` | 로컬 파일 대신 HF Space 샌드박스 사용 |

## 5-6. 슬래시 명령어 (소스 실측)

| 명령어 | 인자 | 설명 |
|---|---|---|
| `/help` | | 도움말 |
| `/new` | | 새 대화 시작 |
| `/clear` | | 화면 정리 + 새 대화 |
| `/undo` | | 마지막 턴 취소 |
| `/compact` | | 컨텍스트 압축 |
| `/resume` | `[index\|id\|path]` | `./session_logs` 에서 이전 세션 복원 |
| `/model` | `[id]` | 모델 목록 조회 / 전환 |
| `/effort` | `[level]` | 추론 강도 설정 |
| `/yolo` | | 자동승인 토글 (⚠️ 주의) |
| `/status` | | 현재 모델·턴 수 확인 |
| `/share-traces` | `[public\|private]` | 대화 기록 공개 설정 |
| `/quit` | | 종료 |

### `/effort` 레벨

```
minimal | low | medium | high | xhigh | max | off
```

- 간단한 질문 → `low`
- 복잡한 ML 설계 → `high` 이상 (느리고 비쌈)

## 5-7. 사용 가능한 모델 (실측)

### 클라우드 모델 (유료)

```bash
/model anthropic/claude-opus-4.8:fal-ai        # Claude Opus 4.8
/model openai/gpt-5.5:fal-ai                   # GPT-5.5
/model MiniMaxAI/MiniMax-M3:novita             # MiniMax M3
/model moonshotai/Kimi-K2.7-Code:novita        # Kimi K2.7 Code
/model zai-org/GLM-5.2:novita                  # GLM 5.2 (기본값)
/model deepseek-ai/DeepSeek-V4-Pro:novita      # DeepSeek V4 Pro
```

기본값이 GLM 5.2인 이유는 **성능 대비 비용 효율**이 좋기 때문입니다.

### 로컬 모델 (무료)

지원 프리픽스: `ollama/`, `vllm/`, `lm_studio/`, `llamacpp/`

```bash
ollama pull llama3.1:8b
ml-intern --model ollama/llama3.1:8b
```

필요 시 `.env` 추가:

```bash
LOCAL_LLM_BASE_URL=http://localhost:8000
LOCAL_LLM_API_KEY=optional
```

> ⚠️ 로컬 모델을 쓰더라도 **샌드박스 도구를 쓰려면 `HF_TOKEN`이 필요**합니다.
> (HF Space를 실제로 생성하기 때문)

## 5-8. 승인 프롬프트 (가장 중요)

에이전트가 비용이 발생하거나 위험한 작업을 하기 전에 묻습니다.

```
Approve item 1? (y=yes, yolo=approve all, n=no, or provide feedback):
```

| 입력 | 동작 |
|---|---|
| `y` / `yes` | 이 항목만 승인 |
| `n` / `no` | 거부 |
| **임의의 문장** | ⭐ **거부하면서 동시에 지시 전달** |
| `yolo` | ⚠️ 이후 전부 자동 승인 |
| `Ctrl+C` | 남은 항목 전부 거부 |

### 활용 팁

`n` 대신 이렇게 입력하면 훨씬 효율적입니다.

```
> A100 말고 T4로 해줘. 그리고 배치사이즈 8로 줄여
```

## 5-9. 로컬 도구 vs 샌드박스

| 모드 | 실행 위치 | 도구 | 사용 시점 |
|---|---|---|---|
| **로컬** (기본) | 내 컴퓨터 | `bash`, `read`, `write`, `edit` | 내 프로젝트 코드 수정 |
| **샌드박스** | HF Space | `sandbox_create` 등 | GPU 필요, 위험한 코드 테스트 |

```bash
ml-intern --sandbox-tools "이 학습 스크립트 GPU에서 테스트해줘"
```

항상 샌드박스로 쓰려면 `~/.config/ml-intern/cli_agent_config.json`:

```json
{ "tool_runtime": "sandbox" }
```

## 5-10. 웹 UI로 실행하기

터미널 2개가 필요합니다.

```bash
# 터미널 1 — 백엔드
cd ~/ml-intern/backend
uv run uvicorn main:app --host ::1 --port 7860

# 터미널 2 — 프론트엔드
cd ~/ml-intern/frontend
npm ci          # 최초 1회 (npm install 대신 npm ci 권장)
npm run dev
```

접속: **http://localhost:5173/**

> `127.0.0.1:7860`이 이미 점유된 경우 `::1` 바인딩이 Vite 프록시와 더 잘 맞습니다. (`AGENTS.md` 참고)

## 5-11. GPU 하드웨어 목록 (실측)

| 등급 | 옵션 | 권장 용도 |
|---|---|---|
| **CPU** | `cpu-basic`(2vCPU/16GB), `cpu-upgrade`(8vCPU/32GB) | 데이터 처리, 테스트 |
| **T4** | `t4-small`, `t4-medium` (VRAM 16GB) | 소형 모델 (~1B) |
| **A10G** | `a10g-small`, `a10g-large`, `a10g-largex2/x4` (24GB) | 중형 모델 (~7B) ⭐권장 |
| **L4 / L40S** | `l4x1`, `l4x4`, `l40sx1/x4/x8` | 추론 최적화 |
| **A100** | `a100-large`, `a100x4`, `a100x8` | 대형 모델 |

> ⚠️ **T4는 Ampere 이전 세대라 flash-attention Hub 커널을 쓸 수 없습니다.**
> 시스템 프롬프트에도 명시적으로 "T4를 선택하지 말라"고 되어 있습니다. 최소 A10G 권장.

## 5-12. 프라이버시 설정

기본값으로 **모든 세션이 본인 HF 계정의 private 데이터셋에 자동 업로드**됩니다.
(`{hf_user}/ml-intern-sessions`)

```bash
/share-traces            # 현재 공개 상태 확인
/share-traces public     # 공개
/share-traces private    # 비공개
```

완전히 끄려면 `~/.config/ml-intern/cli_agent_config.json`:

```json
{ "share_traces": false }
```

> 별개로 `smolagents/ml-intern-sessions` 에는 익명화된 텔레메트리만 수집됩니다.

## 5-13. 문제 해결

| 증상 | 원인 / 해결 |
|---|---|
| `ml-intern` 명령어 없음 | `uv tool install -e .` 재실행 |
| 401 인증 오류 | HF 토큰에 **Inference Providers 권한** 확인 |
| 403 오류 | 토큰 write 권한 부족 |
| GitHub 검색 실패 | `GITHUB_TOKEN` 확인 |
| 응답이 부실함 | `/effort high` 로 상향 |
| 대화가 느려짐 | `/compact` 실행 |
| 터미널 출력 깨짐 | `--no-stream` 옵션 |
| 학습 결과 소실 | `push_to_hub=True` 누락 — 다음엔 명시적으로 지시 |

## 5-14. 권장 시작 순서

```
1. uv sync && uv tool install -e .
2. .env 작성 (HF_TOKEN 필수)
3. ml-intern 실행
4. /status 로 상태 확인
5. 가벼운 질문으로 감 잡기 (예: "트렌딩 데이터셋 알려줘")
6. 익숙해지면 실제 학습 진행
```

### 초보자 금지 사항

- ❌ 처음부터 `/yolo` 켜기
- ❌ 처음부터 A100 사용
- ❌ 헤드리스 모드로 학습 실행

---

# 6. 수익화 아이디어

## 사전 검토

| 항목 | 판단 |
|---|---|
| **라이선스** | ✅ Apache 2.0 — 상업적 이용/수정/재배포 가능 (고지문 유지, 변경사항 표시 조건) |
| **그대로 재판매** | ❌ HF가 이미 무료 호스팅 중 + 추론비 실비 발생으로 마진 얇음 |
| **진짜 가치** | ⭐ ① 이 도구로 대체하는 **노동** ② 내부의 **설계 패턴** |

## A그룹 — "일"을 팔기

### A-1. 소형 특화 모델 제작 대행 ⭐ 최우선 추천

ML 팀이 없는 회사를 대신해 파인튜닝해주고 모델을 납품.

| 항목 | 내용 |
|---|---|
| 고객 | ML 팀 없는 중소기업, 1인 개발사, 에이전시 |
| 가격 | 건당 200~800만원 (모델 + 리포트 + 배포 가이드) |
| 원가 | GPU 비용 + 작업 시간 |
| 강점 | 2주 걸릴 리서치+코딩을 **2~3일**로 단축 → 마진 확보 |
| 난이도 | ⭐⭐☆☆☆ (즉시 가능) |
| 검증법 | 크몽/위시켓에 "sLLM 파인튜닝 대행" 등록 → 1건만 팔려도 검증 |

### A-2. "데이터셋 → 모델" 자판기 서비스

웹에서 CSV 업로드 + 결제 → 학습된 모델이 HF 계정으로 전달.

| 항목 | 내용 |
|---|---|
| 재사용 | 이 저장소의 `frontend` + `backend` 거의 그대로 사용 가능 |
| 이미 있는 것 | `usage.py`, `cost_estimation.py`, `yolo_budget.py` → **과금 로직 절반 완성** |
| 난이도 | ⭐⭐⭐⭐☆ |
| 리스크 | 🚨 원가 관리가 생명. **하드웨어 화이트리스트(T4/A10G만) 필수** |

### A-3. 니치 모델 판매

특정 분야 모델을 미리 만들어 판매 (법률 요약, 의료 분류, 커머스 정규화 등).

- 판매처: HF Hub 유료 모델, Gumroad, 자체 API
- 난이도 ⭐⭐☆☆☆ / 주문 먼저 받고 제작하는 것이 안전

## B그룹 — "부품"을 팔기

### B-1. "에이전트 안전벨트" 라이브러리 ⭐ 확장성 최고

요즘 AI 에이전트 개발사들이 공통으로 겪는 문제:

- "에이전트가 밤새 API 호출해서 수백만원 청구됨"
- "무한루프로 같은 작업 반복"
- "위험 작업을 승인 없이 실행"

**해결책이 이미 이 저장소에 있음:**

| 파일 | 해결 문제 |
|---|---|
| `cost_estimation.py` | 실행 전 비용 예측 |
| `yolo_budget.py` | 예산 상한 강제 |
| `approval_policy.py` | 위험 작업 승인 게이트 |
| `doom_loop.py` | 무한 삽질 감지 |
| `usage_thresholds.py` | 사용량 임계치 알림 |

이걸 **프레임워크 독립적인 파이썬 패키지로 추출**하는 아이디어.

- 수익모델: 오픈소스로 인지도 → 기업용 대시보드/감사로그 유료화
- 난이도 ⭐⭐⭐☆☆
- 강점: 처음부터 만드는 게 아니라 **검증된 코드에서 추출**하는 것

### B-2. 도메인 특화 인턴

ml-intern의 구조를 다른 분야로 이식.

| 후보 | 역할 | 고객 |
|---|---|---|
| `data-intern` | SQL/dbt 파이프라인 자동 작성 | 데이터팀 |
| `devops-intern` | 테라폼/K8s 설정 + 비용 최적화 | 인프라팀 |
| `quant-intern` | 백테스트 자동 실행 | 트레이딩 |

핵심 재사용 자산: **"넌 지식이 낡았으니 문서 먼저 읽어라" + "이런 실수를 할 것이다"** 프롬프트 패턴

## C그룹 — "지식"을 팔기 (가장 빠름)

### C-1. 콘텐츠 + 강의 ⭐ 자본 0원

소재는 이미 확보되어 있음:

- "HF 공식 에이전트 코드 뜯어보기" 시리즈
- "AI가 돈 못 쓰게 막는 법 — 실제 프로덕션 코드 분석"
- "doom loop: AI 무한 삽질 잡는 법"
- "REVIEW.md로 AI 코드리뷰 봇 길들이기"

채널: 블로그/유튜브(애드센스), 인프런/유데미(강의), 유료 뉴스레터

### C-2. 사내 도입 컨설팅

이 저장소를 레퍼런스로 에이전트 아키텍처 설계 자문.
난이도 ⭐⭐⭐☆☆ / 일당 50~150만원

## 추천 실행 순서

```
지금 당장   →  C-1 콘텐츠      (자본 0원, 오늘 시작 가능)
                    ↓ 인지도 → 문의 유입
1~3개월     →  A-1 외주        (현금 흐름 확보)
                    ↓ 반복 패턴 발견
3~6개월     →  B-1 라이브러리   (확장성)
```

각 단계가 다음 단계의 밑거름이 되는 구조입니다.

## 리스크 관리

| 리스크 | 대응 |
|---|---|
| 🚨 GPU 원가 폭탄 | 하드웨어 화이트리스트 + 예산 상한 필수 |
| 🚨 라이선스 | Apache 2.0 고지문·NOTICE 유지, "HF 공식" 사칭 금지 |
| 🚨 고객 데이터 유출 | `share_traces: false` 필수 |
| 🚨 모델 API 의존 | 가격 인상 대비 로컬 모델 옵션 확보 |
| 🚨 품질 보증 책임 | AI 생성 코드라도 책임은 판매자에게 — 계약서에 범위 명시 |

---

# 7. PHP로 구현할 수 있는가?

## 결론

> **가능합니다.** 단, "전부 PHP"보다는 **하이브리드**를 권장합니다.

## 왜 가능한가 — 핵심 발견

`pyproject.toml` 의존성에 **`torch`도 `numpy`도 없습니다.**

즉, **이 프로젝트는 머신러닝 계산을 직접 하지 않습니다.**

```
ml-intern의 실제 동작:
  1. LLM API에 HTTP 요청
  2. 응답에서 tool_calls JSON 파싱
  3. 도구 실행 (= 또 다른 HTTP 요청)
  4. 결과를 다시 넣고 반복
```

학습은 전부 **HF Jobs에 REST API로 위임**합니다.
즉 "고성능 계산"이 아니라 **"HTTP 오케스트레이션"** 이고, PHP가 못 할 이유가 없습니다.

결정적으로 `https://router.huggingface.co/v1` 은 **OpenAI 호환 규격**이라
PHP의 일반 HTTP 클라이언트로 그대로 호출 가능합니다.

## 부품별 이식 난이도

| 파이썬 부품 | 역할 | PHP 대응 | 난이도 |
|---|---|---|---|
| `litellm` completion | LLM 호출 | OpenAI 호환 HTTP 요청 | 🟢 쉬움 |
| `agent_loop.py` | 루프 | 일반 while 문 | 🟢 쉬움 |
| `hf_jobs`, `sandbox` | GPU 잡 제출 | REST API 호출 | 🟢 쉬움 |
| `github_*` | 깃허브 검색 | GitHub REST API | 🟢 쉬움 |
| `approval_policy`, `yolo_budget` | 승인·예산 | 순수 로직 (외부 의존 없음) | 🟢 쉬움 |
| `doom_loop.py` | 삽질 감지 | 순수 로직 | 🟢 쉬움 |
| `thefuzz` | 파일명 유사도 | PHP 내장 `levenshtein()`, `similar_text()` | 🟢 쉬움 |
| `nbformat`/`nbconvert` | .ipynb 파싱 | `.ipynb`는 JSON → `json_decode()` | 🟢 쉬움 |
| `whoosh` | 문서 전문검색 | Meilisearch / Typesense로 대체 | 🟡 보통 |
| `litellm.token_counter` | 토큰 카운팅 | tiktoken PHP 포트 or 근사치 | 🟡 보통 |
| `fastmcp` | MCP 연동 | PHP SDK 미성숙 (선택 기능이라 생략 가능) | 🟠 까다로움 |
| `asyncio` 큐 구조 | 비동기 병렬 | **최대 난관** | 🔴 어려움 |

## 진짜 어려운 3가지

### 1. 비동기 / 장시간 실행

에이전트 한 턴이 수십 분~수 시간 걸리는데, PHP-FPM은 기본 실행시간 제한이 있습니다.

**해결법:**
- ✅ 큐 워커로 실행 — Laravel Queue + Horizon (가장 현실적)
- ✅ PHP CLI SAPI — `max_execution_time = 0`
- ✅ PHP 8.1+ Fibers — amphp v3, ReactPHP
- ✅ Swoole / RoadRunner — 상주형 런타임

> 💡 **"에이전트 한 턴 = 큐 잡 하나"** 로 설계하면 문제의 대부분이 해결됩니다.

### 2. 토큰 카운팅

컨텍스트 압축 트리거용이라 정밀도가 중요하지 않습니다.
`글자수 ÷ 3.5` 근사치로도 충분합니다.

### 3. MCP 프로토콜

PHP MCP SDK가 파이썬만큼 성숙하지 않습니다.
다만 MCP는 **부가 기능**이라 없어도 에이전트는 정상 동작합니다.

## 권장 설계 A — 하이브리드 ⭐

```
┌─────────────────────────────────────────┐
│   PHP / Laravel  (직접 개발)             │
│   • 회원가입 / 로그인                     │
│   • 결제 (구독, 크레딧)                   │
│   • 관리자 대시보드                       │
│   • 사용량·비용 집계, 요금제 제한          │
│   • 큐 관리 (Horizon)                    │
└──────────────┬──────────────────────────┘
               │ 큐 투입 / 결과 수신
               ↓
┌─────────────────────────────────────────┐
│   Python 워커 (ml-intern 거의 그대로)     │
│   헤드리스 모드 or FastAPI 마이크로서비스  │
└─────────────────────────────────────────┘
```

**장점**
- PHP가 잘하는 것(웹·인증·결제·어드민)에 집중
- 이미 완성된 에이전트를 재사용 → 개발 기간 대폭 단축
- 원본 업데이트를 `git pull`로 흡수
- 추후 에이전트만 PHP로 단계적 이전 가능

**연동 예시**

```php
// Laravel 큐 잡 내부
$process = new Process([
    'ml-intern', '--model', 'zai-org/GLM-5.2:novita', $prompt
]);
$process->setTimeout(7200); // 2시간
$process->run(fn($type, $out) => $this->stream($out));
```

## 권장 설계 B — 순수 PHP

| 역할 | PHP 도구 |
|---|---|
| 프레임워크 | Laravel 11+ (PHP 8.3/8.4) |
| HTTP 클라이언트 | Guzzle 또는 Symfony HttpClient (SSE 스트리밍 지원) |
| LLM 호출 | OpenAI 호환 클라이언트 (`base_url`만 HF Router로 변경) |
| 비동기 | Laravel Queue + Horizon, 또는 amphp v3 (Fibers) |
| 상주 런타임 | Laravel Octane (Swoole / RoadRunner) |
| 문서 검색 | Meilisearch 또는 Typesense (whoosh 대체) |
| 어드민 | Filament |
| 결제 | Laravel Cashier(Stripe), 국내는 포트원/토스페이먼츠 |
| 실시간 스트리밍 | Laravel Reverb (WebSocket) 또는 SSE |

**포팅 순서**

```
1주차 → LLM 호출 + while 루프 (도구 2~3개)
2주차 → 도구 추가 (hf_jobs, github, docs)
3주차 → 승인/예산 가드레일 (핵심 가치)
4주차 → 웹 UI + 결제
```

## 아이디어별 PHP 적합도

| 아이디어 | 적합도 | 이유 |
|---|---|---|
| A-1 파인튜닝 외주 | ➖ 무관 | 도구로 쓰는 것이라 언어 무관 |
| **A-2 자판기 서비스** | 🟢🟢🟢 **최적** | 서비스의 90%가 결제·과금·대시보드 → Laravel 강점 |
| B-1 안전벨트 라이브러리 | 🔴 비추 | 타깃 고객이 파이썬 개발자 |
| B-2 도메인 특화 인턴 | 🟡 가능 | 고객이 PHP 생태계면 오히려 유리 |
| C-1 콘텐츠/강의 | ➖ 무관 | — |

## PHP 구현 시 주의사항

| 함정 | 대응 |
|---|---|
| 🚨 웹 요청에서 에이전트 직접 실행 | 절대 금지 — 반드시 큐 워커 |
| 🚨 타임아웃 | CLI `max_execution_time=0`, 큐 timeout 상향 |
| 🚨 스트리밍 끊김 | Nginx `proxy_buffering off` |
| 🚨 토큰 노출 | HF/GitHub 토큰을 프론트엔드로 내려보내지 말 것 |
| 🚨 동시 실행 폭주 | 유저당 동시 세션 제한 (GPU 비용 폭탄 방지) |

---

# 8. 핵심 요약

| 질문 | 답 |
|---|---|
| **이게 뭐야?** | AI 모델을 자동으로 만들어주는 자율 에이전트 (HF 공식) |
| **언제 써?** | ML 파인튜닝, 논문 기반 리서치, 클라우드 GPU 실험 |
| **안 맞는 경우** | 일반 웹개발, HF 생태계를 안 쓰는 작업 |
| **진짜 가치는?** | 도구 자체보다 **"잘 만든 에이전트"의 설계 패턴 교과서** |
| **돈은?** | 추론·GPU 모두 실비 과금 — 예산 관리 필수 |
| **프라이버시는?** | 기본값이 대화 자동 업로드 — `share_traces: false`로 차단 |
| **PHP로 가능?** | ✅ 가능. ML 계산이 없고 HTTP 오케스트레이션이라 문제없음 |
| **PHP 권장 방식?** | Laravel(제품 껍데기) + Python(에이전트 엔진) 하이브리드 |

## 특히 배울 가치가 높은 파일

| 파일 | 배울 점 |
|---|---|
| `agent/core/agent_loop.py` | 큐 기반 비동기 에이전트 루프 설계 |
| `agent/core/doom_loop.py` | AI 무한 반복 감지 및 교정 프롬프트 주입 |
| `agent/core/approval_policy.py` | 위험 작업 승인 게이트 |
| `agent/core/cost_estimation.py` | 실행 전 비용 예측 |
| `agent/core/yolo_budget.py` | 자동 실행 모드의 예산 상한 |
| `agent/context_manager/manager.py` | 컨텍스트 자동 압축 전략 |
| `agent/prompts/system_prompt_v3.yaml` | 실전형 시스템 프롬프트 작성법 |
| `REVIEW.md` | AI 코드리뷰 봇에게 주는 P0/P1/P2 규칙 설계 |

---

## 참고 링크

- 원본 저장소: https://github.com/huggingface/ml-intern
- 내 저장소: https://github.com/bmshin94/ml-intern
- 공식 데모: https://smolagents-ml-intern.hf.space/
- HF Inference Providers: https://huggingface.co/docs/inference-providers/en/index
- GitHub 토큰 발급 가이드: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- HF Agent Trace Viewer: https://huggingface.co/changelog/agent-trace-viewer
