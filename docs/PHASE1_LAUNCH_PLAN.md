# Phase 1 출시 계획 (LAUNCH PLAN)

> **확정일**: 2026-04-25
> **위상**: Phase 1 출시 SSOT (Single Source of Truth)
> **선행 의사결정**: 4단계 출시 로드맵 (Phase 1: 검색+제한AI / Phase 2: K-Beauty / Phase 3: Mate 매칭 / Phase 4: 유료화 FP)
> **검증**: Claude Opus 4.7 + Genspark 2회 왕복 검토 합의

---

## 1. Phase 1 정의

### 1-1. 한 줄 정의
**"축제·명인명장·템플스테이·정원 중심 검색형 사이트 + Lite AI Guide + 저장 기능"**

### 1-2. Phase 1이 **아닌** 것
- 완성형 AI Planner ❌ (Beauty·Mate 비활성)
- K-Beauty 정식 오픈 ❌ (teaser만)
- Mate 매칭 ❌ (Coming Soon)
- 결제·FP ❌
- 다국어 완비 ❌ (일본어 우선·한국어 보조)

### 1-3. 출시 일정 추정
- **4-6주** (오늘 2026-04-25 → 약 6/5~6/15 출시 라인)
- 이후 Phase 1.5 점진 보완 (숙소·쇼핑 외부링크, 민간정원 128개 추가 등)

---

## 2. a~j 결정 사항 (Genspark 합의안)

| # | 항목 | 결정 |
|---|------|------|
| **a** | 정원 카테고리 | ✅ Phase 1 포함 (산림청 12개 시드, 민간 128개는 1.5에 점진) |
| **b** | Lite AI Guide | Festival/Explore 중심, Beauty·Mate intent 비활성, Day Card + Map만, 내부 DTO 재사용 유지 |
| **c** | 최소 법무 문서 | 이용약관·개인정보처리방침(KR/JP, APPI 반영)·AI 면책·플랫폼 디스클레이머·**저작권/출처 정책** |
| **d** | 일본어 깊이 | 일본어 우선·한국어 보조, 핵심 흐름 100% JP, AI 결과 JP 생성, 검색 카드 JP 설명+고유명사 KR/JP 병기 |
| **e** | 숙소·쇼핑 | Phase 1.5 외부링크/큐레이션 카드, 자체 페이지·intent는 Phase 2+ |
| **f** | 도메인 (`festimate.ai`) | 런치 3-5일 전 연결 + QA·SEO·쿠키·리다이렉트 점검 |
| **g** | 결제 사전 설계 | 지금부터 user_id, plan_id, ai_usage_log, estimated_cost, quota_snapshot, request_id, created_at 컬럼 + ledger 분리 |
| **h** | AI 호출 한도 | 로그인 사용자 일 5회 + 월간 cap + admin kill-switch (런치 첫 1주만 일 3회 보수적, 데이터 후 상향) |
| **i** | Mate teaser | 메뉴 노출 + Coming Soon 페이지 + 선택적 대기등록/알림받기 |
| **j** | Admin Console | 별도 UI 만들지 말고 Supabase Studio + 시드 스크립트 + 수동 CSV 운영 |

---

## 3. 작업 백로그

### 카테고리 A — 데이터 시드 (P0, 1-2주)

**기본 방향 변경 (2026-04-25)**: 매주 갱신 필요한 카테고리는 **API 자동화**로 전환. 수동 시드 → 단발성. 자동화 → 영속적 가치.

