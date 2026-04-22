# 클리닉 롱리스트 ETL 설계서 (CLINIC_LONGLIST_ETL_SPEC.md)

> **버전**: v2.0
> **작성**: 2026-04-22
> **상태**: 기획 확정 — CLI 구현 대기
> **담당**: Claude Desktop (설계) → Claude Code CLI (migration + ETL 스크립트 구현)
> **연관 문서**:
> - `CLINIC_DB_STRATEGY.md` v0.3 — 4단계 파이프라인 전략
> - `CLINIC_2NDLIST_FILTER_SPEC.md` v2.0 — 2차 롱리스트 점수 및 판정 (clinic_filter_scores DDL 포함)
> - `CLINIC_ENRICHMENT_SPEC.md` v2.0 — 보강 테이블 DDL
> - `PIPELINE_AUTOMATION_SPEC.md` v1.0 — 실행 순서, job_queue, Railway 워커 구조

---

## §0. 설계 원칙

**원본은 손대지 않는다. canonical은 원본에서 파생한다.**

공공데이터 CSV는 버전에 따라 헤더가 조금 바뀌거나, 지자체별 값 품질이 흔들릴 수 있다.
원문 행을 JSONB로 통째로 남겨두는 것이 가장 안전하다.
파싱 버그를 발견하면 raw 건드리지 않고 canonical만 재생성하면 된다.

---

## §1. 소스 파일 개요

| 항목 | 내용 |
|------|------|
| 파일명 | 행정안전부_건강_의원.csv (data.go.kr) |
| 인코딩 | EUC-KR (UTF-8 변환 후 적재) |
| 총 행수 | 약 118,750행 (수시 자동 갱신) |
| 핵심 컬럼 | 사업장명, 영업상태구분코드, 인허가일자, 소재지주소, 도로명주소, 소재지전화, X좌표, Y좌표 |
| 좌표계 | EPSG:5174 (GRS80 TM 중부원점 2010) → WGS84 변환 필요 |
| 기관 범위 | 의원·보건소·한의원 등 의원급 전체 |
| 갱신 주기 | 수시 (분기 1회 이상 재적재 권장) |

> ⚠️ CSV 실제 헤더명은 파일 버전별로 달라질 수 있음.
> 첫 배치 시 헤더 자동 수집 후 1회 맞춤 보정 필요.
> `raw_payload` 전체 보존으로 컬럼명 변경에도 재파싱 가능.

---

## §2. PostgreSQL 테이블 설계서

### 2-1. 배치 이력 테이블

어느 날짜 파일로 적재했는지, 몇 건이 들어왔는지,
헤더가 바뀌었는지, 어느 배치부터 데이터 품질이 깨졌는지 추적.

```sql
CREATE SCHEMA IF NOT EXISTS festimate_etl;
CREATE SCHEMA IF NOT EXISTS festimate_raw;
CREATE SCHEMA IF NOT EXISTS festimate_core;
CREATE SCHEMA IF NOT EXISTS festimate_ext;

CREATE TABLE festimate_etl.etl_batches (
    batch_id                 BIGSERIAL PRIMARY KEY,
    source_name              TEXT NOT NULL,                 -- 예: mois_clinic_csv
    source_url               TEXT NOT NULL,                 -- 예: data.go.kr 페이지 URL
    source_download_url      TEXT,                          -- 실제 다운로드 URL
    source_version_label     TEXT,                          -- 예: 행정안전부_건강_의원_20251127
    file_name                TEXT,
    file_checksum_sha256     TEXT,
    file_size_bytes          BIGINT,
    row_count_raw            INTEGER,
    row_count_loaded         INTEGER,
    row_count_rejected       INTEGER DEFAULT 0,
    load_status              TEXT NOT NULL DEFAULT 'started',
                             -- started / success / partial_failed / failed
    started_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finished_at              TIMESTAMPTZ,
    error_message            TEXT,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_etl_batches_source_name ON festimate_etl.etl_batches(source_name);
CREATE INDEX idx_etl_batches_started_at  ON festimate_etl.etl_batches(started_at DESC);
```

### 2-2. 원본 적재 테이블

