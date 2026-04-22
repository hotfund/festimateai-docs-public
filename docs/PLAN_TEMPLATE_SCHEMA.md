# 여행 플랜 템플릿 스키마 (v0.1 초안)

> **작성**: 2026-04-14
> **상태**: 초안 (사용자 그림 업로드 후 통합 예정)
> **용도**: AI Guide 출력 + Mate 제안서 공통 포맷

---

## 1. 설계 원칙

1. **AI 결과 = Mate 제안서** 구조 통일 — 게스트가 나란히 비교·병합 가능
2. **공급자 상품 DB와 직접 링크** — 아이템마다 `product_id` 연결
3. **3가지 뷰 동일 데이터로 렌더링**
   - 카드 타임라인
   - 지도 오버레이 (카테고리별 컬러 마커)
   - 비교 모드 (2개 플랜 병치)
4. **예약 가능 / 정보만 구분** — `bookable: true/false`로 표시
5. **다국어 필드** — `_kr / _jp / _en` 접미사

---

## 2. JSON 스키마 (v0.1)

```jsonc
{
  // ── 메타 정보 ────────────────────────────────
  "plan_id": "plan_20260414_0001",
  "author": {
    "type": "ai | mate | hybrid",
    "mate_pair_id": "pair_xxx",          // 메이트 제안 시
    "ai_version": "festimate-guide-v2"
  },
  "status": "draft | shared | booked | completed",
  "created_at": "2026-04-14T10:00:00Z",
  "updated_at": "2026-04-14T12:00:00Z",

  // ── 요약 메타 ────────────────────────────────
  "meta": {
    "title_kr": "3박4일 청주·서울 봄 여행",
    "title_jp": "3泊4日の桜と美容の韓国旅",
    "title_en": "4-Day Cherry Blossom & Beauty Trip",
    "cover_image": "https://cdn.festimate.ai/covers/xxx.jpg",
    "tags": ["spring", "beauty", "masters", "2-female"],
    "target": {
      "persona": "japanese_female_20s",
      "group_size": 2,
      "budget_per_person_krw": 600000
    },
    "dates": {
      "from": "2026-04-10",
      "to": "2026-04-13"
    }
  },

  // ── 요약 정보 (한눈에 보기용) ─────────────────
  "summary": {
    "duration_days": 4,
    "cities": ["청주", "서울"],
    "total_cost_krw": 1200000,
    "total_cost_jpy": 135000,
    "highlights": [
      "이용강 명장 도자기 체험",
      "진주 유등축제",
      "강남 보톡스"
    ]
  },

  // ── Day별 일정 ──────────────────────────────
  "days": [
    {
      "day_number": 1,
      "date": "2026-04-10",
      "city": "청주",
      "weather_hint": "맑음 / 18°C",
      "items": [
        {
          "item_id": "item_001",
          "time_start": "10:00",
          "duration_min": 60,
          "category": "transit",    // 공통 enum - 아래 목록 참조
          "product_id": null,
          "title_kr": "인천공항 → 청주",
          "title_jp": "仁川空港 → 清州",
          "title_en": "ICN → Cheongju",
          "cost_krw": 2800,
          "location": {
            "from": { "lat": 37.4692, "lng": 126.4505, "name": "ICN" },
            "to":   { "lat": 36.6397, "lng": 127.4892, "name": "청주역" }
          },
          "map_color": "#6B7280",
          "bookable": false,
          "external_info": {
            "source": "korail",
            "link": "https://www.korail.com/..."
          }
        },
        {
          "item_id": "item_002",
          "time_start": "14:00",
          "duration_min": 240,
          "category": "master",
          "product_id": "master_yi_yonggang",
          "title_kr": "이용강 명장 도자기 공방 체험",
          "title_jp": "李龍江名匠 陶磁器工房",
          "title_en": "Master Yi Yonggang Ceramics Workshop",
          "description_jp": "日本で博物館学を修めた陶磁器名匠...",
          "cost_krw": 180000,
          "location": {
            "lat": 36.6397,
            "lng": 127.4892,
            "address_kr": "충북 청주시...",
            "address_jp": "忠清北道 清州市..."
          },
          "map_color": "#8B5CF6",
          "bookable": true,
          "booking_url": "/jp/master/yi-yonggang",
          "note_jp": "日本語での講義が可能",
          "recovery_impact": null
        },
        {
          "item_id": "item_003",
          "time_start": "19:00",
          "duration_min": 90,
          "category": "food",
          "product_id": null,
          "title_kr": "청주 맛집 - 서문시장 OO",
          "title_jp": "清州グルメ",
          "cost_krw": 35000,
          "location": {
            "lat": 36.636,
            "lng": 127.47,
            "name": "서문시장"
          },
          "map_color": "#F59E0B",
          "bookable": false,
          "external_info": {
            "source": "naver",
            "link": "https://map.naver.com/..."
          }
        }
      ]
    }
    // Day 2, 3, 4 ...
  ],

  // ── 제약·주의사항 ────────────────────────────
  "constraints_applied": {
    "beauty_recovery_windows": [
      {
        "procedure": "보톡스",
        "scheduled_item_id": "item_015",
        "scheduled_day": 4,
        "reason": "귀국일 직전 배치 — 비행기에서 마스크로 붓기 가림"
      }
    ],
    "transit_buffer_min": 30,
    "mate_session_count": 1,
    "meal_coverage": "3 lunches / 3 dinners"
  },

  // ── 메이트 제안 전용 필드 (optional) ───────────
  "mate_proposal_extras": {
    "proposer_pair_id": "pair_xxx",
    "proposer_names": ["김하나", "박지현"],
    "message_kr": "안녕하세요! 이용강 명장님 일본어 강의 가능해서 저희가 통역 필요 없어요. 벚꽃 포토스팟도 저희가 알려드릴게요 :)",
    "message_jp": "こんにちは！...",
    "expires_at": "2026-04-20T23:59:59Z",
    "available_dates": ["2026-04-10", "2026-04-11", "2026-04-12"]
  },

  // ── 게스트 커스터마이즈 기록 (협업용) ─────────
  "customizations": [
    {
      "timestamp": "2026-04-14T11:23:00Z",
      "actor": "guest_user_123",
      "action": "removed",
      "item_id": "item_007",
      "reason": "관심 없음"
    },
    {
      "timestamp": "2026-04-14T11:25:00Z",
      "actor": "guest_user_123",
      "action": "added",
      "item_id": "item_020",
      "note": "홍대 카페 추가"
    }
  ]
}
```

