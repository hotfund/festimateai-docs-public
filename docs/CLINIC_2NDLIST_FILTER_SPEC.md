# CLINIC_2NDLIST_FILTER_SPEC.md
# 클리닉 2차 롱리스트 필터 규칙 + 점수 기준

> **버전**: v2.0
> **작성**: 2026-04-22
> **상태**: 기획 확정
> **환경**: Claude Desktop (기획 전담)
> **변경**: v1.0 전면 재작성 — 100점 cap 체계, specialty_gate, Stage 2A/2B 분리

---

## §0. 목적 및 입출력

### 입력

```
festimate_core.clinic_longlist_seed
  WHERE canonical_status = 'normalized'
    AND is_active_candidate = TRUE
    AND is_target_region = TRUE
```

### 처리 흐름

```
Hard Gate 통과 여부 판정
  → Pre-score 계산 (max 20pt)
  → Stage 2A Provisional 판정 (pre_raw >= 18)
  → Web Evidence Score 계산 (max 60pt)  ← 크롤링 후
  → Intl/Trust Bonus 계산 (max 20pt)    ← 보강 배치 후
  → specialty_gate / homepage_gate 판정
  → Stage 2B Verified 판정 (final >= 75 + 두 gate)
```

### 출력

```
festimate_core.clinic_filter_scores (stage2_status 업데이트)

  stage2_status = 'provisional'  → Stage 2A: 크롤링 큐 투입 대상
  stage2_status = 'verified'     → Stage 2B: 서비스 노출 후보
  stage2_status = 'review'       → 수동 검토 큐
  stage2_status = 'excluded'     → 제외
```

### 실행 엔진 참조

잡 오케스트레이션, 배치 순서, 재시도 정책은 `PIPELINE_AUTOMATION_SPEC.md` 참조.

---

## §1. 점수 체계 — 100점 Cap

```
final_score = pre_score_capped + web_score_capped + intl_score_capped

pre_score_capped   = LEAST(GREATEST(pre_score_raw,  0), 20)
web_score_capped   = LEAST(GREATEST(web_score_raw,  0), 60)
intl_score_capped  = LEAST(GREATEST(intl_score_raw, 0), 20)

최대 합계: 20 + 60 + 20 = 100점
```

점수와 별도로 **specialty_gate**, **homepage_gate** 두 개를 동시에 통과해야 Stage 2B 진입이 가능합니다.

---

## §2. Hard Gate (점수 계산 전 즉시 제외)

아래 조건 중 하나라도 해당하면 `stage2_status = 'excluded'`. 점수 계산 없음.

| Rule ID | 조건 | 제외 사유 |
|---------|------|----------|
| HG-01 | `is_active_candidate = FALSE` | 폐업/휴업 |
| HG-02 | `is_target_region = FALSE` | 서비스 대상 지역 외 |
| HG-03 | `institution_type_norm = 'public_health'` | 보건소 계열 |
| HG-04 | 기관명 AND 주소 둘 다 NULL | 데이터 불충분 |
| HG-05 | `institution_type_norm = 'dental_clinic'` | 치과 |
| HG-06 | `institution_type_norm = 'oriental_clinic'` | 한의원 (정책 확장 시 review 가능) |

---

## §3. Pre-score (max 20pt)

A 데이터(1차 롱리스트 seed)만으로 계산. 크롤링 불필요.

### 3-1. 양수 규칙

| Rule ID | 대상 필드 | 조건 | Raw 점수 |
|---------|----------|------|---------|
| SP-01 | `institution_name_norm` | `성형` 또는 `plastic` 포함 | +30 |
| SP-02 | `institution_name_norm` | `피부` 또는 `derm` 또는 `skin` 포함 | +28 |
| SP-03 | `institution_name_norm` | `에스테틱`, `aesthetic`, `beauty`, `anti-aging`, `antiaging`, `안티에이징`, `리프팅` 포함 | +18 |
| SP-04 | `institution_name_norm` | `메디컬스킨`, `쁘띠`, `레이저`, `비만` 포함 | +12 |
| SP-05 | `institution_type_norm` | `clinic` | +8 |
| SP-06 | `phone_norm` | NOT NULL | +4 |
| SP-07 | `road_address_raw` | NOT NULL | +4 |
| SP-08 | `district` | 핵심 상권 (강남구·서초구·송파구·해운대구·수영구·수성구·서구) | +3 |

