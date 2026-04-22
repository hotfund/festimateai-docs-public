# 사이트 완성 계획 (Site Completion Plan)

> **작성**: 2026-04-19
> **최종 갱신**: 2026-04-19 (Session 2 — K뷰티 서브플랜 확장 · i18n P1 승격 · admin config v0 추가)
> **배경**: CEO Review에서 FestiMate 원안 유지 결정. 파트너(청주대 등) 미팅을 위해서는 동작하는 데모 사이트가 선행 조건.
> **목표**: 3주 안에 "festimate.ai 접속하면 일본인 여행자 관점과 한국 메이트 관점 둘 다 이해되는 사이트"
> **철학**: 완벽한 프로덕트 X, 첫인상이 프로페셔널한 MVP O

---

## 1. 현재 상태 (2026-04-19 기준)

### 완료된 것 (2026-04-17 세션 기준)

9개 페이지 스캐폴드 + 주요 기능 구현:

| URL | 설명 | 상태 |
|---|---|---|
| `/` | 홈 (히어로+검색+카드+메이트+AI Guide) | 스캐폴드 완료 |
| `/explore` | 탐색 (월별+지역+뷰티+명인+템플 교차 필터) | 스캐폴드 완료 (통합 예정) |
| `/ai-guide` | AI Guide 채팅 (OpenAI GPT-4o 연동) | 실동작 |
| `/mate` | 매칭 (From Mate / From Tourist 모집글) | 스캐폴드 완료 |
| `/plan/demo` | 여행 플랜 템플릿 (지도뷰+일정표뷰) | 스캐폴드 완료 |
| `/login` | 로그인 (Google/Kakao/LINE) | 스캐폴드 완료 |
| `/search` | 검색 (Supabase 축제 DB 연동) | 실동작 |
| `/auth/callback` | OAuth 콜백 | 완료 |
| `/api/ai-guide` | AI API 엔드포인트 | 완료 |

**최근 커밋** (git log 기준):
- `70081c5` feat: TourAPI 동기화 스크립트 + 축제별 다중 이미지 수집 추가
- `f06ac8d` feat: Supabase 스키마 파일 추가

### 남은 것

로컬 환경에서만 작동. 외부 공유 불가. 실데이터 부족. 도메인 미연결.

---

## 2. 할일 리스트

### P0 — 이번 주 (사이트 "존재"하게 만들기, 약 3~4시간)

- [ ] **P0-1** festimate.ai DNS → Vercel 연결 (30분)
  - Vercel 대시보드에서 도메인 추가
  - DNS 레코드 (A 또는 CNAME) 설정
  - SSL 자동 프로비저닝 확인
- [ ] **P0-2** Vercel 프로덕션 배포 현재 9페이지 (1시간)
  - `vercel --prod` 또는 main 브랜치 자동 배포
  - 빌드 에러 0건 확인
- [ ] **P0-3** 환경변수 프로덕션 세팅 (1시간)
  - Supabase URL·anon key
  - OpenAI API key (+ `ai.model.*` 기본 mini 설정 확인)
  - Google/Kakao/LINE OAuth client id·secret
  - `.env.local.example` 기준 누락 변수 체크
- [ ] **P0-4** 홈 히어로 이미지 최소 1장 진짜로 (1시간)
  - Unsplash에서 한국 전통시장·축제 야경·한복 이미지 찾기
  - DESIGN.md §Content Guidelines 준수 (클럽/파티 금지)
  - WebP 변환, 2x 해상도
- [ ] **P0-5** 기본 SEO 메타태그 (30분)
  - `<title>`, `<meta description>`, `og:image`, `og:title`, `twitter:card`
  - 홈·AI Guide·Mate 3개 페이지 최소

### P1 — 다음 주 (사이트 "설명 가능"하게 만들기, 약 18~22시간) ⭐ Session 2 재편

- [ ] **P1-6** 실데이터 축제 30개 (2시간)
  - 기존 TourAPI 스크립트(`70081c5`) 실행
  - Supabase festivals 테이블 채우기
  - 홈·search 페이지에서 실제 노출 확인

- [ ] **P1-7** 실데이터 명인명장 5명 (1시간)
  - `src/data/masters.ts`에서 이용강 포함 상위 5명 추림
  - 상세 페이지 렌더링 확인

