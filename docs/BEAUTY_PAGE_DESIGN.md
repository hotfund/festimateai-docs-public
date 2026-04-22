# K-Beauty AI 서비스 설계 (BEAUTY_PAGE_DESIGN.md)

> **버전**: v0.5
> **작성**: 2026-04-20
> **업데이트**: 2026-04-20
> **상태**: 기획 확정
> **환경**: Claude Desktop (기획 전담)

> 📌 **로컬 규칙**: 모든 화면 설계는 한국어 버전만 작성. 구현 시 일본어·영어 추가.

---

## 개요

FestiMate v1 핵심 수익 드라이버. 메이트 매칭의 치킨-에그 문제를 우회하는 독립 서비스.

- **대상 유저**: 일본인 관광객 (비회원도 검색 가능, AI Planner는 Verified 이상)
- **언어**: 한국어 UI v1 → 구현 시 일본어·영어 추가
- **URL**: `/kr/beauty` (v1 한국어). 향후 `/jp/beauty`, `/en/beauty` 추가
- **v1 목표**: 6개 도시, 주요 시술 카테고리 커버, AI Beauty Planner MVP

---

## 0. Disclaimer 원칙 (법적 필수)

> FestiMate는 정보 제공 플랫폼이다. 의료기관·의료행위 중개가 아니다.
> 의사 역할을 침해하거나 의료법 위반 소지가 있는 모든 화면에 disclaimer를 표시한다.

### 0-1. 공통 Disclaimer 문구

```
[한국어 버전]
⚠ 이 정보는 일반 참고용입니다.
  시술 적합 여부는 반드시 의사와 상담 후 결정하세요.
  FestiMate는 의료기관이 아니며 의료 진단을 제공하지 않습니다.

[일본어 버전 — 구현 시]
⚠ この情報は一般的な参考情報です。
  施術の適否は必ず医師にご相談ください。
  FestiMateは医療機関ではなく、医療診断は提供していません。
```

### 0-2. Disclaimer 표시 위치 (필수)

| 위치 | 트리거 | Disclaimer 형태 |
|------|--------|----------------|
| AI 시술 추천 결과 | 증상 → 시술 매핑 시 | 추천 카드 하단 고정 텍스트 |
| AI Beauty Planner 결과 | 플랜 생성 완료 | 페이지 상단 배너 (항상 노출) |
| 다운타임·안전 경고 | "귀국 N일 전까지" 표시 시 | 경고 문구 바로 아래 |
| 금기 사항 표시 | 임신·알레르기·복용약 관련 | 금기 항목 옆 아이콘 + 툴팁 |
| 문의 템플릿 발송 전 | [발송] 버튼 누르기 전 | 체크박스 동의 (필수) |
| 클리닉 상세 페이지 | 항상 | 페이지 하단 고정 |

### 0-3. 화면별 Disclaimer 상세

**AI 시술 추천 (증상 → 시술 매핑)**
```
"모공이 넓다 → 레이저토닝이 효과적입니다"
아래 항상 표시:
⚠ AI 추천은 일반 정보 제공 목적입니다.
  개인 피부 상태에 따라 적합한 시술이 다를 수 있습니다.
  시술 전 반드시 클리닉 의사와 상담하세요.
```

**AI Beauty Planner 결과 페이지 (상단 고정 배너)**
```
┌─────────────────────────────────────────────────┐
│ ⚠ 이 플랜은 일정·비용 참고용입니다.              │
│   시술 적합 여부는 클리닉 의사가 최종 판단합니다. │
│   FestiMate는 의료기관이 아닙니다.               │
└─────────────────────────────────────────────────┘
```

**다운타임·비행 제한 경고**
```
⚠ 보톡스: 귀국 3일 전까지 받아야 합니다
  ※ 개인 회복 속도에 따라 다를 수 있습니다.
    의사 확인 후 최종 결정하세요.
```

**금기 사항 (임신·약물·알레르기)**
```
🚫 임신 중 이 시술 불가
  ※ 일반적 금기 사항입니다.
    본인의 구체적 상황은 반드시 의사에게 확인하세요.
```

**문의 템플릿 발송 전 (체크박스 필수 동의)**
```
□ FestiMate는 예약·상담을 대행하지 않습니다.
  클리닉과의 상담·계약은 본인 책임이며,
  시술 결정은 의사의 직접 진단 후 이루어져야 합니다.
  [동의 후 발송]
```

### 0-4. 절대 금지 표현

| 금지 표현 | 이유 | 대체 표현 |
|----------|------|---------|
| "~하면 됩니다" (시술 권유) | 의료 권유 = 의료법 위반 | "~에 효과적이라고 알려져 있습니다" |
| "이 시술을 받으세요" | 처방 행위 | "이 시술을 고려해볼 수 있습니다" |
| "안전합니다" | 의료적 보증 불가 | "일반적으로 부작용이 적은 편입니다" |
| "효과가 보장됩니다" | 허위 의료광고 | 표현 자체 사용 금지 |
| "의사 추천" (FestiMate가) | FestiMate는 의사 아님 | "클리닉 의사와 상담하세요" |

---

## 1. 서비스 레이어 구조

```
Layer 1: DB
  ├── clinics (기존)          — 클리닉 기본정보, 도시, 언어, 위치
  ├── procedures (신규)       — 시술 마스터 데이터 (종류, 다운타임, 가격대, 트렌드)
  └── clinic_procedures (신규) — 클리닉 × 시술 연결 (실제 가격, 예약 가능일)

Layer 2: 검색 (7차원)
  — 비회원도 접근 가능
  — 키워드 / 필터 / 랭킹 복합 검색

Layer 3: AI Beauty Planner
  — Verified 이상 (여권 인증 완료자)
  — 7개 입력 → 맞춤형 뷰티 플랜 생성
  — 여행 일정표(AI Planner)와 통합 가능
```

---

## 2. DB 스키마 확장

### 2-1. procedures 테이블 (신규)

