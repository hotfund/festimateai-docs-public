# AI 용어 정의 (AI Terminology)

> **작성**: 2026-04-19
> **위상**: 용어 일관성 SSOT — 코드·문서·마케팅 전체에서 참조
> **선행 문서**: AI_ROLE_DEFINITION.md, AI_COST_MODEL.md
> **현재 상태**: v0.2 (매칭 플로우·쌍방 승인·쌍방향 이니시에이팅 반영)

---

## 1. 목적

사용자·엔지니어·문서 간 **용어 일관성 확보**. 3가지 AI 역할과 2가지 유저 역할을 명확히 정의.

---

## 2. 3가지 AI 역할 (Internal 용어)

### 🗓️ AI Planner — 여행 전

| 항목 | 내용 |
|------|------|
| 작동 시점 | 여행 결정 ~ 예약 완료 전 |
| 주요 사용자 | Tourist (주) + **Local Mate (부, 제안서 작성용)** |
| 핵심 기능 | 여행 일정 생성 · 제안서 작성 |
| 입력 | 조건·선호·담아온 장소 |
| 출력 | Day-by-Day 플랜 (Tourist) 또는 투어 제안서 (Mate) |
| 모델 | `gpt-4o-mini` + validator fallback |
| 구현 | 현재 `/ai-guide` (추후 `/ai/plan` 이동 예정) |

### 🤝 AI Matcher — 매칭 단계

| 항목 | 내용 |
|------|------|
| 작동 시점 | 플랜/제안서 게시 후 ~ 매칭 확정 전 |
| 주요 사용자 | Tourist ↔ Local Mate |
| 핵심 기능 | 양쪽 요청·제안 데이터 기반 최적 페어링 |
| 입력 | Tourist 플랜 + Mate 제안서 |
| 출력 | 추천 매칭 리스트 (Tourist → Mate, Mate → Tourist) |
| 모델 | **LLM 최소 — 벡터 유사도 (pgvector) 중심** |
| 구현 | 미구현 (AI Planner 페이지 내 섹션 + `/ai/matching` 신규) |

### 🧭 AI Guide — 여행 중

| 항목 | 내용 |
|------|------|
| 작동 시점 | 여행 출발 후 ~ 종료 |
| 주요 사용자 | Tourist (Mate 동행 시 보조, 솔로 시 전면) |
| 핵심 기능 | 실시간 안내·번역·위치 기반 추천·긴급 대응 |
| 입력 | 현재 위치·시간·플랜 진행·사용자 질문·사진 |
| 출력 | 실시간 안내 메시지·알림·리플랜 |
| 모델 | `gpt-4o-mini` + vision (카메라 입력) |
| 구현 | **미구현 (M3 2026-07월 예정)** |
| 기술 특징 | **PWA 모바일 중심, 위치·카메라·푸시 알림** |

---

## 3. 2가지 유저 역할

### 🧳 Tourist (수요측)

- **정의**: 한국 여행을 준비·진행하는 방문자
- **주 페르소나**: 일본 20~30대 여성 (primary), 영어권 (secondary)
- **핵심 니즈**: 현지 경험 + 안전 + 편의 + 현지인 연결
- **Planner 사용**: 여행 조건 입력 → 플랜 생성
- **Matcher 사용**: 추천 Mate 브라우징 → 매칭 요청 (또는 Mate 요청 수신)
- **Guide 사용**: 여행 중 상시 동반

### 🎎 Local Mate (공급측)

- **정의**: Tourist에게 현지 경험을 제공하는 한국 거주자
- **주 페르소나**: 한국 대학생 (청주대 포함), 일본어·영어 가능
- **티어 시스템**: Applicant → Rookie → Verified → Featured → Pro Guide (MEMBERSHIP_TIERS.md)
- **Planner 사용**: 제공 가능한 투어 제안서 작성
- **Matcher 사용**: 추천 Tourist 요청 브라우징 → 매칭 수락 (또는 Tourist에게 먼저 제안)
- **Guide 사용**: 세션 중 AI 통역·정보 보조

