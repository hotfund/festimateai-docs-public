# K-Beauty DB 상세 스키마 설계 (KBEAUTY_DB_SCHEMA.md)

> **버전**: v1.3
> **작성**: 2026-04-20
> **업데이트**: 2026-04-22
> **원칙**: 지금 당장 채우지 못해도 필드는 전부 선언. Phase별로 순차 입력.
> **담당**: Claude Desktop (기획) → Claude Code CLI (schema.sql 구현)

---

## 0. 테이블 전체 맵

```
[시술 마스터]
  procedures                  — 시술 종류, 안전 정보, 금기사항

[클리닉]
  clinics (ALTER)             — 기존 테이블 + 의료 특화 컬럼 대폭 추가
  clinic_doctors              — 의사 정보, 자격증, 언어, 경력
  clinic_procedures           — 클리닉 × 시술 연결 (가격, 장비, 조건)
  clinic_events               — OCR 이벤트 가격 (인스타 배너 추출)
  clinic_instagram_posts      — Instagram 포스트 수집 원본
  clinic_reports              — 신고 시스템 (가격차별, 부작용 미고지 등)

[데이터 수집 — ETL 파이프라인 심사 출구]
  raw_clinics                 — Stage 2B 자동 승격 후 관리자 심사 대기 → clinics 승인

[유저 활동]
  beauty_plans                — AI Beauty Planner 생성 결과 (lifecycle 상태 관리)
  clinic_inquiries            — 문의·예약 추적 + 수수료 정산
  procedure_reviews           — 2-phase 리뷰 (당일 평가 + 효과 평가)
  point_transactions          — 포인트 적립·차감 내역
  point_config                — 포인트 설정 (유효기간·한도·중복방지 포함, 관리자 수정 가능)

[운영·감사]
  admin_audit_log             — 전체 상태 전이 감사 로그 (승인/반려/조정 전부)

[ETL 파이프라인 — 별도 스키마 (이 문서 외부)]
  festimate_etl.etl_batches              → CLINIC_LONGLIST_ETL_SPEC.md
  festimate_raw.raw_clinic_source        → CLINIC_LONGLIST_ETL_SPEC.md
  festimate_core.clinic_longlist_seed    → CLINIC_LONGLIST_ETL_SPEC.md
  festimate_core.clinic_filter_scores   → CLINIC_2NDLIST_FILTER_SPEC.md
  festimate_ext.external_provider_raw   → CLINIC_ENRICHMENT_SPEC.md
  festimate_core.clinic_homepages       → CLINIC_ENRICHMENT_SPEC.md
  festimate_core.clinic_accreditations  → CLINIC_ENRICHMENT_SPEC.md
  festimate_core.clinic_language_support → CLINIC_ENRICHMENT_SPEC.md
  festimate_core.job_queue              → PIPELINE_AUTOMATION_SPEC.md
```

---

## 1. procedures (시술 마스터)

> 누가 채우나: 어드민 직접 입력 (Phase 1) → 의료 전문가 검토 (Phase 2)
> Phase 1 필수: name_jp, name_kr, category, downtime_days, flight_restriction_hours, price range, contraindications 핵심 항목

```sql
CREATE TABLE procedures (

  -- ── 식별 ─────────────────────────────────────────────────────────
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,

  -- ── 시술명 (다국어) ───────────────────────────────────────────────
  name_jp                         text    NOT NULL,   -- 施術名（日本語）  Phase 1
  name_kr                         text    NOT NULL,   -- 시술명（한국어）  Phase 1
  name_en                         text,               -- 향후 영어 확장    Phase 2
  aliases_kr                      text[]  DEFAULT '{}', -- 한국어 이명 (물광=수광 등)
  aliases_jp                      text[]  DEFAULT '{}', -- 일본어 이명

  -- ── 분류 ─────────────────────────────────────────────────────────
  category                        text    NOT NULL,
    -- 'skin' | 'hair' | 'nail' | 'dental' | 'eye' | 'body' | 'wellness' | 'color_analysis'
  subcategory                     text,

  -- ── 설명 (일본어) ─────────────────────────────────────────────────
  description_jp                  text,               -- 일반 설명
  mechanism_jp                    text,               -- 원리 ("보톡스는 근육을 일시적으로...")
  expected_results_jp             text,               -- 기대 효과
  real_expectation_jp             text,               -- 현실적 기대치 ("드라마틱한 변화보다...")

  -- ── 시술 기본 정보 ─────────────────────────────────────────────────
  invasiveness_level              int     CHECK (invasiveness_level BETWEEN 1 AND 3),
    -- 1=비침습(레이저·보톡스) 2=미세침습(필러·실) 3=침습(수술)
  doctor_required                 bool    DEFAULT true,  -- 의사만 가능? (간호사 불가?)
  anesthesia_type                 text    DEFAULT 'none',
    -- 'none' | 'cream' | 'local' | 'sedation' | 'general'
  anesthesia_duration_minutes     int,                -- 마취 소요 시간 (분)
  duration_min_minutes            int,                -- 최소 시술 시간
  duration_max_minutes            int,                -- 최대 시술 시간
  total_visit_time_minutes        int,                -- 상담+시술+회복 합계 (예약 시간 계산)
  pain_level                      int     CHECK (pain_level BETWEEN 1 AND 5),
    -- 1=통증없음 3=중간 5=강한통증
  pain_description_jp             text,               -- 통증 수준 설명

  -- ── 가격 (시장 기준 범위) ──────────────────────────────────────────
  price_krw_min                   int,                -- 시장 최저가 (원)
  price_krw_max                   int,                -- 시장 최고가 (원)
  price_jpy_min                   int,                -- 엔화 환산 (자동)
  price_jpy_max                   int,
  price_note_jp                   text,               -- 가격 조건 ("1부위 1회 기준")
  price_last_updated              date,               -- 가격 조사일

  -- ── 다운타임 ───────────────────────────────────────────────────────
  downtime_days_min               int     DEFAULT 0,
  downtime_days_max               int     DEFAULT 0,
  downtime_description_jp         text,               -- 다운타임 상세 ("붉음기 1-2일, 붓기 3-5일")
  downtime_visible_jp             text,               -- 외출 가능 여부 설명
  downtime_work_ok_days           int     DEFAULT 0,  -- 재직장 복귀 가능 일수

  -- ── 귀국 관련 안전 제한 (핵심!) ────────────────────────────────────
  flight_restriction_hours        int     DEFAULT 0,
    -- 0 = 제한없음. 예: 보톡스=0, HIFU=24, 전신마취=48
  flight_restriction_note_jp      text,               -- 비행 제한 이유 설명
  sun_exposure_restriction_days   int     DEFAULT 0,  -- 자외선 금지 일수
  sun_exposure_note_jp            text,               -- SPF 관리 안내
  exercise_restriction_hours      int     DEFAULT 0,  -- 운동 금지 시간
  alcohol_restriction_hours       int     DEFAULT 0,  -- 음주 금지 시간
  smoking_restriction_hours       int     DEFAULT 0,  -- 흡연 금지 시간
  makeup_restriction_hours        int     DEFAULT 0,  -- 화장 금지 시간
  shower_restriction_hours        int     DEFAULT 0,  -- 세안·샤워 금지 시간
  sauna_restriction_days          int     DEFAULT 0,  -- 사우나·찜질방 금지 일수
  contact_lens_restriction_hours  int     DEFAULT 0,  -- 렌즈 착용 금지 (눈 시술)
  facial_restriction_days         int     DEFAULT 0,  -- 다른 얼굴 시술 금지 일수

  -- ── 재시술 ────────────────────────────────────────────────────────
  re_treatment_interval_days      int,                -- 재시술 최소 간격 (일)
  re_treatment_notes_jp           text,               -- 재시술 주의사항
  typical_sessions_recommended    int,                -- 권장 시술 횟수

  -- ── 금기사항 ──────────────────────────────────────────────────────
  pregnant_contraindicated        bool    DEFAULT false,
  breastfeeding_contraindicated   bool    DEFAULT false,
  keloid_skin_risk                bool    DEFAULT false,  -- 켈로이드 피부 위험
  drug_interactions_jp            text[]  DEFAULT '{}',
    -- 금기 약물 목록 ["ワーファリン","アスピリン","血液サラサラ系"]
  allergy_contraindications_jp    text[]  DEFAULT '{}',
    -- 금기 알레르기 ["ボツリヌス毒素","ヒアルロン酸","麻酔薬"]
  skin_condition_contraindications_jp text[] DEFAULT '{}',
    -- 금기 피부 상태 ["活動性ニキビ","炎症中","日焼け直後"]
  medical_condition_contraindications_jp text[] DEFAULT '{}',
    -- 금기 전신 상태 ["心疾患","糖尿病","自己免疫疾患"]
  age_min                         int,                -- 최소 나이 (미성년자 금지 등)
  age_max                         int,                -- 최대 나이 제한
  contraindication_note_jp        text,               -- 금기 종합 안내 (일본어)

  -- ── 시술 전 준비 ──────────────────────────────────────────────────
  pre_care_days                   int     DEFAULT 0,  -- 시술 전 준비 기간 (일)
  pre_care_jp                     text,               -- 시술 전 주의사항
  pre_care_avoid_jp               text[],             -- 시술 전 피해야 할 것 목록
  fasting_required                bool    DEFAULT false, -- 금식 필요 여부
  fasting_hours                   int,                -- 금식 시간

  -- ── 시술 후 관리 ──────────────────────────────────────────────────
  post_care_jp                    text,               -- 시술 후 일반 주의사항
  post_care_critical_jp           text,               -- 절대 금지 사항 (굵게 표시)
  post_care_recommended_products_jp text,             -- 권장 제품/행동
  follow_up_required              bool    DEFAULT false, -- 추후 내원 필요 여부
  follow_up_days                  int,                -- 추후 내원 시기 (일)

  -- ── 2단계 평가 스케줄 ─────────────────────────────────────────────
  review_followup_days            int     DEFAULT 14, -- Phase 2 효과 평가 알림 타이밍 (시술 후 N일)
    -- 보톡스=14, 필러=21, 리쥬란=42, 레이저토닝=30, 울쎄라=90, 써마지=90
  downtime_alert_schedule         jsonb,
    -- {"D0":"격렬한 운동 금지.","D1":"세안 시 마찰 금지.","D3":"효과 나타나기 시작.","D7":"가벼운 운동 가능. 사우나 금지.","D14":"Phase 2 평가 유도"}
    -- 관리자 페이지에서 시술별 편집 가능

  -- ── 부작용 ───────────────────────────────────────────────────────
  common_side_effects_jp          text[]  DEFAULT '{}',
    -- 흔한 부작용 ["赤み（2-3日）","内出血（1週間）","腫れ（1-2日）"]
  rare_side_effects_jp            text[]  DEFAULT '{}',
    -- 드문 부작용 ["感染","アレルギー反応","神経損傷"]
  emergency_signs_jp              text,
    -- "以下の症状が出たら即座に医療機関へ：高熱、膿、ひどい腫れ..."
  side_effect_timeline_jp         text,               -- 부작용 시간대별 경과 설명

  -- ── 법적·동의 ─────────────────────────────────────────────────────
  requires_written_consent        bool    DEFAULT true,
  requires_medical_history        bool    DEFAULT false, -- 병력 청취 필요
  requires_photo_consent          bool    DEFAULT true,  -- 사진 촬영 동의
  legal_note_jp                   text,               -- 법적 고지 사항 (JP)

  -- ── 트렌드·노출 ──────────────────────────────────────────────────
  trending_score                  int     DEFAULT 0   CHECK (trending_score BETWEEN 0 AND 100),
  trending_reason_jp              text,
  seasonal_tags                   text[]  DEFAULT '{}', -- ['spring','summer','autumn','winter','yearround']
  target_areas_jp                 text[]  DEFAULT '{}', -- ['顔全体','額','エラ','首','全身']
  target_concerns_jp              text[]  DEFAULT '{}', -- ['たるみ','毛穴','シミ','小顔']

  -- ── 데이터 품질 ──────────────────────────────────────────────────
  data_completeness_pct           int     DEFAULT 0,  -- 입력 완성도 % (자동 계산)
  data_verified_by                text,               -- 검증한 의료 전문가 이름
  data_verified_at                timestamptz,
  data_source                     text,               -- '내부작성' | '의료자문' | '학술자료'

  -- ── 관리 ─────────────────────────────────────────────────────────
  is_active                       bool    DEFAULT true,
  created_at                      timestamptz DEFAULT now(),
  updated_at                      timestamptz DEFAULT now()
);

CREATE INDEX idx_procedures_category  ON procedures(category);
CREATE INDEX idx_procedures_trending  ON procedures(trending_score DESC);
CREATE INDEX idx_procedures_downtime  ON procedures(downtime_days_max);
CREATE INDEX idx_procedures_flight    ON procedures(flight_restriction_hours);
```

