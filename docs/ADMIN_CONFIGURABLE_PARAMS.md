# 관리자 조정 파라미터 카탈로그 (Admin Configurable Parameters)

> **작성**: 2026-04-19
> **최종 갱신**: 2026-04-19 (v1.3 — §3-3 Offering 2축 모델 + Matching 정책 추가)
> **위상**: 제품 아키텍처 SSOT — Runtime Configuration 원칙
> **선행 문서**: FESTIMATE_PLAYBOOK §7, AI_ROLE_DEFINITION §4-2, AI_COST_MODEL.md, SITEMAP_SPEC §2

---

## 1. 핵심 원칙 (The Architectural Rule)

> **"사업 규칙·가격·운영 기준은 코드에 박지 말고 DB에 두고, 관리자 페이지에서 바꾸게 한다."**

### 왜?
- 가격·정책은 시장 반응·법 개정·시즌에 따라 계속 바뀜
- 배포 없이 바꿀 수 있어야 운영 속도 ↑
- 1인+AI 운영 전제에서 엔지니어 손 타지 않고 의사결정 ↓

### 안티패턴 (하지 말 것)
```typescript
// ❌ 하드코딩
const SESSION_DURATION_MINUTES = 360;
const STARTER_PACK_PRICE = 500;
if (user.sessions_completed >= 3 && user.avg_rating >= 4.5) promote();
```

### 올바른 패턴
```typescript
// ✅ Config에서 읽기
const config = await getConfig();
const duration = config.session.duration_minutes;
const starterPrice = config.pack.starter.price;
if (user.sessions_completed >= config.tier.verified.session_threshold
 && user.avg_rating >= config.tier.verified.rating_threshold) promote();
```

---

## 2. UI/UX 구현 가이드라인 (★ 사용자 요구)

> "구현을 할 때에, 관리자페이지에서 정하는대로 잘 바뀌도록 UI, UX는 신경써라"

### 준수 사항

| 규칙 | 설명 | 예시 |
|------|------|------|
| **값 하드코딩 금지** | UI 텍스트·뱃지·안내문에 숫자 박지 말 것 | "3개 플랜 무료" → `"{freeLimit}개 플랜 무료"` |
| **범위 변동 대응** | 팩이 3개든 7개든 레이아웃 유지 | 그리드는 `repeat(auto-fit, minmax(200px, 1fr))` |
| **0·무제한 특수값** | 0원, 무제한도 자연스럽게 표시 | "무료", "제한 없음" 자동 치환 |
| **통화 포맷** | 엔·원·달러 전환 가능한 i18n 통화 표시 | `formatCurrency(price, locale)` |
| **변경 즉시 반영** | 관리자가 바꾸면 5분 내 반영 | DB 캐시 TTL 5분 or 무효화 이벤트 |
| **폴백 값** | 설정 누락 시에도 안 터지게 | `config.x ?? DEFAULT_X` |
| **변경 이력** | 누가 언제 뭘 바꿨는지 로그 | `config_change_log` 테이블 |

---

## 3. 파라미터 카탈로그 (Default Values + Ranges)

