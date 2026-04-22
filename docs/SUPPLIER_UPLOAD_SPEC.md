# 공급자 상품 업로드 폼 설계 (v1.0)

> **목적**: FestiMate에 상품을 올릴 수 있는 공급자별 업로드 폼 명세.
> **사용**: 개발팀의 구현 스펙, 법무팀의 검토 기준, 운영팀의 검수 기준으로 사용.
> **작성**: 2026-04-14

---

## 1. 공급자 카테고리 (7종)

| 코드 | 카테고리 | 대표 공급자 | 파일럿 우선순위 |
|---|---|---|---|
| `master` | 명인명장 | 이용강 명장 (도자기) | 🔥 1순위 |
| `templestay` | 템플스테이 | 법주사 | 🔥 1순위 |
| `clinic` | 피부과·의료 | 강남 피부과 3~5곳 | 🔥 1순위 |
| `mate` | 학생 메이트 페어 | 청주대 관광·일어과 | 🔥 1순위 |
| `festival` | 축제사무국 | 청주 직지축제 | 🟡 2순위 |
| `garden` | 정원·박물관 | 소쇄원, 명장 박물관 | 🟡 2순위 |
| `chef` | 셰프·맛집·쿠킹 | 지역 유명 셰프 | 🟢 3순위 |

---

## 2. 공통 필드 (모든 카테고리 공유)

### 2.1 공급자 정보 (최초 1회 등록)
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 상호/기관명 | string | ✅ | |
| 사업자등록번호 or 고유번호 | string | ✅ | 개인(메이트)은 학적 대체 |
| 대표자명 | string | ✅ | |
| 담당자 이름·연락처·이메일 | string | ✅ | |
| 은행·계좌번호 (정산용) | string | ✅ | |
| 주소 | string | ✅ | |
| 상호 로고 | image | ⭕ | |
| 소개글 (한/일/영) | rich text | ✅ | 번역은 AI 초안 제공 |
| 인증 문서 업로드 | file | ✅ | 카테고리별 상이 (하단 참조) |

### 2.2 상품 기본 정보 (상품별 입력)
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 상품명 (한/일/영) | string×3 | ✅ | AI 번역 초안 |
| 상세 설명 (한/일/영) | rich text×3 | ✅ | AI 번역 초안 |
| 메인 사진 (1장) | image | ✅ | 1200×800 이상 |
| 갤러리 사진 (3~10장) | image[] | ✅ | |
| 영상 (선택) | video | ⭕ | 최대 60초 |
| 태그 | string[] | ✅ | AI 추천 + 수동 추가 |
| 난이도 | enum | ⭕ | 쉬움/보통/어려움 |
| 적정 연령 | enum | ⭕ | 전연령/13+/19+ |
| 지원 언어 | enum[] | ✅ | KR/JP/EN/CN |

### 2.3 위치 정보
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 주소 (도로명) | string | ✅ | |
| 위도·경도 | coord | ✅ | 자동 지오코딩 |
| 가장 가까운 지하철·KTX | string | ⭕ | |
| 주차 가능 여부 | bool | ✅ | |
| 휠체어 접근 | bool | ✅ | 일본 고객 중요 요소 |

### 2.4 가격·정산
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 가격 (1인 기준) | number | ✅ | 원화 |
| 가격 (외화 표시) | auto | - | 실시간 환율 계산 |
| 포함 항목 | text | ✅ | 예: 재료비·차·사진 포함 |
| 불포함 항목 | text | ✅ | 예: 식사·교통 별도 |
| 추가 비용 가능성 | text | ⭕ | |
| 수수료율 | auto | - | **20%** (고정) |
| 공급자 정산 금액 | auto | - | 자동 계산 표시 |

### 2.5 일정·정원
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 세션 길이 (분) | number | ✅ | |
| 운영 요일 | weekday[] | ✅ | |
| 운영 시간대 | time range | ✅ | |
| 시즌성 (월별 가능 여부) | month[] | ✅ | 벚꽃·단풍 등 |
| 최소 인원 | number | ✅ | |
| 최대 인원 | number | ✅ | |
| 예약 마감 | hours before | ✅ | 기본 24h |

