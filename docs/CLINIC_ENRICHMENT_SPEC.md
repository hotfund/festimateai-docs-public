# CLINIC_ENRICHMENT_SPEC.md
# 클리닉 보강 테이블 설계 + 조인 로직 (PostgreSQL)

> **버전**: v2.0
> **작성**: 2026-04-22
> **상태**: 기획 확정
> **환경**: Claude Desktop (기획 전담)
> **변경**: v1.0 전면 재작성 — external_provider_raw 단일 테이블, accreditations 분리, structured_specialty 컬럼 추가

---

## §0. 목적

Stage 2A Provisional 클리닉에 4종 외부 데이터를 부착한다.

| 소스 | 테이블 | 제공 정보 |
|------|--------|----------|
| KHIDI 외국인환자 유치기관 | `external_provider_raw` (source_system='KHIDI_REGISTERED') | 위치·서비스·시설·담당분야, 유효기간 3년 |
| KAHF 회원 클리닉 | `external_provider_raw` (source_system='MEDICAL_KOREA_KAHF') | 외국인환자 특화서비스·환자안전체계 평가, 유효기간 4년 |
| Medical Korea provider 페이지 | `external_provider_raw` (source_system='MEDICAL_KOREA_PROVIDER') | Website/SNS, Interpretation(언어지원), specialty |
| 홈페이지 발견 | `clinic_homepages` | 공식 홈페이지 URL, 검증 여부 |

### 설계 원칙

```
외부 원천 raw 적재 (external_provider_raw)
  → 정규화 + 매칭 (clinic_external_links)
  → 확정 결과 반영 (clinic_accreditations / clinic_language_support / clinic_homepages)
  → structured_specialty_flag 업데이트 (clinic_filter_scores)
```

raw → link → normalized service tables 구조. 소스가 변경돼도 raw만 재적재하면 됩니다.

---

## §1. 외부 공급자 Raw 테이블

KHIDI / KAHF / Medical Korea 원본을 하나의 테이블에 `source_system`으로 구분하여 적재한다.

```sql
CREATE SCHEMA IF NOT EXISTS festimate_ext;

CREATE TABLE festimate_ext.external_provider_raw (
    ext_raw_id                    BIGSERIAL PRIMARY KEY,
    batch_id                      BIGINT NOT NULL
                                      REFERENCES festimate_etl.etl_batches(batch_id),

    -- 소스 구분
    source_system                 TEXT NOT NULL,
    -- 'KHIDI_REGISTERED'         : 외국인환자 유치기관 등록 CSV
    -- 'MEDICAL_KOREA_KAHF'       : Medical Korea KAHF 인증 기관
    -- 'MEDICAL_KOREA_PROVIDER'   : Medical Korea provider detail 페이지

    source_url                    TEXT,
    source_record_id              TEXT,                   -- 소스 내부 ID

    -- 기관 정보
    provider_name_raw             TEXT NOT NULL,
    provider_name_norm            TEXT,                   -- 매칭용 정규화
    provider_type_raw             TEXT,
    address_raw                   TEXT,
    address_norm                  TEXT,
    city                          TEXT,
    district                      TEXT,
    phone_raw                     TEXT,
    phone_norm                    TEXT,
    email_raw                     TEXT,
    website_url                   TEXT,
    website_domain                TEXT,

    -- 전문분야 (structured_specialty_flag 판정의 핵심)
    specialty_raw                 TEXT,
    -- 예: 'DERMATOLOGY', 'PLASTIC_SURGERY', 'MEDICAL BEAUTY', 'Skin & Plastic Surgery'
    specialty_norm                TEXT,
    -- 정규화값: 'dermatology' / 'plastic_surgery' / 'medical_beauty'

    -- 외국인환자 등록 (KHIDI)
    foreign_patient_registered    BOOLEAN,
    registration_status           TEXT,
    registration_valid_from       DATE,
    registration_valid_until      DATE,                   -- 등록일로부터 3년

    -- 인증 (KAHF)
    accreditation_type            TEXT,
    accreditation_status          TEXT,
    accreditation_valid_from      DATE,
    accreditation_valid_until     DATE,                   -- 인증일로부터 4년

    -- 외국어 지원 (Medical Korea Interpretation 필드)
    interpretation_raw            TEXT,
    -- 예: 'English, Japanese, Chinese, Russian'

    -- 기타 서비스 정보
    service_raw                   TEXT,
    facility_raw                  TEXT,

    -- 원본 전체 보존
    raw_payload                   JSONB NOT NULL,
    scraped_at                    TIMESTAMPTZ,
    created_at                    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ext_raw_source_system
    ON festimate_ext.external_provider_raw (source_system);
CREATE INDEX idx_ext_raw_phone_norm
    ON festimate_ext.external_provider_raw (phone_norm);
CREATE INDEX idx_ext_raw_city_district
    ON festimate_ext.external_provider_raw (city, district);
CREATE INDEX idx_ext_raw_name_trgm
    ON festimate_ext.external_provider_raw
    USING GIN (provider_name_norm gin_trgm_ops);
CREATE INDEX idx_ext_raw_address_trgm
    ON festimate_ext.external_provider_raw
    USING GIN (address_norm gin_trgm_ops);
CREATE INDEX idx_ext_raw_specialty_norm
    ON festimate_ext.external_provider_raw (specialty_norm);
```