```sql
CREATE TABLE procedures (
  id               uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  name_jp          text    NOT NULL,        -- 施術名（日本語）
  name_kr          text,                    -- 시술명 (한국어)
  name_en          text,                    -- 향후 영어 확장용
  category         text    NOT NULL,        -- ENUM 아래 참조
  subcategory      text,                    -- 세부 분류
  description_jp   text,                    -- 일본어 상세 설명
  mechanism_jp     text,                    -- 원리 설명 (AI 프롬프트 재료)
  downtime_days_min int    DEFAULT 0,       -- 최소 다운타임 (일)
  downtime_days_max int    DEFAULT 0,       -- 최대 다운타임 (일)
  downtime_notes_jp text,                   -- 다운타임 주의사항 (JP)
  pain_level       int     CHECK (pain_level BETWEEN 1 AND 5),  -- 1=무통증, 5=강함
  duration_min_minutes int,                 -- 최소 시술 시간 (분)
  duration_max_minutes int,                 -- 최대 시술 시간 (분)
  price_jpy_min    int,                     -- 시장 최소가 (엔)
  price_jpy_max    int,                     -- 시장 최대가 (엔)
  trending_score   int     DEFAULT 0,       -- 0-100, 매주 업데이트
  trending_reason_jp text,                  -- 트렌드 이유 설명 (AI용)
  seasonal_tags    text[]  DEFAULT '{}',    -- ['spring','summer','winter'] 등
  target_area      text[],                  -- ['face','jaw','forehead','arm'] 등
  contraindications_jp text[],              -- 금기 사항 목록 (JP) — 임신, 알레르기 등
  pre_care_jp      text,                    -- 시술 전 주의사항
  post_care_jp     text,                    -- 시술 후 주의사항 (귀국 전 체크)
  before_after_photos bool DEFAULT false,   -- before/after 공개 가능 여부
  invasiveness_level int CHECK (invasiveness_level BETWEEN 1 AND 3),
                                            -- 1=비침습, 2=미세침습, 3=침습
  language_required text DEFAULT 'none',    -- 'none' | 'basic' | 'fluent' (의사소통 필요 수준)
  is_active        bool    DEFAULT true,
  created_at       timestamptz DEFAULT now(),
  updated_at       timestamptz DEFAULT now()
);

-- category ENUM 값
-- 'skin'          → 피부과 시술 (레이저, 필러, 보톡스, 리프팅 등)
-- 'color_analysis'→ 퍼스널컬러 진단
-- 'hair'          → 헤어 시술 (염색, 탈색, 펌)
-- 'nail'          → 네일아트
-- 'body'          → 바디 시술 (제모, 셀룰라이트 등)
-- 'dental'        → 치과 미용 (라미네이트, 미백 등)
-- 'eye'           → 눈 관련 (쌍꺼풀, 눈밑지방 등)
-- 'wellness'      → 한방·스파·마사지 등

CREATE INDEX idx_procedures_category ON procedures(category);
CREATE INDEX idx_procedures_trending ON procedures(trending_score DESC);
```

### 2-2. clinic_procedures 테이블 (신규)

```sql
CREATE TABLE clinic_procedures (
  id             uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id      uuid    REFERENCES clinics(id) ON DELETE CASCADE,
  procedure_id   uuid    REFERENCES procedures(id),
  price_jpy      int,                       -- 이 클리닉의 실제 가격 (엔)
  price_krw      int,                       -- 원화 병기 (환율 참고용)
  is_promoted    bool    DEFAULT false,     -- 클리닉이 프로모션 설정한 시술
  is_flagship    bool    DEFAULT false,     -- 이 클리닉의 대표 시술
  available_days text[]  DEFAULT '{}',      -- ['mon','tue','wed','thu','fri','sat','sun']
  min_booking_days_ahead int DEFAULT 0,     -- 최소 예약 선일 수
  notes_jp       text,                      -- 클리닉 특이사항 (일본어)
  photos         text[]  DEFAULT '{}',      -- before/after 사진 URLs
  is_active      bool    DEFAULT true,
  updated_at     timestamptz DEFAULT now(),
  UNIQUE(clinic_id, procedure_id)
);

CREATE INDEX idx_clinic_procedures_clinic ON clinic_procedures(clinic_id);
CREATE INDEX idx_clinic_procedures_procedure ON clinic_procedures(procedure_id);
```

### 2-3. clinic_events 테이블 (신규 — OCR 이벤트 가격)

```sql
CREATE TABLE clinic_events (
  id               uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id        uuid    REFERENCES clinics(id) ON DELETE CASCADE,
  procedure_id     uuid    REFERENCES procedures(id),
  event_title_jp   text,                      -- OCR 추출 이벤트 제목 (일본어 번역 후 저장)
  event_title_kr   text,                      -- OCR 원본 한국어
  price_krw        int,                       -- OCR 추출 가격 (원)
  price_jpy        int,                       -- 자동 환산 (원 × 환율)
  original_price_krw int,                     -- 정가 (할인율 계산용)
  discount_rate    int,                       -- 할인율 % (계산값)
  valid_from       date,                      -- 이벤트 시작일
  valid_until      date,                      -- 이벤트 종료일
  source_instagram_url text,                  -- 원본 Instagram 포스트 URL
  ocr_raw_text     text,                      -- GPT-4 Vision 원본 추출 텍스트
  ocr_confidence   numeric(3,2),              -- OCR 신뢰도 0.00~1.00
  is_verified      bool    DEFAULT false,     -- 관리자 검토 완료 여부
  is_active        bool    DEFAULT true,
  created_at       timestamptz DEFAULT now()
);

CREATE INDEX idx_clinic_events_clinic ON clinic_events(clinic_id);
CREATE INDEX idx_clinic_events_valid ON clinic_events(valid_until) WHERE is_active = true;
```

### 2-4. raw_clinics 테이블 (신규 — 카카오API 수집 원본)

```sql
CREATE TABLE raw_clinics (
  id               uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  source_type      text    NOT NULL,          -- 'kakao_api' | 'naver_api' | 'admin_manual' | 'supplier_self'
  capture_method   text    NOT NULL,          -- 'api_batch' | 'web_scrape' | 'manual_entry' | 'ocr'
  confidence_level text    DEFAULT 'low',     -- 'low' | 'medium' | 'high' | 'verified'
  evidence_url     text,                      -- 원본 URL (카카오맵 링크 등)
  raw_name_kr      text    NOT NULL,          -- 카카오API 반환 원본 이름
  raw_address      text,
  raw_phone        text,
  raw_category     text,                      -- 카카오 카테고리 코드
  raw_lat          numeric(9,6),
  raw_lng          numeric(9,6),
  raw_instagram    text,                      -- 수동 추가 또는 Instaloader 수집
  review_count     int,
  kakao_place_id   text    UNIQUE,            -- 중복 방지
  status           text    DEFAULT 'pending', -- 'pending' | 'approved' | 'rejected' | 'duplicate'
  approved_clinic_id uuid  REFERENCES clinics(id),  -- 승인 후 연결
  reviewer_note    text,
  captured_at      timestamptz DEFAULT now(),
  reviewed_at      timestamptz
);

CREATE INDEX idx_raw_clinics_status ON raw_clinics(status);
CREATE INDEX idx_raw_clinics_source ON raw_clinics(source_type);
```