### 3-1. 💰 가격·재화 (Pricing)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `pack.starter.price` | ¥500 | ¥100 ~ ¥5,000 | 팩 카드 가격 표시 | 이 문서 |
| `pack.starter.points` | 550 pt | 100~5,000 pt | 팩 카드 포인트 | 이 문서 |
| `pack.light.price` | ¥1,500 | ¥500 ~ ¥10,000 | 동일 | 이 문서 |
| `pack.light.points` | 1,700 pt | 500~10,000 pt | 동일 | 이 문서 |
| `pack.standard.price` | ¥5,000 | ¥2,000 ~ ¥30,000 | 동일 | 이 문서 |
| `pack.standard.points` | 6,000 pt | 2,000~30,000 pt | 동일 | 이 문서 |
| `pack.premium.price` | ¥10,000 | ¥5,000 ~ ¥50,000 | 동일 | 이 문서 |
| `pack.deluxe.price` | ¥20,000 | ¥10,000 ~ ¥100,000 | 동일 | 이 문서 |
| `ai.chat.cost_per_msg` | 0 pt (무료) | 0~100 pt | 포인트 차감 안내 | 이 문서 |
| `ai.plan.cost_per_plan` | 50 pt | 0~500 pt | 동일 | 이 문서 |
| `ai.translate.image_cost` | 100 pt | 50~500 pt | 동일 | 이 문서 |
| `ai.translate.voice_cost_per_min` | 200 pt | 50~1,000 pt | 동일 | 이 문서 |
| `mate.session.base_cost` | 3,000 pt | 1,000~20,000 pt | 메이트 예약 UI | PLAYBOOK §5 |
| `mate.session.featured_cost` | 5,000 pt | 2,000~30,000 pt | 동일 | PLAYBOOK §5 |
| `points.expiry_days` | 730 (2년) | 90~무제한 | 포인트 잔액 안내 | 이 문서 |
| `points.signup_bonus` | 500 pt | 0~5,000 pt | 가입 환영 메시지 | MEMBERSHIP_TIERS |
| `points.referral_bonus` | 500 pt | 0~3,000 pt | 초대 페이지 | 이 문서 |
| `points.review_reward` | 300 pt | 0~1,000 pt | 리뷰 완료 피드백 | 이 문서 |

### 3-2. ⏰ 세션·운영 (Session & Operations)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `session.duration_minutes` | 360 (6시간) | 60~600 | 예약 UI 시간 선택 | PLAYBOOK §5 |
| `session.start_hour_min` | 10 | 0~23 | 시간 선택 드롭다운 | PLAYBOOK §5 |
| `session.start_hour_max` | 13 | 0~23 | 동일 | PLAYBOOK §5 |
| `session.end_hour_min` | 18 | 0~23 | 세션 종료 안내 | PLAYBOOK §5 |
| `session.activity_hour_min` | 10 | 0~23 | 메이트 활동 권장 시간 | PLAYBOOK §5 |
| `session.activity_hour_max` | 19 | 0~23 | 동일 | PLAYBOOK §5 |
| `session.group.free_max` | 6 | 2~20 | 세션 인원 선택 | PLAYBOOK §5 |
| `session.group.paid_max` | 10 | 2~50 | 유료 상품 인원 | PLAYBOOK §7 |
| `session.mate_count` | 2 | 1~5 | 메이트 페어 표시 | PLAYBOOK §5 |

### 3-3. 🎯 매칭·승급 (Matching & Tier)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `tier.verified.session_threshold` | 3 | 1~20 | Mate 프로필 진행도 | MEMBERSHIP_TIERS |
| `tier.verified.rating_threshold` | 4.5 | 3.0~5.0 | 동일 | MEMBERSHIP_TIERS |
| `tier.featured.session_threshold` | 10 | 3~50 | 동일 | MEMBERSHIP_TIERS |
| `tier.featured.rating_threshold` | 4.7 | 4.0~5.0 | 동일 | MEMBERSHIP_TIERS |
| `tier.pro_guide.session_threshold` | 50 | 10~200 | 동일 | MEMBERSHIP_TIERS |
| `tier.pro_guide.rating_threshold` | 4.8 | 4.0~5.0 | 동일 | MEMBERSHIP_TIERS |
| `tier.pro_guide.interview_required` | true | boolean | 승급 절차 안내 | MEMBERSHIP_TIERS |
| `matching.pair_format_allowed` | ["FF","MF"] | 부분집합 of [FF,MF,GG,BB,solo] | Phase별 허용 표시 | PLAYBOOK §5 |
| `matching.solo_allowed` | false | boolean | 솔로 신청 가능 여부 | PLAYBOOK §5 |
| `matching.female_required_count` | 1 | 0~2 | 페어 구성 요구사항 | PLAYBOOK §5 |

#### Offering 2축 모델 ⭐ v1.3 신규 (AI_TERMINOLOGY §6)

