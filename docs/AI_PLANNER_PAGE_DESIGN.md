# AI Planner 페이지 설계 (AI Planner Page Design)

> **작성**: 2026-04-19 (v0.1) → **v1.0 승격**: 2026-04-22
> **위상**: 페이지 설계 SSOT — `/ai/plan` (현재 `/ai-guide`) 구현 기준
> **선행 문서**: `AI_TERMINOLOGY.md` §2~§5, `AI_ROLE_DEFINITION.md` §3, `AI_COST_MODEL.md` §5, `SEARCH_PAGE_DESIGN.md`, `ADMIN_CONFIGURABLE_PARAMS.md` §3-3·§3-4
> **현행 구현**: `app/src/app/ai-guide/page.tsx` (Tourist 모드, 9필드 폼, `gpt-4o` 하드코딩)
> **설계 목표**: 일본 20~30 여성 Tourist + 한국 Local Mate **양쪽 공용 canvas**, 3-Layer 구조 (Entry / Intent / Input)

---

## 🆕 v1.0 통합 구조 (Genspark 조언자 리뷰 반영)

v0.4까지 누적된 6개 충돌을 **레이어 분리·용어 정리**로 해소:

| 변경 | Before (v0.4) | After (v1.0) |
|------|---------------|--------------|
| 진입 순서 | §4 vs §21 충돌 | **3-Layer 분리** (Entry §9 / Intent §21 / Input §4·§5) |
| Level 용어 | §16·§17 이중 사용 | §16 → **`Mate Tier`** / §17 → **`Plan Depth`** |
| Mate 필드 × Depth | 관계 모호 | **직교 2축** (5필드 필수 + Plan Depth 독립) |
| enum 구체 | 정의 없음 | `entrySource`·`intent` enum 확정 |
| 샘플 전략 | 3곳 혼용 | **3분리** (대표 샘플 / Intent 카드 / 갤러리) |
| Mate 등급 | 3단계 or 5단계 혼용 | **Internal 5 / Display 3 + ⭐ 배지** |

---

## 1. 핵심 원칙

### 1-1. 양쪽 공용 Canvas (DRY 정점)
- **같은 URL · 같은 UI 구조 · 같은 AI**
- 역할 토글 (Tourist/Mate)로 입력·샘플·출력 분기
- 지도·템플릿·저장·공유는 100% 공통 코드

### 1-2. 빈 화면 공포 해소
- 입력 필드 **5개로 최소화** (현행 9 → 5)
- 대표 샘플 플랜 **무조건 노출** — 첫 스크롤에 AI 결과물 확인
- "아무 입력 없이도 가치가 보이는" 전략

### 1-3. LLM 최소·구조화 우선 (`AI_COST_MODEL §5`)
- 기본 `gpt-4o-mini`
- 검색·프로필·지도는 non-LLM 경로
- LLM은 플랜 합성·후속 대화에만 사용

### 1-4. 모바일 98% 전제
- 세로 스크롤, chips 기반 선택
- Sticky 하단 액션 버튼

### 1-5. 3-Layer 분리 ⭐ v1.0 신설

```
┌────────────────────────────────────────────┐
│ A. Entry Source Layer  (§2)                │
│    사용자가 어디서 왔는지 (메타데이터)       │
│    enum 5개                                 │
├────────────────────────────────────────────┤
│ B. Intent Layer        (§3)                │
│    지금 뭘 하려는지 (온페이지 분류)          │
│    4개 카드 복수 선택                        │
├────────────────────────────────────────────┤
│ C. Input Layer         (§4·§5)             │
│    실제 AI가 플랜 만들기 위한 데이터          │
│    Tourist 5필드 / Mate 5필드 + Plan Depth │
└────────────────────────────────────────────┘

Rule:
- 진입점(A)이 의도(B) 프리셋 (강제 아님)
- 의도(B) 확정 후 입력(C) 표시
- 입력(C) 완성 시 AI 초안 → 추가 조정
```

---

## 2. Entry Source Layer (구 §9 재편)

페이지 진입 시 자동 판정 + 메타데이터 기록.

### 2-1. enum 정의 (확정)

