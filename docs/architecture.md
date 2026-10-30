# AXeL 기술 아키텍처

## 1. 설계 목표
AXeL은 민원 답변 챗봇이 아니라 **학교 민원 처리 workflow를 조율하는 Agent**입니다.

1. LLM 판단을 공식 근거와 학교 조직 정보에 grounding
2. 사건 총괄과 개별 Task 수행자를 분리
3. 사람의 회신이 들어오면 Agent가 다시 실행되는 multi-step loop 구성
4. 반복·감정 표현만으로 상급기관에 넘기지 않는 단계적 지원 판단

## 2. End-to-End
```text
Intake → Multilingual → LLM Triage
                    ↓
          RAG + History + Duty
                    ↓
               Action Router
                    ↓
            Fact Check / Support
                    ↓
               Human Reply
                    ↓
              Agent Re-plan
                    ↓
          Response → Validator
                    ↓
               Human Review
```

## 3. LLM Triage
`agent/classify.py`가 민원 원문을 structured JSON으로 변환합니다. nature, urgency, domain, 핵심 요구, 사실확인 필요 항목, 위험 신호 등을 추출합니다. 모델을 사용할 수 없을 때는 rule fallback으로 접수 workflow를 유지합니다.

## 4. Official RAG
RAG는 모델이 기억만으로 답하는 대신 관련 공식 문서를 먼저 검색해 함께 제공하는 방식입니다.

```text
PDF/HWPX/HTML → Parser → Chunk → Metadata → Retrieval → Agent
```

`ingestion/`은 텍스트·표·페이지 위치를 추출하고, `agent/retrieval.py`는 source_id, chunk_id, locator, page 등의 metadata를 유지한 채 관련 근거를 검색합니다. 검수 baseline과 parser 결과를 함께 사용하며 corrected chunk가 우선합니다.

## 5. History Agent
`agent/history_agent.py`는 단순 민원 횟수가 아니라 새 사실·새 요구·기존 답변 완료 여부·미해결 여부를 비교합니다.

- NONE
- LINK_UNRESOLVED
- REVIEW_FOLLOW_UP
- REVIEW_REPEAT

## 6. School Duty Tool
`agent/routing.py`는 민원 종류 + 발생 경위 + 원문 + LLM domain을 학교 업무분장 데이터와 매칭합니다. 결과는 1순위 담당, 협조자, 승인자, 부재 대체자입니다.

## 7. Action Router
`agent/action_router.py`는 분석을 FACT_CHECK, COOP_FACT_CHECK, ADMIN_REVIEW, HISTORY_REVIEW, RESPONSE_REVIEW 등 실제 업무로 변환합니다.

핵심 데이터 모델은 `case.owner_id != task.assignee_id`입니다. Task를 받은 교사가 자동으로 사건 총괄자가 되지 않습니다.

## 8. 과잉이관 방지
지원/이관은 키워드 하나로 결정하지 않습니다.

```text
정보성 → FRONT_DESK_RESOLVE
사실확인 → STAFF_OR_TEAM
위험 가능성 → ADMIN_REVIEW
내부 해결 부족 + 근거/이력 확인 → SUPPORT 제안 → 사람 승인
```

History Agent의 미해결/후속 판단, RAG 근거, 현재 Task와 학교 조치를 함께 보고 외부 지원 필요성을 제안합니다. 자동 이관하지 않습니다.

## 9. Agent Re-planning
후속 Agent Run은 사건 원문, 초기 classification/routing, 공식 근거, Task 회신, 지원 기록, 선택한 문서 버전을 snapshot으로 읽습니다. 모델은 FACT / CLAIM / UNVERIFIED를 구분하고 missing information, 지원 방향, 행동 옵션, 답변 초안을 반환합니다.

## 10. Deterministic Validator
`agent/validator.py`는 LLM과 별도로 코드 규칙을 적용합니다.

- UNAUTHORIZED_PROMISE
- UNGROUNDED_POLICY_CLAIM
- RISKY_DIRECT_CONTACT
- HUMAN_REVIEW_CLAIM

`auto_send_allowed=false`, `human_review_required=true`를 유지합니다.

## 11. Multilingual / Support
`agent/multilingual.py`: 외국어 감지 → 한국어 내부 처리 → 동일 Agent workflow → 원 언어 답변 번역.

`agent/report_agent.py`: 사건·Task·지원·timeline을 모아 교육지원청 지원/이관 검토용 DRAFT를 생성하며 자동 제출하지 않습니다.

## 12. Data & Runtime
주요 저장 구조: cases, case_intakes, tasks, agent_runs, agent_run_events, support_requests, audit_events, documents, recordings/transcripts.

- Frontend: React + TypeScript + Vite / Vercel
- Backend: FastAPI / Render
- AI: OpenAI Responses API Structured Output
- DB: SQLite (hackathon demo)
- Worker: Agent background worker

## 13. Safety Boundary
자동으로 하지 않는 것:
- 악성/교육활동 침해 법적 확정
- 교사 책임 확정
- 사건 자동 종결
- 보호자 답변 자동 발송
- 교육지원청 자동 제출
- 무검토 담당/승인 변경

Agent는 판단을 대신하기보다 **판단에 필요한 근거와 다음 행동을 준비합니다.**
