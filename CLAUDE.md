@app/CLAUDE.md

## Role Division (역할 분담) ⭐ 최우선

**설계·구현 작업 환경이 두 개 — 각자 담당 명확히 구분.**

### 🖥️ Claude Desktop (filesystem MCP) — 기획 전담

- 전략·사업 설계 (CEO Review, 페르소나, GTM)
- 페이지 UX 설계 (`docs/*_PAGE_DESIGN.md`, `SEARCH_PAGE_DESIGN`, `AI_PLANNER_PAGE_DESIGN` 등)
- 정책 문서 (`ADMIN_CONFIGURABLE_PARAMS`, `AI_COST_MODEL`, `AI_ROLE_DEFINITION`, `AI_TERMINOLOGY`)
- 마케팅·법무 (`LEGAL_CHECKLIST`, 마케팅 플랜)
- 세션 로그·문서 정리 (`logs/*.md`, 문서 동기화)
- 커밋은 "docs:" 프리픽으로 (기획 문서 작업만)

### 💻 Claude Code CLI — 코딩 전담

- 실제 코드 작성·수정 (`app/src/**/*.tsx`, `*.ts`, `*.js`)
- 빌드·배포·DNS (Vercel 설정, 환경변수)
- 기술 문서 — 구현 관점 (`ARCHITECTURE.md`, `README.md`, `API.md`)
- 마이그레이션·DB 스키마 (`supabase/schema.sql`, seed scripts)
- 디버깅·로그 분석·성능 측정
- 커밋은 "feat:", "fix:", "refactor:", "chore:" 프리픽으로

### 💭 제3자: 정스파크 (Genspark) — 조언자 전용

**역할**: 소스코드·문서 읽기 + 조언·리뷰·스크립트 초안 제공

**권한**:
- ✅ Read 전용 (GitHub public 별경 접근 or 필요 시 선별적 공유)
- ✅ 조언·리뷰·제안·디버깅 힌트·아키텍처 의견
- ✅ 코드 스니펙·SQL·설계 문서 초안 재시
- ❌ 실제 파일 수정·생성·삭제·커밋 불가
- ❌ Git push·pull·배포 불가
- ❌ DB 실제 변경·마이그레이션 실행 불가

**협업 플로우**:
```
정스파크 조언
      ↓
사용자 판단 (채택·수정·거부)
      ↓
Claude Desktop 또는 CLI가 실제 수행
```

**전달 방식**:
- 사용자가 정스파크 조언을 Claude에게 복사·요약 전달
- 또는 정스파크 출력을 `docs/advisors/genspark_review_YYYYMMDD.md` 로 저장 (선택적)
- Claude는 조언 품질 검토 후 적용 또는 사유 밝히고 거부

**중요**: 정스파크 조언이 기존 결정과 충돌하면 사용자 확인 후 적용. 자율 수용 금지.

### 🤝 공통 규칙

- **세션 시작 시**: Desktop·CLI 모두 `logs/*.md` 최신 파일 먼저 읽기
- **각자 담당 파일만 커밋**: 섞지 않음 (Desktop이 코드 안 만지고, CLI가 기획 문서 안 쓰기)
- **경계 모호한 경우**: 사용자에게 묻고 결정
- **두 환경 동시 작업 금지**: Git 충돌 방지 (작업 끝나면 push, 다른 에서 pull)
- **서로 산출물 존중**: Desktop이 만든 맥락을 CLI가 덮어쓰지 않기, 반대도
- **세션 종료 시 push 필수**: 다른 환경에서 이어가려면 작업 끝내기 전 `git push`

---

## 문서 작성 원칙 ⭐ 필독

**문서 작성 전 반드시 대화로 확정 먼저.**

문서화는 시간이 많이 걸린다. 잘못된 방향으로 작성하면 수정 비용이 크다.
따라서 아래 순서를 반드시 지킨다:

1. **대화로 내용 확정**: 사용자와 충분히 논의해서 방향·내용·범위를 먼저 합의
2. **확정 후 문서 작성**: 합의된 내용만 문서화. 미결 사항은 문서에 "미결" 표시
3. **문서 작성 후 즉시 커밋**: 완성된 문서는 바로 커밋·푸시

**금지 행동:**
- 사용자 확인 없이 혼자 판단해서 긴 문서 작성
- 대화 중 나온 아이디어를 미확정 상태로 문서화
- 문서 작성 도중 방향 전환 (확정 후 작성 시작)

**예외:**
- 사용자가 "바로 써라", "만들어라"고 명시한 경우 → 즉시 작성 가능
- 단순 수정·업데이트 (기존 문서에 항목 추가) → 확인 생략 가능

---

## Design System
Always read DESIGN.md before making any visual or UI decisions.
All font choices, colors, spacing, and aesthetic direction are defined there.
Do not deviate without explicit user approval.
In QA mode, flag any code that doesn't match DESIGN.md.

## Skill routing

When the user's request matches an available skill, ALWAYS invoke it using the Skill
tool as your FIRST action. Do NOT answer directly, do NOT use other tools first.
The skill has specialized workflows that produce better results than ad-hoc answers.

Key routing rules:
- Product ideas, "is this worth building", brainstorming → invoke office-hours
- Bugs, errors, "why is this broken", 500 errors → invoke investigate
- Ship, deploy, push, create PR → invoke ship
- QA, test the site, find bugs → invoke qa
- Code review, check my diff → invoke review
- Update docs after shipping → invoke document-release
- Weekly retro → invoke retro
- Design system, brand → invoke design-consultation
- Visual audit, design polish → invoke design-review
- Architecture review → invoke plan-eng-review
- Save progress, checkpoint, resume → invoke checkpoint
- Code quality, health check → invoke health
