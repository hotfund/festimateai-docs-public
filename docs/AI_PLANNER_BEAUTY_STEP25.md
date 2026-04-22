# AI Planner — Step 2.5 Beauty Micro-step UX 상세

> **작성**: 2026-04-22
> **위상**: AI Planner v1.0 구현 가이드 — §3-4 "Beauty Intent 추가 Step" 심화
> **선행 문서**:
> - `AI_PLANNER_PAGE_DESIGN.md` v1.0 §3-4 (개요)
> - `AI_PLANNER_EDGE_CASES.md` v1.1 E. Beauty (EC-07, 15, 16, 17, 18)
> - `BEAUTY_PAGE_DESIGN.md` §5-5·§5-7 (시술 카드·담기 플로우)
> - `KBEAUTY_DB_SCHEMA.md` §1 procedures 테이블

---

## 0. 이 문서가 답하는 질문

AI_PLANNER_PAGE_DESIGN.md §3-4는 Step 2.5를 15줄로 요약. 실제 구현하려면 아래가 미정:

1. 시술 0개인데 beauty intent만 체크한 유저 → 뭘 보여줘야 하나? (EC-16)
2. 다운타임 섞인 시술(보톡스 0일 + 울쎄라 14일) → 어떻게 하나의 패턴으로 묶나? (EC-17)
3. `/beauty`에서 담아온 "보톡스" + 사용자가 수동 추가한 "보톡스" → 어떻게 합치나? (EC-15)
4. beauty 체크 해제하면 시술 데이터 어떻게 보존하나? (EC-07)
5. Step 2.5의 결과물은 AI 호출 payload에 어떤 구조로 들어가나?

---

## 1. 핵심 원칙

### 1-1. Step 2.5는 "데이터 수집"이 아니라 "패턴 확정"

- 시술 정보(이름, 다운타임, 가격)는 이미 `procedures` 테이블에 있음
- 사용자가 Step 2.5에서 하는 일은 **"어떤 시술들을, 여행 일정 대비 어떻게 배치할지"** 확정
- → UI는 **data entry form이 아니라 decision preview**

### 1-2. 3개 모드 분기

| 모드 | 진입 조건 | 주요 과제 |
|------|----------|----------|
| **Cart 모드** | `/beauty`에서 시술 담아옴 (wish_ids에 clinic 타입) | 담은 시술 확정 + 다운타임 패턴 제안 |
| **Manual 모드** | 사용자가 "+ 시술 추가" 직접 | 시술 검색 → 선택 → 패턴 제안 |
| **Discovery 모드** | beauty 체크만, 시술 0개 (EC-16) | 여행 일정 기반 후보 시술 AI 추천 |

세 모드는 같은 Step 2.5 화면에서 **상단 배너·본문 레이아웃만 바뀜**.

### 1-3. 사용자가 고르지 않는 것

- **다운타임 패턴 A/B/C**: AI가 자동 산정 → 사용자는 **확인만**
- **시술 타이밍 (Day N)**: Step 2.5에서 안 정함 → 최종 AI 플랜 결과에 반영
- **클리닉**: Step 2.5에서 안 고름 → AI Matching 이후 단계

---

## 2. 상태 모델

```typescript
interface BeautyMicroState {
  // ── 활성화 ───────────────────────────
  active: boolean;              // beauty intent 체크 여부
  mode: 'cart' | 'manual' | 'discovery';
  
  // ── 담긴 시술 ────────────────────────
  treatments: BeautyTreatment[];
  
  // ── 패턴 (AI 산정, 사용자 승인) ──────
  primaryPattern: 'A' | 'B' | 'C' | null;
  exceptions: TreatmentException[];
  userApprovedPattern: boolean;    // false면 AI 호출 차단
  
  // ── Discovery 모드 전용 ──────────────
  discoveryHints?: {
    travelDays: number;
    returnDate: string;
    candidateCategories: string[];
  };
  
  // ── 유실 방지 (EC-07) ────────────────
  hiddenDraft?: BeautyTreatment[];   // intent off 시 treatments를 여기로 이동
}

interface BeautyTreatment {
  id: string;                    // 내부 uuid
  procedureId: string;           // procedures.id (canonical)
  variant?: string;              // "사각턱", "광대" 등
  source: 'wishlist' | 'manual' | 'discovery_ai';
  priority: 'must' | 'optional'; // 사용자가 고침
  downtimeDays: number;          // procedures.downtime_days_max 복사 (snapshot)
  flightRestrictionHours: number;
  estPriceJpy?: number;
  addedAt: string;
}

interface TreatmentException {
  treatmentId: string;
  reason: 'longer_downtime' | 'flight_restriction' | 'pre_care_required';
  recommendation: string;         // "귀국 직전 권장"
}
```