---

## 2. clinics (ALTER — 기존 테이블에 추가)

> 기존 clinics 테이블에 컬럼 추가. 기존 컬럼(name_i18n, lat, lng, region, district 등)은 유지.
> Phase 1 필수: consultation_jp, operating_hours, instagram_handle, foreigner_same_price, is_verified

```sql
ALTER TABLE clinics ADD COLUMN IF NOT EXISTS

  -- ── 지역 세분화 ──────────────────────────────────────────────────
  city                            text,
    -- '서울' | '부산' | '대구' | '대전' | '청주' | '광주'
  address_kr                      text,               -- 한국어 주소
  address_en                      text,               -- 영어 주소

  -- ── 연락처·SNS ────────────────────────────────────────────────────
  instagram_handle                text,               -- '@oo.clinic' 형태
  kakao_channel_id                text,               -- 카카오 채널 ID
  line_id                         text,               -- LINE ID
  youtube_url                     text,
  naver_blog_url                  text,
  naver_place_id                  text,               -- 네이버 place ID
  kakao_place_id                  text,               -- 카카오 place ID

  -- ── 운영 정보 ─────────────────────────────────────────────────────
  operating_hours                 jsonb,
    -- {"mon":"10:00-19:00","tue":"10:00-19:00","sat":"10:00-17:00","sun":"휴무"}
  holiday_info_jp                 text,               -- 휴일 안내 (JP)
  lunch_break                     jsonb,              -- {"start":"13:00","end":"14:00"}
  appointment_required            bool    DEFAULT true,
  walk_in_available               bool    DEFAULT false,
  same_day_booking_available      bool    DEFAULT false, -- 당일 예약 가능

  -- ── 일본어 대응 (핵심 차별화) ────────────────────────────────────
  consultation_jp                 bool    DEFAULT false,  -- 일본어 상담 가능
  consultation_jp_method          text[]  DEFAULT '{}',
    -- ['in_person','video','chat_line','chat_kakao','phone','email']
  interpreter_available           bool    DEFAULT false,  -- 통역 서비스
  interpreter_note_jp             text,               -- 통역 안내 (JP)
  jp_document_available           bool    DEFAULT false,  -- 일본어 안내문 보유
  jp_website_available            bool    DEFAULT false,  -- 일본어 홈페이지
  jp_review_count                 int     DEFAULT 0,  -- 일본인 리뷰 수 (추적)
  jp_response_time_hours          int,               -- 일본어 문의 평균 응답 시간

  -- ── 신뢰도·인증 ──────────────────────────────────────────────────
  is_verified                     bool    DEFAULT false,  -- FestiMate 관리자 인증
  verified_at                     timestamptz,
  license_no                      text,               -- 의료기관 개설 허가번호
  license_verified                bool    DEFAULT false,  -- 허가번호 검증 완료
  license_verified_at             timestamptz,
  business_registration_no        text,               -- 사업자등록번호
  malpractice_insurance           bool    DEFAULT false,  -- 의료사고 배상보험
  malpractice_insurance_amount_krw bigint,            -- 보험 한도 금액
  foreign_patient_agency_registered bool   DEFAULT false, -- 외국인환자 유치업자 등록
  foreign_patient_agency_no       text,               -- 등록 번호
  emergency_facility              bool    DEFAULT false,  -- 응급처치 시설 보유
  sterile_equipment_certified     bool    DEFAULT false,  -- 멸균 장비 인증

  -- ── 외국인 특화 ──────────────────────────────────────────────────
  foreigner_same_price            bool    DEFAULT false,  -- 외국인 동일가 확인됨
  foreigner_price_confirmed_at    timestamptz,
  foreigner_price_note_jp         text,               -- 외국인 가격 정책 설명 (JP)
  foreigner_price_reported_diff   bool    DEFAULT false,  -- 가격차별 제보 접수됨
  foreign_patient_experience_years int,               -- 외국인 환자 진료 경력 (년)
  japanese_patient_count_approx   text,
    -- '~50명' | '50~200명' | '200~500명' | '500명 이상'
  foreign_patient_total_approx    text,               -- 전체 외국인 환자 수 (범위)

  -- ── 시술 정책 ─────────────────────────────────────────────────────
  pre_consultation_required       bool    DEFAULT false, -- 사전 상담 필수
  pre_consultation_method         text,
    -- 'in_person_only' | 'video_ok' | 'chat_ok' | 'photo_ok'
  written_consent_required        bool    DEFAULT true,  -- 서면 동의서
  written_consent_jp_available    bool    DEFAULT false, -- 일본어 동의서 보유
  refund_policy_jp                text,               -- 환불 정책 (JP)
  revision_policy_jp              text,               -- 재시술 보장 정책 (JP)
  guarantee_policy_jp             text,               -- 결과 보장 정책 (JP)
  after_service_period_days       int,               -- AS 기간 (일)

  -- ── 시설 정보 ─────────────────────────────────────────────────────
  floor_info                      text,               -- "3층" 등
  parking_available               bool    DEFAULT false,
  parking_note_jp                 text,
  nearest_station_jp              text,               -- 가까운 지하철역 (JP)
  station_walk_minutes            int,                -- 도보 분수
  accessibility_wheelchair        bool    DEFAULT false, -- 휠체어 접근 가능

  -- ── 공급자 관리 ──────────────────────────────────────────────────
  supplier_tier                   text    DEFAULT 'pending',
    -- 'pending' | 'verified' | 'preferred'
  supplier_user_id                uuid    REFERENCES users(id),
  supplier_registered_at          timestamptz,
  commission_rate                 numeric(4,2) DEFAULT 20.0, -- 수수료율 %

  -- ── 숏리스트 ─────────────────────────────────────────────────────
  is_shortlisted                  bool    DEFAULT false,
  shortlist_score                 int     DEFAULT 0,
  shortlist_updated_at            timestamptz,
  super_featured                  bool    DEFAULT false,

  -- ── Provenance (데이터 출처 신뢰도) ──────────────────────────────
  source_type                     text    DEFAULT 'admin_manual',
    -- 'admin_manual' | 'kakao_api' | 'naver_api' | 'supplier_self' | 'ocr'
  capture_method                  text    DEFAULT 'manual_entry',
    -- 'manual_entry' | 'api_batch' | 'web_scrape' | 'ocr'
  confidence_level                text    DEFAULT 'medium',
    -- 'low' | 'medium' | 'high' | 'verified'
  evidence_url                    text;   -- 출처 URL
```

