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

### 가상환경에 설치 (권장)

```bash
# conda 환경 예시
conda create -n graphify python=3.11 -y
conda activate graphify
pip install graphifyy

# 또는 기존 환경에 추가
conda activate my-env
pip install graphifyy
```

> **주의**: PyPI 패키지명은 `graphifyy` (y 두 개). `graphify`가 아님.  
> **주의**: base 환경이 아닌 **별도 가상환경에 설치** 권장. base 오염 방지.

### pipx 로 설치 (대안)

```bash
pipx install graphifyy
```

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

### 3.2 폴더 단위 분석 (권장)

프로젝트 전체보다 **모듈/폴더별로 따로 분석**하면 품질이 훨씬 좋음.

```bash
# 전체 (노이즈 많음)
graphify update .

# 특정 폴더만 (깔끔한 결과)
graphify update ./backend
graphify update ./src/agents
```

**왜?** 전체를 돌리면 관련 없는 코드가 섞여서:
- 커뮤니티 수 폭발 (147개 중 100+개가 빈 노드)
- 내장 함수가 God Node 로 잡힘
- INFERRED 비율 증가

실전 사례: 프로젝트 전체 → 3,868 노드, 147 커뮤니티 (노이즈 심함)  
관련 폴더만 → 955 노드, 49 커뮤니티 (핵심 구조 명확)

### 3.3 결과물 설명

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

### 3.4 코드 변경 후 업데이트

```bash
graphify update .
```

변경된 파일만 재파싱해서 그래프 갱신. 전체 재분석 아님 → 빠름.

### 3.5 실시간 감시 (선택)

```bash
graphify watch .
```

파일 저장할 때마다 자동으로 그래프 업데이트.

### 3.6 그래프 쿼리

```bash
# 질문으로 관련 노드 탐색
graphify query "데이터베이스 연결은 어디서 하나?"

# 두 노드 사이 최단 경로
graphify path "FunctionA" "FunctionB"

# 특정 노드 설명
graphify explain "ClassName"
```

---

## 4. .graphifyignore — 분석 제외 설정

프로젝트 루트에 `.graphifyignore` 파일을 만들면 특정 파일/폴더를 분석에서 제외 가능. `.gitignore` 와 같은 문법.

### 예시

```
# 패키지/빌드 산출물
node_modules/
__pycache__/
.venv/
dist/
build/

# 노이즈 파일
__init__.py
*.test.py
*.spec.js

# 관련 없는 하위 프로젝트
_archive/
old_code/
```

### 효과

실전 사례 (AI 에이전트 프로젝트):
- `.graphifyignore` 없이: 438 파일, 3,868 노드, 147 커뮤니티 → 노이즈 심함
- `__init__.py` + `node_modules` 제외 후: 99 파일, 955 노드, 49 커뮤니티 → 핵심 구조 명확

### 제한사항

`.graphifyignore` 는 **파일/폴더 단위만** 제외 가능.  
`get()`, `list()` 같은 **특정 함수명/메서드명** 제외는 안 됨 → 아래 정제 스크립트로 해결.

---

## 5. 내장 함수 노이즈 제거 (정제 스크립트)

### 문제

Tree-sitter가 Python 내장 함수 호출도 엣지로 잡아서 **노이즈 God Node** 가 발생함.

실전 사례:
- `.get()` → 116 edges (모든 dict 접근이 여기로 연결)
- `.clear()` → 10 edges
- `Round` → 34 edges
- `list()`, `len()`, `str()` 등도 마찬가지

이런 노드는 프로젝트의 "핵심 추상화"가 아니라 **언어 수준의 유틸리티**. 제거해야 God Nodes 가 의미 있어짐.

### 해결: 통합 정제 스크립트

#### Windows (.bat)

```bat
@echo off
REM graphify-clean.bat — 분석 + 노이즈 제거 + 리포트 재생성
REM 프로젝트 루트에서 실행. conda 환경에 graphifyy 설치 필요.

echo === [1/4] conda 환경 활성화 ===
call conda activate agent

echo === [2/4] Graphify 코드 분석 ===
graphify update .
if errorlevel 1 (
    echo ERROR: graphify update 실패
    exit /b 1
)

echo === [3/4] 내장함수 노이즈 제거 ===
python -c "import json; BUILTINS={'get','set','list','dict','str','int','float','bool','type','len','print','range','enumerate','zip','map','filter','sorted','reversed','any','all','min','max','sum','abs','round','open','super','iter','next','hash','id','repr','format','input','vars','dir','isinstance','getattr','setattr','hasattr','delattr','callable','staticmethod','classmethod','property','tuple','bytes','bytearray','memoryview','frozenset','chr','ord','hex','oct','bin','pow','divmod','complex','slice','object','Exception','ValueError','TypeError','KeyError','IndexError','AttributeError','RuntimeError','StopIteration','NotImplementedError','ImportError','FileNotFoundError','OSError'}; DOT={'.get()','.set()','.list()','.items()','.keys()','.values()','.append()','.update()','.pop()','.format()','.join()','.split()','.strip()','.replace()','.lower()','.upper()','.extend()','.remove()','.insert()','.copy()','.clear()','.count()','.index()','.sort()','.reverse()'}; g=json.load(open('graphify-out/graph.json',encoding='utf-8')); bn=len(g['nodes']); bl=len(g['links']); rm={n['id'] for n in g['nodes'] if n.get('label','').strip().lower() in BUILTINS or n.get('label','').strip() in BUILTINS or n.get('label','').strip() in DOT}; g['nodes']=[n for n in g['nodes'] if n['id'] not in rm]; g['links']=[e for e in g['links'] if e['source'] not in rm and e['target'] not in rm]; json.dump(g,open('graphify-out/graph.json','w',encoding='utf-8'),ensure_ascii=False); print(f'Nodes: {bn} -> {len(g[\"nodes\"])} ({bn-len(g[\"nodes\"])} removed)'); print(f'Links: {bl} -> {len(g[\"links\"])} ({bl-len(g[\"links\"])} removed)')"
if errorlevel 1 (
    echo ERROR: 노이즈 제거 실패
    exit /b 1
)

echo === [4/4] 클러스터링 + 리포트 재생성 ===
set PYTHONIOENCODING=utf-8
graphify cluster-only .

echo === 완료 ===
echo 결과: graphify-out\GRAPH_REPORT.md, graph.json, graph.html
```