### 2-5. clinic_instagram_posts 테이블 (신규 — Instaloader 수집)

```sql
CREATE TABLE clinic_instagram_posts (
  id               uuid    DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id        uuid    REFERENCES clinics(id) ON DELETE CASCADE,
  instagram_url    text    NOT NULL UNIQUE,
  post_date        date,
  image_urls       text[]  DEFAULT '{}',      -- 게시물 이미지 URL 목록
  caption_kr       text,                      -- 원본 캡션 (한국어)
  is_event_post    bool    DEFAULT false,     -- GPT-4 Vision 판단: 이벤트 가격 포함 여부
  ocr_processed    bool    DEFAULT false,     -- OCR 처리 완료 여부
  ocr_event_id     uuid    REFERENCES clinic_events(id),  -- OCR 결과 연결
  captured_at      timestamptz DEFAULT now()
);

CREATE INDEX idx_instagram_posts_clinic ON clinic_instagram_posts(clinic_id);
CREATE INDEX idx_instagram_posts_event ON clinic_instagram_posts(is_event_post) WHERE ocr_processed = false;
```

### 2-6. clinics 테이블 추가 컬럼 (기존 테이블 ALTER)

```sql
-- 기존 clinics 테이블에 추가 필요한 컬럼들
ALTER TABLE clinics ADD COLUMN IF NOT EXISTS
  city text,                                -- '서울' | '부산' | '대구' | '대전' | '청주' | '광주'
  district text,                            -- '강남' | '홍대' | '명동' 등 세부 지역
  address_kr text,
  address_en text,
  lat numeric(9,6),
  lng numeric(9,6),
  languages_spoken text[] DEFAULT '{}',     -- ['ja','en','zh'] — 일본어 가능 여부 핵심
  consultation_jp bool DEFAULT false,       -- 일본어 상담 가능
  instagram_handle text,
  kakao_channel text,
  line_id text,
  operating_hours jsonb,                    -- {"mon": "10:00-19:00", "sat": "10:00-17:00"}
  is_verified bool DEFAULT false,           -- 관리자 인증 완료
  supplier_tier text DEFAULT 'verified',    -- MEMBERSHIP_TIERS §4 Supplier 등급
  review_count int DEFAULT 0,
  avg_rating numeric(3,2) DEFAULT 0,
  foreigner_same_price bool DEFAULT false,  -- 외국인 동일가 배지 (가격 차별 없음 확인됨)
  -- Provenance 추적 (데이터 출처 신뢰도)
  source_type text DEFAULT 'admin_manual',  -- 'admin_manual' | 'kakao_api' | 'supplier_self'
  capture_method text DEFAULT 'manual_entry', -- 'manual_entry' | 'api_batch' | 'ocr'
  confidence_level text DEFAULT 'high',    -- 'low' | 'medium' | 'high' | 'verified'
  evidence_url text;                        -- 출처 URL (카카오맵, 네이버 등)
```

---

## 3. 데이터 수집 전략 (클리닉 DB)

### Phase 1: 어드민 직접 입력 (시드 데이터)

- 도시별 상위 10-15개 클리닉 (6개 도시 × 10 = ~60개)
- 선별 기준: 일본어 대응 가능 / 리뷰 높음 / 접근성 / SNS 인지도
- 입력 우선순위: 서울(강남·홍대·명동) > 부산 > 나머지
- 어드민 CMS에서 시술 연결까지 완료
- **소요 예상**: 첫 60개 클리닉, 1-2주 집중 작업

### Phase 1.5: 공급자 자가 등록

- Supplier 티어 (MEMBERSHIP_TIERS.md §4) 그대로 활용
  - Pending → Verified (관리자 승인) → Preferred (평점 4.8+ × 20건)
- 등록 동기: 일본인 관광객 유입 → 클리닉 측 강한 유인
- 수수료: Phase 1 일괄 20% (예약 성사 시)
- 등록 페이지: `/supplier/register` (별도 설계)

### Phase 2: 파트너십 & 확장

- 미용 의료 관광 에이전시 B2B 연계
- 강남구청·부산시 의료관광 지원 사업 활용
- 크롤링 **금지** (ToS 위반 + 데이터 신뢰도)

---

## 4. /kr/beauty 메인 랜딩 설계

### 4-1. URL 구조

```
/kr/beauty                        — 메인 랜딩 (4-섹션 데이터형)
/kr/beauty/procedures             — 시술 카탈로그
/kr/beauty/procedures/[id]        — 시술 상세
/kr/beauty/clinics                — 클리닉 목록
/kr/beauty/clinics/[id]           — 클리닉 상세
/kr/beauty/planner                — AI 뷰티 플래너
/kr/beauty/planner/result/[id]    — AI 플랜 결과
```

### 4-2. 메인 랜딩 화면 구조 (확정 — 4-섹션 데이터형)

> 4개 섹션 모두 실제 DB 데이터 자동 반영. 어드민이 수동으로 "인기 시술"을 고르지 않는다.
> "이번 주 트렌드" 제거 — 의료 시술은 주별로 트렌드가 바뀌지 않음.

---

#### 전체 레이아웃 (스크롤 순서)

```
┌─────────────────────────────────────────────────────┐
│  🔍  시술명 또는 고민으로 검색                         │
│  예: 모공 / 보톡스 / 울쎄라 / 퍼스널컬러              │
└─────────────────────────────────────────────────────┘

  ──────────────────────────────────────────────────────
  Section 1: 🇯🇵 일본인이 많이 받는 시술
  ──────────────────────────────────────────────────────
  (inquiry_count + review_count 합산 DESC. 클리닉 문의 및 리뷰 데이터 기반.)

  Section 2: 🔖 지금 이벤트 중
  ──────────────────────────────────────────────────────
  (clinic_events.valid_until >= today. OCR 자동 수집. 매일 업데이트.)

  Section 3: 🌿 이 계절 추천 시술
  ──────────────────────────────────────────────────────
  (procedures.seasonal_tags 기반. 어드민 월별 설정. 현재 달 자동 적용.)

  Section 4: 🆕 새로 뜨는 시술
  ──────────────────────────────────────────────────────
  (procedures.created_at >= 6개월 이내. OR trending_score 최근 급상승.)

  ──────────────────────────────────────────────────────
  하단 액션 섹션
  ──────────────────────────────────────────────────────
  [고민별로 찾기]  [인기 기기·브랜드로 찾기]  [AI 뷰티 플랜 만들기]
```

---

#### Section 1: 일본인이 많이 받는 시술

