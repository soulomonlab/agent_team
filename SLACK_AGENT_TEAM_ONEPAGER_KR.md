# Slack 기반 멀티 에이전트 팀 운영 설계도 (v1)

이 문서는 **"Slack에서 6개 역할 에이전트가 스레드로 협업"** 하는 운영 모델을 바로 구현할 수 있도록,
이벤트/버튼/상태키/DB 스키마/QA 게이트까지 한 장짜리로 정리한 실전 설계안입니다.

---

## 1) 목표와 범위

- 목표: 채널에서 작업 생성 → 스레드에서 역할별 협업 → QA 승인 후 ship(병합/배포 트리거)
- 역할: `captain`, `pm`, `designer`, `fe`, `be`, `qa`
- v1 원칙:
  - Slack App은 **1개 봇**으로 시작
  - 에이전트 정체성은 메시지 prefix로 표현 (`[captain] ...`)
  - 모든 에이전트 산출물은 **JSON only**

---

## 2) 사용자 경험(UX) 플로우

### 2.1 시작

1. 사용자가 채널에서 `/agent start <요청 텍스트>` 실행
2. 봇이 즉시 ack 후, 채널에 "Task 생성" 메시지 게시
3. 해당 메시지 `ts`를 `thread_ts`로 저장하고 스레드를 작업방으로 사용
4. 봇이 스레드 첫 메시지에 작업 카드 + 버튼 표시

### 2.2 스레드 내 운영

- 역할별 산출물은 모두 같은 `thread_ts`에 누적
- `captain`이 분해/할당 → 각 역할 순차 또는 병렬 실행
- 모든 결과는 JSON으로 저장 + Slack에 요약 출력

### 2.3 운영 버튼(스레드 첫 메시지)

- `Status`: 현재 단계/담당/블로커 표시
- `Run next`: 다음 역할 에이전트 실행
- `Ship request`: QA가 pass 상태일 때만 활성 처리
- `QA pass` / `QA fail`: QA 판정 반영

> `/agent status`, `/agent ship` 같은 동작은 스레드 내에서는 slash command 대신 버튼으로 처리

---

## 3) Slack 이벤트 및 액션 라우팅

### 3.1 수신 이벤트

- Slash command: `/agent start`
- Block actions: 버튼 클릭 (`status`, `run_next`, `ship_request`, `qa_pass`, `qa_fail`)
- App mention(선택): 사람이 `@bot rerun fe` 같은 운영 명령

### 3.2 3초 ACK 원칙

- 모든 Slack 진입점에서 먼저 즉시 ack
- 실제 에이전트 실행/LLM 호출은 큐 워커로 비동기 처리
- 완료 후 `chat.postMessage`로 thread 업데이트

### 3.3 중복 처리 방지 키

- 이벤트 단위: `event_id`
- 메시지/액션 단위: `(team_id, channel_id, message_ts|trigger_id|action_ts)`
- 이미 처리된 키는 저장소에서 검사 후 skip

---

## 4) 상태 머신(권장)

`created -> planning -> execution -> qa_pending -> qa_passed|qa_failed -> ship_ready -> shipped`

- `created`: task 생성 직후
- `planning`: captain/pm/design 정리 단계
- `execution`: fe/be 구현 단계
- `qa_pending`: qa 검증 대기
- `qa_passed`: ship 가능
- `qa_failed`: 재작업 필요 (실패 사유 필수)
- `ship_ready`: ship 버튼 승인됨
- `shipped`: 머지/배포 완료

---

## 5) DB 스키마(최소)

### 5.1 tasks

- `task_id` (PK, 예: `TASK-2026-0001`)
- `channel_id`
- `thread_ts`
- `request_text`
- `status` (state machine 값)
- `created_by`
- `created_at`, `updated_at`

### 5.2 task_steps

- `id` (PK)
- `task_id` (FK)
- `agent_name` (`captain|pm|designer|fe|be|qa`)
- `step_order`
- `run_status` (`pending|running|done|failed`)
- `started_at`, `finished_at`

### 5.3 agent_outputs

- `id` (PK)
- `task_id` (FK)
- `agent_name`
- `output_json` (JSONB/TEXT)
- `summary_text`
- `slack_message_ts`
- `created_at`

### 5.4 qa_gate

- `task_id` (PK/FK)
- `qa_status` (`pass|fail|pending`)
- `qa_note`
- `approved_by`
- `approved_at`

### 5.5 idempotency_keys

- `key` (PK)
- `source` (`slash|event|action|worker`)
- `created_at`
- `ttl_at`

---

## 6) 에이전트 실행 규칙(JSON only)

### 6.1 공통 시스템 규칙

- 반드시 단일 JSON 객체만 출력
- 코드블록 금지
- 스키마 불일치 시 재시도 1~2회
- 재시도 실패 시 `run_status=failed` + 에러 로그 남김

### 6.2 권장 공통 스키마

```json
{
  "agent": "captain",
  "task_id": "TASK-2026-0001",
  "status": "done",
  "summary": "한 줄 요약",
  "deliverables": [],
  "risks": [],
  "next_actions": [],
  "needs_human": false
}
```

### 6.3 역할별 최소 책임

- `captain`: 문제 분해, 단계/우선순위/담당 확정
- `pm`: 요구사항 정리, 수용기준(acceptance criteria) 작성
- `designer`: UX 카피/플로우/컴포넌트 제안
- `fe`: 프론트 구현 계획/변경 파일/테스트 포인트
- `be`: API/스키마/비즈니스 로직 계획
- `qa`: 테스트 시나리오, pass/fail, 차단 사유

---

## 7) Slack 메시지 포맷 권장

### 7.1 스레드 첫 카드

- 제목: `Task-xxxx | 현재 상태: planning`
- 본문: 요청 요약, 담당 순서, 현재 블로커
- 액션 버튼: `Status`, `Run next`, `QA pass`, `QA fail`, `Ship request`

### 7.2 역할 보고 메시지

- 헤더: `[agent_name] step done`
- 본문: `summary` + 핵심 3줄
- 첨부: 원본 JSON (짧게 또는 파일/스니펫)

---

## 8) QA 게이트와 GitHub 연동

### 8.1 게이트 규칙

- `qa_status=pass` 전에는 `ship_request` 거부
- `qa_status=fail`이면 상태를 `qa_failed`로 전환, captain 재계획 요청

### 8.2 GitHub 보호 브랜치 연동(v1.5)

- PR 필수 체크: `qa-approved`
- Slack에서 QA pass 시 GitHub Status/Check를 success로 업데이트
- fail이면 failure로 업데이트하여 merge 차단

---

## 9) 구현 순서(권장 5단계)

1. Slack 앱 기동 + `/agent start` + 스레드 생성
2. DB에 `task_id/channel_id/thread_ts/status` 저장
3. `captain -> ... -> qa` 순차 실행 워커 연결
4. 버튼 액션(`status/run_next/qa/ship`) 연결
5. QA 게이트 + GitHub 체크 연동

---

## 10) 운영 체크리스트

- [ ] 모든 Slack 핸들러 3초 내 ack
- [ ] idempotency 저장소 적용
- [ ] JSON 파싱 실패 재시도 + 실패 처리
- [ ] task/thread 단위 로그 추적 가능
- [ ] QA fail 시 재작업 루프 동작
- [ ] ship 전 qa pass 강제