### specialty_norm 정규화 규칙

```sql
UPDATE festimate_ext.external_provider_raw
SET specialty_norm = CASE
    WHEN specialty_raw ILIKE '%dermatol%'     THEN 'dermatology'
    WHEN specialty_raw ILIKE '%plastic%'      THEN 'plastic_surgery'
    WHEN specialty_raw ILIKE '%medical beauty%'
      OR specialty_raw ILIKE '%medical_beauty%' THEN 'medical_beauty'
    WHEN specialty_raw ILIKE '%skin%'         THEN 'dermatology'
    WHEN specialty_raw ILIKE '%aesthetic%'    THEN 'medical_beauty'
    WHEN specialty_raw ILIKE '%성형%'          THEN 'plastic_surgery'
    WHEN specialty_raw ILIKE '%피부%'          THEN 'dermatology'
    WHEN specialty_raw ILIKE '%미용%'          THEN 'medical_beauty'
    ELSE lower(trim(specialty_raw))
END
WHERE specialty_raw IS NOT NULL;
```

---

## §2. Seed ↔ 외부 레지스트리 링크 테이블

```sql
CREATE TABLE festimate_core.clinic_external_links (
    link_id                       BIGSERIAL PRIMARY KEY,
    seed_id                       BIGINT NOT NULL
                                      REFERENCES festimate_core.clinic_longlist_seed(seed_id),
    ext_raw_id                    BIGINT NOT NULL
                                      REFERENCES festimate_ext.external_provider_raw(ext_raw_id),
    source_system                 TEXT NOT NULL,

    -- 매칭 결과
    match_score                   NUMERIC(5,2) NOT NULL,
    match_rule                    TEXT NOT NULL,
    -- 'exact_phone'              : 전화번호 정확 일치
    -- 'exact_name_district'      : 기관명 정확 + 시군구 일치
    -- 'fuzzy_name_address'       : 기관명+주소 퍼지 유사도
    -- 'domain_match'             : 홈페이지 도메인 일치
    -- 'manual'                   : 관리자 수동 확정

    is_confirmed                  BOOLEAN NOT NULL DEFAULT FALSE,
    needs_manual_review           BOOLEAN NOT NULL DEFAULT FALSE,

    linked_at                     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    linked_by                     TEXT NOT NULL DEFAULT 'system',

    UNIQUE (seed_id, ext_raw_id)
);

CREATE INDEX idx_ext_links_seed
    ON festimate_core.clinic_external_links (seed_id);
CREATE INDEX idx_ext_links_ext
    ON festimate_core.clinic_external_links (ext_raw_id);
CREATE INDEX idx_ext_links_confirmed
    ON festimate_core.clinic_external_links (is_confirmed, source_system);
```

### 매칭 우선순위

```
1순위: 전화번호 exact match        → match_score = 100, is_confirmed = TRUE
2순위: 기관명 exact + district 일치 → match_score = 85,  is_confirmed = TRUE
3순위: 기관명+주소 trigram 유사도   → match_score = 0~84 (Python rapidfuzz)
       score >= 85 → is_confirmed = TRUE
       score 65~84 → needs_manual_review = TRUE
4순위: 홈페이지 domain 일치         → match_score = 75,  is_confirmed = FALSE (검토 필요)
5순위: 미매칭 → INSERT 안 함
```

### 매칭 SQL (pg_trgm 사전 설치 필요)

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 전화번호 exact match
INSERT INTO festimate_core.clinic_external_links
    (seed_id, ext_raw_id, source_system, match_score, match_rule, is_confirmed)
