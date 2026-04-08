# Claude Code 플러그인 가이드

> **작성일:** 2026년 4월 8일  
> **환경:** Claude Code v2.1.92+, Claude Pro/Max 구독

---

## 1. 플러그인 시스템 개요

Claude Code 플러그인은 skills, hooks, MCP 서버 등을 패키징한 확장 기능입니다.

### 작동 방식

| 구성 요소 | 작동 | 설명 |
|-----------|------|------|
| **Skills** | 자동 + 수동 | Claude가 작업에 맞으면 자동 사용, `/skill명`으로 수동 호출도 가능 |
| **Hooks** | 자동 | 세션 시작/종료, 도구 사용 등 이벤트에 자동 트리거 |
| **MCP 서버** | 자동 | Claude가 필요할 때 자동으로 도구 사용 |
| **Commands** | 수동 | `/command명`으로 직접 호출 |

### 설치 범위

- **User scope** (기본): `~/.claude/plugins/` — 모든 프로젝트에서 사용
- **Project scope**: `.claude/plugins/` — 해당 프로젝트에서만 사용

### 기본 설치 명령어

```bash
# 마켓플레이스 등록 (한 번만)
/plugin marketplace add <소유자/레포>

# 플러그인 설치
/plugin install <플러그인명>@<마켓플레이스>

# 설치된 플러그인 확인
/plugin list
```

---

## 2. 추천 플러그인

### 2.1 Superpowers

> 코드 품질 자동 관리 — TDD, 디버깅, 코드 리뷰 워크플로우

**소스:** https://github.com/obra/superpowers  
**작동 방식:** 자동 + 수동 혼합  
**추천 대상:** 모든 개발자

**설치:**