---

## 4. 핵심 개념 구분

### 📋 플랜 (Plan) vs 📝 제안서 (Offering)

| | Plan | Offering |
|---|------|---------|
| 작성자 | Tourist | Local Mate |
| 내용 | "이런 여행 하고 싶어요" | "이런 투어 제공할 수 있어요" |
| 예시 | "5/14~17 서울, 여성 2, 축제+K뷰티" | "5월 주말 서울, FF 페어, 일본어 가능, 축제 동반" |
| 저장소 | `plans` 테이블 | `offerings` 테이블 |
| 매칭 대상 | `offerings` 검색 | `plans` 검색 |
| Publish 단위 | 1 Tourist = N plans (여러 여행 가능) | 1 Mate = Tier별 제한 (§6 참조) |

### 🎯 5단계 매칭 플로우 (확정)

```
1. 탐색 (Exploration) — 무료, Tier 1 정보만 공개
   ├ Tourist: Plan 게시 → AI Matcher 인덱싱
   └ Mate:    Offering 게시 → AI Matcher 인덱싱
   → 양쪽 /ai/matching 에서 상대방 요청·제안 브라우징 가능

2. 관심 표시 (Interest) — 무료, 쌍방향 이니시에이팅
   ├ Tourist → Mate에게 "❤️" 클릭 가능
   ├ Mate → Tourist에게 "❤️" 클릭 가능
   └ 어느 쪽이든 먼저 할 수 있음
   
   ✅ 상호 ❤️ 성사 조건:
   → 양쪽 모두 클릭해야 매칭 성사
   → 성사 시 Tier 2 정보 양쪽 공개 (자기소개·SNS·시간대 등)
   → 내장 메시징 자동 활성화
   → 푸시·이메일 알림 발송

3. 예약 확정 (Booking) — 포인트 차감
   ├ 메시징으로 일정·세부 조율
   ├ 양쪽 합의 → Tourist 포인트 차감 → 예약 확정
   └ Tier 3 정보 양쪽 공개 (얼굴·연락처·학적)

4. 세션 진행 (Session)
   ├ 양쪽 모바일 체크인 (GPS 확인)
   ├ AI Guide 활성화 (Role 3)
   └ 체크아웃 → 자동 정산

5. 리뷰 (Review) — 48h 내 필수
   ├ 양쪽 서로 평가
   ├ 평점 → Mate 티어 자동 업데이트
   └ 포인트 적립 (리뷰 작성 reward)
```

### 🔄 쌍방향 이니시에이팅 (확정)

매칭 페이지(`/ai/matching`)는 수직 분할 구조:
- **좌**: Tourist 요청 리스트 (Mate 관점에서 브라우징)
- **우**: Local Mate 제안 리스트 (Tourist 관점에서 브라우징)
- 어느 쪽이든 먼저 ❤️ 클릭 가능 → 알림 → 상대방 ❤️ 응답 시 성사

### 💰 취소·환불·No-show 정책

ADMIN_CONFIG §3-11에서 조정:

| 시점 | 환불 비율 |
|------|---------|
| 세션 72시간 전 | 100% |
| 24~72시간 | 50% |
| 24시간 미만 | 0% |
| Mate 측 취소 | Tourist 100% 환불 + Mate 평점 감점 |

**No-show**:
- Mate: Tourist 환불 + Mate 경고 (3회 자격 정지)
- Tourist: 환불 없음 + 경고

---

## 5. 3-Tier 정보 공개 모델 ⭐

### Tier 1: 🌐 무조건 공개 (누구나 매칭 페이지에서 볼 수 있음)

양쪽 동일하게:
- 성별
- 연령대 (20대 초/중/후 단위)
- 지역 (희망 or 가능)
- 일정 (기간·주말 가능 등)
- 인원 수
- 관심사·카테고리
- 예산 범위 (구간)
- 언어
- 한 줄 소개

### Tier 2: 🔓 상호 관심 해제 (무료, 양쪽 "❤️" 클릭 시 자동)