---

## 3. clinic_doctors (의사 정보)

> 누가 채우나: 어드민 직접 입력 or 공급자 자가등록
> Phase 1: 최소 1명 이상 입력 목표 (원장 정보)

```sql
CREATE TABLE clinic_doctors (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id                       uuid    REFERENCES clinics(id) ON DELETE CASCADE,

  -- ── 기본 정보 ─────────────────────────────────────────────────────
  name_kr                         text    NOT NULL,   -- 한국어 이름
  name_jp                         text,               -- 일본어 표기 (있으면)
  name_en                         text,
  profile_image_url               text,
  bio_jp                          text,               -- 의사 소개 (JP)

  -- ── 자격·면허 ─────────────────────────────────────────────────────
  specialty                       text    NOT NULL,
    -- '피부과전문의' | '성형외과전문의' | '일반의' | '치과의사' | '한의사' | '마취과전문의'
  specialty_jp                    text,               -- 일본어 전공명
  license_no                      text,               -- 의사면허 번호
  license_verified                bool    DEFAULT false,  -- 면허 검증 완료
  license_verified_at             timestamptz,
  board_certified                 bool    DEFAULT false,  -- 전문의 자격 보유
  board_certification_no          text,               -- 전문의 자격 번호
  additional_certifications_jp    text[]  DEFAULT '{}', -- 추가 자격/교육 목록

  -- ── 학력·경력 ─────────────────────────────────────────────────────
  medical_school                  text,               -- 출신 의과대학
  graduated_year                  int,
  training_hospitals              text[]  DEFAULT '{}', -- 수련 병원 목록
  experience_years                int,                -- 전체 경력 연수
  specialty_experience_years      int,                -- 현 전공 경력 연수

  -- ── 언어 능력 ─────────────────────────────────────────────────────
  speaks_korean                   bool    DEFAULT true,
  speaks_japanese                 bool    DEFAULT false,
  speaks_japanese_level           text,               -- 'basic' | 'conversational' | 'fluent' | 'native'
  speaks_english                  bool    DEFAULT false,
  speaks_english_level            text,
  speaks_chinese                  bool    DEFAULT false,

  -- ── 외국인 환자 경험 ──────────────────────────────────────────────
  foreign_patient_experience_years int,
  japanese_patient_count_approx   text,               -- '~50명' | '50~200명' 등
  international_training          text[]  DEFAULT '{}', -- 해외 연수 경력

  -- ── 전문 시술 ─────────────────────────────────────────────────────
  specialty_procedures            uuid[]  DEFAULT '{}',  -- 전문 시술 procedure_id 목록
  specialty_description_jp        text,               -- 전문 시술 설명 (JP)

  -- ── 수상·언론 ─────────────────────────────────────────────────────
  awards_jp                       text[]  DEFAULT '{}',
  media_appearances_jp            text[]  DEFAULT '{}',
  published_papers                text[]  DEFAULT '{}',

  -- ── 상태 ─────────────────────────────────────────────────────────
  is_primary                      bool    DEFAULT false,  -- 대표 원장 여부
  is_active                       bool    DEFAULT true,
  sort_order                      int     DEFAULT 0,
  created_at                      timestamptz DEFAULT now(),
  updated_at                      timestamptz DEFAULT now()
);

CREATE INDEX idx_clinic_doctors_clinic ON clinic_doctors(clinic_id);
CREATE INDEX idx_clinic_doctors_jp     ON clinic_doctors(speaks_japanese) WHERE is_active = true;
```

---

## 4. clinic_procedures (클리닉 × 시술 연결)

> 누가 채우나: 어드민 + 공급자 자가등록
> Phase 1 필수: clinic_id, procedure_id, price_krw, available_days

```sql
CREATE TABLE clinic_procedures (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id                       uuid    REFERENCES clinics(id) ON DELETE CASCADE,
  procedure_id                    uuid    REFERENCES procedures(id),
  doctor_id                       uuid    REFERENCES clinic_doctors(id),  -- 담당 의사

  -- ── 가격 ─────────────────────────────────────────────────────────
  price_krw                       int,                -- 이 클리닉 실제 가격 (원)
  price_jpy                       int,                -- 엔화 환산 (자동)
  original_price_krw              int,                -- 정가 (이벤트 전)
  price_note_jp                   text,
    -- "1部位1回分" | "麻酔クリーム込み" | "カウンセリング別途"
  foreigner_price_confirmed       bool    DEFAULT false, -- 외국인 동일가 이 시술 기준 확인
  price_last_updated              date,

  -- ── 장비 정보 ─────────────────────────────────────────────────────
  device_brand                    text,               -- 예: 'Ulthera', 'Thermage'
  device_model                    text,
  device_serial_no                text,               -- 장비 일련번호 (식약처 확인용)
  device_approved_kr              bool    DEFAULT false,  -- 식약처 허가 장비
  device_purchase_year            int,                -- 장비 도입 연도
  device_last_maintained          date,               -- 마지막 점검일
  device_count                    int,                -- 해당 장비 보유 대수

  -- ── 시술 조건 ─────────────────────────────────────────────────────
  anesthesia_included             bool    DEFAULT false,  -- 마취크림 포함
  aftercare_kit_included          bool    DEFAULT false,  -- 사후 관리 키트 포함
  aftercare_visit_included        bool    DEFAULT false,  -- 사후 내원 포함
  consultation_included           bool    DEFAULT true,
  consultation_duration_minutes   int,
  session_duration_minutes        int,                -- 시술 소요 시간 (분)
  total_time_minutes              int,                -- 방문 총 소요 시간 (분)
  min_age                         int,
  max_age                         int,

  -- ── 예약 ─────────────────────────────────────────────────────────
  available_days                  text[]  DEFAULT '{}',
    -- ['mon','tue','wed','thu','fri','sat','sun']
  available_hours_start           time,               -- 시술 가능 시작 시간
  available_hours_end             time,               -- 시술 가능 종료 시간
  min_booking_days_ahead          int     DEFAULT 0,  -- 최소 예약 선일
  max_booking_days_ahead          int,                -- 최대 예약 선일 (없으면 null)
  same_day_booking                bool    DEFAULT false, -- 당일 예약 가능
  cancellation_hours              int     DEFAULT 24, -- 취소 가능 마지노선 (시간 전)
  cancellation_policy_jp          text,               -- 취소 정책 (JP)

  -- ── 미디어 ───────────────────────────────────────────────────────
  before_after_photos             text[]  DEFAULT '{}',  -- before/after URL 목록
  procedure_video_url             text,               -- 시술 과정 영상
  result_photos                   text[]  DEFAULT '{}',

  -- ── 실적 ─────────────────────────────────────────────────────────
  treatment_count_approx          text,               -- 시술 건수 범위 "1,000건 이상"
  japanese_patient_count_approx   text,               -- 일본인 시술 건수 범위

  -- ── 상태 ─────────────────────────────────────────────────────────
  is_flagship                     bool    DEFAULT false,  -- 이 클리닉 대표 시술
  is_promoted                     bool    DEFAULT false,
  is_active                       bool    DEFAULT true,
  updated_at                      timestamptz DEFAULT now(),

  UNIQUE(clinic_id, procedure_id)
);

CREATE INDEX idx_clinic_procedures_clinic    ON clinic_procedures(clinic_id);
CREATE INDEX idx_clinic_procedures_procedure ON clinic_procedures(procedure_id);
CREATE INDEX idx_clinic_procedures_doctor    ON clinic_procedures(doctor_id);
```