```bash
# Claude Code 세션 안에서
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

**주요 기능:**

- **자동 TDD 워크플로우**: 코드 작성 시 테스트 우선 개발을 자동 유도
- **체계적 디버깅**: 에러 발생 시 근본 원인 분석 프로세스 자동 적용
- **코드 리뷰**: 구현 완료 후 자동 리뷰 수행
- **서브에이전트 개발**: 큰 작업을 서브에이전트에 분배하여 병렬 처리
- **Git 워크트리 활용**: 독립된 환경에서 병렬 작업

**슬래시 커맨드:**

| 커맨드 | 설명 |
|--------|------|
| `/brainstorm` | 아이디어 브레인스토밍 세션 시작 |
| `/write-plan` | 구현 계획 작성 |
| `/execute-plan` | 작성된 계획 실행 |

**특징:**
- 세션 시작 시 자동으로 컨텍스트 주입 (SessionStart hook)
- 20+ 검증된 skills 포함
- Anthropic 공식 마켓플레이스 등록

---

### 2.2 claude-mem

> 무한 메모리 — 모든 작업을 자동 기록하고 다음 세션에서 활용

**소스:** https://github.com/thedotmack/claude-mem  
**작동 방식:** 완전 자동 (hooks 기반)  
**추천 대상:** 장기 프로젝트, 세션이 자주 끊기는 환경

**설치:**

```bash
npx claude-mem install
```

설치 후 별도 설정 불필요. 다음 Claude Code 세션부터 자동 활성화.

**작동 원리:**

```
사용자 프롬프트 → claude-mem이 관련 과거 작업 자동 주입
↓
Claude가 도구 사용 → PostToolUse hook이 자동 기록
↓
세션 종료 → AI가 세션 요약 생성 및 저장
↓
다음 세션 → 관련 메모리 자동 검색 및 주입
```

**5개 라이프사이클 hooks:**

| Hook | 시점 | 동작 |
|------|------|------|
| SessionStart | 세션 시작 | 워커 시작, 관련 메모리 주입 |
| UserPromptSubmit | 프롬프트 제출 | 세션 기록, 관련 과거 작업 검색 |
| PostToolUse | 도구 사용 후 | 관찰 기록 (비동기, 논블로킹) |
| Stop | 세션 중지 | AI 세션 요약 생성 |
| SessionEnd | 세션 종료 | 세션 완료 처리 |

**내장 메모리(MEMORY.md)와의 차이:**

| 항목 | claude-mem | 내장 MEMORY.md |
|------|-----------|---------------|
| 캡처 방식 | 자동 (모든 도구 사용) | 수동 (직접 작성) |
| 검색 방식 | 시맨틱 벡터 검색 | 파일시스템 패턴 매칭 |
| 범위 | 모든 프로젝트 통합 | 프로젝트별 분리 |
| 토큰 효율 | ~10배 (3단계 압축) | 전체 파일 로드 |

**메모리 대시보드:** http://localhost:37777 (로컬 전용)

**설정 파일:** `~/.claude-mem/settings.json`

---

### 2.3 gstack

> Y Combinator 대표 Garry Tan의 개발 워크플로우 — 역할 기반 33개 커맨드

**소스:** https://github.com/garrytan/gstack  
**작동 방식:** 수동 (슬래시 커맨드)  
**추천 대상:** 대규모 프로젝트, 체계적 개발 프로세스 필요 시

**설치:**

```bash
git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup
```

> bun 런타임 필요. Chromium headless 브라우저도 함께 설치됨.

**주요 커맨드:**

| 커맨드 | 역할 | 설명 |
|--------|------|------|
| `/office-hours` | CEO | 프로젝트 전략 및 방향 논의 |
| `/plan-ceo-review` | CEO | 계획 최종 검토 |
| `/design-consultation` | 디자이너 | UI/UX 설계 상담 |
| `/design-review` | 디자이너 | 디자인 리뷰 |
| `/autoplan` | EM | 자동 구현 계획 생성 |
| `/review` | EM | 코드 리뷰 |
| `/qa` | QA | 품질 검증 |
| `/ship` | Release | 배포 프로세스 |
| `/document-release` | Docs | 릴리즈 문서 작성 |
| `/browse` | 도구 | 웹 브라우징 |

**특징:**
- Conductor 기능으로 여러 Claude 세션 병렬 실행 (git worktree 활용)
- 브라우저 내장 (웹 검색, 스크린샷)
- MIT 라이선스

---

### 2.4 Code Review (내장)

> Claude Code에 기본 탑재된 코드 리뷰 기능

**설치:** 불필요 (내장)

**사용법:**

```bash
/review
```

또는 대화 중 "이 코드 리뷰해줘"라고 요청하면 자동 적용.

---

### 2.5 Security Review (내장)

> Claude Code에 기본 탑재된 보안 검토 기능

**설치:** 불필요 (내장)

**사용법:**

```bash
/security-review
```

OWASP Top 10 기반 취약점 검사, 인증/인가 로직 검토 등 수행.

---

### 2.6 Frontend Design

> UI/디자인 관련 플러그인 (상세 미확인)

**소스:** claude.com/plugins/frontend-design  
**상태:** 공식 마켓플레이스에서 확인 필요

**설치 (예상):**

```bash
/plugin install frontend-design@claude-plugins-official
```

> 2026년 4월 기준 상세 기능 및 문서가 충분히 공개되지 않음. 공식 마켓플레이스에서 직접 확인 권장.

---

## 3. 플러그인 비교표

| 플러그인 | 작동 | 용도 | 설치 난이도 | 추천 환경 |
|----------|------|------|------------|----------|
| Superpowers | 자동+수동 | TDD, 코드 품질 | 쉬움 | 모든 프로젝트 |
| claude-mem | 자동 | 세션 간 메모리 | 쉬움 | 장기 프로젝트 |
| gstack | 수동 | 체계적 개발 프로세스 | 보통 | 대규모 프로젝트 |
| Code Review | 자동+수동 | 코드 리뷰 | 없음 (내장) | 모든 프로젝트 |
| Security Review | 자동+수동 | 보안 검토 | 없음 (내장) | 모든 프로젝트 |

---

## 4. 트러블슈팅

| 문제 | 해결 |
|------|------|
| `/plugin` 명령어 미인식 | `/exit` 후 `claude`로 재진입 |
| 플러그인 설치 후 적용 안 됨 | 세션 재시작 필요 (플러그인은 세션 시작 시 로드) |
| claude-mem 대시보드 접속 안 됨 | `npx claude-mem start`로 워커 수동 시작 |
| gstack setup 실패 | `bun --version` 확인, bun 미설치 시 `curl -fsSL https://bun.sh/install \| bash` |
| superpowers skills 미로드 | 마켓플레이스 재등록: `/plugin marketplace add obra/superpowers-marketplace` |
| 플러그인 충돌 | `~/.claude/plugins/installed_plugins.json`에서 문제 플러그인 제거 후 재설치 |

---

## 참고 자료

- Claude Code 플러그인 문서: https://code.claude.com/docs/en/plugins
- Skills 문서: https://code.claude.com/docs/en/skills
- Hooks 문서: https://code.claude.com/docs/en/hooks
- Superpowers: https://github.com/obra/superpowers
- claude-mem: https://github.com/thedotmack/claude-mem
- gstack: https://github.com/garrytan/gstack
