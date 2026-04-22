# 홈페이지 설계 (HOME_PAGE_DESIGN.md)

> **버전**: v1.1
> **작성**: 2026-04-22
> **상태**: 기획 확정 — CLI 구현 대기
> **담당**: Claude Desktop (기획) → Claude Code CLI (구현)
> **연관 문서**:
> - `DESIGN.md` — 디자인 시스템 (색상·폰트·레이아웃 원칙)
> - `KBEAUTY_DB_SCHEMA.md` v1.3 — 전체 스키마 (상태 머신, clinic/beauty_plan/point 포함)
> - `ADMIN_CONSOLE_SPEC.md` v1.0 — 운영 정책 (노출 조건 근거)
> - `CLINIC_2NDLIST_FILTER_SPEC.md` v2.0 — 클리닉 점수 체계

---

## Document Information

| 항목 | 내용 |
|------|------|
| 프로젝트 | FestiMate |
| 버전 | v1.1 |
| 최종 업데이트 | 2026-04-22 |
| 주요 독자 | Product, Design, Frontend, Backend, Admin, AI Automation |
| 레이아웃 우선 | Desktop First, Responsive Ready |

---

## 0. 문서 개요

### 0.1 문서 목적

본 문서는 FestiMate의 메인 진입점인 홈페이지의 역할, 정보 구조, 노출 정책, 사용자 여정, 데이터 연동 기준, CTA 구조를 정의한다.
홈페이지는 단순한 랜딩 페이지가 아니라, 검증된 클리닉 탐색·Beauty Plan 전환·신뢰 신호 전달·외국인 환자 친화 UX를 동시에 수행하는 핵심 제품 화면이다.

이 문서의 목적은 다음 네 가지다.

1. 홈페이지를 브랜드 소개 페이지가 아니라 **전환 중심 탐색 허브**로 정의한다.
2. verified clinic, active beauty plan, language support, trust badge 등 **운영 정책이 UI에 어떻게 반영되는지** 명확히 한다.
3. 관리자 승인과 데이터 파이프라인의 결과물이 **사용자 화면에 어떤 규칙으로 노출되는지** 정의한다.
4. Next.js 기반 구현 시 **컴포넌트 구조와 데이터 읽기 우선순위**를 명확히 한다.

### 0.2 문서 범위

이 문서는 다음을 포함한다.

- 홈 화면의 목적과 KPI
- 메인 IA (Information Architecture)
- Hero, Search, Category, Verified Clinic, Beauty Plan, Point Benefit 등 주요 섹션 설계
- 상태 기반 노출 규칙 (§15)
- 추천 및 큐레이션 원칙
- CTA 정책
- 데이터 소스 및 노출 기준 (§19)
- Desktop 우선 레이아웃 전략
- SEO / 다국어 / 성능 고려

### 0.3 비범위

- 관리자 페이지 상세 설계 (ADMIN_CONSOLE_SPEC.md)
- API 엔드포인트 세부 명세
- Railway 배치 / job_queue / worker orchestration
- 세부 SQL 스키마 (KBEAUTY_DB_SCHEMA.md)
- 클리닉 상세·리스트 페이지 전체 설계 (SEARCH_PAGE_DESIGN.md 등)
- 실제 CSS / 컴포넌트 구현 코드

### 0.4 제품 목표

FestiMate 홈페이지는 다음 목표를 가진다.

- 외국인 환자가 한국 미용의료를 **신뢰할 수 있는 방식**으로 탐색하게 만든다.
- 검증된 병원과 유효한 플랜만 노출하여 **품질 중심의 첫인상**을 제공한다.
- "검색 → 비교 → 문의" 흐름을 짧게 만들어 **상담 전환율**을 높인다.
- KHIDI 등록, KAHF 인증, 외국어 지원, 공식 홈페이지 검증 여부 등 **신뢰 신호**를 명확히 제시한다.
  - KAHF: 외국인 환자 유치 서비스 품질·안전 체계 평가 인증 (유효기간 4년)
  - KHIDI: 외국인환자 유치기관 등록 현황 공공 데이터
- Beauty Plan과 Point Benefit을 통해 단순 정보 제공을 넘어 **플랫폼형 유입 구조**를 만든다.

---

## 1. 홈 화면 전략 정의

### 1.1 홈의 역할

1. **브랜드 신뢰 형성**: FestiMate가 어떤 플랫폼인지 3초 안에 이해시키는 공간
2. **탐색 시작점**: 병원명·시술명·지역·언어 조건으로 바로 들어갈 수 있는 진입구
3. **카테고리 허브**: 검색 의도가 불명확한 사용자를 피부/성형/항노화 등으로 진입시킴
4. **큐레이션 허브**: 검증된 병원과 활성화된 Beauty Plan 중심 노출
5. **전환 허브**: 회원가입·상담 요청·플랜 조회로 이어지는 CTA 구조