```
┌──────────────────────────────────────────────────────┐
│  🇯🇵 일본인이 많이 받는 시술                           │
│  FestiMate 문의·리뷰 데이터 기반                       │
└──────────────────────────────────────────────────────┘

  가로 스크롤 카드 (최대 8개 표시)

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  퍼스널컬러   │  │   물광주사    │  │ 보톡스 (턱)  │
  │    진단       │  │              │  │              │
  │  ★ 4.9 (142) │  │  ★ 4.8 (98)  │  │  ★ 4.7 (87)  │
  │  다운타임 없음│  │  다운타임 없음│  │  1-2일       │
  │  ₩15,000~    │  │  ₩45,000~    │  │  ₩80,000~    │
  │  [자세히 보기]│  │  [자세히 보기]│  │  [자세히 보기]│
  └──────────────┘  └──────────────┘  └──────────────┘

  데이터 출처: clinic_inquiries.procedure_id COUNT + procedure_reviews COUNT
  정렬: (30일 이내 문의 수 × 0.6) + (리뷰 수 × 0.4) DESC
```

---

#### Section 2: 지금 이벤트 중

```
┌──────────────────────────────────────────────────────┐
│  🔖 지금 이벤트 중                                     │
│  인스타그램에서 실시간 수집한 이벤트 가격               │
│  매일 자동 업데이트                        [전체 보기] │
└──────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────┐
  │  강남 OO피부과                                   │
  │  물광주사 특가   ₩38,000 → ₩28,000 (26% 할인)   │
  │  🔖 4월 30일까지   │  ✅ 외국인 동일가            │
  │  [클리닉 보기]                                   │
  └─────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────┐
  │  홍대 OO헤어                                     │
  │  탈색+염색 세트   ₩120,000 → ₩90,000 (25% 할인) │
  │  🔖 5월 5일까지   │  🇯🇵 일본어 상담 가능          │
  │  [클리닉 보기]                                   │
  └─────────────────────────────────────────────────┘

  데이터 출처: clinic_events JOIN clinics WHERE valid_until >= today AND is_active = true
  정렬: discount_rate DESC (할인율 높은 순)
  OCR 신뢰도 < 0.8 이면 관리자 검토 후 노출
```

---

#### Section 3: 이 계절 추천 시술

```
┌──────────────────────────────────────────────────────┐
│  🌿 이 계절 추천 시술                                  │
│  4월 — 봄 시즌 맞춤 추천                               │
└──────────────────────────────────────────────────────┘

  가로 스크롤 카드 (최대 6개)

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  레이저토닝   │  │  피부 필링    │  │ 퍼스널컬러   │
  │  (잡티 제거)  │  │  (봄 환절기)  │  │  진단        │
  │  다운타임 0일 │  │  다운타임 0일 │  │  다운타임 없음│
  │  [자세히 보기]│  │  [자세히 보기]│  │  [자세히 보기]│
  └──────────────┘  └──────────────┘  └──────────────┘

  데이터 출처: procedures WHERE seasonal_tags CONTAINS current_season
  월별 시즌 매핑:
    봄 (3-5월): 잡티 레이저, 필링, 퍼스널컬러
    여름 (6-8월): 미백, 제모, 스파
    가을 (9-11월): 리프팅, 보톡스, 헤어컬러
    겨울 (12-2월): 보습, 물광주사, 치아미백
  어드민이 seasonal_tags 직접 설정 (monthly)
```

---

#### Section 4: 새로 뜨는 시술

```
┌──────────────────────────────────────────────────────┐
│  🆕 새로 뜨는 시술                                    │
│  최근 6개월 이내 등록 또는 급상승 시술                  │
└──────────────────────────────────────────────────────┘

  가로 스크롤 카드 (최대 6개)

  ┌──────────────┐  ┌──────────────┐
  │  리쥬란힐러   │  │  실리프팅     │
  │  🆕 NEW       │  │  📈 급상승    │
  │  다운타임 1일 │  │  다운타임 2일 │
  │  ₩150,000~   │  │  ₩200,000~   │
  │  [자세히 보기]│  │  [자세히 보기]│
  └──────────────┘  └──────────────┘

  데이터 출처:
    - 신규: procedures.created_at >= NOW() - INTERVAL '6 months'
    - 급상승: trending_score가 이전 30일 대비 20점+ 상승
  정렬: 신규 우선, 급상승 그 다음
```

---

#### 하단 액션 섹션 (3버튼)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌────────────────┐  ┌────────────────┐  ┌──────────────────┐
  │  🔍 고민별로   │  │  💎 인기 기기·  │  │  🤖 AI 뷰티      │
  │     찾기       │  │   브랜드로 찾기  │  │   플랜 만들기    │
  │                │  │                │  │                  │
  │  모공 / 잡티   │  │  울쎄라 / 써마지│  │  귀국일·예산에   │
  │  처짐 / 소얼굴 │  │  리쥬란 / 보톡스│  │  맞는 플랜 자동  │
  │  탈색 / 염색   │  │  필러 / HIFU    │  │  생성            │
  │                │  │                │  │                  │
  │  [찾아보기 →]  │  │  [찾아보기 →]  │  │  [시작하기 →]    │
  └────────────────┘  └────────────────┘  └──────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**고민별로 찾기** → L1 키워드 온톨로지 입력 (§5-1)
**인기 기기·브랜드로 찾기** → L4 장비·성분 검색 (울쎄라, 써마지, 리쥬란 등)
**AI 뷰티 플랜 만들기** → `/kr/beauty/planner` (Verified 이상, §6)

---

## 5. 검색 설계

### 5-1. 6-Layer 키워드 온톨로지

유저는 시술명을 모른다. "모공이 넓어서" → "레이저토닝" 까지 AI가 연결해줘야 한다.

```
L1: 고민/효과 (증상·목적 — 유저가 실제로 검색하는 말)
  예: 毛穴が気になる / たるみが気になる / 小顔になりたい / 歯を白くしたい

L2: 시술군 (큰 분류)
  예: レーザー系 / ボトックス系 / フィラー系 / リフトアップ系 / ホワイトニング系

L3: 시술명 (구체적 시술 — procedures.name_jp)
  예: レーザートーニング / ボトックス（エラ） / ヒアルロン酸（涙袋） / HIFU

L4: 장비·성분 (고관여 유저 검색)
  예: ウルセラ / ヴォルマ / ヒアルロン酸 / ボツリヌストキシン

L5: 부위 (신체 부위)
  예: 顔全体 / エラ / 目元 / 唇 / 毛穴 / 体（脱毛）

L6: 운영 태그 (서비스 조건 필터)
  예: 日本語OK / ダウンタイムなし / 外国人同一価格 / 今日予約可 / 🔥トレンド
```