---

## 5. clinic_events (OCR 이벤트 가격)

> 누가 채우나: Instaloader + GPT-4 Vision 자동 → 관리자 검토
> Phase 1: 자동 수집 후 관리자 승인

```sql
CREATE TABLE clinic_events (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id                       uuid    REFERENCES clinics(id) ON DELETE CASCADE,
  procedure_id                    uuid    REFERENCES procedures(id),

  -- ── 이벤트 내용 ──────────────────────────────────────────────────
  event_title_kr                  text,               -- OCR 원본 제목 (한국어)
  event_title_jp                  text,               -- 번역된 제목 (일본어)
  event_description_jp            text,               -- 이벤트 상세 설명 (JP)
  conditions_jp                   text,               -- 조건 ("新規会員限定" 등, JP)

  -- ── 가격 ─────────────────────────────────────────────────────────
  original_price_krw              int,                -- 정가 (원)
  event_price_krw                 int,                -- 이벤트가 (원)
  event_price_jpy                 int,                -- 이벤트가 (엔, 자동 환산)
  discount_rate                   int,                -- 할인율 %
  discount_note_jp                text,               -- 할인 조건 설명 (JP)
  session_count                   int     DEFAULT 1,  -- 포함 회수

  -- ── 기간 ─────────────────────────────────────────────────────────
  valid_from                      date,
  valid_until                     date,
  is_limited_quantity             bool    DEFAULT false, -- 선착순 한정
  limited_quantity                int,

  -- ── 원본 소스 ─────────────────────────────────────────────────────
  source_type                     text    DEFAULT 'instagram',
    -- 'instagram' | 'naver_blog' | 'homepage' | 'kakao' | 'manual'
  source_instagram_url            text,
  source_naver_url                text,
  raw_image_url                   text,               -- 원본 이미지 URL
  stored_image_path               text,               -- Supabase Storage 경로

  -- ── OCR 결과 ──────────────────────────────────────────────────────
  ocr_raw_text                    text,               -- GPT-4 Vision 추출 원문
  ocr_confidence                  numeric(3,2),       -- OCR 신뢰도 0.00~1.00
  ocr_processed_at                timestamptz,

  -- ── 검증 ─────────────────────────────────────────────────────────
  is_verified                     bool    DEFAULT false,  -- 관리자 검토 완료
  verified_at                     timestamptz,
  admin_note                      text,

  -- ── 상태 ─────────────────────────────────────────────────────────
  is_active                       bool    DEFAULT true,
  expired_at                      timestamptz,        -- 만료 처리 일시
  created_at                      timestamptz DEFAULT now()
);

CREATE INDEX idx_clinic_events_clinic  ON clinic_events(clinic_id);
CREATE INDEX idx_clinic_events_valid   ON clinic_events(valid_until) WHERE is_active = true;
CREATE INDEX idx_clinic_events_pending ON clinic_events(is_verified) WHERE is_verified = false;
```

---

## 6. procedure_reviews (2-phase 리뷰)

> 누가 채우나: 실제 시술 받은 Verified 유저만 작성 가능
> Phase 1 평가 (시술 당일, 300pt) → Phase 2 평가 (효과 평가, review_followup_days 후, 400pt)
> 사진 공개 시 추가 포인트 (부위+300pt / 얼굴+600pt)

```sql
CREATE TABLE procedure_reviews (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id                       uuid    REFERENCES clinics(id),
  procedure_id                    uuid    REFERENCES procedures(id),
  doctor_id                       uuid    REFERENCES clinic_doctors(id),
  user_id                         uuid    REFERENCES users(id),
  inquiry_id                      uuid,               -- REFERENCES clinic_inquiries(id) (나중 연결)

  -- ── 방문 인증 ─────────────────────────────────────────────────────
  is_verified_visit               bool    DEFAULT false,
  visit_date                      date,
  verified_via                    text,               -- 'inquiry_record' | 'receipt_photo' | 'admin'

  -- ── Before/After 사진 ─────────────────────────────────────────────
  before_photo_url                text,               -- 시술 전 사진 (비공개 기본: 기기 로컬 → 공개 선택 시 서버)
  after_photo_url                 text,               -- 시술 후/효과 사진
  photo_visibility                text    DEFAULT 'private',
    -- 'private'  — 비공개 (기기 로컬, 서버 전송 없음)
    -- 'area'     — 시술 부위만 크롭, 앱 내만 노출 (+300pt)
    -- 'face'     — 전체 얼굴, 앱 내만 노출, 외부 검색 차단 (+600pt)
  photo_visibility_updated_at     timestamptz,        -- 공개 단계 변경 시각

  -- ── Phase 1 — 시술 당일 평가 (300pt) ────────────────────────────
  phase1_submitted_at             timestamptz,        -- Phase 1 제출 시각 (null이면 미제출)
  p1_hospital_cleanliness         int     CHECK (p1_hospital_cleanliness BETWEEN 1 AND 5),
    -- 병원 청결도 (시설·기구 위생)
  p1_staff_kindness               int     CHECK (p1_staff_kindness BETWEEN 1 AND 5),
    -- 의사·스태프 친절도 (응대 태도)
  p1_jp_communication             int     CHECK (p1_jp_communication BETWEEN 1 AND 5),
    -- 일본어 소통 (일본어 상담 원활 여부)
  p1_wait_time                    int     CHECK (p1_wait_time BETWEEN 1 AND 5),
    -- 대기 시간 (예약 대비 실제 대기)
  p1_price_accuracy               int     CHECK (p1_price_accuracy BETWEEN 1 AND 5),
    -- 예상 가격과 일치 (사전 안내 vs 청구)
  p1_procedure_explanation        int     CHECK (p1_procedure_explanation BETWEEN 1 AND 5),
    -- 시술 과정 설명 (사전 충분한 설명 여부)
  p1_comment                      text,               -- 오늘 느낀 점 (자유 입력, 선택)

  -- ── Phase 2 — 효과 평가 (400pt + procedures.review_followup_days 후) ──
  phase2_submitted_at             timestamptz,        -- Phase 2 제출 시각 (null이면 미제출)
  p2_doctor_skill                 int     CHECK (p2_doctor_skill BETWEEN 1 AND 5),
    -- 의사 실력 (결과물 기준)
  p2_effect_satisfaction          int     CHECK (p2_effect_satisfaction BETWEEN 1 AND 5),
    -- 효과 만족도
  p2_value_for_money              int     CHECK (p2_value_for_money BETWEEN 1 AND 5),
    -- 가격 대비 만족도
  p2_would_revisit                int     CHECK (p2_would_revisit BETWEEN 1 AND 5),
    -- 다시 방문하고 싶다
  p2_would_recommend              int     CHECK (p2_would_recommend BETWEEN 1 AND 5),
    -- 주변에 추천하고 싶다
  p2_review_text                  text,               -- 상세 후기 (자유 입력, 선택)

  -- ── 포인트 ────────────────────────────────────────────────────────
  points_awarded                  int     DEFAULT 0,
    -- Phase 1 300 + Phase 2 400 + photo_visibility bonus (300/600) 합산
    -- 관리자 수정 가능 (point_config 기준)

  -- ── 귀국 후 상태 (Phase 2 작성 시) ───────────────────────────────
  downtime_actual_days            int,                -- 실제 다운타임 일수 (경험)
  flight_ok                       bool,               -- 귀국 비행 문제 없었나

  -- ── 실제 비용 ─────────────────────────────────────────────────────
  actual_price_paid_jpy           int,                -- 실제 지불 금액 (엔)
  foreigner_price_same            bool,               -- 외국인 동일가 경험 여부
  additional_cost_surprised       bool    DEFAULT false,
  additional_cost_note_jp         text,

  -- ── 관리 ─────────────────────────────────────────────────────────
  is_public                       bool    DEFAULT true,
  is_flagged                      bool    DEFAULT false,
  flag_reason                     text,
  helpful_count                   int     DEFAULT 0,
  created_at                      timestamptz DEFAULT now(),
  updated_at                      timestamptz DEFAULT now()
);

CREATE INDEX idx_reviews_clinic    ON procedure_reviews(clinic_id);
CREATE INDEX idx_reviews_procedure ON procedure_reviews(procedure_id);
CREATE INDEX idx_reviews_user      ON procedure_reviews(user_id);
CREATE INDEX idx_reviews_p1        ON procedure_reviews(phase1_submitted_at) WHERE phase1_submitted_at IS NOT NULL;
CREATE INDEX idx_reviews_p2        ON procedure_reviews(phase2_submitted_at) WHERE phase2_submitted_at IS NOT NULL;
CREATE INDEX idx_reviews_photo     ON procedure_reviews(photo_visibility) WHERE is_public = true;
```