### 2.6 취소·환불 정책
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 취소 정책 | enum | ✅ | 유연/보통/엄격 |
| 환불 기한 | days | ✅ | |
| 노쇼 위약금 | % | ✅ | |

### 2.7 안전·보험
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 비상 연락처 | phone | ✅ | |
| 배상책임보험 가입 여부 | bool | ✅ | 카테고리별 권고·의무 상이 |
| 보험 증권 업로드 | file | ⭕ | |
| 안전 주의사항 (한/일/영) | text×3 | ✅ | |

---

## 3. 카테고리별 특화 필드

### 3.1 🏆 명인명장 (master)
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 전승 분야 | enum | ✅ | 도자기/민화/나전/금속공예/한지/목공/서예/전통음식/공연 등 |
| 인증 유형 | enum | ✅ | 대한민국명장/명인/전승자/무형문화재/광역무형문화재 |
| 인증 번호·연도 | string | ✅ | |
| 체험 내용 | multi-checkbox | ✅ | 설명·시연·체험·제작·시음 |
| 제작 결과물 유무 | bool | ✅ | |
| 결과물 배송 | bool | ⭕ | 해외 배송 가능 여부 |
| 강의 언어 구사 | enum[] | ✅ | KR/JP/EN/CN |
| 숙박 연계 | bool | ⭕ | 있을 시 연결 상품 링크 |
| 공방 수용 인원 | number | ✅ | |
| 에어컨·난방 | bool | ⭕ | |

**인증 문서**: 명장 인증서 사본, 명인 소개 자료, 언론 보도 (있을 시)

---

### 3.2 🏯 템플스테이 (templestay)
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 사찰명 | string | ✅ | |
| 종파 | enum | ✅ | 조계종/태고종/천태종 등 |
| 창건 시대 | string | ⭕ | |
| 프로그램 타입 | enum | ✅ | 체험형/휴식형/수행형 |
| 포함 활동 | multi-checkbox | ✅ | 108배/참선/예불/공양/다도/염주/서예 |
| 방사 유형 | enum | ✅ | 1인실/2인실/다인실/남녀분리 |
| 식사 | number | ✅ | 사찰음식 끼니 수 |
| 세면·샤워 | enum | ✅ | 공용/개별 |
| 템플 근처 걷기 코스 | bool | ⭕ | |
| 다국어 가이드 | enum[] | ✅ | |
| 문화재청 등록 템플 | bool | ✅ | 정책적 표시 |

**인증 문서**: 템플스테이운영사업자 지정서, 사찰 등록증

---

### 3.3 🏥 피부과·의료 (clinic) — 법적 민감
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 의료기관명 | string | ✅ | |
| 사업자등록번호 | string | ✅ | |
| **의료기관 허가번호** | string | ✅ | **법정 필수** |
| 대표 전문의 자격 | enum | ✅ | 피부과/성형외과/일반의 |
| **외국인환자유치의료기관 등록번호** | string | ✅ | 법정 요건 |
| 시술명 | string | ✅ | |
| 시술 분류 | enum | ✅ | 보톡스/필러/레이저/HIFU/스케일링/더말토닝 등 |
| 시술 부위 | multi-checkbox | ✅ | 이마/미간/팔자/턱/얼굴전체/목 |
| 사용 브랜드·용량 | string | ✅ | 예: 앨러간 보톡스 100U |
| 시술 시간 (분) | number | ✅ | |
| **회복 기간 (일)** | number | ✅ | **AI 일정 최적화 핵심** |
| 재시술 권고 주기 | months | ⭕ | |
| 부작용·금기 사항 | text | ✅ | |
| 시술 전 주의사항 (한/일/영) | text×3 | ✅ | |
| 시술 후 주의사항 (한/일/영) | text×3 | ✅ | |
| 외국인 전용 의료진 | bool | ✅ | |
| 일본어 통역 상주 | bool | ✅ | |
| 예약 가능 시간대 | time range | ✅ | |
| 상담 언어 | enum[] | ✅ | |

**인증 문서**:
- 의료기관 개설 허가증
- 외국인환자유치의료기관 등록증
- 전문의 자격증
- 사용 약품 식약처 허가증