> Raw 점수가 높아도 cap이 20이므로, SP-01 하나만 매칭돼도 cap 적용. 의도된 설계.

### 3-2. 음수 규칙

| Rule ID | 대상 필드 | 조건 | Raw 점수 |
|---------|----------|------|---------|
| SN-01 | `institution_name_norm` | `내과`, `정형`, `재활`, `신경`, `정신`, `검진`, `소아`, `안과`, `비뇨`, `산부인과` 포함 | -15 |
| SN-02 | `institution_name_norm` | `요양`, `치매` 포함 | -25 |
| SN-03 | `institution_name_norm` | `한의원`, `치과` 포함 | -30 |

### 3-3. Pre-score 계산 SQL

```sql
INSERT INTO festimate_core.clinic_filter_scores (
    seed_id, pre_score_raw, pre_score_capped, stage2_status, pre_scored_at, created_at, updated_at
)
SELECT
    s.seed_id,
    (
      -- 양수: 그룹 A (성형·피부·미용계열, 최고 1개만 적용)
      CASE
          WHEN s.institution_name_norm ~ '(성형|plastic)'                                        THEN 30
          WHEN s.institution_name_norm ~ '(피부|derm|skin)'                                      THEN 28
          WHEN s.institution_name_norm ~ '(에스테틱|aesthetic|beauty|anti.?aging|안티에이징|리프팅)' THEN 18
          WHEN s.institution_name_norm ~ '(메디컬스킨|쁘띠|레이저|비만)'                          THEN 12
          ELSE 0
      END
      + CASE WHEN s.institution_type_norm = 'clinic'      THEN 8 ELSE 0 END
      + CASE WHEN s.phone_norm IS NOT NULL                THEN 4 ELSE 0 END
      + CASE WHEN s.road_address_raw IS NOT NULL          THEN 4 ELSE 0 END
      + CASE WHEN s.district IN (
            '강남구','서초구','송파구',
            '해운대구','수영구',
            '수성구',
            '서구'
        )                                                 THEN 3 ELSE 0 END
      -- 음수
      - CASE WHEN s.institution_name_norm ~ '(내과|정형|재활|신경|정신|검진|소아|안과|비뇨|산부인과)' THEN 15 ELSE 0 END
      - CASE WHEN s.institution_name_norm ~ '(요양|치매)'                                        THEN 25 ELSE 0 END
      - CASE WHEN s.institution_name_norm ~ '(한의원|치과)'                                      THEN 30 ELSE 0 END
    ) AS pre_score_raw,

    LEAST(GREATEST(
      (
        CASE
            WHEN s.institution_name_norm ~ '(성형|plastic)'                                        THEN 30
            WHEN s.institution_name_norm ~ '(피부|derm|skin)'                                      THEN 28
            WHEN s.institution_name_norm ~ '(에스테틱|aesthetic|beauty|anti.?aging|안티에이징|리프팅)' THEN 18
            WHEN s.institution_name_norm ~ '(메디컬스킨|쁘띠|레이저|비만)'                          THEN 12
            ELSE 0
        END
        + CASE WHEN s.institution_type_norm = 'clinic'      THEN 8 ELSE 0 END
        + CASE WHEN s.phone_norm IS NOT NULL                THEN 4 ELSE 0 END
        + CASE WHEN s.road_address_raw IS NOT NULL          THEN 4 ELSE 0 END
        + CASE WHEN s.district IN ('강남구','서초구','송파구','해운대구','수영구','수성구','서구') THEN 3 ELSE 0 END
        - CASE WHEN s.institution_name_norm ~ '(내과|정형|재활|신경|정신|검진|소아|안과|비뇨|산부인과)' THEN 15 ELSE 0 END
        - CASE WHEN s.institution_name_norm ~ '(요양|치매)'                                        THEN 25 ELSE 0 END
        - CASE WHEN s.institution_name_norm ~ '(한의원|치과)'                                      THEN 30 ELSE 0 END
      ), 0), 20
    ) AS pre_score_capped,

    'not_scored' AS stage2_status,
    NOW() AS pre_scored_at,
    NOW() AS created_at,
    NOW() AS updated_at

FROM festimate_core.clinic_longlist_seed s
WHERE s.canonical_status = 'normalized'
  AND s.is_active_candidate = TRUE
  AND s.is_target_region = TRUE

ON CONFLICT (seed_id) DO UPDATE SET
    pre_score_raw    = EXCLUDED.pre_score_raw,
    pre_score_capped = EXCLUDED.pre_score_capped,
    pre_scored_at    = EXCLUDED.pre_scored_at,
    scoring_version  = 'v2_mvp',
    updated_at       = NOW();
```