---

## 3. 모드별 UX

### 3-1. Cart 모드 — `/beauty`에서 담아옴 (가장 일반적)

**진입 조건**: `entrySource=wishlist` + wish_ids에 `procedure` 타입 1개 이상

**화면**:

```
┌────────────────────────────────────────────────┐
│  💆 담으신 시술 3개                              │
│  (K-Beauty 페이지에서 선택하신 내용)              │
├────────────────────────────────────────────────┤
│  ✓ 보톡스 사각턱      다운타임 1-2일   ¥8,000~  │
│    [우선도: ◉ 필수  ○ 선택]           [삭제]    │
│                                                │
│  ✓ 울쎄라              다운타임 7-14일  ¥50,000~│
│    [우선도: ◉ 필수  ○ 선택]           [삭제]    │
│                                                │
│  ✓ 스킨부스터 1회     다운타임 2-3일   ¥15,000~ │
│    [우선도: ○ 필수  ◉ 선택]           [삭제]    │
│                                                │
│  [+ 시술 더 추가]                                │
├────────────────────────────────────────────────┤
│  🤖 AI 일정 배치 제안                             │
│  ─────────────────────────                     │
│  주 패턴: B (여행 먼저 → 마지막에 시술)           │
│                                                │
│  • 보톡스 + 스킨부스터: 같은 날 Day 1 가능        │
│  • 울쎄라: 귀국 1일 전 받기 (다운타임 일본 회복)  │
│  • 비행 제한 없음 확인됨                          │
│                                                │
│  [✓ 이 제안으로 진행]   [패턴 다시 계산]          │
└────────────────────────────────────────────────┘
```

**인터랙션**:
- 상단 시술 목록: 삭제·우선도 변경 시 하단 AI 제안 재계산 (debounce 500ms)
- "+ 시술 더 추가" → 모달 (Manual 모드 검색 UI 재사용)
- 패턴 승인 전에는 Step 3 (Base Input) CTA 비활성화

---

### 3-2. Manual 모드 — 시술 검색·추가

**진입 조건**: Cart 모드 상태에서 "+ 시술 더 추가" / Discovery 모드에서 "직접 선택"

**화면 (모달 or 인라인 확장)**:

```
┌────────────────────────────────────────────────┐
│  🔍 시술 추가                      [닫기 ✕]     │
├────────────────────────────────────────────────┤
│  [ 시술명, 카테고리, 고민 검색... ]             │
│                                                │
│  인기 카테고리:                                  │
│  [피부과] [리프팅] [헤어] [네일] [치과]          │
├────────────────────────────────────────────────┤
│  검색 결과 / 추천                               │
│                                                │
│  ┌──────────────────────────────────────┐    │
│  │ 물광주사                               │    │
│  │ 피부과 · 다운타임 0일 · ¥25,000~      │    │
│  │ [+ 담기]                               │    │
│  └──────────────────────────────────────┘    │
│  ...                                            │
└────────────────────────────────────────────────┘
```

**Canonical dedupe (EC-15)**:
- 담기 시점에 `procedureId` 비교
- 이미 있으면: "이 시술은 이미 담겨 있습니다. 덮어쓰시겠습니까?" 토스트
- variant 다르면 (예: 보톡스 사각턱 vs 보톡스 광대) → variant로 병합 가능 여부 확인

---

### 3-3. Discovery 모드 — beauty 체크만, 시술 0개 (EC-16)

**진입 조건**: beauty intent 체크, treatments 비어 있음, wish_ids에 clinic 타입 없음

**화면**:

```
┌────────────────────────────────────────────────┐
│  💆 어떤 시술이 궁금하세요?                      │
│                                                │
│  아직 구체적인 시술을 정하지 않으셨네요.         │
│  여행 일정을 기준으로 후보를 추천해드릴게요.     │
├────────────────────────────────────────────────┤
│  관심 영역 (복수 선택):                          │
│  ☑ 피부 (보톡스·필러·물광)                      │
│  ☐ 리프팅 (울쎄라·써마지)                       │
│  ☐ 헤어 (펌·시술)                                │
│  ☐ 네일·속눈썹                                   │
│  ☐ 치과 (미백·교정)                              │
│                                                │
│  고민 키워드 (선택):                             │
│  [ 예: 붓기, 탄력, 잡티... ]                    │
├────────────────────────────────────────────────┤
│  [🤖 AI가 후보 3개 추천]                         │
│  or                                             │
│  [직접 선택하러 가기 →]  ← Manual 모드로 전환    │
└────────────────────────────────────────────────┘
```

**AI 호출 시 (선택적)**:
- payload: `{ travelDays, returnDate, categories, keywords }`
- 응답: `[{ procedureId, rationale, downtimeDays, estPrice }, ...]`
- 사용자가 [이 제안으로 진행] 클릭 → Cart 모드로 승격

**주의**:
- Discovery 모드 AI 호출은 **AI Planner 본 호출과 별개의 가벼운 추천 호출** (EC-20 타임아웃 15초)
- 실패 시 → "직접 선택하러 가기" 버튼만 남김

---

## 4. 다운타임 패턴 자동 산정 (EC-17 구현)

### 4-1. 입력

```typescript
interface PatternInput {
  treatments: BeautyTreatment[];
  travelDays: number;            // 여행 일수
  returnDate: string;            // 귀국일
  flightHours: number;           // 비행 소요 (일본↔한국 ~2.5h)
}
```

### 4-2. 알고리즘 (클라이언트 측 pure function)

```typescript
function computePattern(input: PatternInput): PatternResult {
  // 1. 가장 긴 다운타임 기준 primary 결정
  const maxDowntime = Math.max(...input.treatments.map(t => t.downtimeDays));
  
  let primary: 'A' | 'B' | 'C';
  if (maxDowntime <= 1) primary = 'C';        // 언제든 OK
  else if (maxDowntime <= 3) primary = 'A';   // 시술 먼저 → 회복 → 여행
  else primary = 'B';                         // 여행 먼저 → 귀국 직전 시술
  
  // 2. exceptions 산정
  const exceptions: TreatmentException[] = input.treatments
    .filter(t => needsExceptionHandling(t, primary, input))
    .map(t => ({
      treatmentId: t.id,
      reason: decideReason(t, input),
      recommendation: buildRecommendation(t, primary, input),
    }));
  
  // 3. same-day bundle 후보
  const sameDayBundles = findSameDayBundles(input.treatments);
  
  // 4. 비행 제한 경고
  const flightWarnings = input.treatments
    .filter(t => t.flightRestrictionHours > input.flightHours);
  
  return { primary, exceptions, sameDayBundles, flightWarnings };
}
```

### 4-3. 출력 예시 (보톡스 + 울쎄라 + 스킨부스터)

```
primary: 'B'                       // 울쎄라 14일 기준
exceptions: [
  { treatmentId: 울쎄라,
    reason: 'longer_downtime',
    recommendation: '귀국 1일 전 받기 권장 (다운타임 일본에서 회복)' }
]
sameDayBundles: [
  [보톡스, 스킨부스터]             // 둘 다 다운타임 ≤ 3일 → Day 1 함께
]
flightWarnings: []
```

### 4-4. UI 표시 규칙

- primary 패턴은 **한 줄 요약**으로만 노출 ("B 패턴: 여행 먼저")
- exceptions는 **시술별 배지**로 표시 (🔔 "귀국 직전 권장")
- sameDayBundles는 **그룹핑 박스**로 시각화 ("⊕ 같은 날 가능")
- 사용자 액션: `[✓ 이 제안으로 진행]` / `[패턴 다시 계산]` / `[무시하고 직접 배치]`

---

## 5. Beauty off → 데이터 보존 (EC-07)

### 5-1. 해제 시 처리