SELECT
    s.seed_id,
    e.ext_raw_id,
    e.source_system,
    100.0,
    'exact_phone',
    TRUE
FROM festimate_core.clinic_longlist_seed s
JOIN festimate_ext.external_provider_raw e
  ON s.phone_norm IS NOT NULL
 AND s.phone_norm = e.phone_norm
WHERE s.canonical_status IN ('normalized', '2ndlist', 'loaded')
ON CONFLICT (seed_id, ext_raw_id) DO NOTHING;

-- 기관명 exact + district 일치
INSERT INTO festimate_core.clinic_external_links
    (seed_id, ext_raw_id, source_system, match_score, match_rule, is_confirmed)
SELECT
    s.seed_id,
    e.ext_raw_id,
    e.source_system,
    85.0,
    'exact_name_district',
    TRUE
FROM festimate_core.clinic_longlist_seed s
JOIN festimate_ext.external_provider_raw e
  ON s.institution_name_norm = e.provider_name_norm
 AND s.district IS NOT NULL
 AND s.district = e.district
WHERE NOT EXISTS (
    SELECT 1 FROM festimate_core.clinic_external_links l
    WHERE l.seed_id = s.seed_id AND l.ext_raw_id = e.ext_raw_id
)
ON CONFLICT (seed_id, ext_raw_id) DO NOTHING;

-- trigram 유사도 매칭 (Python에서 rapidfuzz로 처리 후 결과 INSERT)
-- Python 처리 후 아래 형태로 INSERT:
-- INSERT INTO festimate_core.clinic_external_links (seed_id, ext_raw_id, source_system, match_score, match_rule, is_confirmed, needs_manual_review)
-- VALUES (..., score, 'fuzzy_name_address', score>=85, score BETWEEN 65 AND 84)
-- ON CONFLICT (seed_id, ext_raw_id) DO NOTHING;
```

---

## §3. 홈페이지 테이블

```sql
CREATE TABLE festimate_core.clinic_homepages (
    homepage_id                   BIGSERIAL PRIMARY KEY,
    seed_id                       BIGINT NOT NULL
                                      REFERENCES festimate_core.clinic_longlist_seed(seed_id),

    homepage_url                  TEXT NOT NULL,
    homepage_domain               TEXT,
    source_system                 TEXT NOT NULL,
    -- 'direct_discovery'         : 직접 검색 (Google CSE / Naver Place)
    -- 'medical_korea'            : Medical Korea provider 페이지에서 확인
    -- 'external_link'            : external_provider_raw.website_url 에서 반영
    -- 'manual'                   : 관리자 수동 입력

    source_url                    TEXT,
    is_official                   BOOLEAN NOT NULL DEFAULT FALSE,
    verification_status           TEXT NOT NULL DEFAULT 'discovered',
    -- 'discovered'               : URL 발견, 미검증
    -- 'verified'                 : 기관명 타이틀 일치 + 서비스 확인
    -- 'rejected'                 : 검증 실패 (기관명 불일치 등)
    -- 'review'                   : 애매함, 수동 검토 필요

    match_confidence              NUMERIC(5,2),
    title_text                    TEXT,
    discovered_at                 TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    verified_at                   TIMESTAMPTZ,

    UNIQUE (seed_id, homepage_url)
);

CREATE INDEX idx_homepages_seed
    ON festimate_core.clinic_homepages (seed_id);
CREATE INDEX idx_homepages_domain
    ON festimate_core.clinic_homepages (homepage_domain);
CREATE INDEX idx_homepages_verified
    ON festimate_core.clinic_homepages (seed_id, verified_at DESC)
    WHERE verification_status = 'verified';
```

### external_provider_raw에서 홈페이지 반영

```sql
INSERT INTO festimate_core.clinic_homepages (
    seed_id, homepage_url, homepage_domain, source_system, source_url,
    is_official, verification_status, match_confidence, discovered_at, verified_at
)
SELECT
    l.seed_id,
    e.website_url,
    regexp_replace(lower(e.website_url), '^https?://(www\.)?', ''),
    e.source_system,
    e.source_url,
    TRUE,
    'verified',
    l.match_score,
    NOW(),
    NOW()
FROM festimate_core.clinic_external_links l
JOIN festimate_ext.external_provider_raw e ON l.ext_raw_id = e.ext_raw_id
WHERE l.is_confirmed = TRUE
  AND e.website_url IS NOT NULL
