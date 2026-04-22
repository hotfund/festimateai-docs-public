# 관리자 콘솔 설계 (ADMIN_CONSOLE_SPEC.md)

> **버전**: v1.0
> **작성**: 2026-04-22
> **상태**: 기획 확정 — CLI UI 구현 대기
> **담당**: Claude Desktop (기획) → Claude Code CLI (구현)
> **연관 문서**:
> - `KBEAUTY_DB_SCHEMA.md` v1.3 — 전체 스키마 (상태 머신, audit_log, point_config 포함)
> - `CLINIC_2NDLIST_FILTER_SPEC.md` v2.0 — 점수 체계, Stage 2A/2B
> - `CLINIC_ENRICHMENT_SPEC.md` v2.0 — 보강 테이블 구조
> - `PIPELINE_AUTOMATION_SPEC.md` v1.0 — job_queue, 배치 실행 구조

---

## 0. 문서 개요

### 0.1 목적

AI 자동화 파이프라인이 생산한 결과를 운영자가 검토·승인·보정하는 콘솔 설계.
클리닉 심사, 포인트 정책, beauty plan 운영, 배치 모니터링을 하나의 관리 화면에서 처리.

**핵심 원칙: 예외만 처리한다.**
자동 승격 가능한 건은 시스템이 처리. 운영자는 애매한 건, 리스크 있는 건, 정책 판단이 필요한 건만 본다.

### 0.2 문서 범위

- Desktop-first admin UI
- 역할: super_admin / operator / reviewer
- 대상 기능: 리뷰 큐, 클리닉 승인, 보강 관리, beauty plan 운영, 포인트 설정, 배치 모니터링, 감사 로그

### 0.3 비범위

- 고객용 프론트 (HOME_PAGE_DESIGN.md 별도)
- Railway 워커 구현 내부 (PIPELINE_AUTOMATION_SPEC.md)
- 세부 API 코드 명세

### 0.4 버전 히스토리

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-04-22 | 최초 작성. MVP 5개 핵심 화면 + 역할/권한/컴포넌트 정의. |

---

## 1. 운영 사용자와 역할

### 1.1 역할 정의

| 역할 | 설명 |
|------|------|
| `super_admin` | 전체 권한. 정책 수정, 삭제, 권한 관리 포함. |
| `operator` | 일반 운영. 승인·반려·보류. 포인트 수동 조정 가능. |
| `reviewer` | 클리닉 심사 전담. 승인/반려/보류 가능. 정책 수정 불가. |
| `analyst` | 읽기 전용. 대시보드·감사 로그 열람만 가능. |

### 1.2 권한 매트릭스

| 기능 | super_admin | operator | reviewer | analyst |
|------|:-----------:|:--------:|:--------:|:-------:|
| 대시보드 열람 | ✅ | ✅ | ✅ | ✅ |
| 클리닉 심사 (승인/반려/보류) | ✅ | ✅ | ✅ | ❌ |
| 클리닉 정보 수동 수정 | ✅ | ✅ | ❌ | ❌ |
| Point Config 수정 | ✅ | ✅ | ❌ | ❌ |
| 포인트 수동 조정 | ✅ | ✅ | ❌ | ❌ |
| Beauty Plan 게시/종료 | ✅ | ✅ | ❌ | ❌ |
| 배치 재실행 | ✅ | ✅ | ❌ | ❌ |
| 감사 로그 열람 | ✅ | ✅ | ✅ | ✅ |
| 역할 관리 | ✅ | ❌ | ❌ | ❌ |
| 시스템 설정 | ✅ | ❌ | ❌ | ❌ |

### 1.3 접근 제어 원칙

- 승인 권한은 최소화. reviewer는 클리닉 심사만.
- 정책 수정(point_config, plan 게시)과 운영 실행(심사)은 역할 분리.
- 민감 액션(삭제, override, 수동 포인트 조정)은 super_admin / operator로 제한.
- 모든 액션은 `admin_audit_log`에 자동 기록.

---

## 2. 정보 구조 (IA)

### 2.1 글로벌 네비게이션 (좌측 사이드바)