```typescript
type EntrySource =
  | "home"          // 홈 히어로 "AI에게 맡기기" 버튼
  | "wishlist"      // /search 위시리스트 → "AI에게 보내기"
  | "venue"         // 장소 상세에서 "이거 포함 플랜"
  | "mate_profile"  // Mate 프로필 "이 Mate와 상담"
  | "shared"        // 공유 URL 방문 "자신도 작성"
```

### 2-2. URL 파라미터 매핑

| Entry Source | URL 파라미터 | 동작 |
|-------------|-------------|------|
| `home` | `/ai/plan` | 빈 상태 진입 |
| `wishlist` | `/ai/plan?wish_ids=w1,w2` | 위시 사전 주입 |
| `venue` | `/ai/plan?venue_id=v123` | 장소 고정 |
| `mate_profile` | `/ai/plan?mate_id=m456` | Mate 지정 |
| `shared` | `/ai/plan?from_plan_id=p789` | 기존 플랜 참조 |

### 2-3. Intent 프리셋 규칙

`entrySource` 감지 시 Intent 카드 정렬·기본값 변경 (**프리셋이지, 강제 아님**):

```
from /beauty 페이지 → intent=beauty 카드 상단·자동 체크
from /search (K-Pop 축제 위시) → intent=festival 카드 상단
기타 → 중립 (홈 순서)
```

사용자는 언제든 Intent 재선택 가능.

---

## 3. Intent Layer (구 §21 재편)

Planner 안에서 "무엇을 우선으로 계획할지" 선택.

### 3-1. enum 정의 (확정)

```typescript
type Intent = 
  | "festival"       // 🎪 축제·K-Pop·이벤트
  | "beauty"         // 💆 K-Beauty 시술
  | "food_shopping"  // 🍜 맛집·쇼핑·관광
  | "mate"           // 🤝 한국 메이트와 함께

// 복수 선택 허용
// 2개 이상 선택 시 내부 flag: mixed = true
```

### 3-2. UX

```
"한국 여행, 어떻게 채우고 싶으세요?"  (복수 선택 가능)

☐ 🎪 축제·K-Pop·이벤트
☐ 💆 K-Beauty 시술
☐ 🍜 맛집·쇼핑·관광
☐ 🤝 한국 메이트와 함께

→ [다음: 기본 정보 입력]
```

### 3-3. 시나리오별 헤드라인 (intent 조합 → 문구)

| 조합 | 헤드라인 |
|------|---------|
| 💆 + 🤝 | "시술도 하고 메이트도 만나고 싶다면, AI가 최적 일정을 제안" |
| 🎵 + 🤝 | "콘서트 일정에 맞춰, 메이트와 함께하는 여행을 만들어" |
| 💆만 | "시술 일정 중심, 귀국일 맞춘 최적 일정" |
| 🎵만 | "콘서트·이벤트 날짜 중심 일정" |
| 🍜만 | "맛집·쇼핑 중심 한국 여행" |
| 전체 선택 (mixed) | "한국 여행의 모든 것, 하나의 일정으로" |

### 3-4. Beauty Intent — 추가 Step (K-Beauty 특수)

`beauty` 선택 시 **Step 2.5: 시술 정보** 삽입:

**Step 2.5-A: 시술 선택**
- `/beauty` 페이지에서 담아온 시술 자동 주입
- 각 시술의 다운타임 표시
- "+ 시술 추가" 가능

**Step 2.5-B: 시술 타이밍 AI 최적화**

| 패턴 | 구조 | AI 추천 조건 |
|------|------|-------------|
| **A** (시술 먼저) | Day 1 시술 → 회복 → 여행+메이트 | 다운타임 짧은 시술 (보톡스 등) |
| **B** (여행 먼저) | 여행+메이트 → Day 마지막 시술 → 귀국 | 다운타임 긴 시술 (울쎄라·써마지) |
| **C** (언제든) | 시술 일정 유연 | 다운타임 거의 없는 시술 |

패턴 B 전략: **귀국 직전 시술 → 다운타임은 일본에서 회복**.

---

## 4. Input Layer — Tourist (구 §4 재편)

### 4-1. 5 필드 Progressive Disclosure