### 3-4. Stage 2A Provisional 판정 SQL

```sql
-- Provisional 진입: pre_score_raw >= 18
UPDATE festimate_core.clinic_filter_scores fs
SET
    stage2_status        = 'provisional',
    stage2_status_reason = 'pre_score_threshold',
    updated_at           = NOW()
WHERE fs.pre_score_raw >= 18
  AND fs.stage2_status = 'not_scored';

-- pre_score_raw < 18이라도 강한 이름 신호가 있으면 provisional
-- (SP-01 또는 SP-02 매칭 시 raw는 28+ 이므로 이미 위에서 처리됨)
-- 하단은 예외적으로 negative가 많아 raw < 18이 된 케이스 보조 처리
UPDATE festimate_core.clinic_filter_scores fs
SET
    stage2_status        = 'provisional',
    stage2_status_reason = 'strong_name_signal_override',
    updated_at           = NOW()
FROM festimate_core.clinic_longlist_seed s
WHERE fs.seed_id = s.seed_id
  AND fs.pre_score_raw < 18
  AND fs.stage2_status = 'not_scored'
  AND s.institution_name_norm ~ '(성형|피부|plastic|derm|skin|aesthetic)';

-- 그 외 → 제외
UPDATE festimate_core.clinic_filter_scores
SET
    stage2_status        = 'excluded',
    stage2_status_reason = 'low_pre_score',
    updated_at           = NOW()
WHERE stage2_status = 'not_scored';
```

---

## §4. Web Evidence Score (max 60pt)

Provisional 상태의 클리닉 홈페이지를 크롤링한 후 계산. `PIPELINE_AUTOMATION_SPEC.md` Step B 참조.

### 4-1. 양수 규칙

| Rule ID | 증거 | 조건 | Raw 점수 | Specialty 해당 |
|---------|------|------|---------|---------------|
| WE-01 | 공식 홈페이지 | 검증된 홈페이지 존재 | +15 | |
| WE-02 | 타이틀 일치 | 기관명이 타이틀/메타에 포함 | +10 | |
| WE-03 | 서비스 메뉴 | 피부·성형·미용·항노화 관련 메뉴 확인 | +20 | **✓** |
| WE-04 | 시술 페이지 | 시술 상세 페이지 3개 이상 | +15 | **✓** |
| WE-05 | 시술 키워드 | 보톡스·필러·리쥬란·울쎄라·써마지·레이저·실리프팅·눈성형·코성형 등 | +20 | **✓** |
| WE-06 | 의료진 페이지 | 원장/의료진 소개 페이지 존재 | +10 | |
| WE-07 | 가격/이벤트 | 가격표·이벤트·프로모션 페이지 존재 | +8 | |
| WE-08 | 예약/상담 채널 | 카톡·폼·전화·WhatsApp 등 예약 흐름 | +5 | |
| WE-09 | Before/After | Before/After 또는 후기 구조 존재 | +4 | |
| WE-10 | 다국어 메뉴 | 영문·중문·일문 페이지 존재 | +8 | |