---

## 3. 카테고리 Enum & 컬러 팔레트

| 카테고리 코드 | 의미 | 지도 마커 컬러 |
|---|---|---|
| `festival` | 🎪 축제 | `#EF4444` 빨강 |
| `beauty` | 💄 K-뷰티 시술 | `#3B82F6` 파랑 |
| `master` | 🏆 명인명장 (체험·쇼핑) | `#8B5CF6` 보라 |
| `garden` | 🌿 정원·박물관 | `#10B981` 녹색 |
| `food` | 🍜 식당·맛집 | `#F59E0B` 주황 |
| `sleep` | 🛏 숙소 | `#06B6D4` 청록 |
| `shopping` | 🛍 쇼핑 | `#EC4899` 분홍 |
| `mate_session` | 🤝 메이트 동행 | `#EAB308` 황금 |
| `transit` | ✈ 이동 (항공·KTX 등) | `#6B7280` 회색 |
| `templestay` | 🏯 템플스테이 | `#14B8A6` 터콰이즈 |
| `custom` | 🧩 사용자 추가 | `#64748B` 슬레이트 |

---

## 4. UI 렌더링 뷰

### 뷰 1: 카드 타임라인
```
🌸 3泊4日 桜と美容の韓国旅
   청주 → 서울 · 4일 · ¥135,000 · 20代女性2名
   ────────────────────────────────────────
   [Day 1] 04.10 (금) · 청주
    ✈ 10:00  ICN → 청주 (60分, ¥2,800)
    🏆 14:00  이용강 명장 도자기 (4h, ¥18,000) [예약]
    🍜 19:00  청주 맛집 (1.5h, ¥1,500)
    🛏 21:00  숙소 체크인

   [Day 2] 04.11 (토) · 청주
    ...

   [지도 보기] [일괄 예약] [저장] [공유]
```

