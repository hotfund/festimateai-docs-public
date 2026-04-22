# 사이트맵 & 페이지 구조 명세 (v1.0)

> **작성**: 2026-04-14
> **용도**: 개발·디자인 착수의 기준 구조

---

## 1. 최상위 구조

```
FestiMate (외국인 인바운드 메인 사이트)
│
├─ 📂 여행 콘텐츠 (7 카테고리, 드롭다운)
│   ├─ KFestival
│   ├─ KBeauty (🔒 한국인 차단)
│   ├─ KMaster (체험 + 쇼핑몰)
│   ├─ KGarden
│   ├─ KFood
│   ├─ KSleep
│   └─ Shopping
│
├─ 🤖 AI Guide (독립 페이지)
│
├─ 🤝 Mate
│   ├─ From Mate (메이트가 모집)
│   └─ From Tourists (여행자가 요청 - 한국인·외국인 모두)
│
├─ ℹ️ Information
│
├─ ⚙️ Admin (관리자)
│
└─ 🌏 KoJap (Phase 2 - 한국인 전용 일본 여행 섹션)
```

---

## 2. 상단 네비게이션 (PC 기준)

**옵션 B 확정 — 드롭다운 방식**

```
[FestiMate]  탐색 ▼   AI Guide   Mate   Info   [🌐 JP ▼]  [로그인]
             │
             ├─ 🎪 KFestival
             ├─ 💄 KBeauty
             ├─ 🏆 KMaster
             ├─ 🌿 KGarden
             ├─ 🍜 KFood
             ├─ 🛏 KSleep
             └─ 🛍 Shopping
```

**모바일**: 햄버거 메뉴 (≡) 내부 동일 구조.

---

## 3. URL 구조

### 여행자 (외국인 인바운드)
```
festimate.ai                          → 언어 자동 감지
festimate.ai/jp                       → 일본어 (메인 타겟)
festimate.ai/en                       → 영어
festimate.ai/tw                       → 대만 (Phase 2)

festimate.ai/jp/festival              카테고리 리스트
festimate.ai/jp/festival/[slug]       축제 상세

festimate.ai/jp/beauty                K-뷰티 (gating)
festimate.ai/jp/beauty/[slug]

festimate.ai/jp/master                명인명장
festimate.ai/jp/master/[slug]?tab=experience
festimate.ai/jp/master/[slug]?tab=shop       ⭐ 쇼핑몰 탭

festimate.ai/jp/garden
festimate.ai/jp/garden/[slug]

festimate.ai/jp/food
festimate.ai/jp/food/[slug]

festimate.ai/jp/sleep
festimate.ai/jp/sleep/[slug]

festimate.ai/jp/shopping
festimate.ai/jp/shopping/[slug]
```

### AI Guide (독립 페이지)
```
festimate.ai/jp/ai-guide              AI 가이드 메인
festimate.ai/jp/ai-guide/[plan-id]    저장된 플랜 공유 URL
```

### Mate
```
festimate.ai/jp/mate                  Mate 허브
festimate.ai/jp/mate/open             From Mate (모집글)
festimate.ai/jp/mate/request          From Tourists (요청글, 한·외국인 모두)
festimate.ai/jp/mate/find             메이트 페어 검색
festimate.ai/jp/mate/[pair-slug]      페어 상세
```

### 여행자 로그인 영역
```
festimate.ai/jp/my                    대시보드
festimate.ai/jp/my/trips              예약 내역
festimate.ai/jp/my/wishlist           위시리스트
festimate.ai/jp/my/messages           메시지
festimate.ai/jp/my/reviews            내 리뷰
festimate.ai/jp/my/settings           계정 설정
```

### 공급자 포털 (KBeauty · KMaster · 템플스테이만 해당)
```
festimate.ai/host                     공급자 메인
festimate.ai/host/signup              가입·사업자 인증
festimate.ai/host/onboarding          카테고리 선택·인증 문서
festimate.ai/host/dashboard           매출·예약·리뷰 요약
festimate.ai/host/products            내 상품 리스트
festimate.ai/host/products/new        상품 등록 (Wizard)
festimate.ai/host/products/[id]/edit  상품 수정
festimate.ai/host/bookings            예약 관리
festimate.ai/host/reviews             받은 리뷰
festimate.ai/host/revenue             정산
festimate.ai/host/messages            고객 문의
festimate.ai/host/settings            계정 설정
```

### 메이트 포털 (학생 전용)
```
festimate.ai/mate-portal              메이트 포털 메인
festimate.ai/mate-portal/signup       학교·학적 인증
festimate.ai/mate-portal/pair         페어 초대·승인
festimate.ai/mate-portal/orientation  오리엔테이션
festimate.ai/mate-portal/dashboard    내 활동 요약
festimate.ai/mate-portal/requests     매칭 요청 수신함
festimate.ai/mate-portal/sessions     세션 내역
festimate.ai/mate-portal/tier         티어 승급 현황
festimate.ai/mate-portal/credits      포인트·크레딧
festimate.ai/mate-portal/profile      공개 프로필 편집
festimate.ai/mate-portal/products     유료 상품 기획 (Verified만)
```

