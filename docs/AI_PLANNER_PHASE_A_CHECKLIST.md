# AI Planner Phase A — 구현 체크리스트

> **작성**: 2026-04-23
> **위상**: `AI_PLANNER_BEAUTY_STEP25.md` v1.2 §14 Phase A 산출물 → 실제 코드 전환 체크리스트
> **목적**: Genspark 검토용 초안. 확정 후 CLI에서 Phase A 착수.
> **스택 전제**: Next.js 16 (App Router) + React 19 + TypeScript 5 + Supabase. 별도 state 라이브러리 없음.

---

## 0. Phase A 범위 (Scope)

### 0-1. 포함

Phase A는 **"프론트 상태 모델만 먼저 고정"**이 목표. UI·서버 호출·AI 프롬프트는 Phase B~C.

- [ ] TypeScript 타입 정의 (`BeautyMicroState`, `PlanDraft`, `TimingPattern` 등)
- [ ] Pure functions: `computePattern`, `resolveEntrySource`, `canonicalDedupe`
- [ ] Reducer + Action 스키마
- [ ] Session storage 레이어 (draft 저장·복원·draftVersion)
- [ ] `hiddenDraft` 전환 로직
- [ ] `resultStale` 마킹 로직
- [ ] 상태 머신 키 7개 derived selectors
- [ ] Feature flag 골격 (`tourist_beauty_only`)
- [ ] Vitest 테스트 환경 구성
- [ ] Unit 테스트: reducer + pure function 전부

### 0-2. 미포함 (Phase B~D)

- UI 컴포넌트 렌더링
- 실제 Step 2.5 화면
- Server route handler
- AI 호출
- Supabase 연동
- Authentication 연동
- E2E 테스트

### 0-3. Feature flag 전제

- 초기 scope: **Tourist + Beauty 경로만**
- Mate 모드, Festival-only 분기는 구현은 하되 flag off
- 이유: scope 통제로 Phase A 완료 기한 단축, 규모 축소

---

## 1. 파일 구조 제안

```
app/src/
├── features/
│   └── ai-planner/                    ← 신규 디렉토리
│       ├── types.ts                   ← 모든 타입 SSOT
│       ├── state/
│       │   ├── reducer.ts
│       │   ├── actions.ts
│       │   ├── selectors.ts
│       │   └── initial-state.ts
│       ├── logic/
│       │   ├── compute-pattern.ts
│       │   ├── resolve-entry-source.ts
│       │   ├── canonical-dedupe.ts
│       │   └── __tests__/             ← Vitest
│       │       ├── compute-pattern.test.ts
│       │       ├── resolve-entry-source.test.ts
│       │       └── canonical-dedupe.test.ts
│       ├── storage/
│       │   ├── session-draft.ts       ← sessionStorage 래퍼
│       │   ├── draft-version.ts
│       │   └── __tests__/
│       │       └── session-draft.test.ts
│       ├── feature-flags.ts
│       └── __tests__/
│           └── reducer.test.ts
└── lib/
    └── plan-types.ts                  ← 기존 (결과 포맷, Phase A에서 건드리지 않음)
```

**결정 근거**:
- `features/` 폴더 패턴 채택: Next.js App Router는 라우트만 `app/`에 두고 feature 코드는 분리
- `state/` `logic/` `storage/` 3-way 분리: testability + dependency direction 명시
- `__tests__/` 동일 디렉토리: Vitest 관례

---

## 2. 타입 정의 (`features/ai-planner/types.ts`)

### 2-1. 외부 enum

