# FestiMate 설계 문서 정합성 감사 (Consistency Audit)

> **감사 대상**: 5개 설계 문서 교차 비교
> **감사 일자**: 2026-04-15
> **감사 범위**: FESTIMATE_PLAYBOOK, SITEMAP_SPEC, MEMBERSHIP_TIERS, PLAN_TEMPLATE_SCHEMA, SUPPLIER_UPLOAD_SPEC
> **발견 이슈**: 9건 (P0: 3 / P1: 3 / P2: 3)

---

## 심각도 기준

| 등급 | 의미 | 기준 |
|---|---|---|
| **P0** | 즉시 해결 필요 | 문서 간 직접 모순으로 구현 시 혼란 또는 버그 유발 |
| **P1** | 다음 세션 해결 | 데이터 모델·스키마에 영향, 구현 전 통일 필요 |
| **P2** | 백로그 등록 | 혼동 가능성 있으나 구현 차단은 아님 |

---

## P0 — 즉시 해결 필요 (3건)

### P0-1. 메이트 티어 명칭 불일치

| 항목 | 내용 |
|---|---|
| **현상** | 메이트 등급 명칭이 두 문서에서 완전히 다름 |
| **문제점** | MEMBERSHIP_TIERS.md §3에서는 `Applicant → Rookie → Verified → Featured → Pro Guide` (5단)을 정의하나, SUPPLIER_UPLOAD_SPEC.md §6에서는 `Applied → Pending → Active → Verified → Featured` (5단)을 정의. 같은 등급 체계인데 이름이 다르고, 단계별 의미도 어긋남. Rookie ≠ Active, Applicant ≠ Applied 등 1:1 매핑이 불가능 |
| **권고안** | 두 문서 중 하나를 정본(canonical)으로 선택하고 나머지를 동기화. MEMBERSHIP_TIERS.md를 정본으로 삼고, SUPPLIER_UPLOAD_SPEC.md §6의 메이트 티어 섹션을 삭제하거나 MEMBERSHIP_TIERS.md 참조 링크로 대체 |
| **영향 파일** | `MEMBERSHIP_TIERS.md` §3, `SUPPLIER_UPLOAD_SPEC.md` §6 |

---

### P0-2. 공급자 포털 대상 범위 불일치

| 항목 | 내용 |
|---|---|
| **현상** | 공급자 포털이 필요한 카테고리 범위가 문서마다 다름 |
| **문제점** | SITEMAP_SPEC.md §3 URL 주석 및 §4 매트릭스에서는 공급자 포털 대상을 **KBeauty · KMaster · Templestay (3개)**로 한정. 반면 SUPPLIER_UPLOAD_SPEC.md §1에서는 **7개 카테고리 전체**(master, templestay, clinic, mate, festival, garden, chef)에 대해 업로드 폼을 정의. PLAYBOOK §4에서도 공급자 타입을 7종으로 명시. 사이트맵만 3개로 좁힘 |
| **권고안** | SITEMAP_SPEC.md의 `/host` 포털 주석을 7개 카테고리 전체로 확장하거나, 카테고리별로 포털 필요/불필요를 명시적으로 재정의. 축제·정원·셰프가 자체 업로드하는 경우 `/host` 라우트에 포함 필요 |
| **영향 파일** | `SITEMAP_SPEC.md` §3·§4, `SUPPLIER_UPLOAD_SPEC.md` §1, `FESTIMATE_PLAYBOOK.md` §4 |

---

### P0-3. 카테고리 코드 3중 네이밍

| 항목 | 내용 |
|---|---|
| **현상** | 동일 카테고리를 가리키는 코드명이 문서마다 다른 표기 사용 |
| **문제점** | 세 문서가 서로 다른 네이밍 체계 사용: (1) SITEMAP_SPEC.md: `KBeauty`, `KMaster`, `KGarden`, `KFood`, `KSleep` (K-접두사 PascalCase), (2) PLAN_TEMPLATE_SCHEMA.md §3: `beauty`, `master`, `garden`, `food`, `sleep` (snake_case), (3) SUPPLIER_UPLOAD_SPEC.md §1: `clinic`, `master`, `garden`, `chef` (snake_case, 일부 다른 단어). 특히 `KBeauty` vs `beauty` vs `clinic`이 모두 피부과/K-뷰티를 가리키는데 세 곳에서 세 이름 사용 |
| **권고안** | 단일 enum 테이블을 정의하여 모든 문서에서 참조. 제안: DB/API 레벨은 snake_case (`beauty`, `master`, `food` 등), UI 레벨은 K-접두사 (`KBeauty` 등), 공급자 레벨은 사업자 관점 명칭 (`clinic` 등). 매핑 테이블을 만들어 PLAYBOOK에 추가 |
| **영향 파일** | `SITEMAP_SPEC.md` §1·§3, `PLAN_TEMPLATE_SCHEMA.md` §3, `SUPPLIER_UPLOAD_SPEC.md` §1 |

