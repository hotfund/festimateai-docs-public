# AI Planner v1.0 — Edge Case 체크리스트 (25개) + 구현 Phase

> **작성**: 2026-04-22 / **갱신**: 2026-04-22 (v1.1)
> **위상**: 구현 전 봉인 필수 — AI Planner v1.0 착수 선행 작업
> **선행 문서**: `AI_PLANNER_PAGE_DESIGN.md` v1.0, `ADMIN_CONFIGURABLE_PARAMS.md` v1.3
> **조언 크레딧**: Claude (Desktop, UI·타임아웃·실무 힌트) + Genspark (Advisor, 상태 머신·derived flag·snapshot 구조)

---

## 0. 사용법

### 심각도 등급
- 🔴 **필수봉인 (1차)**: 구현 시작 **전** 규칙 확정 필수. 규칙 없이 코드 쓰면 버그 확정.
- 🟡 **주요 (2차)**: 구현 중 정리. 미완성 시 사용자 경험 저하.
- 🟢 **선택 (3차)**: Phase 2 이후 허용. MVP에서는 단순 처리 or 스킵 가능.

### 핵심 원칙 (Genspark 통찰)
> **`/ai/plan`은 단일 화면이지만 실제로는 `entryContext + intentState + roleDraft + depthState + beautyMicroState + AIRequestState`의 복합 상태 머신.**
> Edge case 봉인의 목표는 "입력값 검증"이 아니라 **"상태 전이의 일관성"**.

---

## A. EntrySource Layer (4)

### EC-01 🔴 다중 진입 파라미터 충돌

**시나리오**: `/ai/plan?wish_ids=w1,w2&venue_id=v123&mate_id=m456`

**리스크**: 어떤 prefill이 우선인지 불명확 → 잘못된 intent 프리셋·장소 고정·Mate 지정

**권장 처리**:
- 명시적 우선순위 **고정**: `mate_profile > venue > wishlist > shared > home`
- 최우선 소스 1개만 **driving context**로 채택
- 나머지는 `secondary context`로 보존만 (UI 자동 반영 X)
- 디버그 로그 `entry_conflict_resolved` 기록

**구현**: `resolveEntrySource(params)` pure function

---

### EC-02 🔴 Malformed / enum 외 값

**시나리오**: `?entrySource=abc` 또는 파라미터 자체 없음

**리스크**: 프리셋 로직 깨짐 → hydration mismatch, 잘못된 analytics 분류

**권장 처리**:
- 허용 enum 외 값은 모두 `home`으로 fallback
- 내부 로그: `invalid_entry_source` 이벤트 저장
- UI에 절대 에러 노출 X (중립 진입)
- analytics: `resolvedEntrySource=home` + `rawEntrySource=abc` 분리 저장

---

### EC-03 🔴 리소스 삭제·비공개·만료

**시나리오**: `venue_id=v999`인데 해당 venue DB 삭제됨 / `mate_id=m3`인데 Mate 비공개 전환

**리스크**: 특정 대상 포함 기대하고 왔는데 빈 상태 → 기대 배신

**권장 처리**:
- 파라미터 읽되 조회 실패 시 soft fallback
- 화면 상단 작은 배너: "일부 참조 항목을 불러올 수 없어 일반 플래너로 시작합니다"
- 실패한 prefill은 draft에 저장 X
- AI 호출 payload에도 제거

---

### EC-04 🟡 Mate_profile 진입 시 기본 역할 설정

**시나리오**: `?mate_id=m3`로 왔는데 역할 토글 기본이 Tourist

**리스크**: "누구 관점 플랜?" 혼란

**권장 처리**:
- `mate_id` 있으면 Tourist 모드 기본 유지
- Mate 정보 **사이드바에 표시** ("이 Mate와 함께하는 일정을 짜는 중")
- 역할 강제 변경 X (사용자는 여전히 선택 가능)

---

## B. Intent Layer (5)

### EC-05 🔴 Intent 0개 선택

**시나리오**: 모든 카드 해제한 채 [다음] 클릭

**리스크**: AI가 "뭘 원하는지 모름" 상태 → 무의미한 플랜