```
🏠 Dashboard
━━━━━━━━━━━━
📋 Review Queue      (badge: 대기 건수)
  └ 전체 큐
  └ Specialty 미확정
  └ Homepage 미검증
  └ KHIDI/KAHF 매칭 필요
  └ Final Approval 대기

🏥 Clinics
  └ 전체 클리닉 목록
  └ Verified 클리닉
  └ Provisional 클리닉
  └ 반려/정지

🔗 Enrichment
  └ 외부 소스 현황
  └ Homepage 검증
  └ 언어지원 관리
  └ 인증(KHIDI/KAHF)

💄 Beauty Plans
  └ 전체 플랜
  └ Active
  └ Draft / Scheduled

💰 Point Config
  └ 정책 목록
  └ 포인트 이력 조회

⚙️ Operations
  └ 배치 현황
  └ 실패 항목
  └ 재실행 패널

📒 Audit Log
━━━━━━━━━━━━
⚙️ Settings
```

### 2.2 공통 상단 헤더

- 전역 검색 (클리닉명, 허가번호, 플랜명)
- 알림 벨 (우선 처리 필요 항목)
- 현재 로그인 사용자 + 역할 표시

---

## 3. 대시보드 (Dashboard)

### 3.1 운영 현황 카드 (오늘 기준)

| 카드 | 데이터 소스 |
|------|------------|
| 신규 ETL 후보 | `clinic_longlist_seed` 오늘 적재 건수 |
| Review 대기 | `raw_clinics.status = 'pending'` |
| 오늘 승인 | `admin_audit_log.action = 'approve'` + 오늘 |
| 오늘 반려 | `admin_audit_log.action = 'reject'` + 오늘 |
| Active Beauty Plans | `beauty_plans.status = 'active'` |
| Enrichment 실패 | `job_queue.job_status = 'dead_letter'` |

### 3.2 품질 지표 패널

```
Homepage 검증률:  ▓▓▓▓▓▒▒▒▒▒  53%  (검증완료 / 전체 Stage2A+)
Specialty 커버리지: ▓▓▓▓▓▓▒▒▒▒  61%
KHIDI 연결률:     ▓▓▓▓▒▒▒▒▒▒  42%
언어지원 검증:    ▓▓▓▒▒▒▒▒▒▒  31%
```

### 3.3 우선 처리 패널

- High Priority Review 큐 상위 5건 (클릭 시 Review Detail 직행)
- 만료 임박 Beauty Plans (7일 이내)
- 종료 임박 Point Config (기간 종료 7일 전 경고)
- Dead Letter 잡 목록 (즉시 조치 필요)

### 3.4 병목 지표

- 평균 Review Lead Time (pending → 결정까지 시간)
- 큐 누적 속도 (일별 신규 > 처리 건수면 적색 경고)

---

## 4. Review Queue

> **목적**: AI 자동 분류 결과 중 애매한 건만 사람이 처리. 예외 처리 최소화.

### 4.1 큐 유형 정의

| 큐 이름 | 조건 | 우선순위 |
|---------|------|---------|
| **Final Approval** | `raw_clinics.status = 'pending'` | 높음 |
| **Specialty 미확정** | `clinic_filter_scores.structured_specialty_flag IS NULL` AND Stage2A | 높음 |
| **Homepage 미검증** | `homepage_gate = FALSE` AND `web_score_capped >= 40` | 중간 |
| **KHIDI 매칭 필요** | `needs_foreign_patient_join = TRUE` AND job 실패 | 중간 |
| **KAHF 매칭 필요** | KAHF enrichment job dead_letter | 중간 |
| **Under Review** | `raw_clinics.status = 'under_review'` | 현재 처리 중 |
| **Hold** | `raw_clinics.status = 'hold'` | 낮음 |

### 4.2 리스트 화면 컬럼

```
[ ] 병원명         | 지역     | 큐 유형          | 점수  | 상태       | 경과   | 담당자    | 액션
-------------------------------------------------------------------------------------------------------------
[ ] 강남성형외과   | 강남구   | Final Approval   | 82pt  | pending    | 2일    | 미배정    | [심사] [보류]
[ ] 압구정피부과   | 압구정동 | Specialty 미확정  | 71pt  | provisional| 5일    | reviewer1 | [심사] [보류]
```

### 4.3 필터 옵션

- 큐 유형 (멀티셀렉트)
- 상태 (pending / under_review / hold)
- 지역 (시/구 드릴다운)
- 점수 범위 슬라이더
- 경과 기간 (24h / 48h / 7d+)
- 담당자 (미배정 / 특정 reviewer)