**검색 흐름 (3단계 — 증상 우선)**

```
Step 1: 유저가 고민 입력 (L1)
  "毛穴が気になる"

Step 2: AI가 시술군 매핑 (L1→L2→L3)
  → "レーザートーニング / ジェットピール / ケミカルピーリング 이 효과적입니다"
  → 해당 시술 카드 노출

Step 3: 유저가 L6 태그로 필터링
  [ダウンタイムなし ✓]  [日本語OK ✓]  [外国人同一価格 ✓]
  → 매칭 클리닉 리스트 노출
```

**검색창 UI (일본어)**

```
┌──────────────────────────────────────────────────────┐
│  🔍 お悩みや施術名で検索                               │
│  例: 毛穴 / たるみ / ボトックス / ネイル               │
└──────────────────────────────────────────────────────┘
  
  [よく検索されるお悩み]
  #毛穴  #たるみ  #小顔  #ニキビ跡  #歯を白く  #髪色
```

---

### 5-2. 검색 인터페이스 (필터)

```
[도시 ▼]  [카테고리 ▼]  [가격대 ▼]  [다운타임 ▼]  [지금 유행 ▼]  [언어 ▼]  [더 보기 ▼]
```

초기 로드: 트렌딩 시술 카드 그리드. 필터 변경 시 즉시 갱신.

### 5-3. 검색 7차원 정의

| 차원 | 필터 옵션 | 비고 |
|------|-----------|------|
| **도시 (city)** | 서울 / 부산 / 대구 / 대전 / 청주 / 광주 | 복수 선택 |
| **카테고리** | 피부과·컬러진단·헤어·네일·치과·눈·바디·웰니스 | 복수 선택 |
| **가격대** | ~¥10,000 / ¥10k-30k / ¥30k-100k / ¥100k+ | 단일 선택 |
| **다운타임** | 당일 귀가 가능 / 1-2일 / 3-7일 / 1주일+ | 단일 선택 |
| **지금 유행** | 토글 ON/OFF | trending_score > 70 필터링 |
| **언어** | 일본어 가능 클리닉만 | 토글 ON/OFF (기본 ON) |
| **침습도** | 비침습 / 미세침습 / 침습 | 단일 선택 |

### 5-4. 롱리스트 / 숏리스트 분리

경쟁사(강남언니·Creatrip)는 전체를 노출한다. 그러면 광고 올린 클리닉이 상단을 점령한다.
FestiMate는 품질 기반 2단계로 분리한다.

```
롱리스트 (Long List): 200~300개
  - 카카오API + 수동 수집 전체
  - 유저에게 보이지 않음
  - 6-gate 필터로 숏리스트 추출

6-Gate 필터 (모두 통과해야 숏리스트 진입):
  Gate 1: 일본어 대응 가능 (consultation_jp = true OR languages_spoken contains 'ja')
  Gate 2: 외국인 가격 차별 없음 (foreigner_same_price = true OR 미확인이면 보류)
  Gate 3: 기본 정보 완성도 (주소·전화·운영시간 입력 완료)
  Gate 4: 리뷰 최소 5건 이상 (review_count >= 5)
  Gate 5: 관리자 인증 완료 (is_verified = true)
  Gate 6: 최근 90일 이내 이벤트 또는 가격 정보 확인 (최신성)

숏리스트 (Short List): 40~60개
  - 6-gate 통과 + 100점 스코어링 상위
  - 유저에게 실제 노출되는 클리닉
  - 정렬: 스코어 DESC (광고 아님)
```

**100점 스코어링 기준**

| 항목 | 가점 | 비고 |
|------|------|------|
| 일본어 상담 가능 | +15 | consultation_jp = true |
| 외국인 동일가 배지 | +10 | foreigner_same_price = true |
| OCR 이벤트 가격 최신 (30일 이내) | +10 | clinic_events 최신 레코드 |
| 리뷰 Bayesian 보정 점수 4.7+ | +15 | 리뷰 수 반영 공정 비교 |
| 인스타그램 팔로워 1만+ | +5 | clinic_instagram_posts 수집 |
| 일정 예약 가능일 제공 | +5 | clinic_procedures.available_days |
| Preferred 공급자 티어 | +10 | supplier_tier = 'preferred' |
| confidence_level = 'verified' | +10 | 관리자 직접 확인 |
| 데이터 완성도 90%+ | +5 | 빈 컬럼 없음 |
| **감점** | | |
| 외국인 추가요금 제보 있음 | -20 | 신고 접수 시 |
| 최근 6개월 부정 리뷰 2건+ | -15 | |

---

### 5-5. 시술 카드 (검색 결과)

> 💡 **가격 표시 원칙**: 한국 가격을 엔화로만 표시. 일본 시장가 비교 없음.
> 일본인 유저는 자국 가격을 이미 안다. ¥8,000 숫자 하나면 충분.

```
┌─────────────────────────────────────┐
│  [🔥 트렌딩]  보톡스 사각턱 (V라인)   │
│                                     │
│  카테고리: 피부과  │  다운타임: 1-2일 │
│  시술 시간: 15-30분                  │
│  이 클리닉 가격: ¥8,000~             │
│                                     │
│  ⚠ 귀국 3일 전까지 받아야 합니다      │
│                                     │
│  가능 클리닉: 12개                   │
│  [자세히 보기]  [+ 플랜에 담기]       │
└─────────────────────────────────────┘
```

**[+ 플랜에 담기] 버튼 동작:**
- 누르면 화면 하단 플로팅 바에 카운트 표시: "3개 담음 → AI 플랜 만들기"
- 담은 시술은 하이라이트 처리 (이미 담긴 상태 표시)
- 비회원도 담기 가능. AI 플랜 생성 시 Verified 로그인 요청.

### 5-6. 클리닉 카드 (클리닉 검색 시)

```
┌─────────────────────────────────────┐
│  ★ 4.9 (리뷰 87개)  [🇯🇵 日本語OK]   │
│  강남 OO피부과                       │
│  서울 강남구 역삼동                   │
│                                     │
│  [✅ 外国人同一価格]                  │
│                                     │
│  대표 시술: 물광주사 / 보톡스 / 리프팅 │
│  🔖 イベント: 水光注射 ¥28,000（~4/30）│  ← OCR 이벤트 가격
│  운영: 월-토 10:00-19:00             │
│                                     │
│  [詳細を見る]  [問い合わせ]           │
└─────────────────────────────────────┘
```

**배지 정의**