| # | 필드 | UI | 필수 | 기본값 |
|---|------|----|------|-------|
| 1 | 📅 **날짜** | 날짜 범위 picker | ✓ | - |
| 2 | 👥 **인원** | 남/여 분리 +/- (확정/미정) | ✓ | - |
| 3 | 🤝 **Mate 여부** | chips [함께/AI만/나중에] | - | 나중에 |
| 4 | 💰 **예산** | chips [최저/¥5만/¥5~10만/...] | - | 상관없음 |
| 5 | ✏️ **자유 기입** | textarea 140자 | - | (empty) |

**필수 완성 조건**: 날짜 + 인원만 있으면 AI 호출 가능. 나머지는 AI가 인라인 질문으로 수집.

### 4-2. 인원 세부 UX (남/여 분리 + 확정/미정)

```
👥 인원
  ◆ 확정 인원
    👨 남  [−] 0 [+]
    👩 여  [−] 2 [+]
  ◇ 미정 인원 (올 수도 있음)
    👨 남  [−] 1 [+]
    👩 여  [−] 0 [+]
  총 확정 2명 + 미정 1명 = 최대 3명
```

**AI 활용**:
- 확정 2명 (여2) → FF 페어, 식당 2인석 예약
- 미정 남1 → 유동적 플랜 (2명 vs 3명 옵션 양쪽 준비)

### 4-3. 관심 프로그램 분리 (확정 / Explore)

**확정 사항** (검색 위시리스트 자동 주입)
```
[청주 직지축제 ✕] [법주사 ✕] [압구정 OO피부과 ✕]
```
→ AI 플랜 **반드시 포함**

**Explore** (관심 키워드·지역)
```
☐ 🎪축제 ☐ 💄K뷰티 ☑ 🏆명인 ☐ 🏯템플 ...
```
→ AI 플랜 **추천 후보**

### 4-4. 세부 설정 아코디언

숨겨둔 추가 필드 (진입 시 접힌 상태):

```
[세부 설정 ▼]  ← 클릭 시 확장
  • 출발 국가·도시 (AI 자동 감지, 수정 가능)
  • 선호 Mate 소통 언어
  • 기타 제약
```

### 4-5. 숨기는 기존 6필드 (자동 추론)

- 출발 국가·도시: Accept-Language + IP 자동
- 관심사: 검색 히스토리 + 위시리스트 기반
- 희망 지역: AI가 일정·관심사 기반 자동 추천
- 선호 UI 언어: `locale` 설정

---

## 5. Input Layer — Local Mate (구 §5 재편)

### 5-1. Base Offering Input (5필드, 항상 필수)

**Mate Tier·Plan Depth와 무관하게 모든 Offering에 필수**.

| # | 필드 | UI | 설명 |
|---|------|----|------|
| 1 | 📅 **제공 기간** | 날짜 범위 multi | 시간 낼 수 있는 기간 (Tier TTL 적용) |
| 2 | 🧑‍🤝‍🧑 **페어 포맷** | chips [FF/MF/GG/BB/solo] | `matching.pair_format_allowed` 필터 |
| 3 | 💰 **가격** | 숫자 + 단위 토글 [세션/시간] | 포인트 or JPY |
| 4 | 📍 **지역** | 시·도 multi, 프로필 기본값 | Override 가능 |
| 5 | ✏️ **Offering 자기소개** | textarea 140자 | 이 Offering 고유 훅 |

### 5-2. 프로필 위임 (필드에 없음)

- **언어 레벨**: Mate 프로필 전용 (Offering마다 안 바뀜)
- **기본 자기 프로필**: 별도 Mate 포털에서 관리

### 5-3. 지역 Override UX

```
기본값: Mate 프로필의 "활동 가능 지역" 자동 채움 (예: ["서울", "부산"])
→ 이 Offering만 다르게 설정 가능
→ [프로필 값으로 복원] 버튼
```

---

## 6. Plan Depth (구 §17 재편) ⭐ v1.0 재정의

**Mate·Tourist 양쪽 공통**. Plan의 "얼마나 상세한가" 선택. **Base Input과 독립된 2번째 축**.

### 6-1. 3단계 정의

