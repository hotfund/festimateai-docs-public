# PIPELINE_AUTOMATION_SPEC.md
# 클리닉 DB 구축 파이프라인 자동화 설계

> **버전**: v1.0
> **작성**: 2026-04-22
> **상태**: 기획 확정 — CLI 구현 대기
> **환경**: Claude Desktop (기획 전담)
> **실행 플랫폼**: Railway (Primary) + GitHub Actions (Secondary)

---

## §0. 목적

1인 운영자가 "예외 승인만 하는 CEO 모드"로 클리닉 DB를 운영할 수 있도록, 전체 파이프라인을 자동화한다.

```
기술적 완전자동화: 배치 실행, 데이터 적재, 점수 계산, 후보군 갱신 → Railway 워커가 자동 실행
운영적 판단: 애매한 매칭, specialty 미확인 케이스, 리뷰 큐 승인 → 관리자가 하루 20~30분 검수
```

### 읽기/쓰기 테이블 요약

| 잡 타입 | 읽기 | 쓰기 |
|---------|------|------|
| seed_load | festimate_etl.stg_clinic_import | festimate_etl.etl_batches, festimate_raw.raw_clinic_source, festimate_core.clinic_longlist_seed |
| pre_score | festimate_core.clinic_longlist_seed | festimate_core.clinic_filter_scores |
| homepage_discovery | festimate_core.clinic_filter_scores (provisional) | festimate_core.clinic_homepages |
| khidi_enrichment | festimate_ext.external_provider_raw | festimate_core.clinic_external_links, clinic_accreditations |
| kahf_enrichment | festimate_ext.external_provider_raw | festimate_core.clinic_external_links, clinic_accreditations |
| language_enrichment | festimate_core.clinic_external_links | festimate_core.clinic_language_support |
| score_recalc | clinic_homepages, clinic_accreditations, clinic_language_support | festimate_core.clinic_filter_scores |
| stage2_promote | festimate_core.clinic_filter_scores | festimate_core.clinic_filter_scores (stage2_status 업데이트) |

### 연관 문서

| 문서 | 내용 |
|------|------|
| `CLINIC_LONGLIST_ETL_SPEC.md` v2.0 | 테이블 DDL (etl_batches, raw_clinic_source, clinic_longlist_seed) |
| `CLINIC_2NDLIST_FILTER_SPEC.md` v2.0 | 점수 계산 SQL, clinic_filter_scores DDL |
| `CLINIC_ENRICHMENT_SPEC.md` v2.0 | 보강 테이블 DDL, 조인 로직 |

---

## §1. job_queue 테이블 DDL

모든 자동화 작업은 job_queue를 통해 실행된다. 실패 시 자동 재시도, 최종 실패 시 dead_letter로 이동.