**권장 처리**:
- `intent=[]` 허용하되 [다음] CTA **비활성화**
- 최소 1개 선택 시 Input Layer 진입
- entrySource 기반 추천 intent 카드 강조는 유지
- 에러 한 줄: "최소 1개 관심사를 선택해주세요"

---

### EC-06 🔴 Mixed 플래그 — derived flag (저장 enum 아님)

**시나리오**: `festival + beauty` 선택 → `festival`만 남김 → 다시 추가

**리스크**: `mixed`를 DB enum에 저장하면 복구·동기화 깨짐

**권장 처리**:
- `mixed`는 **derived flag**로 계산 (저장값 X)
- Rule: `selectedIntent.length >= 2 => mixed=true`
- DB canonical enum에 저장 금지
- 헤드라인·프롬프트 템플릿에서만 계산값 참조

**구현**: `const mixed = useMemo(() => intents.length >= 2, [intents])`

---

### EC-07 🔴 Beauty 해제 시 Step 2.5 데이터 처리

**시나리오**: beauty 체크 → 시술 3개 선택 → beauty 해제

**리스크**: Step 2.5 입력 데이터 유실 or 남아서 AI에 섞임

**권장 처리**:
- **soft delete** — 해제해도 데이터 보관 (hidden draft)
- 다시 체크하면 복원
- AI 호출 payload에는 beauty off 시 **절대 포함 X**
- 세션 종료 or 명시적 초기화 시만 제거

**구현**: `draft.beauty.treatments` + `draft.beauty.active: boolean` 분리

---

### EC-08 🟡 Intent 재선택 시 Base Input 충돌

**시나리오**: beauty 중심으로 입력 → festival로 변경 → 기존 자유기입·템플릿과 의미 충돌

**리스크**: 기존 맥락이 새 intent와 안 맞아 AI 플랜 왜곡

**권장 처리**:
- Base Input 5필드는 유지
- intent-specific 데이터만 **"검토 필요"** 상태 전환
- 재생성 시 confirm: "기존 입력 유지하고 새 의도로 다시 생성할까요?"
- AI prompt는 현재 intent만 authoritative

---

### EC-09 🟡 Shared 링크 role 불일치

**시나리오**: Tourist 플랜 공유 링크를 Mate 모드로 열거나, role-specific 구조 공유본

**리스크**: 필드 구조 안 맞아 role toggle 후 해석 꼬임

**권장 처리**:
- 공유 링크에 `sourceRole` 메타데이터 포함
- 현재 role과 다르면 2옵션: "참고용 가져오기" / "현재 역할에 맞게 변환"
- 자동 변환은 최소 공통 필드만 허용
- role-specific 필드는 discard or 매핑 테이블 거쳐 변환

---

## C. Tourist/Mate 역할 (3)

### EC-10 🔴 역할 토글 draft 손실

**시나리오**: Tourist 5필드 입력 중 → 실수로 Mate 토글

**리스크**: 공용 canvas 장점이 데이터 유실 UX로 변질

**권장 처리**:
- role별 draft 분리 저장: `draft.tourist` / `draft.mate`
- 역할 전환 시 "현재 역할의 임시 작성 내용을 저장하고 전환" 처리
- 최초 1회만 토스트 안내
- **자동 초기화 금지**

**구현**: sessionStorage 키 2개, 또는 IndexedDB `drafts` 테이블

---

### EC-11 🟡 로그인 자동 역할 vs 사용자 선택 우선

**시나리오**: `user.role === 'mate'`인데 사용자가 직접 Tourist 토글

**리스크**: 시스템이 자동 역할 강요하면 사용자 자율성 침해

**권장 처리**:
- **사용자 선택 우선**, 시스템은 기본값만 제공
- Mate 로그인 유저도 개인 여행 계획 가능
- 단, 기본 랜딩 시엔 `user.role` 기반 프리셋 OK

---

### EC-12 🔴 Role 공통 vs role-specific 필드 분리

**시나리오**: Tourist의 `venue/wishlist` 컨텍스트가 Mate로 넘어가 Offering 데이터에 섞임

**리스크**: 플랜 의미 붕괴, canvas 오염

