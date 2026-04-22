# FestiMate 용어 사전 (Glossary)

> **용도**: 역할·등급·상태 등 핵심 용어의 정본 정의. 모든 설계 문서와 코드에서 이 정의를 따름
> **작성**: 2026-04-15
> **해결 이슈**: CONSISTENCY_AUDIT P0-1, P2-1
>
> **원칙**
> - 동일 단어가 역할별로 다른 의미를 가지면 접두사를 붙여 구분
> - DB enum 값은 `코드` 표기, UI 표시명은 "라벨" 표기
> - 새 용어 추가 시 이 문서에 먼저 등록 후 각 문서 반영

---

## 1. 역할 (Role)

| 코드 | UI 라벨 | 정의 | 정본 문서 |
|---|---|---|---|
| `guest` | 여행자 / Traveler | 플랫폼에서 여행 상품을 탐색·예약하는 사용자 | MEMBERSHIP_TIERS §2 |
| `mate` | 메이트 / Mate | 대학생 페어로 활동하며 여행자를 동행하는 호스트 | MEMBERSHIP_TIERS §3 |
| `supplier` | 공급자 / Host | 상품(체험·시술·숙박 등)을 등록·판매하는 사업자 | MEMBERSHIP_TIERS §4 |
| `admin` | 관리자 | 플랫폼 운영·검수·CS 담당 | SITEMAP_SPEC §3 |

---

## 2. 여행자(Guest) 등급

| Lv | DB 코드 | UI 라벨 | 승급 조건 | 핵심 권한 |
|---|---|---|---|---|
| 0 | `guest_visitor` | Visitor | 비회원 | 검색·열람 |
| 1 | `guest_member` | Member | 이메일/SNS 가입 | 위시리스트, AI Guide, 문의 |
| 2 | `guest_verified` | Verified | 본인 인증 (여권/LINE/카카오) | **예약·결제·메이트 요청·리뷰** |
| 3 | `guest_plus` | Plus | 월 구독 (Phase 2) | 우선 매칭, 쿠폰 배수 |

> **Verified의 의미 (여행자)**: 신원 확인 완료. 여권·LINE 실명·카카오 실명 중 하나로 본인 입증.

---

## 3. 메이트(Mate) 등급 — 정본 (P0-1 해결)

MEMBERSHIP_TIERS.md §3을 정본으로 확정. SUPPLIER_UPLOAD_SPEC.md §6의 기존 메이트 티어(`Applied→Pending→Active→Verified→Featured`)는 **폐기**하고 아래로 통일.

| Lv | DB 코드 | UI 라벨 | 구 SUPPLIER_UPLOAD 명칭 (폐기) | 승급 조건 | 핵심 권한 |
|---|---|---|---|---|---|
| 0 | `mate_applicant` | Applicant | ~~Applied~~ | 학적 증명 제출 | 대기, 활동 불가 |
| 1 | `mate_rookie` | Rookie | ~~Pending / Active~~ | 페어 승인 + 오리엔테이션 수료 | 무료 세션 호스팅, From Mate 모집글 |
| 2 | `mate_verified` | Verified | ~~Verified~~ | 3회 세션 + 평점 4.5+ | **유료 상품 제안**, From Tourists 응답 |
| 3 | `mate_featured` | Featured | ~~Featured~~ | 20회 + 평점 4.8+ + 다국어 | 플랫폼 추천 노출, 수수료 협상 |
| 4 | `mate_pro` | Pro Guide | (없었음) | Phase 3 — 자격증 + 심화 교육 | 단독 활동, B2B 송출 |

> **Verified의 의미 (메이트)**: 실전 검증 완료. 3회 무료 세션을 성공적으로 수행하고 게스트 평점 4.5 이상 획득.

### 3.1. SUPPLIER_UPLOAD_SPEC.md §6 수정 가이드

기존 "메이트 티어 (별도)" 트리를 삭제하고, 다음으로 대체:

```
메이트 티어 → GLOSSARY.md §3 참조
(정본: MEMBERSHIP_TIERS.md §3)
```

---

## 4. 공급자(Supplier) 등급

| Lv | DB 코드 | UI 라벨 | 승급 조건 | 핵심 권한 |
|---|---|---|---|---|
| 0 | `supplier_pending` | Pending | 가입 + 사업자 인증 제출 | 검수 대기 |
| 1 | `supplier_verified` | Verified | 관리자 승인 | 상품 업로드·판매, 수수료 20% |
| 2 | `supplier_preferred` | Preferred | 평점 4.8+ / 완료 20건+ | 검색 상단, 수수료 협상 |
| -1 | `supplier_restricted` | Restricted | 평점 하락 / 분쟁 반복 | 일시 정지·숨김 |