```typescript
// entry source (EC-01, EC-02)
export type EntrySource = 
  | 'home' 
  | 'wishlist' 
  | 'venue' 
  | 'mate_profile' 
  | 'shared';

export const ENTRY_SOURCE_PRIORITY: EntrySource[] = [
  'mate_profile', 'venue', 'wishlist', 'shared', 'home'
];

// intent (v1.0 확정)
export type Intent = 'festival' | 'beauty' | 'food_shopping' | 'mate';

// role (v1.0)
export type PlannerRole = 'tourist' | 'mate';

// timing pattern (v1.1: auto 추가, semantic enum)
export type TimingPattern = 
  | 'procedure_first'   // 내부 A
  | 'travel_first'      // 내부 B
  | 'anytime'           // 내부 C
  | 'auto';             // 서버가 결정

// internal pattern code (computePattern 출력)
export type RecommendedPattern = 'A' | 'B' | 'C';

// beauty micro-step mode
export type BeautyMode = 'cart' | 'manual' | 'discovery';

// plan depth (v1.0)
export type PlanDepth = 'open' | 'guided' | 'detailed';
```

### 2-2. 도메인 인터페이스

```typescript
export interface BeautyTreatment {
  id: string;                    // 내부 uuid (클라 생성)
  procedureId: string;           // procedures.id canonical
  variant?: string;              // "사각턱", "광대" 등
  source: 'wishlist' | 'manual' | 'discovery_ai';
  priority: 'must' | 'optional';
  downtimeSnapshot: number;      // procedures.downtime_days_max 복사
  flightRestrictionHours: number;
  estPriceJpy?: number;
  unknownDowntime: boolean;      // V-BEA-03
  addedAt: string;               // ISO
}

export interface TreatmentException {
  treatmentId: string;
  reason: 'longer_downtime' | 'flight_restriction' | 'pre_care_required';
  recommendation: string;        // 한국어 1줄
}

export interface BeautyMicroState {
  active: boolean;
  mode: BeautyMode;
  
  treatments: BeautyTreatment[];
  
  timingPattern: TimingPattern;
  recommendedPattern: RecommendedPattern | null;
  exceptions: TreatmentException[];
  sameDayBundles: string[][];    // treatmentId[][]
  hasMixedDowntime: boolean;
  
  discoveryHints?: {
    travelDays: number;
    returnDate: string;
    candidateCategories: string[];
  };
  
  hiddenDraft?: BeautyTreatment[];  // intent off 시 treatments 이동지
}

export interface TouristBaseInput {
  dateStart: string | null;
  dateEnd: string | null;
  peopleConfirmedMale: number;
  peopleConfirmedFemale: number;
  peopleTentativeMale: number;
  peopleTentativeFemale: number;
  mateOption: 'with' | 'ai_only' | 'later';
  budgetJpy: 'any' | '50k' | '50_100k' | '100k_plus';
  freeNote: string;              // ≤ 140 chars
}

export interface PlanDraft {
  // 진입
  entrySource: EntrySource;
  rawEntrySource: string;        // 원본 (analytics용, EC-02)
  entryConflictResolved?: string;
  
  // 역할
  role: PlannerRole;
  
  // 의도
  intents: Intent[];
  
  // 입력
  touristInput: TouristBaseInput;
  beauty: BeautyMicroState;
  
  // depth
  targetDepth: PlanDepth;
  fulfilledDepth: PlanDepth;
  
  // 편집 충돌·stale (v1.1)
  draftVersion: number;
  lastEditedAt: string;
  resultStale: boolean;
  
  // 세션 식별
  sessionId: string;
}

// Derived (selector 결과)
export interface PlannerStateKeys {
  intentSelected: boolean;
  beautyDraftEmpty: boolean;
  beautyDraftPrefilled: boolean;
  timingPatternManual: boolean;
  timingPatternAuto: boolean;
  plannerReady: boolean;
  resultStale: boolean;
}
```

### 2-3. 질문 (Genspark 검토)

- Q1. `BeautyTreatment.id`와 `procedureId` 분리가 맞나? (`id`는 클라 인스턴스 ID, `procedureId`는 마스터 DB ID)
- Q2. `TouristBaseInput`은 모든 인원 필드를 4개로 분리하는 게 맞나, 아니면 구조화 객체가 나은가?
- Q3. `PlanDraft.sessionId`를 통해 Supabase 저장까지 고려한 키로 쓸 수 있는가?

---