### 4.4 Bulk Action

- 선택 → [일괄 보류] [일괄 할당] [CSV 내보내기]
- 일괄 승인은 super_admin 전용 (리스크 방지)

---

## 5. Clinic Review Detail

> 운영자가 개별 클리닉을 심사하는 핵심 화면.

### 5.1 화면 레이아웃

```
┌─────────────────────────────────────────────────────────────────┐
│  [← 목록으로]    강남성형외과 — 서울 강남구             [상태: pending] │
├──────────────────────┬──────────────────────────────────────────┤
│  기본 정보           │  점수 브레이크다운                        │
│  증거 패널           │  외부 소스 매칭                           │
│  지도 미리보기       │  액션 패널                                │
├──────────────────────┴──────────────────────────────────────────┤
│  Reviewer 메모                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 기본 정보 영역

| 항목 | 출처 |
|------|------|
| 기관명 (원본 / 정규화) | `clinic_longlist_seed.institution_name_raw / _norm` |
| 주소 | `address_norm`, `city`, `district` |
| 전화 | `phone_norm` |
| 기관 유형 | `institution_type_norm` |
| 개설 인허가일 | `permit_date` |
| ETL 적재일 | `first_loaded_at` |
| 행안부 허가번호 | `raw_clinics.seed_id` |

### 5.3 점수 브레이크다운 패널

```
┌── 최종 점수 ─────────────────────────────────┐
│  Pre Score:   18 / 20   ████████████████░░  │
│  Web Score:   52 / 60   ██████████████████░░│  ← Web 세부 펼치기 가능
│  Intl Score:  12 / 20   ████████████░░░░░░  │
│                                              │
│  FINAL:  82 / 100   ████████████████████░░  │
│                                              │
│  specialty_gate:  ✅ TRUE  (web_specialty ≥ 20) │
│  homepage_gate:   ✅ TRUE  (verified)           │
│  Stage:  2B VERIFIED                        │
└──────────────────────────────────────────────┘
```

Web Score 세부 펼치기:
```
  WE-01 네이버/카카오 리뷰 수    +8
  WE-02 별점 가중치              +7
  WE-03 시술 메뉴 페이지 발견    +10
  WE-04 시술 설명 페이지 발견    +12  ← specialty 카운터
  WE-05 시술 키워드 탐지         +8   ← specialty 카운터
  WE-06 블로그 포스팅 수         +7
                           합계: 52pt
```

### 5.4 증거 패널

**홈페이지 탭**
- URL (클릭 시 새 탭)
- 공식 여부 (official / candidate)
- 마지막 검증일
- 캡처 썸네일 (있을 경우)
- [재검증 요청] 버튼

**시술 증거 탭**
- 시술 페이지 URL 목록
- 발견된 시술 키워드 (한/영)
- specialty 판단 근거 텍스트 발췌

**외국어 지원 탭**
- 지원 언어 목록
- 채널 (홈페이지 / LINE / 카카오 / 전화)
- 검증 소스

**KHIDI / KAHF 탭**
- 매칭 결과 (matched / unmatched / conflict)
- 매칭 점수 + 방법
- 유효기간 (KHIDI 3년 / KAHF 4년)
- 원본 소스 레코드 링크

### 5.5 외부 소스 매칭 패널

```
KHIDI 매칭:
  ┌─────────────────────────────────────────────────────────────┐
  │  강남성형의원 (강남구 역삼동)   match_score: 0.92  [자동매칭] │
  │  [연결 확인] [다른 레코드로 변경] [매칭 해제]                │
  └─────────────────────────────────────────────────────────────┘

KAHF 매칭:
  ┌─────────────────────────────────────────────────────────────┐
  │  매칭 없음 — 검색 결과 없음                                  │
  │  [수동으로 KAHF 레코드 연결]                                 │
  └─────────────────────────────────────────────────────────────┘