- [ ] **P1-8 K-뷰티 서브플랜** ⭐ Session 2 확장 (메인 수익원 격상)
  - [ ] **P1-8a** `/jp/beauty` 랜딩 페이지 (3시간)
    - 홈 히어로 아래 프라임 섹션에서 진입
    - 인기 시술 TOP 5 + VIP 피부과 10선
    - 의료 면책 고지 (LEGAL_CHECKLIST 연동)
  - [ ] **P1-8b** 의료 상담 신청 폼 (2시간, 법무 검토 후)
    - 최소 필드 수집 (이름·연락처·관심 시술·희망 일정)
    - 개인정보처리방침 동의 필수
    - `kbeauty.min_consultation_lead_hours` 48시간 기본
  - [ ] **P1-8c** 클리닉 상세 페이지 템플릿 (2시간)
    - 클리닉 10곳 데이터 입력 (강남 주요)
    - 일본어 응대 여부 배지
    - 리뷰·가격·시술 목록

- [ ] **P1-9** 한국어 진입 페이지 `/kr` (3시간)
  - Mate 관점 메시지 (DESIGN.md §Language-Aware Messaging)
  - 청주대 담당자 이해 가능한 1페이지

- [ ] **P1-10** 회사 소개 페이지 `/about` (2시간)
  - FestiMate 비전·팀·로드맵
  - 파트너 신뢰 지표

- [ ] **P1-11** 연락처/문의 폼 `/contact` (1시간)
  - 이메일 릴레이 (Resend or Formspree)

- [ ] **P1-12** 모바일 반응형 QA (2시간)
  - 홈·AI Guide·Mate·Beauty 4개 페이지
  - iPhone SE / iPhone 15 Pro / iPad 3개 뷰포트

- [ ] **P1-13 i18n 3개 언어 기반** ⭐ Session 2 P3→P1 승격 (4시간)
  - `next-intl` 설치 + 라우팅 (`/jp`, `/en`, `/kr`)
  - `locale.default = "jp"`, `locale.available = ["jp","en","kr"]` config
  - 홈·beauty 랜딩·about·contact 4개 페이지 JP 우선 번역
  - EN·KR은 폴백 체인 (빈 값이면 JP 표시)
  - Accept-Language + IP 감지 리다이렉트

### P2 — 2~3주차 (사이트 "시연 가능"하게 만들기, 약 20시간) ⭐ Session 2 재편

- [ ] **P2-14** 카카오맵 API 실연동 플랜 지도뷰 (4시간)
- [ ] **P2-15** AI Guide 데모 플로우 검증 (2시간)
  - 샘플 질문 3개 → 실제 답변 확인 (`ai.model.plan = gpt-4o-mini`)
  - 에러 케이스 폴백
- [ ] **P2-16** OAuth 실동작 확인 (2시간)
  - Google 1개만이라도 완전 작동
- [ ] **P2-17** Mate 소개 섹션 `/kr/mate` (3시간)
- [ ] **P2-18** 기본 법적 페이지 (2시간)
  - 이용약관 템플릿
  - 개인정보처리방침 템플릿
  - **의료 면책 고지** (K-뷰티 전용) ⭐ Session 2
- [ ] **P2-19** Lighthouse 점수 체크 홈 기준 70+ (1시간)
- [ ] **P2-20** 파비콘·오픈그래프 이미지 (1시간)
- [ ] **P2-21** `/admin/config/*` v0 ⭐ Session 2 신규 (5시간)
  - `platform_config` 테이블 마이그레이션 + 기본값 seed
  - 조회·수정 UI (pricing·ai·locale·kbeauty 4개 탭만 우선)
  - 변경 이력 로그
  - 관리자 인증 (임시: 환경변수 화이트리스트)

### 지금은 미루기 (P3+)

- 결제 시스템 (M6 2026-10월 계획)
- Mate 포털 `/mate-portal` 구현 (MATE_RECRUITMENT_PLAN P2-14)
- 공급자 업로드 위자드 (M4 2026-08월)
- 관리자 대시보드 풀버전 (P2-21은 v0만)
- KoJap·TW 라우트 (Phase 2)
- 카카오맵 완전 커스터마이징

---

## 3. 타임라인 요약 ⭐ Session 2 재계산