### 뷰 2: 지도 오버레이 (AI Guide 메인)
```
┌─────────────────────────────────────┐
│  [지도 전체]                         │
│                                     │
│  🔴축제  🔵뷰티  🟣명인  🟠식당  ...  │
│                                     │
│  마커 클릭 → 아이템 상세 팝업         │
│                                     │
│  Day 필터: [All] [Day1] [Day2] ...  │
└─────────────────────────────────────┘
```

### 뷰 3: 비교 모드 (AI vs Mate 제안)
```
┌─────────────────┐ ┌─────────────────┐
│ [AI 플랜 A]      │ │ [메이트 제안 B]  │
│ Day 1           │ │ Day 1           │
│ Day 2           │ │ Day 2           │
│ 총 ¥135,000     │ │ 총 ¥128,000     │
└─────────────────┘ └─────────────────┘
   [A 선택]  [B 선택]  [A+B 혼합]
```

---

## 5. Tourist 결과물 2종 — 관계 구조

Tourist가 만들 수 있는 최종 결과물은 **독립적인 2개**다.

```
AI Planner 입력
      │
      ▼
┌─────────────────────────────────┐
│  Output 1: TravelPlan (일정표)   │  ← Mate 없어도 완결
│  Day-by-Day 완성 일정            │  ← [저장] 버튼으로 생성
│  저장 / URL공유 / 인쇄           │
└─────────────────────────────────┘
      │
      │ (선택적 연결)
      ▼
┌─────────────────────────────────────────────────────────┐
│  Output 2: MateRequest (Looking for Mate 요청서)         │
│                                                         │
│  Type A 확정형 (Confirmed)    Type B 오픈형 (Open)       │
│  "플랜 다 짰어, 동반 Mate 찾아"  "일부만 정했어, 같이 만들자"│
│  → TravelPlan 첨부 (plan_id)   → 확정 항목 + 오픈 항목   │
│  → Mate는 가이드·동반 역할       → Mate는 공동기획+가이드  │
└─────────────────────────────────────────────────────────┘
```

**핵심 원칙:**
- TravelPlan은 MateRequest 없이 존재 가능 (독립 가치)
- MateRequest는 TravelPlan 없이도 생성 가능 (오픈형의 경우)
- 확정형 MateRequest는 TravelPlan을 참조 (plan_id FK)

---

## 5-1. MateRequest JSON 스키마

### Type A — 확정형 (Confirmed)

"플랜 완성됨, Mate는 동반 가이드로"

```jsonc
{
  "request_id": "req_20260425_001",
  "request_type": "confirmed",           // "confirmed" | "open"

  // ── Tourist 정보 ──
  "tourist_id": "user_xxx",
  "plan_id": "plan_20260425_001",        // TravelPlan 참조 (필수)

  // ── 기본 조건 ──
  "dates": {
    "from": "2026-04-25",
    "to": "2026-04-27"
  },
  "group": {
    "female_confirmed": 2,
    "male_confirmed": 0,
    "female_tentative": 0,
    "male_tentative": 1                  // 올 수도 있는 인원
  },
  "region": ["서울 강남", "홍대"],
  "pair_format_wanted": "FF",            // FF | MF | GG | BB | solo | any
  "budget_per_session_jpy": {
    "min": 8000,
    "max": 15000
  },

  // ── 오픈 항목 (확정형이라도 소수 있을 수 있음) ──
  "open_items": [
    { "category": "food", "note": "저녁 맛집 하나 추천해줘" }
  ],

  // ── Tourist 소개 메시지 ──
  "message_jp": "はじめまして！プランは決まっていますが、一緒に回ってくれるメイトを探しています。ボトックス施術後なので、ゆっくりめのペースでお願いします🙏",

  // ── 상태 ──
  "status": "published",                 // draft | published | matched | expired | closed
  "expires_at": "2026-04-20T23:59:59Z", // 여행 7일 전 자동 만료
  "created_at": "2026-04-15T10:00:00Z",
  "updated_at": "2026-04-15T10:00:00Z"
}
```

