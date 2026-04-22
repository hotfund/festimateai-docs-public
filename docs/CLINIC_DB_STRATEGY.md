# 클리닉 DB 수집 전략 (CLINIC_DB_STRATEGY.md)

> **버전**: v0.3
> **작성**: 2026-04-20
> **업데이트**: 2026-04-21
> **상태**: 전략 확정
> **담당**: Claude Desktop (기획) + Claude Code CLI (파이프라인 구현)

---

## 0. 왜 이 작업이 경쟁 우위인가

- **강남언니·바비톡**: 한국 내수 사용자 대상. 일본어 없음. AI 일정 연동 없음.
- **Naver·Kakao 지도**: 클리닉 목록은 있지만 시술·가격·다운타임 구조화 없음.
- **여행사 패키지**: 소수 제휴 클리닉만, 투명한 가격 없음.

우리가 구축하는 것:
```
행정안전부 공공데이터 (전국 의원 인허가)
+ KHIDI 외국인 환자 등록 여부
+ KAHF 인증 태그
+ 웹 증거 (홈페이지, 인스타, LINE)
+ 실시간 이벤트 가격 (OCR)
+ 다운타임·주의사항 (AI 구조화)
+ AI Planner 일정 통합
= 경쟁자가 수년 걸려도 따라오기 어려운 데이터 레이어
```

시드 데이터를 공공데이터에서 출발하는 것이 핵심이다. 처음부터 전국 의원 인허가를 다 들고 시작하면, 롱리스트 구성에 드는 발굴 비용이 0에 가깝다. 이후 웹 증거로 보완하고, KHIDI·KAHF로 신뢰도를 올리고, 숏리스트를 선별한다.

---

## 1. 수집 대상 전체 지도

### 1-1. 클리닉 카테고리 (K-Beauty 집중)

| 카테고리 | 진료과목 코드 | 일본어 수요 |
|---------|-----------|-----------|
| 피부과 | 05 (피부과) | ⭐⭐⭐⭐⭐ |
| 성형외과 | 08 (성형외과) | ⭐⭐⭐⭐ |
| 치과 (미용) | 04 (치과) | ⭐⭐⭐ |
| 한의원 (미용) | 11 (한방) | ⭐⭐ |
| 헤어샵·네일·반영구 | 의원급 외 (별도 소스) | ⭐⭐⭐⭐⭐ |

> 행정안전부 CSV는 의원급(의료기관)만 포함. 헤어샵·네일은 카카오 Local API 별도 수집.

### 1-2. 대상 도시 × 지역

```
서울:  강남구(역삼·논현·신사)  /  마포구(홍대·합정·상수)  /  중구(명동·을지로)
       용산구(이태원·한남)  /  서초구(강남역)
부산:  서면  /  해운대  /  광안리
대구:  동성로  /  수성구
대전:  둔산동  /  유성구
청주:  성안길  /  복대동
광주:  충장로  /  봉선동
```

### 1-3. 데이터 소스 맵 (v0.3 개정)

| 소스 | 역할 | 수집 방법 | 난이도 |
|------|------|---------|--------|
| **행정안전부_건강_의원 CSV** | **1차 롱리스트 시드 (핵심)** | 공공데이터 포털 다운로드 | ★☆☆ |
| **카카오 Local API** | 2차 웹증거 보완 (좌표·전화) | 공식 API (월 30만 무료) | ★☆☆ |
| **KHIDI** | 외국인환자 등록 여부 오버레이 | API or 공공데이터 | ★★☆ |
| **KAHF** | 인증 클리닉 신뢰 태그 (숏리스트용) | 수동 or API | ★★☆ |
| **클리닉 홈페이지** | 시술·가격·일본어 대응 확인 | Playwright | ★★★ |
| **인스타그램** | 이벤트 배너, 프로모션 | Instaloader | ★★★ |
| **네이버 블로그 API** | 이벤트·가격 보완 | 공식 API (일 25,000 무료) | ★★☆ |
| 강남언니·바비톡 | 가격 벤치마크 (참고용) | 수동 참고 (ToS 위험) | — |

---

## 2. 수집 파이프라인 (4단계)

### 전체 흐름