| ID | 제목 | 산출물 | 예상시간 | 의존성 | 담당 |
|----|------|-------|---------|--------|------|
| **A1** | **축제 TourAPI ETL 자동화** (7-step, 아래 sub-task) | `festivals` 테이블 + 매주 cron + publishable 80-120개 | 3-5일 | - | Opus(설계) + Sonnet(실행) |
| **A1.1** | API 매핑 표 작성 | `docs/data/festival_api_mapping.md` (응답 JSON → FestiMate 필드 매핑, 필수/선택/폐기 3분류) | 0.5일 | TourAPI 키 | Opus |
| **A1.2** | API 호출 샘플 수집 | `data/raw/festivals/sample_response_*.json` 3-5건 | 0.5일 | A1.1 | Sonnet |
| **A1.3** | 최소 정규화 스키마 확정 | `docs/data/festival_schema_v1.md` (source_id, title_ko/ja, summary_ko/ja, dates, region, lat/lng, source_url, fetched_at 등) + migration | 0.5일 | A1.2 | Opus 설계 + Sonnet 실행 |
| **A1.4** | fetch → normalize → upsert 3단 파이프라인 | `scripts/etl/festivals/fetch_raw.py`, `normalize.py`, `upsert.py` (재실행 가능, dedupe) | 1일 | A1.3 | Sonnet (구현) + Opus (normalize 규칙) |
| **A1.5** | publish 규칙 정의 + subset 선별 | `docs/data/festival_publish_rules.md` + 80-120개 publishable 태깅 | 0.5일 | A1.4 | Opus 규칙 + Sonnet 적용 |
| **A1.6** | 일본어 카드 설명 규칙 + 생성 | `docs/copy/festival_jp_copy_rules.md` (일본 20-30대 여성 톤, 80-100자) + `summary_ja` 적재 | 0.5-1일 | A1.5 | Opus (규칙·생성) + 대표님 (네이티브 검수) |
| **A1.7** | `/search` 소비용 view + 운영 runbook | `festival_card_view` + `docs/ops/festival_etl_runbook.md` (Railway cron 매주 실행) | 0.5일 | A1.4-A1.5 | Sonnet 구현 + Opus 체크리스트 |
| **A2** | 명인명장 시드 50개 | `masters` 테이블 신규 + 한국문화재재단 API or 수동 | 2-3일 | A1.3 | 대표님 (수집) + 향후 API 자동화 |
| **A3** | **템플스테이 한국불교문화사업단 API** | `templestays` 테이블 + API ETL (A1과 같은 7-step 패턴 적용) | 2-3일 | A1.3 | Opus + Sonnet |
| **A4** | 정원 시드 12개 (국가 2 + 지방 10) | `gardens` 테이블 + 12행 시드 (산림청 — API 없음, 수동) | 1일 | - | 대표님 + Sonnet(스크립트) |
| **A5** | A2/A4 일본어 설명 생성 | A1.6 규칙 재사용 (Phase 1.5에 점진 보완) | 1-2일 | A1.6 | Opus + 대표님 검수 |

### 카테고리 B — Search 페이지 (P0, 1-2주)

| ID | 제목 | 산출물 | 예상시간 | 의존성 | 담당 |
|----|------|-------|---------|--------|------|
| **B1** | `/search` 페이지 라우트 | Next.js 16 page + layout | 1일 | - | Sonnet 4.6 |
| **B2** | 카테고리 필터 UI | 축제·명인·템플스테이·정원 4-tab + 지역 필터 | 2-3일 | B1 | Sonnet 4.6 |
| **B3** | 카드 리스트 컴포넌트 | 카드 디자인 + 일본어/한국어 병기 | 2-3일 | A5 | Sonnet 4.6 |
| **B4** | 지도 통합 (Kakao Map) | 카드 ↔ 지도 핀 양방향 | 3-5일 | B3 | Sonnet 4.6 |
| **B5** | 위시리스트 (담기 → AI Guide) | localStorage + AI Guide entry param 연결 | 2일 | B3 | Sonnet 4.6 |
| **B6** | 모바일 반응형 (iOS Safari 우선) | 일본 사용자 대상 모바일 우선 검증 | 2일 | B1-B5 | Sonnet + 대표님 (실기기) |

### 카테고리 C — Lite AI Guide (`/ai/plan` 변형) (P0, 1주)

| ID | 제목 | 산출물 | 예상시간 | 의존성 | 담당 |
|----|------|-------|---------|--------|------|
| **C1** | Beauty/Mate intent 비활성 모드 | feature flag로 토글 | 0.5일 | - | Opus 설계 + Sonnet 실행 |
| **C2** | 결과 화면: Day Card + Map (Timeline·Share·Match·Regen 숨김) | `PlannerGenerateResponse` typed DTO + 렌더링 | 2-3일 | A1-A5, B4 | Opus DTO + Sonnet 렌더 |
| **C3** | AI 프롬프트 일본어 결과 생성 | OpenAI 호출 시 system prompt 일본어 강제 | 1일 | A5 | Opus |
| **C4** | 저장 기능 (Save) | `plans` 테이블 + plan_id 발급 + 사용자 매칭 | 1-2일 | D1 (Auth) | Sonnet |
| **C5** | AI 호출 한도 시스템 | quota table + admin kill-switch + 일 5회 (1주 후 상향) | 2-3일 | G1 (사전 설계) | Opus 설계 + Sonnet 실행 |

### 카테고리 D — Auth (P0, 0.5주)