CSV 원문을 최대한 그대로 저장. 핵심은 `raw_payload JSONB`.
자주 쓰는 핵심 필드만 별도 캐싱해 쿼리 편의 확보.

```sql
CREATE TABLE festimate_raw.raw_clinic_source (
    raw_id                    BIGSERIAL PRIMARY KEY,
    batch_id                  BIGINT NOT NULL REFERENCES festimate_etl.etl_batches(batch_id),

    -- 원본 파일에서 읽은 전체 한 행을 JSON으로 보관
    raw_payload               JSONB NOT NULL,

    -- 자주 쓰는 핵심 원본 필드만 별도 캐싱
    raw_business_name         TEXT,
    raw_business_status       TEXT,
    raw_business_status_code  TEXT,
    raw_permit_date_text      TEXT,
    raw_full_address          TEXT,
    raw_road_address          TEXT,
    raw_phone                 TEXT,
    raw_x_coord               TEXT,
    raw_y_coord               TEXT,
    raw_medical_type          TEXT,
    raw_local_govt_name       TEXT,

    source_row_number         INTEGER,
    raw_hash                  TEXT,   -- 한 행 중복 적재 방지용 해시
    created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_raw_clinic_source_batch_id      ON festimate_raw.raw_clinic_source(batch_id);
CREATE INDEX idx_raw_clinic_source_raw_hash      ON festimate_raw.raw_clinic_source(raw_hash);
CREATE INDEX idx_raw_clinic_source_business_name ON festimate_raw.raw_clinic_source(raw_business_name);
CREATE INDEX idx_raw_clinic_source_address       ON festimate_raw.raw_clinic_source(raw_full_address);
CREATE INDEX idx_raw_clinic_source_payload_gin   ON festimate_raw.raw_clinic_source USING GIN(raw_payload);
```

### 2-3. 1차 롱리스트 canonical 테이블

서비스의 실제 1차 long list seed master.
이후 2차 웹증거, KHIDI/KAHF 오버레이, 숏리스트 승격까지 이 테이블을 중심으로 확장.

