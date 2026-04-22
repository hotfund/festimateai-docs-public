# FestiMate 인프라 현황

> **작성**: 2026-04-15
> **용도**: 현재 인프라 상태 추적. 무엇이 있고, 무엇이 없는지 한눈에 파악
> **주의**: API 키·시크릿은 이 문서에 기록 금지. 환경변수는 `.env.local`에만 보관

---

## 1. 현재 상태 요약

```
상태: 설계 단계 (코드 없음)
레포: github.com/festimateai/festimateai
브랜치: main
내용: 설계 문서 12개 + 세션 로그 1개
코드: 없음 (Next.js 프로젝트 미초기화)
DB: 없음 (Supabase 프로젝트 미생성)
배포: 없음 (Vercel 미연결)
도메인: festimate.ai (보유, DNS 미설정)
```

---

## 2. 보유 자산

### 2.1. 레포지토리

| 항목 | 값 |
|---|---|
| GitHub Org | `festimateai` |
| 레포 | `festimateai/festimateai` |
| 브랜치 | `main` |
| 가시성 | Private (추정) |
| 설계 문서 | 12개 (`docs/`) |
| 세션 로그 | 1개 (`logs/`) |
| 코드 | **없음** |

### 2.2. 도메인

| 항목 | 값 |
|---|---|
| 메인 도메인 | `festimate.ai` |
| DNS 설정 | **미완료** (Vercel 연결 시 설정 필요) |
| SSL | Vercel 자동 (연결 후) |

### 2.3. 기존 코드 (별도 레포)

| 항목 | 값 |
|---|---|
| 레포 | `hotfund/festimate` |
| 스택 | Next.js 16 + Supabase + OpenAI |
| 상태 | 기존 프로덕트 (리뉴얼 대상) |
| 활용 | 참조용. 새 레포에서 재설계 |

### 2.4. 외부 계정 (확인 필요)

| 서비스 | 계정 보유 | 프로젝트 생성 | 비고 |
|---|---|---|---|
| Supabase | 확인 필요 | **미생성** | 기존 `hotfund/festimate`용 프로젝트 있을 수 있음 |
| Vercel | 확인 필요 | **미연결** | |
| OpenAI | 확인 필요 | - | API 키 보유 여부 확인 필요 |
| Google OAuth | 확인 필요 | **미설정** | Google Cloud Console 프로젝트 필요 |
| Kakao Developers | 확인 필요 | **미설정** | 앱 등록 필요 |
| LINE Developers | 확인 필요 | **미설정** | Channel 생성 필요 |
| 토스페이먼츠 | 확인 필요 | **미설정** | 결제 도입(M6) 시 필요 |

---

## 3. 목표 아키텍처 (RELEASE_PLAN M1 기준)

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Client    │────▶│   Vercel     │────▶│  Supabase    │
│  (Browser)  │     │  (Next.js)   │     │ (PostgreSQL) │
│  Mobile 1st │     │  App Router  │     │ + pgvector   │
└─────────────┘     │  API Routes  │     │ + PostGIS    │
                    │  i18n (3lang)│     │ + Auth       │
                    └──────┬───────┘     │ + Storage    │
                           │             └──────────────┘
                    ┌──────▼───────┐
                    │   OpenAI     │
                    │   GPT-4o    │
                    │  (AI Guide)  │
                    └──────────────┘
```

### 기술 스택

| 레이어 | 기술 | 선정 이유 |
|---|---|---|
| 프론트엔드 | Next.js 16 (App Router) + TypeScript | 기존 코드 기반, SSR/SSG, SEO |
| 스타일 | Tailwind CSS | 유틸리티 퍼스트, 빠른 프로토타이핑 |
| DB | Supabase (PostgreSQL 15+) | 무료 티어, Auth/Storage 내장, 실시간 |
| 벡터 검색 | pgvector | AI 추천, 유사 상품 검색 |
| 지리 검색 | PostGIS | 위치 기반 검색, 지도 기능 |
| AI | OpenAI GPT-4o | AI Guide, 번역, CS 자동 응대 |
| 호스팅 | Vercel | Next.js 최적, 무료 → Pro |
| 파일 저장 | Supabase Storage | S3 호환, 이미지·영상 |
| 인증 | Supabase Auth | Google/Kakao/LINE OAuth |
| CI/CD | GitHub Actions | 자동 빌드·배포·린트 |
| 모니터링 | Vercel Analytics | 무료 기본 메트릭 |

### 비용 전망 (Phase 0~1)

| 서비스 | 무료 티어 | 유료 전환 시점 | 월 예상 비용 |
|---|---|---|---|
| Supabase | 500MB DB, 1GB Storage, 50K Auth | DAU 1K+ | $25/월 (Pro) |
| Vercel | 100GB 대역폭 | 상용 도메인 연결 시 | $20/월 (Pro) |
| OpenAI | - | 즉시 (사용량 기반) | $30~100/월 (추정) |
| 도메인 | - | - | $15/년 |
| **합계** | | | **$75~145/월** |

---

## 4. 미구축 항목 & 액션

### 즉시 (M0 완료 전)

| 항목 | 액션 | 담당 |
|---|---|---|
| Supabase 프로젝트 | 신규 생성, 리전 선택 (Northeast Asia) | 관리자 |
| Vercel 프로젝트 | 레포 연결, 도메인 설정 | 관리자 |
| Google OAuth | Cloud Console 프로젝트 + OAuth 동의 화면 | 관리자 |
| Kakao Developers | 앱 등록 + Kakao Login 활성화 | 관리자 |
| LINE Developers | Channel 생성 + LINE Login 활성화 | 관리자 |
| OpenAI | API 키 발급 (또는 기존 키 확인) | 관리자 |

### M1 (인프라 구축)

| 항목 | 액션 |
|---|---|
| Next.js 프로젝트 초기화 | `npx create-next-app@latest` + TypeScript + Tailwind |
| Supabase 스키마 | SUPPLIER_UPLOAD_SPEC §8 + PLAN_TEMPLATE_SCHEMA §6 기반 마이그레이션 |
| 환경변수 | `.env.local` 설정 (레포 기록 금지, `.gitignore` 확인 완료) |
| CI/CD | GitHub Actions: lint → type-check → build → deploy |
| i18n | next-intl 또는 next-i18next 셋업 (jp/en/kr) |

---

## 5. 보안 원칙

| # | 원칙 |
|---|---|
| 1 | **API 키·시크릿은 `.env.local`에만 보관**. 레포·문서·로그에 절대 기록 금지 |
| 2 | Supabase Row Level Security (RLS) 전면 적용 |
| 3 | 여권 사본은 **암호화 저장** (Supabase Storage + 서버 사이드 암호화) |
| 4 | 관리자 경로 (`/admin/*`) 는 `ik@incutek.net` 화이트리스트 |
| 5 | CORS: `festimate.ai` 도메인만 허용 |
| 6 | Rate limiting: AI API 호출 (ACCESS_CONTROL §4 기준) |

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|---|---|---|
| 2026-04-15 | 최초 작성 | 현재 인프라 상태 스냅샷 + 목표 아키텍처 |

---

*버전: v1.0 / 작성: 2026-04-15*