### 1.2 타깃 사용자 시나리오

**A. 외국인 미용의료 탐색 고객**
한국에서 피부과 또는 성형외과를 찾고 싶지만 어디서 시작해야 할지 모르는 사용자.
신뢰 배지, 언어 지원, 공식 홈페이지, 지역별 추천이 중요하다.

**B. 고의도 비교 고객**
이미 특정 시술이나 지역을 염두에 두고 있으며 빠르게 병원 리스트와 검증 상태를 확인하고 싶은 사용자.
검색바·필터·verified badge·상담 CTA를 우선 사용한다.

**C. 프로모션/기획상품 관심 고객**
단순 병원 탐색보다 패키지형 제안이나 시즌별 플랜을 보고 싶은 사용자.
Beauty Plans 섹션과 기간성 혜택, 포인트 혜택에 반응한다.

**D. 재방문 고객**
이전에 본 병원이나 관심 플랜이 있으며, 최근 업데이트된 정보나 혜택을 확인하려는 사용자.
개인화 요소와 포인트 혜택 반응이 높다.

### 1.3 핵심 KPI

| KPI | 설명 |
|-----|------|
| Search Start Rate | 홈 방문자 중 검색 시작 비율 |
| Clinic Detail CTR | 클리닉 카드 클릭률 |
| Beauty Plan CTR | 플랜 카드 클릭률 |
| Consultation CTA Click Rate | 상담 CTA 클릭률 |
| Verified Clinic Exposure CTR | 검증 클리닉 노출 → 클릭 |
| Language-Friendly Clinic CTR | 언어 지원 섹션 클릭률 |
| Point Benefit CTA Click Rate | 포인트 섹션 CTA 클릭률 |
| Homepage to Signup Conversion | 홈 → 회원가입 전환율 |
| Homepage to Inquiry Conversion | 홈 → 상담 전환율 |

---

## 2. 정보 구조 (IA)

### 2.1 글로벌 네비게이션

```
[로고 FestiMate]  [Clinics] [Beauty Plans] [메이트] [축제] [포인트]  [🌐 JP/EN/KO] [로그인]  [Find Clinics →]
```

- 언어 스위치: JP 기본 (일본 20~30대 여성 1차 타깃), EN / KO 지원
- 모바일: 햄버거 메뉴 + 하단 탭바 (홈/검색/플랜/메이트/내 정보)
- Primary CTA: MVP에서 `Find Clinics` 고정

### 2.2 홈 메인 섹션 구조

아래 순서는 "브랜드 인지 → 탐색 시작 → 신뢰 확보 → 전환 제안" 흐름을 따른다.

```
① Hero
② Quick Search
③ Category Discovery
④ Featured Verified Clinics
⑤ Trust Signals
⑥ Active Beauty Plans
⑦ Language-Friendly Clinics
⑧ Region Discovery
⑨ Point Benefits
⑩ Why FestiMate (Editorial)
⑪ Footer
```

### 2.3 섹션 우선순위 원칙

- 첫 화면에서 **검색 시작 + 핵심 가치 제안**이 동시에 보여야 한다.
- 검증되지 않은 정보보다 **검증된 정보가 먼저** 노출되어야 한다.
- 배지와 신뢰 설명은 **과장 없이 짧고 명확**해야 한다.
- Beauty Plan은 판매성보다 **큐레이션된 제안**으로 보여야 한다.
- 포인트 혜택은 보조 전환 장치로 쓰되, **메인 브랜드 톤을 해치지** 않아야 한다.

---

## 3. 사용자 여정 설계

### 3.1 첫 방문자 여정

```
홈 진입 → Hero 메시지 이해 → Quick Search 또는 Category 선택
→ Clinic List 진입 → Clinic Detail 확인 → 상담 CTA 또는 Beauty Plan 확인
```

홈의 핵심 역할: "복잡한 정보 중 어디서부터 봐야 할지 모르는 상태"를 제거하는 것.

### 3.2 재방문자 여정

```
홈 재방문 → 최근 인기/추천 클리닉 또는 Beauty Plan 재노출
→ 이전 관심 주제 재자극 → 상세 페이지 복귀 → 문의 또는 저장
```

향후: 로그인 사용자 기준으로 "최근 본 클리닉", "추천 플랜", "관심 지역" 개인화 슬롯 확장.

### 3.3 고의도 사용자 여정

```
홈 진입 → 통합 검색바 사용 → 필터 적용
→ Verified Clinic 리스트 확인 → 병원 상세 → 직접 문의
```

검색 영역은 Hero 아래에서 즉시 사용 가능해야 한다. UI 상 가장 빠른 행동 경로 제공.

---

## 4. 메인 Hero 섹션 설계

### 4.1 목적

Hero는 브랜드 소개 이미지가 아니라 **명확한 가치 제안 + 즉시 탐색 시작**을 담당한다.

