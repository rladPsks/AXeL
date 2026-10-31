# AXeL | 학교 민원 대응 AI Agent

> **민원이 교사에게 닿기 전에, AXeL이 먼저 일합니다.**

[🌐 Live Demo](https://axel-axedu.vercel.app) · [🏗 기술 구조](docs/architecture.md) · [🎬 시연 가이드](docs/demo-scenario.md)

AXeL은 학교 대표 창구로 들어온 민원을 단순히 교사에게 전달하지 않습니다. **LLM이 민원의 핵심 요구·긴급도·사실확인 필요성을 구조화하고, 공식 문서 RAG·과거 민원 이력·학교 업무분장을 함께 확인해 학교 차원의 대응계획을 준비하는 AI Agent**입니다.

필요한 사실만 담당 교직원에게 요청하고, 회신이 모이면 Agent가 다시 사건을 검토해 보호자 답변 초안과 지원 방향을 제안합니다.

> 공개 배포본은 가상 학교·가상 민원을 사용하는 해커톤 시연 환경입니다.

---

## 문제 | 창구는 하나가 되었지만, 교사는 여전히 혼자일 수 있습니다

학교 대표번호·온라인 창구가 생겨도 민원의 성격을 판단하고 담당을 배분하는 체계가 약하면 민원은 다시 담임·업무담당 교사에게 돌아갑니다.

- **판단 부담** — 정보 문의인지, 사실확인인지, 관리자 검토가 필요한지 사람이 매번 판단
- **근거 탐색 부담** — 학교민원 처리 매뉴얼·법령·정책·업무분장이 서로 흩어짐
- **응대 책임 집중** — 사실을 가장 잘 아는 교사가 확인뿐 아니라 보호자 응대까지 맡기 쉬움
- **과잉이관 위험** — 반복 횟수나 강한 표현만으로 상급자·교육지원청 이관을 서두를 수 있음

AXeL은 교사를 AI로 대체하지 않습니다. **민원과 교사 사이에 학교 차원의 AI 대응 계층을 만듭니다.**

---

## Solution | Agent가 먼저 읽고, 찾고, 비교하고, 일을 나눕니다

```text
민원 접수
   ↓
LLM TRIAGE
유형 · 긴급도 · 핵심 요구 · 사실확인 필요성
   ↓
┌───────────────┬────────────────┬────────────────┐
│ Official RAG  │ History Agent  │ School Duty    │
│ 공식 근거 검색 │ 과거 사건 비교   │ 학교 업무분장 검색 │
└───────────────┴────────────────┴────────────────┘
   ↓
Action Router
담당 · 협조 · 승인 · 필요한 업무 결정
   ↓
필요한 사람에게만 Fact Check / Support Task
   ↓
교직원 회신
   ↓
Agent Re-planning
FACT / CLAIM / UNVERIFIED 재구성
   ↓
Response Agent → Deterministic Validator
   ↓
Human Review → 학교 공식 답변 / 필요한 지원
```

### 핵심 설계

**사건 총괄 ≠ Task 수행자**

담임교사가 사실확인 Task를 수행하더라도 사건 총괄과 보호자 답변 책임은 학교 대응체계에 유지됩니다.

---

## 핵심 기능

| 기능 | 역할 |
|---|---|
| **LLM Triage** | 민원 원문에서 유형·긴급도·핵심 요구·사실확인 필요성을 Structured Output으로 추출 |
| **Official RAG** | 학교민원 처리 매뉴얼·법령·교육부 정책자료에서 관련 근거를 찾아 문서명·페이지와 함께 제공 |
| **History Agent** | 과거 사건과 비교해 미해결 연결·후속·반복 검토 후보를 구분 |
| **School Duty Tool** | 학교 업무분장표에서 1순위 담당·협조자·승인자·대체 담당 탐색 |
| **Action Router** | 분석을 FACT_CHECK·ADMIN_REVIEW·HISTORY_REVIEW 등 실제 업무로 변환 |
| **Agent Re-planning** | 교직원 회신 이후 사건 기록과 근거를 다시 읽고 다음 대응을 재계획 |
| **Response + Validator** | 보호자 답변 초안을 만들고 책임 인정·근거 없는 법령·위험한 직접 연락 등을 코드 규칙으로 재검사 |
| **Multilingual / Support** | 외국어 민원 처리 및 교육지원청 지원·이관 검토 보고서 초안 |

---

## RAG | LLM의 기억이 아니라 공식 근거를 먼저 찾습니다

**RAG(Retrieval-Augmented Generation)** 는 LLM이 기억만으로 답하지 않고 외부 문서에서 관련 내용을 먼저 검색해 그 근거와 함께 판단하도록 하는 방식입니다.

```text
공식 PDF / HWPX / HTML
        ↓
Parser
텍스트 · 표 · 페이지 위치 추출
        ↓
Chunking
검색 가능한 단위로 분할
        ↓
Metadata
source_id · chunk_id · 문서명 · 조문 · PDF page
        ↓
Retrieval
현재 민원과 관련된 근거 검색
        ↓
LLM / Agent
근거 + 사건 기록을 함께 보고 판단
```

RAG에 실제 답이 없으면 학교별 날짜·사실을 LLM이 임의로 만들지 않고 **담당자 사실확인**으로 전환합니다.

---

## History Agent | 반복 횟수만으로 민원을 낙인찍지 않습니다

과거 민원이 있다는 사실만으로 반복·악성 민원으로 판단하지 않습니다.

- `LINK_UNRESOLVED` — 이전 사건이 아직 실질적으로 해결되지 않음
- `REVIEW_FOLLOW_UP` — 기존 답변 이후 새로운 사실·요구가 추가됨
- `REVIEW_REPEAT` — 답변 완료 후 동일 내용 반복 여부를 사람이 검토
- `NONE` — 관련 과거 사건 없음

**민원 횟수보다 기존 답변 완료 여부·새 사실·새 요구·미이행 업무를 먼저 확인합니다.**

---

## 과잉이관 방지 | 필요한 지원은 놓치지 않되, 불필요한 이관은 만들지 않습니다

AXeL은 “위험해 보이는 단어가 있다 → 교육지원청 이관”처럼 동작하지 않습니다.

```text
정보성 문의
→ FRONT_DESK_RESOLVE

사실확인 필요
→ STAFF_OR_TEAM
→ 담당자 확인

안전·침해 가능성
→ ADMIN_REVIEW
→ 관리자 검토

학교 내부 대응만으로 부족
+ 사실/근거/이력 확인
→ SPECIALIST / EXTERNAL SUPPORT 제안
→ 사람 승인
```

반복 횟수·감정·신고 표현만으로 부당 민원이나 외부 이관을 확정하지 않습니다. **공식 근거, 이전 학교 대응, 미해결 업무, 현재 위험 신호를 함께 보고 필요한 경우에만 지원 방향을 제안합니다.**

---

## Example | 생활지도 민원이 들어오면

> “우리 아이가 수업 시간에 다른 학생보다 자주 지적받는다고 합니다. 어떤 상황에서 지도가 이루어졌는지 확인해 주세요. 담임선생님과 통화하고 싶습니다.”

```text
LLM      → 생활지도 관련 사실확인 필요
RAG      → 학교민원 처리 매뉴얼의 담당자 협조 근거
History  → 동일 학생/보호자의 과거 사건 확인
Duty     → 생활체육부장(1순위) + 담임(협조) + 교감(승인)
Action   → 필요한 사실확인 Task 생성
Teacher  → 수업 당시 확인 가능한 사실만 회신
Direct   → 보호자 직접 연락은 현재 요청하지 않음
```

교직원 회신 이후 Agent가 다시 실행되어 **확인된 사실 / 당사자 주장 / 미확인 내용**을 분리하고 보호자 답변 초안을 준비합니다.

---

## Safety | AI는 준비하고, 학교가 결정합니다

- **RAG Grounding** — 법령·정책 주장은 공식 evidence와 연결
- **Deterministic Validator** — LLM과 별도로 코드 규칙으로 위험 표현 재검사
- **No Auto-send** — `auto_send_allowed = false`
- **Human Review** — 담당·승인·콜백·대외 답변·지원/이관은 학교 권한자가 확정
- **Audit Trail** — 사건·Task·Agent Run·검토 이력을 DB에 기록
- **Privacy** — 실제 개인정보·민원 DB·API Key는 Public repository에 포함하지 않음

---

## Architecture

```text
┌──────────────────────────────┐
│ React + TypeScript + Vite    │  Vercel
│ Intake / Case / Task / Review│
└──────────────┬───────────────┘
               │ REST API
┌──────────────▼───────────────┐
│ FastAPI                      │  Render
│ Case / Task / Agent Worker   │
└───────┬───────────────┬──────┘
        │               │
        ▼               ▼
┌───────────────┐  ┌──────────────┐
│ AXeL Agent    │  │ SQLite       │
│ LLM / RAG     │  │ Case / Task  │
│ History / Duty│  │ Agent / Audit│
│ Validator     │  └──────────────┘
└───────┬───────┘
        ▼
   OpenAI API
```

### Tech Stack

- **Frontend:** React 19, TypeScript, Vite, TanStack Query, React Router
- **Backend:** Python, FastAPI, SQLite, Background Worker
- **AI:** OpenAI Structured Output, Agent orchestration, deterministic validation
- **RAG:** PDF/HWPX/HTML parsing, chunking, metadata normalization, lexical retrieval
- **Deployment:** Vercel + Render

[기술 구조 자세히 보기 →](docs/architecture.md)

---

## 해커톤 요구 기능 대응

### Level 1
- 민원 유형·긴급도·담당 분류
- 정보성 민원 공식 근거 검색 및 응답 초안
- 사건·업무·Agent 처리 이력

### Level 2
- 반복·누적 민원 비교 및 미해결/후속/반복 검토
- 공식 근거 기반 관리자·지원 방향 제안
- 응답 초안 및 안전 검증
- 책임 이관 방지 + 과잉이관 방지

### Level 3
- 교육지원청 지원·이관 보고서 초안
- 활동·지원 현황 대시보드
- 다국어 민원 workflow
- 동의 기반 녹취·전사·요약 workflow

> Level 3은 해커톤 데모 구현 범위이며 실제 기관 시스템과의 자동 연계를 의미하지 않습니다.

---

## Live Demo

**https://axel-axedu.vercel.app**

권장 시연 흐름은 [Demo Scenario](docs/demo-scenario.md)를 참고하세요.

1. 생활지도 민원 접수
2. `LLM 분류: LLM`
3. RAG 공식 근거 + 페이지
4. 생활체육부장 / 담임 협조 / 교감 승인
5. `보호자 직접 연락: 현재 단계에서는 요청하지 않음`
6. 교직원 Fact Check
7. Agent 재검토
8. Response + Validator + Human Review

---

## Local Run

### Backend

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python -m backend.seed
$env:SCHOOL_AGENT_RUN_WORKER="1"
python -m uvicorn backend.main:create_app --factory --reload --port 8000
```

### Frontend

```powershell
cd frontend
npm ci
npm run dev
```

브라우저: http://127.0.0.1:5173

환경변수는 `.env.example`을 참고하세요. **API Key·개발 세션 토큰·실제 민원 데이터는 Git에 올리지 마세요.**

---

## Repository

```text
frontend/      React 웹앱
backend/       FastAPI · DB · Task · Agent Run · Audit
agent/         LLM · RAG · History · Duty · Response · Validator
ingestion/     공식 문서 parsing & chunking
data/          공개·가상 RAG 및 업무분장 데이터
docs/          architecture / demo guide
```

---

## Disclaimer

본 저장소와 공개 배포본은 **해커톤 시연용 프로토타입**입니다. 가상 학교·가상 민원 데이터를 사용하며, 실제 학교 도입 시 기관 인증·개인정보보호·학교별 업무분장 및 규정 검증·교육청 연계 절차에 대한 별도 운영 검토가 필요합니다.