---

## P1 — 다음 세션 해결 (3건)

### P1-1. 메이트 세션 활동 시간대 모호

| 항목 | 내용 |
|---|---|
| **현상** | 메이트 활동 가능 시간대 정의가 같은 문서 내에서도 불명확 |
| **문제점** | PLAYBOOK §5에서 "활동 시간대: **10~19시**만"이라 하면서 바로 위에서 "세션 길이: 6시간 **(12~18시)**"로 고정. PLAN_TEMPLATE_SCHEMA.md §7 AI 가이드 규칙 #8도 "12~18시"를 따름. 10시~12시, 18시~19시 구간은 활동 가능하나 세션은 불가능한 회색 구간이 되어 구현 시 혼란 |
| **권고안** | "활동 시간대 10~19시"는 플랫폼 전체의 안전 가드레일, "12~18시"는 기본 세션 시간대로 구분하여 명시. 또는 10~19시 내에서 유연한 6시간 블록을 허용할지 확정 |
| **영향 파일** | `FESTIMATE_PLAYBOOK.md` §5, `PLAN_TEMPLATE_SCHEMA.md` §7 |

---

### P1-2. 플랜 생명주기 status enum 불일치

| 항목 | 내용 |
|---|---|
| **현상** | 같은 문서 내에서 플랜 status 값이 섹션마다 다름 |
| **문제점** | PLAN_TEMPLATE_SCHEMA.md §2 JSON 스키마에서 status를 `draft | shared | booked | completed` (4개)로 정의하나, §5 생명주기 다이어그램에서는 `draft → shared → booked → in_progress → completed` (5개)로 `in_progress` 추가. §6 DB 스키마의 enum도 5개. JSON 스키마만 `in_progress`가 누락됨 |
| **권고안** | §2 JSON status에 `in_progress` 추가하여 §5·§6과 통일 |
| **영향 파일** | `PLAN_TEMPLATE_SCHEMA.md` §2·§5·§6 |

---

### P1-3. 카테고리 전체 집합 불일치 (공급자 vs 콘텐츠 vs 플랜)

| 항목 | 내용 |
|---|---|
| **현상** | 카테고리 전체 목록이 문서마다 원소가 다름 |
| **문제점** | SITEMAP_SPEC.md 네비게이션: 7개 (KFestival, KBeauty, KMaster, KGarden, KFood, KSleep, Shopping). PLAN_TEMPLATE_SCHEMA.md enum: 11개 (festival, beauty, master, garden, food, sleep, shopping, mate_session, transit, templestay, custom). SUPPLIER_UPLOAD_SPEC.md: 7개 (master, templestay, clinic, mate, festival, garden, chef). 겹치지 않는 항목 다수: `sleep`·`shopping`은 공급자 폼 없음, `chef`는 플랜 enum에 없음, `clinic`은 플랜에서 `beauty`로 불림, `templestay`는 사이트맵 네비에 없음 |
| **권고안** | 마스터 카테고리 테이블 신설. 각 카테고리별로 "네비 노출 여부 / 공급자 폼 필요 여부 / 플랜 enum 코드 / 데이터 소스"를 한 곳에서 정의 |
| **영향 파일** | `SITEMAP_SPEC.md` §1, `PLAN_TEMPLATE_SCHEMA.md` §3, `SUPPLIER_UPLOAD_SPEC.md` §1, `FESTIMATE_PLAYBOOK.md` §4 |

---

## P2 — 백로그 등록 (3건)

### P2-1. "Verified" 용어 3중 오버로딩