| ID | 제목 | 산출물 | 예상시간 | 의존성 | 담당 |
|----|------|-------|---------|--------|------|
| **D1** | Supabase Auth 통합 | 기존 Google·Kakao OAuth 연결 (4/17 완료분) | 1일 | - | Sonnet |
| **D2** | 로그인·회원가입 UI | 일본어 우선 디자인 | 2일 | D1 | Sonnet |
| **D3** | 사용자 프로필 최소 (이름·이메일) | profile 테이블 시드 | 1일 | D1 | Sonnet |

### 카테고리 E — 홈페이지 + 일본어 (P0, 1주)

| ID | 제목 | 산출물 | 예상시간 | 의존성 | 담당 |
|----|------|-------|---------|--------|------|
| **E1** | 홈 / 메인 랜딩 | Hero + 카테고리 4 + 추천 + CTA | 3일 | A4 | Opus 카피 + Sonnet 구현 |
| **E2** | 헤더·내비·푸터 | 일본어 우선 | 1일 | E1 | Sonnet |
| **E3** | 일본어 카피 (홈·검색·Lite AI Guide·법무) | i18n 정리 | 3-5일 | A5 | Opus + 대표님 (네이티브 검수) |
| **E4** | 한국어 카피 (보조) | 일본어 카피 → 한국어 미러 | 2일 | E3 | Opus |

### 카테고리 F — 법무·정책 (P0, 1주)

| ID | 제목 | 산출물 | 예상시간 | 의존성 | 담당 |
|----|------|-------|---------|--------|------|
| **F1** | 이용약관 (KR/JP) | `/terms` 페이지 | 2일 | - | Opus 초안 + 대표님 (법무 검토) |
| **F2** | 개인정보처리방침 (KR/JP, APPI 반영) | `/privacy` 페이지 | 2-3일 | - | Opus 초안 + 대표님 |
| **F3** | AI 사용/면책 고지 | `/ai-disclaimer` 또는 통합 | 1일 | - | Opus |
| **F4** | 플랫폼 일반 디스클레이머 | 푸터 + 통합 | 0.5일 | - | Opus |
| **F5** | **저작권/출처 정책** | 5개 항목: 출처표기 / 공공데이터 vs 외부 / UGC 권리 / 무단복제 금지 / 삭제 절차 | 1-2일 | - | Opus |

### 카테고리 G — 인프라·운영 (P1, 병렬)

| ID | 제목 | 산출물 | 예상시간 | 의존성 | 담당 |
|----|------|-------|---------|--------|------|
| **G1** | 결제 사전 설계 컬럼 | ai_usage_log + ledger 테이블 + 컬럼 (user_id, plan_id, estimated_cost, quota_snapshot, request_id, created_at) | 2일 | - | Opus |
| **G2** | ✅ Supabase Storage 버킷 + CSV 업로드 | **2026-04-25 완료** — `etl-raw-csv` 버킷 + 8개 CSV (clinic_seoul.csv 등) + `upload_to_storage.py` 스크립트 (커밋 `1cbadaf`) | - | - | ✅ 완료 |
| **G3** | Railway GitHub 자동 배포 | GitHub 복구 후 연동 | 0.5일 | GitHub | 대표님 + Sonnet |
| **G4** | Naver API 토큰 재발급 | 노출된 토큰 정리 | 0.5일 | - | 대표님 |
| **G5** | Admin: Supabase Studio 운영 가이드 | 시드 입력·검수·published 승급 SOP 문서 | 1일 | - | Opus |

### 카테고리 H — Teaser 페이지 (P1, 0.5주)

| ID | 제목 | 산출물 | 예상시간 | 의존성 | 담당 |
|----|------|-------|---------|--------|------|
| **H1** | K-Beauty teaser | `/beauty` Coming Soon + 브랜드 포지션 | 1-2일 | E2 | Sonnet |
| **H2** | Mate teaser | `/mate` Coming Soon + 알림받기 폼 (이메일 수집) | 1-2일 | E2, D1 | Sonnet |

### 카테고리 I — 출시 직전 (P0, 0.5주)

| ID | 제목 | 산출물 | 예상시간 | 의존성 | 담당 |
|----|------|-------|---------|--------|------|
| **I1** | 도메인 연결 (`festimate.ai`) | Vercel + DNS | 0.5일 | 모든 P0 완료 | 대표님 |
| **I2** | QA·SEO·쿠키·리다이렉트 | 체크리스트 기반 | 2-3일 | I1 | 대표님 + Sonnet |
| **I3** | iOS Safari 실기기 검증 | 모바일 우선 | 1일 | I2 | 대표님 |

