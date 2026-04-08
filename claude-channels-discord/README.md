# 오픈클로(OpenClaw) 대안: Claude Code Channels 세팅 가이드

> **작성일:** 2026년 4월 8일  
> **배경:** 2026년 4월 4일부터 Anthropic이 Claude 구독의 오픈클로(OpenClaw) 등 서드파티 도구 사용을 차단함에 따라, 공식 대안인 Claude Code Channels로의 전환 가이드를 정리합니다.

---

## 1. Claude Code Channels란?

Claude Code Channels는 2026년 3월 20일 Anthropic이 출시한 기능으로, 텔레그램이나 디스코드를 통해 실행 중인 Claude Code 세션에 메시지를 보내고 응답을 받을 수 있는 기능입니다.

**핵심 구조:**
- PC에서 Claude Code를 `--channels` 플래그로 실행
- MCP(Model Context Protocol) 플러그인이 메시징 앱과 Claude Code 세션 사이의 양방향 브리지 역할
- 폰에서 디스코드/텔레그램으로 작업 지시 → Claude Code가 로컬에서 실행 → 결과를 같은 채널로 반환

**오픈클로와의 주요 차이:**

| 항목 | 오픈클로 | Claude Code Channels |
|------|---------|---------------------|
| 지원 플랫폼 | WhatsApp, Telegram, Slack, Discord 등 거의 전부 | Telegram, Discord, iMessage (2026.03 기준) |
| 모델 | 모델 무관 (Claude, GPT, 로컬 등) | Claude 전용 |
| 보안 | 취약점 다수 발견 (CVE-2026-25253 등) | Anthropic 공식 관리, 3계층 보안 |
| 세팅 복잡도 | 자가 호스팅, 복잡한 설정 | 플러그인 설치 후 5~10분 세팅 |
| 범위 | 코딩 + 생활 자동화 | 코딩 중심 |
| 비용 | 무료 (API 비용 별도) | Claude Pro $20/월 이상 필요 |

---

## 2. 사전 준비

### 2.1 시스템 요구사항

- **OS:** macOS, Linux (Ubuntu 등), Windows (WSL 필요)
- **Claude 계정:** Pro ($20/월) 이상 유료 구독 필수 (무료 플랜 불가)
- **Bun 런타임:** 채널 플러그인 실행에 필수 (Node.js로는 작동 안 함)

### 2.2 Bun 설치

```bash
# unzip이 없으면 먼저 설치 (Ubuntu)
sudo apt install unzip -y

# Bun 설치
curl -fsSL https://bun.sh/install | bash

# 환경변수 적용
source ~/.bashrc   # bash
source ~/.zshrc    # zsh (맥 기본)

# 설치 확인
bun --version
```

---

## 3. Claude Code 설치 및 인증

### 3.1 설치

```bash
# Native 설치 (권장)
# macOS / Linux
curl -fsSL https://claude.ai/install.sh | bash

# Windows (PowerShell)
irm https://claude.ai/install.ps1 | iex

# 설치 확인
claude --version
```

### 3.2 인증 (첫 실행)

```bash
claude
```

1. Login Method 선택 → **Claude.ai account** 선택
2. 브라우저가 자동으로 열림 (안 열리면 `c` 키로 URL 복사)
3. claude.ai에서 Pro/Max 계정으로 로그인
4. **Authorize** 버튼 클릭
5. 터미널로 돌아와서 Enter
6. 안내 문구 확인 후 Enter
7. Claude Code 프롬프트(`>`)가 뜨면 성공

**인증 확인:**
```
/status
```
`login method: Claude Pro Account` 또는 `Claude Max Account`가 표시되면 정상.

---

## 4. 디스코드 채널 연결 (단일 채널)

### 4.1 플러그인 설치

Claude Code 세션 안에서:

```
/plugin marketplace add anthropics/claude-plugins-official
/plugin marketplace update claude-plugins-official
/plugin install discord@claude-plugins-official
/reload-plugins
```

