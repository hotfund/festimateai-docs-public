# AI Planner — Step 2.5 Beauty Micro-step UX 상세

> **작성**: 2026-04-22 / **갱신**: 2026-04-23 (v1.2)
> **위상**: AI Planner v1.0 구현 가이드 — §3-4 "Beauty Intent 추가 Step" 심화
> **조언 크레딧**: Claude (초안 v1.0) + Genspark (Advisor v1.1 Wireflow·Validation·State machine·auto 모드, v1.2 soft-block 정제·hybrid auto 프롬프트·stale top banner)
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

### 1-3. 사용자가 고르지 않는 것 (UX 마찰 최소화)

- **시술 타이밍 (Day N)**: Step 2.5에서 안 정함 → 최종 AI 플랜 결과에 반영
- **클리닉**: Step 2.5에서 안 고름 → AI Matching 이후 단계

### 1-4. 타이밍 패턴은 "선택"이지만 강제 아님 (v1.1 수정)

v1.0에서는 AI 자동 산정 후 "사용자 승인"을 강제했으나, UX 마찰이 큼 → v1.1에서 완화:

- 시스템이 downtime 기반으로 **추천 패턴** 제시 (`AI 추천` 배지)
- 사용자 택1:
  - `procedure_first` (A) — 시술 먼저
  - `travel_first` (B) — 여행 먼저
  - `anytime` (C) — 언제든
  - **`auto` — AI에게 맡기기** (승인 없이 Step 3 진행 허용, AI가 플랜 생성 시점에 최종 결정)
- 내부 코드는 A/B/C 유지, 외부 payload에는 semantic enum 사용

---

## 2. Wireflow — 단계별 Step ID (v1.1 신규)

프론트 라우팅·QA 핸드오프용 공식 Step ID.

| Step ID | 화면 | 목적 | Primary CTA | Secondary CTA | 완료 조건 | 다음 |
|---------|------|------|-------------|---------------|----------|------|
| **WF-00** | Role 토글 | Tourist/Mate 선택 | `여행 플랜 만들기` | `역할 바꾸기` | Tourist 선택 | WF-01 |
| **WF-01** | Intent 선택 | 의도 카드 복수 선택 | `다음` | — | intent ≥ 1 | beauty 포함 시 WF-02, 아니면 WF-04 |
| **WF-02** | Step 2.5-A 시술 선택 | beauty 컨텍스트 확정 | `시술 타이밍 정하기` | `건너뛰고 추천받기` | 시술 ≥ 1 or discovery mode | WF-03 |
| **WF-03** | Step 2.5-B 타이밍 패턴 | 시술↔여행 배치 원칙 | `이 방식으로 진행` | `🤖 AI에게 맡기기` | 패턴 선택 or `auto` | WF-04 |
| **WF-04** | Step 3 기본 정보 | 5필드 입력 | `AI 플랜 생성` | `이전` | 필수값 통과 | WF-05 |
| **WF-05** | Step 4 생성 중 | AI 호출 진행 | (disabled) | `취소` (2초 윈도우) | 응답 수신 | WF-06 |
| **WF-06** | Step 5 결과 | 플랜 검토·액션 | `💎 AI Matching` | `💾 저장` `🔗 공유` `🔄 재생성` | 렌더 성공 | 후속 플로우 |

### 핵심 규칙
- **Beauty는 conditional step**: beauty intent 없으면 WF-02/03 건너뜀
- **WF-02 skip = discovery mode**: 완전 차단 X
- **WF-03 skip = `auto` 모드**: AI가 생성 시점에 패턴 결정

---

## 3. 상태 모델