> **Specialty 해당 컬럼 ✓**: WE-03 + WE-04 + WE-05 합산이 `web_score_specialty_raw`.

### 4-2. 음수 규칙

| Rule ID | 조건 | Raw 점수 |
|---------|------|---------|
| WN-01 | 공식 홈페이지 발견 실패 | 0 (감점 없음, 단 homepage_gate = FALSE) |
| WN-02 | 홈페이지 있으나 기관명 불일치 | -20 |
| WN-03 | 미용 관련 증거 전무 (홈페이지 있음) | -25 |

### 4-3. specialty_gate 판정

```
web_score_specialty_raw = WE-03 score + WE-04 score + WE-05 score

specialty_gate = TRUE 조건 (하나 이상):
  web_score_specialty_raw >= 20
  OR COALESCE(structured_specialty_flag, FALSE) = TRUE

specialty_gate = FALSE 조건:
  web_score_specialty_raw < 20
  AND COALESCE(structured_specialty_flag, FALSE) = FALSE
```

> `structured_specialty_flag`는 MVP 1차 배치에서 NULL. 판정 시 `COALESCE(_, FALSE)`로 처리. 추후 Medical Korea / KHIDI enrichment 배치에서 업데이트.

### 4-4. homepage_gate 판정

```
homepage_gate = TRUE 조건 (하나 이상):
  festimate_core.clinic_homepages에서 verification_status = 'verified' 존재
  OR 공식 provider profile page 확인 (Medical Korea 상세 페이지 등)
```

---

## §5. Intl/Trust Bonus (max 20pt)

KHIDI/KAHF/언어 지원 보강 배치 완료 후 계산. `CLINIC_ENRICHMENT_SPEC.md` 참조.

| Rule ID | 증거 | 조건 | Raw 점수 |
|---------|------|------|---------|
| IB-01 | KHIDI 등록 | 유효한 외국인환자 유치기관 등록 | +10 |
| IB-02 | KAHF 인증 | 유효한 KAHF 인증 (4년 유효) | +12 |
| IB-03 | 외국어 지원 | 영·중·일 중 1개 이상 | +6 |
| IB-04 | 외국어 3개 | 영·중·일 모두 지원 (IB-03 대체, 중복 적용 안 함) | +10 |
| IB-05 | 국제환자 연락처 | 국제센터 이메일·전화·메신저 전담 | +5 |
| IB-06 | 국제환자 전용 서비스 | 공항픽업·통역·컨시어지 기재 | +4 |

> IB-04는 IB-03을 대체. 3개 언어 지원 시 +10만 적용, +6+10 스택 아님.

---

## §6. 최종 판정 기준

### Stage 2B Verified (자동 편입)

```sql
final_score >= 75
AND specialty_gate = TRUE
AND homepage_gate = TRUE
```

### Review Queue

```sql
final_score BETWEEN 55 AND 74
OR (intl_score_capped >= 15 AND specialty_gate = FALSE)   -- 국제 신뢰는 강하나 specialty 미확인
OR (specialty_gate = TRUE AND homepage_gate = FALSE)       -- specialty 있으나 홈페이지 미검증
```

### Exclude

```sql
final_score < 55
OR hard_gate_fail = TRUE
```

---

## §7. clinic_filter_scores DDL