---

## 4. 시작 순서 (Genspark 8단계 + 병렬화)

```
주차 1-2:
  ├─ E1·E2 홈/내비 (디자인 시작)
  ├─ E3·E4 일본어/한국어 카피
  ├─ A1-A5 데이터 시드 (병렬)
  └─ G1 결제 사전 설계 컬럼

주차 3:
  ├─ B1-B6 /search 구현 (E·A 의존)
  ├─ F1-F5 법무 문서 (병렬)
  └─ D1-D3 Auth 통합

주차 4:
  ├─ C1-C5 Lite AI Guide 연결 (A·D 의존)
  ├─ H1·H2 Teaser 페이지
  └─ G2-G5 인프라 정리 (병렬)

주차 5:
  ├─ 통합 QA + 일본어 네이티브 검수
  └─ 미결 항목 정리

주차 6:
  ├─ I1 도메인 연결
  ├─ I2-I3 최종 QA
  └─ 🚀 출시
```

**병렬화 가능**: E + A + G1 + F (디자인/데이터/법무는 독립적)
**Critical path**: B (검색) → C (AI Guide) → I (출시)

---

## 5. 데이터 시드 상세 명세

### 5-1. 정원 (12개) — 산림청 공식

```
국가정원 2개:
  1. 순천만 국가정원 (전남 순천시 국가정원 1호길 47)
  2. 태화강 국가정원 (울산광역시 중구 태화동 일원)

지방정원 10개:
  1. 세미원 (경기 양평군 양서면 양수로 93)
  2. 죽녹원 (전남 담양군 담양읍 죽녹원로 119)
  3. 거창창포원 (경남 거창군 남상면 창포원길 21-1)
  4. 영월 동·서강정원 연당원 (강원 영월군 남면 연당리 865-8)
  5. 정읍 구절초정원 (전북 정읍시 산내면 청정로 926-89)
  6. 줄포만 노을빛정원 (전북 부안군 줄포면 우포리 516-1)
  7. 해뜰마루정원 (전북 부안군 부안읍 선온리 7-4)
  8. 경북천년숲정원 (경북 경주시 통일로 366-4)
  9. 화개정원 (인천 강화군 교동면 교동동로 471번길 6-62)
 10. 부산 낙동강 정원 (부산 사상구 삼락동 29-61)
```

→ A4 작업: 위 데이터를 `gardens` 테이블에 INSERT.

### 5-2. 축제 (~100개) — 기존 + 정비

기존 `festivals` 테이블에 ~100행 있음 (4/17 세션 로그). 정확도 검증 + 일본어 설명 보강.

### 5-3. 명인명장 (50개) — 신규 수집

추천 출처: 한국문화재재단, 무형문화재 보유자 명단.

### 5-4. 템플스테이 (30-50개) — 신규 수집

추천 출처: 한국불교문화사업단 공식.

---

## 6. 미결 사항·리스크

### 6-1. 미결
- 일본어 카피 네이티브 검수자 (대표님 직접 vs 외주?)
- iOS 실기기 테스트 환경 (시뮬레이터 vs 실기기)
- 모델 운영 (Opus 설계 + Sonnet 실행 패턴 정착)

### 6-2. 리스크
- **AI 비용 폭주** — kill-switch + 한도 + 모니터링으로 통제
- **저작권 분쟁** — F5 정책 + 출처 명시 + 사진 자체 촬영/공공데이터만
- **데이터 부족** — 명인명장 50개 수집 가능성 검증 필요
- **LINE OAuth 미지원** — Phase 1엔 Google·Kakao로 한국 우선, Phase 1.5 추가 검토
- **Naver API 의존** — homepage_discovery용 quota 관리 (현재 25,000/일)

---

## 7. Phase 1.5 / 2 / 3 / 4 차후 추가

### Phase 1.5 (Phase 1 출시 후 즉시)
- 숙소·쇼핑 외부링크 카드
- 민간정원 128개 단계적 추가
- 명인명장·템플스테이 추가 시드
- AI 한도 상향 (데이터 기반)

### Phase 2 (1.5 안정 후)
- /beauty 정식 오픈 (시술 카드·클리닉 카드)
- AI Planner Beauty Step 2.5 활성화
- 시술 마스터 30개
- 의료광고 디스클레이머 전면 적용

### Phase 3
- /ai/matching 5단계 매칭
- Mate 검수 (Admin)
- AI Guide (여행 시작 후)

