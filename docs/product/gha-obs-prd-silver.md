# PRD-3: Bronze → Silver 정규화 (repo / workflow / run / job / step)

> 상위 문서: [gha-obs-values.md](./gha-obs-values.md)

## 달성 가치

- **V1: 조직 전체 CI 현황 가시성** — 레포 경계 없이 조직 전체를 하나의 테이블로 질의할 수 있게 한다
- **V2: 관측 데이터의 완결성** — 중복·순서 뒤바뀜·재실행을 흡수해 하나의 확정 상태로 수렴시킨다
- **V3: 질문 단위 분해 가능성** — 이 PRD가 V3의 본체다

## 배경 / 문제

Bronze는 **이벤트 스트림**이지 **현재 상태**가 아니다. 하나의 워크플로 실행은 `requested → in_progress → completed`로 최소 3건의 이벤트를 남기고, 재실행하면 같은 `run_id`로 이벤트가 또 온다. 이벤트는 순서가 뒤바뀌어 도착할 수 있고, backfill로 중복 도착할 수도 있다.

원본 이벤트에 직접 질의하면 "실행 1건"을 세는 것조차 어렵다. → 이벤트를 **계층 구조를 가진 확정 상태**로 접어 넣는다.

## 범위

**포함**
- Silver 5계층 스키마 정의
- 중복 제거 및 상태 수렴 규칙
- 파생 지표 계산
- 멱등 재처리

**제외**
- Gold 레이어(사용 목적별 집계 테이블), 대시보드, 알림

---

## Silver 스키마

### 차원

**`dim_repo`** — `repo_id`(PK), `full_name`, `owner`, `visibility`, `default_branch`, `updated_at`

**`dim_workflow`** — `workflow_id`(PK), `repo_id`, `name`, `path`, `updated_at`

### 사실

**`fact_run`** — PK `(run_id, run_attempt)`
`repo_id`, `workflow_id`, `run_number`, `event`(push/pull_request/schedule/…), `status`, `conclusion`, `head_branch`, `head_sha`, `actor`, `created_at`, `run_started_at`, `updated_at`, `duration_sec`, `is_rerun`, `business_date`

**`fact_job`** — PK `job_id`
`run_id`, `run_attempt`, `repo_id`, `name`, `status`, `conclusion`, `queued_at`, `started_at`, `completed_at`, `queue_wait_sec`, `duration_sec`, `runner_name`, `runner_group`, `labels[]`, `business_date`

**`fact_step`** — PK `(job_id, step_number)`
`run_id`, `run_attempt`, `repo_id`, `name`, `status`, `conclusion`, `started_at`, `completed_at`, `duration_sec`, `business_date`

> 요청서에는 repo/workflow/job/step 4계층이 명시되었으나, **run 계층을 추가**했다. run 없이는 재실행(`run_attempt`)을 구분할 수 없어 성공률과 소요시간이 모두 왜곡된다. 자세한 근거는 가치 문서 V3 참조.

---

## Acceptance Criteria

### AC3-1: 5계층 스키마 구현

- **설명**: 위 정의대로 `dim_repo`, `dim_workflow`, `fact_run`, `fact_job`, `fact_step` 5개 테이블을 생성한다. 각 계층은 상위 계층의 키를 보유하여 조인 없이도 필터링이 가능해야 한다(예: `fact_step`에 `repo_id` 비정규화 보유).
- **달성 가치**: V3
- **검증 방법**: 5계층 각각을 group-by 축으로 하는 대표 질의 5종이 모두 성공하고 결과가 상호 정합(예: run별 job 수 합계 = 전체 job 수)
- **설계 근거**: `repo_id`를 하위 팩트에 중복 보관하는 것은 정규화 관점에서는 후퇴지만, "특정 레포의 느린 step 찾기"가 3단 조인 없이 파티션 프루닝만으로 끝난다.

### AC3-2: 중복 제거 및 상태 수렴

- **설명**: 동일 엔티티에 대한 다중 이벤트를 하나의 최신 확정 상태로 수렴시킨다.
  - 키: run은 `(run_id, run_attempt)`, job은 `job_id`, step은 `(job_id, step_number)`
  - 우선순위: **터미널 상태(`completed`)가 비터미널 상태를 항상 이긴다.** 같은 등급이면 페이로드의 `updated_at`이 큰 쪽이 이긴다
  - 순서가 뒤바뀐 이벤트(늦게 도착한 `in_progress`)가 이미 확정된 `completed`를 되돌리지 않아야 한다