Mate가 생성·보유할 수 있는 Offering(제안서) 제한. Tier별 TTL + 동시 활성 수.

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `offerings.rookie.ttl_days` | 30 | 7~180 | 만료 알림 | Session 3 |
| `offerings.rookie.concurrent_max` | 1 | 1~5 | 추가 생성 차단 안내 | Session 3 |
| `offerings.rookie.extendable` | false | boolean | 연장 버튼 노출 | Session 3 |
| `offerings.verified.ttl_days` | 60 | 30~365 | 동일 | Session 3 |
| `offerings.verified.concurrent_max` | 3 | 1~10 | 동일 | Session 3 |
| `offerings.verified.extendable` | false | boolean | 동일 | Session 3 |
| `offerings.featured.ttl_days` | 90 | 30~365 | 동일 | Session 3 |
| `offerings.featured.concurrent_max` | 5 | 1~20 | 동일 | Session 3 |
| `offerings.featured.extendable` | false | boolean | 동일 | Session 3 |
| `offerings.pro_guide.ttl_days` | 180 | 60~365 | 동일 | Session 3 |
| `offerings.pro_guide.concurrent_max` | 10 | 3~50 | 동일 | Session 3 |
| `offerings.pro_guide.extendable` | true | boolean | **Pro Guide만 연장 옵션** | Session 3 |
| `offerings.applicant.allowed` | false | boolean | Applicant Offering 차단 | Session 3 |

**원칙**: 월별 생성 throttle 없음. 동시 활성 상한이 역할 대체. 기간 만료 시 자동 비활성화.

#### Matching 정책 ⭐ v1.3 신규 (AI_TERMINOLOGY §4)

5단계 매칭 플로우의 정책 파라미터:

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `matching.approval_mode` | `"mutual"` | "mutual"/"one_way"/"hybrid" | ❤️ 성사 조건 | Session 3 |
| `matching.initiating_mode` | `"bidirectional"` | "bidirectional"/"tourist_only"/"mate_only" | 누가 먼저 제안? | Session 3 |
| `matching.tier2_unlock_mode` | `"mutual_interest"` | "mutual_interest"/"point_spend"/"hybrid" | Tier 2 해제 방식 | Session 3 |
| `matching.tier3_unlock_mode` | `"booking_confirmed"` | "booking_confirmed"/"point_spend" | Tier 3 해제 방식 | Session 3 |
| `matching.tier3_requires_point_spend` | true | boolean | 예약 시 포인트 차감 | Session 3 |
| `matching.review_deadline_hours` | 48 | 24~168 | 리뷰 작성 마감 | Session 3 |
| `matching.review_mandatory` | true | boolean | 리뷰 강제 여부 | Session 3 |
| `matching.checkin_gps_required` | true | boolean | 세션 체크인 GPS 확인 | Session 3 |
| `matching.no_show_grace_minutes` | 30 | 15~60 | 지각 허용 시간 | Session 3 |
| `matching.no_show_strikes_until_suspension` | 3 | 1~5 | Mate 자격 정지 기준 | Session 3 |
| `matching.interest_notification_channels` | ["push","email"] | 부분집합 of [push,email,sms] | ❤️ 알림 채널 | Session 3 |
| `matching.messaging_activated_on_tier` | 2 | 1~3 | 메시징 활성화 Tier | Session 3 |

#### Tier 정보 공개 필드 리스트 ⭐ v1.3 신규 (AI_TERMINOLOGY §5)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `matching.tier1_fields` | `["gender","age_range","region","dates","group_size","interests","budget_range","language","one_line_intro"]` | 배열 | 매칭 페이지 카드 표시 필드 | Session 3 |
| `matching.tier2_fields` | `["bio_detail","sns_handle","rating","session_count","time_preference","pair_format","visit_detail"]` | 배열 | 상호 ❤️ 후 공개 | Session 3 |
| `matching.tier3_fields` | `["photo","exact_age","real_name","contact","affiliation","identity_verified"]` | 배열 | 예약 확정 후 공개 | Session 3 |

### 3-4. 🤖 AI 모델·Fallback·캐시·Batch ⭐ v1.2 확장