ON CONFLICT (seed_id, homepage_url) DO NOTHING;
```

---

## §4. 인증 테이블

KHIDI (3년 유효) 와 KAHF (4년 유효) 를 분리 행으로 저장한다.

```sql
CREATE TABLE festimate_core.clinic_accreditations (
    accreditation_id              BIGSERIAL PRIMARY KEY,
    seed_id                       BIGINT NOT NULL
                                      REFERENCES festimate_core.clinic_longlist_seed(seed_id),

    accreditation_type            TEXT NOT NULL,
    -- 'KHIDI_REGISTERED'         : 외국인환자 유치기관 등록 (3년 유효)
    -- 'KAHF'                     : 외국인환자 안전 인증 (4년 유효)

    accreditation_status          TEXT NOT NULL,
    -- 'active'                   : 유효
    -- 'expired'                  : 만료
    -- 'pending'                  : 심사 중
    -- 'revoked'                  : 취소

    valid_from                    DATE,
    valid_until                   DATE,
    is_currently_valid            BOOLEAN GENERATED ALWAYS AS (
                                      valid_until IS NULL OR valid_until >= CURRENT_DATE
                                  ) STORED,

    source_system                 TEXT NOT NULL,
    source_url                    TEXT,
    ext_raw_id                    BIGINT
                                      REFERENCES festimate_ext.external_provider_raw(ext_raw_id),

    created_at                    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE (seed_id, accreditation_type, source_system,
            COALESCE(valid_from, DATE '1900-01-01'))
);

CREATE INDEX idx_accreditations_seed_type
    ON festimate_core.clinic_accreditations (seed_id, accreditation_type);
CREATE INDEX idx_accreditations_active
    ON festimate_core.clinic_accreditations (accreditation_status, valid_until)
    WHERE accreditation_status = 'active';
```

### KHIDI 인증 반영 SQL

```sql
INSERT INTO festimate_core.clinic_accreditations (
    seed_id, accreditation_type, accreditation_status,
    valid_from, valid_until, source_system, source_url, ext_raw_id
)
SELECT
    l.seed_id,
    'KHIDI_REGISTERED',
    CASE
        WHEN e.registration_valid_until IS NULL            THEN 'active'
        WHEN e.registration_valid_until >= CURRENT_DATE    THEN 'active'
        ELSE 'expired'
    END,
    e.registration_valid_from,
    e.registration_valid_until,
    e.source_system,
    e.source_url,
    e.ext_raw_id
FROM festimate_core.clinic_external_links l
JOIN festimate_ext.external_provider_raw e ON l.ext_raw_id = e.ext_raw_id
WHERE l.is_confirmed = TRUE
  AND e.source_system = 'KHIDI_REGISTERED'
ON CONFLICT DO NOTHING;
```

### KAHF 인증 반영 SQL

```sql
INSERT INTO festimate_core.clinic_accreditations (
    seed_id, accreditation_type, accreditation_status,
    valid_from, valid_until, source_system, source_url, ext_raw_id
)
SELECT
    l.seed_id,
    'KAHF',
    CASE
        WHEN e.accreditation_valid_until IS NULL           THEN 'active'
        WHEN e.accreditation_valid_until >= CURRENT_DATE   THEN 'active'
        ELSE 'expired'
    END,
    e.accreditation_valid_from,
    e.accreditation_valid_until,
    e.source_system,
    e.source_url,
    e.ext_raw_id
FROM festimate_core.clinic_external_links l
JOIN festimate_ext.external_provider_raw e ON l.ext_raw_id = e.ext_raw_id
WHERE l.is_confirmed = TRUE
  AND e.accreditation_type = 'KAHF'
ON CONFLICT DO NOTHING;
```

---

## §5. 외국어 지원 테이블

```sql
CREATE TABLE festimate_core.clinic_language_support (
    language_id                   BIGSERIAL PRIMARY KEY,
    seed_id                       BIGINT NOT NULL
                                      REFERENCES festimate_core.clinic_longlist_seed(seed_id),

    language_code                 TEXT NOT NULL,   -- en / zh / ja / ru / vi / mn / th
    support_type                  TEXT NOT NULL,
    -- 'interpretation'           : 통역 서비스
    -- 'website'                  : 다국어 웹사이트
    -- 'coordinator'              : 전담 코디네이터
    -- 'sns'                      : 다국어 SNS

    source_system                 TEXT NOT NULL,
    source_url                    TEXT,
    ext_raw_id                    BIGINT
                                      REFERENCES festimate_ext.external_provider_raw(ext_raw_id),
    is_active                     BOOLEAN NOT NULL DEFAULT TRUE,
    confidence_score              NUMERIC(5,2),

    created_at                    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (seed_id, language_code, support_type, source_system)
);