- **달성 가치**: V2, V3
- **검증 방법**: `completed` → `in_progress` 순서로 이벤트를 주입 → 최종 상태가 `completed`로 유지되는지 확인. 동일 이벤트 3중 주입 → 결과 행 1건
- **설계 근거**: 수신 시각 기준 last-write-wins만 쓰면 backfill이 최신 상태를 과거 상태로 덮어쓴다. 완결성을 위한 backfill이 오히려 데이터를 망가뜨리는 전형적인 사고다.

### AC3-3: 파생 지표 계산

- **설명**: 원본에 없는 다음 지표를 적재 시점에 계산해 컬럼으로 보유한다.
  - `queue_wait_sec`: job의 `queued` 이벤트 시각 → `in_progress` 이벤트 시각 차이 (**수신 시각이 아닌 페이로드 타임스탬프 기준**)
  - `duration_sec`: run/job/step 각각의 시작~종료 차이
  - `is_rerun`: `run_attempt > 1`
  - `business_date`: `run_started_at`의 UTC 일자
- **달성 가치**: V1, V3
- **검증 방법**: 실제 실행 1건에 대해 GitHub UI에 표시된 소요 시간과 계산값을 대조 → 오차 ±2초 이내
- **비고**: 큐 대기 시간은 셀프호스티드 러너 용량 판단의 핵심 지표이므로, 실행 시간과 반드시 분리해 보관한다.

### AC3-4: 비즈니스 일자 기준 재파티셔닝

- **설명**: Silver는 `business_date` 기준으로 파티셔닝한다(`year=/month=/day=`). Bronze는 수신 시각 기준이므로(PRD-2 / AC2-2), 변환 시 파티션 기준이 전환된다. 지각 도착 이벤트는 **해당 비즈니스 일자 파티션을 재작성**한다.
- **달성 가치**: V2, V3
- **검증 방법**: 3일 전 이벤트를 backfill로 주입 → 3일 전 파티션이 갱신되고, 해당 일자 집계값이 변화하는지 확인
- **비고**: 파티션 재작성 중에도 읽기가 깨지지 않도록, 임시 위치에 쓴 뒤 원자적으로 교체하는 방식을 쓴다.

### AC3-5: 멱등 재처리 및 전체 재구축

- **설명**: 동일 Bronze 구간을 몇 번 재처리해도 Silver 결과가 동일하다. Silver 전체를 삭제한 뒤 Bronze만으로 재구축했을 때, 삭제 전과 동일한 결과를 얻는다(Bronze TTL 보관 범위 내).
- **달성 가치**: V2
- **검증 방법**: 특정 일자 Silver 파티션의 체크섬 기록 → 삭제 → 재처리 → 체크섬 일치
- **설계 근거**: 이 성질이 없으면 스키마 변경이나 버그 수정 시 과거 데이터를 고칠 방법이 없어, 잘못된 데이터를 그대로 안고 가게 된다.

### AC3-6: 조직 전체 단일 질의

- **설명**: 레포를 지정하지 않고 조직 전체를 대상으로 하는 질의가 가능하다. 최소한 다음 질문에 단일 쿼리로 답할 수 있어야 한다.
  - 조직 전체에서 최근 7일간 실패율이 가장 높은 워크플로 상위 N개
  - 조직 전체에서 총 실행 시간을 가장 많이 소비하는 job 상위 N개
  - 특정 step이 전체 레포에서 얼마나 자주 실패하는가
- **달성 가치**: V1
- **검증 방법**: 위 세 질의를 실행하여 결과가 반환되고, 레포별 개별 집계의 합과 일치하는지 확인
- **비고**: 이 AC가 V1(가시성)을 데이터 계층에서 담보하는 지점이다. **다만 사람이 보는 화면은 아직 없다** — 조회 인터페이스는 별도 PRD가 필요하다.

---

## 미결 사항

- 변환 실행 방식: 마이크로배치(수 분 주기) vs 일 1회 배치 — 신선도(V1)와 재작성 비용(V4)의 트레이드오프
- `dim_repo` / `dim_workflow`의 이력 관리: 레포 이름 변경 시 과거 데이터를 어떻게 볼 것인가 (현재 스키마는 최신 상태만 보유)
- 삭제된 레포·워크플로의 처리 정책