```sql
CREATE TABLE festimate_core.clinic_longlist_seed (
    seed_id                         BIGSERIAL PRIMARY KEY,
    batch_id                        BIGINT NOT NULL REFERENCES festimate_etl.etl_batches(batch_id),
    raw_id                          BIGINT REFERENCES festimate_raw.raw_clinic_source(raw_id),

    -- 식별
    source_dataset                  TEXT NOT NULL DEFAULT 'mois_clinic_csv',
    source_row_hash                 TEXT,
    source_business_key             TEXT,   -- name+address+phone 등으로 생성한 내부 키

    -- 기관 기본
    institution_name_raw            TEXT NOT NULL,
    institution_name_norm           TEXT,
    institution_type_raw            TEXT,   -- 원본 기관 구분 / 의료기관 세부유형
    institution_type_norm           TEXT,   -- clinic / dental_clinic / oriental_clinic / public_health 등
    provider_model                  TEXT NOT NULL DEFAULT 'clinic',
                                    -- clinic 고정, 추후 B(병원급) 결합 대비
    business_status_raw             TEXT,
    business_status_code_raw        TEXT,
    is_active_candidate             BOOLEAN NOT NULL DEFAULT FALSE,

    -- 인허가
    permit_date_raw                 TEXT,
    permit_date                     DATE,
    suspended_or_closed_at          DATE,

    -- 주소
    full_address_raw                TEXT,
    road_address_raw                TEXT,
    address_norm                    TEXT,
    city                            TEXT,
    district                        TEXT,
    subdistrict                     TEXT,
    postal_code                     TEXT,

    -- 연락처
    phone_raw                       TEXT,
    phone_norm                      TEXT,

    -- 좌표 (원본 좌표계 보존 + 변환)
    x_coord_raw                     TEXT,
    y_coord_raw                     TEXT,
    x_coord_5174                    NUMERIC(14, 4),   -- EPSG:5174 원본
    y_coord_5174                    NUMERIC(14, 4),
    longitude_wgs84                 NUMERIC(10, 7),   -- WGS84 변환 결과
    latitude_wgs84                  NUMERIC(10, 7),

    -- 지역/사업 판단 보조
    local_government_name           TEXT,
    is_target_region                BOOLEAN NOT NULL DEFAULT FALSE,
    region_filter_reason            TEXT,

    -- 후속 파이프라인용 플래그
    needs_homepage_discovery        BOOLEAN NOT NULL DEFAULT TRUE,
    needs_category_classification   BOOLEAN NOT NULL DEFAULT TRUE,
    needs_foreign_patient_join      BOOLEAN NOT NULL DEFAULT TRUE,
    needs_manual_review             BOOLEAN NOT NULL DEFAULT FALSE,

    -- 품질관리
    data_quality_score              NUMERIC(5,2),
    dedupe_group_key                TEXT,
    canonical_status                TEXT NOT NULL DEFAULT 'loaded',
                                    -- loaded / normalized / deduped / excluded
    exclusion_reason                TEXT,

    -- 감사/운영
    first_loaded_at                 TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_normalized_at              TIMESTAMPTZ,
    updated_at                      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clinic_longlist_seed_batch_id
    ON festimate_core.clinic_longlist_seed(batch_id);
CREATE INDEX idx_clinic_longlist_seed_name_norm
    ON festimate_core.clinic_longlist_seed(institution_name_norm);
CREATE INDEX idx_clinic_longlist_seed_city_district
    ON festimate_core.clinic_longlist_seed(city, district);
CREATE INDEX idx_clinic_longlist_seed_is_active
    ON festimate_core.clinic_longlist_seed(is_active_candidate);
CREATE INDEX idx_clinic_longlist_seed_is_target_region
    ON festimate_core.clinic_longlist_seed(is_target_region);
CREATE INDEX idx_clinic_longlist_seed_provider_model
    ON festimate_core.clinic_longlist_seed(provider_model);
CREATE INDEX idx_clinic_longlist_seed_phone_norm
    ON festimate_core.clinic_longlist_seed(phone_norm);
CREATE INDEX idx_clinic_longlist_seed_dedupe_group_key
    ON festimate_core.clinic_longlist_seed(dedupe_group_key);
CREATE INDEX idx_clinic_longlist_seed_name_trgm
    ON festimate_core.clinic_longlist_seed
    USING GIN (institution_name_norm gin_trgm_ops);
CREATE INDEX idx_clinic_longlist_seed_address_trgm
    ON festimate_core.clinic_longlist_seed
    USING GIN (address_norm gin_trgm_ops);
```

### 2-4. 제외/분류 로그 테이블

"왜 이 병원이 long list에서 빠졌는가"를 설명할 수 있는 테이블.
2차 long list로 넘어갈 때, 단계별 판단 근거가 남는다.

```sql
CREATE TABLE festimate_core.clinic_seed_classification_log (
    log_id                         BIGSERIAL PRIMARY KEY,
    seed_id                        BIGINT NOT NULL REFERENCES festimate_core.clinic_longlist_seed(seed_id),
    classification_stage           TEXT NOT NULL,
                                   -- region_filter / status_filter / category_filter / dedupe
    decision                       TEXT NOT NULL,   -- include / exclude / review
    decision_reason                TEXT,
    evidence_payload               JSONB,
    decided_at                     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    decided_by                     TEXT NOT NULL DEFAULT 'system'
);

CREATE INDEX idx_clinic_seed_classification_log_seed_id
    ON festimate_core.clinic_seed_classification_log(seed_id);
```

---

## §2-5. 점수 및 2차 판정 테이블 (참조)

`clinic_filter_scores` 테이블은 2차 롱리스트 점수 및 Stage 2A/2B 상태를 관리한다.
DDL 전문은 `CLINIC_2NDLIST_FILTER_SPEC.md` §7 참조.

주요 컬럼:
- `stage2_status` — not_scored / provisional / review / verified / excluded
- `pre_score_capped` / `web_score_capped` / `intl_score_capped` — 분야별 cap 점수
- `final_score` — 3개 합산 (max 100)
- `specialty_gate` / `homepage_gate` — Stage 2B 진입 필수 gate
- `structured_specialty_flag` — 외부 소스 specialty 확인 여부 (MVP 초기값 NULL)
- `scoring_version` — 'v2_mvp'