```typescript
function onBeautyIntentOff(state: BeautyMicroState): BeautyMicroState {
  return {
    ...state,
    active: false,
    hiddenDraft: state.treatments,   // 통째로 이동
    treatments: [],
    primaryPattern: null,
    exceptions: [],
    userApprovedPattern: false,
  };
}
```

### 5-2. 재체크 시 복원

```typescript
function onBeautyIntentOn(state: BeautyMicroState): BeautyMicroState {
  if (state.hiddenDraft && state.hiddenDraft.length > 0) {
    return {
      ...state,
      active: true,
      treatments: state.hiddenDraft,
      hiddenDraft: undefined,
      // 패턴은 재계산 필요
      primaryPattern: null,
      userApprovedPattern: false,
    };
  }
  return { ...state, active: true, mode: 'discovery' };
}
```

### 5-3. 완전 삭제 조건

- 세션 종료 (sessionStorage 만료)
- 사용자가 명시적으로 "시술 전체 초기화" 버튼 클릭
- draft를 다른 사용자 계정으로 전환 (로그인 변경)

---

## 6. AI 호출 Payload 계약

Step 2.5 결과물은 canonical payload에 다음과 같이 포함:

```json
{
  "intent": ["beauty", "festival"],
  "beauty": {
    "treatments": [
      { "procedureId": "uuid-보톡스",
        "variant": "사각턱",
        "priority": "must",
        "source": "wishlist",
        "downtimeSnapshot": 2 },
      { "procedureId": "uuid-울쎄라",
        "priority": "must",
        "source": "wishlist",
        "downtimeSnapshot": 14 },
      { "procedureId": "uuid-스킨부스터",
        "priority": "optional",
        "source": "wishlist",
        "downtimeSnapshot": 3 }
    ],
    "primaryPattern": "B",
    "exceptions": [
      { "treatmentId": "uuid-울쎄라",
        "reason": "longer_downtime",
        "recommendation": "귀국 1일 전" }
    ],
    "sameDayBundles": [
      ["uuid-보톡스", "uuid-스킨부스터"]
    ],
    "userApprovedPattern": true,
    "mode": "cart"
  }
}
```

**필드 설계 원칙**:
- `mixed` 같은 derived flag는 **포함 안 함** (EC-06)
- `downtimeSnapshot`은 요청 시점 값 고정 (나중에 DB 업데이트돼도 이 요청은 snapshot 유지)
- `procedureId`는 canonical UUID, variant는 메타
- `userApprovedPattern: false`면 서버가 400 반환 (AI 호출 방지)

---

## 7. Edge Case 매핑 (구현 체크리스트)

| EC | 적용 위치 | 이 문서 참조 |
|----|----------|--------------|
| EC-07 Beauty on/off soft delete | §5 hiddenDraft 패턴 | 복원 로직 구현 |
| EC-15 Canonical dedupe | §3-2 Manual 모드 담기 | procedureId 비교 모달 |
| EC-16 Discovery mode | §3-3 전체 | travelDays 기반 추천 호출 |
| EC-17 Primary + exceptions | §4 알고리즘 | `computePattern` pure function |
| EC-18 Auto-check on wishlist prefill | 진입 로직 | wish_ids→beauty intent 자동 체크 토스트 |

---

## 8. 구현 Phase 매핑 (EDGE_CASES §구현 순서)

| Phase | 이 문서 작업 |
|-------|-------------|
| **A (상태 모델)** | §2 타입 정의 + reducer, §4 `computePattern` pure function, §5 hidden draft 전환 |
| **B (보호 UX)** | §3-1 패턴 승인 CTA 잠금, §3-3 Discovery 실패 fallback, §5 재체크 복원 UI |
| **C (서버 계약)** | §6 canonical payload 스키마 (zod), `userApprovedPattern` 서버 검증 |
| **D (QA 회귀)** | §9 테스트 시나리오 자동화·수동 체크 |

---

## 9. QA 테스트 시나리오 (Step 2.5 전용)

### Unit (Vitest)
- `computePattern([보톡스 0일, 울쎄라 14일, 스킨부스터 3일])` → primary='B', exceptions에 울쎄라
- `computePattern([보톡스 0일])` → primary='C'
- canonical dedupe: `{procedureId:'X', variant:'사각턱'}` + `{procedureId:'X', variant:'사각턱'}` → 1개 유지
- canonical dedupe: `{procedureId:'X', variant:'사각턱'}` + `{procedureId:'X', variant:'광대'}` → 2개 유지

