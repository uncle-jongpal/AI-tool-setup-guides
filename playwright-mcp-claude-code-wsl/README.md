# Playwright MCP × Claude Code (WSL) 셋업 가이드

> **작성일:** 2026년 5월 14일
> **환경:** Windows 11 + WSL2 (Ubuntu) + Claude Code v2.1.119 + nvm + Node 22
> **배경:** Claude Code 에이전트에게 "실제 브라우저를 띄워서 페이지 자동 테스트"를 시키려면 Playwright MCP가 필요합니다. 공식 가이드는 macOS/Linux 기준이라 WSL+nvm 환경에서 따라하면 4가지 함정에 차례로 걸려서 시간이 많이 듭니다. 이 가이드는 그 4가지를 모두 우회한 검증된 절차입니다.

---

## 1. 이 가이드로 얻는 것

- Claude Code 안에서 "이 사이트 들어가서 로그인 한번 해봐", "이 화면 캡쳐해줘" 같은 자연어 지시로 실제 브라우저 자동화
- 로컬/원격 웹앱의 회귀 테스트, 로그인 흐름 검증, 스크린샷 기반 보고를 에이전트에게 위임 가능
- 다른 에이전트(다른 PC의 Claude Code)에게 같은 환경을 5분 안에 복제 가능

## 2. 사전 준비

| 항목 | 요구 사항 |
|------|-----------|
| OS | Windows 10/11 + WSL2 (Ubuntu 권장) |
| Claude Code | v2.1.119 이상 설치 + 로그인 완료 |
| Node.js | nvm 경유 v18 이상 (이 가이드는 v22.22.0 기준) |
| 권한 | sudo 사용 가능 (Chrome 설치 단계에서만 필요) |
| 디스크 | 약 700MB (Playwright + Chromium + Chrome) |

확인 명령어:

```bash
claude --version
node --version
which npx
```

세 개 모두 정상 출력되면 다음으로 진행합니다.

## 3. 설치 단계 (이 순서 그대로)

### 3.1 Playwright MCP 패키지 글로벌 설치

```bash
npm install -g @playwright/mcp
```

설치 위치 확인 (이 경로를 뒤에서 씁니다):

```bash
which playwright-mcp
# 출력 예: /home/<USER>/.nvm/versions/node/v22.22.0/bin/playwright-mcp

# 실제 npm 모듈 위치
ls $(npm root -g)/@playwright/mcp/cli.js
```

### 3.2 Chromium 브라우저 다운로드 (Playwright용)

```bash
# 임시 폴더에서 진행 (Playwright의 install 명령은 package.json이 있는 디렉토리에서 동작)
mkdir -p /tmp/playwright-install && cd /tmp/playwright-install
npm init -y
npm install playwright
npx playwright install chromium
```

다운로드 위치는 `~/.cache/ms-playwright/chromium-*/chrome-linux64/chrome` 입니다.

### 3.3 Google Chrome 설치 (시스템 영역)

Playwright MCP는 기본적으로 시스템 Chrome을 찾으므로 별도 설치가 필요합니다 (4.3절 참고).

```bash
cd /tmp/playwright-install
npx playwright install chrome
```

설치 도중 sudo 비밀번호 입력 프롬프트가 뜨면 입력합니다. 설치 후 `/opt/google/chrome/chrome`에 약 270MB 바이너리가 생깁니다.

### 3.4 Claude Code MCP 설정 등록

```bash
claude mcp add playwright -s user -- npx @playwright/mcp@latest
```

이 명령은 `~/.claude.json`의 `mcpServers.playwright` 항목을 추가합니다. **이대로는 동작 안 합니다** — 다음 단계에서 수정해야 합니다.

### 3.5 MCP 설정을 검증된 형태로 교체

`~/.claude.json`을 열어 `mcpServers.playwright` 블록을 아래로 교체합니다:

```json
"playwright": {
  "type": "stdio",
  "command": "/home/<USER>/.nvm/versions/node/v22.22.0/bin/node",
  "args": [
    "/home/<USER>/.nvm/versions/node/v22.22.0/lib/node_modules/@playwright/mcp/cli.js"
  ],
  "env": {}
}
```

`<USER>`와 Node 버전(`v22.22.0`)은 본인 환경에 맞게 바꿉니다.

### 3.6 Claude Code 재시작

```bash
# 기존 세션 종료 후
claude --continue --channels plugin:discord@claude-plugins-official
```

`--continue`는 직전 대화 맥락을 그대로 이어받습니다. 다른 채널 옵션 안 쓰면 그냥 `claude --continue`.

## 4. 검증

### 4.1 MCP 연결 상태

```bash
claude mcp list | grep playwright
```

`playwright: ... - ✓ Connected` 가 출력되면 정상.

### 4.2 도구 노출 확인

Claude Code 세션 안에서 에이전트에게 "playwright 도구 떴는지 확인해줘" 라고 요청하면 `browser_navigate`, `browser_click`, `browser_snapshot` 등 23개 도구가 보여야 합니다.

### 4.3 실제 페이지 테스트

```
세션 안에서:
> https://example.com 접속해서 페이지 제목 확인해줘
```

페이지 제목이 보고되면 셋업 완료입니다.

## 5. 트러블슈팅 (실제로 겪은 4가지 함정)

### 5.1 함정 1 — `claude mcp list`가 "Failed to connect"

**증상:** MCP 등록은 됐는데 연결이 안 됨.

**원인:** `npx @playwright/mcp@latest`는 매번 npm 레지스트리에 `@latest` 태그를 조회하느라 콜드 스타트가 약 15초 걸립니다. Claude Code의 MCP 연결 타임아웃 안에 핸드셰이크를 못 끝내서 실패합니다.