```

### 5.6 액션 패널 (오른쪽 고정)

```
┌── 심사 액션 ──────────────────────────────────┐
│                                               │
│  [✅ 승인]  — clinics 테이블로 복사            │
│  [❌ 반려]  — 반려 사유 필수 입력              │
│  [⏸ 보류]  — 보류 사유 입력 (선택)            │
│  [👁 검토중] — under_review로 전환 (열람 시 자동) │
│                                               │
│  Structured Specialty 수동 설정:              │
│  ○ 피부과 (dermatology)                       │
│  ○ 성형외과 (plastic_surgery)                 │
│  ○ 미용 (medical_beauty)                      │
│  ● 미정 (NULL)                                │
│                                               │
│  Manual Override:                             │
│  [홈페이지 URL 직접 입력]                     │
│  [언어지원 수동 추가]                          │
└──────────────────────────────────────────────┘
```

**승인 확인 모달:**
```
클리닉을 승인하시겠습니까?
병원명: 강남성형외과
최종 점수: 82pt / 100pt
  specialty_gate: ✅  homepage_gate: ✅
[확인] [취소]
```

**반려 모달:**
```
반려 사유 (필수):
○ 전문과 증거 불충분
○ 홈페이지 미확인
○ 폐업/정지 의심
○ 중복 기관
○ 기타: [직접 입력]

내부 메모 (선택): ___________
[반려 처리] [취소]
```

### 5.7 Reviewer 메모 영역

- 내부 메모 입력 (타 reviewer에게 공유)
- 이전 심사 이력 표시 (같은 기관 재심사 시)
- 관련 audit_log 항목 인라인 표시

---

## 6. Clinic Master 관리

> 이미 승인된 clinics 테이블 데이터 관리.

### 6.1 클리닉 리스트 필터

- 상태 (approved / suspended / archived)
- 지역, 카테고리, 언어지원
- is_verified 여부
- 최근 수정일

### 6.2 클리닉 상세 편집

- 기본 정보 편집 (이름, 주소, 연락처)
- 노출 상태 변경 (is_verified, supplier_tier, is_shortlisted)
- SNS/링크 추가
- 의사 정보 연결

### 6.3 상태 전이 액션

```
현재 상태: approved
  → [정지] (suspected → suspend_reason 필수)
  → [보관] (더 이상 노출하지 않음)
  → [영구 삭제] (super_admin 전용, 이중 확인)
```

---

## 7. Enrichment 관리

### 7.1 외부 소스 현황 (festimate_ext.external_provider_raw)

| 탭 | 내용 |
|----|------|
| KHIDI | 총 레코드 수, 매칭률, 최근 업데이트 |
| KAHF | 총 레코드 수, 인증 만료 임박 건수 |
| Medical Korea | 총 레코드 수, 미매칭 건수 |

미매칭 레코드: 수동 연결 버튼 + 유사 클리닉 추천 표시.

### 7.2 Homepage 검증 관리

| 컬럼 | 내용 |
|------|------|
| URL | 링크 |
| Official | ✅ / ❌ |
| Reachable | 최근 체크 결과 |
| Last Checked | 날짜 |
| 액션 | [재검증] [공식 확인] [삭제] |

### 7.3 인증 만료 관리

- KHIDI: valid_until 30일 전 경고
- KAHF: valid_until 30일 전 경고
- 배지 표시: 만료 임박은 주황, 만료는 적색

---

## 8. Beauty Plan 관리

### 8.1 플랜 리스트

| 컬럼 | 내용 |
|------|------|
| 플랜명 | |
| 연결 클리닉 | |
| 상태 | draft / scheduled / active / expired / archived |
| 게시 기간 | start_at ~ end_at |
| 노출 범위 | visibility_scope |
| 승인 필요 | approval_required |
| 액션 | [편집] [게시] [종료] [보관] |

### 8.2 상태 전이 액션 버튼

```
draft:
  → [승인 요청] (approval_required=true인 경우)
  → [즉시 게시] (operator/super_admin 전용)
  → [예약 게시] (publish_at 설정)

scheduled:
  → [예약 취소] (draft로 복귀)
  → [즉시 활성화]

active:
  → [일시 중지] (paused)
  → [즉시 종료] (expired)

expired:
  → [보관] (archived)
  → [재게시] (draft로 복귀)
```

### 8.3 타이밍 설정 UI

```
게시 설정:
  publish_at:  [날짜 피커] [시간] (비워두면 즉시)
  start_at:    [날짜 피커] [시간] (비워두면 publish_at과 동일)
  end_at:      [날짜 피커] [시간] (비워두면 무기한)
  timezone:    Asia/Seoul (고정)

자동 처리:
  ☑ publish_at 도달 시 자동 활성화
  ☑ end_at 도달 시 자동 만료
