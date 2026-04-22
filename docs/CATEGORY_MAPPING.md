# FestiMate 카테고리 마스터 매핑 테이블

> **용도**: 모든 설계 문서에서 카테고리를 참조할 때 이 문서를 단일 소스 오브 트루스(SSOT)로 사용
> **작성**: 2026-04-15
> **해결 이슈**: CONSISTENCY_AUDIT P0-2, P0-3, P1-3
>
> **원칙**
> - DB/API 코드(`enum_code`)가 정본. 다른 표기는 이 코드에서 파생
> - 새 카테고리 추가 시 이 테이블에 먼저 등록 후 각 문서 반영

---

## 1. 마스터 카테고리 테이블

| # | enum_code | UI 라벨 (네비) | 공급자 폼 코드 | 지도 마커 | 데이터 소스 | 네비 노출 | 공급자 포털 | 플랜 포함 | 파일럿 우선순위 |
|---|---|---|---|---|---|---|---|---|---|
| 1 | `festival` | KFestival | `festival` | `#EF4444` 빨강 | KTO API 주간 | O | `/host` (2순위) | O | 2순위 |
| 2 | `beauty` | KBeauty | `clinic` | `#3B82F6` 파랑 | 파트너 DB + 공급자 직접 | O (외국인만) | `/host` | O | 1순위 |
| 3 | `master` | KMaster | `master` | `#8B5CF6` 보라 | 공급자 직접 업로드 | O | `/host` | O | 1순위 |
| 4 | `templestay` | KTemple | `templestay` | `#14B8A6` 터콰이즈 | 공급자 직접 | O (KMaster 하위 or 독립) | `/host` | O | 1순위 |
| 5 | `garden` | KGarden | `garden` | `#10B981` 녹색 | 큐레이션 + 일부 공급자 | O | `/host` (선택적) | O | 2순위 |
| 6 | `food` | KFood | `chef` | `#F59E0B` 주황 | 네이버 API + 공급자 직접 | O | `/host` (3순위) | O | 3순위 |
| 7 | `sleep` | KSleep | - | `#06B6D4` 청록 | Agoda/Booking API | O | X (외부 어필리에이트) | O | Phase 1 후반 |
| 8 | `shopping` | Shopping | - | `#EC4899` 분홍 | 네이버 + 지역 관광청 | O | X (정보 제공만) | O | Phase 2 |
| 9 | `mate_session` | Mate | `mate` | `#EAB308` 황금 | 학생 UGC | O (별도 섹션) | `/mate-portal` (별도) | O | 1순위 |
| 10 | `transit` | - | - | `#6B7280` 회색 | Korail 등 외부 | X (플랜 전용) | X | O | - |
| 11 | `custom` | - | - | `#64748B` 슬레이트 | 사용자 입력 | X (플랜 전용) | X | O | - |

---

## 2. 레이어별 네이밍 규칙

| 레이어 | 규칙 | 예시 | 사용처 |
|---|---|---|---|
| **DB/API** | snake_case, `enum_code` 그대로 | `beauty`, `master`, `templestay` | DB enum, API 요청/응답, JSON 스키마 |
| **UI (네비게이션)** | K-접두사 PascalCase | `KBeauty`, `KMaster`, `KTemple` | 사이트맵, 네비 드롭다운, URL 경로명 |
| **공급자 폼** | 사업자 관점 명칭 | `clinic`, `chef`, `festival` | 공급자 포털 UI, 업로드 위자드 |
| **URL 경로** | 소문자 하이픈 | `/beauty`, `/master`, `/temple` | SITEMAP_SPEC URL 구조 |

### 2.1. 혼동 주의 매핑

아래 항목은 레이어별 이름이 달라 특히 주의 필요:

| enum_code | UI 라벨 | 공급자 폼 코드 | 혼동 포인트 |
|---|---|---|---|
| `beauty` | KBeauty | `clinic` | 여행자에겐 "뷰티", 공급자(병원)에겐 "클리닉". DB는 `beauty`로 통일 |
| `food` | KFood | `chef` | 여행자에겐 "맛집", 공급자(셰프)에겐 "쿠킹클래스 포함 셰프 상품". DB는 `food`로 통일 |
| `templestay` | KTemple | `templestay` | 네비 라벨만 축약. DB·공급자 동일 |
| `mate_session` | Mate | `mate` | 플랜 내 enum은 `mate_session` (다른 카테고리와 구분), 공급자 폼은 `mate` |

---

## 3. 공급자 포털 접근 권한 매트릭스 (P0-2 해결)

SITEMAP_SPEC.md는 `/host` 포털 대상을 3개로 좁혔으나, 실제 7개 카테고리에 공급자 폼이 필요. 아래가 정본:

| 포털 | 라우트 | 대상 카테고리 (enum_code) | Phase |
|---|---|---|---|
| **공급자 포털** (`/host`) | `/host/*` | `beauty`, `master`, `templestay`, `festival`, `garden`, `food` | Phase 0~1 (beauty/master/templestay), Phase 2 (festival/garden/food) |
| **메이트 포털** (`/mate-portal`) | `/mate-portal/*` | `mate_session` | Phase 0 |
| **포털 불필요** | - | `sleep`, `shopping`, `transit`, `custom` | 외부 API / 정보 제공 / 시스템 전용 |

### 3.1. SITEMAP_SPEC.md 수정 가이드

`/host` URL 주석을 다음으로 변경:

```
### 공급자 포털 (beauty · master · templestay · festival · garden · food)
```

§4 매트릭스의 "공급자 포털 필요" 열도 위 테이블에 맞춰 갱신.

---

## 4. 플랜 스키마 카테고리 enum 정본 (P1-3 해결)

PLAN_TEMPLATE_SCHEMA.md §3의 카테고리 enum은 이 문서의 `enum_code` 11개를 정본으로 사용:

```typescript
type PlanItemCategory =
  | 'festival'      // 축제
  | 'beauty'        // K-뷰티 시술 (= 공급자 폼의 clinic)
  | 'master'        // 명인명장
  | 'templestay'    // 템플스테이
  | 'garden'        // 정원·박물관
  | 'food'          // 식당·맛집·셰프 (= 공급자 폼의 chef)
  | 'sleep'         // 숙소
  | 'shopping'      // 쇼핑
  | 'mate_session'  // 메이트 동행
  | 'transit'       // 이동
  | 'custom';       // 사용자 추가
```

---

## 5. 카테고리 추가/변경 절차

1. 이 문서 §1 마스터 테이블에 행 추가
2. `enum_code`, UI 라벨, 공급자 폼 코드, 지도 마커 컬러 확정
3. 영향 문서 목록 작성:
   - `SITEMAP_SPEC.md` — 네비·URL 추가
   - `PLAN_TEMPLATE_SCHEMA.md` — enum·컬러 팔레트 추가
   - `SUPPLIER_UPLOAD_SPEC.md` — 공급자 폼 정의 (해당 시)
   - `MEMBERSHIP_TIERS.md` — 공급자 등급 적용 범위 (해당 시)
4. 각 문서 갱신 후 이 문서 하단 변경 이력 기록

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|---|---|---|
| 2026-04-15 | 최초 작성 | CONSISTENCY_AUDIT P0-2, P0-3, P1-3 해결 |

---

*버전: v1.0 / 작성: 2026-04-15*