```
[1단계] 1차 롱리스트 — 행정안전부 CSV 파싱
    행정안전부_건강_의원.csv (118,750개)
    → K-Beauty 진료과목 코드 필터 (피부과·성형외과)
    → 영업 중 (정상) 필터
    → raw_clinics 테이블 (source_dataset = 'MOHW_CLINIC_2024')
    → 예상: 수천 개 (서울 피부과+성형외과 기준 1,000+)

[2단계] 2차 롱리스트 — 웹 증거 수집
    1차에서 주요 도시·진료과 후보 선별
    → 카카오맵 API: 기관명 검색 → 좌표·전화 보완
    → 홈페이지 URL 탐색 (Naver/Google 검색)
    → 인스타그램 핸들 탐색
    → 일본어 대응 시그널 감지
    → raw_clinics.status = 'enriched'
    → 예상: 수백 개 (웹 증거 있는 클리닉만)

[3단계] KHIDI/KAHF 오버레이 — 신뢰 태그
    KHIDI 외국인환자 유치 등록 기관 (6,000+)
    → 기관명+주소 매칭 → foreign_patient_registered = true
    KAHF 인증 클리닉
    → 매칭 → kahf_certified = true / kahf_valid_until 설정
    (별도 리스트가 아님 — 신뢰도 태그)

[4단계] 숏리스트 — 6-gate 필터 + 스코어링
    2차 롱리스트 중 gate 통과 클리닉
    → 100점 스코어링 (BEAUTY_PAGE_DESIGN §5-4 기준)
    → is_shortlisted = true → clinics 테이블 반영
    → 목표: 40~60개
```

---

## 3. Step별 상세 설계

### Step 0: 행정안전부 CSV 파싱 (1차 롱리스트)

#### 3-0-1. CSV 원본 개요

```
파일명: 행정안전부_건강_의원.csv (혹은 유사 명칭)
행수: 118,750개 (전국 의원급 의료기관 전체)
키 컬럼:
  - 개설허가번호 (고유 식별자)
  - 요양기관명 (기관명)
  - 요양기관종별코드 (의원/병원/종합병원 등)
  - 진료과목코드 (피부과=05, 성형외과=08 등)
  - 영업상태 (정상/폐업/정지)
  - 개설일자 (허가일)
  - 소재지전체주소 (지번 주소)
  - 도로명전체주소 (도로명 주소)
```

#### 3-0-2. K-Beauty 필터 조건

```python
# 1차 롱리스트 추출 조건
KBEAUTY_SPECIALTY_CODES = [
    '05',  # 피부과
    '08',  # 성형외과
    '27',  # 미용의학 (일부 CSV에 존재)
]

TARGET_CITIES = ['서울', '부산', '대구', '대전', '청주', '광주']

df_filtered = df[
    (df['진료과목코드'].isin(KBEAUTY_SPECIALTY_CODES)) &
    (df['영업상태'] == '정상') &
    (df['소재지전체주소'].str.contains('|'.join(TARGET_CITIES)))
]
# 예상: 서울 피부과만 1,000+ / 서울 성형외과 500+
```

#### 3-0-3. raw_clinics 적재

```sql
-- 배치 적재 시
INSERT INTO raw_clinics (
  seed_id, source_dataset,
  institution_name_raw, institution_type_raw,
  business_status_raw, permit_date,
  full_address_raw, road_address_raw,
  specialty_codes_raw,
  is_active_candidate,
  load_batch_date,
  status, source_type, capture_method, confidence_level
)
VALUES (
  $1,  -- 개설허가번호
  'MOHW_CLINIC_2024',
  $2,  -- 요양기관명
  $3,  -- 요양기관종별코드
  $4,  -- 영업상태
  $5,  -- 개설일자
  $6,  -- 소재지전체주소
  $7,  -- 도로명전체주소
  $8,  -- 진료과목코드 배열
  true,
  CURRENT_DATE,
  'pending', 'public_data', 'csv_batch', 'medium'
);
```

---

### Step 1: 웹 증거 수집 (2차 롱리스트)

#### 3-1-1. 카카오맵 API 보완