## 3. Pure Functions

### 3-1. `computePattern` (`logic/compute-pattern.ts`)

```typescript
interface PatternInput {
  treatments: BeautyTreatment[];
  travelDays: number;
  returnDate: string;
  flightHours: number;           // default 2.5
  config: {
    thresholdC: number;          // default 1
    thresholdA: number;          // default 3
    sameDayMaxDowntime: number;  // default 3
    longDowntimeDays: number;    // default 7 (V-TIME-02)
    shortDowntimeDays: number;   // default 1 (V-TIME-03)
  };
}

interface PatternResult {
  recommendedPattern: RecommendedPattern;
  exceptions: TreatmentException[];
  sameDayBundles: string[][];
  flightWarnings: string[];      // treatmentId[]
  hasMixedDowntime: boolean;
}

export function computePattern(input: PatternInput): PatternResult;
```

**결정론적 알고리즘** — 테스트 가능한 pure function. 외부 의존성 0.

### 3-2. `resolveEntrySource` (`logic/resolve-entry-source.ts`)

```typescript
interface EntryParams {
  entrySource?: string;
  wishIds?: string[];
  venueId?: string;
  mateId?: string;
  shareId?: string;
}

interface EntryResolution {
  resolved: EntrySource;
  driving: 'mate_profile' | 'venue' | 'wishlist' | 'shared' | 'home';
  secondary: string[];           // 보조 컨텍스트 IDs
  conflictLog?: string;          // analytics
  invalidValue?: string;         // EC-02
}

export function resolveEntrySource(params: EntryParams): EntryResolution;
```

**우선순위**: `mate_profile > venue > wishlist > shared > home` (EC-01)

### 3-3. `canonicalDedupe` (`logic/canonical-dedupe.ts`)

```typescript
type DedupeMode = 'auto' | 'confirm' | 'keep_both';

interface DedupeResult {
  merged: BeautyTreatment[];
  conflicts: Array<{
    procedureId: string;
    variants: string[];          // 충돌된 variant들
  }>;
}

export function canonicalDedupe(
  existing: BeautyTreatment[],
  incoming: BeautyTreatment,
  mode: DedupeMode
): DedupeResult;
```

**규칙** (EC-15):
- `procedureId` + `variant` 동일 → 1개 유지 (기존 우선)
- `procedureId` 같고 `variant` 다름 → `mode` 따라 분기
- `confirm` 모드에서는 `conflicts`로 반환 (UI에서 모달 처리)

### 3-4. 질문 (Genspark 검토)

- Q4. `computePattern` 입력에 `config`를 받는 게 맞나, 아니면 모듈 상수로 처리해야 하나? (admin config 런타임 변경 고려)
- Q5. `flightHours` default 2.5h는 일본-한국 고정값으로 충분한가?
- Q6. `canonicalDedupe`의 `mode` enum이 올바른가? `merge` 같은 다른 모드 필요한가?

---

## 4. Reducer + Actions

### 4-1. Action 스키마 (`state/actions.ts`)