> 설계 근거·시뮬레이션·의사결정 프레임은 `docs/AI_COST_MODEL.md` 참조.
> 여기서는 **config 키만** 명세.

#### 모델 기본값 + Fallback

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `ai.model.plan` | `gpt-4o-mini` | OpenAI 모델 목록 | 없음 | AI_COST_MODEL §5 |
| `ai.model.plan_fallback` | `gpt-4o` | OpenAI 모델 목록 | 없음 | AI_COST_MODEL §5 |
| `ai.model.plan_fallback_trigger` | `validator_or_user_signal` | "never" / "validator_only" / "user_signal_only" / "validator_or_user_signal" | 없음 | AI_COST_MODEL §5 |
| `ai.model.sampling_rate` | 0.05 (5%) | 0.0~0.50 | 없음 | AI_COST_MODEL §5 |
| `ai.model.user_upgrade_cost_points` | 500 pt | 0~5,000 pt | 플랜 하단 버튼 라벨 | AI_COST_MODEL §5 |
| `ai.model.chat` | `gpt-4o-mini` | 동일 | 없음 | AI_ROLE §7 |
| `ai.model.translate` | `gpt-4o-mini` | OpenAI + DeepL + Gemini | 없음 | AI_COST_MODEL §5 |
| `ai.model.vision` | `gpt-4o-mini` | OpenAI vision 모델 | 없음 | 동일 |
| `ai.temperature.default` | 0.7 | 0.0~2.0 | 없음 | 이 문서 |
| `ai.max_tokens.default` | 2000 | 500~8000 | 없음 | 이 문서 |

**Fallback trigger enum 설명**:
- `never`: 무조건 mini 사용, fallback 없음
- `validator_only`: JSON invalid·필수 필드 누락 등 결정적 실패 시 4o 재시도
- `user_signal_only`: 사용자가 "더 정교하게" 버튼 클릭 시에만
- `validator_or_user_signal`: 위 둘 중 하나만 발생해도 4o 호출 (권장)

#### 캐시

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `ai.cache.enabled` | true | boolean | 없음 | AI_COST_MODEL §6 |
| `ai.cache.ttl_hours` | 24 | 1~720 | 없음 | AI_COST_MODEL §6 |
| `ai.cache.max_size_mb` | 1000 | 100~10,000 | 없음 | AI_COST_MODEL §6 |
| `ai.cache.rag_ttl_hours` | 1 | 1~24 | 없음 | AI_COST_MODEL §6 |
| `ai.cache.invalidate_on_venue_update` | true | boolean | 없음 | AI_COST_MODEL §6 |

#### Batch API

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `ai.batch_api.enabled` | false | boolean | 없음 | AI_COST_MODEL §7 |
| `ai.batch_api.use_cases` | `[]` | 문자열 배열 (`"translation"`, `"quality_sampling"`, `"analytics"`, `"supplier_review"`) | 없음 | AI_COST_MODEL §7 |
| `ai.batch_api.enable_after_milestone` | "M5" | "M3"/"M4"/"M5"/"M6"/"never" | 없음 | AI_COST_MODEL §7 |

#### 사용량 제한

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `ai.limit.free.daily_messages` | 20 | 5~무제한 | 무료 유저 한도 안내 | AI_ROLE_DEFINITION |
| `ai.limit.free.monthly_plans` | 3 | 1~무제한 | 동일 | AI_ROLE_DEFINITION |
| `ai.limit.verified.daily_messages` | 100 | 20~무제한 | Verified 혜택 | MEMBERSHIP_TIERS |
| `ai.limit.verified.monthly_plans` | 20 | 3~무제한 | 동일 | MEMBERSHIP_TIERS |
| `ai.limit.subscription.monthly_plans` | -1 (무제한) | 10~무제한 | 구독자 혜택 | 이 문서 |

**정책**: 품질·비용 로그 기반 판단. 업그레이드 결정 체크리스트는 `AI_COST_MODEL §10` 참조.