### Type B — 오픈형 (Open)

"일부만 정했어, 나머지는 Mate와 같이 만들자"

```jsonc
{
  "request_id": "req_20260425_002",
  "request_type": "open",

  // ── Tourist 정보 ──
  "tourist_id": "user_xxx",
  "plan_id": null,                       // 없거나 부분 draft

  // ── 기본 조건 (확정) ──
  "dates": {
    "from": "2026-04-25",
    "to": "2026-04-27"
  },
  "group": {
    "female_confirmed": 2,
    "male_confirmed": 0,
    "female_tentative": 0,
    "male_tentative": 0
  },
  "region": ["서울"],                    // 넓게 지정 가능
  "pair_format_wanted": "FF",
  "budget_per_person_jpy": {
    "min": 80000,
    "max": 150000
  },
  "interests": ["K-뷰티", "카페투어", "쇼핑"],  // 관심 카테고리

  // ── 확정 항목 (Wishlist에서 가져온 것) ──
  "confirmed_items": [
    {
      "item_id": "ci_001",
      "category": "beauty",
      "venue_name_jp": "カロスキル パーソナルカラー診断",
      "date": "2026-04-25",              // 날짜 확정된 경우
      "note_jp": "予約済み、午前10時",
      "tag": "confirmed"                 // ✅확정
    },
    {
      "item_id": "ci_002",
      "category": "festival",
      "venue_name_jp": "ソウル桜祭り",
      "date": null,                      // 날짜 미정
      "note_jp": "行きたい！日程はメイトと相談",
      "tag": "suggest"                   // ✨추천
    }
  ],

  // ── 오픈 항목 (Mate와 같이 결정할 것) ──
  "open_items": [
    { "category": "food",     "note_jp": "韓国グルメを案内してほしい" },
    { "category": "beauty",   "note_jp": "おすすめのコスメショップ教えて" },
    { "category": "transit",  "note_jp": "移動ルートどうすればいい?" },
    { "category": "custom",   "note_jp": "その他おすすめあれば！" }
  ],

  // ── 공동 기획 플래그 ──
  "co_planning": true,                   // Mate가 일정 제안 권한 가짐

  // ── Tourist 소개 메시지 ──
  "message_jp": "초등학교때부터 좋아한 한국에 처음 가요！퍼스널컬러는 예약했는데 나머지는 같이 정하고 싶어요. 맛집이나 카페 잘 아는 메이트 환영해요☕",

  // ── 상태 ──
  "status": "published",
  "expires_at": "2026-04-20T23:59:59Z",
  "created_at": "2026-04-15T09:00:00Z",
  "updated_at": "2026-04-15T09:00:00Z"
}
```

---

## 5-2. MateRequest 생명주기

```
draft      → Tourist가 작성 중 (비공개)
   ↓
published  → Matching 페이지에 공개됨
   ↓
matched    → Mate와 상호 ❤️ 성사 (Tier 2 언락)
   ↓
booked     → 예약 확정 (Tier 3 언락)
   ↓
completed  → 여행 종료 → 리뷰 요청
   ↓
expired    → 자동 만료 (여행 7일 전 미매칭 시)
   ↓
closed     → Tourist가 직접 닫음
```

**TravelPlan 생명주기 (기존):**

```
draft → saved → in_progress → completed
```

두 객체는 독립적으로 상태 관리. TravelPlan이 `completed`여도 MateRequest는 `expired`일 수 있음 (Mate 없이 혼자 여행한 경우).

---

## 5-3. UI 진입 흐름

```
AI Planner 결과 화면
   ├── [💾 저장]          → TravelPlan 저장 (Output 1)
   │                         status: saved
   │
   └── [🤝 Looking for Mate] → MateRequest 생성 모달
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
           확정형으로 올리기             오픈형으로 올리기
           "플랜대로 가이드해줘"         "같이 만들어요"
           (plan_id 자동 첨부)         (확정·오픈 항목 선택)
```