```python
# 기관명 + 도시로 카카오 검색 → 좌표·전화·카테고리 매칭
queries = [
    f"{row['institution_name_raw']} {city}"
    for row, city in 1차_롱리스트_후보
]

# 수집 필드
{
    "kakao_place_id": "...",
    "name": "OO피부과",
    "category": "피부과의원",
    "address": "서울 강남구 역삼동 ...",
    "phone": "02-xxx-xxxx",
    "lat": 37.xxx,
    "lng": 127.xxx,
    "homepage_url": "https://...",
    "instagram_handle": "@oo.clinic",
    "source": "kakao_api",
}
```

#### 3-1-2. 홈페이지 → 일본어 대응 감지

```python
# 일본어 가능 판별 시그널 (우선순위 순)
signals = [
    "홈페이지 내 '日本語' 또는 '日本語対応' 텍스트",
    "일본어 페이지 링크 (/jp, /japanese)",
    "LINE 계정 보유 (일본인 대응 지표)",
    "인스타그램 일본어 포스팅",
    "네이버 리뷰 중 일본어 리뷰 존재",
]
# → has_japanese_page = true
# → has_line_id = true
# → languages_spoken 컬럼 ['ja'] 설정 (clinics 반영 시)
```

---

### Step 2: KHIDI/KAHF 오버레이

#### 3-2-1. KHIDI 외국인환자 등록 매칭

```python
# KHIDI (한국보건산업진흥원) 외국인환자 유치 등록 기관
# 공공데이터 포털 또는 KHIDI 직접 조회

# 매칭 로직: 기관명 + 주소 유사도 매칭 (퍼지)
from rapidfuzz import fuzz

def match_khidi(raw_clinic, khidi_records):
    best_score = 0
    for khidi in khidi_records:
        name_score = fuzz.token_sort_ratio(
            raw_clinic['institution_name_raw'],
            khidi['institution_name']
        )
        if name_score >= 85:
            return True, khidi
    return False, None

# 매칭 성공 시
raw_clinics.foreign_patient_registered = True
raw_clinics.supported_languages = khidi['support_languages']
raw_clinics.interpreter_available = khidi['interpreter_yn']
raw_clinics.intl_coordinator_available = khidi['coordinator_yn']
```

#### 3-2-2. KAHF 인증 태그

```python
# KAHF (Korea Allied Healthcare Facilities)
# 인증 클리닉 목록 취득 후 매칭

# 매칭 성공 시
raw_clinics.kahf_certified = True
raw_clinics.kahf_valid_until = khaf['expiry_date']
```

> KAHF는 롱리스트 시드가 아니다. 숏리스트 후보에 얹는 신뢰 태그.
> KAHF 인증 없어도 숏리스트에 올라갈 수 있다. 있으면 점수가 높아진다.

---

### Step 3: 이벤트·프로모션 수집

#### 3-3-1. 인스타그램 수집 전략

```
도구: Instaloader (Python, 공개 계정)

수집 대상:
  - 클리닉 인스타그램 계정 최근 포스트 (50개)
  - 이벤트성 포스트 필터링 (해시태그 기준)
  - 이미지 다운로드 (배너 이미지)

이벤트 포스트 판별 해시태그:
  #이벤트 #할인 #프로모션 #특가 #기간한정
  #이벤트가격 #신규회원 #이벤트중

수집 필드:
  - post_id, shortcode
  - image_urls (배너 이미지)
  - caption (텍스트)
  - posted_at
  - like_count, comment_count
  - hashtags[]
```

#### 3-3-2. 네이버 블로그 이벤트 수집

```
수집 쿼리: "[클리닉명] 이벤트 2026"
           "[클리닉명] 할인 프로모션"

네이버 블로그 검색 API (공식, 일 25,000 무료)
  - 키워드: "강남 피부과 이벤트", "홍대 성형 할인" 등

추출 목표:
  - 시술명
  - 이벤트 가격
  - 유효 기간
  - 조건 (신규 회원 한정 등)
```

#### 3-3-3. 이벤트 관련 테이블