```sql
CREATE TABLE festimate_core.clinic_filter_scores (
    seed_id                     BIGINT PRIMARY KEY
                                    REFERENCES festimate_core.clinic_longlist_seed(seed_id),

    -- Pre-score
    pre_score_raw               INTEGER NOT NULL DEFAULT 0,
    pre_score_capped            INTEGER NOT NULL DEFAULT 0,   -- LEAST(GREATEST(raw,0), 20)

    -- Web Evidence Score
    web_score_raw               INTEGER NOT NULL DEFAULT 0,
    web_score_capped            INTEGER NOT NULL DEFAULT 0,   -- LEAST(GREATEST(raw,0), 60)
    web_score_specialty_raw     INTEGER NOT NULL DEFAULT 0,   -- WE-03+WE-04+WE-05 합산

    -- Intl/Trust Bonus
    intl_score_raw              INTEGER NOT NULL DEFAULT 0,
    intl_score_capped           INTEGER NOT NULL DEFAULT 0,   -- LEAST(GREATEST(raw,0), 20)

    -- Final
    final_score                 INTEGER NOT NULL DEFAULT 0,   -- 세 capped 합산

    -- Gates
    specialty_gate              BOOLEAN NOT NULL DEFAULT FALSE,
    homepage_gate               BOOLEAN NOT NULL DEFAULT FALSE,

    -- Structured Specialty (enrichment 배치 후 업데이트)
    structured_specialty_flag   BOOLEAN,                      -- NULL=미확인 / TRUE=확인 / FALSE=없음
    structured_specialty_source TEXT,                         -- medical_korea_specialty / khidi_field / manual_verified
    structured_specialty_value  TEXT,                         -- 'DERMATOLOGY' / 'PLASTIC_SURGERY' / 'MEDICAL_BEAUTY'

    -- Stage
    stage2_status               TEXT NOT NULL DEFAULT 'not_scored',
                                                              -- not_scored / provisional / review / verified / excluded
    stage2_status_reason        TEXT,

    -- Metadata
    scoring_version             TEXT NOT NULL DEFAULT 'v2_mvp',
    pre_scored_at               TIMESTAMPTZ,
    web_scored_at               TIMESTAMPTZ,
    intl_scored_at              TIMESTAMPTZ,
    final_decided_at            TIMESTAMPTZ,
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_filter_scores_stage2_status
    ON festimate_core.clinic_filter_scores (stage2_status);
CREATE INDEX idx_filter_scores_final_score
    ON festimate_core.clinic_filter_scores (final_score DESC);
CREATE INDEX idx_filter_scores_gates
    ON festimate_core.clinic_filter_scores (specialty_gate, homepage_gate);
CREATE INDEX idx_filter_scores_specialty_flag
    ON festimate_core.clinic_filter_scores (structured_specialty_flag);
```

---

## §8. Stage 2B 최종 판정 SQL

Web + Intl 점수가 모두 들어온 뒤 실행.

```sql
-- final_score 계산
UPDATE festimate_core.clinic_filter_scores
SET
    final_score      = pre_score_capped + web_score_capped + intl_score_capped,
    specialty_gate   = (
        web_score_specialty_raw >= 20
        OR COALESCE(structured_specialty_flag, FALSE) = TRUE
    ),
    homepage_gate    = EXISTS (
        SELECT 1
        FROM festimate_core.clinic_homepages h
        WHERE h.seed_id = clinic_filter_scores.seed_id
          AND h.verification_status = 'verified'
    ),
    updated_at       = NOW()
WHERE stage2_status = 'provisional';

-- Verified 판정
UPDATE festimate_core.clinic_filter_scores
SET
    stage2_status        = 'verified',
    stage2_status_reason = 'auto_include',
    final_decided_at     = NOW(),
    updated_at           = NOW()
WHERE stage2_status   = 'provisional'
  AND final_score     >= 75
  AND specialty_gate  = TRUE
  AND homepage_gate   = TRUE;

-- Review Queue 판정
UPDATE festimate_core.clinic_filter_scores
SET
    stage2_status        = 'review',
    stage2_status_reason = CASE
        WHEN final_score BETWEEN 55 AND 74                        THEN 'score_borderline'
        WHEN intl_score_capped >= 15 AND specialty_gate = FALSE   THEN 'strong_intl_weak_specialty'
        WHEN specialty_gate = TRUE AND homepage_gate = FALSE      THEN 'specialty_ok_no_homepage'
        ELSE 'manual_review'
    END,
    final_decided_at     = NOW(),
    updated_at           = NOW()
WHERE stage2_status = 'provisional'
  AND (
      final_score BETWEEN 55 AND 74
      OR (intl_score_capped >= 15 AND specialty_gate = FALSE)
      OR (specialty_gate = TRUE AND homepage_gate = FALSE)
  );

-- Exclude
UPDATE festimate_core.clinic_filter_scores
SET
    stage2_status        = 'excluded',
    stage2_status_reason = 'low_final_score',
    final_decided_at     = NOW(),
    updated_at           = NOW()
WHERE stage2_status = 'provisional';

-- classification_log 기록
INSERT INTO festimate_core.clinic_seed_classification_log (
    seed_id, classification_stage, decision, decision_reason, evidence_payload, decided_by
)
SELECT
    seed_id,
    'stage2_final',
    CASE WHEN stage2_status = 'verified' THEN 'include' ELSE stage2_status END,
    stage2_status_reason,
    jsonb_build_object(
        'pre_score_capped',      pre_score_capped,
        'web_score_capped',      web_score_capped,
        'intl_score_capped',     intl_score_capped,
        'final_score',           final_score,
        'specialty_gate',        specialty_gate,
        'homepage_gate',         homepage_gate,
        'structured_specialty',  structured_specialty_flag
    ),
    'v2_mvp_batch'
FROM festimate_core.clinic_filter_scores
WHERE final_decided_at >= NOW() - INTERVAL '1 hour';
```