```sql
CREATE TABLE festimate_core.job_queue (
    job_id                        BIGSERIAL PRIMARY KEY,

    -- 잡 식별
    job_type                      TEXT NOT NULL,
    -- 'seed_load'                : 행정안전부 CSV 적재
    -- 'pre_score'                : Pre-score 계산 (Stage 2A 판정 포함)
    -- 'homepage_discovery'       : 홈페이지 탐색 + 검증
    -- 'khidi_enrichment'         : KHIDI CSV 적재 + 매칭
    -- 'kahf_enrichment'          : KAHF 스크래핑 + 매칭
    -- 'language_enrichment'      : 외국어 지원 파싱 + 적재
    -- 'score_recalc'             : Web/Intl 점수 재계산
    -- 'stage2_promote'           : Stage 2B Verified 판정

    -- 잡 상태
    job_status                    TEXT NOT NULL DEFAULT 'pending',
    -- 'pending'                  : 실행 대기
    -- 'running'                  : 실행 중
    -- 'success'                  : 완료
    -- 'failed'                   : 실패 (재시도 대기)
    -- 'retry_wait'               : 재시도 대기 (backoff 중)
    -- 'dead_letter'              : 최대 재시도 초과, 수동 처리 필요

    -- 대상 범위
    target_seed_id                BIGINT REFERENCES festimate_core.clinic_longlist_seed(seed_id),
    -- NULL이면 배치 전체 대상, 특정 seed_id면 단건 재처리
    target_batch_id               BIGINT REFERENCES festimate_etl.etl_batches(batch_id),
    batch_date                    DATE,                           -- 배치 날짜 (idempotency key 구성)

    -- 실행 메타
    priority                      INTEGER NOT NULL DEFAULT 100,  -- 낮을수록 먼저 실행
    scheduled_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at                    TIMESTAMPTZ,
    finished_at                   TIMESTAMPTZ,
    worker_id                     TEXT,                          -- 실행한 워커 식별자

    -- 재시도
    attempt_count                 INTEGER NOT NULL DEFAULT 0,
    max_attempts                  INTEGER NOT NULL DEFAULT 3,
    next_retry_at                 TIMESTAMPTZ,
    last_error                    TEXT,
    error_type                    TEXT,
    -- 'network_error'            : 네트워크 실패 → 재시도
    -- 'parsing_error'            : 파싱 실패 → 재시도 제한
    -- 'matching_error'           : 매칭 실패 → review queue 투입
    -- 'rate_limit'               : API 한도 → backoff 후 재시도
    -- 'permanent_error'          : 영구 실패 → dead_letter

    -- 결과
    result_payload                JSONB,
    rows_processed                INTEGER,
    rows_succeeded                INTEGER,
    rows_failed                   INTEGER,

    -- 감사
    created_at                    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_job_queue_status_scheduled
    ON festimate_core.job_queue (job_status, scheduled_at)
    WHERE job_status IN ('pending', 'retry_wait');
CREATE INDEX idx_job_queue_type_date
    ON festimate_core.job_queue (job_type, batch_date);
CREATE INDEX idx_job_queue_seed_id
    ON festimate_core.job_queue (target_seed_id)
    WHERE target_seed_id IS NOT NULL;
CREATE INDEX idx_job_queue_dead_letter
    ON festimate_core.job_queue (job_status, created_at)
    WHERE job_status = 'dead_letter';
```

---

## §2. Idempotency 규칙

같은 작업을 두 번 실행해도 데이터가 중복되거나 깨지지 않아야 한다.

### 2-1. 중복 방지 키 구성

| 잡 타입 | Idempotency Key |
|---------|----------------|
| seed_load | `(source_name, batch_date)` |
| pre_score | `(seed_id, scoring_version)` |
| homepage_discovery | `(seed_id, discovery_method)` |
| khidi_enrichment | `(ext_raw_id, seed_id)` |
| kahf_enrichment | `(ext_raw_id, seed_id)` |
| language_enrichment | `(seed_id, language_code, support_type, source_system)` |
| score_recalc | `(seed_id, scoring_version)` |
| stage2_promote | `(seed_id, scoring_version)` |

### 2-2. 중복 잡 생성 방지 SQL

```sql
-- 같은 job_type + batch_date가 pending/running/success면 새 잡 생성 안 함
INSERT INTO festimate_core.job_queue (job_type, batch_date, scheduled_at)
SELECT 'seed_load', CURRENT_DATE, NOW()
WHERE NOT EXISTS (
    SELECT 1 FROM festimate_core.job_queue
    WHERE job_type = 'seed_load'
      AND batch_date = CURRENT_DATE
      AND job_status IN ('pending', 'running', 'success')
);
```

### 2-3. 재실행 안전성 원칙

모든 SQL은 `ON CONFLICT ... DO UPDATE` 또는 `ON CONFLICT DO NOTHING` 패턴 사용. INSERT + DELETE 조합 금지.

```
seed_load      → clinic_longlist_seed ON CONFLICT (source_dataset, source_business_key) DO UPDATE
pre_score      → clinic_filter_scores ON CONFLICT (seed_id) DO UPDATE
homepage       → clinic_homepages ON CONFLICT (seed_id, homepage_url) DO NOTHING
enrichment     → clinic_external_links ON CONFLICT (seed_id, ext_raw_id) DO NOTHING
accreditation  → clinic_accreditations ON CONFLICT (...) DO NOTHING
language       → clinic_language_support ON CONFLICT (...) DO NOTHING
```

---

## §3. Retry 정책

오류 유형별로 재시도 전략을 분리한다.

### 3-1. 오류 유형별 정책

| error_type | 설명 | 최대 재시도 | Backoff | 최종 처리 |
|-----------|------|-----------|---------|----------|
| `network_error` | HTTP 타임아웃, 연결 실패 | 5회 | Exponential (30s→60s→120s→300s→600s) | dead_letter |
| `rate_limit` | API 429 응답 | 10회 | Fixed 60s (API 한도 회복 대기) | dead_letter |
| `parsing_error` | HTML 구조 변경, 파싱 실패 | 2회 | Fixed 300s | review queue 투입 |
| `matching_error` | fuzzy 매칭 임계값 미달 | 1회 | 없음 | review queue 투입 |
| `permanent_error` | DB 제약 위반, 스키마 오류 | 0회 | 없음 | dead_letter + 알림 |