```typescript
interface BeautyMicroState {
  // ── 활성화 ───────────────────────────
  active: boolean;              // beauty intent 체크 여부
  mode: 'cart' | 'manual' | 'discovery';
  
  // ── 담긴 시술 ────────────────────────
  treatments: BeautyTreatment[];
  
  // ── 패턴 (v1.1: auto 모드 추가) ──────
  timingPattern: 'procedure_first' | 'travel_first' | 'anytime' | 'auto';
  // 내부 매핑: procedure_first=A, travel_first=B, anytime=C
  recommendedPattern: 'A' | 'B' | 'C' | null;    // AI 추천 (참조용)
  exceptions: TreatmentException[];
  hasMixedDowntime: boolean;                     // derived flag
  
  // ── Discovery 모드 전용 ──────────────
  discoveryHints?: {
    travelDays: number;
    returnDate: string;
    candidateCategories: string[];
  };
  
  // ── 유실 방지 (EC-07) ────────────────
  hiddenDraft?: BeautyTreatment[];   // intent off 시 treatments를 여기로 이동
  
  // ── 편집 충돌·stale (v1.1 신규) ──────
  draftVersion: number;              // 편집할 때마다 증가
  resultStale: boolean;              // 결과 생성 후 입력 변경되면 true
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

## 4. 모드별 UX

### 4-1. Cart 모드 — `/beauty`에서 담아옴 (가장 일반적)

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
│  [✓ 이 패턴으로 진행]  [🤖 AI에게 맡기기]         │
│                             [패턴 다시 계산]       │
└────────────────────────────────────────────────┘
```

**인터랙션**:
- 상단 시술 목록: 삭제·우선도 변경 시 하단 AI 제안 재계산 (debounce 500ms)
- "+ 시술 더 추가" → 모달 (Manual 모드 검색 UI 재사용)
- **v1.1**: 패턴 승인은 선택 사항. `auto` 모드로 진행 시 AI가 플랜 생성 시점에 최종 결정
- `auto` 선택 시 `timingPattern: 'auto'`, `recommendedPattern`은 참조용으로 보존

---

### 4-2. Manual 모드 — 시술 검색·추가

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

### 4-3. Discovery 모드 — beauty 체크만, 시술 0개 (EC-16)

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

## 5. 다운타임 패턴 자동 산정 (EC-17 구현)

### 5-1. 입력

```typescript
interface PatternInput {
  treatments: BeautyTreatment[];
  travelDays: number;            // 여행 일수
  returnDate: string;            // 귀국일
  flightHours: number;           // 비행 소요 (일본↔한국 ~2.5h)
}
```

### 5-2. 알고리즘 (클라이언트 측 pure function)

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

### 5-3. 출력 예시 (보톡스 + 울쎄라 + 스킨부스터)

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

### 5-4. UI 표시 규칙

- primary 패턴은 **한 줄 요약**으로만 노출 ("B 패턴: 여행 먼저") + `AI 추천` 배지
- exceptions는 **시술별 배지**로 표시 (🔔 "귀국 직전 권장")
- sameDayBundles는 **그룹핑 박스**로 시각화 ("⊕ 같은 날 가능")
- 사용자 액션 (v1.1): `[✓ 이 패턴으로 진행]` / `[🤖 AI에게 맡기기]` / `[패턴 다시 계산]`

---

## 6. Beauty off → 데이터 보존 (EC-07)

### 6-1. 해제 시 처리

```typescript
function onBeautyIntentOff(state: BeautyMicroState): BeautyMicroState {
  return {
    ...state,
    active: false,
    hiddenDraft: state.treatments,   // 통째로 이동
    treatments: [],
    timingPattern: 'auto',
    recommendedPattern: null,
    exceptions: [],
    resultStale: true,
  };
}
```

### 6-2. 재체크 시 복원

```typescript
function onBeautyIntentOn(state: BeautyMicroState): BeautyMicroState {
  if (state.hiddenDraft && state.hiddenDraft.length > 0) {
    return {
      ...state,
      active: true,
      treatments: state.hiddenDraft,
      hiddenDraft: undefined,
      // 패턴은 재계산 필요 → auto 기본값
      timingPattern: 'auto',
      recommendedPattern: null,
      resultStale: true,
    };
  }
  return { ...state, active: true, mode: 'discovery' };
}
```

### 6-3. 완전 삭제 조건

- 세션 종료 (sessionStorage 만료)
- 사용자가 명시적으로 "시술 전체 초기화" 버튼 클릭
- draft를 다른 사용자 계정으로 전환 (로그인 변경)