**법적 주의사항 (플랫폼 자동 처리)**:
- 한국 IP·한국어 UI 접속자에게는 가격·사진 **블록**
- 의료광고법 56조 관련 문구 하단 자동 삽입
- 공지: "본 정보는 외국인 환자 대상입니다"

---

### 3.4 🤝 학생 메이트 페어 (mate)
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 학교명 | string | ✅ | 학교 제휴 리스트 |
| 학과 | string | ✅ | |
| 학년 | number | ✅ | |
| 성별 | enum | ✅ | 매칭 룰 적용 |
| 페어 파트너 ID | user_id | ✅ | 별도 가입 → 연결 |
| **페어 타입** | auto | - | FF / MF 자동 판정 (MM 금지) |
| 본인 인증 문서 | file | ✅ | 학생증·재학증명서 |
| 신분증 대조 사진 | image | ✅ | 본인확인 |
| 일본어 수준 | enum | ✅ | 초급/중급/상급/원어민 수준 |
| JLPT 점수 | string | ⭕ | 있을 시 |
| 지역 전문성 | string[] | ✅ | 거주/학교/방문 많은 지역 |
| 관심 분야 | multi-checkbox | ✅ | K-POP/맛집/쇼핑/역사/자연/뷰티/미술 |
| 가능 요일·시간대 | calendar | ✅ | |
| 본인 사진 | image | ✅ | 얼굴 확인 |
| 자기소개 영상 | video | ✅ | 30초, 일본어 시도 권장 |
| 오리엔테이션 수료 | bool | ✅ | 학교 파트너십 내 |
| 활동 반경 | km | ✅ | 숙박비 본인 부담 원칙 |

**인증 문서**:
- 재학증명서 (학기마다 갱신)
- 신분증
- 학교 추천서 (학과장 서명)
- 보험 가입 확인서

**특별 룰**:
- MM 페어 등록 불가 (시스템 차단)
- 둘 다 동시 승인되어야 매칭 풀 진입
- 3회 세션 + 평점 4.5+ → Verified 승급 → 유료 상품 권한

---

### 3.5 🎪 축제사무국 (festival)
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 축제명 | string | ✅ | |
| 주관·주최 | string | ✅ | |
| 개최 기간 | date range | ✅ | |
| 개최지 (회장) | string | ✅ | |
| 예상 방문객 수 | number | ⭕ | |
| 입장 방식 | enum | ✅ | 무료/유료/사전예약제 |
| 티켓 유형 | string[] | ⭕ | 일반/VIP/외국인특별 |
| 프로그램 시간표 | table | ✅ | 시간대별 공연·행사 |
| 일본어 안내 책자 | bool | ✅ | |
| 외국인 특별 부스 | bool | ⭕ | |
| 촬영 허용 구역 | text | ⭕ | |
| 교통편·주차 | text | ✅ | |
| KTO 등록 번호 | string | ⭕ | 정부 축제 목록 연계 |

**인증 문서**: 축제 주관 기관 공문, 지자체 후원 서류

---

### 3.6 🌿 정원·박물관 (garden)
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 기관명 | string | ✅ | |
| 유형 | enum | ✅ | 전통정원/현대정원/미술관/박물관/공예관 |
| 개장시간 | time range | ✅ | |
| 휴무일 | weekday[] | ✅ | |
| 입장료 | number | ✅ | 일반/학생/외국인 구분 |
| 도슨트 투어 | bool | ✅ | |
| 가이드 언어 | enum[] | ⭕ | |
| 계절 하이라이트 | text | ✅ | 3~5월 벚꽃, 10월 단풍 등 |
| 촬영 가능 | bool | ✅ | |
| 상설 전시 | text | ⭕ | |
| 기획 전시 일정 | table | ⭕ | |

---

### 3.7 🍲 셰프·맛집·쿠킹 (chef)
| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 식당명 | string | ✅ | |
| 음식 종류 | enum | ✅ | 한식/분식/디저트/면/고기 등 |
| 쿠킹클래스 여부 | bool | ✅ | |
| 대표 메뉴·가격 | table | ✅ | |
| 좌석 수 | number | ✅ | |
| 예약 가능 | bool | ✅ | |
| 알레르기 대응 | multi-checkbox | ⭕ | 견과/갑각류/유제품/글루텐 |
| 채식·할랄 | enum | ⭕ | 일반/채식/비건/할랄 |
| 주류 판매 | bool | ✅ | 20세+ 확인 필요 |
| 미쉐린·블루리본 수상 | string | ⭕ | |