```typescript
export type PlannerAction =
  // ── 진입 ──
  | { type: 'ENTRY_RESOLVED'; payload: EntryResolution }
  
  // ── 역할 ──
  | { type: 'ROLE_TOGGLE'; payload: { role: PlannerRole; confirmed: boolean } }
  
  // ── Intent ──
  | { type: 'INTENT_ADD'; payload: Intent }
  | { type: 'INTENT_REMOVE'; payload: Intent }
  | { type: 'INTENT_TOGGLE'; payload: Intent }
  
  // ── Beauty Micro ──
  | { type: 'BEAUTY_INTENT_ON' }
  | { type: 'BEAUTY_INTENT_OFF' }
  | { type: 'BEAUTY_TREATMENT_ADD'; payload: BeautyTreatment }
  | { type: 'BEAUTY_TREATMENT_REMOVE'; payload: { id: string } }
  | { type: 'BEAUTY_TREATMENT_PRIORITY'; payload: { id: string; priority: 'must' | 'optional' } }
  | { type: 'BEAUTY_MODE_SET'; payload: BeautyMode }
  | { type: 'BEAUTY_PATTERN_COMPUTED'; payload: PatternResult }
  | { type: 'BEAUTY_TIMING_SET'; payload: TimingPattern }
  | { type: 'BEAUTY_DISCOVERY_HINTS_SET'; payload: BeautyMicroState['discoveryHints'] }
  
  // ── Tourist Input ──
  | { type: 'TOURIST_INPUT_PATCH'; payload: Partial<TouristBaseInput> }
  
  // ── Depth ──
  | { type: 'DEPTH_TARGET_SET'; payload: PlanDepth }
  | { type: 'DEPTH_FULFILLED_SET'; payload: PlanDepth }
  
  // ── 편집·stale ──
  | { type: 'DRAFT_VERSION_BUMP' }
  | { type: 'RESULT_STALE_SET'; payload: boolean }
  
  // ── 복원·리셋 ──
  | { type: 'DRAFT_HYDRATE'; payload: PlanDraft }
  | { type: 'DRAFT_RESET' };
```

### 4-2. Reducer 동작 요약 (`state/reducer.ts`)

| Action | 주요 효과 | 결합된 Edge Case |
|--------|---------|----------------|
| `ENTRY_RESOLVED` | entrySource 고정, rawEntrySource 저장 | EC-01, EC-02, EC-03 |
| `INTENT_ADD`/`REMOVE`/`TOGGLE` | intents[] 갱신, draftVersion++ | EC-05, EC-06 |
| `BEAUTY_INTENT_OFF` | treatments → hiddenDraft, timingPattern='auto', resultStale=true | EC-07 |
| `BEAUTY_INTENT_ON` | hiddenDraft 있으면 복원, 없으면 mode='discovery' | EC-07 |
| `BEAUTY_TREATMENT_ADD` | canonicalDedupe 호출, draftVersion++ | EC-15 |
| `BEAUTY_PATTERN_COMPUTED` | recommendedPattern 설정 (사용자 아직 미선택) | EC-17 |
| `BEAUTY_TIMING_SET` | timingPattern 저장. 'auto'는 명시 허용 | v1.1 auto |
| `ROLE_TOGGLE` | 현재 draft 저장 후 전환. Phase A에선 Tourist만 지원 | EC-10, EC-12 |
| `TOURIST_INPUT_PATCH` | partial merge, draftVersion++, 결과 있으면 resultStale=true | EC-13 |
| `DEPTH_FULFILLED_SET` | targetDepth 대비 fulfilledDepth만 변경 | EC-13 |
| `DRAFT_HYDRATE` | sessionStorage → state 복원 | EC-21, EC-23 |

**원칙**:
- 모든 input-changing action은 `draftVersion++` + `lastEditedAt = now()`
- 결과 이후 편집은 `resultStale = true`
- **삭제보다 숨김** — `BEAUTY_INTENT_OFF`/`DEPTH_FULFILLED_SET` 다운그레이드는 데이터 보존

### 4-3. Selectors (`state/selectors.ts`)

```typescript
export function selectStateKeys(draft: PlanDraft): PlannerStateKeys {
  return {
    intentSelected: draft.intents.length >= 1,
    beautyDraftEmpty: draft.beauty.active && draft.beauty.treatments.length === 0,
    beautyDraftPrefilled: draft.beauty.treatments.some(t => t.source === 'wishlist'),
    timingPatternManual: draft.beauty.timingPattern !== 'auto',
    timingPatternAuto: draft.beauty.timingPattern === 'auto',
    plannerReady: computePlannerReady(draft),
    resultStale: draft.resultStale,
  };
}

export function selectBeautyMixed(beauty: BeautyMicroState): boolean {
  return beauty.treatments.length >= 2;
}

export function selectOverLimitTreatments(beauty: BeautyMicroState, max: number): boolean {
  return beauty.treatments.length > max;
}

export function selectCanProceedFromStep25A(beauty: BeautyMicroState, max: number): boolean {
  // V-BEA-04 v1.2: 6개 이상이면 block_until_user_reduces
  return beauty.treatments.length <= max;
}
```