```sql
CREATE TABLE clinic_events (
  id              uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id       uuid REFERENCES clinics(id),
  source          text,  -- 'instagram' | 'naver_blog' | 'homepage' | 'manual'
  source_url      text,
  raw_image_url   text,
  stored_image_path text,
  raw_caption     text,
  ocr_status      text DEFAULT 'pending',
  ocr_raw_text    text,
  extracted_json  jsonb,
  procedure_id    uuid REFERENCES procedures(id),
  price_jpy       int,
  price_krw       int,
  valid_from      date,
  valid_until     date,
  conditions_jp   text,
  is_active       bool DEFAULT true,
  admin_verified  bool DEFAULT false,
  collected_at    timestamptz DEFAULT now(),
  verified_at     timestamptz
);

CREATE TABLE clinic_instagram_posts (
  id              uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  clinic_id       uuid REFERENCES clinics(id),
  instagram_handle text,
  shortcode       text UNIQUE,
  caption         text,
  image_urls      text[],
  hashtags        text[],
  posted_at       timestamptz,
  like_count      int,
  comment_count   int,
  is_event_post   bool DEFAULT false,
  event_id        uuid REFERENCES clinic_events(id),
  collected_at    timestamptz DEFAULT now()
);
```

---

### Step 4: OCR + AI 구조화 (핵심 파이프라인)

#### 3-4-1. 이벤트 배너 OCR 파이프라인

```
이미지 입력 (JPG/PNG)
    ↓
[OCR 엔진]
  GPT-4 Vision (권장)
    - 한국어 이미지 텍스트 인식 정확도 높음
    - 가격 형식, 날짜 형식 자동 인식
    - 비용: $0.008/이미지 (1024×1024 기준)
    - 월 1,000개 이미지 → 약 $8

    ↓
[AI 구조화 프롬프트]
  OUTPUT_JSON:
  {
    "clinic_name": "OO피부과",
    "procedures": [
      {
        "name_kr": "물광주사",
        "original_price_krw": 150000,
        "event_price_krw": 89000,
        "discount_pct": 40,
        "session_count": 1,
        "notes": "1인 1회 한정"
      }
    ],
    "valid_from": "2026-04-01",
    "valid_until": "2026-04-30",
    "conditions": ["신규 회원 한정", "예약 필수"],
    "confidence": 0.92
  }

    ↓
[후처리]
  - 시술명 → procedures 마스터 테이블 매핑 (퍼지 매칭)
  - KRW → JPY 환율 변환
  - confidence < 0.7 → 관리자 검토 큐
  - confidence >= 0.7 → clinic_events 자동 반영
```

#### 3-4-2. 시술명 표준화 (Fuzzy Matching)

```python
# "물광주사" = "수광주사" = "물광 주사" = "スキンbooster"
# 1. procedures 마스터 테이블 aliases 컬럼 활용
# 2. 임베딩 기반 유사도 (text-embedding-3-small)
# 3. 매핑 안 되면 → new_procedure 큐 → 관리자 검토

ALTER TABLE procedures ADD COLUMN IF NOT EXISTS
  aliases_kr text[] DEFAULT '{}',
  aliases_jp text[] DEFAULT '{}';
```

---

### Step 5: 검증 및 반영

```
검토 큐 1: raw_clinics (pending/enriched 상태)
  → 이 클리닉을 clinics 테이블에 추가할지 결정
  → 카테고리 보정, 도시/지역 태그 확인

검토 큐 2: clinic_events (admin_verified = false)
  → OCR 결과 맞는지 확인
  → 가격, 기간, 시술명 확인 → 승인

검토 큐 3: 새 시술 (procedures 마스터 미매핑)
  → 신규 시술 인정? → procedures 마스터 추가

목표: 관리자 1인이 하루 1-2시간으로 검토 가능한 양
```

---

### Step 6: 자동 갱신 (Monitoring)

```
주기:
  인스타그램 신규 포스트 — 매일 새벽 3시
  네이버 블로그 이벤트 — 주 2회
  홈페이지 가격 변경 감지 — 주 1회 (해시 비교)

만료 이벤트 자동 비활성화:
  clinic_events.valid_until < today → is_active = false

trending_score 재계산:
  → 매주 일요일 새벽 3시 (BEAUTY_PAGE_DESIGN §8 참조)
```

---