**해결:** 글로벌 설치 후 `cli.js`를 node로 직접 호출 (3.5절 설정). 콜드 스타트가 1초 이내로 줄어듭니다.

### 5.2 함정 2 — `/usr/bin/env: 'node': No such file or directory`

**증상:** MCP 로그 파일(`~/.cache/claude-cli-nodejs/<repo>/mcp-logs-playwright/*.jsonl`)에 이 에러가 찍힘.

**원인:** `playwright-mcp` 스크립트의 shebang이 `#!/usr/bin/env node`로 되어 있습니다. Claude Code가 자식 프로세스를 띄울 때 PATH에 nvm 경로가 빠져있어서 env가 node를 못 찾습니다.

**해결:** shebang을 우회하기 위해 **node 절대경로 + cli.js 절대경로**로 호출합니다 (3.5절 설정). 이게 가장 신뢰성 높은 방식입니다.

### 5.3 함정 3 — `Chromium distribution 'chrome' is not found at /opt/google/chrome/chrome`

**증상:** MCP는 붙었는데 `browser_navigate` 호출 직후 이 에러.

**원인:** `npx playwright install chromium`은 Playwright 번들 Chromium만 깝니다. Playwright MCP의 기본 동작은 "시스템에 깔린 Google Chrome (chrome 채널)"을 찾는 것이라 둘이 다릅니다. WSL 안 리눅스에 Chrome이 없으면 (Windows에 Chrome이 있어도) 이 에러가 납니다.

**해결책 두 가지:**

방법 A — Chrome을 시스템에 설치 (3.3절). 가장 깨끗하고 표준적. sudo 필요.

방법 B — 이미 깐 Chromium 경로를 직접 지정.

```json
"args": [
  "/home/<USER>/.nvm/versions/node/v22.22.0/lib/node_modules/@playwright/mcp/cli.js",
  "--executable-path",
  "/home/<USER>/.cache/ms-playwright/chromium-1223/chrome-linux64/chrome"
]
```

이 가이드는 방법 A 기준입니다.

### 5.4 함정 4 — 스크린샷이 모바일 레이아웃으로 찍힘

**증상:** 데스크탑 사이트인데 폭이 좁고 햄버거 메뉴가 나오는 모바일 UI로 보임.

**원인:** Playwright MCP 기본 뷰포트가 약 1024픽셀 미만이라, 반응형 사이트가 모바일 레이아웃으로 전환됨.

**해결:** MCP `args`에 `--viewport-size 1920x1080`을 추가하거나, 세션 안에서 `browser_resize`로 매번 조정.

```json
"args": [
  "/home/<USER>/.nvm/versions/node/v22.22.0/lib/node_modules/@playwright/mcp/cli.js",
  "--viewport-size",
  "1920x1080"
]
```

## 6. 다른 에이전트에 넘길 때 체크리스트

새 PC에 동일 환경을 세팅한다면 이 순서로 확인:

1. `claude --version`이 v2.1.119 이상인지
2. `which npx` 결과가 nvm 경로(`~/.nvm/...`)에서 나오는지
3. `npm ls -g @playwright/mcp` 출력에 패키지가 있는지
4. `ls ~/.cache/ms-playwright/chromium-*` 폴더가 존재하는지
5. `ls /opt/google/chrome/chrome` 파일이 존재하는지
6. `~/.claude.json`의 `mcpServers.playwright.command`가 **node 절대경로**인지 (`npx` 아님)
7. `claude mcp list | grep playwright`가 `✓ Connected`인지

전부 통과하면 세션 재시작 후 바로 사용 가능합니다.

## 7. 알려진 제한사항

1. **WSL 안의 Chrome ≠ Windows의 Chrome:** 종팔님 Windows 쪽 Chrome 프로필(북마크, 로그인 상태)과 무관하게 깨끗한 새 프로필로 동작합니다.
2. **헤드 모드 + WSL:** 기본 헤드 모드는 WSLg가 켜져 있어야 창이 보입니다. 대부분의 자동화엔 어차피 헤드리스가 충분하므로 `--headless` 플래그를 추가하는 것도 고려해볼 만합니다.
3. **세션 시작 시 콜드 스타트:** Claude Code 첫 부팅 후 첫 `browser_navigate`는 Chrome 프로세스 띄우느라 1~2초 더 걸립니다. 이후 호출은 빠릅니다.
4. **인증된 세션:** 로그인 상태를 유지하고 싶으면 `--user-data-dir` 옵션으로 프로필 폴더 지정 가능. 단 비밀번호가 디스크에 남는 점 주의.

## 8. 트러블슈팅 빠른 참조

| 증상 | 1순위 확인 |
|------|-----------|
| `Failed to connect` | `~/.claude.json`이 `command: "npx"`가 아니라 node 절대경로인지 |
| `env: 'node': No such file` | shebang 우회 (node + cli.js 절대경로 조합) 적용 여부 |
| `chrome is not found` | `/opt/google/chrome/chrome` 존재 여부 또는 `--executable-path` 지정 여부 |
| 모바일 레이아웃 | `--viewport-size` args 추가 여부 |
| 도구 안 보임 | Claude Code 세션을 새로 띄웠는지 (설정 변경 후 재시작 필수) |

## 9. 참고 자료

- Playwright MCP 공식: https://github.com/microsoft/playwright-mcp
- Playwright 공식 문서: https://playwright.dev/docs/getting-started-mcp
- Claude Code MCP 문서: https://code.claude.com/docs/en/mcp
- 이 가이드의 검증 환경: Windows 11 + WSL2 Ubuntu 22.04 + nvm 0.39 + Node 22.22.0 + Claude Code 2.1.119 + @playwright/mcp 0.0.75