### 3-5. 🔒 접근 제어·다국어 (Access & Locale)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `locale.default` | `"jp"` | "jp"/"en"/"kr" | 첫 접속 언어 | Session 2 |
| `locale.available` | `["jp","en","kr"]` | 부분집합 | 언어 스위처 노출 | Session 2 |
| `locale.fallback_chain` | `["jp","en"]` | 순서 있는 배열 | 번역 누락 시 | Session 2 |
| `access.kbeauty.block_kr_ip` | true | boolean | IP 차단 활성 | SITEMAP_SPEC §6 |
| `access.kbeauty.hide_from_kr_lang` | true | boolean | 한국어 UI 숨김 | SITEMAP_SPEC §6 |
| `access.kbeauty.require_jp_login` | false | boolean | 로그인 검증 | SITEMAP_SPEC §6 |
| `access.kojap.phase_2_enabled` | false | boolean | KoJap 섹션 노출 | SITEMAP_SPEC §1 |
| `access.tw.phase_2_enabled` | false | boolean | 대만 섹션 노출 | Session 2 |
| `access.medical_disclaimer_required` | true | boolean | 의료 정보 고지 | LEGAL_CHECKLIST |

### 3-6. 🚨 긴급·안전 (Emergency & Safety)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `emergency.contact_num_kr` | "112" | 문자열 | 긴급 버튼 | PLAYBOOK §10 |
| `emergency.contact_med_kr` | "119" | 문자열 | 동일 | PLAYBOOK §10 |
| `emergency.cs_email` | "help@festimate.ai" | email | 문의 페이지 | 이 문서 |
| `emergency.cs_response_hours` | 24 | 1~72 | SLA 안내 | 이 문서 |
| `safety.checkin_required` | true | boolean | 세션 체크인 프로세스 | PLAYBOOK §5 |
| `safety.mandatory_review` | true | boolean | 리뷰 강제 | PLAYBOOK §5 |

### 3-7. 🎨 콘텐츠·프로모션 (Content & Promo)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `hero.headline_jp` | "韓国に、親友ができる。" | 문자열 | 홈 히어로 | PLAYBOOK §1 |
| `hero.headline_en` | "Find Your Korean Bestie." | 문자열 | 동일 | PLAYBOOK §1 |
| `hero.headline_kr` | "일본 친구와 함께, 한국을 알려주세요." | 문자열 | 동일 | DESIGN.md |
| `promo.active_banner` | null | 문자열 or null | 상단 배너 | 이 문서 |
| `promo.seasonal_bonus_rate` | 1.0 | 0.5~3.0 | 포인트 충전 시 보너스 | 이 문서 |

### 3-8. 🚩 기능 플래그 (Feature Flags)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `feature.ai_voice_mode` | false | boolean | 음성 버튼 표시 | AI_ROLE_DEFINITION §9 |
| `feature.ai_camera_mode` | false | boolean | 카메라 버튼 표시 | AI_ROLE_DEFINITION §9 |
| `feature.mate_portal_beta` | false | boolean | Mate 포털 접근 | RELEASE_PLAN M5 |
| `feature.kojap_section` | false | boolean | KoJap 메뉴 | RELEASE_PLAN |
| `feature.payment_enabled` | false | boolean | 결제 UI 노출 | RELEASE_PLAN M6 |
| `feature.ai_subscription` | false | boolean | 월구독 옵션 | 이 문서 |

### 3-9. 💄 K-뷰티 전용 (K-Beauty Specialized)

K-뷰티는 **메인 수익원**으로 격상됨 (Session 2). 법무·운영·UX 파라미터 분리 관리.

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `kbeauty.consultation_required` | true | boolean | 상담 플로우 강제 | Session 2 |
| `kbeauty.referral_fee_jpy` | 0 (TBD) | ¥0~¥100,000 | 내부만, UI 미노출 | Session 2 |
| `kbeauty.treatment_categories` | ["피부과","성형외과","미용클리닉","줄기세포","치과","한의원"] | 배열 | 카테고리 필터 | Session 2 |
| `kbeauty.hospital_verification_required` | true | boolean | 공급자 온보딩 | Session 2 |
| `kbeauty.medical_disclaimer_version` | "v1.0" | 문자열 | 의료 정보 고지 | Session 2 |
| `kbeauty.min_consultation_lead_hours` | 48 | 24~168 | 상담 신청 최소 리드타임 | Session 2 |
| `kbeauty.language_support_required` | true | boolean | 클리닉 일본어 응대 여부 검증 | Session 2 |