**권장 처리**:
- 필드 분리 명시:
  - **공통**: 날짜, 지역, 장소 참조
  - **Tourist 전용**: 인원, 여행 예산, Mate 동반 희망
  - **Mate 전용**: 가격, 페어 포맷, Offering 자기소개
- role 전환 시 **공통 컨텍스트만 carry over**
- role-specific 필드는 절대 자동 이식 X

---

## D. Plan Depth (2)

### EC-13 🟡 Depth 업그레이드 — target vs fulfilled 분리

**시나리오**: Base Input만 입력한 상태에서 Depth를 Detailed로 올림

**리스크**: 상세 항목 비어 있어도 Detailed로 표기 → 허위 완성, AI 호출 타이밍 애매

**권장 처리**:
- `targetDepth`와 `fulfilledDepth` **분리**
- 사용자 선택은 target, 실제 채움 정도는 fulfilled
- 예: `target=detailed / fulfilled=open` → UI에서 부족 항목 체크리스트
- AI 호출은 fulfilled 상태 기반 (빈 Detailed 호출 방지)

**구현**: `plan.target_depth` + `plan.fulfilled_depth` 두 필드

---

### EC-14 🔴 Depth 다운그레이드 — hide, not delete

**시나리오**: Detailed (Day-by-Day + 항목별 태그) → Open으로 변경

**리스크**: 고비용 데이터 즉시 삭제 → 되돌리기 불가능

**권장 처리**:
- **hide, not delete**
- Detailed 데이터 hidden state로 유지
- 다시 Detailed로 올리면 복원
- 명시적 "상세 내용 완전 삭제" 액션이 있을 때만 제거

**구현**: `plan.depth_data.detailed` JSONB + `plan.current_depth` 분리

---

## E. Beauty Micro-step (4)

### EC-15 🔴 Beauty prefill 중복 (canonical dedupe)

**시나리오**: `/beauty`에서 "보톡스" 담아옴 + Step 2.5에서 "보톡스" 또 추가

**리스크**: 같은 시술 중복 계산 → 다운타임·비용·일정 제약 오염

**권장 처리**:
- treatment item에 `source`, `treatmentId`, `variant` 저장
- canonical treatment ID 기준 **merge UX**
- 중복 감지 시: "같은 시술이 이미 담겨 있습니다. 병합할까요?"
- 단순 문자열 비교 금지 (오타·변형 구분)

---

### EC-16 🟡 Beauty intent on but 시술 empty (discovery mode)

**시나리오**: beauty 체크만 하고 Step 2.5에서 시술 0개

**리스크**: beauty intent를 구조화 시술 선택과 묶으면 전환율 ↓, 비워두면 AI 결과 모호

**권장 처리**:
- 빈 상태 **허용**
- `beautyDiscoveryMode=true` 분기
- AI 프롬프트: "시술 확정 전, 여행 일정 기준 후보 시술 추천"
- 패턴 A/B/C는 확정 아닌 **추천 모드**로 제안

---

### EC-17 🔴 다운타임 혼합 — primary pattern + exception note

**시나리오**: 보톡스(0일) + 울쎄라(14일) + 스킨부스터(3일) 혼합

**리스크**: 단일 패턴 A/B/C 고정하면 현실 오류

**권장 처리**:
- 패턴은 단일값 X, **primary pattern + exception note 구조**
- 가장 긴 다운타임 기준 primary pattern 결정 (예: 울쎄라 → 패턴 B)
- 짧은 다운타임은 same-day bundle 후보로 묶기
- UI 설명: "일부 시술은 함께 가능, 일부는 귀국 직전 권장"

**구현**: `plan.beauty.primary_pattern` + `plan.beauty.exceptions[]`

---

### EC-18 🟡 Beauty prefill 있음 but intent 미선택

**시나리오**: `/beauty`에서 시술 담고 `/ai/plan`으로 왔는데 beauty intent 체크 안 됨

**리스크**: 담아온 시술 무시됨