### 4-4. 질문 (Genspark 검토)

- Q7. Action 개수(~22개)가 너무 많은지? 일부 합쳐야 하나? (예: BEAUTY_* 묶음)
- Q8. `DRAFT_VERSION_BUMP`를 별도 action으로 두는 게 맞나, 아니면 모든 mutating action이 자동 bump하는 게 맞나?
- Q9. Selector 위치 — `state/selectors.ts`가 맞나, 아니면 `lib/`로 분리?

---

## 5. Storage Layer (`storage/session-draft.ts`)

### 5-1. sessionStorage 키 스키마

```typescript
const KEYS = {
  draft: (sessionId: string, role: PlannerRole) => 
    `festimate.planner.draft.${role}.${sessionId}`,
  activeSession: 'festimate.planner.active_session',
  version: 'festimate.planner.schema_version',
} as const;

const SCHEMA_VERSION = 1;
```

**결정**:
- role별 draft 분리 (EC-10, EC-12)
- `activeSession`으로 현재 세션 ID 추적
- `schema_version`로 마이그레이션 대비

### 5-2. API

```typescript
export function saveDraft(draft: PlanDraft): void;
export function loadDraft(sessionId: string, role: PlannerRole): PlanDraft | null;
export function deleteDraft(sessionId: string, role: PlannerRole): void;
export function listDrafts(): Array<{ sessionId: string; role: PlannerRole; lastEditedAt: string }>;

// auto-save debounce (Phase A에선 호출 주체만 정의, 실제 throttle은 Phase B)
export function saveDraftThrottled(draft: PlanDraft, delayMs: number): void;
```

### 5-3. 질문 (Genspark 검토)

- Q10. iOS Safari private mode에서 sessionStorage 용량 제한 시 fallback 전략? (in-memory only?)
- Q11. `deleteDraft`는 사용자 명시 삭제에만 사용. 자동 삭제 조건을 여기서 처리해야 하나?

---

## 6. Feature Flags (`feature-flags.ts`)

```typescript
export interface PlannerFeatureFlags {
  touristBeautyOnlyScope: boolean;   // Phase A 기본 true
  mateRoleEnabled: boolean;          // Phase A 기본 false
  festivalIntentEnabled: boolean;    // Phase A 기본 true (beauty와 공존 가능)
  foodShoppingIntentEnabled: boolean;// Phase A 기본 false
  discoveryModeEnabled: boolean;     // Phase A 기본 true
  autoPatternEnabled: boolean;       // Phase A 기본 true (v1.1)
}

export const PHASE_A_FLAGS: PlannerFeatureFlags = {
  touristBeautyOnlyScope: true,
  mateRoleEnabled: false,
  festivalIntentEnabled: true,
  foodShoppingIntentEnabled: false,
  discoveryModeEnabled: true,
  autoPatternEnabled: true,
};
```

**활용**:
- Action dispatch 전 flag check → `ROLE_TOGGLE({role: 'mate'})` → `mateRoleEnabled=false`면 무시
- UI 렌더에서 비활성 intent 카드 숨김 (Phase B)

### 질문 (Genspark 검토)

- Q12. Feature flag는 빌드 타임 상수인가, 런타임 (Supabase config)인가? Phase A는 상수로 충분한가?

---

## 7. 테스트 케이스 매트릭스 (Vitest)

### 7-1. `compute-pattern.test.ts`