| Level | 이름 | 내용 | 수정시간 | AI Planner |
|-------|------|------|----------|-----------|
| **Open** | 간단 | Base 5필드만 | ~5분 | 사용 안 함 |
| **Guided** | 중간 | Base + 추천 장소·대략 흐름 | ~10분 | 선택적 |
| **Detailed** | 상세 | Guided + Day-by-Day + 항목별 태그 | ~20분 | 필수 |

### 6-2. Tourist·Mate 예시

**Mate Offering**:
- **Open**: "주말 강남 일본 여성 OK, 맛집 투어, 5만원"
- **Guided**: Open + "압구정 갈비·가로수길 디저트·신사역 화장품"
- **Detailed**: Guided + "13:00 압구정역 미팅 → 13:15 갈비 → 14:30 이동..." (시간 단위)

**Tourist Plan**:
- **Open**: "5월 서울 3박, 여자 2, 축제+뷰티"
- **Guided**: Open + 담고 싶은 장소 3곳
- **Detailed**: Guided + Day-by-Day 완성 + 항목별 태그

### 6-3. 항목별 태그 (Detailed 전용)

Plan의 각 시간대·장소에 4가지 태그:

| 태그 | 의미 | AI·UX 동작 |
|------|------|----------|
| ✅ **확정** | 이 시간·장소 고정 | 변경 불가, 정확 매칭 |
| 💬 **협의** | 의견 교환 대상 | 상대방 대안 제안 가능 |
| ✨ **추천** | 제안자 의견, 변경 허용 | 바꿔도 OK |
| ❓ **미정** | 빈 항목 | 매칭 후 같이 채움 |

**예시**:
```
Day 1:
  13:00 압구정역 미팅       [✅ 확정]
  13:15 갈비집 점심         [✨ 추천 — 비건 시 대체]
  15:00 카페 "OOO"         [💬 함께 정함]
  22:00 자유 시간           [❓ 아직]
```

### 6-4. Config 매핑

```
plan.depth_enabled              = ["open", "guided", "detailed"]
plan.depth.default              = "guided"
plan.depth.open.fields          = 5 (base only)
plan.depth.guided.extras        = ["recommended_venues", "rough_flow"]
plan.depth.detailed.requires    = ["day_plan", "item_tags"]

plan.item_tags.enabled          = ["confirmed", "discuss", "suggest", "empty"]
plan.item_tags.default          = "suggest"
plan.matching.confirmed_weight  = 1.0
plan.matching.discuss_weight    = 0.5
plan.matching.suggest_weight    = 0.3
```

---

## 7. 역할 토글 & 페이지 진입 플로우

```
┌──────────────── /ai/plan ────────────────┐
│ [AI 인사말] (entrySource·context-aware)    │
├──────────────────────────────────────────┤
│ [🧳 Tourist] [🎎 Local Mate]             │ Step 0: Role
├──────────────────────────────────────────┤
│ "한국 여행, 어떻게 채우고 싶으세요?"         │ Step 2: Intent
│ (복수 선택, entrySource로 프리셋)           │
│ ☐ 🎪축제  ☐ 💆뷰티  ☐ 🍜맛집  ☐ 🤝메이트 │
├──────────────────────────────────────────┤
│ [시술 Step: beauty 선택 시만]               │ Step 2.5
├──────────────────────────────────────────┤
│ 📝 기본 정보 (5필드 Progressive)            │ Step 3: Input
│ 📅 날짜 | 👥 인원 (필수)                   │
│ [다음: 예산·Mate 여부 →]                   │
│                                           │
│ [Mate 모드 시] Plan Depth 선택             │
│ ● Open ○ Guided ○ Detailed                │
├──────────────────────────────────────────┤
│ [ ✨ AI Planner + Matching 시작 ]         │ Step 4: Generate
├──────────────────────────────────────────┤
│ 🎬 AI 플랜 결과                            │ Step 5: Output
│ [📋 Day 카드][🗺 지도][📅 타임라인]        │
│                                           │
│ 하단 액션:                                 │
│ [💎 AI Matching] [💾 저장] [🔗 공유] [🔄 재생성]
├──────────────────────────────────────────┤
│ 대표 샘플 1개 (Trust, §8-①)               │
│ (Post-MVP) 갤러리 4개 (§8-③)              │
└──────────────────────────────────────────┘
```