**권장 처리**:
- `entrySource=wishlist` + wish_ids에 `clinic` 타입 있으면
- **beauty intent 자동 체크 + 사용자 안내**
- 토스트: "시술을 담아오셔서 자동 선택했어요. 원치 않으시면 해제하세요"

---

## F. Fallback & 초기 화면 (2)

### EC-19 🟡 샘플·갤러리 데이터 비어있음 (초기 화면)

**시나리오**: 신규 배포 초기 콘텐츠 부족, 특정 locale에서 데이터 없음

**리스크**: "빈 화면 공포 해소"가 v1.0 핵심 원칙인데 반대 효과

**권장 처리**:
- 샘플 3분리 모두 데이터 의존 X, **최소 1개 정적 fallback** 보유
- 우선순위: `personalized sample > locale sample > global default`
- 갤러리가 비면 숨기고, 대표 샘플 + intent 카드 유지
- 정적 fallback은 JS bundle 내 하드코딩 (CDN 실패 대비)

**구현**: `const FALLBACK_SAMPLE = { ... }` (3줄 플랜)

---

### EC-20 🟡 AI 호출 네트워크 타임아웃

**시나리오**: "플랜 생성" 클릭 → 20초 후 응답 없음

**리스크**: 사용자 새로고침 → 입력 유실

**권장 처리**:
- 로딩 30초 넘으면 "잠시 기다려주세요" + 취소 버튼
- 60초 넘으면 "시간이 오래 걸려요, 다시 시도" + **입력 보존**
- 모든 draft는 sessionStorage 자동 저장 (auto-save 2초 간격)
- 재시도 시 동일 `requestId` 재사용 (EC-24 연계)

---

## G. 기술·운영 (5)

### EC-21 🔴 브라우저 뒤로가기·새로고침·URL 편집

**시나리오**: 유저가 intent 바꾸고 뒤로가기 / 새로고침 / URL 직접 수정

**리스크**: 화면 state ↔ URL state 불일치 → hydration mismatch, 잘못된 prefill, 예상치 못한 재생성

**권장 처리**:
- URL에 **최소 공유 가능한 state만** 반영 (`entrySource`, selected IDs, share refs)
- intent/input/draft는 **local state + session storage** 관리
- 새로고침 시 session draft 복구
- **URL authoritative**, draft는 "복구 가능한 임시값"

**구현**: Next.js App Router + history API, `useSearchParams()` 기반

---

### EC-22 🟡 다중 탭 편집 충돌

**시나리오**: 같은 사용자가 `/ai/plan` 두 탭에서 서로 다르게 수정

**리스크**: 나중 탭이 먼저 탭의 draft 덮어쓰기

**권장 처리**:
- draft에 `draftVersion` + `lastEditedAt` 부여
- 같은 draft key 다른 탭 수정 시 "새 버전이 다른 탭에서 열려 있음" 경고
- MVP에서는 감지 + 경고만 (resolve는 Phase 2)
- BroadcastChannel API 활용

---

### EC-23 🔴 인증 만료 중 저장/공유 실패

**시나리오**: 1시간 편집 → 저장 클릭 → 세션 만료 → 로그인 튕김

**리스크**: draft 유실

**권장 처리**:
- 저장/공유 전 auth check
- 인증 만료 시 **로그인 완료 후 draft restore 보장**
- 비로그인 사용자도 sessionStorage에 draft 자동 저장
- 로그인 성공 → draft 자동 인수

**구현**: `onAuthStateChanged` + `restoreDraftFromSession()`

---

### EC-24 🔴 AI 호출 실패·재생성 연타

**시나리오**: 생성 버튼 연타 / 네트워크 오류 후 재시도

**리스크**: 중복 AI 호출, 비용 폭증, race condition

**권장 처리**:
- 생성 요청마다 `requestId` 발급
- 같은 입력 해시 + 짧은 시간 윈도우에서 **중복 요청 차단**
- 버튼 첫 클릭 후 loading + debounce
- 실패 시: "같은 요청 다시 시도" vs "입력 수정 후 재생성" **분리**
- 마지막 성공 응답만 화면 반영

**구현**: `useMutation` + `optimistic UI` + `inputHash === lastRequestHash` 검사

---