## 3-7. 롱리스트 → 숏리스트 파이프라인 (품질 필터)

```
[1차 롱리스트] 수천 개
  행정안전부 CSV → K-Beauty 필터 → raw_clinics (is_active_candidate = true)

    ↓ 2차 작업 (웹 증거 + KHIDI/KAHF)

[2차 롱리스트] 수백 개
  raw_clinics (status = 'enriched')
  웹 증거 있음 (홈페이지 or 인스타 or 카카오 확인)
  + 수동 추가 (어드민 직접 입력)

    ↓ 6-Gate 필터 (자동 적용)

Gate 1: 일본어 대응 확인
  consultation_jp = true  OR  languages_spoken @> '["ja"]'
  → 미확인이면 보류 (pending_jp_check)

Gate 2: 외국인 동일가 확인
  foreigner_same_price = true  OR  아직 미확인(보류)
  → 제보로 차별 확인된 클리닉 자동 제외

Gate 3: 기본 정보 완성도
  address IS NOT NULL AND phone IS NOT NULL

Gate 4: 리뷰 최소 기준
  review_count >= 5

Gate 5: 관리자 인증 완료
  is_verified = true

Gate 6: 데이터 최신성
  updated_at >= now() - interval '90 days'

    ↓ 100점 스코어링 (BEAUTY_PAGE_DESIGN §5-4 참조)

[숏리스트] 40~60개
  → clinics 테이블 (is_shortlisted = true)
  → 유저에게 실제 노출
  → 정렬: score DESC
```

**숏리스트 자동 재계산 주기**: 매주 일요일 새벽 3시
**수동 강제 편입**: 관리자 override 가능 (super_featured 플래그)

```sql
ALTER TABLE clinics ADD COLUMN IF NOT EXISTS
  is_shortlisted bool DEFAULT false,
  shortlist_score int DEFAULT 0,
  shortlist_updated_at timestamptz,
  super_featured bool DEFAULT false;
```

---

## 4. 도시별 수집 우선순위 및 목표 수량

### Phase 1 목표 (v1 런칭 전)

| 도시 | 지역 | 피부과 | 헤어 | 네일 | 기타 | 합계 |
|------|------|--------|------|------|------|------|
| **서울 강남** | 역삼·논현·신사 | 15 | 5 | 5 | 5 | 30 |
| **서울 홍대** | 홍대·합정·상수 | 10 | 8 | 8 | 4 | 30 |
| **서울 명동** | 명동·을지로 | 5 | 5 | 5 | 5 | 20 |
| **부산** | 서면·해운대 | 8 | 4 | 4 | 4 | 20 |
| 대구 | 동성로 | 3 | 2 | 2 | 1 | 8 |
| 대전 | 둔산 | 3 | 2 | 2 | 1 | 8 |
| 청주 | 성안길 | 2 | 1 | 1 | 1 | 5 |
| 광주 | 충장로 | 2 | 1 | 1 | 1 | 5 |
| **합계** | | **48** | **28** | **28** | **22** | **126** |

> Phase 1 목표: **약 120~130개 클리닉**, 일본어 가능 클리닉 우선.
> 피부과·성형외과는 행정안전부 CSV에서 출발. 헤어·네일은 카카오 API 별도 수집.

### Phase 2 목표 (런칭 후 3개월)

- 서울 추가 지역 (이태원·강남역·신촌 등)
- 전체 **300개+**
- 공급자 자가등록 시스템 오픈 → 클리닉 직접 입력

---

## 5. 수집 기술 스택