### Phase 4
- FestiPoint 충전·차감
- 결제 게이트웨이 (토스 등)
- 유료 regenerate

---

## 8. 모델 운영 패턴

오늘 발견한 Opus 4.7 vs Sonnet 4.6 차이를 작업에 반영:

| 작업 성격 | 권장 모델 | 이유 |
|---------|----------|------|
| 설계 (DTO, schema, 카피, 법무 초안) | **Opus 4.7** | 깊은 분석 강점 |
| 실행 (코드 구현, 테스트, 배포) | **Sonnet 4.6** | 빠른 실행 |
| 운영 (Supabase Studio 시드 입력) | 대표님 직접 | 가장 효율 |

각 작업 항목에 **담당** 컬럼에 명시.

---

## 9. 다음 세션 시작 메시지 (제안)

```
docs/PHASE1_LAUNCH_PLAN.md 읽고 시작.

오늘부터 Phase 1 작업 시작.
어느 카테고리부터?
- A: 데이터 시드 (정원 12개부터, 가장 빨리 끝남)
- B: /search 페이지 (B1-B6)
- C: Lite AI Guide 변형
- E: 홈페이지 + 일본어 카피

작업별 모델 분리: Opus 설계, Sonnet 실행, 대표님 운영.
```

---

## 10. 변경 이력

| 날짜 | 변경 | 사유 |
|------|------|------|
| 2026-04-25 | v1.0 최초 작성 | Genspark 2회 왕복 검토 + Claude Opus 합의 |
| 2026-04-25 | v1.1 — A1 축제 시드를 TourAPI ETL 자동화 7-step으로 변경 | 매주 갱신 필요한 데이터는 API 자동화 가치 큼. Genspark 축제 API 작업 착수안 v1 반영 |

---

## 11. 축제 ETL 7-step 작업 원칙 (Genspark v1 합의)

### 핵심 원칙
- **Phase 1 핵심**: 검색형 사이트 + Lite AI Guide. 축제 데이터는 `/search`의 첫 실전 카테고리.
- **고품질 subset 우선**: 처음부터 500개 전체 완벽 정제 X. publishable 80-120개 먼저.
- **API 적재 ↔ JP 카피 분리**: Step 1-4(데이터 적재)와 Step 5(일본어 카피) 의존성 풀어줌.
- **`/search` 연결 염두**: 최소 스키마부터 잠그기.
- **출처 보존 필수**: `source_id`, `source_name`, `source_url`, `fetched_at` 항상 남김.

### 모델 분리 (A1 7-step에서)

| Step | Opus 4.7 (설계·검수) | Sonnet 4.6 (실행) |
|------|--------------------|------------------|
| A1.1 매핑 표 | ✅ 응답 JSON 분석·필드 분류 | - |
| A1.2 샘플 수집 | - | ✅ API 호출·파일 저장 |
| A1.3 스키마 | ✅ 컬럼·dedupe 기준 설계 | ✅ migration 작성 |
| A1.4 ETL 파이프라인 | ✅ normalize 규칙 | ✅ Python 구현 |
| A1.5 publish 규칙 | ✅ 기준 정의 | ✅ subset 태깅 |
| A1.6 JP 카피 | ✅ 톤·규칙·생성 | ✅ DB 적재 |
| A1.7 view + runbook | ✅ 체크리스트 | ✅ view·cron 구현 |

### Definition of Done (오늘은 X — Phase 1 진행 중 충족)
- 스크립트 1회 실행으로 raw → normalized → DB 반영 완료
- 중복 삽입 없이 재실행 가능
- publishable subset 80-120개 `is_published=true` 태깅
- `festival_card_view`로 `/search`에서 즉시 consume 가능
- 매주 Railway cron 실행 (Phase 1 출시 직전)

### 하지 말 것 (이번 단계)
- 축제 500개 전체 완벽 정제 집착
- AI Planner 연동까지 한 번에
- 다국어 전체 완성 (KR/JP만)
- 카드 디자인·지도 UX 동시 깊이 파기
- 이미지 수집 무리하게 확대

### 참고: TourAPI 키 발급
- 신청: `https://www.data.go.kr/data/15101578/openapi.do`
- 클릭: 👉 [TourAPI 활용신청](https://www.data.go.kr/data/15101578/openapi.do)
- 승인 1-2일 소요 — **지금 바로 신청** 권장
- 다국어 지원 (ko/en/ja/zh)

---

*최종 작성: 2026-04-25 / v1.1 SSOT for Phase 1 launch*