---

## 7. clinic_inquiries (문의·예약 추적 + 수수료 정산)

> 누가 채우나: 유저 문의 시 자동 생성 → 완료 시 어드민 확인
> Phase 1: LINE/카카오 문의 링크 클릭 시 생성

```sql
CREATE TABLE clinic_inquiries (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id                         uuid    REFERENCES users(id),
  clinic_id                       uuid    REFERENCES clinics(id),
  procedure_id                    uuid    REFERENCES procedures(id),
  doctor_id                       uuid    REFERENCES clinic_doctors(id),
  beauty_plan_id                  uuid,               -- REFERENCES beauty_plans(id) (나중 연결)

  -- ── 문의 정보 ─────────────────────────────────────────────────────
  inquiry_channel                 text    NOT NULL,
    -- 'line' | 'kakao' | 'email' | 'phone' | 'festimate_chat'
  inquiry_message_jp              text,               -- 문의 내용 (일본어)
  preferred_date                  date,               -- 희망 시술일 1순위
  preferred_date_alt1             date,               -- 희망 시술일 2순위
  preferred_date_alt2             date,               -- 희망 시술일 3순위
  preferred_time                  text,               -- '오전' | '오후' | '저녁' | '상관없음'

  -- ── 유저 안전 정보 (문의 시 입력) ────────────────────────────────
  departure_date                  date,               -- 귀국일 (다운타임 계산)
  special_notes_jp                text,               -- 특이사항 (임신, 알레르기, 복용약 등)
  downtime_tolerance              text,               -- '없음' | '1-2일' | '3일이상'

  -- ── 진행 상태 ─────────────────────────────────────────────────────
  status                          text    DEFAULT 'sent',
    -- 'sent' → 'responded' → 'consultation_done' → 'booked' → 'completed' → 'cancelled' → 'no_show'
  sent_at                         timestamptz DEFAULT now(),
  responded_at                    timestamptz,
  consultation_done_at            timestamptz,
  booked_at                       timestamptz,
  completed_at                    timestamptz,
  cancelled_at                    timestamptz,
  cancellation_reason             text,

  -- ── 실제 시술 정보 (완료 후 기록) ────────────────────────────────
  actual_procedure_date           date,
  actual_procedure_id             uuid    REFERENCES procedures(id),  -- 실제 받은 시술
  actual_doctor_id                uuid    REFERENCES clinic_doctors(id),
  actual_price_paid_krw           int,                -- 실제 지불 금액 (원)
  actual_price_paid_jpy           int,                -- 실제 지불 금액 (엔)
  payment_method                  text,               -- 'card' | 'cash' | 'mixed'
  receipt_image_url               text,               -- 영수증 사진

  -- ── 수수료 정산 ──────────────────────────────────────────────────
  commission_rate                 numeric(4,2),       -- 적용 수수료율 %
  commission_krw                  int,                -- 수수료 금액 (원)
  commission_paid                 bool    DEFAULT false,
  commission_paid_at              timestamptz,
  invoice_no                      text,               -- 정산 청구서 번호

  -- ── 빠른 만족도 (완료 직후) ──────────────────────────────────────
  quick_rating                    int     CHECK (quick_rating BETWEEN 1 AND 5),
  review_requested_at             timestamptz,        -- 리뷰 요청 이메일 발송 시각
  review_id                       uuid,               -- REFERENCES procedure_reviews(id)

  created_at                      timestamptz DEFAULT now(),
  updated_at                      timestamptz DEFAULT now()
);

CREATE INDEX idx_inquiries_user    ON clinic_inquiries(user_id);
CREATE INDEX idx_inquiries_clinic  ON clinic_inquiries(clinic_id);
CREATE INDEX idx_inquiries_status  ON clinic_inquiries(status);
CREATE INDEX idx_inquiries_commission ON clinic_inquiries(commission_paid) WHERE status = 'completed';
```

---

## 8. clinic_reports (신고 시스템)

> 누가 채우나: 유저 신고
> Phase 1: 신고 폼 → 어드민 이메일 알림 → Supabase Studio에서 처리

```sql
CREATE TABLE clinic_reports (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id                       uuid    REFERENCES clinics(id),
  reporter_user_id                uuid    REFERENCES users(id),
  inquiry_id                      uuid,               -- 관련 문의 (있으면)

  -- ── 신고 내용 ─────────────────────────────────────────────────────
  report_type                     text    NOT NULL,
    -- 'price_discrimination'      외국인 추가 요금 청구
    -- 'side_effect_not_disclosed' 부작용 미고지
    -- 'unlicensed_procedure'      무면허 시술
    -- 'false_before_after'        가짜 before/after
    -- 'forced_upsell'             강제 추가 시술 권유
    -- 'refund_refused'            환불 거부
    -- 'hygiene_issue'             위생 문제
    -- 'language_misrepresentation' 일본어 가능이라 했으나 불가
    -- 'other'
  description_jp                  text    NOT NULL,   -- 신고 내용 (일본어)
  description_kr                  text,               -- 한국어 번역 (어드민용)
  incident_date                   date,               -- 사건 발생일
  amount_disputed_jpy             int,                -- 분쟁 금액 (엔)
  evidence_urls                   text[]  DEFAULT '{}', -- 증거 사진/영상/캡처

  -- ── 처리 ─────────────────────────────────────────────────────────
  status                          text    DEFAULT 'received',
    -- 'received' | 'investigating' | 'resolved' | 'dismissed'
  admin_note                      text,
  resolved_at                     timestamptz,
  resolution_jp                   text,               -- 처리 결과 안내 (JP → 신고자에게)

  -- ── 자동 제재 ─────────────────────────────────────────────────────
  penalty_applied                 text,
    -- null | 'warning' | 'shortlist_removed' | 'suspended' | 'banned'
  penalty_applied_at              timestamptz,

  created_at                      timestamptz DEFAULT now()
);

CREATE INDEX idx_reports_clinic ON clinic_reports(clinic_id);
CREATE INDEX idx_reports_status ON clinic_reports(status);
CREATE INDEX idx_reports_type   ON clinic_reports(report_type);
```

---

## 9. beauty_plans (AI Beauty Planner 결과)

> 누가 채우나: AI가 자동 생성 → 유저 저장
> Phase 1: 필수 (이게 있어야 AI Planner 작동)

