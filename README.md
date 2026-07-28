# gha-observability

조직 전체 GitHub Actions 실행을 수집·저장·정규화하여, CI 현황을 한 곳에서 볼 수 있게 하는 관측 파이프라인.

현재 단계: **제품 문서 정의** (구현 미착수)

## 파이프라인 개요

```
GitHub Actions (조직 전체 레포)
        │  workflow_run / workflow_job webhook
        ▼
   수집기 ──── 누락 탐지 ──── backfill (delivery 재전송 / REST API)
        │
        ▼
   S3 bronze/   원본 무손실 · 수신시각 기준 일 단위 파티션 · TTL 90일
        │
        ▼
   S3 silver/   repo · workflow · run · job · step 5계층 · 비즈니스 일자 파티션 · TTL 730일
```

## 문서

| 문서 | 내용 |
|------|------|
| [문서 체계 상태 추적](docs/product/gha-obs-doc-tracker.md) | 연결 매트릭스, 위험 진단, 다음 단계 — **여기부터** |
| [제품 가치](docs/product/gha-obs-values.md) | V1~V4 정의 및 우선순위 |
| [PRD-1 이벤트 수집](docs/product/gha-obs-prd-ingestion.md) | Webhook + backfill (AC1-1~1-6) |
| [PRD-2 데이터레이크](docs/product/gha-obs-prd-datalake.md) | S3 bronze 적재 · 파티션 · TTL (AC2-1~2-5) |
| [PRD-3 Silver 정규화](docs/product/gha-obs-prd-silver.md) | Bronze→Silver 5계층 (AC3-1~3-6) |
| [테스트: 이벤트 수집](docs/product/gha-obs-test-ingestion.md) | 시나리오 11개 |
| [테스트: 데이터레이크](docs/product/gha-obs-test-datalake.md) | 시나리오 8개 |
| [테스트: Silver 정규화](docs/product/gha-obs-test-silver.md) | 시나리오 11개 |

가치 4개 → PRD 3개(AC 17개) → 테스트 3개(시나리오 30개). 모든 AC가 가치에 연결되고 테스트로 커버된다.

## 미해결 사항

- 🔴 **제품 소유자 미지정** — TTL·임계값·변환 주기 결정 주체 없음
- 🟡 **조회 인터페이스 부재** — 데이터는 쿼리 가능하나 화면이 없어 V1(가시성) 미완. PRD-4 필요 여부 검토 중

세부 내용은 [상태 추적 문서](docs/product/gha-obs-doc-tracker.md)의 위험 진단 절 참조.