```
[1차 롱리스트 — 공공데이터]
  - pandas / Python CSV 파싱  — 행정안전부 CSV 처리
  - rapidfuzz                  — KHIDI/KAHF 기관명 퍼지 매칭

[2차 웹증거 수집]
  - 카카오 Local API           — 기관명 검색, 좌표·전화 보완 (월 30만 무료)
  - Playwright (Python)        — 홈페이지 스크래핑 (JS 렌더링)
  - Instaloader (Python)       — 인스타그램 공개 계정
  - Naver Blog Search API      — 공식 API (일 25,000 무료)
  - httpx + BeautifulSoup      — 정적 페이지 파싱

[OCR + AI]
  - GPT-4 Vision               — 이벤트 배너 OCR + 구조화
  - GPT-4o                     — 텍스트 원문 구조화
  - text-embedding-3-small     — 시술명 퍼지 매칭

[이미지 처리]
  - Pillow (PIL)               — 이미지 전처리
  - OpenCV                     — 텍스트 영역 감지

[저장·오케스트레이션]
  - Supabase (PostgreSQL)      — DB
  - Supabase Storage           — 이벤트 배너 이미지
  - Python scripts             — 수집 스크립트 (cron)
  - GitHub Actions             — 자동 갱신 크론

[관리자 UI]
  - Supabase Studio            — 초기 관리자 UI
  - Next.js /admin 페이지     — 검토 큐 UI (Phase 2)
```

---

## 6. 법적 리스크 관리

### 수집 가능 (안전)

| 소스 | 근거 |
|------|------|
| 행정안전부 공공데이터 | 공공데이터 포털 개방 데이터, 상업적 이용 허용 |
| 카카오맵 API | 공식 API, ToS 허용 |
| 네이버 블로그 검색 API | 공식 API, ToS 허용 |
| KHIDI 공공데이터 | 공공데이터 포털 개방 |
| 공개 인스타그램 계정 | 공개 계정의 공개 포스트 (개인정보 없음) |
| 클리닉 홈페이지 | 공개 정보 (robots.txt 준수 필수) |

### 주의 필요

| 소스 | 리스크 | 대응 |
|------|--------|------|
| 네이버 지도 직접 스크래핑 | ToS 위반 가능성 | 카카오 API로 대체 |
| 강남언니·바비톡 스크래핑 | ToS 명시적 금지 | **절대 금지** — 수동 참고만 |
| 이미지 무단 게재 | 저작권 | OCR 결과(텍스트)만 DB 저장 |
| 개인 의사 정보 | 개인정보보호법 | 의료진 이름·사진 수집 최소화 |

### robots.txt 준수

```python
import urllib.robotparser
rp = urllib.robotparser.RobotFileParser()
rp.set_url(f"{base_url}/robots.txt")
rp.read()
if not rp.can_fetch("*", target_url):
    skip(target_url)

import time, random
time.sleep(random.uniform(2, 5))
```

---

## 7. Phase별 실행 계획

### Phase 1-A: 시드 데이터 구축 (런칭 전, 4-6주)

```
Week 1: 행정안전부 CSV 파싱 + 1차 롱리스트 구성
  [ ] 행정안전부_건강_의원.csv 다운로드 (공공데이터 포털)
  [ ] Python 파싱 스크립트 작성
  [ ] K-Beauty 진료과목 코드 필터 적용
  [ ] raw_clinics 테이블 적재 (서울 피부과+성형외과 우선)

Week 2: KHIDI/KAHF 오버레이 + 카카오 보완
  [ ] KHIDI 외국인환자 등록 기관 데이터 취득
  [ ] 퍼지 매칭으로 foreign_patient_registered 태깅
  [ ] 카카오맵 API로 좌표·전화 보완

Week 3-4: 서울 강남 30개 → 숏리스트 후보 선별
  [ ] 홈페이지 URL 수동 확인
  [ ] 일본어 가능 여부 확인
  [ ] 인스타그램 핸들 수집
  [ ] 기본 시술 목록 입력 (procedures 마스터)

Week 5-6: OCR 파이프라인 가동
  [ ] 이벤트 배너 500장+ OCR 처리
  [ ] clinic_events 테이블 채우기
  [ ] 관리자 검토 후 숏리스트 40~60개 확정
```

### Phase 1-B: 자동화 (런칭 후)

```
Month 2:
  [ ] 인스타그램 신규 포스트 자동 감지
  [ ] OCR 자동 실행
  [ ] 이벤트 만료 자동 처리
  [ ] 공급자 자가등록 베타 (일부 클리닉 초대)

Month 3:
  [ ] trending_score 자동 계산
  [ ] 검토 큐 관리자 UI
  [ ] 300개 클리닉 목표 달성
```

---

## 8. 데이터 품질 지표 (KPI)