사용자가 Hero를 본 순간 세 가지를 이해해야 한다.
- FestiMate는 한국 미용의료 탐색 플랫폼이다.
- 신뢰 가능한 병원과 정보를 찾을 수 있다.
- 지금 바로 검색하거나 플랜을 볼 수 있다.

### 4.2 핵심 메시지 구조

```
[실제 사진: 서울 골목/클리닉 거리 — 낮, 따뜻한 톤 / 의료광고법 준수]

韓国の美容クリニック、信頼できる場所へ。          ← Noto Serif JP / 42px / #1C1917
検証済みクリニック、外国語対応、Beauty Plan。    ← Noto Sans JP / 17px / #78716C

[Find Clinics →]      [Beauty Plan を見る]       ← Primary(#E85D3A) + Ghost CTA
```

Trust Strip (Hero 하단):
```
✅ 검증 클리닉 OO+   🗣️ 日本語対応   🏥 KHIDI 등록   ⭐ 실방문 리뷰 기반
```
숫자는 `v_public_clinics COUNT` 실시간 집계. 하드코딩 금지.

### 4.3 이미지 정책

- 실제 서울 거리/클리닉 외관 사진. AI 생성 이미지 금지.
- 모델 사진 시술 전후 비교 이미지 금지 (의료광고법).
- 허용: 상담 장면, 클리닉 로비, 서울 풍경 (DESIGN.md §Content Guidelines 준수).

### 4.4 Hero 상태별 변형

| 상태 | 변형 |
|------|------|
| 비로그인 | 기본 Hero + CTA 2개 |
| 로그인 (포인트 있음) | "XXpt 보유 중" 배지 추가 |
| 특정 언어 방문자 | "日本語対応クリニック" 보조 메시지 |
| 캠페인 유입 | campaign_tag 배너 교체 (관리자 featured 설정 시) |

---

## 5. 검색 진입 설계

### 5.1 통합 검색 바

```
┌──────────────────────────────────────────────────────────┐
│  🔍  병원명, 시술명, 지역으로 검색...        [검색]        │
└──────────────────────────────────────────────────────────┘
     [피부과] [성형외과] [항노화] [리프팅] [日本語OK] [Verified]
```

MVP: 단일 입력창. 검색 대상: 병원명 / 시술명 / 지역 / 카테고리.

### 5.2 빠른 필터 (칩형)

고빈도 필터를 검색바 아래 배치. 클릭 시 해당 조건의 Clinic List로 직행.

```
피부과 / 성형외과 / 항노화 / 리프팅 / 다국어 지원 / Verified Only
```

### 5.3 검색 추천어

초기 고정 텍스트, 향후 인기 검색 기반 동적 갱신:

```
Skin Clinic in Seoul · Plastic Surgery in Gangnam
Anti-aging · English-speaking · Botox / Filler / Laser
```

### 5.4 검색 실패 UX

결과 없음 시: 비슷한 카테고리 추천 → 인접 지역 추천 → Verified Clinic 추천 → 상담 CTA.
빈 화면으로 끝내지 않는다.

---

## 6. 카테고리 탐색 섹션

### 6.1 목적

검색 의도가 불명확한 사용자를 자연스럽게 탐색 흐름으로 이동시키는 역할.

### 6.2 카테고리 체계 (MVP)

| 카테고리 | 대표 시술 |
|---------|---------|
| 피부과 | 보톡스, 필러, 레이저토닝 |
| 성형외과 | 쌍꺼풀, 코수술, 윤곽 |
| 항노화 | HIFU, 써마지 |
| 쁘띠시술 | 쁘띠 보톡스·필러 패키지 |
| 리프팅 | 실리프팅, 써마지 |
| 스킨부스터/레이저 | 리쥬란, 물광주사 |
| 더보기 | 전체 카테고리 |

한방미용: 정책 검토 후 별도 분기.

### 6.3 카테고리 카드 구성

```
┌──────────────┐
│   [아이콘]   │
│   피부과     │  ← 17px / semibold / #1C1917
│  보톡스·필러  │  ← 14px / #78716C
│  클리닉 OO개  │  ← 13px / #A8A29E  (v_public_clinics 실시간 집계)
└──────────────┘
```

카드: `rounded-xl(16px)` / `border: 1px solid #E7E5E4` / hover: `shadow-md + translateY(-2px)`.

---

## 7. Featured Verified Clinics 섹션

> 사용자가 FestiMate를 신뢰하게 만드는 핵심 영역.

### 7.1 노출 원칙

**공개 홈에 노출 가능한 조건:**
```
clinics.is_verified = TRUE
AND raw_clinics.status = 'approved'
AND clinics.is_active = TRUE
```

`stage2a_provisional`, `pending`, `review_required` 는 절대 외부 홈 노출 금지.
홈은 내부 운영 화면이 아니다. 품질이 확인된 정보만 보여야 한다.

### 7.2 정렬 기준