**AI Planner 없이 직접 MateRequest 생성:**
→ `/ai/matching` 진입 시 하단 "나도 올리기" 버튼
→ 오픈형으로만 생성 (플랜 없으니까)

---

## 6. 플랜 생명주기 (구 §5)

```
draft        → AI가 생성 or 메이트가 작성 중
   ↓
shared       → 게스트에게 공유됨 (URL 생성)
   ↓
booked       → 게스트가 전체 or 일부 예약
   ↓
in_progress  → 여행 진행 중
   ↓
completed    → 여행 종료, 리뷰 가능
```

---

## 6. DB 스키마 매핑 (Supabase)

```sql
travel_plans (
  id uuid primary key,
  author_type enum('ai', 'mate', 'hybrid'),
  author_mate_pair_id uuid,
  ai_version text,
  status enum('draft', 'shared', 'booked', 'in_progress', 'completed'),
  meta jsonb,          -- title, tags, target, dates
  summary jsonb,       -- duration, cities, cost, highlights
  days jsonb,          -- day-by-day items
  constraints jsonb,
  mate_extras jsonb,
  created_at timestamptz,
  updated_at timestamptz
)

plan_items (
  id uuid primary key,
  plan_id uuid references travel_plans,
  day_number int,
  order_in_day int,
  item_data jsonb,     -- 전체 item JSON
  product_id uuid,     -- FK to products (if bookable)
  created_at timestamptz
)

plan_customizations (
  id uuid primary key,
  plan_id uuid,
  actor_id uuid,
  timestamp timestamptz,
  action enum('added', 'removed', 'edited', 'reordered'),
  item_id uuid,
  details jsonb
)

-- ── MateRequest (Output 2) ──────────────────────────────
mate_requests (
  id uuid primary key,
  tourist_id uuid references users(id) not null,

  -- 타입 분기
  request_type text check (request_type in ('confirmed', 'open')) not null,

  -- TravelPlan 참조 (confirmed형 필수, open형 optional)
  plan_id uuid references travel_plans(id),

  -- 기본 조건
  dates jsonb not null,             -- { from, to }
  group_composition jsonb not null, -- { female_confirmed, male_confirmed, ... }
  region text[] not null,
  pair_format_wanted text,          -- FF|MF|GG|BB|solo|any
  budget_jpy jsonb,                 -- { min, max }
  interests text[],                 -- 오픈형 전용

  -- 항목 (오픈형 전용)
  confirmed_items jsonb,            -- [{ item_id, category, venue_name_jp, date, note_jp, tag }]
  open_items jsonb,                 -- [{ category, note_jp }]
  co_planning boolean default false,

  -- 메시지
  message_jp text,

  -- 상태
  status text check (status in (
    'draft', 'published', 'matched', 'booked', 'completed', 'expired', 'closed'
  )) default 'draft',
  expires_at timestamptz,
  matched_mate_pair_id uuid,        -- 매칭된 Mate pair

  created_at timestamptz default now(),
  updated_at timestamptz default now()
)

-- ── MateOffering ────────────────────────────────────────
mate_offerings (
  id uuid primary key,
  mate_pair_id uuid not null,
  level int check (level in (1, 2, 3)) default 1,

  -- Level 1 필드
  title_jp text not null,
  available_dates jsonb not null,       -- { type, from, to, specific_dates, days_of_week }
  pair_format text not null,
  price jsonb not null,                 -- { amount, currency, unit, session_hours, includes_note_jp }
  region text[] not null,
  intro_jp text,

  -- Level 2 필드
  recommended_venues jsonb,            -- [{name_jp, category, note_jp, bookable, product_id}]
  rough_flow_jp text,

  -- Level 3 필드 (TravelPlan.days와 동일 구조)
  days jsonb,

  -- Mate 메타
  specialties text[],
  language_level text,

  -- 상태·정책
  status text check (status in (
    'draft', 'published', 'matched', 'expired', 'closed'
  )) default 'draft',
  expires_at timestamptz,
  concurrent_index int default 1,
  response_to_request_id uuid references mate_requests(id),

  created_at timestamptz default now(),
  updated_at timestamptz default now()
)

-- ── 매칭 관심 표시 ────────────────────────────────────────
mate_request_interests (
  id uuid primary key,
  request_id uuid references mate_requests(id),
  mate_pair_id uuid,                -- Mate 쪽 관심 표시
  tourist_id uuid,                  -- Tourist 쪽 관심 표시
  direction text check (direction in ('tourist_to_mate', 'mate_to_tourist')),
  status text check (status in ('pending', 'mutual', 'declined')),
  created_at timestamptz default now()
)
```