---

## 7. AI 호출 Payload 계약 (v1.1 — semantic enum + stale/version)

Step 2.5 결과물은 canonical payload에 다음과 같이 포함:

```json
{
  "intents": ["beauty", "festival"],
  "beauty": {
    "mode": "cart",
    "discoveryMode": false,
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
    "timingPattern": "travel_first",
    "recommendedPattern": "B",
    "primaryDowntimeDays": 14,
    "hasMixedDowntime": true,
    "exceptions": [
      { "treatmentId": "uuid-울쎄라",
        "reason": "longer_downtime",
        "recommendation": "귀국 1일 전" }
    ],
    "sameDayBundles": [
      ["uuid-보톡스", "uuid-스킨부스터"]
    ],
    "notes": "여행 첫날 붓기는 피하고 싶음"
  },
  "draftVersion": 7,
  "resultStale": false
}
```

**필드 설계 원칙**:
- `timingPattern` enum: `procedure_first | travel_first | anytime | auto`
  - `auto` → 서버가 AI 생성 시점에 최종 결정 (v1.1 추가)
  - 내부 A/B/C 코드는 `recommendedPattern`에만 노출 (참조용)
- `mixed` 같은 derived flag는 **포함 안 함** (EC-06); `hasMixedDowntime`은 **계산값 snapshot**으로 포함 (AI 프롬프트 분기용)
- `downtimeSnapshot`은 요청 시점 값 고정 (나중에 DB 업데이트돼도 이 요청은 snapshot 유지)
- `procedureId`는 canonical UUID, variant는 메타
- `draftVersion`: 다중 탭 편집 충돌 감지 (EC-22)
- `resultStale`: 결과 생성 후 입력 변경 시 true, 서버는 요청 거부 or 재생성 유도

### 7-1. `timingPattern='auto'` 서버 프롬프트 전략 (v1.2 신규)

완전 자유(model-free)로 두면 출력 분산이 커져 QA 비용 폭증. 반대로 `recommendedPattern`을 사실상 강제하면 auto의 의미가 없어짐.

**채택 전략: default-follow + explicit override (hybrid)**

#### 프롬프트 구조
```
[입력 변수]
- timingPattern: "auto"
- recommendedPattern: "travel_first"        (클라 pure function 결과)
- reason.maxDowntimeDays: 14
- reason.hasMixedDowntime: true
- trip.lengthDays: 4
- trip.returnDayFixed: true
- userNote: "귀국 전 시술은 피하고 싶어요"   (선택)

[모델 지시]
1. recommendedPattern을 우선 채택하라.
2. 다음 제약이 recommendedPattern과 충돌하면 override 허용:
   - 여행일 < max(downtime) + 1
   - mixed downtime + 짧은 일정
   - 사용자 메모가 명시적 선호 표현
   - recovery metadata 불완전 (unknownDowntime=true)
3. override 시 반드시 result.note에 1줄 이유 남길 것.
   예: "귀국 전 시술 회피 메모로 procedure_first로 변경"
```

#### 상황별 의사결정 가이드

| 상황 | 권장 분기 |
|------|----------|
| 단일 시술 + downtime 명확 | `recommendedPattern` 거의 고정 |
| mixed downtime | `recommendedPattern` 우선, 제약 충돌 시 override |
| 사용자 메모가 강한 선호 표현 | **사용자 메모 최우선** |
| 여행일 < max(downtime) + 1 | 보수적 패턴(anytime 또는 travel_first) |
| recovery metadata 불완전 | 보수적 패턴 + note에 불확실성 명시 |

#### 서버 응답 구조 (auto 모드 추가 필드)
```json
{
  "plan": { ... },
  "beauty": {
    "finalPattern": "procedure_first",
    "patternDecisionMode": "auto_override",    // "auto_follow" | "auto_override"
    "overrideReason": "귀국 전 시술 회피 메모 반영"
  }
}
```

