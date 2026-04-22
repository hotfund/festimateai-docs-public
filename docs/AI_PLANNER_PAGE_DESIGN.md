# AI Planner 페이지 설계 (AI Planner Page Design)

> **작성**: 2026-04-19
> **위상**: 페이지 설계 SSOT — `/ai/plan` (현재 `/ai-guide`) 구현 기준
> **선행 문서**: `AI_TERMINOLOGY.md` §2~§5 (용어·Tier), `AI_ROLE_DEFINITION.md` §3 (3역할), `AI_COST_MODEL.md` §5 (Fallback), `SEARCH_PAGE_DESIGN.md` (패턴 참조), `ADMIN_CONFIGURABLE_PARAMS.md` §3-4·§3-14 (config)
> **현행 구현**: `app/src/app/ai-guide/page.tsx` (Tourist 모드만, 9필드 폼, 한국어 UI, `gpt-4o` 하드코딩)
> **설계 목표**: 일본 20~30 여성 Tourist + 한국 Local Mate **양쪽 공용 canvas**로 DRY 정점 구현

---

## 1. 핵심 원칙

### 1-1. 양쪽 공용 Canvas (DRY 정점)
- **같은 URL · 같은 UI 구조 · 같은 AI**
- 역할 토글(Tourist/Mate)로 입력·샘플·출력 분기
- 지도·템플릿·저장·공유는 100% 공통 코드

### 1-2. 빈 화면 공포 해소
- 입력 필드 **5개로 최소화** (현행 9 → 5)
- 샘플 플랜 **무조건 노출** — 첫 스크롤에 AI 결과물 확인
- "아무 입력 없이도 가치가 보이는" 전략

### 1-3. LLM 최소·구조화 우선 (`AI_COST_MODEL §5`)
- 기본 `gpt-4o-mini`
- 검색·프로필·지도는 non-LLM 경로
- LLM은 플랜 합성·후속 대화에만 사용

### 1-4. 모바일 98% 전제
- 3-Zone 세로 스크롤 (Nav → Role → Input → Gallery → Sample)
- chips 기반 선택 (키보드 입력 최소)
- Sticky 하단 액션 버튼

---

## 2. 역할 토글 (최상단)

페이지 진입 시 자동 판정 후 수동 전환 허용.

```
┌──────────────────────────────────────┐
│ FestiMate        🌐 JP ▼       ログイン │
├──────────────────────────────────────┤
│  私は:                                 │
│  ┌─────────────┐  ┌─────────────┐    │
│  │ 🧳 旅行者     │  │ 🎎 ローカル  │    │
│  │ Tourist      │  │ メイト       │    │
│  └─────────────┘  └─────────────┘    │
└──────────────────────────────────────┘
```

**자동 판정 우선순위**:
1. 쿼리 `?role=mate` → 강제 Mate (Mate 모집 페이지·공유링크에서 오는 경우)
2. 로그인 사용자 `user.role === 'mate'` → Mate 기본
3. 기타 → Tourist 기본

**전환 시**: 입력 필드·샘플·액션 버튼 전체 교체. URL은 그대로.

---

## 3. 페이지 3-Zone 구조

```
┌──────────────────────────────────────┐
│ [Nav]                                │
├──────────────────────────────────────┤
│ [Role Toggle]                  §2    │
├──────────────────────────────────────┤
│ Zone 1: 입력 티저 (5필드)       §4/5 │
├──────────────────────────────────────┤
│ Zone 2: 템플릿 갤러리 (Quick-start) §6│
├──────────────────────────────────────┤
│ Zone 3: 샘플 쇼케이스 (Full)    §6    │
├──────────────────────────────────────┤
│ [Sticky Footer: 액션 4버튼]    §8    │
└──────────────────────────────────────┘
```

**모바일**: 세로 스크롤. **데스크톱** (1024px+): Zone 2·3을 2-column 나란히 배치 옵션.

---

## 4. Tourist 모드 — 입력 5필드

| # | 필드 | UI | 필수 | 기본값 |
|---|---|---|---|---|
| 1 | 📅 **날짜** | 날짜 범위 picker | ✓ | - |
| 2 | 👥 **인원** | chips [1/2/3/4/5+] | ✓ | - |
| 3 | 🤝 **Mate 여부** | chips [함께/혼자/모름] | - | 모름 |
| 4 | 💰 **예산** | chips [최저/¥5만/¥5~10만/¥10~20만/¥20만+/상관없음] | - | 상관없음 |
| 5 | ✏️ **자유 기입** | textarea 140자 | - | (empty) |

### 숨기는 기존 6필드 (자동 추론 or 프로필)
- **출발 국가·도시**: Accept-Language + IP
- **관심사**: 검색 히스토리 + 위시리스트 기반
- **희망 지역**: AI가 일정·관심사 기반 자동 추천
- **선호 언어**: `locale` 설정 사용