CREATE INDEX idx_language_seed    ON festimate_core.clinic_language_support (seed_id);
CREATE INDEX idx_language_code    ON festimate_core.clinic_language_support (language_code, support_type);
```

### interpretation_raw 파싱 + 적재 SQL

```sql
WITH lang_src AS (
    SELECT
        l.seed_id,
        e.ext_raw_id,
        e.source_system,
        e.source_url,
        trim(value) AS lang_raw
    FROM festimate_core.clinic_external_links l
    JOIN festimate_ext.external_provider_raw e ON l.ext_raw_id = e.ext_raw_id
    CROSS JOIN LATERAL regexp_split_to_table(
        coalesce(e.interpretation_raw, ''), ','
    ) AS value
    WHERE l.is_confirmed = TRUE
      AND e.interpretation_raw IS NOT NULL
      AND trim(value) <> ''
)
INSERT INTO festimate_core.clinic_language_support (
    seed_id, language_code, support_type,
    source_system, source_url, ext_raw_id, is_active, confidence_score
)
SELECT
    seed_id,
    CASE
        WHEN upper(lang_raw) LIKE '%ENGLISH%'    THEN 'en'
        WHEN upper(lang_raw) LIKE '%CHINESE%'
          OR upper(lang_raw) LIKE '%MANDARIN%'   THEN 'zh'
        WHEN upper(lang_raw) LIKE '%JAPANESE%'   THEN 'ja'
        WHEN upper(lang_raw) LIKE '%RUSSIAN%'    THEN 'ru'
        WHEN upper(lang_raw) LIKE '%VIETNAMESE%' THEN 'vi'
        WHEN upper(lang_raw) LIKE '%MONGOLIAN%'  THEN 'mn'
        WHEN upper(lang_raw) LIKE '%THAI%'       THEN 'th'
        ELSE lower(trim(lang_raw))
    END AS language_code,
    'interpretation',
    source_system,
    source_url,
    ext_raw_id,
    TRUE,
    90.0
FROM lang_src
ON CONFLICT DO NOTHING;
```

---

## §6. structured_specialty_flag 업데이트

KHIDI / Medical Korea 보강 완료 후 `clinic_filter_scores`에 반영한다.

```sql
-- K-Beauty 관련 specialty가 외부 소스에서 확인된 경우 flag = TRUE
UPDATE festimate_core.clinic_filter_scores fs
SET
    structured_specialty_flag   = TRUE,
    structured_specialty_source = e.source_system,
    structured_specialty_value  = e.specialty_norm,
    updated_at                  = NOW()
FROM festimate_core.clinic_external_links l
JOIN festimate_ext.external_provider_raw e ON l.ext_raw_id = e.ext_raw_id
WHERE l.seed_id = fs.seed_id
  AND l.is_confirmed = TRUE
  AND e.specialty_norm IN (
      'dermatology',
      'plastic_surgery',
      'medical_beauty'
  )
  AND (fs.structured_specialty_flag IS NULL
       OR fs.structured_specialty_flag = FALSE);

-- 보강 완료됐지만 K-Beauty specialty 아닌 경우 → flag = FALSE (미확인 아님, 명시적 없음)
UPDATE festimate_core.clinic_filter_scores fs
SET
    structured_specialty_flag   = FALSE,
    updated_at                  = NOW()
WHERE structured_specialty_flag IS NULL
  AND EXISTS (
      SELECT 1
      FROM festimate_core.clinic_external_links l
      WHERE l.seed_id = fs.seed_id
        AND l.is_confirmed = TRUE
  );