**핵심 원칙**: auto는 "모델이 아무렇게나 정함"이 아니라 **설명 가능한 자동결정**. 사용자가 결과 화면에서 왜 이 패턴이 선택됐는지 이해 가능해야 함.

---

## 8. Validation 규칙 (v1.1 신규)

프론트/서버 양쪽에서 공유되는 공식 rule ID. 구현 시 각 rule마다 테스트 케이스 1개 이상.

### 8-1. Step 2.5-A 시술 선택

| Rule ID | 조건 | 메시지 | 차단 | 처리 |
|---------|------|--------|------|------|
| **V-BEA-01** | `beauty=true` & treatments 비어 있음 | "시술 없이도 진행할 수 있어요. AI 추천 모드로 넘어갑니다." | 비차단 | discovery mode 전환 |
| **V-BEA-02** | 동일 canonical treatment 중복 | "이미 추가된 시술입니다." | 부분차단 | dedupe 확인 모달 |
| **V-BEA-03** | downtime 메타 없음 (procedures 레코드 누락) | "회복 정보가 불완전해 AI가 보수적으로 계산합니다." | 비차단 | `unknownDowntime: true` flag |
| **V-BEA-04** | 시술 > 5개 | "한 번에 너무 많은 시술은 일정 품질을 낮출 수 있어요. 5개 이하로 정리해주세요." | **Step-local soft-block** | **자동 절단 금지** — CTA(`시술 타이밍 정하기`) 비활성화 유지. 사용자가 직접 5개 이하로 줄일 때만 진행 허용. payload에 사용자 몰래 top-N 전송 X (v1.2 정제) |
| **V-BEA-05** | `/beauty` prefill 파싱 실패 | "가져온 시술 정보를 다시 확인해주세요." | 비차단 | prefill 제거 후 수동 선택 유도 |

### 8-2. Step 2.5-B 타이밍 패턴

| Rule ID | 조건 | 메시지 | 차단 | 처리 |
|---------|------|--------|------|------|
| **V-TIME-01** | `timingPattern` 값 없음 | "AI에게 맡기기로 자동 설정됩니다." | 비차단 | `auto` 저장 |
| **V-TIME-02** | 긴 downtime(≥7일)인데 A 선택 | "이 시술은 여행 초반보다 귀국 직전이 더 적합할 수 있어요." | 비차단 | 경고 노출 후 허용 |
| **V-TIME-03** | 짧은 downtime(≤1일)인데 B 선택 | "더 이른 일정에도 배치할 수 있어요." | 비차단 | 경고 노출 후 허용 |
| **V-TIME-04** | `hasMixedDowntime=true` | "가장 긴 회복 시간을 기준으로 우선 배치합니다." | 비차단 | `exceptions[]` 생성 |
| **V-TIME-05** | 여행일 < max(downtime) + 1 | "회복 시간을 고려하면 추천 가능한 시술이 제한될 수 있어요." | 비차단 | conservative prompt 추가 |

### 8-3. Step 3 기본 입력

| Rule ID | 조건 | 메시지 | 차단 | 처리 |
|---------|------|--------|------|------|
| **V-INP-01** | 날짜 start/end 없음 | "여행 날짜를 입력해주세요." | 차단 | submit 불가 |
| **V-INP-02** | end < start | "종료일이 시작일보다 빠를 수 없어요." | 차단 | inline error |
| **V-INP-03** | 인원 < 1 | "인원 수를 다시 확인해주세요." | 차단 | field reset |
| **V-INP-04** | 예산 없음 | "예산이 없어도 생성 가능하지만 추천 정확도가 낮아질 수 있어요." | 비차단 | `budget: 'unknown'` 허용 |
| **V-INP-05** | beauty on + notes 없음 | "원하시는 분위기나 회복 부담 정도를 적어주시면 더 정확해져요." | 비차단 | helper text만 |

---

## 9. Back / Skip / Edit 내비게이션 규칙 (v1.1 신규)