---

## §9. Materialized View — Stage 2B 결과

```sql
CREATE MATERIALIZED VIEW festimate_core.v_clinic_stage2_verified AS
SELECT
    s.seed_id,
    s.institution_name_raw,
    s.institution_name_norm,
    s.city,
    s.district,
    s.phone_norm,
    s.latitude_wgs84,
    s.longitude_wgs84,

    fs.pre_score_capped,
    fs.web_score_capped,
    fs.intl_score_capped,
    fs.final_score,
    fs.specialty_gate,
    fs.homepage_gate,
    fs.structured_specialty_value,

    -- 보강 플래그
    EXISTS (
        SELECT 1 FROM festimate_core.clinic_accreditations a
        WHERE a.seed_id = s.seed_id
          AND a.accreditation_type = 'KHIDI_REGISTERED'
          AND a.accreditation_status = 'active'
    ) AS khidi_registered,
    EXISTS (
        SELECT 1 FROM festimate_core.clinic_accreditations a
        WHERE a.seed_id = s.seed_id
          AND a.accreditation_type = 'KAHF'
          AND a.accreditation_status = 'active'
    ) AS kahf_active,
    EXISTS (
        SELECT 1 FROM festimate_core.clinic_language_support l
        WHERE l.seed_id = s.seed_id AND l.language_code = 'ja'
    ) AS has_japanese_support,

    fs.stage2_status,
    fs.final_decided_at

FROM festimate_core.clinic_longlist_seed s
JOIN festimate_core.clinic_filter_scores fs ON fs.seed_id = s.seed_id
WHERE fs.stage2_status IN ('verified', 'review');

CREATE UNIQUE INDEX ON festimate_core.v_clinic_stage2_verified (seed_id);
```

---

## §10. 예상 집계 (전국 118,750건 기준)

| 단계 | 추정 건수 |
|------|----------|
| 1차 롱리스트 (지역·활성 통과) | ~4,000~4,500 |
| Hard Gate 통과 | ~3,500~4,000 |
| Stage 2A Provisional (pre_raw >= 18) | ~1,200~1,800 |
| Web 크롤링 완료 | ~1,200~1,800 |
| Stage 2B Verified (final >= 75 + gates) | **~200~400** |
| Review Queue | **~300~500** |
| Excluded (낮은 최종 점수) | 나머지 |

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-04-22 | 초안 — raw 합산 키워드 점수 3그룹 + 지역 + 품질 |
| v2.0 | 2026-04-22 | 전면 재작성 — 100점 cap(Pre20/Web60/Intl20), specialty_gate, homepage_gate, Stage 2A/2B 분리, structured_specialty_flag NULL 초기화 정책 |

*담당: Claude Desktop (기획)*