### 세부 조정 UX
```
[세부 설정 ▼]  ← 접힌 상태 기본
    └ 클릭 시 6필드 인라인 확장
```

**필수 완성 조건**: 날짜 + 인원만 있으면 즉시 AI 호출 가능.

---

## 5. Local Mate 모드 — 입력 5필드 (확정: 옵션 C)

Tourist 5 ↔ Mate 5 **1:1 대칭**.

| # | 필드 | UI | 필수 | 설명 |
|---|---|---|---|---|
| 1 | 📅 **제공 기간** | 날짜 범위 multi | ✓ | 시간 낼 수 있는 기간 |
| 2 | 🧑‍🤝‍🧑 **페어 포맷** | chips [FF/MF/GG/BB/solo] | ✓ | `matching.pair_format_allowed` 필터 |
| 3 | 💰 **가격** | 숫자 + 단위 토글 [세션/시간] | ✓ | 포인트 or JPY 환산 표시 |
| 4 | 📍 **지역** | 시·도 multi, 프로필 기본값 | ✓ | Override 가능 |
| 5 | ✏️ **Offering 자기소개** | textarea 140자 | ✓ | 이 Offering 고유 훅 |

### 프로필 위임 항목 (필드에 없음)
- **언어**: Mate 프로필 전용. Offering마다 바뀌지 않음 → 중복 입력 제거.
- **자기 기본 프로필**: 별도 Mate 프로필 페이지에서 관리.

### 지역 필드 상세
```
기본값: Mate 프로필의 "활동 가능 지역" 자동 채움
예) ["서울", "부산"]
→ 이 Offering만 다르게 하려면 편집
→ [프로필 값으로 복원] 버튼 제공
```

### Offering 정책 (`ADMIN_CONFIGURABLE_PARAMS §3-3` v1.3 — Tier 2축 모델)

| Tier | TTL | 동시 활성 | 연장 |
|---|---|---|---|
| Applicant | — | 0 (금지) | — |
| Rookie | 30일 | 1 | ✗ |
| Verified | 60일 | 3 | ✗ |
| Featured | 90일 | 5 | ✗ |
| Pro Guide | 180일 | 10 | ✓ |

Config 키: `offerings.{tier}.ttl_days` · `offerings.{tier}.concurrent_max` · `offerings.{tier}.extendable` · `offerings.applicant.allowed`

---

## 6. 샘플 플랜 미리보기 (Zone 3 — 정적 전략)

### 6-1. Tourist 샘플

**기본** (첫 방문, 입력 없음):
- 큐레이션 1개: **"3泊4日 ソウル 桜 + K-뷰티 + 名匠"**
- 일본 20~30 여성 최적 페르소나 (K뷰티 메인 수익원 격상 반영)

**맥락 있을 때** (검색·위시 히스토리):
- 사용자 클릭 이력 기반 샘플 재조합 (LLM 미사용, 규칙 기반)
- 예: 위시에 "강남 클리닉 3곳" → K-뷰티 중심 샘플

**메타 라벨**:
> 「こちらはサンプル — あなたの好みで変わります」

### 6-2. Mate 샘플

**기본**: 샘플 Offering 1개
- **"주말 서울 FF 페어 · 일본어 가능 · ¥8,000/세션"**
- "Verified Mate들이 주로 올리는 Offering 예시"

### 6-3. 렌더 방식

- Day-by-Day 카드 (현행 `PlanGrid` 재활용)
- 지도 뷰 토글 (현행 `PlanMap` 재활용)
- **[このプランを試す]** 버튼 → Zone 1 폼 자동 채움 + 재생성

### 6-4. Zone 2 템플릿 갤러리 (4개)

| # | 제목 | 기둥 |
|---|---|---|
| 1 | 3泊4日 ソウル 美容 + ショッピング | K-뷰티 (메인) |
| 2 | 2泊3日 桜 + 伝統工芸 | Festival + Master |
| 3 | 4泊5日 テンプルステイ + 自然 | Templestay + Garden |
| 4 | 1週間 K-POP 聖地巡礼 | Long-form premium |

각 카드 [試す] 클릭 → Tourist 폼 자동 채움 + Zone 3 해당 플랜 전체 렌더.

---

## 7. AI Matching 진입 UX

플랜 완성 후 하단 sticky 영역에 강조 CTA:

### Tourist
```
┌─────────────────────────────────────┐
│ 💎 このプランに合うメイトを探す       │
│ (無料 · Tier 1 公開情報のみ閲覧)      │
└─────────────────────────────────────┘
  → /ai/matching?plan_id=xxx
```

### Mate
```
┌─────────────────────────────────────┐
│ 💎 この提案を気に入る旅行者を探す     │
│ (無料 · Tier 1 公開情報のみ閲覧)      │
└─────────────────────────────────────┘
  → /ai/matching?offering_id=xxx
```