- 자기소개 상세 (한 문단)
- Mate 평점·세션 횟수
- SNS 핸들 (선택 공개)
- 구체 희망 시간대
- 선호 페어 포맷 (FF·MF 등)
- 방문 목적 디테일

**포인트 차감 없음.** 상호 동의가 이미 스팸 방지 역할.

### Tier 3: 💎 예약 확정 시 공개 (포인트 차감 + 매칭 확정 후)

- 얼굴 사진
- 정확한 나이
- 본명 (닉네임 대체)
- 연락처 (LINE ID·전화)
- 학적·소속 (Mate)
- 여권·신분 인증 배지 (Tourist, Verified 사용자만)

**법적 함의**: Tier 3는 민감정보 유사 취급. 명시적 동의 + 수집·보관 고지 + 파기 기한 명시.

### Config 매핑 (ADMIN_CONFIGURABLE_PARAMS 연동)

```
matching.tier1_fields = ["gender","age_range","region","dates","group_size",
                          "interests","budget_range","language","one_line_intro"]
matching.tier2_fields = ["bio_detail","sns_handle","rating","session_count",
                          "time_preference","pair_format","visit_detail"]
matching.tier3_fields = ["photo","exact_age","real_name","contact",
                          "affiliation","identity_verified"]
matching.tier2_unlock_mode = "mutual_interest"
matching.tier3_unlock_mode = "booking_confirmed"
matching.tier3_requires_point_spend = true
matching.approval_mode = "mutual"
matching.initiating_mode = "bidirectional"
```

→ `ADMIN_CONFIGURABLE_PARAMS.md` v1.3 §3-3에서 관리.

---

## 6. Offering 2축 모델 (Tier별 제한)

Mate가 생성·보유할 수 있는 Offering(제안서) 수:

| Tier | 유효기간 (TTL) | 동시 활성 수 | 연장 가능 |
|------|-------------|-----------|---------|
| Applicant | N/A | 0 (불가) | - |
| Rookie | 30일 | 1 | X |
| Verified | 60일 | 3 | X |
| Featured | 90일 | 5 | X |
| Pro Guide | 180일 | 10 | ✅ |

**원칙**:
- 월별 생성 throttle 없음 (동시 활성 상한이 역할 대체)
- 기간 만료 시 자동 비활성화 (Pro Guide만 연장 옵션)
- 모든 수치 `ADMIN_CONFIGURABLE_PARAMS §3-3` config화

---

## 7. 내부 용어 vs 외부 UI 표기

### 내부 사용 (코드·DB·문서·개발 대화)

```
AI Planner · AI Matcher · AI Guide
Tourist · Local Mate
Plan · Offering · Matching · Booking
Tier 1 · Tier 2 · Tier 3
```

### 외부 사용 (UI·마케팅·고객 대면)

| 용어 | 한국어 | 일본어 | 영어 |
|------|-------|-------|------|
| AI Planner | AI 플래너 | AIプランナー | AI Planner |
| AI Matcher | AI 매처 | AIマッチャー | AI Matcher |
| **AI Guide** | AI 여행 동반자 | AI 旅のパートナー | AI Travel Companion |
| Tourist | 여행자 | 旅行者 | Traveler |
| Local Mate | 로컬 메이트 | ローカルメイト | Local Mate |

**⚠️ "AI Guide"는 내부 전용**. 외부 표기는 관광진흥법 §38 회피 위해 "AI 여행 동반자 / Travel Companion" 사용.

---

## 8. 법적 고려사항

### 관광진흥법 §38 (통역안내사)
- AI 자체는 법 적용 대상 아님 (자연인 대상)
- 하지만 마케팅 멘트 주의:
  - ❌ "AI Guide가 통역안내사 역할을 대신합니다"
  - ❌ "전문 가이드 없이도 여행 가능"
  - ✅ "여행 도우미 역할"
  - ✅ "여행 중 정보·번역 지원"

### 개인정보보호법
- Tier 3 정보 수집·보관 시 별도 동의
- 정기 파기 기한 고지 (예: 예약 완료 후 1년)
- 민감정보 (의료·시술 등) 특별 취급