> **실행 순서 및 잡 오케스트레이션은 `PIPELINE_AUTOMATION_SPEC.md` 참조.**

```
```

---

## §3. ETL 컬럼 매핑표

### 3-1. 핵심 매핑표

원본 헤더 후보는 버전별로 상이할 수 있음. 첫 배치 시 헤더 확인 후 1회 보정.

| 우선순위 | 소스 의미 | 원본 헤더 후보 (버전별 상이 가능) | 타깃 컬럼 | 변환 규칙 |
|---------|---------|-------------------------------|---------|---------|
| 1 | 사업장명 | 사업장명 | `institution_name_raw` | trim, 연속 공백 제거 |
| 2 | 영업상태명 | 영업상태명, 영업상태 | `business_status_raw` | 원문 보존 |
| 3 | 영업상태코드 | 영업상태구분코드, 영업상태코드 | `business_status_code_raw` | 문자열 보존 |
| 4 | 인허가일자 | 인허가일자, 허가일자 | `permit_date_raw`, `permit_date` | YYYYMMDD / YYYY-MM-DD 파싱 |
| 5 | 소재지 전체주소 | 소재지전체주소, 소재지주소 | `full_address_raw` | 원문 보존 후 정규화 |
| 6 | 도로명주소 | 도로명전체주소, 도로명주소 | `road_address_raw` | 없으면 NULL |
| 7 | 전화번호 | 전화번호, 소재지전화 | `phone_raw`, `phone_norm` | 숫자/하이픈 정규화 |
| 8 | X좌표 | 좌표정보x(epsg5174), x좌표, 좌표X | `x_coord_raw`, `x_coord_5174` | numeric 캐스팅 |
| 9 | Y좌표 | 좌표정보y(epsg5174), y좌표, 좌표Y | `y_coord_raw`, `y_coord_5174` | numeric 캐스팅 |
| 10 | 기관 세부유형 | 업태구분명, 의료기관종별명, 영업장업종명 | `institution_type_raw` | 의원/한의원/보건소 등 분류 |
| 11 | 지자체명 | 관리기관명, 개방자치단체코드 기반 | `local_government_name` | 코드→명칭 변환 가능 |
| 12 | 폐업/휴업일자 | 폐업일자, 휴업종료일자 등 | `suspended_or_closed_at` | 존재 시 날짜 파싱 |
| 13 | 우편번호 | 우편번호 | `postal_code` | 문자열 보존 |
| 14 | 데이터 업데이트 기준 | 파일 배치 메타 | `batch_id` 연계 | ETL 메타에서 관리 |

### 3-2. 파생 컬럼 규칙

| 타깃 컬럼 | 생성 규칙 |
|---------|---------|
| `institution_name_norm` | 기관명 소문자화(영문), 특수문자 제거, 괄호/지점명 보조 분리 |
| `institution_type_norm` | `institution_type_raw` → clinic / oriental_clinic / dental_clinic / public_health / unknown 매핑 |
| `is_active_candidate` | 영업상태가 운영/영업/정상 계열이면 true, 휴업/폐업/말소 계열이면 false |
| `address_norm` | 도로명 우선, 없으면 지번주소. 시/구/동 공백 정리 |
| `city` | 주소 파싱 — 서울/인천/부산/대구/대전/광주/청주 추출 |
| `district` | 주소의 시군구 추출 |
| `subdistrict` | 동/읍/면 추출 |
| `is_target_region` | `city IN ('서울','인천','부산','대구','대전','광주','청주')` |
| `provider_model` | A 데이터는 기본값 `'clinic'` |
| `source_business_key` | `hash(name_norm + address_norm + phone_norm)` |
| `dedupe_group_key` | `name_norm + city + district + phone_norm` 기반 생성 |
| `needs_manual_review` | 전화번호 없음 + 주소 불명확 + 이름 중복 심한 경우 true |

---

## §4. ETL 처리 순서

**Step 1. 파일 수집**
```
data.go.kr → 행정안전부_건강_의원 최신 파일 다운로드
파일명 / 버전 / 크기 / SHA-256 체크섬 → etl_batches INSERT (load_status = 'started')
동일 체크섬이면 SKIP
```