> **참고:** `plugin:discord@...`로 안 되면 `discord@claude-plugins-official`로 시도

### 4.2 디스코드 봇 생성

1. https://discord.com/developers/applications 접속
2. **New Application** → 이름 지정 (예: `Claude-Agent`)
3. 왼쪽 **Bot** 메뉴 → **Reset Token** → 토큰 복사
4. 같은 페이지에서 **Message Content Intent** 반드시 활성화
5. 왼쪽 **OAuth2 → URL Generator** → Scopes에서 `bot` 체크 → 권한 선택
6. 생성된 URL로 접속하여 봇을 본인 서버에 초대

> **기존 오픈클로용 봇이 있다면:** 해당 봇의 Reset Token으로 새 토큰만 발급받아 사용 가능

### 4.3 봇 토큰 등록

Claude Code 세션 안에서:

```
/discord:configure 복사한_봇_토큰
```

> `Unknown skill: discord:configure` 에러 시: `/exit` 후 `claude`로 재진입하면 해결

### 4.4 채널 모드로 실행

```bash
# 세션 나가기
/exit

# 채널 모드로 재실행 (봇이 온라인으로 전환됨)
claude --channels plugin:discord@claude-plugins-official
```

### 4.5 페어링

1. 디스코드에서 봇에게 **DM(개인 메시지)** 전송 (서버 채널 아님)
2. 봇이 6자리 페어링 코드 응답
3. Claude Code 터미널에서:

```
/discord:access pair 받은코드
```

### 4.6 서버 채널 등록

DM 페어링 완료 후, 서버 채널에서도 사용하려면:

1. 디스코드 **설정 → 고급 → 개발자 모드** 활성화
2. 원하는 채널 우클릭 → **채널 ID 복사**
3. Claude Code에서:

```
/discord:access channel add 복사한_채널_ID
```

### 4.7 보안 잠금

```
/discord:access policy allowlist
```

이 설정을 안 하면 아무나 봇에게 메시지를 보내 페어링 코드를 받을 수 있으므로 반드시 실행.

---

## 5. 다중 채널에 개별 세션 연결하기

채널별로 독립된 Claude Code 세션을 연결하려면 봇을 각각 만들어야 합니다.

### 5.1 채널별 봇 생성

디스코드 개발자 포털에서 채널 수만큼 봇 생성 (예: `개발-bot`, `문서-bot`)

### 5.2 봇별 상태 디렉토리 분리

```bash
# 개발 채널용
mkdir -p ~/.claude/channels/discord-dev
echo "DISCORD_BOT_TOKEN=개발봇_토큰" > ~/.claude/channels/discord-dev/.env

# 문서 채널용
mkdir -p ~/.claude/channels/discord-docs
echo "DISCORD_BOT_TOKEN=문서봇_토큰" > ~/.claude/channels/discord-docs/.env
```

### 5.3 각각 별도 tmux 세션에서 실행

```bash
# 개발 채널
tmux new -s claude-dev
DISCORD_STATE_DIR=~/.claude/channels/discord-dev claude --channels plugin:discord@claude-plugins-official

# 문서 채널 (Ctrl+B → D로 위 세션 분리 후)
tmux new -s claude-docs
DISCORD_STATE_DIR=~/.claude/channels/discord-docs claude --channels plugin:discord@claude-plugins-official
```

### 5.4 각 세션마다 페어링 + 채널 등록

```
/discord:access pair 코드
/discord:access channel add 해당_채널_ID
/discord:access policy allowlist
```

---

## 6. 상시 운영 설정

### 6.1 tmux로 세션 유지

```bash
# 새 tmux 세션 생성
tmux new -s claude

# Claude Code 채널 모드 실행
claude --channels plugin:discord@claude-plugins-official

# 세션 분리 (백그라운드 유지): Ctrl+B → D
# 세션 복귀: tmux attach -t claude
# 세션 목록: tmux ls
```

### 6.2 편의용 alias 등록