### 관리자
```
festimate.ai/admin                    대시보드
festimate.ai/admin/approvals          승인 큐 (공급자·상품·메이트)
festimate.ai/admin/users              회원 관리
festimate.ai/admin/bookings           예약 모니터
festimate.ai/admin/disputes           분쟁·CS
festimate.ai/admin/payouts            정산 실행
festimate.ai/admin/translations       번역 교정
festimate.ai/admin/content            블로그·공지
festimate.ai/admin/analytics          KPI
```

### 정보·법무
```
festimate.ai/jp/info/about
festimate.ai/jp/info/how-it-works
festimate.ai/jp/info/membership
festimate.ai/jp/info/faq
festimate.ai/jp/info/safety
festimate.ai/jp/info/terms
festimate.ai/jp/info/privacy
festimate.ai/jp/info/contact
```

---

## 4. 데이터 소스 & 예약/결제 매트릭스

| 카테고리 | 데이터 소스 | 예약 | 결제 | 우리 수익 | 공급자 포털 필요 |
|---|---|---|---|---|---|
| KFestival | KTO API 주간 | 대부분 무료 / 일부 대행 | - | 스폰서십·광고 | ❌ |
| KBeauty | 파트너 DB + 공급자 직접 | ✅ 우리 | ✅ 우리 | 🔥 외국인환자 유치 수수료 | ✅ |
| KMaster (체험) | 공급자 직접 업로드 | ✅ 우리 | ✅ 우리 | 수수료 20% | ✅ |
| KMaster (쇼핑몰) | 공급자 직접 업로드 | - | ✅ 우리 | 커머스 수수료 + 배송 | ✅ |
| KGarden | 큐레이션 + 일부 공급자 | 무료 입장 중심 | - | 정보 제공 | 선택적 |
| KFood | 네이버 API 월간 | 캐치테이블 제휴 (Phase 2) | 현장 | 제휴 커미션 (Phase 2) | ❌ |
| KSleep | Agoda/Booking API (Phase 1 후반) | 외부 어필리에이트 | 외부 | 어필리에이트 3~4% | ❌ |
| Shopping | 네이버 + 지역 관광청 | - | - | 정보 + 광고 | ❌ |
| Mate 매칭 | 학생 UGC | ✅ 우리 | Phase 1 무료 / Phase 2 유료 | 매칭 수수료 20% | 메이트 포털 |
| Templestay | 공급자 직접 | ✅ 우리 | ✅ 우리 | 수수료 20% | ✅ |

**요약**:
- **공급자 포털이 필요한 카테고리**: KBeauty, KMaster, Templestay
- **자동 DB 갱신**: KFestival, KFood, KSleep, Shopping
- **사용자 생성(UGC)**: Mate (학생 UGC)

---

## 5. 핵심 페이지 레이아웃 가이드

### 홈 (`/jp`)
1. 히어로 + 슬로건 + 통합 검색창
2. 3-카드: Festival · Mate · Beauty
3. 이번 달 축제 (6개)
4. 명인명장 하이라이트 (6명)
5. 일본 여성 후기 (Instagram 임베드)
6. 비교표 (ChatGPT / Klook / Creatrip / FestiMate)
7. CTA: AI Guide / 회원가입

### 카테고리 리스트 (`/jp/festival` 등)
- 필터 (지역·날짜·가격·태그)
- 지도 토글
- 카드 그리드 (사진·제목·가격·평점)
- "AI Guide로 조합하기" CTA

### 상품 상세 (`/jp/master/yi-yonggang`)
1. 갤러리 슬라이더
2. 제목·평점·위치·언어·가격
3. 예약 달력·정원
4. 【이 체험의 특별함】
5. 【진행 일정】 (시간별)
6. **【자주 함께 예약되는 상품】** (AI 추천, 여러 카테고리 혼합)
7. 지도
8. 리뷰
9. 취소 정책·FAQ

### KMaster 상세 특수 구조 (체험 + 쇼핑몰)
```
[탭 1: 🎨 공방 체험]          [탭 2: 🛒 작품 쇼핑]
 - 1박2일 도자기 체험           - 명장 제작 도자기
 - 가격·일정·예약                - 민화 작품
                                - 장바구니 → 배송 주문
```

### AI Guide (`/jp/ai-guide`)
- 좌: 채팅 패널
- 우: 지도 + 타임라인 (카테고리 컬러 마커)
- 하단: Day별 카드 리스트
- 버튼: 지도 뷰 / 타임라인 뷰 / 비교 모드