**인증 문서**: 사업자등록증, 위생업 허가증

---

## 4. 업로드 UX 플로 (Step-by-step Wizard)

```
Step 1: 카테고리 선택 (7개 중 1)
   ↓
Step 2: 공급자 인증 (최초 1회만, 이미 등록되어 있으면 스킵)
   - 사업자 정보 입력
   - 인증 문서 업로드
   - 담당자 정보
   - 은행 계좌
   → 운영팀 검수 후 승인 (1~3일)
   ↓
Step 3: 상품 기본 정보
   - 상품명 (한글 입력 → AI 번역 → 수정)
   - 설명 (한글 → AI 번역 → 수정)
   - 메인 사진 + 갤러리
   - 태그 (AI 추천 + 수동)
   ↓
Step 4: 위치·시간·가격
   - 주소 (지도 찍기)
   - 운영 시간·요일·시즌
   - 최소·최대 인원
   - 가격 + 포함/불포함
   ↓
Step 5: 카테고리 특화 정보
   - 카테고리별 필드 동적 표시
   ↓
Step 6: 안전·정책
   - 취소 정책
   - 안전 주의사항
   - 보험
   ↓
Step 7: 미리보기 + 제출
   - 고객이 볼 화면 미리보기
   - 번역 최종 확인
   - 검수 요청 제출
   ↓
Step 8: 운영 검수 (1~3일)
   - 가격 적정성
   - 사진·설명 품질
   - 법적 요건 (특히 clinic)
   - 중복·스팸 체크
   → 승인 / 수정 요청 / 반려
   ↓
Step 9: 공개 (승인 후 자동)
   - 3개 언어 동시 노출
   - AI 가이드 풀에 편입
   - 검색 인덱스 등록
```

---

## 5. 검수 기준 (운영팀 체크리스트)

### 공통
- [ ] 상호·사업자번호 실제 확인 (국세청 홈택스)
- [ ] 인증 문서 유효성 (만료되지 않았는지)
- [ ] 사진 품질 (저해상도·도용 의심 차단)
- [ ] 설명 번역 품질 (AI 초안 오류 수정)
- [ ] 가격 시장가 대비 적정 (과장·저가 의심)
- [ ] 중복 등록 여부
- [ ] 금칙어·과장 광고 표현
- [ ] 안전 주의사항 기재

### 카테고리 특화
- `master`: 명장·명인 인증서 진위, 연락처 응대 확인
- `templestay`: 문화재청 등록 여부, 사찰 공식 연락망 통화
- `clinic`: **외국인환자유치의료기관 등록 필수**, 의사 자격증, 약품 허가 내역
- `mate`: 학교 공문·재학증명, 페어 둘 다 인증, 오리엔테이션 이수
- `festival`: 주관 기관 공식 공문
- `garden`: 개장 여부 현장 확인
- `chef`: 위생업 허가, 실제 영업 중 확인

---

## 6. 승급·권한 구조

```
공급자 티어:
├─ Pending    → 신청 후 검수 중
├─ Verified   → 공개 판매 가능, 수수료 20%
├─ Preferred  → Verified 중 평점 4.8+ / 완료 20건+
│               → 검색 상단 노출, 수수료 협상 가능
└─ Restricted → 평점 하락 or 분쟁 이력
                → 일시 정지, 자동 숨김

메이트 티어 (별도):
├─ Applied    → 학교 등록 후 파트너 페어 미승인
├─ Pending    → 페어 승인 + 오리엔테이션 대기
├─ Active     → 무료 세션 호스팅 가능 (최대 3회 기본)
├─ Verified   → 3회 × 평점 4.5+ 달성
│               → 유료 상품 기획 권한 활성
└─ Featured   → 유료 상품 10건 + 평점 4.8+
                → 플랫폼 대표 메이트 추천
```

---

## 7. API 엔드포인트 스케치 (개발팀용)