```bash
# ~/.bashrc 또는 ~/.zshrc에 추가
echo "alias clauded='claude --channels plugin:discord@claude-plugins-official'" >> ~/.bashrc
source ~/.bashrc
```

이후 `clauded`만 입력하면 채널 모드로 바로 실행.

---

## 7. 알려진 제한사항

1. **권한 프롬프트:** Claude가 파일 삭제나 셸 명령 등 위험한 작업 시 터미널에서 직접 승인해야 함. 디스코드에서 원격 승인 불가.
2. **세션 의존:** PC가 꺼지거나 Claude Code 세션이 닫히면 봇도 오프라인. 항시 운영하려면 tmux + 상시 가동 PC 필요.
3. **플랫폼 제한:** 2026년 3월 기준 Telegram, Discord, iMessage만 지원. Slack, WhatsApp은 미지원.
4. **Research Preview:** `--channels` 플래그 문법과 알림 프로토콜이 2026년 Q2~Q3에 변경될 수 있음.
5. **무인 자동화 시:** `--dangerously-skip-permissions` 플래그 사용 가능하나, 모든 권한 체크를 우회하므로 샌드박스 환경에서만 사용 권장.

---

## 8. 오픈클로 정리 (삭제 가이드)

오픈클로를 더 이상 사용하지 않는다면 완전히 제거하는 것이 보안상 필수입니다.

### 8.1 서비스 중지 및 삭제

```bash
# CLI가 있는 경우
openclaw gateway stop
openclaw uninstall --all --yes --non-interactive

# CLI가 없는 경우 (macOS)
launchctl bootout gui/$UID/ai.openclaw.gateway
rm -f ~/Library/LaunchAgents/ai.openclaw.gateway.plist
rm -f ~/Library/LaunchAgents/com.openclaw.gateway.plist
rm -f ~/Library/LaunchAgents/com.clawdbot.gateway.plist

# CLI가 없는 경우 (Linux)
systemctl --user disable --now openclaw-gateway.service
rm -f ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
```

### 8.2 잔여 파일 삭제

```bash
rm -rf ~/.openclaw ~/.clawdbot ~/.moltbot ~/.molthub ~/clawd
rm -rf /Applications/OpenClaw.app  # macOS
npm rm -g openclaw
```

### 8.3 OAuth 토큰 해지 (필수)

로컬 삭제만으로는 부족합니다. 연동했던 서비스에서 직접 권한을 해지해야 합니다:

- **Google:** myaccount.google.com → 보안 → 서드파티 앱 액세스 → OpenClaw/ClawdBot 제거
- **GitHub:** Settings → Applications → Authorized OAuth Apps → 해지
- **Slack:** 워크스페이스 설정 → 앱 관리 → 제거
- **Notion:** Settings → My Connections → 연결 해제
- **Discord:** 설정 → 승인된 앱 → 해지

---

## 9. 트러블슈팅

| 문제 | 해결 |
|------|------|
| `plugin not found in any marketplace` | `/plugin marketplace update claude-plugins-official` 실행 후 재시도 |
| `Unknown skill: discord:configure` | `/exit` 후 `claude`로 재진입하면 플러그인 로드됨 |
| 봇이 오프라인 | `--channels` 플래그 없이 실행한 경우. `/exit` 후 `claude --channels plugin:discord@claude-plugins-official`로 재실행 |
| DM에 봇 응답 없음 | Claude Code가 `--channels` 모드로 실행 중인지 확인. 터미널에 에러 메시지 확인 |
| `unzip is required` (Bun 설치 시) | `sudo apt install unzip -y` 실행 후 재시도 |
| 페어링 후 서버 채널에서 응답 없음 | `/discord:access channel add 채널ID`로 채널 등록 필요 |

---

## 참고 자료

- Claude Code 공식 문서: https://code.claude.com/docs
- Claude Code Channels 문서: https://code.claude.com/docs/en/channels
- Discord 플러그인 소스: https://github.com/anthropics/claude-plugins-official
- Anthropic 공식 채널 안내: https://code.claude.com/docs/en/channels-reference