**자동 판정 우선순위 (역할 토글)**:
1. `?role=mate` 강제 Mate
2. 로그인 사용자 `user.role === 'mate'` → Mate 기본
3. 기타 → Tourist 기본

---

## 8. 샘플·갤러리 전략 (구 §6 재편)

Genspark 지적: "샘플 / Intent 카드 / 갤러리"는 **역할 다른 3개**.

### 8-①. 대표 샘플 1개 — Trust (MVP 필수)

```
🎬 AI Planner가 만드는 여행 예시
"3박4일 서울 — K뷰티 · 축제 · 명장"
[펼쳐서 상세 보기]
```

- 화면 하단에 항상 노출
- 완성된 Day-by-Day 플랜 + 지도 + 예산
- **목적**: "AI가 뭘 해주는지" 증명 → 신뢰·전환

### 8-②. Intent 카드 4개 — Routing (MVP 필수)

위 §3 Intent Layer 의 카드들이 여기에 해당. **별도 섹션 없이 재활용**.

### 8-③. 템플릿 갤러리 4개 — Inspiration (Post-MVP)

```
🔖 빠른 시작 템플릿 (P2 이후)
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ 서울 3박 K뷰티 │ 부산 주말     │ 제주 일주일    │ K-POP 성지   │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

- 각 카드 [試す] 클릭 → Tourist 폼 자동 채움
- MVP에서 제외, 콘텐츠 큐레이션 여력 생긴 후

### 8-④. Config

```
ai.planner.sample_strategy    = "hero_single"
ai.planner.sample_count       = 1
ai.planner.intent_cards_count = 4
ai.planner.gallery_enabled    = false  # MVP 이후 true
ai.planner.gallery_count      = 4
```

---

## 9. 출력 포맷 & 액션 버튼

### 9-1. Day 카드 (기본 뷰)
```
┌─────────────────────────────┐
│ DAY 1 · 4/10 (金) · 서울     │
├─────────────────────────────┤
│ 🎪 14:00 景福宮 夜間開場     │
│ 🍜 19:00 光化門食堂           │
│ 🛏 21:00 明洞ホテル           │
│                             │
│ この日の予想: ¥12,000        │
└─────────────────────────────┘
```

### 9-2. 뷰 토글 (상단 탭)
- **[📋 日程表]** Day 카드 (기본)
- **[🗺 地図]** 카카오맵 + 카테고리 컬러 마커
- **[📅 タイムライン]** 시간순 리스트

### 9-3. 액션 4 버튼 (Sticky Bottom)

| 버튼 | 동작 | 비용 |
|------|------|------|
| 💎 **AI Matching** | `/ai/matching` 이동 (가장 강조) | 무료 |
| 💾 **저장** | DB 저장, 비로그인 시 로그인 유도 | 무료 |
| 🔗 **공유** | `/ai/plan/[id]` URL 생성·복사 | 무료 |
| 🔄 **재생성** | `user_signal` → 4o fallback | 500pt |

---

## 10. AI Matching 진입 UX (구 §7 유지)

플랜 완성 후 sticky 영역:

### Tourist
```
💎 このプランに合うメイトを探す
(無料 · Tier 1 公開情報のみ閲覧)
→ /ai/matching?plan_id=xxx
```

### Mate
```
💎 この提案を気に入る旅行者を探す
(無料 · Tier 1 公開情報のみ閲覧)
→ /ai/matching?offering_id=xxx
```

**클릭 시 플로우**:
1. Plan/Offering DB 저장 (비로그인 시 로그인 유도)
2. `/ai/matching` 수직 분할 리스트로 이동
3. 별도 문서: `MATCHING_PAGE_DESIGN.md`

---

## 11. 모드 자동 전환 (PLAN → MATCH → GUIDE)

`AI_ROLE_DEFINITION §4-3` 단일 페이지 원칙. 사용자는 하나의 앱만 알고 **내부에서 자동 전환**.

| 현재 | 전환 트리거 | 다음 |
|------|-----------|------|
| **PLAN** (플랜 작성 중) | [AI Matching] 클릭 | **MATCH** (`/ai/matching`) |
| **MATCH** | ❤️ 상호 해제 + 예약 확정 | **BOOKED** (Plan 확정) |
| **BOOKED** | 여행 시작 + 위치 한국 | **GUIDE** (`/ai/guide`, M3) |

**UI 뱃지**: 현재 모드 상단 표시
```
今: ✨ プラン作成中 (PLAN)
```

**매칭 정책 키** (`ADMIN_CONFIGURABLE_PARAMS §3-3` v1.3):
- `matching.approval_mode = "mutual"` (쌍방 ❤️ 필수)
- `matching.initiating_mode = "bidirectional"` (양쪽 이니시에이팅)
- `matching.tier2_unlock_mode = "mutual_interest"`
- `matching.tier3_unlock_mode = "booking_confirmed"`
- `matching.review_deadline_hours = 48`

---

## 12. Mate Tier (구 §16 재편) ⭐ v1.0 용어·매핑 확정

### 12-1. Internal Canonical Tier — 5단계 (AI_TERMINOLOGY 준수)

| Tier | 진입 조건 | 자동 상승 조건 |
|------|-----------|---------------|
| **Applicant** | 가입 신청 | 관리자 승인 → Rookie |
| **Rookie** | 관리자 승인 | 5회 완료 + 평점 4.5 이상 → Verified |
| **Verified** | 자동 | 30회 + 평점 4.7 이상 + 관리자 pick → Featured |
| **Featured** | 관리자 pick | 50회 + 평점 4.8 이상 → Pro Guide 심사 |
| **Pro Guide** | 관리자 최종 심사 | (최상위) |

### 12-2. Display Tier — 3단계 + ⭐ 배지 (UI)

| Internal | Display | 배지 |
|----------|---------|------|
| Applicant | (숨김, 활성 X) | - |
| Rookie | 🌱 **New Mate** | 🌱 |
| Verified | ✓ **Verified Mate** | ✓ |
| Featured | ✓ **Verified Mate** + ⭐ **Top Pick** | ✓ ⭐ |
| Pro Guide | 💎 **Pro Guide** | 💎 |

### 12-3. UI 예시

```
[매칭 페이지 Mate 카드]