| 동작 | 발생 위치 | 시스템 동작 | 데이터 보존 | UX |
|------|----------|-----------|-----------|-----|
| **Back** | WF-02 → WF-01 | Intent 화면 복귀 | 시술은 hiddenDraft | beauty 해제 전까지 복원 |
| **Back** | WF-03 → WF-02 | 시술 선택 복귀 | 패턴 선택 유지 | 시술 수정 시 "패턴 재검토" 배너 |
| **Back** | WF-04 → WF-03 | 타이밍 패턴 복귀 | base input 임시저장 | "변경 시 플랜이 달라질 수 있어요" |
| **Skip** | WF-02 "건너뛰고 추천받기" | discovery mode 진입 | `treatments=[]` 유지 | AI가 후보 제안 |
| **Skip** | WF-03 "AI에게 맡기기" | `timingPattern='auto'` | 추천 패턴 참조로 보존 | 결과에서 AI가 선택 패턴 설명 |
| **Edit from Result** | WF-06 → beauty summary 클릭 | 해당 step 모달/복귀 | 기존 결과 stale marking | **"수정됨, 다시 생성 필요"** read-only 표시 |
| **Intent off** | WF-01에서 beauty 해제 | Step 2.5 전체 비활성 | beautyDraft hidden 저장 | 재선택 시 복원, AI payload 제외 |
| **Role toggle** | Tourist → Mate | Tourist draft 분리 저장 | beautyDraft는 Tourist 내부에만 | 역할 간 **자동 상속 금지** |

### 핵심 원칙

1. **삭제보다 숨김이 우선** — beauty off, depth downshift, back 모두 hidden draft 패턴 따름
2. **결과 수정 = stale marking, 아님 patch** — WF-06에서 입력 변경 시 기존 결과 덮어쓰기 금지, `resultStale=true` 마킹 → 재생성 강제
3. **Role 간 draft 공유 금지** — Tourist의 beauty 데이터가 Mate 모드로 넘어가면 의미 오염 (EC-12)

---

## 10. 상태 머신 키 (v1.1 신규)

7개 상태 키. 디버그·analytics·QA 시나리오 작성 시 참조.

| 상태 키 | 설명 | 진입 조건 | 이탈 조건 |
|--------|------|----------|----------|
| `intentSelected` | intent 1개 이상 선택됨 | intent ≥ 1 | intent 수정 |
| `beautyDraft.empty` | beauty on + 시술 0개 | beauty=true & treatments=[] | 시술 추가 or discovery 진입 |
| `beautyDraft.prefilled` | `/beauty`에서 자동 유입 | entrySource=wishlist & clinic wish_ids | 사용자 삭제 |
| `timingPattern.manual` | A/B/C 사용자 직접 선택 | 패턴 카드 탭 | 편집 or `auto` 전환 |
| `timingPattern.auto` | "AI에게 맡기기" 또는 skip | skip or 명시 선택 | 수동 패턴 선택 |
| `plannerReady` | AI 생성 가능 상태 | 모든 필수 validation pass | validation fail |
| `result.stale` | 결과 생성 후 입력 변경 | result 이후 편집 | regenerate 완료 |

---

## 11. CTA 문구 매트릭스 (v1.1 신규)

### 11-1. Step별 CTA

| 위치 | 상황 | 문구 | 타입 | 비고 |
|------|------|------|------|------|
| WF-01 Intent | beauty 포함 | `다음` | Primary | → WF-02 |
| WF-02 | 시술 ≥ 1 | `시술 타이밍 정하기` | Primary | → WF-03 |
| WF-02 | 시술 0개 | `건너뛰고 추천받기` | Secondary | discovery mode |
| WF-03 | 패턴 탭 선택 | `이 방식으로 진행` | Primary | manual 저장 |
| WF-03 | 선택 어려움 | `🤖 AI에게 맡기기` | Secondary | auto 저장 |
| WF-04 Input | validation pass | `AI 플랜 생성` | Primary | → WF-05 |
| WF-06 Result | 생성 완료 | `💎 AI Matching` | **Primary strong** | 가장 강조 |
| WF-06 | 로그인 | `💾 저장` | Secondary | 비로그인 시 로그인 유도 |
| WF-06 | 공유 | `🔗 공유` | Secondary | share URL 생성 |
| WF-06 | 재생성 | `🔄 재생성` | Secondary warning | **"500pt 사용됩니다" 확인 모달** |