---

## 7. AI Guide 프롬프트 지시 (AI가 이 스키마로 출력하도록)

```
너는 FestiMate AI Guide다. 사용자 입력(여행 기간·도시·관심사·예산·게스트 구성)
을 받아 다음 JSON 스키마에 맞춰 여행 플랜을 생성한다.

【필수 규칙】
1. 모든 아이템은 category enum 중 하나
2. bookable 상품은 실제 DB의 product_id 사용
3. 다국어 필드 3개 언어 모두 채움 (_kr/_jp/_en)
4. 뷰티 시술 포함 시 recovery_impact·scheduled_day 명시
5. transit_buffer_min 30분 이상
6. 하루 2~4 아이템 (과잉 금지)
7. 귀국일엔 transit만 남기고 시술·무거운 활동 금지
8. 메이트 세션은 12~18시 범위 내에서 제안

【출력】 JSON only, 설명 없이.
```

---

## 8. 미결 사항

1. `product_id` 참조 시 바로 가격·재고 동기화 vs 스냅샷 저장?
2. 커스터마이즈 이력 저장 기간 (영구? 90일?)
3. 공유 URL 만료 정책
4. 메이트 제안서 내 "`available_dates`" vs 상품 자체 캘린더 우선순위
5. Day 0 (이동일) / Day N+1 (귀국일) 표기법
6. 다중 국가(한→일 KoJap) 통합 플랜 시 URL 구조

---

## 5-4. MateOffering JSON 스키마

Mate가 올리는 "Looking for Tourist" 투어 제안서. Level 1/2/3 누적 구조.