### Mate Hub (`/jp/mate`)
- 탭: `From Mate` · `From Tourists` · `Find Mate`
- 카드 리스트 (사진·학교·언어·평점·모집 정보)

---

## 6. K-뷰티 한국인 차단 구현

**다층 방어 (IP + 언어 + 로그인)**

1. **Level 1 (IP)**: 한국 IP 접속 시 `/beauty` 라우트 **403 차단**
2. **Level 2 (언어 UI)**: `lang=kr` 설정 시 KBeauty 네비·홈 섹션 숨김
3. **Level 3 (로그인)**: 로그인 사용자 국적 정보로 이중 확인
4. **Level 4 (KoJap 전환)**: 한국인 사용자에게 KoJap 사이트 자동 추천

**KoJap (Phase 2) 별도 섹션 구조 예정** — 한국인이 일본 여행할 때 사용.

---

## 7. 네비게이션 상세 로직

### 언어 전환 시 메뉴 변경
```
JP/EN 사용자 (외국인 인바운드):
  탐색 ▼ │ AI Guide │ Mate │ Info

KR 사용자 (한국인):
  탐색 ▼ │ AI Guide │ Mate │ Info │ KoJap (Phase 2)
  
  ※ 탐색 드롭다운에서 KBeauty 제거
```

### 로그인 상태별 네비
```
비로그인:  [로그인] [회원가입]
로그인:    [🔔 알림] [Profile ▼]
                      │
                      ├─ 내 여행 (My Trips)
                      ├─ 위시리스트
                      ├─ 메시지
                      ├─ 내 리뷰
                      ├─ 계정 설정
                      └─ 로그아웃

메이트:    [🔔] [Profile ▼]
                      ├─ 메이트 포털
                      └─ (이하 동일)

공급자:    [🔔] [Profile ▼]
                      ├─ 공급자 포털
                      └─ (이하 동일)
```

---

## 8. 상태별 페이지 플로

### 여행자 전체 여정
```
1. 랜딩 → 검색 or AI Guide or 카테고리 탐색
2. 상품 상세 열람
3. "자주 함께 예약되는 상품" 통해 플랜 확장
4. My Trip에 담기 (장바구니 대체)
5. AI Guide에서 전체 플랜 확인·조정
6. Verified 인증 (최초 1회)
7. 일괄 예약·결제
8. 예약 확정 → 호스트/메이트와 메시지
9. 여행 진행 (체크인·체크아웃)
10. 리뷰 작성 → 포인트 적립
```

### 공급자 전체 여정
```
1. /host/signup → 사업자 인증 서류 제출
2. 관리자 승인 대기 (1~3일)
3. /host/products/new → Wizard 9-Step 완료
4. 관리자 상품 검수 (1~3일)
5. 승인 → 공개 노출
6. 예약 수신 → 수락/거절 (24h)
7. 세션 진행
8. 게스트 리뷰 수신
9. 월 정산 수령 (수수료 20% 제외)
```

### 메이트 전체 여정
```
1. 학교 가입 공지 통해 /mate-portal/signup
2. 학적 증명·신분증 제출
3. 페어 파트너 초대 (친구) or 솔로 신청 (대면 검증)
4. 페어 확정 + 오리엔테이션 수료
5. Rookie 티어 활성화
6. From Mate 모집글 작성 or From Tourists 요청 대기
7. 세션 진행 (체크인·체크아웃)
8. 평점 수신
9. 3회 × 4.5+ 달성 → Verified 승급
10. 유료 상품 제안 권한 활성화
11. 지속 활동 → Featured → Pro Guide 진로
```

---

## 9. 반응형 & 모바일 최우선

- **일본 20~30 여성 = 98% 모바일 유입**
- 모든 페이지 모바일 퍼스트 설계
- 터치 타깃 44px 이상
- 이미지 WebP + lazy loading
- Lighthouse 성능 90+ 목표
- 오프라인 대응 (예약 확인서 PWA 캐시)

---

## 10. 다국어 SEO 전략

- URL 서브패스 방식 (`/jp`, `/en`) → 도메인 권위 통합
- `hreflang` 태그 정확 설정
- 카테고리별 롱테일 SEO
  - 예: `/jp/festival/jinju-lantern-2026` 형식으로 연도·지역 포함
- 구조화 데이터 (Schema.org) 전면 적용
  - TouristAttraction, Event, LocalBusiness, Product
- 인플루언서 UGC 통한 백링크 확보

---

## 11. 미결 사항 (다음 세션에서)

1. 상단 네비 순서 (탐색·AI·Mate·Info) 확정
2. Footer 구성
3. 검색 UX 상세 (필터·정렬·지도 토글)
4. 상품 상세의 "함께 예약" AI 추천 로직
5. 모바일 바텀 내비 여부 (홈·탐색·AI·Mate·My)

---

*버전: v1.0 / 작성: 2026-04-14*