| TC ID | 입력 | 기대 출력 | 커버 EC |
|-------|------|---------|--------|
| CP-01 | 단일 시술 downtime 0 | `A` (procedure_first), exceptions=[] | 기본 |
| CP-02 | 단일 시술 downtime 1 | `C` (anytime) | 임계값 C |
| CP-03 | 단일 시술 downtime 3 | `A` | 임계값 A |
| CP-04 | 단일 시술 downtime 4 | `B` | 임계값 B |
| CP-05 | 보톡스 0 + 울쎄라 14 + 스킨부스터 3 | `B` + 울쎄라 exception | EC-17 |
| CP-06 | 다운타임 모두 0 | `C`, 모두 same-day bundle | Bundle |
| CP-07 | 다운타임 ≤ 3 여러 개 | 모두 same-day bundle 가능 후보 | Bundle |
| CP-08 | `hasMixedDowntime` 검증 | 단일 0, 다수 14 → true | Mixed flag |
| CP-09 | `flightRestrictionHours > flightHours` | flightWarnings 포함 | 비행 제한 |
| CP-10 | 빈 treatments | 예외 throw or 기본값 반환 (결정 필요) | Edge |

### 7-2. `resolve-entry-source.test.ts`

| TC ID | 입력 | 기대 출력 | EC |
|-------|------|---------|----|
| RS-01 | `mate_id=m1, wish_ids=w1, venue_id=v1` | driving='mate_profile', conflictLog 기록 | EC-01 |
| RS-02 | `entrySource=abc` (invalid) | resolved='home', invalidValue='abc' | EC-02 |
| RS-03 | 파라미터 없음 | resolved='home' | EC-02 |
| RS-04 | `wish_ids=w1` 만 | resolved='wishlist' | 기본 |
| RS-05 | `share_id=s1` 만 | resolved='shared' | 기본 |

### 7-3. `canonical-dedupe.test.ts`

| TC ID | 입력 | 기대 출력 | EC |
|-------|------|---------|----|
| CD-01 | 동일 procedureId + variant | 1개 유지, conflicts=[] | EC-15 |
| CD-02 | 동일 procedureId 다른 variant (`auto`) | 자동 병합, variant 배열화 | EC-15 |
| CD-03 | 동일 procedureId 다른 variant (`confirm`) | conflicts 반환 | EC-15 |
| CD-04 | 동일 procedureId 다른 variant (`keep_both`) | 2개 유지 | EC-15 |

### 7-4. `reducer.test.ts`

| TC ID | 액션 | 초기 상태 | 기대 후 상태 | EC |
|-------|-----|---------|-----------|----|
| RD-01 | `BEAUTY_INTENT_OFF` | treatments=[t1,t2] | treatments=[], hiddenDraft=[t1,t2] | EC-07 |
| RD-02 | `BEAUTY_INTENT_ON` (hiddenDraft 있음) | hiddenDraft=[t1] | treatments=[t1], timingPattern='auto', resultStale=true | EC-07 |
| RD-03 | `BEAUTY_INTENT_ON` (hiddenDraft 없음) | 초기 | mode='discovery' | EC-16 |
| RD-04 | `BEAUTY_TREATMENT_ADD` 동일 procedure | 기존 1개 | 중복 없음 (dedupe) | EC-15 |
| RD-05 | `TOURIST_INPUT_PATCH` 결과 있을 때 | resultStale=false | resultStale=true | v1.1 |
| RD-06 | `DRAFT_HYDRATE` | 빈 state | 복원된 draft | EC-21 |
| RD-07 | `ROLE_TOGGLE` (Phase A: mate disabled) | role='tourist' | role='tourist' 유지 (no-op) | flag |
| RD-08 | `INTENT_TOGGLE('beauty')` 해제 | intents=['beauty'] | intents=[], beauty.active=false + hiddenDraft 저장 | EC-06, EC-07 |

### 7-5. `session-draft.test.ts`

| TC ID | 시나리오 | 기대 결과 |
|-------|--------|---------|
| SD-01 | save → load roundtrip | 동일 draft 복원 |
| SD-02 | role별 분리 | tourist draft와 mate draft 독립 저장 |
| SD-03 | listDrafts | 모든 draft 메타 반환 |
| SD-04 | sessionStorage full | 에러 처리 or in-memory fallback |

### 7-6. 테스트 커버리지 목표