```sql
CREATE TABLE beauty_plans (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id                         uuid    REFERENCES users(id),
  travel_plan_id                  uuid    REFERENCES travel_plans(id),  -- 여행 일정 연결

  -- ── AI 입력 파라미터 ──────────────────────────────────────────────
  input_cities                    text[]  NOT NULL,
  input_duration_days             int     NOT NULL,
  input_budget_jpy                int,
  input_categories                text[]  DEFAULT '{}',
  input_downtime_tolerance        text,               -- 'zero' | '1-2days' | '3plus'
  input_departure_date            date,               -- 귀국일 (핵심)
  input_special_notes             text,               -- 특이사항 자유입력
  travel_purpose                  jsonb   DEFAULT '{}',
    -- AI 플래너 진입 선택 결과
    -- {"kpop": true, "kbeauty": false, "food": true, "mate": true}
    -- 4옵션: K-Pop·K-Beauty·맛집·메이트 (복수 선택 가능)
  procedure_timing_pattern        text,
    -- 'A' — 시술 먼저: 한국 도착 직후 시술 → 다운타임 중 관광
    -- 'B' — 여행 먼저: 여행 온전히 즐기고 귀국 직전 시술 → 다운타임은 일본에서 (핵심 차별점)
    -- 'C' — 다운타임 없음: 여행 중 언제든 (보톡스·레이저토닝 등 저침습 시술만)

  -- ── AI 생성 결과 ─────────────────────────────────────────────────
  plan_title_jp                   text,
  plan_summary_jp                 text,
  plan_json                       jsonb   NOT NULL,   -- 전체 플랜 (days 배열)
  total_budget_estimate_jpy       int,
  ai_model_used                   text,               -- 'gpt-4o' 등
  ai_prompt_version               text,               -- 프롬프트 버전 추적

  -- ── 추천 클리닉·시술 ─────────────────────────────────────────────
  recommended_clinic_ids          uuid[]  DEFAULT '{}',
  recommended_procedure_ids       uuid[]  DEFAULT '{}',
  warnings_jp                     text[]  DEFAULT '{}',  -- 안전 경고 목록
  excluded_procedures_jp          jsonb   DEFAULT '[]',   -- 제외된 시술 + 이유

  -- ── 상태 (lifecycle) ─────────────────────────────────────────────
  status                          text    DEFAULT 'draft',
    -- 'draft'     — AI 생성 직후, 저장 전 (유저에게만 보임)
    -- 'scheduled' — 유저가 저장 완료, 예약 게시 대기 중
    -- 'active'    — 노출 중 (public이면 검색에도 나옴)
    -- 'expired'   — end_at 도과 or 여행 종료 후 자동 만료
    -- 'archived'  — 유저가 직접 보관처리 or 관리자 처리

  -- ── 타이밍 ────────────────────────────────────────────────────────
  publish_at                      timestamptz,        -- 예약 게시 시각 (NULL=즉시)
  start_at                        timestamptz,        -- 플랜 활성 시작 (NULL=publish_at 동일)
  end_at                          timestamptz,        -- 자동 만료 예정 시각 (NULL=무기한)

  -- ── 노출 권한 ─────────────────────────────────────────────────────
  visibility_scope                text    DEFAULT 'user_only',
    -- 'user_only'  — 본인만 볼 수 있음
    -- 'public'     — 검색·추천에 노출 (is_public=true 연동)
    -- 'internal'   — 관리자·내부 전용 (미승인 클리닉 포함된 플랜 등)
  is_public                       bool    DEFAULT false,  -- visibility_scope='public' 시 true
  approval_required               bool    DEFAULT false,  -- 관리자 승인 후 public 허용 여부
  approved_by                     uuid,               -- 승인한 관리자 ID
  approved_at                     timestamptz,

  -- ── 집계 ─────────────────────────────────────────────────────────
  view_count                      int     DEFAULT 0,
  save_count                      int     DEFAULT 0,

  -- ── 연결된 문의 ──────────────────────────────────────────────────
  inquiry_ids                     uuid[]  DEFAULT '{}',  -- 이 플랜에서 생성된 문의

  -- ── FestiPoint 차감 ──────────────────────────────────────────────
  fp_used                         int     DEFAULT 0,  -- 사용된 FestiPoint
  fp_transaction_id               uuid,               -- 포인트 차감 기록 ID

  created_at                      timestamptz DEFAULT now(),
  updated_at                      timestamptz DEFAULT now()
);

CREATE INDEX idx_beauty_plans_user       ON beauty_plans(user_id);
CREATE INDEX idx_beauty_plans_travel     ON beauty_plans(travel_plan_id);
CREATE INDEX idx_beauty_plans_public     ON beauty_plans(is_public, created_at DESC);
CREATE INDEX idx_beauty_plans_status     ON beauty_plans(status);
CREATE INDEX idx_beauty_plans_publish_at ON beauty_plans(publish_at) WHERE status = 'scheduled';
CREATE INDEX idx_beauty_plans_end_at     ON beauty_plans(end_at) WHERE status = 'active';
```

---

## 10. raw_clinics (ETL 파이프라인 심사 대기 — Stage 2B → 관리자 승인 → clinics)

> 누가 채우나: ETL 파이프라인 자동 승격 (job_type: stage2_promote). 관리자 수동 추가도 가능.
> 처리 흐름: `festimate_core.clinic_longlist_seed` (ETL 1차) → `clinic_filter_scores` (점수 판정) → **raw_clinics** (관리자 심사) → `clinics` (승인 완료)
> 승격 조건 (Stage 2B): final_score ≥ 75 AND specialty_gate = TRUE AND homepage_gate = TRUE
> 연관 문서: CLINIC_LONGLIST_ETL_SPEC.md, CLINIC_2NDLIST_FILTER_SPEC.md, PIPELINE_AUTOMATION_SPEC.md

```sql
CREATE TABLE raw_clinics (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,

  -- ── ETL 파이프라인 연동 ────────────────────────────────────────────
  etl_seed_id                     bigint  REFERENCES festimate_core.clinic_longlist_seed(seed_id),
    -- ETL 파이프라인에서 승격된 경우 채움. 관리자 직접 추가는 NULL.
  etl_final_score                 int,    -- 승격 시점의 final_score (100pt 기준)
  etl_specialty_gate              bool,   -- 승격 시점의 specialty_gate 값
  etl_homepage_gate               bool,   -- 승격 시점의 homepage_gate 값
  etl_promoted_at                 timestamptz, -- ETL → raw_clinics 자동 승격 시각

  -- ── 행정안전부 CSV 원본 (1차 롱리스트) ───────────────────────────
  seed_id                         text    UNIQUE,     -- 행안부 개설허가번호 (≠ ETL 내부 bigint seed_id)
  source_dataset                  text,               -- 'MOHW_CLINIC_2024' (행안부 버전 태그)
  institution_name_raw            text,               -- CSV 원문 기관명
  institution_type_raw            text,               -- CSV 원문 기관종별 코드
  business_status_raw             text,               -- '정상' | '폐업' | '정지'
  permit_date                     date,               -- 개설 인허가일
  full_address_raw                text,               -- 지번 주소 원문
  road_address_raw                text,               -- 도로명 주소 원문
  specialty_codes_raw             text[]  DEFAULT '{}', -- 진료과목 코드 ['05','08']
  is_active_candidate             bool    DEFAULT false, -- K-Beauty 필터 통과 여부
  load_batch_date                 date,               -- CSV 적재 날짜

  -- ── 수집 메타 ─────────────────────────────────────────────────────
  source_type                     text    NOT NULL DEFAULT 'public_data',
    -- 'public_data' | 'kakao_api' | 'naver_api' | 'admin_manual' | 'supplier_self'
  capture_method                  text    NOT NULL DEFAULT 'csv_batch',
    -- 'csv_batch' | 'api_batch' | 'web_scrape' | 'manual_entry'
  confidence_level                text    DEFAULT 'medium',
    -- 'low' | 'medium' | 'high' | 'verified'
  evidence_url                    text,               -- 카카오맵 URL 등

  -- ── 카카오 API 보완 (2차 롱리스트) ──────────────────────────────
  raw_name_kr                     text,               -- 최종 사용 기관명 (CSV or 카카오)
  raw_category                    text,               -- 카카오 카테고리 원문
  raw_address                     text,
  raw_phone                       text,
  raw_lat                         numeric(9,6),
  raw_lng                         numeric(9,6),
  raw_homepage_url                text,
  raw_instagram                   text,
  kakao_place_id                  text,
  kakao_category_group            text,
  kakao_road_address              text,

  -- ── 웹 증거 수집 결과 ─────────────────────────────────────────────
  naver_rating                    numeric(3,2),
  naver_review_count              int,
  naver_blog_review_count         int,
  has_japanese_page               bool,               -- 일본어 페이지 발견 여부
  has_line_id                     bool,               -- LINE ID 발견 여부
  instagram_post_count            int,                -- 인스타 포스트 수 (Instaloader)

  -- ── KHIDI/KAHF 오버레이 (3단계) ──────────────────────────────────
  foreign_patient_registered      bool    DEFAULT false, -- KHIDI 외국인환자 유치 등록 여부
  kahf_certified                  bool    DEFAULT false, -- KAHF 인증 클리닉 여부
  kahf_valid_until                date,               -- KAHF 인증 유효기간
  supported_languages             text[]  DEFAULT '{}', -- 지원 언어 코드 ['ja','en']
  interpreter_available           bool    DEFAULT false, -- 통역사 보유 여부
  intl_coordinator_available      bool    DEFAULT false, -- 국제 코디네이터 유무
  intl_service_evidence_url       text,               -- 외국인 서비스 증거 URL

  -- ── 처리 상태 ─────────────────────────────────────────────────────
  status                          text    DEFAULT 'pending',
    -- 'pending'       — ETL 자동 승격 직후, 심사 대기
    -- 'under_review'  — 관리자가 심사 중 (클릭해서 상세 열람 상태)
    -- 'hold'          — 보완 필요, 자동 거절은 아닌 대기
    -- 'approved'      — 관리자 승인 완료 → clinics 테이블로 복사
    -- 'rejected'      — 관리자 반려 (rejection_reason 필수)
    -- 'duplicate'     — 이미 clinics에 동일 기관 존재
  approved_clinic_id              uuid    REFERENCES clinics(id),  -- 승인 후 생성된 clinic ID
  reviewed_by                     uuid,               -- 심사한 관리자 ID
  reviewer_note                   text,               -- 내부 메모
  rejection_reason                text,               -- 반려 사유 (status='rejected' 시 필수)
  hold_reason                     text,               -- 보류 사유

  captured_at                     timestamptz DEFAULT now(),
  reviewed_at                     timestamptz
);

CREATE INDEX idx_raw_clinics_status   ON raw_clinics(status);
CREATE INDEX idx_raw_clinics_source   ON raw_clinics(source_type);
CREATE INDEX idx_raw_clinics_seed     ON raw_clinics(seed_id);
CREATE INDEX idx_raw_clinics_active   ON raw_clinics(is_active_candidate) WHERE business_status_raw = '정상';
CREATE INDEX idx_raw_clinics_khidi    ON raw_clinics(foreign_patient_registered) WHERE is_active_candidate = true;
```