```jsonc
{
  "offering_id": "off_20260425_001",
  "offering_type": "looking_for_tourist",
  "level": 2,                            // 1 | 2 | 3

  // Mate 정보 (Tier 1 공개: pair_id, level, specialty / Tier 2: 실명)
  "mate_pair_id": "pair_xxx",
  "mate_level": "Verified",              // Rookie|Verified|Featured|ProGuide
  "specialties": ["K-뷰티", "카페투어", "쇼핑"],
  "language_level": "N1",               // N1|N2|intermediate|basic

  // ── Level 1 (필수 5필드) ──────────────────────────────
  "title_jp": "강남 K-뷰티 + 카페 투어 ✨ 日本語N1",
  "available_dates": {
    "type": "range",                     // "range" | "specific"
    "from": "2026-04-20",
    "to": "2026-05-15",
    "specific_dates": [],                // type="specific"일 때
    "days_of_week": [6, 0]              // 0=일요일, 6=토요일 (주말만)
  },
  "pair_format": "FF",                  // FF|MF|GG|BB|solo|any
  "price": {
    "amount": 8000,
    "currency": "JPY",
    "unit": "session",                  // "session" | "hour"
    "session_hours": 6,                 // session일 때 포함 시간
    "includes_note_jp": "交通費・入場料は別途"
  },
  "region": ["서울 강남", "홍대", "압구정"],
  "intro_jp": "일본어과 4학년. 압구정·가로수길 단골 많아요 ✨ K-뷰티 시술 후 회복 코스 전문",

  // ── Level 2 (선택 — 추천 장소 + 대략 흐름) ───────────
  "recommended_venues": [
    {
      "name_jp": "カロスキル パーソナルカラー診断",
      "category": "beauty",
      "note_jp": "사전 예약 필요, 도와드릴게요",
      "bookable": true,
      "product_id": "prod_xxx"          // DB 상품 연결 (있으면)
    },
    {
      "name_jp": "圧鴎亭 コスメ通り",
      "category": "shopping",
      "note_jp": "Mate 단골 매장, 할인 가능",
      "bookable": false,
      "product_id": null
    }
  ],
  "rough_flow_jp": "오전 강남 시술 → 점심 현지 맛집 → 오후 쇼핑·카페 → 저녁 홍대",

  // ── Level 3 (선택 — Day-by-Day, TravelPlan.days와 동일 구조) ──
  "days": [
    {
      "day_number": 1,
      "items": [
        {
          "item_id": "oi_001",
          "time_start": "13:00",
          "duration_min": 30,
          "category": "transit",
          "title_jp": "강남역 11번 출구 집합",
          "location": { "name": "강남역 11번 출구", "lat": 37.498, "lng": 127.027 },
          "cost_jpy": 0,
          "item_tag": "confirmed",      // ✅확정 | 💬협의 | ✨추천 | ❓미정
          "note_jp": "ここで会いましょう！"
        },
        {
          "item_id": "oi_002",
          "time_start": "14:00",
          "duration_min": 120,
          "category": "beauty",
          "title_jp": "파스텔 퍼스널컬러 진단",
          "location": { "name": "가로수길역 근처", "lat": 37.519, "lng": 127.020 },
          "cost_jpy": 2000,
          "item_tag": "suggest",
          "note_jp": "예약 필요하면 도와드려요"
        },
        {
          "item_id": "oi_003",
          "time_start": "17:00",
          "duration_min": 90,
          "category": "shopping",
          "title_jp": "압구정 로데오 쇼핑",
          "item_tag": "discuss",
          "note_jp": "어떤 걸 찾으시나요? 같이 정해요"
        }
      ]
    }
  ],

  // ── 상태·정책 ──────────────────────────────────────────
  "status": "published",               // draft|published|matched|expired|closed
  "expires_at": "2026-05-31T23:59:59Z",
  "concurrent_index": 1,              // 이 Mate 활성 Offering 중 몇 번째 (tier 한도 내)
  "response_to_request_id": null,     // 특정 MateRequest에 응답이면 채움

  "created_at": "2026-04-15T10:00:00Z",
  "updated_at": "2026-04-15T10:00:00Z"
}
```

**Item Tag 의미 (Level 3 전용):**

| 태그 | 아이콘 | 의미 | Mate 수정 가능? |
|------|--------|------|----------------|
| `confirmed` | ✅ | 이 시간·장소 고정 | 불가 |
| `discuss` | 💬 | Tourist와 협의할 항목 | 매칭 후 채팅에서 |
| `suggest` | ✨ | 추천이지만 변경 OK | Tourist 요청 시 |
| `empty` | ❓ | 미정, 나중에 채움 | 매칭 후 같이 |

---

## 5-5. 결과물 3종 관계도

```
Tourist                          Mate
───────                          ────
TravelPlan (Output 1)            MateOffering
  └─ Day-by-Day 완성본            └─ Level 1: 조건만
  └─ 독립 저장·공유               └─ Level 2: + 추천 장소
                                  └─ Level 3: + Day-by-Day (동일 구조)
MateRequest (Output 2)                         │
  └─ confirmed형: TravelPlan 첨부 ←────────────┘ 비교 가능
  └─ open형: 조건 + 확정·오픈 항목

매칭 성사 →  MateRequest ❤️ MateOffering
         →  채팅으로 일정 조율
         →  최종: 합의된 TravelPlan 1개 완성
```

---

## 6. DB 스키마 (offerings 테이블 추가)