```

### 8.4 검수 체크리스트

```
게시 전 검수:
□ 의료광고법 위반 표현 없음
□ 과장/보증 표현 없음 ("반드시", "확실히" 등)
□ 가격 표기 방식 준수
□ 연결 클리닉이 approved 상태
□ 포인트 정책 연동 확인
[검수 완료 — 게시] [반려]
```

---

## 9. Point Config 관리

### 9.1 정책 리스트

```
이벤트          | 적립/차감 | 현재값 | 상태   | 유효기간            | 액션
----------------------------------------------------------------------------------
earn_review_p1  | 적립      | 300pt | ✅ 활성 | 2026-04-01 ~ 무기한 | [편집] [비활성화]
earn_review_p2  | 적립      | 400pt | ✅ 활성 | 2026-04-01 ~ 무기한 | [편집] [비활성화]
earn_photo_area | 적립      | 300pt | ✅ 활성 | 2026-04-01 ~ 무기한 | [편집] [비활성화]
earn_photo_face | 적립      | 600pt | ✅ 활성 | 2026-04-01 ~ 무기한 | [편집] [비활성화]
spend_ai_planner| 차감      | 0pt   | ✅ 활성 | 2026-04-01 ~ 무기한 | [편집]
```

### 9.2 정책 편집 모달

```
이벤트 유형: earn_review_p1 (변경 불가)
설명: Phase 1 당일 평가 제출 시 적립 포인트

포인트 값:        [300] pt
포인트 유효기간:  [365] 일  (NULL = 무기한)
1회 최대 한도:    [없음] pt
유저 생애 한도:   [없음] pt

중복 적립 방지:
  방식: ● per_object (리뷰 1건당 1회)
        ○ per_window (_시간 이내 1회)
        ○ lifetime (생애 1회)

활성화:  ☑ 활성
수동 조정 허용: ☑ 허용

정책 기간:
  시작일: [2026-04-01]
  종료일: [없음 (무기한)]