1. `super_featured = TRUE` (운영자 수동 지정)
2. `approved` 우선 > `verified`
3. 카테고리·지역 문맥 적합성
4. 신뢰 배지 풍부도 (KHIDI + KAHF + 언어지원 수)
5. `jp_review_count DESC`

단순 점수만으로 일괄 정렬하지 않음. 브랜드 신뢰와 탐색 편의 병행.

### 7.3 클리닉 카드 컴포넌트

```
┌─────────────────────────────────────────────────────────────┐
│  [클리닉 대표 이미지 — WebP, lazy-load]                      │
│                                                             │
│  강남 성형외과          [✅ 검증] [🗣️ 日本語OK]              │
│  서울 강남구 · 피부과·성형외과                               │
│                                                             │
│  [KHIDI] [KAHF] [홈페이지 확인됨]                           │
│                                                             │
│  ⭐ 4.8   리뷰 127개                                        │
│  보톡스 · 필러 · 레이저토닝                                  │
│                                                             │
│  [상세 보기]                      [상담 문의 →]             │
└─────────────────────────────────────────────────────────────┘
```

**배지 우선순위 및 색상 (DESIGN.md 기준):**

| 배지 | 색상 | 조건 |
|------|------|------|
| ✅ 검증 | `#16A34A` | `is_verified = TRUE` |
| 🗣️ 日本語OK | `#E85D3A` | `consultation_jp = TRUE` |
| KHIDI | `#3B82F6` | KHIDI 매칭 + valid_until 유효 |
| KAHF | `#8B5CF6` | KAHF 매칭 + valid_until 유효 |
| 홈페이지 확인 | `#78716C` | `homepage_gate = TRUE` |

카드 한 개에 최대 3개 배지 우선 표시. 나머지는 상세 페이지.

### 7.4 빈 슬롯 처리

`is_verified = TRUE` 건수 < 6 이면: "현재 OO개 클리닉 검증 완료" 문구.
미검증 클리닉으로 채우지 않는다.

---

## 8. Trust Signals 섹션

### 8.1 목적

한국 미용의료를 처음 접하는 외국인 사용자는 "좋은 병원"보다 먼저 "믿을 수 있나"를 묻는다.
이 섹션이 그 질문에 답한다.

### 8.2 구성 요소

```
FestiMate가 클리닉을 검증하는 방법

[🏥 공공 DB 조회]      [🌐 홈페이지 확인]    [🗣️ 언어 지원 검증]   [📋 KAHF 인증]
행정안전부+KHIDI 데이터  공식 홈페이지 존재    일본어 상담 채널 확인   서비스 품질 평가
외국인 유치기관 여부 확인
```

### 8.3 표현 정책

허용 표현:
```
✅ Verified homepage
✅ Language support available
✅ KHIDI-registered institution
✅ KAHF-accredited institution
```

금지 표현:
```
❌ Best clinic / Guaranteed outcome / No.1 / Most trusted
❌ 효과 보장 / 반드시 / 확실히
```

홈은 정보 신뢰를 강화하는 장소. 의료광고가 아니다.

---

## 9. Active Beauty Plans 섹션

### 9.1 목적

단순 병원 탐색을 넘어 "실제로 고려 가능한 제안"을 보여주는 전환 섹션.

### 9.2 노출 규칙

```
노출 조건:
  beauty_plans.status = 'active'
  AND visibility_scope = 'public'
  AND start_at <= NOW()
  AND (end_at IS NULL OR end_at > NOW())
  AND 연결 clinic.is_verified = TRUE AND raw_clinics.status = 'approved'

노출 불가:
  draft / scheduled / paused / expired / archived — 절대 홈 미노출
```

### 9.3 플랜 카드 컴포넌트

```
┌───────────────────────────────────────────────────────────┐
│  [D-7]                                       [피부 케어]  │  ← 만료임박(#E85D3A) + 카테고리
│                                                           │
│  서울 피부과 스킨케어 3종 패키지                           │  ← 17px / semibold
│  강남 OO피부과                                             │  ← #78716C
│                                                           │
│  리쥬란 힐러 + 수광 주사 + 보톡스                          │
│                                                           │
│  ¥ 85,000 ~                          [포인트 적립 가능]   │  ← point_config.config_value 동적
│                                                           │
│  [플랜 보기]                                              │
└───────────────────────────────────────────────────────────┘
```

가격 불명확 시 "Contact for details" 형태 허용. 단 관리자 승인 정책과 일치해야 함.

### 9.4 정렬 기준

1. `featured_flag = TRUE`
2. 만료 임박 (end_at 오름차순)
3. 전환 가능성
4. 지역·언어 적합도
5. 최신 게시순

자동 추천 70% + 운영자 수동 큐레이션 30% 하이브리드.

---

## 10. Language-Friendly Clinics 섹션

### 10.1 목적