### 의료법 §56 (의료광고)
- AI Planner가 K-뷰티 플랜 생성 시 치료 효과 단정 금지
- "의료 상담은 병원 동의 후 진행" 명시
- `kbeauty.medical_disclaimer_version` config 준수

---

## 9. URL 구조 (잠정 채택, 구현 시 최종)

**Option B (중첩)** — 잠정:
```
/ai/plan       AI Planner (Tourist + Mate 공용 canvas)
/ai/matching   AI Matcher (수직 분할 리스트)
/ai/guide      AI Guide (여행 중, PWA)

/mate          Mate 공급측 랜딩 + 가입 통합
/register      Tourist 기본 가입 (+ Mate 선택 시 /mate 안내)
/mate-portal/* Mate 전용 대시보드
/beauty        K-뷰티 전용 페이지 (메인 수익원)
```

---

## 10. 변경 영향 — 기존 문서 업데이트

### 업데이트 필요

- [ ] `AI_ROLE_DEFINITION.md` — 3 모드 이름 이 문서 용어로 일치 (PLAN → AI Planner)
- [x] `ADMIN_CONFIGURABLE_PARAMS.md` v1.3 — Tier 1/2/3 필드 + Offering 2축 + matching policy (Session 3 완료)
- [ ] `SITEMAP_SPEC.md` — URL 구조 확정 시 반영
- [ ] `LEGAL_CHECKLIST.md` — Tier 3 개인정보 처리 항목 추가
- [ ] `SITE_COMPLETION_PLAN.md` — `/ai-guide` → `/ai/plan` 이동 작업 추가

### 코드 영향

- 현재 `app/src/app/ai-guide/page.tsx` → 용어상 "Guide" 아닌 "Planner"
- 리네이밍 필요: 파일 구조 + 네비게이션 링크
- 타이밍: 입력폼 5개 축소 작업할 때 동반

---

## 11. 미결 사항 (다음 세션)

다음 세션에 결정:

1. ✅ **~~AI Planner Mate 모드 입력 필드~~** — 해소: 5필드 옵션 C (제공기간·페어·가격·지역·자기소개, Tourist와 1:1 대칭) / `AI_PLANNER_PAGE_DESIGN.md` §5
2. ✅ **~~AI Planner 출력 포맷~~** — 해소: Day 카드 기본 + 日程表/地図/比較 3-뷰 토글 / `AI_PLANNER_PAGE_DESIGN.md` §8
3. ✅ **~~AI Planner 액션 버튼~~** — 해소: AI Matching·저장·공유·재생성 4개 (Sticky bottom) / `AI_PLANNER_PAGE_DESIGN.md` §8
4. **매칭 페이지 UX 세부** — 페이지 레이아웃, 카드 디자인, 필터·정렬, ❤️ 이후 플로우, 메시징 UX, 예약 UI, 체크인 UX, 리뷰 UX 등 10+개
5. **URL 구조 최종** — `/ai/plan·/ai/matching·/ai/guide` 잠정 채택
6. **AI Guide (TRIP 모드) PWA 구현 세부** — M3 (2026-07월)
7. **/mate 페이지 구체 디자인** — 랜딩+가입 통합 레이아웃

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|------|------|------|
| 2026-04-19 | Draft v0.1 최초 작성 | 사용자 용어 정리 요청 + 3-Tier 공개 모델 확정 |
| 2026-04-19 | v0.2 — §4 5단계 매칭 플로우·쌍방 ❤️·쌍방향 이니시에이팅 확정 반영, §6 Offering 2축 모델 추가 | Session 3 후반 매칭 관련 3개 결정 SSOT 반영 |
| 2026-04-19 | v0.2.1 (patch) — §11 미결 1~3번 해소 표시 | `AI_PLANNER_PAGE_DESIGN.md` v0.1 작성으로 AI Planner 세부 확정 |

---

*버전: v0.2.1 / 마지막 업데이트: 2026-04-19*