**클릭 시**:
1. Plan/Offering DB 저장 (비로그인이면 로그인 유도 모달)
2. `/ai/matching`으로 라우팅
3. 수직 분할 리스트 (Tourist 측·Mate 측 나란히 — 별도 문서 `MATCHING_PAGE_DESIGN.md` 예정)

---

## 8. 출력 포맷 & 액션 버튼

### 8-1. Day 카드 (기본 뷰)
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

### 8-2. 뷰 토글 (상단)
- **[📋 日程表]** (기본)
- **[🗺 地図]** — 카카오맵 + 마커 색상 (`PLAN_TEMPLATE_SCHEMA §3` 준수)
- **[🔀 比較]** — AI Plan vs Mate Offering 비교 (M4 이후)

### 8-3. 액션 4버튼 (Sticky Bottom)

| 버튼 | 동작 | 비용 |
|---|---|---|
| 💎 **AI Matching** | 다음 단계 (가장 강조) | 무료 |
| 💾 **저장** | Plan/Offering DB 저장, 비로그인시 로그인 유도 | 무료 |
| 🔗 **공유** | `/ai/plan/[id]` URL 생성 + 복사 | 무료 |
| 🔄 **재생성** | `user_signal` → 4o fallback (고품질) | 500pt (`ai.model.user_upgrade_cost_points`) |

재생성 버튼 클릭 시 확인 모달:
> 「500ポイントで高精度AIで作り直します。よろしいですか?」

---

## 9. 진입점 (5개)

모두 같은 `/ai/plan`으로 수렴. 상태만 다르게 초기화.

| # | 위치 | 동작 | 우선순위 | 구현 상태 |
|---|---|---|---|---|
| A | 홈 히어로 "AIに任せる" | 빈 상태 진입 | **P1** | 미구현 |
| B | `/search` 위시 → "AIに送る" | 위시 장소 사전 주입 | **P1** | 미구현 |
| C | 장소 상세 "これ含めてプラン" | 해당 장소 고정 + 조합 | P2 | 미구현 |
| E | Mate 프로필 "このメイトと相談" | Mate 지정 + 프로필 반영 | P2 | 미구현 |
| F | 공유 URL 방문자 "自分も作る" | 받은 플랜 기반 수정본 | P3 | 미구현 |

**URL 파라미터** (사전 주입):
```
/ai/plan                                 # 빈 상태 (A)
/ai/plan?wish_ids=w1,w2,w3               # 위시 주입 (B)
/ai/plan?venue_id=v123&require=true      # 장소 고정 (C)
/ai/plan?mate_id=m456                    # Mate 지정 (E)
/ai/plan?from_plan_id=p789               # 기존 플랜 참조 (F)
```

---

## 10. 모드 자동 전환 (PLAN → MATCH → GUIDE)

`AI_ROLE_DEFINITION §4-3` 단일 페이지 원칙. **사용자는 하나의 앱만 알고, 내부에서 자동 전환**.

| 현재 | 전환 트리거 | 다음 |
|---|---|---|
| **PLAN** (플랜 작성 중) | [AI Matching] 클릭 | **MATCH** (`/ai/matching`) |
| **MATCH** | ❤️ 상호 해제 + 예약 확정 | Plan 확정 (BOOKED 상태) |
| **BOOKED** | 여행 시작일 도래 + 위치 한국 | **GUIDE** (`/ai/guide`, M3 예정) |

**UI 뱃지**: 현재 모드는 상단에 작게 표시.
```
今: ✨ プラン作成中 (PLAN)
```

전환 시 부드러운 화면 전환 애니메이션 + 모드 뱃지 변경.

**매칭 정책 키** (`ADMIN_CONFIGURABLE_PARAMS §3-3` v1.3):
- `matching.approval_mode = "mutual"` — PLAN → MATCH 진입 후 ❤️ 상호 승인이 성사 조건
- `matching.initiating_mode = "bidirectional"` — Tourist·Mate 어느 쪽도 먼저 관심 표시 가능 (쌍방향)
- `matching.tier2_unlock_mode = "mutual_interest"` — Tier 2 정보는 상호 ❤️ 시 자동 해제
- `matching.tier3_unlock_mode = "booking_confirmed"` — Tier 3는 예약 확정 후 해제
- `matching.review_deadline_hours = 48` — 세션 종료 후 양쪽 리뷰 마감

---

## 11. 현행 구현 Gap (개발 체크리스트)

`app/src/app/ai-guide/page.tsx` + `api/ai-guide/route.ts` 기준.

### 🔴 구조 변경
- [ ] 파일 이동: `/ai-guide/` → `/ai/plan/`
- [ ] 네비게이션 링크 수정 (layout.tsx 등)
- [ ] **역할 토글 최상단 신규**
- [ ] 2단계(form→chat) → 3-Zone 스크롤 구조로 재설계
- [ ] 세부 설정 아코디언 (숨기기 기능)