### EC-25 🔴 포인트 부족 / 비용 정책 변경 중

**시나리오**: 재생성 도중 관리자가 포인트 정책 변경 / 사용자 포인트 부족

**리스크**: 사용자가 생성되는 줄 알고 입력 끝냈는데 마지막에 막힘 → 전환 손실

**권장 처리**:
- 생성 시작 전 예상 비용·차감 여부 **미리 표시**
- 제출 시점에 최종 재검증
- 부족 시: 입력 보존 + 옵션 제공 (`paywall` / `recharge` / `later save`)
- 정책 변경은 요청 시작 시 **snapshot version 고정** → 해당 요청 동안 일관성 유지

**구현**: `requestSnapshot = { pointCost: 500, policyVersion: 'v1.3' }` → 완료까지 동일 snapshot 사용

---

## 📊 카테고리·심각도 분포

| 카테고리 | 개수 | 🔴 | 🟡 |
|---------|------|------|------|
| A. EntrySource | 4 | 3 | 1 |
| B. Intent | 5 | 3 | 2 |
| C. 역할 | 3 | 2 | 1 |
| D. Plan Depth | 2 | 1 | 1 |
| E. Beauty | 4 | 2 | 2 |
| F. Fallback | 2 | 0 | 2 |
| G. 기술·운영 | 5 | 4 | 1 |
| **합계** | **25** | **15** | **10** |

---

## 🎯 1차 봉인 순서 (구현 전 필수 15개)

구현 시작 **전** 규칙 확정 필수:

```
EC-01  다중 진입 파라미터 충돌
EC-02  Malformed entrySource
EC-03  리소스 삭제·비공개·만료
EC-05  Intent 0개 선택 CTA 차단
EC-06  Mixed derived flag (저장 X)
EC-07  Beauty on/off soft delete
EC-10  역할 토글 draft 분리
EC-12  Role 공통 vs role-specific 분리
EC-14  Depth 다운그레이드 hide-not-delete
EC-15  Beauty prefill canonical dedupe
EC-17  다운타임 혼합 primary+exception
EC-21  브라우저 뒤로가기 URL authoritative
EC-23  인증 만료 draft restore
EC-24  재생성 requestId dedupe
EC-25  정책 변경 request snapshot
```

---

## 🎯 2차 봉인 (구현 중 정리 10개)

```
EC-04  Mate_profile 진입 역할 기본
EC-08  Intent 재선택 Base Input 충돌
EC-09  Shared 링크 role 불일치
EC-11  로그인 자동 역할 vs 수동
EC-13  Depth 업그레이드 target vs fulfilled
EC-16  Beauty discovery mode
EC-18  Beauty prefill 자동 intent 체크
EC-19  샘플 fallback 우선순위
EC-20  AI 호출 타임아웃
EC-22  다중 탭 draftVersion
```

---

## 🔄 Config 매핑 (ADMIN_CONFIGURABLE_PARAMS 연계)

Edge case 처리 정책은 Runtime Config로 조정 가능해야 함:

```
# 우선순위
ai.planner.entry_source.priority      = ["mate_profile","venue","wishlist","shared","home"]

# Intent
ai.planner.intent.min_selection       = 1
ai.planner.intent.mixed_threshold     = 2   # derived flag 계산 기준

# 역할
ai.planner.role.draft_isolation       = true
ai.planner.role.common_fields         = ["dates","region","venue_refs"]
ai.planner.role.tourist_only_fields   = ["group_size","budget","mate_wish"]
ai.planner.role.mate_only_fields      = ["price","pair_format","intro"]

# Depth
plan.depth.track_target_vs_fulfilled  = true
plan.depth.downgrade_mode             = "hide"   # "hide" | "delete"

# Beauty
plan.beauty.dedupe_by                 = "canonical_id"
plan.beauty.discovery_mode_enabled    = true
plan.beauty.pattern_structure         = "primary_plus_exceptions"

# AI 요청
ai.request.dedup_window_seconds       = 3
ai.request.snapshot_on_start          = true
ai.request.timeout_warning_seconds    = 30
ai.request.timeout_retry_seconds      = 60

# 다중 탭
plan.multi_tab.detection_enabled      = true    # Phase 1: 감지만
plan.multi_tab.resolve_strategy       = "warn"  # Phase 2: "merge" | "latest_wins"
```