### 3-10. 💰 수수료·정산 (Commission & Payout)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `supplier.commission.default_rate` | 0.00 (0%) | 0.00~0.50 | 공급자 대시보드 | PLAYBOOK §7 |
| `supplier.commission.mate_session_rate` | 0.00 | 0.00~0.50 | Mate 정산 | PLAYBOOK §7 |
| `supplier.referral_fee.kbeauty` | 0 (TBD) | ¥0~¥100,000 | 내부 | Session 2 |
| `supplier.payout.cycle_days` | 30 | 7~90 | 정산 일정 안내 | Session 2 |
| `supplier.payout.min_threshold_jpy` | 5000 | ¥1,000~¥50,000 | 최소 정산 금액 | Session 2 |

### 3-11. 🔄 환불·취소 정책 (Refund & Cancellation)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `refund.policy.session_cancel_hours` | 24 | 0~168 | 취소 UI | Session 2 |
| `refund.policy.points_expiry_grace_days` | 14 | 0~90 | 만료 알림 | Session 2 |
| `refund.policy.full_refund_hours` | 72 | 24~240 | 환불 정책 표시 | Session 2 |
| `refund.policy.half_refund_hours` | 24 | 0~72 | 동일 | Session 2 |

### 3-12. 💳 결제 수단·통화 (Payment Methods)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `payment.methods.enabled` | ["jcb","visa","master","paypay","line_pay"] | 부분집합 | 결제 선택 UI | Session 2 |
| `payment.methods.kr_enabled` | ["kakao_pay","toss","naver_pay"] | 부분집합 | 한국 결제 UI | Session 2 |
| `payment.fx.default_margin` | 0.02 (2%) | 0.00~0.10 | FX 수수료 명시 | Session 2 |
| `payment.fx.base_currency` | "JPY" | "JPY"/"KRW"/"USD" | 결제 표시 기준 | Session 2 |
| `payment.min_charge_jpy` | 500 | ¥100~¥5,000 | 최소 결제 | Session 2 |

### 3-13. 📋 검수 SLA (Review SLAs)

| Key | 기본값 | 허용 범위 | UI 영향 | 출처 |
|-----|--------|----------|---------|------|
| `review.supplier.response_hours` | 72 | 24~168 | 공급자 가입 SLA | Session 2 |
| `review.mate.signup_days` | 3 | 1~7 | 메이트 가입 SLA | Session 2 |
| `review.product.days` | 3 | 1~7 | 상품 검수 SLA | Session 2 |
| `review.dispute.response_hours` | 24 | 4~72 | 분쟁 응대 SLA | Session 2 |
| `review.medical_disclaimer.days` | 1 | 1~3 | 의료 고지 검수 | Session 2 |

---

## 4. DB 스키마 제안

```sql
-- 관리자 설정 테이블
CREATE TABLE platform_config (
  key VARCHAR(128) PRIMARY KEY,
  value JSONB NOT NULL,
  value_type VARCHAR(32) NOT NULL,
  category VARCHAR(64) NOT NULL,
  default_value JSONB,
  min_value JSONB,
  max_value JSONB,
  allowed_values JSONB,
  description TEXT,
  updated_at TIMESTAMP DEFAULT NOW(),
  updated_by UUID REFERENCES users(id)
);

-- 변경 이력
CREATE TABLE config_change_log (
  id UUID PRIMARY KEY,
  key VARCHAR(128) NOT NULL,
  old_value JSONB,
  new_value JSONB,
  changed_by UUID REFERENCES users(id),
  changed_at TIMESTAMP DEFAULT NOW(),
  reason TEXT
);
```

---

## 5. 캐싱 전략