[저장] [취소]
```

수정 시 `admin_audit_log`에 before/after JSON 자동 기록.

### 9.3 포인트 이력 조회

- 유저 ID / 이메일로 검색
- 이벤트 유형 필터
- 날짜 범위
- 잔액 추이 표시

### 9.4 수동 포인트 조정 (operator 이상)

```
대상 유저: [이메일 입력]
유형:  ○ 지급 (+)  ○ 차감 (-)
금액: [    ] pt
사유: [직접 입력] (필수)
내부 메모: [선택]
[조정 실행]
```

실행 시 txn_type='admin_adjust'로 point_transactions에 기록.

---

## 10. Operations 화면

> 배치 워커 상태 모니터링 + 실패 항목 처리.

### 10.1 배치 현황 테이블

```
배치 유형             | 최근 실행   | 상태    | 처리량   | 실패 건  | 소요 시간
------------------------------------------------------------------------------------
seed_load             | 02:05      | ✅ success | 1,247   | 3       | 4분 12초
pre_score             | 02:20      | ✅ success | 1,200   | 8       | 2분 55초
homepage_discovery    | 03:00      | ⚠️ partial | 423     | 47      | 18분 01초
khidi_enrichment      | 03:30      | ✅ success | 380     | 12      | 6분 44초
stage2_promote        | 04:00      | ✅ success | 23      | 0       | 0분 31초
```

### 10.2 실패 항목 모니터링

Dead Letter 목록:
```
Seed ID | 기관명         | 잡 유형              | 오류 유형     | 시도 횟수 | 마지막 오류 | 액션
-------------------------------------------------------------------------------------------------------
10234   | 홍대피부과     | homepage_discovery   | parsing_error | 5/5       | 12시간 전  | [수동 처리] [영구 제외]
20891   | 신촌성형외과   | khidi_enrichment     | matching_error| 5/5       | 3일 전     | [수동 처리] [영구 제외]
```

### 10.3 재실행 정책

| 버튼 | 동작 | 권한 |
|------|------|------|
| [단건 재실행] | job_status = 'pending'으로 리셋 | operator |
| [배치 재실행] | 특정 job_type 전체 실패 건 리셋 | super_admin |
| [영구 제외] | job_status = 'dead_letter' + exclusion_reason 기록 | operator |

**주의**: 재실행은 ON CONFLICT 패턴으로 처리되므로 데이터 손상 없음. 단, 배치 재실행은 Railway 사용량(비용) 영향.

### 10.4 시스템 헬스 위젯

```
큐 대기 잡:           127건   (정상 < 500)
Stale Records:        23건    (7일 이상 미처리, 경고)
만료 임박 인증:       8건     (30일 이내)
만료 임박 Beauty Plan: 3건    (7일 이내)
```

---

## 11. Audit Log 화면

### 11.1 로그 리스트

```
시각                | 처리자          | 액션         | 대상         | 이전 상태 → 이후 상태
--------------------------------------------------------------------------------------------
2026-04-22 14:32   | reviewer1      | approve      | raw_clinic   | pending → approved
2026-04-22 14:28   | operator1      | config_update| point_config | 300pt → 350pt
2026-04-22 14:15   | system:etl     | promote      | raw_clinic   | NULL → pending
2026-04-22 13:55   | reviewer1      | hold         | raw_clinic   | pending → hold
```

### 11.2 필터

- 처리자 (admin / system / 특정 reviewer)
- 액션 유형 (approve / reject / hold / config_update / ...)
- 대상 유형 (raw_clinic / beauty_plan / point_config / ...)
- 날짜 범위

### 11.3 상세 뷰 (행 클릭 시 사이드 패널)

```
액션: approve
대상: 강남성형외과 (raw_clinic #4521)
처리자: reviewer1 (이성준)
시각: 2026-04-22 14:32:17 KST
IP: 211.xxx.xxx.xxx

이전 상태: pending
이후 상태: approved

사유: (없음)
메모: (없음)

Before JSON:
{ "status": "pending", "etl_final_score": 82, ... }

After JSON:
{ "status": "approved", "approved_clinic_id": "uuid-...", ... }
```

### 11.4 민감 액션 하이라이트

- `manual_adjust` (포인트 수동 조정): 🟠 주황 표시
- `override` (수동 재정의): 🟠 주황 표시
- 삭제류 액션: 🔴 적색 표시

---

## 12. 상태 배지 시스템

### 12.1 Clinic 상태 배지

| 상태 | 배지 | 색상 |
|------|------|------|
| pending | `대기중` | 회색 |
| under_review | `검토중` | 파란색 |
| hold | `보류` | 주황 |
| approved | `승인` | 초록 |
| rejected | `반려` | 빨강 |
| duplicate | `중복` | 보라 |

### 12.2 Beauty Plan 상태 배지

| 상태 | 배지 | 색상 |
|------|------|------|
| draft | `초안` | 회색 |
| scheduled | `예약됨` | 파란색 |
| active | `게시중` | 초록 |
| expired | `만료` | 회색 (어두운) |
| archived | `보관` | 회색 |

### 12.3 위험 배지

- `🔴 만료 임박` (7일 이내 KHIDI/KAHF 만료)
- `⚠️ 재시도 초과` (dead_letter)
- `⚠️ Stale` (7일 이상 미처리 큐)

---

## 13. 공통 컴포넌트

### 13.1 Data Table

- 컬럼 정렬 (오름/내림)
- 행 클릭 → 사이드 패널 또는 상세 페이지
- 체크박스 멀티 선택 → Bulk Action
- 페이지네이션 (기본 50건)
- 컬럼 표시/숨김 토글

### 13.2 Action Modal

모든 상태 변경 액션은 모달로 확인:
- 대상 오브젝트 정보 요약
- 변경 내용 표시
- 사유 입력 (필수/선택 명시)
- [확인] [취소]
- Destructive 액션은 이중 확인 텍스트 입력

### 13.3 Evidence Viewer

증거 URL을 클릭 없이 인라인에서 확인:
- iFrame 프리뷰 (가능한 경우)
- URL + 캡처 이미지
- 마지막 확인 날짜

### 13.4 Score Breakdown Component

재사용 가능한 점수 시각화:
- 게이지 바 (Pre / Web / Intl / Final)
- gate 통과 여부 아이콘
- 세부 항목 Accordion

---

## 14. 관리자 액션 정책

### 14.1 승인 정책

- Stage 2B Verified (final_score ≥ 75 AND specialty_gate AND homepage_gate) 건은 원칙적으로 승인 가능.
- 예외: 운영자 판단으로 추가 검토 필요 시 hold.
- 승인 시 자동: `raw_clinics.status = 'approved'` + `clinics` 테이블 INSERT + `admin_audit_log` 기록.

### 14.2 반려 정책

- 반려 사유 코드 필수 선택.
- 반려된 seed는 `clinic_longlist_seed.canonical_status = 'excluded'` + `exclusion_reason` 기록.
- 재검토 요청 가능 (운영자가 hold로 전환 후 재심사).

### 14.3 보류 정책

- 명확하지 않은 건 hold 처리.
- hold_reason 입력 권장.
- hold 상태는 7일 경과 시 Dashboard에 경고 표시.

### 14.4 Destructive 액션 확인

| 액션 | 확인 방식 |
|------|----------|
| 클리닉 영구 삭제 | "삭제합니다" 텍스트 타이핑 확인 |
| 포인트 대량 차감 | 금액 + 사유 이중 확인 |
| 배치 전체 재실행 | super_admin 이중 확인 |

---

## 15. 화면별 데이터 연동 명세

### 15.1 Dashboard

```sql
-- 오늘 Review 대기
SELECT COUNT(*) FROM raw_clinics WHERE status = 'pending' AND DATE(captured_at) = CURRENT_DATE;

-- 품질 지표: Homepage 검증률
SELECT
  COUNT(*) FILTER (WHERE homepage_gate = TRUE) * 100.0 / COUNT(*) AS homepage_rate
FROM clinic_filter_scores
WHERE stage2_status IN ('provisional', 'verified');
```

### 15.2 Review Queue

```sql
-- Final Approval 큐
SELECT r.*, s.institution_name_norm, f.final_score, f.specialty_gate, f.homepage_gate
FROM raw_clinics r
JOIN festimate_core.clinic_filter_scores f ON f.seed_id = r.etl_seed_id
JOIN festimate_core.clinic_longlist_seed s ON s.seed_id = r.etl_seed_id
WHERE r.status = 'pending'
ORDER BY f.final_score DESC, r.captured_at ASC;
```

### 15.3 Clinic Review Detail

- `raw_clinics` + `clinic_longlist_seed` + `clinic_filter_scores` JOIN
- `clinic_homepages` WHERE seed_id 연결
- `clinic_language_support`, `clinic_accreditations`, `clinic_external_links` 각 탭
- `admin_audit_log` WHERE target_id = raw_clinic.id (이전 심사 이력)

### 15.4 Beauty Plan

- `beauty_plans` 전체 필드
- `clinics` JOIN (연결 클리닉 이름)
- `admin_audit_log` WHERE target_type = 'beauty_plan'

### 15.5 Point Config

- `point_config` 전체
- `point_transactions` WHERE txn_type = 'admin_adjust' (수동 조정 이력)

### 15.6 Operations

- `festimate_core.job_queue` WHERE job_status IN ('dead_letter', 'failed')
- `festimate_etl.etl_batches` ORDER BY started_at DESC LIMIT 20

---

## 16. MVP 구현 우선순위

### 16.1 Phase 1 (코딩 착수 시 필수)

| 화면 | 이유 |
|------|------|
| Dashboard (기본 카드) | 운영 상황 파악 |
| Review Queue (Final Approval) | 클리닉 승인 플로우 |
| Clinic Review Detail | 승인/반려 실행 핵심 |
| Point Config 리스트 + 편집 | 런칭 전 초기값 확인 |
| Beauty Plan 리스트 + 상태 전이 | AI Planner 연동 |
| Audit Log 리스트 | 운영 투명성 최소 요건 |

### 16.2 Phase 2 (런칭 후)

- Enrichment 관리 화면 (KHIDI/KAHF 수동 연결)
- Operations 배치 재실행 UI
- 포인트 이력 상세 조회
- Reviewer 할당 + SLA 관리
- 대시보드 품질 지표 심화

### 16.3 미결 사항

- 관리자 로그인 방식: Supabase Auth (별도 role 컬럼) or 별도 admin users 테이블?
  → **결정 필요**: CLI 구현 전에 확정 요망
- 배치 재실행 버튼 → Railway API 호출 방식 정의 필요
- 이미지/캡처 증거 저장 위치: Supabase Storage 사용 여부 확인 필요

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-04-22 | 최초 작성. 5개 핵심 화면 + 역할/권한/컴포넌트/데이터 연동 명세 포함. |

*담당: Claude Desktop (기획) → Claude Code CLI (구현)*