### 11-2. 마이크로카피 (서브 텍스트)

- **WF-02 헤더**: "뷰티 일정도 함께 최적화할까요?"
- **WF-02 서브**: "시술을 고르면 회복 시간을 포함한 일정으로 최적화됩니다."
- **WF-02 discovery 안내**: "아직 시술을 정하지 않으셨다면, 여행 일정에 맞는 후보를 AI가 먼저 제안해드릴게요."
- **WF-03 헤더**: "시술 타이밍을 어떻게 잡을까요?"
- **WF-03 서브**: "짧은 다운타임은 여행 초반, 긴 다운타임은 귀국 직전이 유리할 수 있어요."
- **WF-06 Regenerate 확인**: "현재 조건으로 다시 생성합니다. 500pt가 사용됩니다."
- **WF-06 Stale 배지**: "입력이 변경되었습니다. 다시 생성해주세요."

---

## 12. 결과 화면 연동 규칙 (v1.1 신규 / v1.2 stale UX 정제)

Step 2.5 결과물이 WF-06 (결과 화면)에서 어떻게 시각화되는가.

### 12-1. 필드 반영

| 항목 | 반영 위치 | 규칙 |
|------|----------|------|
| **선택 시술** | Day Card 상단 요약 | 대표 시술명 + "외 N개" |
| **회복 시간** | Timeline 블록 | 회복 일자를 반투명 블록으로 표시 |
| **타이밍 패턴** | Plan note 태그 | `시술 먼저` / `여행 먼저` / `유연 배치` / `AI 자동` |
| **Auto override reason** | Plan note 1줄 | `patternDecisionMode=auto_override`일 때만 노출 |
| **Exceptions** | Warning box | 긴 downtime·귀국 직전 권장 등 |
| **sameDayBundles** | Day Card 그룹핑 | "같은 날 가능" 묶음 표시 |
| **후속 액션** | Sticky footer | 💎 Matching / 💾 / 🔗 / 🔄 |
| **재생성 기준** | input hash | beauty params 포함 hash 기준, 불변 시 재호출 X |

### 12-2. Stale UX (v1.2 — top banner 중심)

**결정**: 일본 20-30대 여성 UX 기준 full overlay는 "막힘·오류" 감각 → 이탈 위험. 대신 **top banner + sticky regenerate CTA 강조** 패턴 채택.

```
┌────────────────────────────────────────────────┐
│  ⓘ 입력이 변경되었어요.                           │ ← top banner (pulse 애니)
│    최신 추천을 보려면 다시 생성해 주세요.         │
│                          [🔄 다시 생성]          │
├────────────────────────────────────────────────┤
│  Day 1                        (10~15% dim)     │
│  ...기존 결과 계속 읽기 가능...                  │
│                                                │
│  Day 2                                          │
│  ...                                            │
├────────────────────────────────────────────────┤
│  [💎 Matching] [💾 저장] [🔗 공유] [🔄 재생성] │
│      ↑disabled   ↑draft만  ↑disabled  ↑pulse   │
└────────────────────────────────────────────────┘
```

#### stale 상태 액션별 정책

| 액션 | stale 상태 동작 | 이유 |
|------|---------------|------|
| **결과 읽기** | 그대로 가능 (10-15% dim) | 비교 감상 흐름 유지 |
| **🔄 재생성** | 강조 (pulse 애니·색 대비) | 핵심 회복 경로 |
| **💾 저장 (draft)** | 허용 | 작업 손실 방지 |
| **💎 AI Matching** | **비활성** 또는 "최신 결과 생성 후 가능" 모달 | stale 데이터로 downstream 오염 방지 |
| **🔗 공유** | **비활성** | 공유 URL은 최신 결과 기반이어야 함 |

#### 금지 사항