---

## 📝 구현 노트 — 복합 상태 머신으로서의 Planner

Genspark의 핵심 통찰 인용:

> "FestiMate AI Planner v1.0의 edge case 핵심은 **"입력값 검증"이 아니라 "상태 전이의 일관성"**입니다.  
> `/ai/plan`은 단일 화면처럼 보여도 실제로는 `entry context + intent state + role draft + depth state + beauty micro-state + AI request state`가 겹친 복합 상태 머신입니다."

이 관점에서 구현 시 **state machine 도구** 고려:
- `XState` (React state machines)
- `Zustand` + slice 패턴 (가벼움)
- Redux Toolkit (팀·도구 호환)

MVP는 `useReducer` + 명시적 action type으로 충분. Phase 2에서 XState 검토.

---

## 🚀 구현 순서 권장 (Phase A~D)

25개를 한꺼번에 처리하지 않는다. **상태 모델 → 보호 UX → 서버 계약 → QA 회귀** 순서로 겹겹이 쌓는다.

### Phase A — 상태 모델 먼저 (구현 착수 전 1~2일)

**목표**: 모든 edge case의 근본 원인인 "상태 전이"를 먼저 확정.

| 작업 | 대상 EC | 산출물 |
|------|--------|-------|
| `resolveEntrySource(params)` pure function | EC-01, EC-02 | priority 기반 driving context 1개 결정 |
| Intent reducer (multi-select + derived `mixed`) | EC-05, EC-06 | `useReducer` action: ADD/REMOVE/TOGGLE |
| Role-based draft store (`draft.tourist` / `draft.mate`) | EC-10, EC-12 | sessionStorage 또는 IndexedDB 2-key |
| `targetDepth` vs `fulfilledDepth` state machine | EC-13, EC-14 | `current_depth` + `depth_data.*` JSONB |
| Beauty micro-state (`active` + `treatments[]` + `discoveryMode`) | EC-07, EC-15, EC-17 | soft-delete pattern + canonical dedupe |

**검증**: state diagram 그리기 → 모든 전이가 복구 가능한지 시각적 확인.

---

### Phase B — 보호 UX (Phase A 완료 후 1~2일)

**목표**: 사용자 실수·네트워크 오류·인증 만료 상황에서 **draft 유실 제로**.

| 작업 | 대상 EC | 산출물 |
|------|--------|-------|
| 역할 전환 confirm modal | EC-10 | "현재 작성 내용을 저장하고 전환할까요?" |
| Beauty 해제 hidden draft 복원 | EC-07 | 재체크 시 자동 복원, 세션 종료 시만 제거 |
| 잘못된 참조 상단 배너 | EC-03 | "일부 참조를 불러올 수 없습니다" |
| AI 호출 loading/retry/cancel 토스트 | EC-20, EC-24 | 30초 경고, 60초 재시도, requestId 연계 |
| 다중 탭 감지 경고 | EC-22 | BroadcastChannel + draftVersion 비교 |

**검증**: Chrome DevTools로 네트워크 offline·throttle·세션 만료 시뮬레이션 통과.

---

### Phase C — 서버 계약 (Phase B 완료 후 1일)

**목표**: 클라이언트 상태가 서버에 도달할 때 **일관된 canonical payload**.

| 작업 | 대상 EC | 산출물 |
|------|--------|-------|
| Canonical request payload 스키마 확정 | EC-06, EC-12, EC-17 | derived flag 제외, primary+exceptions 구조 |
| `requestId` + `inputHash` 발급 | EC-24 | 중복 요청 dedupe window |
| 포인트·정책 precheck API | EC-25 | `requestSnapshot = { pointCost, policyVersion }` |
| Auth restore flow | EC-23 | `onAuthStateChanged` → `restoreDraftFromSession()` |

**검증**: Supabase Edge Function / Next.js Route Handler에서 payload 스키마 enforce (zod).

---

### Phase D — QA 회귀 (Phase C 완료 후 1~2일)