| 배지 | 조건 | 설명 |
|------|------|------|
| `🇯🇵 日本語OK` | consultation_jp = true | 일본어 상담 가능 |
| `✅ 外国人同一価格` | foreigner_same_price = true | 내국인과 동일 가격 적용 확인 |
| `🔖 イベント中` | clinic_events 유효 레코드 존재 | OCR 수집 이벤트 가격 안내 |
| `🔥 トレンド` | trending_score > 70 | 이번 주 급상승 시술 |

### 5-7. 담기 → AI 플래너 연결 플로우

일본인 관광객은 항공+숙박 고정비용이 이미 지출됐다. 시술을 여러 개 받을수록 이득이다.
이 행동 패턴을 자연스럽게 유도하는 것이 담기 → AI 플래너 플로우.

```
유저 저니 전체 맵:

1단계: 뭘 할까? (검색·발견)
  └→ 시술 카드 탐색
  └→ [+ 플랜에 담기] × 2~3개

  ┌─────────────────────────────────────────┐
  │  🛒 3개 담음                             │
  │  보톡스 사각턱 / 물광주사 / 퍼스널컬러   │
  │                                         │
  │  [AI 플랜 만들기 →]                     │
  └─────────────────────────────────────────┘

  └→ 귀국일 입력 → AI 플래너 (§6)
  └→ 다운타임 역산 → 최적 일정 생성

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1.5단계: 사전 온라인 상담
  └→ AI 플랜 결과 들고 클리닉에 문의
  └→ 일본어 문의 템플릿 자동 생성
  └→ LINE/카카오로 발송

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

2단계: 방문·시술·평가·귀국 후 다운타임 관리
  └→ 방문 확정 + AI 조언 (사전/사후)
  └→ Phase 1 당일 평가 (병원 경험, 300pt)
  └→ 귀국 후 다운타임 알림 (푸시, 시술별)
  └→ Phase 2 효과 평가 (시술별 타이밍, 400pt~)
  └→ Before/After 공개 보상

※ 3단계 없음. 2단계에 전부 흡수.
```

---

## 6. AI Beauty Planner 설계

### 6-0. AI 역할 원칙 (확정)

| 구분 | AI 가능 | AI 불가 |
|------|---------|---------|
| 퍼스널컬러·피부타입 분석 | ✅ (Phase 2, 사진 기반) | |
| 시술 후보 추천 | ✅ "이런 고민엔 이 시술" | ❌ "당신은 이 시술 필요" (의료 진단) |
| 다운타임·안전 경고 | ✅ 데이터 기반 자동 표시 | |
| 귀국일 역산 일정 | ✅ 핵심 기능 | |
| 최종 의료 판단 | ❌ | ✅ 의사만 가능 (1.5단계 상담) |

**AI 신뢰도는 의료 진단이 아닌 데이터 정확성에서 온다.**
- 다운타임 계산은 틀릴 수 없는 데이터 — 이게 신뢰의 기반
- 얼굴 사진 분석 → Phase 2에서 비의료 영역만 (퍼스널컬러, 피부타입)
- "후보를 좁혀주는 AI, 최종 판단은 의사" 원칙 유지

---

### 6-1. 접근 권한

| 기능 | 비회원 | Member | Verified |
|------|--------|--------|---------|
| 시술 검색·열람 | ✅ | ✅ | ✅ |
| 클리닉 상세 열람 | ✅ | ✅ | ✅ |
| AI Beauty Planner | ❌ | ❌ | ✅ (월 5회 무료, 초과 시 3 FP) |
| 뷰티 플랜 저장 | ❌ | ❌ | ✅ |
| 여행 일정 통합 | ❌ | ❌ | ✅ |

> FestiPoint 소모 정책: MEMBERSHIP_TIERS.md §7.3 (AI Guide와 동일 구조 적용)

---

### 6-2. AI Beauty Planner 입력 폼 (7 fields, 3 sections)

URL: `/jp/beauty/planner`

---

**Section A: 기본 조건 (必須)**

```
Field 1: 방문 도시 (chips, 복수 선택)
  [서울] [부산] [대구] [대전] [청주] [광주]

Field 2: 체류 일수 (chips)
  [1日] [2日] [3日] [4-5日] [1週間以上]

Field 3: 총 예산 (chips, JPY 기준)
  [~¥30,000] [¥30k-100k] [¥100k-200k] [¥200k-500k] [¥500k+]
```

**Section B: 관심·제약 (必須)**

```
Field 4: 관심 카테고리 (chips, 복수 선택)
  [スキンケア] [パーソナルカラー] [ヘアカラー] [ネイル] 
  [デンタル] [目元] [ボディ] [ウェルネス]

Field 5: 다운타임 허용 (chips, 단일 선택)
  [ゼロがいい（当日帰国可）] [1-2日なら大丈夫] [3日以上でもOK]

Field 6: 귀국 예정일 (date picker)
  — 귀국일 기준 다운타임 역산 핵심 데이터
  — 플레이스홀더: "귀国日を教えてください（施術の安全スケジュールを計算します）"
```

**Section C: 개인 특이사항 (선택)**

```
Field 7: 특이사항 및 피부 고민 (text, 자유입력)
  플레이스홀더: 
  "例：現在妊娠中 / ニキビ肌が気になる / 以前リフティングを受けた経験あり / 
   アレルギーあり（花粉症）/ 特になし"
  — 금기 사항 판단에 사용
  — 최대 300자
```

**[AIビューティープランを作る] 버튼**

---

### 6-3. AI 프롬프트 구조 (Beauty Planner)