- ❌ Full-screen blocking overlay (사용자 혼남 느낌)
- ❌ 결과 카드 완전 가림
- ❌ "결과 사라짐" 착시 (이탈 유발)

#### 허용 되는 약한 시각 효과

- 10-15% 전체 dim
- Banner pulse 애니메이션
- Regenerate CTA 색 대비·크기 강조
- stale 시술별 "변경됨" 작은 배지 (선택)

---

## 13. Edge Case 매핑 (구현 체크리스트)

| EC | 적용 위치 | 이 문서 참조 |
|----|----------|--------------|
| EC-07 Beauty on/off soft delete | §6 hiddenDraft 패턴 | 복원 로직 구현 |
| EC-15 Canonical dedupe | §4-2 Manual 모드 담기 | procedureId 비교 모달, V-BEA-02 |
| EC-16 Discovery mode | §4-3 전체 | travelDays 기반 추천 호출, V-BEA-01 |
| EC-17 Primary + exceptions | §5 알고리즘 | `computePattern` pure function, V-TIME-04 |
| EC-18 Auto-check on wishlist prefill | 진입 로직 | wish_ids→beauty intent 자동 체크 토스트 |
| EC-22 다중 탭 draftVersion | §7 payload | `draftVersion` 필드 |

---

## 14. 구현 Phase 매핑 (EDGE_CASES §구현 순서)

| Phase | 이 문서 작업 |
|-------|-------------|
| **A (상태 모델)** | §3 타입 정의 + reducer, §5 `computePattern` pure function, §6 hidden draft 전환, §10 상태 머신 키 7개 확정 |
| **B (보호 UX)** | §4-3 Discovery 실패 fallback, §6 재체크 복원 UI, §9 Back/Skip/Edit 내비게이션, §12 stale overlay, §11 CTA 문구·모달 |
| **C (서버 계약)** | §7 canonical payload (zod), `timingPattern='auto'` 서버 처리, `resultStale=true` 400 반환, §8 Validation rule ID 공유 |
| **D (QA 회귀)** | §15 테스트 시나리오 자동화·수동 체크, 15개 rule ID별 테스트 케이스 |

---

## 15. QA 테스트 시나리오 (Step 2.5 전용)

### Unit (Vitest)
- `computePattern([보톡스 0일, 울쎄라 14일, 스킨부스터 3일])` → primary='B', exceptions에 울쎄라
- `computePattern([보톡스 0일])` → primary='C'
- canonical dedupe: `{procedureId:'X', variant:'사각턱'}` + `{procedureId:'X', variant:'사각턱'}` → 1개 유지
- canonical dedupe: `{procedureId:'X', variant:'사각턱'}` + `{procedureId:'X', variant:'광대'}` → 2개 유지

### Integration (React Testing Library)
- Cart 모드 진입 → 시술 3개 렌더링 → 1개 삭제 → 패턴 재계산 호출 확인
- beauty 체크 해제 → treatments 비워짐 + hiddenDraft에 보존 확인
- beauty 재체크 → treatments 복원 + `timingPattern='auto'` + `resultStale=true` 확인
- Discovery 모드 "AI 후보 추천" 실패 → "직접 선택하러 가기" CTA만 남는지 확인
- WF-03에서 `AI에게 맡기기` 클릭 → `timingPattern='auto'` 저장 + WF-04 진입 가능 확인
- WF-06 결과 상태에서 beauty summary edit → `resultStale=true` + overlay 노출 확인

### E2E (Playwright)
- `/beauty`에서 시술 2개 담기 → `/ai/plan` 진입 → Step 2.5 자동 표시 + 자동 beauty intent 체크 확인
- WF-02 skip → discovery mode → WF-03 스킵 → WF-04 진입 가능 (auto mode 전체 플로우)
- WF-06 재생성 버튼 → 500pt 확인 모달 노출