```
POST   /api/suppliers/register          공급자 최초 등록
POST   /api/suppliers/:id/verify-doc    인증 문서 업로드
GET    /api/suppliers/:id/status        승인 상태 조회

POST   /api/products                    상품 신규 등록
GET    /api/products/:id                상품 조회
PATCH  /api/products/:id                상품 수정
DELETE /api/products/:id                상품 삭제 (soft)
POST   /api/products/:id/submit         검수 요청
GET    /api/products/:id/preview        미리보기

POST   /api/admin/products/:id/approve  검수 승인 (운영팀)
POST   /api/admin/products/:id/reject   검수 반려
POST   /api/admin/products/:id/request-revision   수정 요청

POST   /api/translation                 AI 번역 (한→일·영)
POST   /api/media/upload                사진·영상 업로드 (S3)

POST   /api/mates/pairs                 메이트 페어 생성
POST   /api/mates/pairs/:id/confirm     페어 파트너 확인
```

---

## 8. DB 스키마 스케치 (Supabase 기반)

```sql
-- 공급자
suppliers (
  id, type (master|templestay|clinic|mate|festival|garden|chef),
  business_name, business_number, owner_name,
  contact_name, contact_phone, contact_email,
  bank_account, address, tier (pending|verified|preferred|restricted),
  verified_at, created_at, updated_at
)

-- 공급자 인증 문서
supplier_documents (
  id, supplier_id, doc_type, file_url, verified, verified_at
)

-- 상품
products (
  id, supplier_id, category,
  title_kr, title_jp, title_en,
  description_kr, description_jp, description_en,
  main_image, gallery (jsonb), video_url,
  tags (text[]), difficulty, age_rating,
  languages (text[]),
  address, lat, lng, transit_info, parking, wheelchair,
  price_krw, session_minutes,
  operating_days (int[]), operating_time_start, operating_time_end,
  seasonality (int[] of months),
  min_participants, max_participants,
  booking_deadline_hours,
  cancellation_policy, refund_days, noshow_fee_pct,
  emergency_contact, insurance, safety_notes_kr|jp|en,
  category_data (jsonb), -- 카테고리별 필드
  status (draft|submitted|approved|rejected|archived),
  created_at, updated_at, approved_at, approved_by
)

-- 상품 카테고리별 세부 (선택적으로 별도 테이블)
clinic_procedures (
  product_id, procedure_type, body_parts, brand, dosage,
  recovery_days, side_effects, pre_care, post_care,
  foreign_staff, interpreter_languages
)

master_experiences (
  product_id, craft_type, certification_type, certification_number,
  activity_types (text[]), produces_result, teaching_languages,
  workshop_capacity
)

-- 메이트 페어
mate_pairs (
  id, mate_a_id, mate_b_id, pair_type (FF|MF),
  school, department, region_expertise (text[]),
  japanese_level, interests (text[]),
  sessions_completed, avg_rating,
  tier (applied|pending|active|verified|featured),
  orientation_completed, created_at
)

-- 예약·세션
bookings (
  id, product_id, guest_id, mate_pair_id,
  scheduled_at, duration_minutes,
  guest_count, total_price,
  status (requested|confirmed|in_progress|completed|cancelled|noshow),
  check_in_at, check_out_at,
  created_at
)

-- 리뷰
reviews (
  id, booking_id, reviewer_id (guest|mate),
  rating, content, photos (jsonb),
  created_at
)
```

---

## 9. 다음 결정 필요

1. **번역 파이프라인**: OpenAI GPT-4 vs DeepL vs 자체 학습 모델?
2. **사진 저장소**: Supabase Storage vs AWS S3 vs Cloudflare R2?
3. **예약 결제**: Phase 1은 "문의→계좌이체" vs 바로 PG (토스·포트원)?
4. **검수 체제**: 풀타임 검수자 몇 명부터 시작? 초기엔 대표가 직접?
5. **상품 수정 후 재검수**: 전면 재검수 vs 차분만 재검수?
6. **외국인환자유치의료기관 미등록 클리닉**: 자격 확보 전까지 노출 보류?

---

*버전: v1.0 / 작성: 2026-04-14*