언어 장벽만 낮아져도 문의 확률이 크게 올라간다. 검색 전에 신뢰를 주는 장치.

### 10.2 노출 규칙

```
노출 조건:
  clinics.is_verified = TRUE AND raw_clinics.status = 'approved'
  AND (
    consultation_jp = TRUE
    OR EXISTS (
      SELECT 1 FROM clinic_language_support
      WHERE clinic_id = clinics.id AND verification_status = 'verified'
    )
  )
```

### 10.3 언어 필터 UI

```
日本語で相談できます

언어 필터: [🇯🇵 日本語] [🇺🇸 English] [🇨🇳 中文] [🇷🇺 Русский]  ← 단일 선택 칩
```

카드에 상담 가능 채널 표시: `LINE` / `카카오` / `전화` / `이메일`.
상위 2~3개 언어 카드에서 먼저 노출. 전체 상세 페이지에서 확장.

---

## 11. Region Discovery 섹션

### 11.1 목적

외국인 환자는 지역 키워드에 민감하다. "서울", "강남", "부산" 중심 진입 허브.

### 11.2 노출 지역

| 지역 | 일본어 | 노출 조건 |
|------|--------|---------|
| 서울 강남 | 江南 | verified clinic ≥ 1 |
| 서울 홍대·마포 | 弘大·麻浦 | verified clinic ≥ 1 |
| 서울 명동·중구 | 明洞 | verified clinic ≥ 1 |
| 부산 | 釜山 | verified clinic ≥ 1 |
| 대구 | 大邱 | verified clinic ≥ 1 |
| 기타 | その他 | — |

### 11.3 지역 카드 구성

```
┌────────────────────────┐
│  [지역 대표 이미지]     │
│  江南                  │
│  클리닉 OO개 · 플랜 OO개│  ← v_public_clinics, v_active_beauty_plans 집계
└────────────────────────┘
```

클릭 → `/clinics?region=gangnam` (지역 필터 적용 검색 페이지). 이미지보다 탐색 중심.

---

## 12. Point Benefits 섹션

### 12.1 목적

플랫폼 차별화 보조 장치. 사용자 행동 유도·멤버십 혜택 수준으로 표현.
메인으로 앞세우지 않되, 보조 전환 유인으로 사용.

### 12.2 노출 원칙

```
노출 조건: point_config.is_active = TRUE
노출 값: point_config.config_value (동적 — 하드코딩 금지)
노출 불가: is_active = FALSE / valid_until 만료 항목
```

```
리뷰를 쓰고 포인트를 받으세요

[📝 Phase 1 리뷰]      [📸 사진 공개]         [📋 Phase 2 효과 평가]
당일 평가 +OOpt        부위/얼굴 +OO~OOpt     14일 후 +OOpt
```

포인트 값은 point_config에서 실시간 읽기. "earn", "benefit", "member reward" 톤 유지.
현금성 오해를 유발하는 표현 금지.

---

## 13. 추천 로직과 콘텐츠 큐레이션 정책

### 13.1 홈 추천 데이터 소스

- approved / verified clinic pool (`v_public_clinics`)
- active beauty plans (`v_active_beauty_plans`)
- clinic_language_support (verified 항목)
- clinic_homepages (official verified)
- clinic_accreditations (KHIDI/KAHF valid_until 유효)
- editorial featured settings (super_featured, featured_flag)

### 13.2 자동 추천 vs 수동 큐레이션

| 슬롯 유형 | 방식 | 비율 |
|---------|------|-----|
| 자동 추천 | 점수·상태·언어·최신성 기반 정렬 | 70% |
| 수동 큐레이션 | 운영자 featured 설정 | 30% |

### 13.3 데이터 부족 시 fallback

- verified clinic 부족 → approved clinic으로 확장
- 특정 언어 섹션 부족 → 영어 지원 병원 fallback
- 특정 지역 플랜 부족 → 인접 지역 또는 전체 추천 플랜

**단, provisional clinic을 홈 fallback으로 사용하지 않는다.**

---

## 14. CTA 정책

### 14.1 주요 CTA 유형

| CTA | 텍스트 (JP) | 연결 | 조건 |
|-----|-----------|------|------|
| Find Clinics | クリニックを探す | `/clinics` | 항상 |
| 클리닉 상세 | 詳しく見る | `/clinics/{id}` | approved clinic |
| Beauty Plan | プランを見る | `/plans/{id}` | active plan |
| 상담 문의 | 相談する | `/clinics/{id}/inquiry` | 로그인 후 |
| 포인트 혜택 | ポイントを貯める | `/points` | is_active 정책 있을 때 |

### 14.2 우선순위

- 탐색 초기: `Find Clinics` + `Explore Beauty Plans`
- 비교 단계: `View Clinic Details`
- 전환 단계: `Start Consultation`

### 14.3 로그인 상태별 CTA