**Step 2. raw 적재**
```
CSV 헤더 그대로 읽기
각 행 전체 → raw_payload JSONB 저장
핵심 필드 → raw_* 캐싱 컬럼
raw_hash 기준 중복 SKIP
```

**Step 3. canonical 적재**
```
핵심 필드 정규화 → clinic_longlist_seed INSERT
주소 파싱, 전화 정규화, 상태 플래그 계산
지역 필터는 우선 플래그만 (is_target_region) — 물리 삭제 없음
source_business_key ON CONFLICT → UPSERT
```

**Step 4. 품질 점검**
```
institution_name_raw IS NULL
주소 전체 공란
좌표 비정상 (한국 범위 이탈)
상태 미해석 (is_active_candidate가 명확히 설정 안 됨)
중복 과다
→ clinic_seed_classification_log에 기록
```

**Step 5. 중복 키 생성**
```
완전 중복 → canonical_status = 'excluded', exclusion_reason = 'duplicate'
유사 중복 → dedupe_group_key로 묶고 후속 단계에서 처리
```

---

## §5. PostgreSQL ETL SQL

### 5-1. canonical 적재 (raw → clinic_longlist_seed)

```sql
INSERT INTO clinic_longlist_seed (
    batch_id, raw_id,
    source_dataset, source_row_hash, source_business_key,
    institution_name_raw, institution_name_norm,
    institution_type_raw, institution_type_norm,
    provider_model,
    business_status_raw, business_status_code_raw, is_active_candidate,
    permit_date_raw, permit_date,
    full_address_raw, road_address_raw, address_norm,
    city, district, subdistrict,
    phone_raw, phone_norm,
    x_coord_raw, y_coord_raw, x_coord_5174, y_coord_5174,
    local_government_name,
    is_target_region,
    needs_homepage_discovery, needs_category_classification, needs_foreign_patient_join,
    data_quality_score, canonical_status,
    first_loaded_at, updated_at
)
SELECT
    r.batch_id,
    r.raw_id,
    'mois_clinic_csv',
    r.raw_hash,
    md5(
        coalesce(lower(trim(r.raw_business_name)), '') || '|' ||
        coalesce(lower(trim(r.raw_full_address)), '') || '|' ||
        coalesce(regexp_replace(coalesce(r.raw_phone, ''), '[^0-9]', '', 'g'), '')
    ) AS source_business_key,

    r.raw_business_name AS institution_name_raw,
    lower(regexp_replace(trim(coalesce(r.raw_business_name, '')), '\s+', ' ', 'g'))
        AS institution_name_norm,

    r.raw_medical_type AS institution_type_raw,
    CASE
        WHEN r.raw_medical_type ILIKE '%치과%'  THEN 'dental_clinic'
        WHEN r.raw_medical_type ILIKE '%한의%'  THEN 'oriental_clinic'
        WHEN r.raw_medical_type ILIKE '%보건소%' THEN 'public_health'
        WHEN r.raw_medical_type IS NOT NULL      THEN 'clinic'
        ELSE 'unknown'
    END AS institution_type_norm,

    'clinic' AS provider_model,

    r.raw_business_status,
    r.raw_business_status_code,
    CASE
        WHEN r.raw_business_status IN ('영업', '정상', '운영중') THEN TRUE
        ELSE FALSE
    END AS is_active_candidate,

    r.raw_permit_date_text,
    CASE
        WHEN r.raw_permit_date_text ~ '^\d{8}$'
            THEN to_date(r.raw_permit_date_text, 'YYYYMMDD')
        WHEN r.raw_permit_date_text ~ '^\d{4}-\d{2}-\d{2}$'
            THEN to_date(r.raw_permit_date_text, 'YYYY-MM-DD')
        ELSE NULL
    END AS permit_date,

    r.raw_full_address,
    r.raw_road_address,
    COALESCE(NULLIF(trim(r.raw_road_address), ''), trim(r.raw_full_address))
        AS address_norm,

    CASE
        WHEN coalesce(r.raw_road_address, r.raw_full_address) LIKE '서울%'           THEN '서울'
        WHEN coalesce(r.raw_road_address, r.raw_full_address) LIKE '인천%'           THEN '인천'
        WHEN coalesce(r.raw_road_address, r.raw_full_address) LIKE '부산%'           THEN '부산'
        WHEN coalesce(r.raw_road_address, r.raw_full_address) LIKE '대구%'           THEN '대구'
        WHEN coalesce(r.raw_road_address, r.raw_full_address) LIKE '대전%'           THEN '대전'
        WHEN coalesce(r.raw_road_address, r.raw_full_address) LIKE '광주%'           THEN '광주'
        WHEN coalesce(r.raw_road_address, r.raw_full_address) LIKE '충청북도 청주시%' THEN '청주'
        WHEN coalesce(r.raw_road_address, r.raw_full_address) LIKE '청주시%'         THEN '청주'
        ELSE NULL
    END AS city,

    NULL AS district,     -- Python ETL에서 파싱 권장
    NULL AS subdistrict,

    r.raw_phone,
    NULLIF(regexp_replace(coalesce(r.raw_phone, ''), '[^0-9]', '', 'g'), '')
        AS phone_norm,

    r.raw_x_coord,
    r.raw_y_coord,
    NULLIF(r.raw_x_coord, '')::numeric,
    NULLIF(r.raw_y_coord, '')::numeric,

    r.raw_local_govt_name,

    CASE
        WHEN coalesce(r.raw_road_address, r.raw_full_address) LIKE '서울%'
          OR coalesce(r.raw_road_address, r.raw_full_address) LIKE '인천%'
          OR coalesce(r.raw_road_address, r.raw_full_address) LIKE '부산%'
          OR coalesce(r.raw_road_address, r.raw_full_address) LIKE '대구%'
          OR coalesce(r.raw_road_address, r.raw_full_address) LIKE '대전%'
          OR coalesce(r.raw_road_address, r.raw_full_address) LIKE '광주%'
          OR coalesce(r.raw_road_address, r.raw_full_address) LIKE '충청북도 청주시%'
          OR coalesce(r.raw_road_address, r.raw_full_address) LIKE '청주시%'
        THEN TRUE ELSE FALSE
    END AS is_target_region,

    TRUE, TRUE, TRUE,   -- needs_* 초기값

    50.00,              -- data_quality_score 초기값 (정규화 단계에서 재계산)
    'loaded',
    NOW(), NOW()

FROM raw_clinic_source r
WHERE r.batch_id = :batch_id

ON CONFLICT (source_business_key) DO UPDATE SET
    batch_id               = EXCLUDED.batch_id,
    raw_id                 = EXCLUDED.raw_id,
    institution_name_raw   = EXCLUDED.institution_name_raw,
    institution_name_norm  = EXCLUDED.institution_name_norm,
    business_status_raw    = EXCLUDED.business_status_raw,
    business_status_code_raw = EXCLUDED.business_status_code_raw,
    is_active_candidate    = EXCLUDED.is_active_candidate,
    address_norm           = EXCLUDED.address_norm,
    city                   = EXCLUDED.city,
    phone_norm             = EXCLUDED.phone_norm,
    is_target_region       = EXCLUDED.is_target_region,
    updated_at             = NOW()
    -- needs_* FALSE가 된 컬럼은 보존
    -- canonical_status가 'deduped' 이상이면 덮어쓰지 않음 (WHERE 조건 추가 권장)
;
```