- **서버사이드 캐시**: Next.js `unstable_cache` 또는 Redis, TTL 5분
- **변경 시 무효화**: 관리자 변경 API가 캐시 키 invalidate
- **클라이언트**: SWR 또는 Tanstack Query, staleTime 1분
- **긴급 반영 필요한 값** (예: 차단 플래그): TTL 1분 + 서버 push

> AI 관련 캐시 (`ai.cache.*`)는 별도 레이어. `AI_COST_MODEL §6` 참조.

---

## 6. 관리자 페이지 구조 (SITEMAP_SPEC 보완)

`/admin/config` 하위 탭:
```
/admin/config/pricing        # 팩·AI·Mate 가격
/admin/config/sessions       # 세션 규칙
/admin/config/tiers          # 승급 기준
/admin/config/ai             # AI 모델·fallback·캐시·batch (§3-4)
/admin/config/locale         # 다국어 (§3-5)
/admin/config/access         # KBeauty 차단·접근 제어
/admin/config/safety         # 긴급·안전
/admin/config/content        # 히어로·프로모션
/admin/config/features       # 기능 플래그
/admin/config/kbeauty        # K-뷰티 전용 (§3-9)
/admin/config/commission     # 수수료·정산 (§3-10)
/admin/config/refund         # 환불·취소 (§3-11)
/admin/config/payment        # 결제·통화 (§3-12)
/admin/config/review         # 검수 SLA (§3-13)
/admin/config/history        # 변경 이력 조회
```

각 탭 기능:
- 현재 값 표시
- 인라인 수정
- 저장 시 확인 모달
- 변경 사유 입력 (audit trail)
- 이력 링크

---

## 7. 개발 체크리스트 (구현 시 준수)

- [ ] 새 기능 개발 시 "이 숫자·정책이 향후 바뀔 가능성 있나?" 질문
- [ ] Yes면 → 이 문서에 파라미터 추가 → config에서 읽기
- [ ] UI 텍스트에 하드코딩된 숫자 없는지 lint
- [ ] 0·음수·극단값에서 UI 깨지지 않는지 테스트
- [ ] i18n 통화 포맷 적용
- [ ] 캐시 TTL·무효화 설계

---

## 8. 변경 영향 — 기존 문서 업데이트 필요

- [x] **AI_ROLE_DEFINITION.md** §7 (mini 기본 원칙 반영 완료)
- [x] **AI_COST_MODEL.md** 신규 작성 (§3-4 설계 근거)
- [ ] **FESTIMATE_PLAYBOOK.md** §7: FestiPoint 구체 수치를 "관리자 조정 대상 (이 문서 참조)"로 변경
- [ ] **SITEMAP_SPEC.md** §3: `/admin/config/*` 라우트 15개 추가
- [ ] **MEMBERSHIP_TIERS.md**: 승급 기준을 "기본값 (관리자 조정 가능)"으로 명시
- [ ] **LEGAL_CHECKLIST.md**: K-뷰티 법무 항목 5건 추가

---

## 변경 이력

| 날짜 | 변경 | 사유 |
|------|------|------|
| 2026-04-19 | 최초 작성 | 사용자 통찰 "관리자 페이지에서 수시 변경 가능하도록" |
| 2026-04-19 (Session 2) | §3-4 AI 모델 전부 mini, §3-5 locale 추가, §3-9 K뷰티 신설, §3-10~13 Tier 1~2 누락 보완 | CLI 세션에서 gap 식별 |
| 2026-04-19 (v1.2) | §3-4 Fallback·Cache·Batch 확장. AI_COST_MODEL.md 신규 작성하여 설계 근거 분리 | 사용자 제안 (fallback·cache·batch) + threshold 0.7 추상성 해소 |
| 2026-04-19 (v1.3) | §3-3 Offering 2축 모델(13키) + Matching 정책(12키) + Tier 필드 리스트(3키) 추가 | Session 3 매칭 플로우 3개 결정 반영 |

---

*버전: v1.3 / 마지막 업데이트: 2026-04-19*