| 영역 | 비로그인 | 로그인 |
|------|---------|-------|
| 상담 문의 | [로그인 후 문의] → 로그인 페이지 | [상담 문의] → 문의 폼 |
| 포인트 | [포인트 받는 법] → 안내 | [XXpt 보유] → 포인트 이력 |
| 저장 | [저장하려면 로그인] | [❤️ 저장됨] |

### 14.4 CTA 금지 표현

```
❌ "지금 바로 예약" → "상담 문의"로
❌ "최저가 보장" → 가격 정책 미확정
❌ "치료" → "시술" 또는 "케어"로
❌ "보장", "확실히", "반드시"
```

---

## 15. 상태 기반 노출 규칙 (핵심)

> **이 규칙이 틀리면 미검증 클리닉이 공개 노출된다. 구현 시 최우선 검증 항목.**

### 15.1 Clinic 상태별 노출

| raw_clinics.status | is_verified | 홈 공개 노출 | 어드민 |
|---|:---:|:---:|:---:|
| seed_loaded | — | ❌ | ✅ |
| stage2a_provisional | — | ❌ | ✅ |
| review_required | — | ❌ | ✅ |
| pending | — | ❌ | ✅ |
| under_review | — | ❌ | ✅ |
| hold | — | ❌ | ✅ |
| **approved** | **TRUE** | **✅** | ✅ |
| approved | FALSE | ❌ | ✅ |
| rejected | — | ❌ | ✅ (감사) |
| suspended | — | ❌ | ✅ |
| archived | — | ❌ | ✅ |

### 15.2 Beauty Plan 상태별 노출

| status | visibility_scope | 홈 공개 노출 |
|--------|:---------------:|:----------:|
| draft | any | ❌ |
| scheduled | any | ❌ |
| **active** | **public** | **✅** |
| active | user_only | ❌ |
| active | internal | ❌ |
| paused | any | ❌ |
| expired | any | ❌ |
| archived | any | ❌ |

### 15.3 Point 정책 노출 규칙

| is_active | valid_until | 홈 노출 |
|:---------:|:----------:|:------:|
| TRUE | 유효 or NULL | ✅ config_value 동적 표시 |
| TRUE | 만료 | ❌ |
| FALSE | any | ❌ |

### 15.4 구현 권장 패턴

홈 데이터는 DB View로 처리. 컴포넌트에서 직접 테이블 JOIN 금지.

```sql
-- 공개 노출 가능한 클리닉 (Materialized View — 15분 갱신)
CREATE MATERIALIZED VIEW v_public_clinics AS
SELECT c.*
FROM clinics c
JOIN raw_clinics r ON r.approved_clinic_id = c.id
WHERE r.status = 'approved'
  AND c.is_verified = TRUE
  AND c.is_active = TRUE;

-- Active Beauty Plans (Materialized View — 15분 갱신)
CREATE MATERIALIZED VIEW v_active_beauty_plans AS
SELECT bp.*
FROM beauty_plans bp
WHERE bp.status = 'active'
  AND bp.visibility_scope = 'public'
  AND (bp.end_at IS NULL OR bp.end_at > NOW())
  AND EXISTS (
    SELECT 1 FROM clinics c
    JOIN raw_clinics r ON r.approved_clinic_id = c.id
    WHERE c.id = ANY(bp.recommended_clinic_ids)
      AND r.status = 'approved'
      AND c.is_verified = TRUE
  );
```

MV 갱신 주기: 15분 (Railway cron or Supabase pg_cron).
관리자가 `is_verified = FALSE` 변경 시 → MV 수동 REFRESH 트리거.

---

## 16. 컴포넌트 설계 기준

### 16.1 카드 시스템

카드 유형을 과도하게 늘리지 않는다.

| 카드 유형 | 컴포넌트명 |
|---------|---------|
| 클리닉 카드 | `ClinicCard` |
| Beauty Plan 카드 | `BeautyPlanCard` |
| 카테고리 카드 | `CategoryCard` |
| 지역 카드 | `RegionCard` |
| 신뢰 배지 카드 | `TrustBadgeCard` |

공통: `rounded-xl(16px)` / `shadow-sm` / hover: `shadow-md + translateY(-2px)`.

### 16.2 배지 시스템

- 카드당 최대 3개 우선 표시
- 텍스트 + 아이콘 병행
- hover/tap 시 배지 설명 tooltip
- 과도하게 자극적이지 않은 색상 (DESIGN.md §Color 준수)

### 16.3 로딩 / 빈 상태 / 에러 상태

각 섹션은 반드시 세 상태를 가진다.

```
Loading:  Skeleton UI (카드 형태 유지)
Empty:    "현재 검증 중인 클리닉이 늘어나고 있어요." + [카테고리 보기] CTA
Error:    "잠시 문제가 있어요. 새로고침해 주세요." + [재시도] 버튼
```