> `ON CONFLICT (source_business_key)` 동작을 위해 `source_business_key`에 UNIQUE 인덱스 필요:
> `CREATE UNIQUE INDEX idx_longlist_seed_biz_key ON clinic_longlist_seed(source_business_key);`

---

## §6. 정규화 룰

### 6-1. 기관명 정규화

- `( )`, `[ ]`, 지점명, 불필요 특수문자 분리
- "의원", "클리닉", "피부과", "성형외과"는 원본 보존
- `institution_name_norm`: 비교용 소문자 정규화 (퍼지 매칭·dedup 기준)

### 6-2. 전화번호 정규화

```python
import re

def normalize_phone(raw: str) -> str | None:
    if not raw or not raw.strip():
        return None
    digits = re.sub(r'[^\d]', '', raw.strip())
    if len(digits) == 10 and digits.startswith('02'):
        return f"{digits[:2]}-{digits[2:6]}-{digits[6:]}"
    if len(digits) == 11 and digits.startswith('0'):
        return f"{digits[:3]}-{digits[3:7]}-{digits[7:]}"
    if len(digits) == 9 and digits.startswith('02'):
        return f"{digits[:2]}-{digits[2:5]}-{digits[5:]}"
    return raw.strip()

# 010- 계열 개인휴대폰은 향후 병원 대표번호와 구분 태그 필요
```