### Manual (iPhone Safari 우선)
- [ ] Cart 모드에서 시술 우선도 변경 시 AI 패턴 재계산 (debounce 500ms)
- [ ] same-day bundle 그룹핑 박스가 시각적으로 구분되는지
- [ ] Discovery 모드 AI 추천 실패 시 사용자 당황 없는지 (직접 선택 CTA 명확성)
- [ ] 비행 제한 시술(예: 눈 시술 24h) 경고 배너 노출 확인
- [ ] hiddenDraft 복원 시 `timingPattern='auto'`로 리셋되는지
- [ ] 결과 화면에서 beauty 편집 후 stale overlay 시각 확인

---

## 16. Config 연결 (ADMIN_CONFIGURABLE_PARAMS)

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

# 시술 개수 상한 (V-BEA-04, v1.2 정제)
beauty.treatments.max_per_plan          = 5
beauty.treatments.over_limit_behavior   = "block_until_user_reduces"
                                        # "block_until_user_reduces" (v1.2 기본, 자동 절단 금지)
                                        # "auto_truncate" (레거시, 사용 권장 안 함)
beauty.treatments.soft_limit_message    = "한 번에 너무 많은 시술은 일정 품질을 낮출 수 있어요. 5개 이하로 정리해주세요."

# 타이밍 패턴 기본값
beauty.timing.default_when_skipped      = "auto"      # "auto" | "prompt_required"
beauty.timing.long_downtime_days        = 7           # V-TIME-02 기준
beauty.timing.short_downtime_days       = 1           # V-TIME-03 기준

# Auto 모드 서버 프롬프트 (v1.2 신규)
beauty.auto.strategy                    = "default_follow_with_override"
                                        # "default_follow_with_override" (v1.2 기본)
                                        # "strict_follow" (override 금지)
                                        # "free" (레거시)
beauty.auto.override_reason_required    = true
beauty.auto.conservative_on_unknown     = true        # metadata 불완전 시 보수적 패턴

# Stale result (v1.2 — top banner 전환)
plan.result.stale_on_edit               = true
plan.result.stale_ux_mode               = "top_banner"
                                        # "top_banner" (v1.2 기본)
                                        # "soft_overlay" (선택, 10-15% dim만)
                                        # "full_overlay" (사용 금지)
plan.result.stale_dim_percent           = 12          # 0~20 권장
plan.result.stale_disables              = ["matching", "share"]   # 저장은 draft만 허용
plan.result.stale_regenerate_pulse      = true
```

---

## 17. Open Questions (Phase 2 이후)

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

## 18. 관련 문서

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
| 2026-04-23 | v1.1 | Genspark 조언 병합: §2 Wireflow(WF-00~06), §8 Validation rule ID 15개, §9 Back/Skip/Edit 내비게이션, §10 상태 머신 키 7개, §11 CTA·마이크로카피 매트릭스, §12 결과 화면 연동 규칙. `auto` 모드 추가(강제 승인 완화), `draftVersion`/`resultStale` 필드, semantic enum(`procedure_first/travel_first/anytime/auto`) |
| 2026-04-23 | v1.2 | Genspark 2차 조언 정제: ① V-BEA-04 자동 top-N 절단 금지 + block_until_user_reduces 원칙, ② §7-1 auto 모드 hybrid 프롬프트 전략(default-follow + explicit override + override reason 필수), ③ §12 stale UX full overlay → top banner 전환(10-15% dim만, matching·share disabled, 저장은 draft만), ④ Config 7개 추가 (over_limit_behavior, auto.strategy, stale_ux_mode 등). **다음 단계: Phase A 착수 (Tourist+Beauty feature-flag scope)** |

---

## 📍 다음 단계 (v1.2 확정 후)

**→ Phase A 상태 모델 구현** (이 문서 §14 참조)

- Scope: **Tourist + Beauty 경로만 먼저** (feature-flag으로 Mate·Festival-only 차단)
- 산출물: 타입 정의 · reducer · `computePattern` · hiddenDraft 전환 · resultStale 마킹
- Phase A 완료 후: Phase B (UI·validation·stale banner) → Phase C (서버 계약) → Phase D (QA)
- `/ai/matching` 보강은 **Planner 상태 계약 안정 이후**로 미룸

---

*버전: v1.2 / 마지막 업데이트: 2026-04-23*
