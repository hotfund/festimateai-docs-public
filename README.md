# FestiMate New

> FestiMate 플랫폼 재설계 작업 공간.
> 기존 코드(`github.com/hotfund/festimate`)와 분리하여 새로 설계·개발 중.

---

## 📁 구조

```
C:\FestimateNew\
├── README.md                  ← 이 파일
├── .gitignore
├── docs\                      설계 문서 SSOT
│   ├── FESTIMATE_PLAYBOOK.md     사업 설계 통합 기록
│   ├── SITEMAP_SPEC.md           페이지 구조·URL·네비게이션
│   ├── MEMBERSHIP_TIERS.md       회원·메이트·공급자 등급 체계
│   ├── PLAN_TEMPLATE_SCHEMA.md   AI/Mate 공용 여행 플랜 JSON
│   ├── SUPPLIER_UPLOAD_SPEC.md   공급자 업로드 폼 명세
│   ├── ACCESS_CONTROL.md         접근 권한·결제·정책
│   ├── CATEGORY_MAPPING.md       카테고리 마스터 매핑
│   ├── GLOSSARY.md               용어 사전
│   ├── CONSISTENCY_AUDIT.md      정합성 감사
│   ├── RELEASE_PLAN.md           릴리즈 마일스톤
│   ├── LEGAL_CHECKLIST.md        법적 체크리스트
│   └── INFRA_STATUS.md           인프라 현황
├── logs\                      세션 로그 (날짜별)
└── app\                       Next.js 프로젝트 (신규 2026-04-16)
    ├── src/app/                  App Router
    ├── package.json
    └── ...
```

## 🚀 개발 환경

- **프레임워크**: Next.js 16 + TypeScript + Tailwind CSS
- **DB/인증**: Supabase (미연결)
- **호스팅**: Vercel (미연결)
- **AI**: OpenAI GPT-4o

### 로컬 실행

```bash
cd app
npm install
npm run dev      # http://localhost:3000
```

환경변수는 `app/.env.local` 에 설정 (레포 커밋 금지, `.gitignore`에 등록됨).

---

## 🖥 다른 컴퓨터에서 이어받는 법

### 1. Git 설치 (없으면)
- Windows: https://git-scm.com/download/win
- macOS: `brew install git`

### 2. Repo clone
```bash
cd C:\          # 또는 원하는 경로
git clone https://github.com/festimateai/festimateai.git FestimateNew
cd FestimateNew
```

### 3. Claude Code 실행
```bash
cd C:\FestimateNew
claude
```

### 4. 첫 메시지 제안
```
docs/FESTIMATE_PLAYBOOK.md 읽고 현재 상태 파악해줘.
이어서 작업 시작.
```

---

## 🔄 작업 동기화 (여러 PC 사용 시)

### 작업 시작 전 (pull)
```bash
cd C:\FestimateNew
git pull
```

### 작업 끝난 후 (commit + push)
```bash
cd C:\FestimateNew
git add .
git commit -m "작업 내용 요약"
git push
```

또는 Claude에게 **"지금까지 작업 커밋하고 푸시해줘"** 라고 요청.

---

## 📝 현재 작업 상태

- **2026-04-14**: 초기 브레인스토밍, 5개 설계 문서 생성
- **2026-04-15**: 정합성 감사 9건, 카테고리 SSOT·용어 사전·접근 권한 문서 추가, 정책 6건 결정·반영
- **2026-04-16**: Next.js 프로젝트 초기화, 수익 모델 재설계 논의 (미확정)

자세한 내용은 [`logs/`](logs/) 및 [`docs/FESTIMATE_PLAYBOOK.md`](docs/FESTIMATE_PLAYBOOK.md) 참조.

---

## 🏢 관련 링크

- 기존 코드: https://github.com/hotfund/festimate
- 프로덕션 사이트: https://festimate.ai
- 이 repo: https://github.com/festimateai/festimateai