| 지표 | Phase 1 목표 | Phase 2 목표 |
|------|-------------|-------------|
| 1차 롱리스트 | 3,000개+ (서울 기준) | 전국 |
| 2차 롱리스트 | 수백 개 | |
| 숏리스트 (is_shortlisted) | 40~60개 | 100개+ |
| 일본어 가능 클리닉 비율 | 40%+ | 60%+ |
| KHIDI 등록 클리닉 비율 | 30%+ | 50%+ |
| 시술별 실제 가격 보유율 | 60%+ | 80%+ |
| OCR 자동화율 | 60% | 85% |
| 이벤트 데이터 신선도 | 2주 이내 | 1주 이내 |

---

## 9. 경쟁 우위 분석

```
우리가 가진 것 (차별화 레이어):

1. 공공데이터 기반 전수 출발
   → 강남언니·바비톡은 직접 영업 발굴. 우리는 전국 인허가 DB에서 시작.
   → 발굴 비용 0, 누락 없음

2. KHIDI 외국인환자 등록 여부 가시화
   → "이 클리닉은 보건복지부 외국인환자 유치기관입니다" 배지
   → 일본인 유저 신뢰도 직결

3. 일본인 관점 필터링
   → "일본어 가능" 클리닉만 노출
   → 일본어 이벤트 설명 자동 생성

4. 실시간 이벤트 가격
   → 인스타그램 이벤트 배너 OCR
   → "지금 이 클리닉에서 물광주사 40% 할인 중"

5. AI 일정 통합
   → 다운타임 계산 포함 일정표
   → 귀국일 역산 안전 스케줄
   → 강남언니는 이걸 못 한다
```

---

## 10. 확정된 결정

| 항목 | 결정 | 근거 |
|------|------|------|
| **1차 롱리스트 소스** | 행정안전부_건강_의원 CSV (118,750개) | 공공데이터 전수, 무료, 인허가 기준 신뢰성 |
| **카카오 Local API 역할** | 2차 웹증거 보완 (좌표·전화), 1차 시드 아님 | CSV에 좌표·전화 없으면 카카오로 보완 |
| **KAHF 포지션** | 롱리스트 시드 → 숏리스트 trust tag | KAHF 인증 클리닉 수가 적어 시드로는 부족. 신뢰 점수로만 활용 |
| **KHIDI 오버레이** | 기관명 퍼지 매칭으로 tagging | 외국인 유치 등록 = 핵심 신뢰 지표. 배지로 노출 |
| **롱리스트 → 숏리스트** | 6-gate 자동 필터 + 100점 스코어링. 주 1회 재계산 | BEAUTY_PAGE_DESIGN §5-4 기준 적용 |
| **이미지 저장소** | Phase 1: Supabase Storage (1GB 무료 이내) | Phase 1 규모(500장 이내) Supabase 충분 |
| **OCR 비용 상한** | 월 $50 한도 (약 6,000장). 초과 시 배치 중단 | GPT-4 Vision $0.008/장 × 6,000 = $48 |
| **공급자 자가등록 시점** | 런칭 후 2개월 (Phase 1-B) | Phase 1 시드 품질 확인 후 외부 개방 |
| **데이터 갱신 주기** | 이벤트: 매일 새벽 3시. 기본정보: 주 1회. trending_score: 주 1회 | 이벤트 유효기간 짧음. 기본정보 안정적 |
| **외국인 동일가 확인** | 클리닉 자기신고 + 유저 신고 시 제외 | 100% 사전 확인 불가. 사후 제재 구조 |
| **doctors_raw.xls.xlsx** | data/seed/ 보관. 롱리스트 시드 아님. 의사 정보 보강용 참고 | 병원명·인스타 비어있음, 전화 16%만. 보완용 한계 |

---

## 11. raw_clinics 테이블 (전체 스키마)

행정안전부 CSV + 카카오 보완 + KHIDI/KAHF 오버레이 + 수집 메타를 하나의 테이블에서 관리.