데이터가 없을 수 있다는 사실보다, **플랫폼이 망가졌다고 느끼는 경험**이 더 치명적이다.

---

## 17. 페이지 연결 구조

```
Home → Clinic List        (/clinics)
Home → Clinic Detail      (/clinics/{id})
Home → Beauty Plan List   (/plans)
Home → Beauty Plan Detail (/plans/{id})
Home → Login / Signup     (/auth)
Home → Consultation       (/clinics/{id}/inquiry)
Home → Points             (/points)
Home → About / Trust      (/about)
```

MVP에서는 사용자가 어디로 가야 할지 헷갈리지 않도록 링크 구조를 최대한 단순하게 유지.

---

## 18. SEO / 다국어 / 퍼포먼스

### 18.1 SEO

- 명확한 H1 구조 (한 페이지 H1 하나)
- 지역·카테고리 기반 내부 링크
- 신뢰 배지를 구조화된 텍스트로 표현 (이미지 only 금지)
- `MedicalBusiness` Structured Data (verified clinic만)
- 과도한 JS 의존 최소화

### 18.2 다국어

- 기본: 일본어 (`/ja/`, `<html lang="ja">`)
- 지원: 영어 (`/en/`), 한국어 (`/ko/`)
- `hreflang` 태그 필수
- 향후 확장: ZH, RU, TH
- 한국어 페이지: 메이트(공급측) 관점으로 별도 구성 (DESIGN.md §Language-Aware Messaging)

### 18.3 퍼포먼스

- Hero 이미지: WebP, LCP 최적화 우선
- 클리닉 카드 이미지: lazy-load (below-fold)
- Featured Clinics: SSR (검색엔진 노출 필수)
- Beauty Plans: ISR `revalidate: 900` (15분)
- `Cache-Control: public, max-age=300` (홈 API 응답)
- point_config: SSR on request (변경 즉시 반영)

---

## 19. 데이터 연동 명세

### 19.1 섹션별 데이터 소스

| 섹션 | 데이터 소스 | 갱신 주기 |
|------|-----------|---------|
| Trust Strip 숫자 | `v_public_clinics COUNT(*)` | 15분 (MV) |
| Featured Clinics | `v_public_clinics ORDER BY shortlist_score` | 15분 (MV) |
| 카테고리 클리닉 수 | `v_public_clinics GROUP BY category` | 15분 (MV) |
| Active Beauty Plans | `v_active_beauty_plans` | 15분 (MV) |
| Language-Friendly | `v_public_clinics WHERE consultation_jp = TRUE` | 15분 (MV) |
| 지역 클리닉/플랜 수 | MV 집계 | 15분 (MV) |
| 포인트 값 표시 | `point_config WHERE is_active = TRUE` | SSR (실시간) |

### 19.2 캐시 전략

| 데이터 유형 | 전략 |
|-----------|------|
| 클리닉·플랜 목록 | MV 15분 갱신 + ISR revalidate:900 |
| 포인트 정책 | SSR (config 변경 즉시 반영) |
| 지역·카테고리 집계 | MV 15분 갱신 |
| 관리자 featured 설정 | MV 갱신 시 포함 |

### 19.3 잘못된 데이터 노출 방지

- `v_public_clinics` 뷰 경유 필수 — 원본 테이블 직접 쿼리 금지
- suspended clinic: `is_active = FALSE` → MV 갱신 시 자동 제거
- 만료된 beauty_plan: `end_at < NOW()` → MV에서 처리
- 관리자 manual hide: `is_active = FALSE` 설정 즉시 MV REFRESH

---

## 20. 측정 및 분석 이벤트

| 이벤트명 | 트리거 | 포함 속성 |
|---------|-------|---------|
| `home_view` | 홈 로드 | — |
| `hero_cta_click` | Hero CTA | `cta_type` |
| `search_start` | 검색창 입력 시작 | — |
| `category_click` | 카테고리 카드 클릭 | `category` |
| `clinic_card_click` | 클리닉 카드 클릭 | `clinic_id`, `section` |
| `plan_card_click` | Beauty Plan 카드 클릭 | `plan_id` |
| `language_filter_click` | 언어 필터 선택 | `language_code` |
| `region_card_click` | 지역 카드 클릭 | `region` |
| `consultation_cta_click` | 상담 문의 CTA | `clinic_id` |
| `point_section_view` | 포인트 섹션 뷰포트 진입 | — |

이벤트는 단순 클릭 수집이 아니라 **"어떤 섹션이 실제 전환에 기여했는지"** 추적 가능하게.

---

## 21. 와이어프레임 서술

### 21.1 Above the Fold (첫 화면)

동시에 보여야 하는 것:
- 명확한 브랜드 가치 제안 (Hero 카피)
- 빠른 검색 시작 UI
- Trust Strip (신뢰 지표 요약)
- Primary / Secondary CTA

첫 화면에 clinic card나 플랜 정보를 과도하게 밀어넣지 않는다.