- Pure functions: **100% 라인 커버리지**
- Reducer: **모든 action type 1개 이상 케이스**
- Storage: roundtrip + 에러 경로

---

## 8. Vitest 환경 구성 (선행 작업)

Phase A 착수 전 해야 할 설정:

```bash
cd app
npm install -D vitest @vitest/ui @testing-library/react @testing-library/jest-dom jsdom
```

`app/vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test-setup.ts'],
  },
});
```

`package.json` scripts 추가:
```json
"test": "vitest",
"test:run": "vitest run",
"test:ui": "vitest --ui"
```

---

## 9. Phase A 완료 기준 (Acceptance Criteria)

Phase A는 아래 모두 green일 때 완료:

- [ ] `features/ai-planner/types.ts` 모든 타입 export
- [ ] 3 pure function 모두 구현 + 테스트 커버리지 100%
- [ ] Reducer + 22개 action 구현 + 최소 1개 테스트/action
- [ ] sessionStorage 저장·복원 테스트 green
- [ ] Feature flag 구조 정의
- [ ] `vitest run` 에러 0, fail 0
- [ ] TypeScript `tsc --noEmit` 에러 0
- [ ] ESLint 에러 0

**비포함**: UI 렌더링, 서버 호출, 실제 `/ai/plan` 페이지 동작.

---

## 10. 리스크·주의

### 10-1. Next.js 16 App Router + RSC 경계

- sessionStorage는 **Client Component에서만** 접근 가능
- Reducer를 Client Component 바운더리 안에 둘 것
- `'use client'` 지시어 필요한 파일: `state/reducer.ts` (실제 hook 사용하는 측), `storage/session-draft.ts` (window 접근)

### 10-2. React 19 useActionState / useOptimistic

- Phase B에서 서버 액션 연동 시 고려
- Phase A는 순수 reducer로 진행 (기존 방식)

### 10-3. Draft 크기 한계

- sessionStorage는 보통 5-10MB
- treatments 10개 × 평균 500 bytes = 5KB, 충분

### 10-4. 테스트 격리

- Vitest는 test 파일별 기본 격리, sessionStorage mock은 `beforeEach`로 리셋

---

## 11. Genspark 검토 요청 포인트 요약

12개 결정 포인트:

| # | 영역 | 질문 |
|---|-----|------|
| Q1 | 타입 | `BeautyTreatment.id` vs `procedureId` 분리가 맞나 |
| Q2 | 타입 | 인원 4개 필드 분리 vs 구조화 객체 |
| Q3 | 타입 | `sessionId`를 Supabase 저장 키로도 쓸 수 있나 |
| Q4 | Pure | `computePattern` config를 인자로 받기 vs 모듈 상수 |
| Q5 | Pure | `flightHours` 기본값 2.5h 충분한가 |
| Q6 | Pure | `canonicalDedupe` mode enum 완전한가 |
| Q7 | Reducer | Action 22개 너무 많은가 |
| Q8 | Reducer | `DRAFT_VERSION_BUMP` 별도 action vs 자동 |
| Q9 | Structure | Selector 위치 — `state/` vs `lib/` |
| Q10 | Storage | iOS Safari private mode fallback |
| Q11 | Storage | 자동 삭제 조건 처리 위치 |
| Q12 | Flag | Feature flag 빌드타임 vs 런타임 |

---

## 12. 관련 문서

- `AI_PLANNER_BEAUTY_STEP25.md` v1.2 — 상위 설계 SSOT
- `AI_PLANNER_EDGE_CASES.md` v1.1 — EC-01~25 처리 규칙
- `AI_PLANNER_PAGE_DESIGN.md` v1.0 — 3-Layer 구조·Plan Depth
- `ADMIN_CONFIGURABLE_PARAMS.md` v1.3 — Runtime config

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|------|------|------|
| 2026-04-23 | v1.0 최초 작성 | Phase A 착수 전 Genspark 검토용 |

---

*버전: v1.0 / 마지막 업데이트: 2026-04-23*