```sql
CREATE TABLE raw_clinics (
  id                              uuid DEFAULT gen_random_uuid() PRIMARY KEY,

  -- ── 행정안전부 CSV 원본 ────────────────────────────────────────────
  seed_id                         text UNIQUE,          -- 개설허가번호 (행안부 고유 ID)
  source_dataset                  text,                 -- 'MOHW_CLINIC_2024'
  institution_name_raw            text NOT NULL,        -- CSV 원문 기관명
  institution_type_raw            text,                 -- CSV 원문 기관종별 코드
  business_status_raw             text,                 -- '정상' | '폐업' | '정지'
  permit_date                     date,                 -- 개설 인허가일
  full_address_raw                text,                 -- 지번 주소 원문
  road_address_raw                text,                 -- 도로명 주소 원문
  specialty_codes_raw             text[] DEFAULT '{}',  -- 진료과목 코드 배열 ['05','08']
  is_active_candidate             bool DEFAULT false,   -- K-Beauty 필터 통과 여부
  load_batch_date                 date,                 -- CSV 적재 날짜

  -- ── 카카오 API 보완 ───────────────────────────────────────────────
  kakao_place_id                  text,
  kakao_category_group            text,
  kakao_road_address              text,
  raw_lat                         numeric(9,6),
  raw_lng                         numeric(9,6),
  raw_phone                       text,
  raw_homepage_url                text,
  raw_instagram                   text,

  -- ── 수집 메타 ─────────────────────────────────────────────────────
  source_type                     text NOT NULL DEFAULT 'public_data',
    -- 'public_data' | 'kakao_api' | 'admin_manual' | 'supplier_self'
  capture_method                  text NOT NULL DEFAULT 'csv_batch',
    -- 'csv_batch' | 'api_batch' | 'web_scrape' | 'manual_entry'
  confidence_level                text DEFAULT 'medium',
    -- 'low' | 'medium' | 'high' | 'verified'
  evidence_url                    text,

  -- ── 웹 증거 수집 결과 ─────────────────────────────────────────────
  naver_rating                    numeric(3,2),
  naver_review_count              int,
  has_japanese_page               bool,
  has_line_id                     bool,
  instagram_post_count            int,

  -- ── KHIDI/KAHF 오버레이 ──────────────────────────────────────────
  foreign_patient_registered      bool DEFAULT false,   -- KHIDI 등록 여부
  kahf_certified                  bool DEFAULT false,   -- KAHF 인증 여부
  kahf_valid_until                date,                 -- KAHF 인증 유효기간
  supported_languages             text[] DEFAULT '{}',  -- 지원 언어 코드 ['ja','en']
  interpreter_available           bool DEFAULT false,   -- 통역사 보유 여부
  intl_coordinator_available      bool DEFAULT false,   -- 국제 코디네이터 유무
  intl_service_evidence_url       text,                 -- 외국인 서비스 증거 URL

  -- ── 처리 상태 ─────────────────────────────────────────────────────
  status                          text DEFAULT 'pending',
    -- 'pending' | 'enriched' | 'approved' | 'rejected' | 'duplicate'
  approved_clinic_id              uuid REFERENCES clinics(id),
  reviewer_note                   text,
  rejection_reason                text,

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

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v0.1 | 2026-04-20 | 초안 — 6단계 수집 파이프라인, OCR 설계, 법적 리스크, Phase별 실행 계획 |
| v0.2 | 2026-04-20 | 롱리스트/숏리스트 분리, 6-gate 필터, is_shortlisted/shortlist_score 컬럼, 모든 미결 결정 확정 |
| v0.3 | 2026-04-21 | **대폭 개정**: 행정안전부_건강_의원 CSV를 1차 롱리스트 시드로 채택. 4단계 파이프라인 재설계. KAHF 포지션 변경 (롱리스트 시드 → 숏리스트 trust tag). KHIDI 외국인환자 오버레이 추가. raw_clinics 테이블 신규 컬럼 (seed_id, source_dataset, institution_name_raw, institution_type_raw, business_status_raw, permit_date, full_address_raw, road_address_raw, is_active_candidate, load_batch_date, foreign_patient_registered, kahf_certified, kahf_valid_until, supported_languages, interpreter_available, intl_coordinator_available, intl_service_evidence_url). 롱리스트 규모 수정 (200-300 → 1차 수천/2차 수백/숏리스트 40-60). |

*담당: Claude Desktop (기획)*