### 6-3. 주소 정규화

- 도로명주소 우선, 없으면 지번주소 fallback
- 서울특별시 → 서울, 부산광역시 → 부산 시도 표준화
- `district`: `address_norm`에서 시군구 파싱 (Python 권장, SQL fallback 제공)

### 6-4. 상태값 정규화

| 원본값 패턴 | is_active_candidate |
|-----------|-------------------|
| 영업 / 정상 / 운영중 | TRUE |
| 폐업 / 휴업 / 말소 / 취소 | FALSE |
| 그 외 (미해석) | FALSE + `needs_manual_review = TRUE` |

### 6-5. 좌표 변환 (EPSG:5174 → WGS84)

```python
from pyproj import Transformer

_tf = Transformer.from_crs("EPSG:5174", "EPSG:4326", always_xy=True)

def convert_coords(x: float, y: float) -> tuple[float, float] | tuple[None, None]:
    if not x or not y:
        return None, None
    try:
        lon, lat = _tf.transform(x, y)
        if not (33 <= lat <= 38 and 124 <= lon <= 130):  # 한국 범위 검증
            return None, None
        return round(lon, 7), round(lat, 7)
    except Exception:
        return None, None
```

> SQL 단계에서는 x_coord_5174 / y_coord_5174까지만 저장.
> longitude_wgs84 / latitude_wgs84는 Python ETL 정규화 단계에서 채움.

---

## §7. 테이블 관계도 (ERD)

```
etl_batches
    │ batch_id (PK)
    │
    ├──────────────────────────────────────┐
    │                                      │
    ▼                                      ▼
raw_clinic_source                 clinic_longlist_seed
    │ raw_id (PK)                      │ seed_id (PK)
    │ batch_id (FK)                    │ batch_id (FK → etl_batches)
    │ raw_payload JSONB                │ raw_id (FK → raw_clinic_source)
    │ raw_* 캐싱 컬럼                  │ source_business_key (UNIQUE)
    │ raw_hash                         │ is_active_candidate
    │                                  │ is_target_region
    │                                  │ needs_* 플래그
    │                                  │ canonical_status
    │                                  │
    │                                  ▼
    │                        clinic_seed_classification_log
    │                              log_id (PK)
    │                              seed_id (FK → clinic_longlist_seed)
    │                              classification_stage
    │                              decision (include/exclude/review)
    │                              evidence_payload JSONB
    │
    └── [2차 단계: CLINIC_2NDLIST_SPEC.md 참조]
            clinic_seed_enrichment_web
            clinic_seed_enrichment_khidi
```

---

## §8. 배치 실행 순서

```
[Step 0] 파일 검증
  SHA-256 계산 → etl_batches.file_checksum_sha256 중복 확인 → 동일하면 SKIP

[Step 1] etl_batches INSERT
  load_status = 'started'

[Step 2] raw_clinic_source 적재 (Python pandas → psycopg2)
  EUC-KR → UTF-8 변환
  전체 행 raw_payload JSONB 저장
  핵심 필드 캐싱
  raw_hash 중복 행 SKIP
  → row_count_raw 집계

[Step 3] clinic_longlist_seed UPSERT (§5-1 SQL 실행)
  source_business_key ON CONFLICT → 갱신
  → row_count_loaded 집계

[Step 4] Python 정규화 보완
  district / subdistrict 파싱 (SPLIT_PART SQL fallback 검증)
  longitude_wgs84 / latitude_wgs84 변환 (pyproj)
  data_quality_score 계산
  canonical_status → 'normalized'

[Step 5] 분류 로그 기록
  region_filter: is_target_region = FALSE → 로그
  status_filter: is_active_candidate = FALSE → 로그
  category_filter: K-Beauty 키워드 미해당 → 로그

[Step 6] etl_batches 완료 갱신
  load_status = 'success'
  finished_at = NOW()
  row_count_raw / row_count_loaded / row_count_rejected 확정
```