### Integration (React Testing Library)
- Cart 모드 진입 → 시술 3개 렌더링 → 1개 삭제 → 패턴 재계산 호출 확인
- beauty 체크 해제 → treatments 비워짐 + hiddenDraft에 보존 확인
- beauty 재체크 → treatments 복원 + `userApprovedPattern=false` 확인
- Discovery 모드 "AI 후보 추천" 실패 → "직접 선택하러 가기" CTA만 남는지 확인

### E2E (Playwright)
- `/beauty`에서 시술 2개 담기 → `/ai/plan` 진입 → Step 2.5 자동 표시 + 자동 beauty intent 체크 확인
- 패턴 승인 전 [다음] 버튼 비활성화 상태 확인
- 패턴 승인 후 Step 3 진입 가능 확인

### Manual
- [ ] Cart 모드에서 시술 우선도 변경 시 AI 패턴 재계산 (debounce 500ms)
- [ ] same-day bundle 그룹핑 박스가 시각적으로 구분되는지 (iPhone Safari)
- [ ] Discovery 모드 AI 추천 실패 시 사용자 당황 없는지
- [ ] 비행 제한 시술(예: 눈 시술 24h) 경고 배너 노출 확인
- [ ] hiddenDraft 복원 시 `userApprovedPattern` 리셋되어 다시 승인 유도되는지

---

## 10. Config 연결 (ADMIN_CONFIGURABLE_PARAMS)

```
# 패턴 산정 임계값
beauty.pattern.downtime_threshold_c     = 1      # ≤ 1일 → C
beauty.pattern.downtime_threshold_a     = 3      # ≤ 3일 → A, 초과 → B
beauty.pattern.same_day_max_downtime    = 3      # 같은 날 bundle 가능 상한

# 비행 제한
beauty.flight.default_japan_korea_hours = 2.5

# Discovery 모드
beauty.discovery.candidate_count        = 3      # AI 추천 후보 개수
beauty.discovery.timeout_seconds        = 15     # 본 Planner 호출보다 짧음
beauty.discovery.enabled                = true   # 기능 토글

# Dedupe
beauty.dedupe.merge_on_variant_conflict = "confirm"   # "auto" | "confirm" | "keep_both"
```

---

## 11. Open Questions (Phase 2 이후)

1. **variant의 canonical 수준 어디까지?**
   - 현재: procedureId + variant 문자열
   - Phase 2: procedure_variants 테이블 정규화 검토

2. **시술 간 상호작용 (medical contraindication)**
   - 예: 울쎄라 후 2주 내 레이저 금지
   - Phase 1: UI 경고만, Phase 2: AI가 자동 제거·재배치

3. **재시술 간격 (procedures.re_treatment_interval_days)**
   - 현재 여행에선 대부분 의미 없음 (단일 방문)
   - Phase 2: 재방문 유저 추천 시 활용

4. **클리닉 선택과의 결합**
   - Step 2.5는 시술 레벨만 확정
   - 클리닉은 AI Planner 결과 → AI Matching → 사용자 선택 단계
   - 경계 명확히 유지

---

## 12. 관련 문서

- `AI_PLANNER_PAGE_DESIGN.md` v1.0 §3-4 — Step 2.5 개요 + 3패턴 정의
- `AI_PLANNER_EDGE_CASES.md` v1.1 §E — Beauty 5개 edge case
- `BEAUTY_PAGE_DESIGN.md` v0.4 §5-5·§5-7 — 시술 카드·담기 플로우
- `KBEAUTY_DB_SCHEMA.md` §1 — procedures 테이블 (downtime, flight_restriction, pre_care)
- `ADMIN_CONFIGURABLE_PARAMS.md` v1.3 §3-9 — K-Beauty config

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|------|------|------|
| 2026-04-22 | v1.0 최초 작성 | EC-07/15/16/17/18 구체화, 3모드(Cart/Manual/Discovery) 분기, primary+exceptions 알고리즘 |

---

*버전: v1.0 / 마지막 업데이트: 2026-04-22*