---

## 11. clinic_instagram_posts (Instagram 수집)

```sql
CREATE TABLE clinic_instagram_posts (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id                       uuid    REFERENCES clinics(id) ON DELETE CASCADE,
  instagram_handle                text,
  shortcode                       text    UNIQUE,     -- Instagram 포스트 고유 ID
  instagram_url                   text,

  -- ── 포스트 내용 ──────────────────────────────────────────────────
  caption_kr                      text,               -- 원본 캡션
  image_urls                      text[]  DEFAULT '{}',
  posted_at                       timestamptz,
  like_count                      int,
  comment_count                   int,
  hashtags                        text[]  DEFAULT '{}',

  -- ── OCR 처리 ─────────────────────────────────────────────────────
  is_event_post                   bool    DEFAULT false,  -- 이벤트 가격 포함 여부
  ocr_processed                   bool    DEFAULT false,
  ocr_event_id                    uuid    REFERENCES clinic_events(id),

  captured_at                     timestamptz DEFAULT now()
);

CREATE INDEX idx_instagram_posts_clinic  ON clinic_instagram_posts(clinic_id);
CREATE INDEX idx_instagram_posts_pending ON clinic_instagram_posts(is_event_post)
  WHERE ocr_processed = false AND is_event_post = true;
```

---

## 12. point_transactions (포인트 적립·차감 내역)

> 누가 채우나: 시스템 자동 생성 (리뷰 제출 시, AI Planner 사용 시 등)
> Phase 1: procedure_reviews 제출 시 적립. AI Planner 포인트 차감.

```sql
CREATE TABLE point_transactions (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id                         uuid    REFERENCES users(id) ON DELETE CASCADE,

  -- ── 거래 정보 ─────────────────────────────────────────────────────
  txn_type                        text    NOT NULL,
    -- 'earn_review_p1'      Phase 1 평가 제출 (+300pt 기본)
    -- 'earn_review_p2'      Phase 2 효과 평가 제출 (+400pt 기본)
    -- 'earn_photo_area'     Before/After 부위 공개 (+300pt 기본)
    -- 'earn_photo_face'     Before/After 얼굴 공개 (+600pt 기본)
    -- 'spend_ai_planner'    AI Planner 생성 사용 (-포인트)
    -- 'admin_adjust'        관리자 수동 조정
  amount                          int     NOT NULL,  -- 양수=적립, 음수=차감
  balance_after                   int     NOT NULL,  -- 거래 후 잔액

  -- ── 연결 오브젝트 ──────────────────────────────────────────────────
  review_id                       uuid    REFERENCES procedure_reviews(id),
  beauty_plan_id                  uuid    REFERENCES beauty_plans(id),

  -- ── 메모 ─────────────────────────────────────────────────────────
  note                            text,               -- 관리자 메모 or 시스템 메시지
  admin_user_id                   uuid,               -- admin_adjust 시 처리자

  created_at                      timestamptz DEFAULT now()
);

CREATE INDEX idx_point_txn_user    ON point_transactions(user_id, created_at DESC);
CREATE INDEX idx_point_txn_type    ON point_transactions(txn_type);
```

---

## 13. point_config (포인트 설정 — 관리자 수정 가능)

> 누가 채우나: 어드민 직접 수정
> 포인트 구조 전 항목 관리자 페이지에서 수정 가능. 유효기간·한도·중복방지까지 포함.

```sql
CREATE TABLE point_config (
  id                              uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  config_key                      text    UNIQUE NOT NULL,
    -- 'earn_review_p1'       Phase 1 평가 적립 포인트
    -- 'earn_review_p2'       Phase 2 효과 평가 적립 포인트
    -- 'earn_photo_area'      부위 공개 추가 포인트
    -- 'earn_photo_face'      얼굴 공개 추가 포인트
    -- 'spend_ai_planner'     AI Planner 1회 차감 포인트
  config_value                    int     NOT NULL,   -- 현재 설정값 (양수=적립, 음수=차감)
  config_value_default            int     NOT NULL,   -- 초기 기본값 (리셋용)
  description                     text,               -- 관리자 UI 표시 설명

  -- ── 유효기간 ──────────────────────────────────────────────────────
  point_validity_days             int,                -- 적립된 포인트 유효기간 (일수, NULL=무기한)
  valid_from                      date,               -- 이 정책 시작일 (NULL=즉시)
  valid_until                     date,               -- 이 정책 종료일 (NULL=무기한)

  -- ── 한도 ─────────────────────────────────────────────────────────
  max_per_event                   int,                -- 단일 이벤트당 최대 적립 한도 (NULL=무제한)
  max_per_user_lifetime           int,                -- 유저 생애 최대 누적 한도 (NULL=무제한)

  -- ── 중복 방지 ─────────────────────────────────────────────────────
  dedup_window_hours              int,
    -- 동일 이벤트(config_key) 중복 적립 방지 윈도우 (시간 단위)
    -- NULL = 제한 없음
    -- 예: earn_review_p1의 경우 리뷰 1건당 1회만 적립되므로 dedup은 review_id로 처리
  dedup_scope                     text    DEFAULT 'per_object',
    -- 'per_object'  — 동일 오브젝트(리뷰 ID 등)당 1회
    -- 'per_window'  — dedup_window_hours 내 1회
    -- 'lifetime'    — 유저 생애 1회

  -- ── 운영 제어 ─────────────────────────────────────────────────────
  is_active                       bool    DEFAULT true,  -- false면 이 이벤트로 포인트 발생 안 함
  allow_manual_adjust             bool    DEFAULT true,  -- 관리자 수동 조정(admin_adjust) 허용 여부

  -- ── 감사 ─────────────────────────────────────────────────────────
  updated_by                      uuid,               -- 마지막 수정한 관리자 ID
  updated_at                      timestamptz DEFAULT now()
);

-- 초기 기본값 삽입
INSERT INTO point_config (
  config_key, config_value, config_value_default, description,
  point_validity_days, dedup_scope, is_active, allow_manual_adjust
) VALUES
  ('earn_review_p1',  300, 300, 'Phase 1 당일 평가 제출 시 적립 포인트',   365, 'per_object', true, true),
  ('earn_review_p2',  400, 400, 'Phase 2 효과 평가 제출 시 적립 포인트',   365, 'per_object', true, true),
  ('earn_photo_area', 300, 300, 'Before/After 부위 공개 시 추가 포인트',    365, 'per_object', true, true),
  ('earn_photo_face', 600, 600, 'Before/After 얼굴 공개 시 추가 포인트',    365, 'per_object', true, true),
  ('spend_ai_planner',  0,   0, 'AI Planner 1회 사용 시 차감 포인트 (0=무료)', NULL, 'per_window', true, false);
```

---

## 14. admin_audit_log (상태 전이 감사 로그)

> 누가 채우나: 시스템 자동 삽입 (상태 변경 시 트리거 or 앱 레이어). 관리자 액션 전부 기록.
> 삭제 금지. 수정 금지. append-only 설계.