**목표**: 25개 edge case를 테스트 시나리오로 변환, P0 15개는 **자동화 or 수동 체크리스트** 필수.

- Unit test: state reducer, canonical dedupe, `resolveEntrySource` pure function
- Integration test: draft 복원, role 전환, beauty on/off
- E2E test (Playwright): 3~5개 핵심 시나리오만 (EC-07, EC-10, EC-21, EC-23, EC-24)
- Manual checklist: 아래 §QA 전략 참조

**봉인 기준**: P0 15개 전부 green → v1.0 릴리즈 가능.

---

## 🧪 QA 테스트 전략

### 테스트 레벨별 커버리지

| 레벨 | 도구 | 대상 EC | 비중 |
|------|------|--------|------|
| **Unit** | Vitest / Jest | EC-01, 02, 06, 15, 17 (순수 함수·reducer) | 70% |
| **Integration** | React Testing Library | EC-07, 10, 13, 14, 22 (상태 전이·UI) | 20% |
| **E2E** | Playwright | EC-21, 23, 24 (브라우저·네트워크) | 5% |
| **Manual** | 체크리스트 | EC-03, 09, 18, 25 (데이터 의존·정책) | 5% |

### 브라우저 우선순위

일본 20-30대 여성 타겟 → **모바일 퍼스트**.

| 우선순위 | 브라우저 | 이유 |
|---------|---------|------|
| P0 | **iPhone Safari** (iOS 17+) | 일본 iOS 점유율 ~70% |
| P0 | **Android Chrome** (최신) | 일본 Android 대부분 |
| P1 | Desktop Chrome | 기획·개발 검증 기본 |
| P2 | Desktop Safari | Mac 사용자 |
| P3 | Desktop Edge | 호환성 체크만 |

**특이사항**:
- iOS Safari는 sessionStorage·BroadcastChannel 제약 있음 → Phase B에서 fallback 경로 확인
- Android Chrome에서 한국 IP 막히는 Google 서비스 호출 주의

### 반드시 수동 검증할 항목 (자동화 어려움)

릴리즈 전 **사람이 직접 클릭해서 확인**:

- [ ] 두 탭에서 같은 draft 동시 편집 → 경고 노출 확인 (EC-22)
- [ ] 로그인 만료 → 저장 클릭 → 재로그인 → draft 복원 확인 (EC-23)
- [ ] Beauty 시술 3개(보톡스+울쎄라+스킨부스터) 혼합 → primary pattern B + exception 노출 확인 (EC-17)
- [ ] Tourist 공유 링크를 Mate 모드 탭에서 열기 → 2옵션 다이얼로그 확인 (EC-09)
- [ ] AI 생성 버튼 연타 → 1회만 호출되는지 Network 탭 확인 (EC-24)
- [ ] `/beauty`에서 시술 담아 `/ai/plan` 진입 → beauty intent 자동 체크 + 토스트 확인 (EC-18)
- [ ] 포인트 부족 상태에서 생성 시도 → paywall / recharge / later save 3옵션 확인 (EC-25)

### 릴리즈 기준

- **v1.0 봉인**: P0 15개 전부 green (Unit + Integration + Manual)
- **v1.1 허용**: P1 10개 중 8개 이상 green
- **P1 2개 미만 green**: 릴리즈 연기

---

## 🔗 관련 문서

- `AI_PLANNER_PAGE_DESIGN.md` v1.0 — 3-Layer 구조·필드·Plan Depth
- `ADMIN_CONFIGURABLE_PARAMS.md` v1.3 — Offering·Matching·Tier
- `AI_COST_MODEL.md` — Fallback·재생성·캐시
- `AI_TERMINOLOGY.md` — Mate Tier·Plan Depth·Intent

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|------|------|------|
| 2026-04-22 | 최초 작성 (v1.0) | Claude + Genspark 교차 검증 병합 (20+20 → 25) |
| 2026-04-22 | v1.1 | Phase A~D 구현 순서 + QA 테스트 전략 추가 (Genspark 거버넌스 경량 차용) |

---

*버전: v1.1 / 마지막 업데이트: 2026-04-22*