```
주차 1: P0  (3~4시간)    → 도메인·배포·기본 인상
주차 2: P1  (18~22시간)  → 실데이터·K뷰티 서브플랜·i18n·한국어 진입점·모바일
주차 3: P2  (20시간)     → 카카오맵·OAuth·법무 페이지·admin config v0
────────────────────────────────────────────
총 소요: 약 41~46시간 = 집중 6~7일치 (기존 30~33에서 증가)
결과: 청주대·파트너 미팅 가능 수준 + K뷰티 파트너 병원 접촉 가능
```

---

## 4. 한국 국내 확장 대비 아키텍처 고려사항

> 이 섹션은 **지금 구현 X, 설계 단계에서 막히지 않도록 씨앗만** 심어두는 용도.
> CEO Review에서 "한국 국내 동행 매칭" 아이디어가 나왔고 향후 확장 가능성 있음.

### 4-1. 라우팅 구조에 시장 분기 공간 열어두기

```
festimate.ai/jp/*       → 일본인 인바운드 (현재 메인)
festimate.ai/kr/*       → 한국 메이트 관점 + 향후 한국 국내 확장
festimate.ai/en/*       → 영어
festimate.ai/tw/*       → 대만 (Phase 2, 미구현)
```

현재는 `/jp`만 진짜 구현. `/kr`은 Mate 랜딩페이지 1장만. 라우팅 틀은 처음부터 열어둠. 나중에 `/kr/domestic` 같은 섹션 추가 용이.

### 4-2. DB 스키마에 타겟 마켓 필드 예약

`venues`, `plans`, `matches` 테이블에 `target_market` enum 컬럼 추가:
- 값: `inbound_jp` / `inbound_en` / `domestic_kr` / `outbound_kojap`
- 지금은 전부 `inbound_jp`로 채움
- 향후 한국 국내 확장 시 `domestic_kr` 추가만 하면 UI는 필터링만 바꾸면 됨

### 4-3. Matching 로직을 "persona pair" 추상화

```typescript
interface MatchingRule {
  demand_persona: 'japanese_female_tourist' | 'korean_middle_age' | ...
  supply_persona: 'korean_student_mate' | 'korean_peer' | ...
  pair_format: 'FF' | 'MF' | 'GROUP' | ...
}
```

지금은 하드코딩해도 되지만, 스키마를 이렇게 열어두면 한국 국내 동행 매칭 추가 시 규칙만 추가하면 됨.

---

## 5. 연관 문서

- `FESTIMATE_PLAYBOOK.md` — 사업 설계 SSOT
- `DESIGN.md` — 디자인 시스템 SSOT
- `SITEMAP_SPEC.md` — 전체 사이트맵 (이번 미팅용 MVP는 이 중 일부만)
- `SEARCH_PAGE_DESIGN.md` — /search 상세 설계
- `AI_ROLE_DEFINITION.md` — AI 3가지 역할 + LLM 최소 원칙
- `ADMIN_CONFIGURABLE_PARAMS.md` — 관리자 조정 파라미터 (P2-21 입력)
- `MATE_RECRUITMENT_PLAN.md` — 사이트 완성 후 착수할 Mate 공급 계획
- `LEGAL_CHECKLIST.md` — 법무 체크리스트 (K뷰티 항목 추가 예정)
- `RELEASE_PLAN.md` — 전체 마일스톤 (M0~M7)
- `docs/BEAUTY_PAGE_DESIGN.md` — ⭐ **P1-8 실행 전 신규 작성 필요** (K-뷰티 전용 페이지 상세 설계)

---

## 6. 체크포인트

**다음 세션 시작 시**:
1. `/checkpoint` 스킬로 진행 상황 복원
2. 이 문서의 "할일 리스트" 중 체크 안 된 가장 상단 항목 실행
3. 완료 시 이 문서 업데이트 후 커밋

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|------|------|------|
| 2026-04-19 | 최초 작성 | CEO Review 후 사이트 완성 3주 플랜 |
| 2026-04-19 (Session 2) | P1-8 K뷰티 8a/8b/8c로 확장, P1-13 i18n 신규 (P3→P1), P2-21 admin config v0 신규, 타임라인 재계산 30→41시간 | CLI 세션에서 K뷰티 격상 + i18n 우선 + config UI 우선 확정 |

---

*버전: v1.1 / 마지막 업데이트: 2026-04-19*