### 🔴 데이터
- [ ] 9필드 → 5필드 축소 (Tourist)
- [ ] **Mate 모드 5필드 UI 신규 구현**
- [ ] Offering/Plan Supabase 저장 (현재 in-memory)
- [ ] 프로필 값 Override UX (Mate 지역 필드)

### 🔴 AI 백엔드
- [ ] 모델 `gpt-4o` 하드코딩 제거 → `ai.model.plan` config 읽기 (기본 `gpt-4o-mini`)
- [ ] Validator 구현 (`AI_COST_MODEL §5` Trigger A):
  - JSON schema 검증 (Zod/Ajv)
  - 필수 필드 (dates, venues, activities)
  - 이동시간 합리성 (지리 거리 계산)
  - 금지 표현 (의료법 위반)
  - 언어 일치 (langdetect)
- [ ] Fallback 로직 (`ai.model.plan_fallback_trigger = "validator_or_user_signal"`)
- [ ] Response/Prompt 캐싱 (`ai.cache.*`)
- [ ] Sampling (`ai.model.sampling_rate = 0.05`, Trigger C)

### 🔴 다국어
- [ ] 한국어 UI → 일본어 기본 + EN/KR 전환 (`locale.available`)
- [ ] AI 응답 언어 `locale.default`와 일치 (현재 "한국어로 답변" 하드코딩)
- [ ] 시스템 프롬프트 다국어 버전

### 🟡 신규 기능
- [ ] Zone 2 템플릿 갤러리 (4개 정적, 큐레이션 콘텐츠 필요)
- [ ] Zone 3 샘플 플랜 정적 렌더 (컨텐츠 제작)
- [ ] [AI Matching] 버튼 → `/ai/matching` 라우트 (별도 설계)
- [ ] 저장·공유 URL (`/ai/plan/[id]`)
- [ ] 재생성 버튼 + 500pt 차감 UX + 확인 모달

### 🟢 디테일
- [ ] Day 카드 `DESIGN.md §Color` 준수 (카테고리 컬러 11개)
- [ ] 지도 뷰 카카오맵 API 연동
- [ ] 액션 버튼 Sticky bottom
- [ ] 로딩 단계별 애니메이션 (현행 유지 OK)
- [ ] 모드 뱃지 표시 (PLAN/MATCH/GUIDE)

---

## 12. Config 파라미터

이 페이지가 runtime에 읽는 키 (`ADMIN_CONFIGURABLE_PARAMS` 확장 필요):

```
# ─── 모드·필드 (v1.3 §3-14에서 신설)
ai.planner.tourist_fields_required    = ["dates", "group_size"]
ai.planner.tourist_fields_visible     = ["dates","group_size","mate_wish","budget","note"]
ai.planner.mate_fields_required       = ["dates","pair_format","price","region","intro"]
ai.planner.mate_fields_visible        = (same as required)
ai.planner.show_detail_toggle         = true
ai.planner.detail_fields              = ["origin","interests","preferred_region","language"]

# ─── 모델 (§3-4 기존)
ai.model.plan                         = "gpt-4o-mini"
ai.model.plan_fallback_trigger        = "validator_or_user_signal"
ai.model.user_upgrade_cost_points     = 500

# ─── 샘플·갤러리
ai.planner.sample_strategy            = "static_curated"
  # enum: "static_curated" | "context_aware" | "live_generated"
ai.planner.sample_count               = 1
ai.planner.template_gallery_count     = 4
ai.planner.gallery_items              = [...]  # 콘텐츠 ID 배열

# ─── Matching 연결
ai.planner.matching_cta_enabled       = true
ai.planner.matching_route             = "/ai/matching"

# ─── Offering 2축 (v1.3 §3-3, Tier별 TTL + 동시 활성)
offerings.rookie.ttl_days             = 30
offerings.rookie.concurrent_max       = 1
offerings.verified.ttl_days           = 60
offerings.verified.concurrent_max     = 3
offerings.featured.ttl_days           = 90
offerings.featured.concurrent_max     = 5
offerings.pro_guide.ttl_days          = 180
offerings.pro_guide.concurrent_max    = 10
offerings.pro_guide.extendable        = true
offerings.applicant.allowed           = false

# ─── Matching 정책 (v1.3 §3-3)
matching.approval_mode                = "mutual"
matching.initiating_mode              = "bidirectional"
matching.tier2_unlock_mode            = "mutual_interest"
matching.tier3_unlock_mode            = "booking_confirmed"
matching.review_deadline_hours        = 48
matching.review_mandatory             = true
matching.checkin_gps_required         = true

# ─── Plan 정책 (v1.4+ 예정 — 현재 Tourist는 Plan 무제한)
# plan.max_per_tourist               = ∞
# plan.default_visibility            = "matched_only"
```

---

## 13. 미결 사항

다음 세션·구현 시 결정:

1. **모바일 vs 데스크톱 레이아웃 상세** — 데스크톱에서 Zone 2·3 2-column vs 1-column 유지?
2. **샘플 플랜 실제 콘텐츠 1개** — 큐레이션 상세 (축제·장소·시간·가격 실제 값) 필요
3. **템플릿 갤러리 4개 콘텐츠** — K뷰티·Festival·Templestay·K-POP 구체 구성
4. **시간 표시 포맷** — 24h (한국·일본 혼용) vs 12h — 일본 20~30 여성 선호 확인 필요
5. **재생성 버튼 UX 세부** — 확인 모달 vs 토스트 after-click? 포인트 부족 시 충전 안내 UI
6. **Mate 모드 지역 Override UX** — 프로필 기본값 → 편집 → 복원 버튼 배치
7. **Plan·Offering 비공개 모드** — Tier 1 `default_visibility`와 연계, UI 토글 필요 여부
8. **AI Matching 진입 시 편집 잠금** — `plan.edit_after_match_allowed = false` UX
9. **샘플에 Mate 프로필 예시** — 실제 Mate 노출 vs 가상 페르소나? 개인정보 이슈
10. **Premium 구독자 UX** — 재생성 무제한이면 포인트 안내 숨기고 뱃지만 표시
11. **Offering 가격 단위** — 세션당 vs 시간당 혼용? 사용자 혼란 방지 장치
12. **비로그인 사용자 Plan 저장** — 익명 세션 기반 저장 vs 로그인 강제?

---

## 14. 변경 영향 — 기존 문서 업데이트 필요

- [ ] **`SITEMAP_SPEC.md`** §3 — `/ai-guide` → `/ai/plan` URL 이동 반영
- [x] **`ADMIN_CONFIGURABLE_PARAMS.md`** v1.3 — §3-3에 Offering 2축·Matching 정책·Tier 필드 리스트 추가 완료 (commit 1a18ffa)
- [ ] **`ADMIN_CONFIGURABLE_PARAMS.md`** v1.4 (향후) — §3-4에 `ai.planner.*` 전용 키 추가 (필드·샘플·갤러리)
- [ ] **`SITE_COMPLETION_PLAN.md`** P1-14 — "AI Planner 구조 재편" 세부 업데이트 (5필드·토글·3-Zone·샘플)
- [ ] **`AI_ROLE_DEFINITION.md`** §3 Role 1 — "추가 필요 기능"에 역할 토글·샘플·Gallery 3항목 추가
- [ ] **`AI_TERMINOLOGY.md`** §8 — URL 구조 확정 시 `/ai/plan` 명시
- [x] **신규 작성**: `docs/MATCHING_PAGE_DESIGN.md` — AI Matching 페이지 설계 (2026-04-20 완료)

---

## 15. 관련 Config (ADMIN_CONFIGURABLE_PARAMS 크로스레퍼런스)

이 페이지 작동에 영향:

| Config 키 | 영향 |
|---|---|
| `ai.model.plan` | 기본 AI 모델 |
| `ai.model.plan_fallback_trigger` | 재시도 정책 |
| `ai.model.user_upgrade_cost_points` | [재생성] 버튼 비용 |
| `ai.planner.*` (v1.3 신설) | 필드·샘플·갤러리 |
| `ai.cache.*` | 응답 캐싱 |
| `plan.*` / `offering.*` (v1.3 §3-14) | 저장·만료·가시성 정책 |
| `matching.pair_format_allowed` | Mate 필드 2 (페어) 선택지 |
| `matching.tier1_fields` | 저장된 Plan/Offering의 공개 범위 |
| `locale.default` / `locale.available` | UI·AI 응답 언어 |
| `hero.headline_jp/en/kr` | (해당 없음, 홈 전용) |

---

## 16. Mate 등급 시스템 (Mate Level) ⭐ v0.3 신규

### 16-1. 등급 기준 (활동 실적 기반, 자동 상승)

| Level | 이름 | 진입 조건 | 자동 상승 조건 |
|-------|------|-----------|---------------|
| **Lv 1** | Rookie Mate | 신청 → 관리자 승인 | 가이드 5회 완료 + 평점 4.5 이상 |
| **Lv 2** | Verified Mate | Lv 1 달성 (자동) | 가이드 30회 + 평점 투표 20개 + 관리자 최종 승인 |
| **Lv 3** | Pro Guide | 관리자 심사 통과 | (최상위, 수동 심사) |

- 상승은 **자동 계산** (시스템이 조건 충족 감지 → 자동 승급 + 이메일 알림)
- Lv 3는 자동 불가, **관리자 최종 심사** 필수
- 등급 **강등**: 리뷰 평점 4.0 미만 3회 연속 → 한 단계 강등 (관리자 알림)

### 16-2. Tourist 관점 노출

AI 플랜 생성 후 Mate 카드에 Level 배지 표시:

```
┌─────────────────────────────────┐
│ 🤝 このプランに合うメイト         │
│                                 │
│ [PRO] 김지수 - K뷰티 전문가       │
│ ⭐ 4.9 (127 가이드 완료)          │
│ 서울 강남·홍대 전문 / 日本語 N1  │
│                                 │
│ [VER] 박민준 - 서울 로컬          │
│ ⭐ 4.7 (43 가이드 완료)           │
│ 서울 전 지역 / 日本語 中級        │
│                                 │
│ [RKY] 이하나 - 서울 대학생        │
│ ⭐ 4.5 (8 가이드 완료)            │
│ 강남·홍대 / 日本語 初級           │
└─────────────────────────────────┘
```

모든 레벨(Lv1~3) 노출, Tourist가 직접 선택.

### 16-3. v1 Mate 매칭: 티저 + 사전등록 모드

**v1.0 (현재 목표)**: 실제 매칭 기능 없음, 티저 UI만 구현.
- Mate 카드는 렌더링하되 **비활성 상태** (클릭 시 사전등록 유도)
- CTA: 「このメイトに会いたい → メイト先行登録」
- 이메일 수집 → 메이트 확보 시 우선 알림

**v1.5 (Mate 확보 후)**: 실제 매칭 기능 활성화.
- 조건: Mate DB에 최소 10명 이상 승인된 Mate 보유 시
- Config 키: `matching.v1_teaser_mode = true/false` (Admin에서 토글)

```
matching.v1_teaser_mode              = true    # v1: 티저
matching.v1_waitlist_enabled         = true    # 사전등록 수집
matching.v1_activate_threshold       = 10      # Mate 10명 시 자동 활성화
```

### 16-4. Tourist 접근 자격

| 상태 | AI Planner | Mate 매칭 | 플랜 저장 |
|------|-----------|----------|---------|
| 비로그인 | ✅ 사용 가능 | ❌ (로그인 유도) | ❌ (로그인 유도) |
| 로그인 | ✅ | ✅ (v1.5 이후) | ✅ |

자격 없음. 누구나 AI 플래너 즉시 사용 가능.

---

## 17. Offering Level 시스템 ⭐ v0.2 신규 (저녁 세션)

Mate·Tourist 모두 제안·요청 생성 시 **상세도 3단계 선택**:

| Level | 이름 | 내용 | 수정시간 | AI Planner |
|-------|------|------|----------|-----------|
| **Level 1** | 간단 (Open) | 조건만 (기간·지역·인원·관심·가격) | ~5분 | 사용 안 함 |
| **Level 2** | 중간 (Guided) | Level 1 + 추천 장소 + 대략 흐름 | ~10분 | 선택적 |
| **Level 3** | 상세 (Detailed) | Level 2 + Day-by-Day + 항목별 태그 | ~20분 | 필수 |

### 예시 (Mate Offering)
- **Level 1**: "주말 강남 일본 여성 OK, 맛집 투어, 5만원"
- **Level 2**: Level 1 + "압구정 갈비·가로수길 디저트·신사역 화장품"
- **Level 3**: Level 2 + "13:00 압구정역 미팅 → 13:15 갈비 → 14:30 도보 이동" (시간 단위)

### UX
- Level 1/2/3 선택 = §4·§5 입력 폼 5필드는 공통
- Level 2 추가 시: "+ 추천 장소" 섹션 확장
- Level 3 추가 시: "+ 일정표 (AI Planner 호출)" 전체 Day-by-Day 편집기

### 운영 함의
- 새·바쁜 Mate → Level 1 (진입 배리어 낮음)
- 열심·전문 Mate → Level 3 (AI Planner와 함께)
- Tourist도 동일 (간단 친구찾기 vs 완성 플랜)

### Config 매핑
```
offering.levels_enabled              = [1, 2, 3]
offering.level1.required_fields      = 5
offering.level2.extras               = ["recommended_venues", "rough_flow"]
offering.level3.requires_day_plan    = true
offering.level3.requires_item_tags   = true
```

---

## 17. 항목별 태그 (Level 3 전용) ⭐ v0.2 신규

Level 3 플랜의 각 시간대·장소에 4가지 태그:

| 태그 | 의미 | AI·UX 동작 |
|------|------|----------|
| ✅ **확정** | 이 시간·장소 고정 | 변경 불가, 매칭 시 정확 매치 |
| 💬 **협의** | 의견 교환 대상 | 상대방 대안 제안 가능 |
| ✨ **추천** | 제안자 의견, 변경 허용 | 위시로 표시, 바꿔도 OK |
| ❓ **미정** | 빈 항목 | 매칭 후 같이 채움 |

### 예시
```
Day 1:
  13:00 압구정역 미팅       [✅ 확정]
  13:15 갈비집 점심         [✨ 추천 — 비건 시 대체]
  15:00 카페 "OOO"         [💬 함께 정함]
  22:00 자유 시간            [❓ 아직]
```