```

---

## §7. Intl Bonus 점수 반영

보강 완료 후 `clinic_filter_scores.intl_score_raw` 업데이트.

```sql
UPDATE festimate_core.clinic_filter_scores fs
SET
    intl_score_raw = (
        -- IB-01: KHIDI 등록
        CASE WHEN EXISTS (
            SELECT 1 FROM festimate_core.clinic_accreditations a
            WHERE a.seed_id = fs.seed_id
              AND a.accreditation_type = 'KHIDI_REGISTERED'
              AND a.accreditation_status = 'active'
        ) THEN 10 ELSE 0 END

        -- IB-02: KAHF 인증
        + CASE WHEN EXISTS (
            SELECT 1 FROM festimate_core.clinic_accreditations a
            WHERE a.seed_id = fs.seed_id
              AND a.accreditation_type = 'KAHF'
              AND a.accreditation_status = 'active'
        ) THEN 12 ELSE 0 END

        -- IB-03/IB-04: 외국어 지원 (IB-04가 IB-03 대체)
        + CASE
            WHEN (
                SELECT COUNT(DISTINCT language_code)
                FROM festimate_core.clinic_language_support l
                WHERE l.seed_id = fs.seed_id AND l.language_code IN ('en','zh','ja')
            ) >= 3 THEN 10   -- IB-04
            WHEN (
                SELECT COUNT(DISTINCT language_code)
                FROM festimate_core.clinic_language_support l
                WHERE l.seed_id = fs.seed_id AND l.language_code IN ('en','zh','ja')
            ) >= 1 THEN 6    -- IB-03
            ELSE 0
        END
    ),
    intl_score_capped = LEAST(GREATEST(
        (
            CASE WHEN EXISTS (
                SELECT 1 FROM festimate_core.clinic_accreditations a
                WHERE a.seed_id = fs.seed_id
                  AND a.accreditation_type = 'KHIDI_REGISTERED'
                  AND a.accreditation_status = 'active'
            ) THEN 10 ELSE 0 END
            + CASE WHEN EXISTS (
                SELECT 1 FROM festimate_core.clinic_accreditations a
                WHERE a.seed_id = fs.seed_id
                  AND a.accreditation_type = 'KAHF'
                  AND a.accreditation_status = 'active'
            ) THEN 12 ELSE 0 END
            + CASE
                WHEN (SELECT COUNT(DISTINCT language_code) FROM festimate_core.clinic_language_support l
                      WHERE l.seed_id = fs.seed_id AND l.language_code IN ('en','zh','ja')) >= 3 THEN 10
                WHEN (SELECT COUNT(DISTINCT language_code) FROM festimate_core.clinic_language_support l
                      WHERE l.seed_id = fs.seed_id AND l.language_code IN ('en','zh','ja')) >= 1 THEN 6
                ELSE 0
            END
        ), 0), 20
    ),
    intl_scored_at = NOW(),
    updated_at     = NOW()
WHERE fs.stage2_status = 'provisional';
```

---

## §8. 보강 파이프라인 실행 순서

```
Step E-0: etl_batches에 소스별 배치 등록
Step E-1: external_provider_raw 적재 (KHIDI CSV)
Step E-2: external_provider_raw 적재 (Medical Korea 스크래핑 — KAHF / Provider)
Step E-3: provider_name_norm, specialty_norm 정규화
Step E-4: 전화번호 exact 매칭 → clinic_external_links
Step E-5: 기관명+district exact 매칭 → clinic_external_links
Step E-6: trigram 퍼지 매칭 (Python rapidfuzz) → clinic_external_links
Step E-7: domain 매칭 → clinic_external_links
Step E-8: clinic_accreditations 반영 (KHIDI + KAHF)
Step E-9: clinic_homepages 반영 (website_url에서)
Step E-10: clinic_language_support 반영 (interpretation_raw 파싱)
Step E-11: structured_specialty_flag 업데이트 → clinic_filter_scores
Step E-12: intl_score 계산 → clinic_filter_scores
```

잡 오케스트레이션 및 재시도 정책은 `PIPELINE_AUTOMATION_SPEC.md` 참조.

---

## §9. 읽기/쓰기 의존 테이블 요약

| 단계 | 읽기 | 쓰기 |
|------|------|------|
| raw 적재 | etl_batches | external_provider_raw |
| 매칭 | clinic_longlist_seed, external_provider_raw, clinic_homepages | clinic_external_links |
| 인증 반영 | clinic_external_links, external_provider_raw | clinic_accreditations |
| 홈페이지 반영 | clinic_external_links, external_provider_raw | clinic_homepages |
| 언어 반영 | clinic_external_links, external_provider_raw | clinic_language_support |
| specialty 반영 | clinic_external_links, external_provider_raw | clinic_filter_scores |
| intl 점수 | clinic_accreditations, clinic_language_support | clinic_filter_scores |

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-04-22 | 초안 — KHIDI/KAHF 별도 테이블, 홈페이지/언어 분리 |
| v2.0 | 2026-04-22 | 전면 재작성 — external_provider_raw 단일 테이블(source_system 구분), clinic_accreditations KHIDI 3년/KAHF 4년 분리, structured_specialty_flag 컬럼 추가, Intl 점수 SQL 추가 |

*담당: Claude Desktop (기획)*