| 항목 | 내용 |
|---|---|
| **현상** | 세 역할(게스트·메이트·공급자)이 모두 "Verified"라는 동일 티어명을 사용하나 의미가 전혀 다름 |
| **문제점** | 게스트 Verified = 본인 인증(여권/LINE), 메이트 Verified = 3회 세션 + 평점 4.5+, 공급자 Verified = 관리자 승인. DB·API·UI에서 `tier = 'verified'`를 역할 구분 없이 쿼리하면 혼동. 코드 리뷰에서 버그 유발 가능 |
| **권고안** | 역할별 접두사 추가 검토 (예: `guest_verified`, `mate_verified`, `supplier_verified`) 또는 용어 DB 통일 세션에서 별도 명칭 부여 |
| **영향 파일** | `MEMBERSHIP_TIERS.md` §2·§3·§4 |

---

### P2-2. PLAYBOOK 용어 백로그와 실제 문서 간 괴리

| 항목 | 내용 |
|---|---|
| **현상** | PLAYBOOK의 "용어 정리 백로그"가 이미 다른 문서에서 확정된 내용과 어긋남 |
| **문제점** | PLAYBOOK §용어정리에서 "메이트 등급: Bronze / Silver / Gold (한국적 이름으로 변경 예정)"이라 기재되어 있으나, MEMBERSHIP_TIERS.md에서는 이미 Applicant/Rookie/Verified/Featured/Pro Guide로 확정. Bronze/Silver/Gold는 어디에서도 사용되지 않으나 삭제되지 않아 혼동 유발 |
| **권고안** | PLAYBOOK 용어 백로그 섹션을 MEMBERSHIP_TIERS.md 확정 내용으로 업데이트하거나, "결정됨 → MEMBERSHIP_TIERS.md 참조" 표시 |
| **영향 파일** | `FESTIMATE_PLAYBOOK.md` §용어정리, `MEMBERSHIP_TIERS.md` §3 |

---

### P2-3. 메이트 최대 인원 제약이 플랜 스키마에 미반영

| 항목 | 내용 |
|---|---|
| **현상** | PLAYBOOK에서 정의한 메이트 세션 인원 제약이 플랜 JSON 스키마에 없음 |
| **문제점** | PLAYBOOK §5에서 "메이트 2명 + 게스트 2~4명 = 총 4~6명", §7에서 "유료 상품 최대 10명 (무료는 6명)"으로 명시. 그러나 PLAN_TEMPLATE_SCHEMA.md의 JSON이나 constraints 필드에는 인원 상한 검증 필드가 없어, AI Guide가 7명 이상의 메이트 세션을 제안할 수 있음 |
| **권고안** | PLAN_TEMPLATE_SCHEMA.md의 `constraints_applied`에 `mate_session_max_participants` 필드 추가. AI Guide 프롬프트 규칙에도 인원 제한 조건 추가 |
| **영향 파일** | `PLAN_TEMPLATE_SCHEMA.md` §2·§7, `FESTIMATE_PLAYBOOK.md` §5·§7 |

---

## 요약 매트릭스

| 이슈 ID | 심각도 | 핵심 키워드 | 관련 문서 수 |
|---|---|---|---|
| P0-1 | **P0** | 메이트 티어 명칭 | 2 |
| P0-2 | **P0** | 공급자 포털 범위 | 3 |
| P0-3 | **P0** | 카테고리 3중 네이밍 | 3 |
| P1-1 | P1 | 세션 시간대 모호 | 2 |
| P1-2 | P1 | 플랜 status enum | 1 (내부) |
| P1-3 | P1 | 카테고리 집합 불일치 | 4 |
| P2-1 | P2 | Verified 오버로딩 | 1 |
| P2-2 | P2 | 용어 백로그 괴리 | 2 |
| P2-3 | P2 | 인원 제약 미반영 | 2 |

---

## 권고 우선순위

1. **즉시**: P0-3 카테고리 마스터 테이블 신설 → P0-1, P0-2, P1-3 동시 해결 가능
2. **다음 세션**: P1-1 시간대 확정, P1-2 status enum 수정
3. **용어 통일 세션**: P2-1 Verified 리네이밍, P2-2 백로그 정리
4. **스키마 설계 시**: P2-3 인원 제약 필드 추가

---

*감사 버전: v1.0 / 작성: 2026-04-15*
