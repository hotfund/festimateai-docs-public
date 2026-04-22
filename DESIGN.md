# Design System — FestiMate

## Product Context
- **What this is:** 한국 인바운드 여행 플랫폼 (축제 + K-뷰티 + 메이트 매칭)
- **Who it's for:** 일본 20~30대 여성 여행자 (primary), 한국 대학생 메이트 (supply side)
- **Space/industry:** 여행 플랫폼 (Klook, Creatrip 경쟁)
- **Project type:** Web app, mobile-first
- **Differentiator:** "친구의 추천" 느낌. 트랜잭션이 아닌 친밀감

## Language-Aware Messaging
- **한국어 (메이트용):** "일본 친구와 함께, 한국을 알려주세요."
- **일본어 (여행자용):** "韓国に、親友ができる。"
- **영어 (글로벌):** "Find Your Korean Bestie."
- 한국어 페이지는 메이트(공급측) 관점, 일본어/영어는 여행자(수요측) 관점

## Aesthetic Direction
- **Direction:** Organic/Warm
- **Decoration level:** Intentional (subtle warmth, not decorative)
- **Mood:** 친구가 보내준 여행 추천 같은 따뜻함. 예약 사이트가 아닌 친구의 편지
- **Anti-patterns:** 보라색 그라디언트, AI slop, 클럽/파티/로맨스 이미지 절대 금지

## Typography
- **Display/Hero:** Noto Serif JP — 일본어 세리프. 고급스러우면서 친근함. "편지" 느낌
- **Body:** Noto Sans JP + Pretendard — 일본어/한국어 가독성 최고
- **UI/Labels:** Geist — 버튼/레이블 깔끔함
- **Data/Tables:** Geist (tabular-nums)
- **Code:** Geist Mono
- **Loading:** Google Fonts CDN
- **Scale:** 13px(caption) / 14px(ui) / 15px(body) / 17px(card-title) / 24px(h2) / 28px(section) / 42px(hero)

## Color
- **Approach:** Restrained + Warm Accent
- **Primary:** #E85D3A (warm coral) — 축제의 활기, K-뷰티의 디테일. 경쟁사 누구도 안 쓰는 색
- **Primary hover:** #D14E2E
- **Primary light:** #FFF1ED
- **Background:** #FAF8F6 (warm gray, 순백 아님)
- **Card/Surface:** #FFFFFF
- **Text:** #1C1917
- **Text muted:** #78716C
- **Text light:** #A8A29E
- **Border:** #E7E5E4
- **Semantic:** success #16A34A, warning #F59E0B, error #DC2626, info #3B82F6
- **Dark mode:** BG #1C1917, Card #292524, Text #FAF8F6, Border #44403C
- **Category markers:** festival #EF4444, beauty #3B82F6, master #8B5CF6, templestay #14B8A6, garden #10B981, food #F59E0B, sleep #06B6D4, shopping #EC4899, mate #EAB308, transit #6B7280

## Spacing
- **Base unit:** 4px
- **Density:** Comfortable (일본 사용자 기대에 맞는 적당한 정보 밀도)
- **Scale:** 2xs(2) xs(4) sm(8) md(16) lg(24) xl(32) 2xl(48) 3xl(64)

## Layout
- **Approach:** Grid-disciplined, mobile-first
- **Grid:** 1col(mobile) / 2col(tablet) / 3col(desktop)
- **Max content width:** 960px
- **Border radius:** sm:4px, md:8px, lg:12px, xl:16px, full:9999px
- **Hero:** 포스터처럼 대담하게. 실제 사진 + 세리프 헤딩
- **Cards:** rounded-xl(16px), subtle shadow, hover lift

## Motion
- **Approach:** Minimal-functional
- **Easing:** enter(ease-out) exit(ease-in) move(ease-in-out)
- **Duration:** micro(50-100ms) short(150-250ms) medium(250-400ms)
- **Only:** 페이지 전환, 카드 hover, 탭 전환. 불필요한 애니메이션 없음

## Content Guidelines
- **로맨스 마케팅 절대 금지** — 한일 커플 자연 발생은 허용하되, 광고/마케팅에 로맨스 요소 사용 불가
- **이미지 톤:** 축제 야경, 전통시장, 공방 체험, 사찰, 자연. 클럽/파티/노출 이미지 사용 금지
- **매칭 페이지 비주얼:** 한국 대학생 + 일본 여행자가 시장에서 같이 먹거나, 축제에서 셀카 찍는 장면
- **신뢰 지표 필수:** 숫자(500+ 축제, 193 명장, 30+ 클리닉, 22+ 도시) 히어로 아래 배치

## Decisions Log
| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-04-17 | Warm coral #E85D3A 확정 | Klook(주황)/Creatrip(파랑)/Jalan(붉노랑) 모두와 차별화 |
| 2026-04-17 | Noto Serif JP hero 확정 | 산세리프 일색인 여행 플랫폼에서 "친구의 편지" 차별화 |
| 2026-04-17 | 보라색 그라디언트 제거 | 기존 사이트의 AI slop 톤 탈피 |
| 2026-04-17 | 한국어=메이트 관점 확정 | 한국 페이지는 공급측(대학생), 일본어/영어는 수요측(여행자) |
| 2026-04-17 | 로맨스 마케팅 금지 확정 | PLAYBOOK 정책 준수. 사업/법적/안전 리스크 |