**차별화**: Airbnb·네이버 예약에 없는 **상호작용 메리트**. Tourist·Mate 공동 설계 문화.

### Config
```
plan.item_tags.enabled         = ["confirmed", "discuss", "suggest", "empty"]
plan.item_tags.default         = "suggest"
plan.matching.confirmed_weight = 1.0   # AI 매칭 점수 가중치
plan.matching.discuss_weight   = 0.5
plan.matching.suggest_weight   = 0.3
```

---

## 18. 매칭 경로·방식 (Tourist·Mate 관점) ⭐ v0.2 신규

### Tourist 매칭 경로 3가지
1. **외부 검색** — 구글·네이버 (우리 페이지 떠남, 관여 X)
2. **AI Planner** — `/ai/plan`에서 내 플랜 작성 → AI Matching
3. **Mate 제안서 브라우징** — `/ai/matching`에서 직접 Mate Offering 고름

### 매칭 방식 4가지 (공통)
1. 하나씩 브라우징 + ❤️
2. 키워드 검색
3. AI 유사도 검색
4. 직접 간단 제안 등록 (Matching 페이지 안에서)

### Mate 탭 목적 2가지
상단 토글:
- **Looking for Tourist** — Offering 작성 (Level 1/2/3)
- **Looking for Mate** — `/ai/matching` 이동, Tourist 요청 브라우징

---

## 19. Tourist 인원 필드 확장 ⭐ v0.2 신규

§4 Tourist 5필드 중 #2 인원 세부 UX:

```
👥 인원
  ◆ 확정 인원
    👨 남 [−] 0 [+]
    👩 여 [−] 2 [+]
  ◇ 미정 인원 (올 수도 있음)
    👨 남 [−] 1 [+]
    👩 여 [−] 0 [+]
  총 확정 2명 + 미정 1명 = 최대 3명
```

**AI에게 주는 정보**:
- 확정 2명 (여2) = 반드시 이 인원 기준 (식당·Mate 페어 결정)
- 미정 남1 = 유동적 → AI가 2명·3명 양쪽 플랜 준비

**Mate 페어 포맷 영향**: 확정 여2 = FF, 확정 여2+미정 남1 = 유동적

---

## 20. 관심 프로그램 분리 ⭐ v0.2 신규

§4 Tourist 관심사를 2섹션으로:

### 20-1. 확정 사항 (Wishlist Pinned)
- 검색에서 담아온 구체 장소 칩
- 예: `[청주 직지축제 ✕] [법주사 ✕] [압구정 XX피부과 ✕]`
- AI 플랜 생성 시 **반드시 포함**

### 20-2. Explore (관심 키워드)
- 아직 결정 안 했지만 관심
- 태그 선택: 🎪축제·💄K뷰티·🏆명인·🏯템플·🛍쇼핑·🍜맛집
- 지역 태그도 선택 가능
- AI 플랜 생성 시 **추천 후보**

---

## 21. AI 플래너 진입 재설계 ⭐ v0.4 신규

### 21-1. 전략적 포지셔닝 확정

**FestiMate 정체성**: 축제·문화 여행을 오는 일본인 + 메이트 연결
**K-Beauty 위상**: 수익 엔진 (전체 유저의 ~40%가 시술 관심)

```
시술 주목적          10~15%
시술 + 관광 병행     20~25%
────────────────────
합계                 ~40%
```

K-Beauty가 주목적인 유저는 소수. 대부분은 여행하다가 시술을 추가하는 구조.
따라서 AI 플래너는 **여행으로 시작, 시술은 동등한 옵션**으로 배치.

### 21-2. 진입 화면 — "어떻게 채우고 싶으세요?"

기존 5필드 폼보다 먼저 나오는 **목적 선택 화면** (복수 선택 가능):

```
┌──────────────────────────────────────────┐
│                                          │
│     한국 여행, 어떻게 채우고 싶으세요?     │
│                                          │
│  (복수 선택 가능)                         │
│                                          │
│  🎵  K-Pop 콘서트·이벤트                 │
│  💆  K-Beauty 시술                       │
│  🍜  맛집·쇼핑·관광                      │
│  🤝  한국 메이트와 함께                   │
│                                          │
│          [ 다음 ]                        │
└──────────────────────────────────────────┘
```

선택 조합에 따라 다음 단계와 시나리오 헤드라인이 달라진다.

### 21-3. 시나리오별 헤드라인 (선택 조합 → 카피)

| 선택 조합 | 헤드라인 |
|----------|---------|
| 💆 + 🤝 | "시술도 하고 메이트도 만나고 싶다면, AI가 최적 일정을 제안해 드립니다" |
| 🎵 + 🤝 | "콘서트 일정에 맞춰, 메이트와 함께하는 한국 여행을 만들어 드립니다" |
| 💆만 | "시술 일정을 중심으로, 귀국일에 맞춘 최적 일정을 만들어 드립니다" |
| 🎵만 | "콘서트·이벤트 날짜를 중심으로 일정을 짜드립니다" |
| 🍜만 | "맛집·쇼핑 중심 한국 여행 일정을 만들어 드립니다" |
| 전체 선택 | "한국 여행의 모든 것, AI가 하나의 일정으로 연결해 드립니다" |