### 3-2. Backoff 계산 SQL

```sql
-- next_retry_at 계산
UPDATE festimate_core.job_queue
SET
    job_status    = 'retry_wait',
    attempt_count = attempt_count + 1,
    next_retry_at = NOW() + CASE error_type
        WHEN 'network_error' THEN
            INTERVAL '30 seconds' * POWER(2, LEAST(attempt_count, 4))
        WHEN 'rate_limit' THEN
            INTERVAL '60 seconds'
        WHEN 'parsing_error' THEN
            INTERVAL '300 seconds'
        ELSE INTERVAL '60 seconds'
    END,
    updated_at    = NOW()
WHERE job_id = :job_id
  AND attempt_count < max_attempts;

-- 최대 재시도 초과 → dead_letter
UPDATE festimate_core.job_queue
SET
    job_status = 'dead_letter',
    updated_at = NOW()
WHERE job_id = :job_id
  AND attempt_count >= max_attempts;
```

### 3-3. Review Queue 투입

`parsing_error` 또는 `matching_error` 최종 실패 시 seed를 수동 검토 큐로 이동.

```sql
UPDATE festimate_core.clinic_longlist_seed
SET
    needs_manual_review = TRUE,
    updated_at          = NOW()
WHERE seed_id = :seed_id;

INSERT INTO festimate_core.clinic_seed_classification_log (
    seed_id, classification_stage, decision, decision_reason, decided_by
)
VALUES (
    :seed_id, 'pipeline_error', 'review',
    'job_id=' || :job_id || ' error_type=' || :error_type,
    'auto_retry_exhausted'
);
```

---

## §4. Railway 워커 구조

### 4-1. 배포 단위

```
Railway Project: festimate-pipeline
  ├── Service: pipeline-worker      # Python, 메인 배치 워커
  └── Service: pipeline-crawler     # Python + Playwright, 크롤링 전용 워커
```

MVP에서는 `pipeline-worker` 하나로 시작. 크롤링 부하가 커지면 `pipeline-crawler` 분리.

### 4-2. Worker 실행 루프 (Python)

```python
# pipeline_worker.py (pseudocode)
import time
import psycopg2

def poll_and_execute():
    while True:
        job = claim_next_job()       # job_queue에서 pending 잡 1개 atomic claim
        if job is None:
            time.sleep(10)
            continue
        try:
            execute_job(job)
            mark_success(job)
        except NetworkError as e:
            mark_retry(job, 'network_error', str(e))
        except RateLimitError as e:
            mark_retry(job, 'rate_limit', str(e))
        except ParsingError as e:
            mark_retry(job, 'parsing_error', str(e))
        except Exception as e:
            mark_dead_letter(job, 'permanent_error', str(e))
            alert_operator(job, e)   # Slack / 이메일 알림

def claim_next_job():
    # FOR UPDATE SKIP LOCKED: 다중 워커 동시 실행 시 중복 처리 방지
    return db.fetchone("""
        UPDATE festimate_core.job_queue
        SET job_status = 'running',
            started_at = NOW(),
            worker_id  = %s,
            updated_at = NOW()
        WHERE job_id = (
            SELECT job_id FROM festimate_core.job_queue
            WHERE job_status IN ('pending', 'retry_wait')
              AND (next_retry_at IS NULL OR next_retry_at <= NOW())
            ORDER BY priority ASC, scheduled_at ASC
            LIMIT 1
            FOR UPDATE SKIP LOCKED
        )
        RETURNING *
    """, (WORKER_ID,))
```

### 4-3. 잡 타입별 구현 모듈

| job_type | Python 모듈 | 주요 의존성 |
|---------|-----------|-----------|
| `seed_load` | `workers/seed_load.py` | psycopg2, csv |
| `pre_score` | `workers/pre_score.py` | psycopg2 (SQL 직접 실행) |
| `homepage_discovery` | `workers/homepage_discovery.py` | requests, Google CSE API |
| `khidi_enrichment` | `workers/khidi_enrichment.py` | psycopg2, csv, rapidfuzz |
| `kahf_enrichment` | `workers/kahf_enrichment.py` | requests, BeautifulSoup |
| `language_enrichment` | `workers/language_enrichment.py` | psycopg2 (SQL 직접 실행) |
| `score_recalc` | `workers/score_recalc.py` | psycopg2 (SQL 직접 실행) |
| `stage2_promote` | `workers/stage2_promote.py` | psycopg2 (SQL 직접 실행) |

