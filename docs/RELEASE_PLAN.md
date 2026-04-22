# FestiMate 릴리즈 플랜

> **작성**: 2026-04-15
> **기준**: FESTIMATE_PLAYBOOK.md 로드맵 + ACCESS_CONTROL.md 결제·정책 + 2026-04-15 세션 결정
> **운영 전제**: 1인 + AI (ACCESS_CONTROL.md §1 #4)

---

## 1. 릴리즈 원칙

| # | 원칙 |
|---|---|
| 1 | **설계 먼저, 코드 나중** — 문서 정합성 확보 후 개발 착수 |
| 2 | **MVP는 콘텐츠 포털** — 결제·매칭 없이 검색·열람·AI Guide만으로 오픈 |
| 3 | **결제 순차 도입** — ACCESS_CONTROL.md §7 로드맵 순서 준수 |
| 4 | **법적 요건 선행** — 외국인환자유치업 등록 완료 전 `beauty` 카테고리 결제 불가 |
| 5 | **1인 운영 가능한 범위만** — 자동화 불가능한 기능은 연기 |

---

## 2. 마일스톤 총괄

| 마일스톤 | 목표 시점 | 핵심 산출물 | 선행 조건 |
|---|---|---|---|
| **M0: 설계 완료** | 2026-04월 말 | 설계 문서 정합성 100%, DB 스키마 확정 | 현재 진행 중 |
| **M1: 인프라 구축** | 2026-05월 | Next.js + Supabase + Vercel 배포, CI/CD | M0 |
| **M2: 콘텐츠 MVP** | 2026-06월 | 카테고리 검색·열람, 다국어, 소셜 로그인 | M1 |
| **M3: AI Guide** | 2026-07월 | AI 일정 생성, 지도 뷰 | M2 |
| **M4: 공급자 포털** | 2026-08월 | `/host` 업로드 위자드, 관리자 검수 | M3 |
| **M5: 매칭 MVP** | 2026-09월 | 메이트 포털, FF/MF 페어, 세션 관리 | M4 + 청주대 MOU |
| **M6: 결제 v1** | 2026-10월 | 명인 쇼핑 PG, 포인트 결제 | M4 + PG 계약 |
| **M7: 파일럿** | 2026-09~11월 | 청주 허브 실제 운영 (가을 성수기) | M5 |

---

## 3. M0: 설계 완료 (현재 ~ 2026-04월 말)

### 완료 항목
- [x] FESTIMATE_PLAYBOOK.md — 사업 설계 SSOT (v1.0)
- [x] SITEMAP_SPEC.md — 페이지 구조·URL (v1.0)
- [x] MEMBERSHIP_TIERS.md — 등급 체계 (v1.1)
- [x] PLAN_TEMPLATE_SCHEMA.md — 플랜 JSON (v0.1)
- [x] SUPPLIER_UPLOAD_SPEC.md — 공급자 폼 (v1.0)
- [x] CONSISTENCY_AUDIT.md — 정합성 감사 9건
- [x] CATEGORY_MAPPING.md — 카테고리 SSOT (v1.0)
- [x] GLOSSARY.md — 용어 사전 (v1.0)
- [x] ACCESS_CONTROL.md — 접근 권한·결제·정책 (v2.0)
- [x] RELEASE_PLAN.md — 본 문서
- [x] LEGAL_CHECKLIST.md — 법적 체크리스트
- [x] INFRA_STATUS.md — 인프라 현황

### 잔여 항목
- [ ] 잔여 감사 이슈 3건 해결 (P1-2, P2-2, P2-3)
- [ ] DB 스키마 v1 확정 (Supabase 테이블 설계)
- [ ] API 설계 v1 (엔드포인트·인증·에러 코드)
- [ ] 디자인 시스템 기초 (색상·타이포·컴포넌트)

---

## 4. M1: 인프라 구축 (2026-05월)

| 항목 | 선택지 | 비고 |
|---|---|---|
| 프레임워크 | Next.js 16 (App Router) | 기존 코드 기반 |
| DB | Supabase (PostgreSQL + pgvector + PostGIS) | 무료 티어 시작 |
| 호스팅 | Vercel (무료 → Pro) | Next.js 최적 |
| 파일 스토리지 | Supabase Storage | S3 호환, 무료 1GB |
| AI | OpenAI GPT-4o | API 비용 모니터링 필수 |
| 인증 | Supabase Auth + Google/Kakao/LINE OAuth | ACCESS_CONTROL §2.2 |
| CI/CD | GitHub Actions | 자동 빌드·배포 |
| 도메인 | festimate.ai (보유) | |
| 모니터링 | Vercel Analytics (무료) | |

### 산출물
- [ ] 프로젝트 초기화 (Next.js + TypeScript + Tailwind)
- [ ] Supabase 프로젝트 생성 + 스키마 마이그레이션
- [ ] Vercel 배포 파이프라인
- [ ] 환경변수 설정 (API 키는 .env.local, 레포 기록 금지)
- [ ] 다국어 i18n 셋업 (jp/en/kr)

---

## 5. M2: 콘텐츠 MVP (2026-06월)

오픈 시 **결제·매칭 없음**. 콘텐츠 포털로 트래픽 확보.

### 포함 기능
- [ ] 홈 (히어로 + 카테고리 카드 + 축제 하이라이트)
- [ ] 카테고리 리스트·필터·지도 토글 (SITEMAP_SPEC §5)
- [ ] 상품 상세 (갤러리·설명·위치·"AI Guide로 조합하기" CTA)
- [ ] 소셜 로그인 (Google/Kakao/LINE) → Social 계층
- [ ] 위시리스트
- [ ] 다국어 (jp/en/kr) 자동 전환
- [ ] 반응형 (모바일 퍼스트)
- [ ] SEO (hreflang, Schema.org)

### 미포함 (후속 마일스톤)
- AI Guide, 예약, 결제, 매칭, 공급자 포털, 관리자 대시보드

### 콘텐츠 초기 데이터
- KTO API → `festival` 카테고리 자동 수집
- 수동 시드: 이용강 명장, 법주사, 강남 피부과 3곳 (상품 상세)

---

## 6. M3~M7 요약

### M3: AI Guide (2026-07월)
- AI 일정 생성 (PLAN_TEMPLATE_SCHEMA 기반)
- 지도 + 타임라인 뷰
- Social 1회/일, Verified 5회/일

### M4: 공급자 포털 (2026-08월)
- `/host` 업로드 위자드 9-Step (SUPPLIER_UPLOAD_SPEC)
- 관리자 검수 큐
- AI 번역 (한→일·영)

### M5: 매칭 MVP (2026-09월)
- `/mate-portal` 메이트 가입·페어·오리엔테이션
- From Mate / From Tourists
- 체크인·체크아웃·평가
- 긴급 버튼 (외부 기관 안내)

### M6: 결제 v1 (2026-10월)
- 명인 쇼핑 PG (토스페이먼츠)
- 포인트 결제 (30% 상한)
- 정산 시스템

### M7: 파일럿 (2026-09~11월)
- 청주 허브: 이용강 명장 + 법주사 + 강남 피부과
- 청주대 메이트 20~40명
- 일본 인플루언서 3~5명 초청

---

## 7. 리스크 & 의존성

| 리스크 | 영향 | 완화 |
|---|---|---|
| 외국인환자유치업 등록 지연 (60~90일) | M6 `beauty` 결제 차단 | 등록 즉시 신청, `beauty`는 정보 제공만 선행 |
| 청주대 MOU 미체결 | M5 매칭 불가 | 대안 학교 병행 접촉 (한국교통대, 충북대) |
| AI API 비용 초과 | 운영비 부담 | 계층별 횟수 제한 + 관리자 알림 + 캐싱 |
| 변호사 검토 결과 K-뷰티 차단 필요 | M2 재작업 | 관리자 플래그로 대응 (ACCESS_CONTROL §8.3) |
| 1인 운영 한계 | 검수·CS 병목 | AI 1차 스크리닝 극대화, 검수 기준 자동화 |

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|---|---|---|
| 2026-04-15 | 최초 작성 | 세션 결정 반영, 마일스톤 7단계 정의 |

---

*버전: v1.0 / 작성: 2026-04-15*