#### Linux/macOS (.sh)

```bash
#!/bin/bash
# graphify-clean.sh — 분석 + 노이즈 제거 + 리포트 재생성

set -e

echo "=== [1/4] 환경 활성화 ==="
# conda 환경명을 본인 환경에 맞게 수정
eval "$(conda shell.bash hook 2>/dev/null)" && conda activate agent 2>/dev/null || true

echo "=== [2/4] Graphify 코드 분석 ==="
graphify update .

echo "=== [3/4] 내장함수 노이즈 제거 ==="
python3 -c "
import json
BUILTINS={'get','set','list','dict','str','int','float','bool','type','len','print','range','enumerate','zip','map','filter','sorted','reversed','any','all','min','max','sum','abs','round','open','super','iter','next','hash','id','repr','format','input','vars','dir','isinstance','getattr','setattr','hasattr','delattr','callable','staticmethod','classmethod','property','tuple','bytes','bytearray','memoryview','frozenset','chr','ord','hex','oct','bin','pow','divmod','complex','slice','object','Exception','ValueError','TypeError','KeyError','IndexError','AttributeError','RuntimeError','StopIteration','NotImplementedError','ImportError','FileNotFoundError','OSError'}
DOT={'.get()','.set()','.list()','.items()','.keys()','.values()','.append()','.update()','.pop()','.format()','.join()','.split()','.strip()','.replace()','.lower()','.upper()','.extend()','.remove()','.insert()','.copy()','.clear()','.count()','.index()','.sort()','.reverse()'}
g=json.load(open('graphify-out/graph.json',encoding='utf-8'))
bn=len(g['nodes']); bl=len(g['links'])
rm={n['id'] for n in g['nodes'] if n.get('label','').strip().lower() in BUILTINS or n.get('label','').strip() in BUILTINS or n.get('label','').strip() in DOT}
g['nodes']=[n for n in g['nodes'] if n['id'] not in rm]
g['links']=[e for e in g['links'] if e['source'] not in rm and e['target'] not in rm]
json.dump(g,open('graphify-out/graph.json','w',encoding='utf-8'),ensure_ascii=False)
print(f'Nodes: {bn} -> {len(g[\"nodes\"])} ({bn-len(g[\"nodes\"])} removed)')
print(f'Links: {bl} -> {len(g[\"links\"])} ({bl-len(g[\"links\"])} removed)')
"

echo "=== [4/4] 클러스터링 + 리포트 재생성 ==="
graphify cluster-only .

echo "=== 완료 ==="
```

### 사용법

```bash
# Windows
graphify-clean.bat

# Linux/macOS
chmod +x graphify-clean.sh
./graphify-clean.sh
```

### 정제 전후 비교 (실전)

정제 전 God Nodes:
1. `.get()` — 116 edges ← **노이즈**
2. `AnomalyResult` — 82 edges
3. `Round` — 34 edges ← **노이즈**

정제 후 God Nodes:
1. `AnomalyResult` — 82 edges ← **실제 핵심 클래스**
2. `PredictionResult` — 77 edges
3. `Recommendation` — 72 edges
4. `AnomalyDetectorTool` — 65 edges
5. `AgentOrchestrator` — 47 edges

---

## 6. 결과 해석 가이드

### God Nodes

"가장 많은 연결을 가진 노드" = 프로젝트의 핵심 추상화.

- 실제 핵심 클래스가 잡혀야 정상 (예: `AgentOrchestrator`, `ProcessSimulator`)
- `get()`, `list()`, `str()` 같은 내장 함수가 잡히면 → 정제 스크립트 필요

### INFERRED 비율

- `EXTRACTED` = Tree-sitter 가 코드에서 직접 추출한 확실한 관계
- `INFERRED` = 파일 위치, 이름 유사성 등으로 추론한 관계 (신뢰도 0.5~0.8)
- INFERRED 가 50% 이상이면 → 추론 기반 연결이 많다는 뜻. 에이전트한테 넘길 때 "INFERRED 엣지는 추측이므로 확인 필요" 주의사항 추가 권장
- 폴더 단위 분석으로 범위를 좁히면 INFERRED 비율 낮아짐