---

## §5. Cron 스케줄

### 5-1. Railway Cron 설정

```yaml
# railway.json 또는 Procfile 기준
cron:
  - name: daily-seed-check
    schedule: "0 2 * * *"          # 매일 새벽 2시 (KST 기준 UTC+9 → UTC 17:00 전날)
    command: python schedule_jobs.py --job seed_load

  - name: hourly-worker
    schedule: "*/15 * * * *"       # 15분마다 job_queue polling
    command: python pipeline_worker.py --once

  - name: weekly-health-check
    schedule: "0 9 * * 1"          # 매주 월요일 오전 9시
    command: python health_report.py
```

### 5-2. 잡 스케줄 기준

| 잡 타입 | 실행 주기 | 트리거 방식 |
|---------|---------|-----------|
| `seed_load` | 분기 1회 (또는 공공데이터 갱신 시) | 수동 트리거 또는 cron |
| `pre_score` | seed_load 완료 후 자동 | job_queue 연쇄 |
| `homepage_discovery` | pre_score 완료 후 자동 + 주 1회 미발견 재시도 | job_queue 연쇄 + cron |
| `khidi_enrichment` | 반기 1회 (KHIDI 데이터 갱신 주기) | 수동 트리거 |
| `kahf_enrichment` | 월 1회 | cron |
| `language_enrichment` | khidi/kahf_enrichment 완료 후 자동 | job_queue 연쇄 |
| `score_recalc` | enrichment 완료 후 자동 + 수동 재계산 가능 | job_queue 연쇄 |
| `stage2_promote` | score_recalc 완료 후 자동 | job_queue 연쇄 |

---

## §6. 전체 파이프라인 흐름

```
Phase 1. Seed 적재 (seed_load)
  행정안전부 CSV 다운로드
  → stg_clinic_import TRUNCATE + COPY
  → raw_clinic_source INSERT
  → clinic_longlist_seed UPSERT
  → etl_batches 완료 기록
  → pre_score 잡 자동 큐 투입

Phase 2. Pre-score + Stage 2A (pre_score)
  clinic_longlist_seed (normalized, active, target_region)
  → clinic_filter_scores INSERT/UPSERT (pre_score_raw, pre_score_capped)
  → stage2_status = 'provisional' (pre_raw >= 18) 또는 'excluded'
  → homepage_discovery 잡 자동 큐 투입 (provisional 대상)

Phase 3. 홈페이지 탐색 (homepage_discovery)
  provisional 클리닉 대상
  → Google CSE API 또는 Naver Place API 호출
  → clinic_homepages INSERT (verification_status = 'discovered')
  → 기관명 타이틀 매칭으로 'verified' / 'rejected' 판정
  → web_score 계산 → clinic_filter_scores.web_score_raw 업데이트

Phase 4. 외부 보강 (khidi_enrichment → kahf_enrichment → language_enrichment)
  KHIDI CSV 적재 → external_provider_raw
  → clinic_external_links 매칭 (전화 exact → 기관명+지역 exact → fuzzy)
  → clinic_accreditations 반영 (KHIDI 3년 유효)

  KAHF 스크래핑 → external_provider_raw
  → clinic_external_links 매칭
  → clinic_accreditations 반영 (KAHF 4년 유효)

  Medical Korea 스크래핑 → external_provider_raw (specialty, interpretation)
  → clinic_language_support 반영
  → clinic_filter_scores.structured_specialty_flag 업데이트

Phase 5. 점수 재계산 + Stage 2B (score_recalc → stage2_promote)
  clinic_filter_scores.intl_score_raw 계산
  → final_score = pre_capped + web_capped + intl_capped
  → specialty_gate / homepage_gate 판정
  → stage2_status: 'verified' / 'review' / 'excluded'
  → v_clinic_stage2_verified materialized view REFRESH
```

---

## §7. MVP 운영 원칙