### 21.2 Mid Page

```
Category Discovery     → 탐색 의도 확장
Featured Clinics       → 신뢰 + 전환 핵심
Trust Signals          → 검증 기준 설명
Active Beauty Plans    → 기획 상품 전환
```

### 21.3 Lower Page

```
Language-Friendly Clinics  → 외국어 신뢰 보강
Region Discovery           → 지역 탐색 허브
Point Benefits             → 보조 전환 유도
Why FestiMate (Editorial)  → 브랜드 설득
Footer                     → 신뢰·법적 고지
```

---

## 22. MVP 구현 우선순위

### 22.1 Phase 1 필수 (코딩 착수 시)

| 섹션 | 이유 |
|------|------|
| Hero + Quick Search | 첫인상, 탐색 시작점 |
| Category Discovery | 의도 불명확 유저 진입 |
| Featured Verified Clinics | 핵심 전환 섹션 (v_public_clinics) |
| Trust Signals | 검증 기준 설명 |
| Active Beauty Plans | AI Planner 연동 전환 (v_active_beauty_plans) |
| Point Benefits (동적 값) | 리뷰 유도 |
| Footer | 법적 고지 필수 |
| §15 노출 규칙 (MV) | 미검증 클리닉 노출 방지 |

### 22.2 Phase 2

- Language-Friendly Clinics
- Region Discovery
- 개인화 (최근 본 클리닉, 추천 플랜)
- SEO Structured Data
- 국가/언어별 홈 변형

### 22.3 미결 사항

| 항목 | 내용 |
|------|------|
| 의료광고 법률 검토 | 배지 표현·툴팁 문구 의료광고법 위반 여부 (LEGAL_CHECKLIST.md 업데이트 필요) |
| 클리닉 이미지 소스 | 실제 사진 수급 정책 (클리닉 제공 vs 직접 촬영) |
| 가격 표기 방식 | 엔화(¥) vs 원화(₩) + 환율 반영 방식 |
| 관리자 로그인 방식 | Supabase Auth role vs 별도 admin_users 테이블 (ADMIN_CONSOLE_SPEC §16.3) |

---

## 23. 향후 확장 고려사항

- 국가·언어별 홈 버전 분기
- 개인화 추천 엔진 슬롯
- 캠페인 Landing 자동 생성
- 지역·시술별 SEO Landing 확장
- AI 상담 진입점 통합
- 포인트·멤버십 기반 홈 personalization

홈페이지는 단순 소개 페이지가 아니라, 장기적으로 **데이터 기반 의료 미용 탐색 플랫폼의 전면 인터페이스**가 되어야 한다.

---

## 24. 최종 의사결정 요약

1. 홈은 **verified + approved** 중심으로만 노출한다.
2. provisional 데이터는 내부 운영에서만 사용하고 **외부 홈에 절대 노출하지 않는다**.
3. Beauty Plan은 **active + visibility_scope=public + 승인 완료** 항목만 노출한다.
4. KHIDI / KAHF / 공식 홈페이지 / 외국어 지원은 **신뢰 배지로 표현하되 과장하지 않는다**.
5. 홈은 브랜드 소개보다 **검색과 전환이 우선인 구조**로 설계한다.

---

## Appendix A. 초기 컴포넌트 목록

```
HomeHero
QuickSearchBar
CategoryGrid
VerifiedClinicSection
  └ ClinicCard
TrustSignalsSection
BeautyPlanSection
  └ BeautyPlanCard
LanguageFriendlySection
RegionDiscoverySection
  └ RegionCard
PointBenefitSection
Footer
```

---

## Appendix B. 디자인 톤앤매너 메모

- 미래 지향적이되 과도하게 테크 스타트업처럼 차갑지 않을 것
- 의료 신뢰감 + 글로벌 친화성 + 프리미엄 탐색 경험을 동시에 줄 것
- 밝고 깨끗하지만 광고 배너처럼 보이지 않게 할 것
- 배지와 정보 구조는 명확하되, 화면은 가볍고 빠르게 느껴져야 할 것
- "친구가 보내준 여행 추천" 톤 — 트랜잭션이 아닌 친밀감 (DESIGN.md §Differentiator)
- Hero: Noto Serif JP (세리프로 "편지" 느낌). Body: Noto Sans JP + Pretendard.
- 로맨스 마케팅 절대 금지. 클럽·파티·노출 이미지 사용 금지 (DESIGN.md §Content Guidelines).

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-04-22 | 최초 작성. CLI 버전 — 구현 명세 중심. |
| v1.1 | 2026-04-22 | Desktop 버전 구조 채택 + 합산. meducationK → FestiMate 전체 치환. DESIGN.md 토큰 추가. MV SQL + 캐시 전략 추가. Appendix A/B 추가. 24개 섹션 완성. |

*담당: Claude Desktop (기획) → Claude Code CLI (구현)*