```sql
-- ── TravelPlan ───────────────────────────────────────────
travel_plans (
  id uuid primary key,
  tourist_id uuid references users(id),
  author_type text check (author_type in ('ai', 'mate', 'hybrid')),
  status text check (status in ('draft', 'saved', 'in_progress', 'completed')) default 'draft',
  meta jsonb,          -- title, tags, target, dates
  summary jsonb,       -- duration, cities, cost, highlights
  days jsonb,          -- day-by-day items
  constraints jsonb,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
)

plan_items (
  id uuid primary key,
  plan_id uuid references travel_plans(id),
  day_number int,
  order_in_day int,
  item_data jsonb,
  product_id uuid,
  created_at timestamptz default now()
)

-- ── MateRequest ──────────────────────────────────────────
mate_requests (
  id uuid primary key,
  tourist_id uuid references users(id) not null,
  request_type text check (request_type in ('confirmed', 'open')) not null,
  plan_id uuid references travel_plans(id),
  dates jsonb not null,
  group_composition jsonb not null,
  region text[] not null,
  pair_format_wanted text,
  budget_jpy jsonb,
  interests text[],
  confirmed_items jsonb,
  open_items jsonb,
  co_planning boolean default false,
  message_jp text,
  status text check (status in (
    'draft', 'published', 'matched', 'booked', 'completed', 'expired', 'closed'
  )) default 'draft',
  expires_at timestamptz,
  matched_mate_pair_id uuid,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
)

-- ── MateOffering ─────────────────────────────────────────
mate_offerings (
  id uuid primary key,
  mate_pair_id uuid not null,
  level int check (level in (1, 2, 3)) default 1,
  title_jp text not null,
  available_dates jsonb not null,
  pair_format text not null,
  price jsonb not null,
  region text[] not null,
  intro_jp text,
  recommended_venues jsonb,
  rough_flow_jp text,
  days jsonb,
  specialties text[],
  language_level text,
  status text check (status in (
    'draft', 'published', 'matched', 'expired', 'closed'
  )) default 'draft',
  expires_at timestamptz,
  concurrent_index int default 1,
  response_to_request_id uuid references mate_requests(id),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
)

-- ── 매칭 관심 표시 ────────────────────────────────────────
mate_request_interests (
  id uuid primary key,
  request_id uuid references mate_requests(id),
  offering_id uuid references mate_offerings(id),
  direction text check (direction in ('tourist_to_mate', 'mate_to_tourist')),
  status text check (status in ('pending', 'mutual', 'declined')) default 'pending',
  created_at timestamptz default now()
)
```

---

## 7. AI Guide 프롬프트 지시

```
너는 FestiMate AI Guide다. 사용자 입력(여행 기간·도시·관심사·예산·게스트 구성)을
받아 TravelPlan JSON 스키마에 맞춰 여행 플랜을 생성한다.

【필수 규칙】
1. 모든 아이템은 category enum 중 하나
2. bookable 상품은 실제 DB의 product_id 사용
3. 다국어 필드 _kr/_jp/_en 모두 채움
4. 뷰티 시술 포함 시 recovery_impact·scheduled_day 명시
5. transit_buffer_min 30분 이상
6. 하루 2~4 아이템 (과잉 금지)
7. 귀국일엔 transit만 남기고 시술·무거운 활동 금지
8. 메이트 세션은 12~18시 범위 내에서 제안

【출력】 JSON only, 설명 없이.
```

---

## 8. 미결 사항

1. `product_id` 참조 시 실시간 가격·재고 동기화 vs 스냅샷 저장?
2. 공유 URL 만료 정책 (TravelPlan: 90일? MateRequest: 여행 7일 전 자동?)
3. MateOffering `available_dates` vs 상품 자체 캘린더 우선순위
4. Day 0 (이동일) / Day N+1 (귀국일) 표기법
5. 채팅에서 합의된 최종 플랜 — TravelPlan 새 버전으로 저장? 아니면 별도 `agreed_plan`?
6. MateRequest 오픈형에서 Mate가 제안한 내용 — MateOffering으로 자동 생성? 채팅 첨부?

---

## 변경 이력

| 날짜 | 버전 | 내용 |
|------|------|------|
| 2026-04-14 | v0.1 | 초안 — TravelPlan JSON + DB 스키마 |
| 2026-04-20 | v0.2 | Tourist 결과물 2종 — TravelPlan + MateRequest (confirmed/open 타입) |
| 2026-04-20 | v0.3 | MateOffering JSON 스키마 (Level 1/2/3) + offerings DB 테이블 + 결과물 3종 관계도 |

*버전: v0.3 / 마지막 업데이트: 2026-04-20*