> **Verified의 의미 (공급자)**: 관리자 심사 완료. 사업자 서류·인증 문서가 검수를 통과하여 상품 공개 판매 승인됨.

---

## 5. "Verified" 용어 정리 (P2-1 해결)

세 역할이 모두 "Verified"라는 UI 라벨을 사용하지만, **DB 코드는 접두사로 구분**하여 혼동 방지:

| 역할 | DB 코드 | 의미 | 승급 기준 |
|---|---|---|---|
| 여행자 | `guest_verified` | 신원 확인 (Identity) | 여권/LINE/카카오 본인 인증 |
| 메이트 | `mate_verified` | 실전 검증 (Performance) | 3회 세션 + 평점 4.5+ |
| 공급자 | `supplier_verified` | 심사 통과 (Approval) | 관리자 서류 검수 승인 |

### 5.1. 코드 구현 규칙

```typescript
// DB 스키마: role + tier 복합으로 저장
type UserTier = {
  role: 'guest' | 'mate' | 'supplier';
  tier: string;  // 'visitor' | 'member' | 'verified' | ...
};

// 쿼리 시 반드시 role과 함께 조건 지정
// BAD:  WHERE tier = 'verified'
// GOOD: WHERE role = 'mate' AND tier = 'verified'
```

### 5.2. UI 표시 규칙

- 동일 화면에 여러 역할의 Verified가 노출되는 경우 (예: 관리자 대시보드)
  → 뱃지에 역할 아이콘 병기: `🤝 Verified` (메이트), `🏥 Verified` (공급자), `✅ Verified` (여행자)
- 단일 역할 맥락에서는 "Verified"만으로 충분 (예: 메이트 포털 내)

---

## 6. 기타 핵심 용어

| 용어 | 정의 | 사용처 |
|---|---|---|
| **페어 (Pair)** | 메이트 2명으로 구성된 활동 단위. FF(여여) 또는 MF(혼성). MM 금지 | PLAYBOOK §5, MEMBERSHIP_TIERS §3 |
| **세션 (Session)** | 메이트 페어와 게스트가 함께하는 단위 활동. 기본 6시간 (12~18시) | PLAYBOOK §5 |
| **Rails / Recommended / Free** | 개입 수준 3단계. Rails=강제, Recommended=AI초안(수정가능), Free=자율 | PLAYBOOK §6 |
| **FestiPoint** | 플랫폼 포인트. 1P = 10원 환산. KoJap 교환 가능 | MEMBERSHIP_TIERS §7 |
| **플랜 (Plan)** | AI 또는 메이트가 생성하는 여행 일정 JSON. `travel_plans` 테이블 | PLAN_TEMPLATE_SCHEMA §2 |
| **공급자 폼 코드** | 업로드 위자드에서 사용하는 카테고리 명칭. DB enum과 다를 수 있음 → CATEGORY_MAPPING.md 참조 | SUPPLIER_UPLOAD_SPEC §1 |
| **K-접두사 라벨** | 네비게이션 UI에서 사용하는 카테고리 표시명 (KBeauty 등) → CATEGORY_MAPPING.md 참조 | SITEMAP_SPEC §1 |

---

## 7. 폐기된 용어

| 폐기 용어 | 원래 문서 | 대체 용어 | 사유 |
|---|---|---|---|
| Bronze / Silver / Gold (메이트 등급) | PLAYBOOK 용어 백로그 | Applicant → Rookie → Verified → Featured → Pro Guide | MEMBERSHIP_TIERS.md에서 확정, PLAYBOOK 미갱신 (P2-2) |
| Applied (메이트 Lv0) | SUPPLIER_UPLOAD_SPEC §6 | `mate_applicant` / Applicant | P0-1 통일 |
| Pending (메이트 Lv1) | SUPPLIER_UPLOAD_SPEC §6 | `mate_rookie` / Rookie | P0-1 통일. 공급자의 Pending과 혼동 방지 |
| Active (메이트 Lv2) | SUPPLIER_UPLOAD_SPEC §6 | `mate_rookie` / Rookie | P0-1 통일. Rookie가 활동 가능 상태를 포함 |

---

## 8. 용어 추가/변경 절차

1. 이 문서에 용어 추가 또는 수정
2. 영향 문서 식별 (PLAYBOOK, SITEMAP_SPEC, MEMBERSHIP_TIERS, PLAN_TEMPLATE_SCHEMA, SUPPLIER_UPLOAD_SPEC)
3. 각 문서 갱신
4. 폐기 용어는 §7에 기록 (삭제하지 않고 이력 보존)
5. 변경 이력 기록

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|---|---|---|
| 2026-04-15 | 최초 작성 | CONSISTENCY_AUDIT P0-1, P2-1 해결 |

---

*버전: v1.0 / 작성: 2026-04-15*