```
SYSTEM:
당신은 한국 뷰티 여행 전문 AI 플래너입니다.
일본인 관광객에게 안전하고 효과적인 한국 뷰티 일정을 제안합니다.
반드시 다운타임, 귀국일, 예산, 금기 사항을 고려하세요.
응답은 항상 일본어로 하세요.

USER_INPUT:
- 도시: {cities}
- 체류: {duration}일
- 예산: {budget_jpy}
- 관심 카테고리: {categories}
- 다운타임 허용: {downtime_tolerance}
- 귀국 예정일: {departure_date}
- 특이사항: {special_notes}

PROCEDURE_DATA: (DB에서 해당 조건으로 필터링한 시술 목록 JSON)
[{
  "name_jp": "...",
  "category": "...",
  "price_jpy_min": ...,
  "price_jpy_max": ...,
  "downtime_days_min": ...,
  "downtime_days_max": ...,
  "duration_minutes_min": ...,
  "contraindications_jp": [...],
  "post_care_jp": "..."
}, ...]

INSTRUCTION:
1. 특이사항과 금기 사항을 먼저 체크하세요. 해당 시술이 있으면 제외하고 경고를 표시하세요.
2. 귀국일을 역산하여, 다운타임이 귀국 전에 충분히 지나는 시술만 포함하세요.
3. 예산 범위 내에서 효과가 큰 시술 조합을 구성하세요.
4. 같은 날 다운타임이 겹치는 시술을 피하세요.
5. 일정에 식사·이동 시간을 포함하여 현실적으로 구성하세요.
6. OUTPUT_FORMAT에 맞게 응답하세요.

OUTPUT_FORMAT:
{
  "plan_title_jp": "...",
  "summary_jp": "...",
  "total_budget_estimate_jpy": ...,
  "days": [
    {
      "day": 1,
      "city": "서울",
      "items": [
        {
          "time": "午前10:00",
          "procedure_name_jp": "パーソナルカラー診断",
          "duration": "2時間",
          "price_estimate_jpy": 15000,
          "downtime": "なし",
          "tag": "confirmed",  // confirmed / recommended / negotiate / uncertain
          "note_jp": "..."
        }
      ]
    }
  ],
  "recommended_clinics": [
    {"clinic_name": "...", "clinic_id": "...", "reason_jp": "..."}
  ],
  "warnings": [
    "보톡스は귀国 3日前までに受けてください",
    "レーザー施術後48時間は日焼け止め必須"
  ],
  "not_recommended": [
    {"procedure_jp": "...", "reason_jp": "特異事項に該当するため除外しました"}
  ]
}
```

---

### 6-4. AI Beauty Planner 출력 화면

URL: `/jp/beauty/planner/result/[plan_id]`

```
┌─────────────────────────────────────────────────────┐
│  ✨ あなたのK-ビューティープラン                      │
│  ソウル2泊3日 │ 予算: ¥120,000〜                     │
└─────────────────────────────────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Day 1 │ ソウル (江南エリア)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  午前 10:00  パーソナルカラー診断
              ⏱ 2時間  │  💰 ¥15,000  │  ✅ ダウンタイムなし
              ────────────────────────────────────
              💡 診断結果は当日もらえます。
                 コスメ購入の参考にも使えます。

  午後 13:00  昼食（江南カフェ街で休憩）

  午後 14:30  水光注射
              ⏱ 30分  │  💰 ¥35,000  │  ✅ ダウンタイムなし
              ────────────────────────────────────
              💡 直後から肌が潤います。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Day 2 │ ソウル (弘大エリア)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  午前 10:00  ボトックス（エラ）
              ⏱ 15分  │  💰 ¥80,000  │  ⚠ 帰国3日前まで
              ────────────────────────────────────
              ⚠ 帰国日が3日後なので本日が受けられる最終日です。

  午後 14:00  ネイルアート
              ⏱ 2時間  │  💰 ¥8,000  │  ✅ ダウンタイムなし

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Day 3 │ 帰国日（施術なし推奨）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  帰国前はショッピングでお土産を。
  クリニックで購入できる韓国コスメも人気です。

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  📊 プランサマリー
  合計予算（目安）: ¥138,000
  施術数: 4件  │  内訳: スキン×2, ネイル×1, カラー×1

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  🏥 推薦クリニック

  [江南 OOクリニック]
   ★4.9 │ 日本語OK ✓ │ 水光注射・ボトックス対応
   [詳細を見る] [問い合わせ]

  [弘大 OOネイルサロン]
   ★4.8 │ 日本語OK ✓ │ 韓国アート専門
   [詳細を見る] [問い合わせ]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ⚠ 注意事項
  • ボトックス施術後は1週間程度、うつ伏せ就寝は禁止です
  • レーザー系施術後は48時間、日焼け止めを必ず使用してください
  • 帰国便が早朝の場合、前日夕方以降の施術は避けましょう

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────────────────────────────┐
  │  🌟 このプランを旅行スケジュールに追加しますか？    │
  │                                                    │
  │  FestiMate AI Plannerと連携することで、            │
  │  K-POPコンサート・グルメ・ショッピングも含めた      │
  │  完全な韓国旅行プランが作れます。                   │
  │                                                    │
  │  [旅行プランに追加する]   [このまま保存する]        │
  └──────────────────────────────────────────────────┘
```

---

### 6-5. 여행 일정 통합 흐름 (AI Planner 연계)

```
/jp/beauty/planner/result/[beauty_plan_id]
  ↓  [旅行プランに追加する] 클릭
/ai-guide?beauty_plan_id=[beauty_plan_id]
  → AI Planner가 뷰티 플랜의 시술 일정을 Day별 일정으로 자동 삽입
  → 남은 시간에 관광·식사·쇼핑 등 제안
  → 최종 TravelPlan에 뷰티 블록이 포함됨
```

**TravelPlan JSON 확장 (beauty_items 추가)**

```jsonc
// PLAN_TEMPLATE_SCHEMA.md day_items 내 beauty 타입 추가
{
  "type": "beauty",           // 기존: activity / food / transport / free
  "procedure_id": "proc_xxx",
  "procedure_name_jp": "水光注射",
  "clinic_id": "clinic_yyy",
  "clinic_name": "강남 OO클리닉",
  "price_estimate_jpy": 35000,
  "downtime_days": 0,
  "booking_status": "inquiry_sent",  // 예약 상태
  "tag": "confirmed"
}
```

---

## 7. v1 시술 마스터 데이터 초기 목록

Phase 1 런칭에 필요한 시술 마스터. 어드민이 직접 입력.

### 7-1. 피부과·에스테틱 (skin)

| 시술명 (JP) | 카테고리 | 다운타임 | 가격대 (JPY) | 트렌드 |
|------------|---------|---------|------------|-------|
| パーソナルカラー診断 | color_analysis | 0일 | 10k-25k | ⭐⭐⭐⭐ |
| 水光注射 | skin | 0일 | 25k-50k | ⭐⭐⭐⭐⭐ |
| ボトックス（エラ） | skin | 1-2일 | 50k-120k | ⭐⭐⭐⭐⭐ |
| ボトックス（額・目尻） | skin | 1-2일 | 30k-80k | ⭐⭐⭐⭐ |
| フィラー（唇） | skin | 1-3일 | 40k-100k | ⭐⭐⭐ |
| フィラー（涙袋） | skin | 1-3일 | 40k-100k | ⭐⭐⭐⭐ |
| レーザートーニング | skin | 0-1일 | 15k-40k | ⭐⭐⭐ |
| ピーリング（ジェットピール） | skin | 0일 | 10k-30k | ⭐⭐⭐ |
| リフトアップ（HIFU） | skin | 0-3일 | 80k-300k | ⭐⭐⭐⭐ |
| ニキビ跡治療 | skin | 1-3일 | 20k-80k | ⭐⭐⭐ |