[✓Verified] [⭐Top Pick]
김지수 · 서울 · FF · 日本語 N1
⭐ 4.9 · 43 세션

[💎 Pro Guide]
박민준 · 서울 · 관광통역사 자격
⭐ 4.95 · 127 세션
```

### 12-4. Tier별 Offering 정책 (`ADMIN_CONFIGURABLE_PARAMS §3-3`)

| Internal Tier | TTL | 동시 활성 | 연장 |
|---|---|---|---|
| Applicant | — | 0 (금지) | — |
| Rookie | 30일 | 1 | ✗ |
| Verified | 60일 | 3 | ✗ |
| Featured | 90일 | 5 | ✗ |
| Pro Guide | 180일 | 10 | ✓ |

### 12-5. Tourist 접근 자격

| 상태 | AI Planner | Mate 매칭 | 플랜 저장 |
|------|-----------|----------|---------|
| 비로그인 | ✅ | ❌ (로그인 유도) | ❌ |
| 로그인 | ✅ | ✅ (v1.5 이후) | ✅ |

**v1 티저 모드** (`matching.v1_teaser_mode`):
- v1.0: 실제 매칭 기능 없음, 티저 UI만
- Mate 10명+ 확보 시 `matching.v1_teaser_mode = false` → 실제 활성화

---

## 13. 매칭 경로 (구 §18)

### Tourist 매칭 경로 3가지
1. **외부 검색** (우리 외부, 관여 X)
2. **AI Planner** — `/ai/plan` → AI Matching
3. **Mate 제안서 브라우징** — `/ai/matching` 직접

### 매칭 방식 4가지 (공통)
1. 하나씩 브라우징 + ❤️
2. 키워드 검색
3. AI 유사도 검색
4. 직접 간단 제안 등록

### Mate 탭 목적 토글
- **Looking for Tourist** — Offering 작성 (Plan Depth 선택)
- **Looking for Mate** — `/ai/matching` 이동, Tourist 요청 브라우징

---

## 14. 현행 구현 Gap (개발 체크리스트)

`app/src/app/ai-guide/page.tsx` + `api/ai-guide/route.ts` 기준.

### 🔴 구조 변경 (3-Layer)
- [ ] 파일 이동: `/ai-guide/` → `/ai/plan/`
- [ ] Entry Source 메타 수집 구현 (URL 파라미터 파싱)
- [ ] Intent 카드 4개 컴포넌트 신규
- [ ] Intent → Input progressive disclosure 전환 로직
- [ ] 세부 설정 아코디언

### 🔴 데이터 모델
- [ ] Tourist 5필드 (남녀 분리 인원 포함)
- [ ] Mate 5필드 + Plan Depth
- [ ] 항목별 태그 (Detailed only)
- [ ] `plans` · `offerings` 테이블 schema
- [ ] `intents` 복수 선택 JSONB
- [ ] `plan_depth` enum 컬럼

### 🔴 AI 백엔드
- [ ] 모델 `gpt-4o` → `gpt-4o-mini` config 교체
- [ ] Validator 구현 (AI_COST_MODEL §5 Trigger A)
- [ ] Fallback 로직 (validator_or_user_signal)
- [ ] Response/Prompt 캐싱
- [ ] Beauty intent micro-step (시술 타이밍 3패턴)

### 🔴 다국어
- [ ] 한국어 UI → 일본어 기본 + EN/KR
- [ ] AI 응답 언어 locale 일치

### 🟡 신규 기능
- [ ] 대표 샘플 1개 (하드코딩, K뷰티+축제+명장)
- [ ] Mate Tier 배지 (New/Verified/Top Pick/Pro)
- [ ] AI Matching 버튼 → `/ai/matching` 라우트
- [ ] 저장·공유 URL (`/ai/plan/[id]`)
- [ ] 재생성 버튼 + 500pt 차감

### 🟢 디테일
- [ ] Day 카드 DESIGN.md §Color 준수
- [ ] 지도 뷰 카카오맵 API
- [ ] Sticky 액션 bottom
- [ ] 모드 뱃지 (PLAN/MATCH/GUIDE)

---

## 15. Config 파라미터

```
# ─── 3-Layer 구조
ai.planner.entry_source.enum     = ["home","wishlist","venue","mate_profile","shared"]
ai.planner.intent.enum            = ["festival","beauty","food_shopping","mate"]
ai.planner.intent.multi_select    = true
ai.planner.intent.default         = []