### Community (커뮤니티)

- 관련 노드끼리 자동 묶은 그룹
- 실제 아키텍처 계층과 일치하면 분석 품질 좋은 것
- **빈 커뮤니티** (노드 1~2개) 가 많으면 → `.graphifyignore` 로 노이즈 파일 제거 필요
- 의미 있는 커뮤니티만 참고하고 나머지는 무시

### Surprising Connections

- "예상 못 한 연결" 목록
- 아키텍처 검토 시 의도하지 않은 의존성 발견에 유용
- `[INFERRED]` 태그가 붙으면 추측 기반이므로 실제 코드 확인 필요

---

## 7. AI 에이전트에 프로젝트 이해시키기

### 방법 A: CLAUDE.md 에 한 줄 추가 (수동, 간단)

프로젝트의 `CLAUDE.md` 파일에 아래 내용 추가:

```markdown
## 코드 구조
프로젝트 코드 구조는 `graphify-out/GRAPH_REPORT.md` 참고.
아키텍처 관련 질문 시 이 파일을 먼저 읽을 것.
주의: INFERRED 엣지는 추측이므로 확인 필요. 내장 함수 노드(get, list 등)가 있으면 무시.
```

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

## 8. 실전 예시

### 예시 1: everyone-play-safety 프로젝트

```bash
cd /home/weplay/dev/everyone-play-safety
graphify update .
```

**결과:**
- 544 nodes → 정제 후 542 nodes
- 1,028 edges → 정제 후 983 edges
- 37 → 36 communities
- God Nodes: `APIResponse`, `ApiService`, `ObsClient`, `Program` — 전부 실제 핵심 클래스

### 예시 2: AI 에이전트 프로젝트 (대규모)

```bash
cd /path/to/multi-ai-agent
# 전체 대신 관련 폴더만 분석
graphify update ./04_simulation
graphify update ./06_agent
```

**전체 vs 폴더 단위 비교:**

전체 분석: 3,868 노드, 147 커뮤니티, `.get()` 116 edges 노이즈  
`.graphifyignore` 적용: 955 노드, 49 커뮤니티  
정제 스크립트 적용: 952 노드, God Nodes 전부 실제 핵심 클래스

---

## 9. 워크플로우 요약

```
1. (최초 1회) graphifyy 설치 — 가상환경에
2. (최초 1회) .graphifyignore 작성 — __init__.py, node_modules 등 제외
3. graphify-clean.bat (또는 .sh) 실행 — 분석 + 정제 + 리포트 한 번에
4. CLAUDE.md 에 참조 추가 (또는 graphify install)
5. 에이전트가 코드 질문 받으면 → GRAPH_REPORT.md 먼저 읽고 → 필요한 파일만 선택적으로 읽음
6. 코드 변경 후 → graphify-clean.bat 재실행
7. graph.html 브라우저로 열어서 시각적 탐색 (선택)
```

---

## 10. 트러블슈팅

### `error: unknown command '.'`
- `graphify .` 가 아니라 `graphify update .` 로 실행

### `pip install graphify` 로 설치했는데 안 됨
- 패키지명이 `graphifyy` (y 두 개). `pip install graphifyy` 로 재설치

### Windows 에서 `cp949 UnicodeEncodeError`
- `graphify cluster-only .` 실행 시 em-dash 문자 출력에서 발생
- 해결: 실행 전에 `set PYTHONIOENCODING=utf-8` (통합 스크립트에 이미 포함)

### `.graphifyignore` 로 특정 함수 제외 안 됨
- `.graphifyignore` 는 파일/폴더 단위만 지원
- 함수/메서드 제거 → 정제 스크립트 사용 (위 5번 섹션)

### `--obsidian` 옵션?
- Obsidian vault 용 .md 파일 생성 기능. Obsidian 안 쓰면 필요 없음
- 기본 `graphify update .` 만으로 에이전트 연동에 충분

### 지원 언어 (25개)
Python, JavaScript, TypeScript, Go, Rust, Java, C, C++, C#, Kotlin, Scala, PHP, Swift, Ruby, Lua, Zig, PowerShell, Elixir, Objective-C, Julia, Verilog, SystemVerilog, Vue, Svelte, Dart

### Vue 컴포넌트가 고립 노드로 잡힘
- Tree-sitter의 Vue SFC(Single File Component) 파서 한계
- `.vue` 파일 안의 import 관계를 잘 못 추출함
- 프론트엔드 구조는 별도로 파악 필요

---

## 11. 참고 링크

- [Graphify GitHub](https://github.com/safishamsi/graphify)
- [Graphify 공식 사이트](https://graphify.net/)
- [PyPI 패키지](https://pypi.org/project/graphifyy/)
- [Claude Code 연동 가이드](https://graphify.net/graphify-claude-code-integration.html)
- [CLI 명령어 레퍼런스](https://graphify.net/graphify-cli-commands.html)