### 7-2. 헤어 (hair)

| 시술명 (JP) | 카테고리 | 다운타임 | 가격대 (JPY) | 트렌드 |
|------------|---------|---------|------------|-------|
| ブリーチ（脱色） | hair | 0일 | 15k-40k | ⭐⭐⭐⭐⭐ |
| ヘアカラー（韓国ヘア） | hair | 0일 | 10k-30k | ⭐⭐⭐⭐⭐ |
| パーマ（韓国パーム） | hair | 0일 | 20k-50k | ⭐⭐⭐⭐ |
| ストレートパーマ | hair | 0일 | 20k-50k | ⭐⭐⭐ |

### 7-3. 네일 (nail)

| 시술명 (JP) | 카테고리 | 다운타임 | 가격대 (JPY) | 트렌드 |
|------------|---------|---------|------------|-------|
| ジェルネイル（韓国アート） | nail | 0일 | 8k-25k | ⭐⭐⭐⭐⭐ |
| ネイルケア | nail | 0일 | 5k-12k | ⭐⭐⭐ |

### 7-4. 치과 (dental)

| 시술명 (JP) | 카테고리 | 다운타임 | 가격대 (JPY) | 트렌드 |
|------------|---------|---------|------------|-------|
| 歯のホワイトニング | dental | 0일 | 15k-50k | ⭐⭐⭐⭐ |
| ラミネートベニア（1本） | dental | 0-1일 | 50k-150k | ⭐⭐ |

### 7-5. 웰니스 (wellness)

| 시술명 (JP) | 카테고리 | 다운타임 | 가격대 (JPY) | 트렌드 |
|------------|---------|---------|------------|-------|
| 韓方スパ・チムジルバン | wellness | 0일 | 3k-15k | ⭐⭐⭐ |
| アカスリ（韓国式垢すり） | wellness | 0일 | 5k-20k | ⭐⭐⭐ |
| 韓方美容（경락마사지） | wellness | 0일 | 20k-60k | ⭐⭐ |

---

## 8. 트렌드 스코어 업데이트 로직

```
trending_score = 
  (최근 30일 열람 수 × 0.3) +
  (최근 30일 AI 플랜 포함 횟수 × 0.4) +
  (최근 7일 SNS 언급 지수 × 0.2) +
  (현재 시즌 적합도 × 0.1)

→ 0-100으로 정규화
→ 매주 일요일 새벽 3시 크론 실행
→ trending_score > 70 → 🔥 트렌딩 배지 표시
```

Phase 1에서는 어드민이 수동으로 trending_score 설정 (크론 구현은 Phase 2).

---

## 9. SEO 전략 (일본어 검색 유입)

### 타겟 키워드 (JP)

```
1차: 韓国 美容 旅行 / 韓国 美容医療 / 韓国 整形 / 韓国 エステ
2차: 韓国 ボトックス 安い / 韓国 水光注射 / 韓国 パーソナルカラー
3차: 江南 クリニック / ソウル 美容 / 釜山 エステ
```

### 콘텐츠 SEO

- 시술별 설명 페이지 (JP) → `/jp/beauty/procedures/[id]`
- 도시별 뷰티 가이드 블로그 포스트
- 인스타그램 #韓国美容 연계 UGC

---

## 10. 확정된 설계 결정

| 항목 | 결정 | 근거 |
|------|------|------|
| **클리닉 예약 방식** | Phase 1: LINE/카카오채널 문의 링크 연결. Phase 2: FestiMate 직접 예약 | 의료법 리스크 최소화. 수수료 구조 확인 후 직접 예약 전환 |
| **리뷰 시스템** | FestiMate 자체 리뷰 (Verified 회원만 작성). 네이버·구글 리뷰 수는 참고 링크만 | 데이터 품질 통제. 위조 리뷰 방지 |
| **결제 수수료** | 의원급 20% (법적 상한 30% 이내). Preferred 19% | MEMBERSHIP_TIERS.md §4. 의료법 수수료 상한 준수 |
| **시술 가격 표시** | 카드: 최소가 "¥60,000~". 상세 페이지: 범위 "¥60,000~120,000" | 유저 클릭 유도 (낮은 가격 노출) + 상세에서 투명성 확보 |
| **다국어 확장** | 일본어 → 영어 → 중국어(번체) 순서 | v1 일본어 집중. 영어는 동남아·서구 관광객 대비 v2 |
| **트렌딩 SNS 지수** | Phase 1 어드민 수동 입력. Phase 2 Instagram API | Phase 1에서 API 연동 비용·복잡도 대비 가치 낮음 |

---

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v0.1 | 2026-04-20 | 초안 — 서비스 레이어, DB 스키마, /jp/beauty 페이지, AI Beauty Planner 설계 |
| v0.2 | 2026-04-20 | 6-layer 키워드 온톨로지 추가, 외국인 동일가 배지, OCR 이벤트 가격 배지, 롱리스트/숏리스트 분리, 6-gate 필터, 100점 스코어링, clinic_events/raw_clinics/clinic_instagram_posts 테이블, Provenance 컬럼 |
| v0.3 | 2026-04-20 | §4 메인 랜딩 한국어 4-섹션으로 교체. "이번 주 트렌드" 제거 → 일본인 많이 받는 시술/이벤트중/계절추천/새로뜨는 4섹션 확정. URL /jp/beauty → /kr/beauty. 한국어 우선 로컬 규칙 적용. |
| v0.4 | 2026-04-20 | §5-5 시술 카드에 [+ 플랜에 담기] 장바구니 버튼 추가. 가격 표시 원칙 확정 (한국 JPY만, 일본 비교 없음). §5-7 담기→AI플래너 연결 플로우 신규 (유저 저니 맵). §6-0 AI 역할 원칙 신규 (후보 좁히기/진단 아님/얼굴사진 Phase 2). |
| v0.5 | 2026-04-20 | §0 Disclaimer 원칙 신규 (법적 필수). 표시 위치 6개 확정. 화면별 문구 상세. 절대 금지 표현 5개 정의. |
| v0.6 | 2026-04-20 | TAX REFUND(부가세 환급) 전면 제거. 조세특례제한법 제107조의3 2025-12-31 폐지. vat_refund_eligible 컬럼, 배지, 스코어링 항목 삭제. |
| v0.7 | 2026-04-21 | §5-7 유저 저니 맵: 3단계 제거. 귀국 후 다운타임 관리를 2단계에 흡수. |

*담당: Claude Desktop (기획)*