# ─── 필드
ai.planner.tourist_fields_required = ["dates", "group_size"]
ai.planner.tourist_fields_visible  = ["dates","group_size","mate_wish","budget","note"]
ai.planner.mate_fields_required    = ["period","pair_format","price","region","intro"]
ai.planner.mate_fields_visible     = (same as required)
ai.planner.show_detail_toggle      = true
ai.planner.detail_fields           = ["origin","interests","preferred_region","language"]

# ─── Plan Depth
plan.depth_enabled                 = ["open","guided","detailed"]
plan.depth.default                 = "guided"
plan.item_tags.enabled             = ["confirmed","discuss","suggest","empty"]
plan.item_tags.default             = "suggest"

# ─── 모델
ai.model.plan                      = "gpt-4o-mini"
ai.model.plan_fallback_trigger     = "validator_or_user_signal"
ai.model.user_upgrade_cost_points  = 500

# ─── 샘플·갤러리
ai.planner.sample_strategy         = "hero_single"
ai.planner.sample_count            = 1
ai.planner.intent_cards_count      = 4
ai.planner.gallery_enabled         = false
ai.planner.gallery_count           = 4

# ─── Matching 연결
ai.planner.matching_cta_enabled    = true
ai.planner.matching_route          = "/ai/matching"

# ─── Offering 2축 (§3-3)
offerings.rookie.ttl_days          = 30
offerings.rookie.concurrent_max    = 1
# ... (Verified/Featured/Pro_guide 동일)

