# Graphify — 코드베이스 구조 그래프 생성 가이드

> **작성일:** 2026년 4월 20일  
> **환경:** Python 3.9+, Claude Code / Codex / Cursor 등 AI 코딩 어시스턴트

---

## 1. Graphify 란?

코드를 로컬에서 파싱해서 **프로젝트 전체의 구조 그래프**를 만들어주는 도구.

- 함수, 클래스, 모듈 간의 호출/의존 관계를 **트리시터(Tree-sitter) AST 파서**로 추출
- LLM API 호출 없음 → **비용 0, 오프라인 가능**
- 결과물을 AI 에이전트에 넘기면 **프로젝트를 훨씬 적은 토큰으로 이해**시킬 수 있음

### 핵심 원리

```
소스코드 → Tree-sitter(로컬 파서) → AST → Graphify(그래프 조립) → 결과물
```

- **Tree-sitter**: 25개 언어 지원 (Python, JS, TS, Go, Rust, Java, C, C++, Kotlin 등)
- **Graphify**: AST에서 노드(함수/클래스/모듈) + 엣지(호출/import 관계) 추출 → 그래프 생성 + 클러스터링

### LLM 에 왜 좋은가?

일반적으로 AI 에이전트에 코드를 이해시키려면 소스 파일 전체를 읽혀야 함 → 토큰 비용 폭발.

Graphify는 코드의 **구조 요약(목차 + 색인)**을 먼저 만들어서:
- 에이전트가 전체 파일을 읽지 않고도 "어디에 뭐가 있는지" 파악
- 필요한 부분만 선택적으로 읽음
- 토큰 사용량 대폭 절감 (최대 71.5배 절약 사례 보고)

---

## 2. 설치

```bash
# pip (권장)
pip install graphifyy

# 또는 pipx (환경 격리)
pipx install graphifyy
```

> **주의**: PyPI 패키지명은 `graphifyy` (y 두 개). `graphify`가 아님.

### 설치 확인

```bash
graphify --help
```

---

## 3. 사용법

### 3.1 프로젝트 분석 (기본)

```bash
cd /path/to/your/project
graphify update .
```

결과물이 `graphify-out/` 폴더에 생성됨:

```
graphify-out/
├── GRAPH_REPORT.md    ← 구조 요약 리포트 (에이전트가 읽는 용)
├── graph.json         ← 노드+엣지 원본 데이터
├── graph.html         ← 브라우저에서 보는 인터랙티브 시각화
└── cache/             ← 내부 캐시 (무시해도 됨)
```

### 3.2 결과물 설명

**GRAPH_REPORT.md**
- 프로젝트의 모듈, 함수, 클래스 목록
- 호출 관계, 의존성, 클러스터 분류
- AI 에이전트가 이 파일 하나만 읽으면 프로젝트 구조 파악 가능

**graph.html**
- 브라우저에서 열면 노드-엣지 네트워크 시각화
- 노드 클릭 → 해당 함수/클래스의 연결 관계 확인
- 개발자 본인이 프로젝트 구조를 시각적으로 파악할 때 유용

**graph.json**
- Graphify CLI 의 쿼리 기능에서 사용
- `graphify query "이 프로젝트에서 인증 처리는 어떻게 되나?"` 같은 질문 가능

### 3.3 코드 변경 후 업데이트

```bash
graphify update .
```

변경된 파일만 재파싱해서 그래프 갱신. 전체 재분석 아님 → 빠름.

### 3.4 실시간 감시 (선택)

```bash
graphify watch .
```

파일 저장할 때마다 자동으로 그래프 업데이트.

### 3.5 그래프 쿼리

```bash
# 질문으로 관련 노드 탐색
graphify query "데이터베이스 연결은 어디서 하나?"

# 두 노드 사이 최단 경로
graphify path "FunctionA" "FunctionB"

# 특정 노드 설명
graphify explain "ClassName"
```

---

## 4. AI 에이전트에 프로젝트 이해시키기

### 방법 A: CLAUDE.md 에 한 줄 추가 (수동, 간단)

프로젝트의 `CLAUDE.md` 파일에 아래 내용 추가:

```markdown
## 코드 구조
프로젝트 코드 구조는 `graphify-out/GRAPH_REPORT.md` 참고.
아키텍처 관련 질문 시 이 파일을 먼저 읽을 것.
```

→ 에이전트가 코드 관련 질문 받으면 GRAPH_REPORT.md 를 먼저 참고함.

### 방법 B: `graphify install` 로 자동 연동 (권장)

```bash
cd /path/to/your/project
graphify install            # Claude Code 기본
graphify install --platform codex    # Codex
graphify install --platform cursor   # Cursor
```

이 명령이 하는 일:
- CLAUDE.md (또는 해당 플랫폼 설정 파일)에 Graphify 참조 섹션 자동 추가
- 에이전트가 코드 검색(Glob/Grep) 하기 전에 GRAPH_REPORT.md 를 먼저 읽도록 hook 설치

### 방법 C: 수동으로 에이전트에게 지시

대화 중에 직접:
```
"graphify-out/GRAPH_REPORT.md 읽어보고 이 프로젝트 구조 파악해"
```

---

## 5. 실전 예시

### everyone-play-safety 프로젝트 분석 결과

```bash
cd /home/weplay/dev/everyone-play-safety
graphify update .
```

**결과:**
- 544 nodes (함수/클래스/모듈)
- 1,028 edges (호출/의존 관계)
- 37 communities (자동 클러스터)

에이전트에게 `GRAPH_REPORT.md` 를 넘기면:
- 전체 소스 파일을 읽히는 것 대비 토큰 90%+ 절감
- "이 프로젝트에서 카메라 관련 코드는 어디에 있어?" 같은 질문에 즉시 답변 가능

---

## 6. 워크플로우 요약

```
1. 프로젝트에서 graphify update . 실행 (1회, 이후 코드 바뀔 때마다)
2. graphify-out/ 폴더 생성됨
3. AI 에이전트가 GRAPH_REPORT.md 를 참조하도록 설정 (CLAUDE.md 또는 graphify install)
4. 에이전트가 코드 질문 받으면 → 그래프 리포트 먼저 읽고 → 필요한 파일만 선택적으로 읽음
5. 본인은 graph.html 로 시각적 탐색 (선택)
```

---

## 7. 트러블슈팅

### `error: unknown command '.'`
- `graphify .` 가 아니라 `graphify update .` 로 실행

### `pip install graphify` 로 설치했는데 안 됨
- 패키지명이 `graphifyy` (y 두 개). `pip install graphifyy` 로 재설치

### `--obsidian` 옵션?
- Obsidian vault 용 .md 파일 생성 기능. Obsidian 안 쓰면 필요 없음
- 기본 `graphify update .` 만으로 에이전트 연동에 충분

### 지원 언어 확인
- Python, JavaScript, TypeScript, Go, Rust, Java, C, C++, C#, Kotlin, Scala, PHP, Swift, Ruby, Lua, Zig, PowerShell, Elixir, Objective-C, Julia, Verilog, SystemVerilog, Vue, Svelte, Dart (25개)

---

## 8. 참고 링크

- [Graphify GitHub](https://github.com/safishamsi/graphify)
- [Graphify 공식 사이트](https://graphify.net/)
- [PyPI 패키지](https://pypi.org/project/graphifyy/)
- [Claude Code 연동 가이드](https://graphify.net/graphify-claude-code-integration.html)
- [CLI 명령어 레퍼런스](https://graphify.net/graphify-cli-commands.html)