```
1. 동적 룰 엔진 없음
   MVP는 하드코딩 SQL로 점수 계산. clinic_filter_rules 테이블은 감사·확장용으로만 유지.
   V2에서 동적 룰 엔진 전환 예정.

2. 예외는 review queue로
   판정 불확실 → stage2_status = 'review' → 관리자 화면에 표시.
   대표님이 하루 20~30분 검수 후 승인/거부.

3. specialty gate는 엄격하게
   web_score_specialty_raw < 20 AND structured_specialty_flag IS NOT TRUE
   → specialty_gate = FALSE → Stage 2B 자동 편입 불가.

4. structured_specialty_flag 초기값 NULL
   MVP 1차 배치에서는 NULL. COALESCE(_, FALSE)로 처리.
   KHIDI CSV 실파일 컬럼 확인 후 enrichment 배치에서 업데이트.

5. 재실행 안전
   모든 INSERT는 ON CONFLICT 패턴 사용. 배치 실패 후 재실행해도 데이터 중복 없음.
```

---

## §8. 장애 복구 및 재실행 규칙

### 8-1. 배치 전체 재실행

```sql
-- 특정 batch_date의 모든 잡을 pending으로 초기화
UPDATE festimate_core.job_queue
SET
    job_status    = 'pending',
    attempt_count = 0,
    next_retry_at = NULL,
    last_error    = NULL,
    started_at    = NULL,
    finished_at   = NULL,
    updated_at    = NOW()
WHERE batch_date = '2026-04-22'
  AND job_status IN ('failed', 'dead_letter');
```

### 8-2. 단건 seed 재처리

```sql
-- 특정 seed_id만 재점수 계산
INSERT INTO festimate_core.job_queue (job_type, target_seed_id, priority, scheduled_at)
VALUES ('score_recalc', :seed_id, 10, NOW());
```

### 8-3. Dead Letter 확인

```sql
-- 수동 처리 필요한 dead_letter 잡 목록
SELECT
    job_id, job_type, target_seed_id, error_type, last_error, attempt_count, updated_at
FROM festimate_core.job_queue
WHERE job_status = 'dead_letter'
ORDER BY updated_at DESC
LIMIT 50;
```

---

## §9. 헬스체크 리포트

매주 월요일 GitHub Actions가 아래 쿼리를 실행하고 Slack/이메일로 요약 전송.

```sql
SELECT
    job_type,
    COUNT(*) FILTER (WHERE job_status = 'success')     AS success,
    COUNT(*) FILTER (WHERE job_status = 'dead_letter') AS dead_letter,
    COUNT(*) FILTER (WHERE job_status = 'retry_wait')  AS retry_wait,
    AVG(EXTRACT(EPOCH FROM (finished_at - started_at))) AS avg_duration_sec
FROM festimate_core.job_queue
WHERE batch_date >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY job_type
ORDER BY job_type;
```

```sql
-- Stage 2 현황
SELECT
    stage2_status,
    COUNT(*) AS cnt
FROM festimate_core.clinic_filter_scores
GROUP BY stage2_status
ORDER BY stage2_status;
```

---

## §10. GitHub Actions 보조 역할

```yaml
# .github/workflows/health-check.yml
name: Weekly Health Check
on:
  schedule:
    - cron: '0 0 * * 1'   # 매주 월요일 00:00 UTC
  workflow_dispatch:        # 수동 트리거 가능

jobs:
  health-report:
    runs-on: ubuntu-latest
    steps:
      - name: Run health check SQL
        run: python scripts/health_report.py
        env:
          DATABASE_URL: ${{ secrets.SUPABASE_DB_URL }}

# .github/workflows/migration.yml
name: DB Migration
on:
  push:
    paths:
      - 'supabase/migrations/**'
  workflow_dispatch:

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - name: Run migration
        run: supabase db push
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
```

---

## §11. 자동화 진행률 로드맵

| 시점 | 자동화율 | 대표님 개입 |
|------|---------|-----------|
| 1개월차 | 60~75% | Review queue 많음, 룰 보정 중 |
| 2~3개월차 | 80~90% | 주 2~3회 검수, 대부분 예외만 |
| 4개월차+ | 90%+ | 월 1~2회 검수, 신규 소스 변경만 |

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-04-22 | 신규 작성 — job_queue DDL, 8개 잡 타입, Idempotency 규칙, Retry 정책, Railway 워커 구조, Cron 스케줄, 전체 파이프라인 흐름(Phase 1~5), MVP 운영 원칙, 장애 복구, 헬스체크 |

*담당: Claude Desktop (기획)*