### 21-4. K-Beauty 선택 시 추가 Step

💆 선택한 경우에만 아래 Step이 삽입됨.

**Step A — 시술 선택**
```
┌──────────────────────────────────────────┐
│  어떤 시술을 생각하고 계신가요?            │
│                                          │
│  담아둔 시술 (자동 불러오기)              │
│  ┌─────────────┐ ┌─────────────┐        │
│  │ ✅ 울쎄라   │ │ ✅ 보톡스   │        │
│  │ 다운타임    │ │ 다운타임    │        │
│  │ 약 2주      │ │ 거의 없음   │        │
│  └─────────────┘ └─────────────┘        │
│                                          │
│  + 시술 추가하기                          │
└──────────────────────────────────────────┘
```

**Step B — AI가 시술 타이밍 최적화**

다운타임 × 여행 일수 × 메이트 희망 여부를 자동 계산.

| 패턴 | 구조 | AI 추천 조건 |
|------|------|-------------|
| **패턴 A** (시술 먼저) | Day 1 시술 → 회복 → 여행+메이트 | 다운타임 짧은 시술 |
| **패턴 B** (여행 먼저) | 여행+메이트 → Day 마지막 시술 → 귀국 | 다운타임 긴 시술 (울쎄라·써마지 등) |
| **패턴 C** (언제든) | 시술 일정 유연 | 다운타임 거의 없는 시술 (보톡스·물광주사) |

패턴 B가 핵심: 귀국 직전 시술 → 다운타임은 일본에서 회복. 여행 전체를 온전히 즐김.

### 21-5. AI 결과 화면 — 일정 카드

```
┌──────────────────────────────────────────┐
│  AI 추천 일정                             │
│  3박 4일 · 울쎄라 + 보톡스               │
└──────────────────────────────────────────┘

  📋 AI 추천 이유
  울쎄라는 다운타임이 길어 귀국 직전에 받는 것을
  추천합니다. 보톡스는 표시가 거의 없어 어느 날이든
  가능합니다.

  DAY 1  🟢 메이트 동행 가능
  └ 오후: 보톡스 (강남 OO피부과)
  └ 저녁: 메이트와 한강 등

  DAY 2  🟢 메이트 동행 가능
  └ 자유 여행 + 메이트

  DAY 3  🔴 시술일 — 메이트 미배정
  └ 오전: 울쎄라 (강남 OO피부과)
  └ 오후: 휴식

  DAY 4  🟡 회복일 — 가벼운 일정만
  └ 오전: 귀국

  [ 이 일정으로 메이트 찾기 ]
  [ 일정 수정하기 ]
  [ 클리닉에 문의하기 ]
```

**일정 색상 코드**:
- 🟢 메이트 동행 가능
- 🔴 시술일
- 🟡 회복일 (다운타임)

### 21-6. 메이트 선택 시 추가 Step

🤝 선택 + K-Beauty 없는 경우: 기존 Mate 매칭 플로우 (§7) 그대로.
🤝 선택 + K-Beauty 있는 경우: §21-5 결과 화면에서 🟢 날만 메이트 매칭.

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|------|------|------|
| 2026-04-19 | Draft v0.1 최초 작성 | Session 3 §6 이월된 "AI Planner Mate 필드·세부 마무리" 해소 — Mate 5필드 확정 (옵션 C, Tourist와 1:1 대칭) |
| 2026-04-19 (저녁) | v0.2 — Level 시스템 3단계·항목별 태그 4종·매칭 경로 3가지·인원 남녀 분리·관심 프로그램 2섹션 추가 | 저녁 Desktop 세션 사용자 통찰 반영 |
| 2026-04-20 | v0.3 — Mate 등급 시스템 (활동실적 기반·구체 수치 확정)·v1 티저+사전등록 모드·Tourist 자격 = 오픈 액세스 명시 | office-hours 세션: Tourist/Mate 여정 분석, high-involvement 여행 특성, Level 기준 결정 |
| 2026-04-21 | v0.4 — §21 AI 플래너 진입 재설계. "한국 여행 어떻게 채우고 싶으세요?" 4옵션 진입 화면. K-Beauty 전략 포지션 확정 (수익 40%, 정체성 아님). 시술 타이밍 3패턴 (A/B/C). 시나리오별 헤드라인. 다운타임 색상 코드. | 전략 재검토: 시술 진입 → 여행 진입으로 무게중심 원위치 |

---

*버전: v0.4 (Draft) / 마지막 업데이트: 2026-04-21*