```sql
CREATE TABLE admin_audit_log (
  id                              bigserial PRIMARY KEY,

  -- ── 대상 오브젝트 ──────────────────────────────────────────────────
  target_type                     text    NOT NULL,
    -- 'raw_clinic'      — raw_clinics 심사 (승인/반려/보류)
    -- 'clinic'          — clinics 수정 (is_verified, supplier_tier 등)
    -- 'beauty_plan'     — beauty_plans 상태 변경
    -- 'point_config'    — point_config 값 수정
    -- 'procedure_review' — 리뷰 승인/숨김/삭제
    -- 'clinic_report'   — 신고 처리
    -- 'point_txn'       — 포인트 수동 조정
  target_id                       text    NOT NULL,   -- UUID or BIGINT (text로 통합 저장)
  target_label                    text,               -- 화면 표시용 (클리닉명, 유저명 등)

  -- ── 상태 전이 ─────────────────────────────────────────────────────
  action                          text    NOT NULL,
    -- 'approve'         — 승인
    -- 'reject'          — 반려
    -- 'hold'            — 보류
    -- 'promote'         — 상태 승격 (ETL → raw_clinics 등)
    -- 'deactivate'      — 비활성화
    -- 'activate'        — 재활성화
    -- 'manual_adjust'   — 포인트 수동 조정
    -- 'archive'         — 보관처리
    -- 'restore'         — 복원
    -- 'flag'            — 위험 플래그 표시
    -- 'config_update'   — 설정값 변경
  from_state                      text,               -- 이전 상태값
  to_state                        text,               -- 이후 상태값
  delta_value                     int,                -- 포인트 조정 등 수치 변화 (해당 시)

  -- ── 처리 주체 ─────────────────────────────────────────────────────
  actor_type                      text    NOT NULL,
    -- 'admin'           — 관리자 직접 액션
    -- 'system'          — ETL 워커, 크론 등 자동 처리
    -- 'user'            — 유저 직접 액션 (beauty_plan 보관 등)
  actor_id                        uuid,               -- 관리자/유저 UUID (system은 NULL)
  actor_label                     text,               -- 관리자 이름 or 'system:etl_worker' 등

  -- ── 근거·메모 ─────────────────────────────────────────────────────
  reason                          text,               -- 처리 사유 (reject/hold 시 필수)
  note                            text,               -- 내부 추가 메모
  evidence_url                    text,               -- 참조 URL (홈페이지 증거 등)

  -- ── 메타 ─────────────────────────────────────────────────────────
  ip_address                      inet,               -- 관리자 접속 IP (보안 감사용)
  user_agent                      text,               -- 브라우저 정보
  created_at                      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_target      ON admin_audit_log(target_type, target_id);
CREATE INDEX idx_audit_log_actor       ON admin_audit_log(actor_id, created_at DESC);
CREATE INDEX idx_audit_log_action      ON admin_audit_log(action, created_at DESC);
CREATE INDEX idx_audit_log_created_at  ON admin_audit_log(created_at DESC);
```

---

## ETL 파이프라인 연동 참조

`raw_clinics`는 4-schema ETL 파이프라인의 최종 출구입니다. 아래 흐름으로 데이터가 이동합니다.

### 파이프라인 흐름

```
행정안전부 CSV (118,750행)
  ↓  festimate_etl.etl_batches + festimate_raw.raw_clinic_source
festimate_core.clinic_longlist_seed (1차 롱리스트 canonical)
  ↓  festimate_core.clinic_filter_scores (Pre 20 + Web 60 + Intl 20 = 100pt)
Stage 2B verified (final_score ≥ 75 AND specialty_gate AND homepage_gate)
  ↓  job_type: stage2_promote (PIPELINE_AUTOMATION_SPEC.md)
raw_clinics (status = 'pending')  ← 이 문서 §10
  ↓  관리자 심사 (admin_audit_log 기록)
clinics (status = 'approved', approved_clinic_id 채워짐)
```

### 관련 문서

| 문서 | 내용 |
|------|------|
| `CLINIC_LONGLIST_ETL_SPEC.md` v2.0 | ETL 소스 파일, etl_batches / raw_clinic_source / clinic_longlist_seed DDL |
| `CLINIC_2NDLIST_FILTER_SPEC.md` v2.0 | 100pt 점수 체계, Stage 2A Provisional / Stage 2B Verified 판정 로직 |
| `CLINIC_ENRICHMENT_SPEC.md` v2.0 | external_provider_raw, clinic_homepages, clinic_accreditations, clinic_language_support |
| `PIPELINE_AUTOMATION_SPEC.md` v1.0 | job_queue DDL, Railway 워커, 실행 스케줄, 재시도 정책 |

---

## Phase별 입력 계획

| 테이블 | Phase 1 (런칭 전) | Phase 1.5 (런칭 후 2개월) | Phase 2 |
|--------|-----------------|--------------------------|---------|
| procedures | 20개 시술 + review_followup_days + downtime_alert_schedule | 의료 전문가 검토 | 확장 |
| clinics ALTER | 120~130개 핵심 필드 | 미입력 필드 채우기 | 자가등록 |
| clinic_doctors | 클리닉당 1명 (원장) | 전체 의사 | |
| clinic_procedures | 클리닉당 주요 3~5개 | 전체 | |
| clinic_events | OCR 자동 수집 | 자동화 | |
| procedure_reviews | 유저 생성 시작 (2-phase 구조) | 쌓이기 시작 | |
| point_transactions | 리뷰 제출 시 자동 생성 | | |
| point_config | 초기값 삽입 (런칭 전, validity/dedup 포함) | 포인트 정책 조정 | |
| clinic_inquiries | 문의 시작 시 자동 생성 | | |
| clinic_reports | 신고 폼 오픈 | | |
| beauty_plans | AI Planner 오픈 시 자동 (lifecycle 상태, publish_at/end_at 포함) | visibility_scope 운용 | |
| raw_clinics | ETL Stage 2B 자동 승격 (final_score ≥ 75) → 관리자 심사 | | |
| clinic_instagram_posts | Instaloader 가동 | | |
| admin_audit_log | 모든 관리자 액션 자동 기록 (append-only) | | |

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-04-20 | 의료 특화 전체 스키마 설계. 11개 테이블. 안전·신뢰·리뷰·신고·수수료 레이어 포함. |
| v1.1 | 2026-04-20 | clinics 테이블에서 vat_refund_eligible, vat_refund_guide_jp 컬럼 제거. 부가세 환급 2025-12-31 폐지. |
| v1.2 | 2026-04-21 | procedures: review_followup_days + downtime_alert_schedule jsonb 추가. procedure_reviews: 2-phase 완전 재설계 (p1 당일 6항목/p2 효과 5항목, photo_visibility, phase1/2_submitted_at, points_awarded). beauty_plans: travel_purpose jsonb + procedure_timing_pattern 추가. raw_clinics: 행정안전부 CSV 컬럼 (seed_id, source_dataset, institution_name_raw, institution_type_raw, business_status_raw, permit_date, full_address_raw, road_address_raw, is_active_candidate, load_batch_date) + KHIDI/KAHF 오버레이 컬럼 (foreign_patient_registered, kahf_certified, kahf_valid_until, supported_languages, interpreter_available, intl_coordinator_available, intl_service_evidence_url) 추가. point_transactions + point_config 신규 테이블 추가 (포인트 전 항목 관리자 수정 가능). |
| v1.3 | 2026-04-22 | [ETL 파이프라인 연동] raw_clinics: etl_seed_id (FK→clinic_longlist_seed) + etl_final_score/etl_specialty_gate/etl_homepage_gate/etl_promoted_at 추가. status 머신 재정의 (pending/under_review/hold/approved/rejected/duplicate) + reviewed_by/hold_reason 추가. 테이블 맵에 ETL 파이프라인 스키마 전체 열거. [beauty_plans lifecycle] status 재정의 (draft/scheduled/active/expired/archived) + publish_at/start_at/end_at/visibility_scope/approval_required/approved_by/approved_at 추가. 인덱스 3개 추가. [point_config 정책 확장] point_validity_days/valid_from/valid_until/max_per_event/max_per_user_lifetime/dedup_window_hours/dedup_scope/is_active/allow_manual_adjust 추가. INSERT 초기값 dedup_scope 포함 재작성. [admin_audit_log 신규] §14 append-only 감사 테이블. target_type/action/from_state/to_state/actor_type/actor_id/reason 포함. [ETL 파이프라인 연동 참조] 섹션 신규 추가 (파이프라인 흐름 + 관련 문서 표). |

*담당: Claude Desktop (기획)*