# ─── Matching 정책 (§3-3)
matching.approval_mode             = "mutual"
matching.initiating_mode           = "bidirectional"
matching.tier2_unlock_mode         = "mutual_interest"
matching.tier3_unlock_mode         = "booking_confirmed"
matching.review_deadline_hours     = 48

# ─── Mate Tier Display 매핑
tier.display.applicant             = null          # 숨김
tier.display.rookie                = "new"
tier.display.verified              = "verified"
tier.display.featured              = "verified+top_pick"
tier.display.pro_guide             = "pro_guide"

# ─── v1 티저
matching.v1_teaser_mode            = true
matching.v1_waitlist_enabled       = true
matching.v1_activate_threshold     = 10
```

---

## 16. 미결 사항 (v1.0에서도 남음)

### 🟡 UX 세부
1. 모바일 vs 데스크톱 레이아웃 상세 (2-column 옵션)
2. 대표 샘플 실제 콘텐츠 (축제·장소·시간·가격)
3. 재생성 버튼 세부 UX (확인 모달·포인트 부족 시)
4. Mate 모드 지역 Override UX
5. Plan 비공개 모드 (tier1_visibility 연계)

### 🟡 비즈니스
6. Premium 구독자 재생성 무제한
7. 비로그인 Plan 저장 (익명 세션 vs 로그인 강제)
8. Beauty micro-step 실제 시술 DB 연동 시점

### 🟢 장기
9. AI Matching 진입 후 편집 잠금 정책
10. 공유 URL 방문자 "자신도 작성" 변형 시나리오
11. Offering 가격 단위 (세션 vs 시간) 혼용 방지

---

## 17. 변경 영향 — 기존 문서 업데이트 필요

- [ ] **`SITEMAP_SPEC.md`** §3 — `/ai-guide` → `/ai/plan` URL
- [x] **`ADMIN_CONFIGURABLE_PARAMS.md`** v1.3 — §3-3 Offering·Matching·Tier (완료)
- [ ] **`ADMIN_CONFIGURABLE_PARAMS.md`** v1.4 — §3-14 `ai.planner.*` 신설
- [ ] **`SITE_COMPLETION_PLAN.md`** P1-14 — 3-Layer 구조 재편 반영
- [ ] **`AI_ROLE_DEFINITION.md`** §3 Role 1 — Intent Layer 명시
- [ ] **`AI_TERMINOLOGY.md`** §8 — URL `/ai/plan` 확정
- [x] **`docs/MATCHING_PAGE_DESIGN.md`** — AI Matching 페이지 (완료)

---

## 18. 관련 Config 크로스레퍼런스

| Config 키 | 영향 |
|---|---|
| `ai.model.plan` | 기본 AI 모델 |
| `ai.planner.*` (§3-14 신설) | Layer·필드·샘플 |
| `plan.depth_*` / `plan.item_tags_*` | Plan Depth·태그 |
| `ai.cache.*` | 응답 캐싱 |
| `matching.approval_mode` / `initiating_mode` | 매칭 정책 |
| `matching.tier1/2/3_fields` | Plan/Offering 공개 범위 |
| `offerings.{tier}.ttl_days` / `concurrent_max` | Mate Tier별 Offering 정책 |
| `tier.display.*` | Mate Tier UI 매핑 |
| `locale.default` | UI·AI 응답 언어 |

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|------|------|------|
| 2026-04-19 | v0.1 최초 | Mate 5필드 확정 (Tourist 대칭) |
| 2026-04-19 (저녁) | v0.2 | Level 시스템 3단계·항목별 태그·매칭 경로·인원 분리·관심 2섹션 추가 |
| 2026-04-20 | v0.3 | Mate 등급 시스템·v1 티저 모드·Tourist 오픈 액세스 |
| 2026-04-21 | v0.4 | §21 진입 재설계, K-Beauty 전략 포지션, 시술 타이밍 3패턴 |
| **2026-04-22** | **v1.0 승격** | **Genspark 조언자 리뷰 반영. 6개 충돌 통합: 3-Layer 분리·Mate Tier/Plan Depth 용어·직교 2축·enum 확정·샘플 3분리·Internal 5/Display 3 매핑** |

---

*버전: v1.0 / 마지막 업데이트: 2026-04-22*