---

## §9. 인덱스 전략 요약

| 인덱스 | 용도 |
|--------|------|
| `(source_name)` on etl_batches | 소스별 배치 조회 |
| `(started_at DESC)` on etl_batches | 최신 배치 빠른 접근 |
| `(batch_id)` on raw_clinic_source | 배치별 raw 조회 |
| `(raw_hash)` on raw_clinic_source | 중복 행 방지 |
| `GIN(raw_payload)` on raw_clinic_source | JSONB 내부 키 검색 |
| `UNIQUE(source_business_key)` on clinic_longlist_seed | UPSERT 기준키 |
| `(city, district)` on clinic_longlist_seed | 지역별 필터 |
| `(is_active_candidate)` on clinic_longlist_seed | 영업 중 필터 |
| `(is_target_region)` on clinic_longlist_seed | 대상 지역 필터 |
| `(dedupe_group_key)` on clinic_longlist_seed | 중복 그룹 조회 |
| `(seed_id)` on clinic_seed_classification_log | 기관별 로그 조회 |

---

## §10. 재적재 절차

```
1. 새 파일 다운로드 (data.go.kr)
2. SHA-256 확인 → 동일 파일이면 SKIP
3. etl_batches INSERT (load_status = 'started')
4. raw_clinic_source: raw_hash 기준 중복 SKIP
5. clinic_longlist_seed: source_business_key ON CONFLICT UPSERT
   - is_active_candidate / address_norm / phone_norm 갱신
   - needs_* = FALSE 이미 된 것 보존
   - canonical_status = 'deduped' 이상이면 최소 변경
6. etl_batches 완료 기록
```

---

## §11. 예상 수량

| 단계 | 행수 |
|------|------|
| raw_clinic_source 전체 적재 | ~118,750 |
| is_active_candidate = true (서울, K-Beauty 키워드) | ~3,000~5,000 |
| 2차 웹증거 후 유효 | ~500~1,000 |
| 숏리스트 후보 (6-gate) | ~100~200 |
| 최종 숏리스트 | 40~60 |

---

## §12. CLI 구현 인수인계

```
[우선 1] 마이그레이션 파일
  supabase/migrations/ — 4개 테이블 순서대로
  etl_batches → raw_clinic_source → clinic_longlist_seed → clinic_seed_classification_log
  + UNIQUE INDEX on clinic_longlist_seed(source_business_key)

[우선 2] Python ETL 스크립트
  data/scripts/load_mohw_clinic.py
  의존: pandas, psycopg2, pyproj
  인터페이스:
    python load_mohw_clinic.py --file 파일.csv [--dry-run] [--batch-label v20260422]
  dry-run: 적재 없이 row_count + is_kbeauty_candidate 수량만 출력

[우선 3] 필터 검증
  dry-run → is_active_candidate 수량 확인
  너무 적으면 키워드 추가, 너무 많으면 정밀화

[우선 4] 실 적재 후 확인 쿼리
  SELECT canonical_status, COUNT(*) FROM clinic_longlist_seed GROUP BY 1;
  SELECT classification_stage, decision, COUNT(*)
    FROM clinic_seed_classification_log GROUP BY 1, 2 ORDER BY 1, 2;
```

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-04-22 | 초안 |
| v1.1 | 2026-04-22 | 확정 DDL 반영. clinic_seed_classification_log 추가. |
| v1.2 | 2026-04-22 | 전면 보강. 핵심 매핑표 14행 + 파생 컬럼 규칙 추가. ETL 처리 순서 Step 1-5 추가. canonical 적재 INSERT/UPSERT SQL 전체본 추가. 정규화 룰 4종 추가. ERD 관계도 추가. 배치 실행 순서 Step 0-6 추가. 인덱스 전략 표 추가. |

*담당: Claude Desktop (설계)*
*구현: Claude Code CLI — `data/scripts/load_mohw_clinic.py`, `supabase/migrations/`*
