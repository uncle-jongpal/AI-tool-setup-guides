# Cloudflare Zero Trust Tunnel 설정 가이드

> **example.com** 기준 | 최종 수정: 2026-04-07

---

## 목차

1. [개요](#1-개요)
2. [아키텍처](#2-아키텍처)
3. [사전 준비](#3-사전-준비)
4. [대시보드 설정 (공통)](#4-대시보드-설정-공통)
5. [서버 설정 — Linux (Ubuntu/Debian)](#5-서버-설정--linux-ubuntudebian)
6. [서버 설정 — macOS](#6-서버-설정--macos)
7. [서버 설정 — Windows](#7-서버-설정--windows)
8. [클라이언트 설정 (SSH 접속하는 쪽)](#8-클라이언트-설정-ssh-접속하는-쪽)
9. [서비스 추가 (Public Hostname)](#9-서비스-추가-public-hostname)
10. [Access 정책 설정](#10-access-정책-설정)
11. [트러블슈팅](#11-트러블슈팅)
12. [서브도메인 네이밍 컨벤션](#12-서브도메인-네이밍-컨벤션)

---

## 1. 개요

### Tunnel이란?

Cloudflare Tunnel은 서버에서 Cloudflare Edge로 **아웃바운드 연결**을 맺는 방식이다. 기존 포트포워딩 방식과 달리 서버 쪽 인바운드 포트를 열 필요가 없다.

```
서버 (cloudflared) ────아웃바운드────→ Cloudflare Edge ←──── 외부 사용자
     포트 안 열어도 됨                  SSL/인증 처리          도메인으로 접속
```

### Tunnel vs 포트포워딩

| 항목 | 포트포워딩 | Cloudflare Tunnel |
|------|-----------|-------------------|
| 인바운드 포트 | 필요 (22, 80 등) | 불필요 |
| 공인 IP | 필요 | 불필요 |
| SSL 인증서 | 직접 관리 | Cloudflare 자동 |
| 접근 제어 | 자체 구현 | Zero Trust Access |
| IP 변경 대응 | DDNS 필요 | 자동 (도메인 고정) |

### 핵심 원칙

- **서버(PC) 1대 = Tunnel 1개**
- 하나의 Tunnel 안에서 여러 서브도메인을 ingress rule로 매핑
- 서브도메인 = 서비스 구분, Tunnel = 물리 서버 구분

---

## 2. 아키텍처

### 전체 구조 (예시)

```
Cloudflare Edge (*.example.com)
        │
        ├── tunnel-db (DB PC)
        │     ├── ssh-db.example.com    → localhost:22   (SSH)
        │     ├── db.example.com        → localhost:5050 (pgAdmin)
        │     └── pms.example.com       → localhost:3000 (PMS)
        │
        ├── tunnel-web (Web PC)
        │     ├── ssh-web.example.com   → localhost:22   (SSH)
        │     ├── demo.example.com      → localhost:3000 (Demo)
        │     └── api.example.com       → localhost:8000 (API)
        │
        ├── tunnel-ws1 (워크스테이션 1)
        │     ├── ssh-ws1.example.com   → localhost:22   (SSH)
        │     └── gpu-demo.example.com  → localhost:7860 (Gradio)
        │
        └── tunnel-ws2 (워크스테이션 2)
              └── ssh-ws2.example.com   → localhost:22   (SSH)
```

### 접근 제어 레이어

```
외부 요청 → Cloudflare Edge → Access 정책 확인 → Tunnel → 서버 로컬 서비스
                                  │
                          인증 실패 시 차단
```

---

## 3. 사전 준비

### 필수 요건

- Cloudflare 계정 (무료 플랜 OK, 50 seat 제한)
- 도메인이 Cloudflare DNS에 등록되어 있어야 함
- 서버에 SSH가 활성화되어 있어야 함

### 무료 플랜 제한 사항

| 항목 | 무료 | 유료 ($7/user/mo) |
|------|------|-------------------|
| 사용자 수 | 최대 50명 | 무제한 |
| 디바이스 수 | 제한 없음 | 제한 없음 |
| 네트워크 Location | 최대 3곳 | 확장 가능 |
| 로그 보존 | 24시간 | 30일 |
| 기술 지원 | 없음 | 있음 |
| SLA (업타임 보장) | 없음 | 100% |

---

## 4. 대시보드 설정 (공통)

> 모든 OS에서 동일하게 대시보드에서 먼저 Tunnel을 생성한다.

### 4.1 Tunnel 생성

1. [Zero Trust 대시보드](https://one.dash.cloudflare.com/) 접속
2. **Networks** → **Tunnels** → **Create a tunnel**
3. Tunnel type: **Cloudflared** 선택
4. Tunnel 이름 입력 (예: `tunnel-db`, `tunnel-web`)
5. 생성 후 나오는 **토큰을 복사**해둔다
   ```
   eyJhIjoiNmRk....(긴 문자열)
   ```

### 4.2 Public Hostname 추가

Tunnel 생성 후 → **Public Hostnames** 탭 → **Add a public hostname**

#### SSH 서비스 예시

| 필드 | 값 |
|------|-----|
| Subdomain | `ssh-db` |
| Domain | `example.com` |
| Path | **(비워둠)** |
| Service Type | `SSH` |
| URL | `localhost:22` |

#### 웹 서비스 예시

| 필드 | 값 |
|------|-----|
| Subdomain | `pms` |
| Domain | `example.com` |
| Path | **(비워둠)** |
| Service Type | `HTTP` |
| URL | `localhost:3000` |

> ⚠️ **주의**: Path 필드에 값을 넣지 말 것. 특히 SSH는 path 라우팅이 불가능하다.

---

## 5. 서버 설정 — Linux (Ubuntu/Debian)

### 5.1 cloudflared 설치

```bash
# 방법 1: 패키지 매니저
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update
sudo apt install cloudflared -y

# 방법 2: 직접 다운로드
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb
```

### 5.2 서비스 등록 (토큰 방식)

```bash
# 기존 서비스가 있으면 제거
sudo cloudflared service uninstall

# 토큰으로 설치 (대시보드에서 복사한 토큰)
sudo cloudflared service install eyJhIjoiNmRk....(토큰)

# 서비스 활성화 및 시작
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

### 5.3 상태 확인

```bash
# 서비스 상태
sudo systemctl status cloudflared

# 로그 실시간 확인
sudo journalctl -u cloudflared -f
```

### 5.4 정상 연결 시 로그 예시

```
INF Registered tunnel connection connIndex=0 location=icn05 protocol=quic
INF Registered tunnel connection connIndex=1 location=icn06 protocol=quic
INF Registered tunnel connection connIndex=2 location=icn06 protocol=quic
INF Registered tunnel connection connIndex=3 location=icn01 protocol=quic
```

`connIndex` 0~3까지 4개 연결이 등록되면 정상이다.

### 5.5 서비스 관리 명령어

```bash
sudo systemctl start cloudflared     # 시작
sudo systemctl stop cloudflared      # 중지
sudo systemctl restart cloudflared   # 재시작
sudo systemctl status cloudflared    # 상태 확인
sudo cloudflared service uninstall   # 서비스 제거
```

---

## 6. 서버 설정 — macOS

### 6.1 cloudflared 설치

```bash
brew install cloudflared
```

### 6.2 서비스 등록 (토큰 방식)

```bash
# 기존 서비스가 있으면 제거
sudo cloudflared service uninstall

# 토큰으로 설치
sudo cloudflared service install eyJhIjoiNmRk....(토큰)
```

> `service install`만 하면 launchd에 등록 + 자동 시작까지 처리된다. `systemctl enable/start` 불필요.

### 6.3 상태 확인

```bash
# 서비스 목록에서 확인
sudo launchctl list | grep cloudflared

# 로그 확인
cat /Library/Logs/com.cloudflare.cloudflared.log
```

### 6.4 서비스 관리 명령어

```bash
sudo launchctl start com.cloudflare.cloudflared    # 시작
sudo launchctl stop com.cloudflare.cloudflared     # 중지
sudo cloudflared service uninstall                  # 서비스 제거
```

### 6.5 macOS SSH 서비스 활성화

macOS는 기본적으로 SSH가 꺼져있을 수 있다:

```bash
# SSH 활성화
sudo systemsetup -setremotelogin on

# 확인
sudo systemsetup -getremotelogin
```

또는 **시스템 설정** → **일반** → **공유** → **원격 로그인** 켜기

---

## 7. 서버 설정 — Windows

### 7.1 cloudflared 설치

```powershell
# winget 사용
winget install cloudflare.cloudflared

# 또는 직접 다운로드
# https://github.com/cloudflare/cloudflared/releases/latest
# cloudflared-windows-amd64.msi 설치
```

### 7.2 서비스 등록 (토큰 방식)

PowerShell을 **관리자 권한**으로 실행:

```powershell
# 기존 서비스가 있으면 제거
cloudflared service uninstall

# 토큰으로 설치
cloudflared service install eyJhIjoiNmRk....(토큰)
```

Windows 서비스로 자동 등록된다.

### 7.3 상태 확인

```powershell
# 서비스 상태
Get-Service cloudflared

# 또는 services.msc에서 "Cloudflare Tunnel Agent" 확인
```

### 7.4 서비스 관리

```powershell
Start-Service cloudflared      # 시작
Stop-Service cloudflared       # 중지
Restart-Service cloudflared    # 재시작
```

### 7.5 Windows OpenSSH 서버 활성화

SSH 접속을 받으려면 OpenSSH Server가 필요하다:

```powershell
# 설치
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# 서비스 시작 및 자동 시작 설정
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

---

## 8. 클라이언트 설정 (SSH 접속하는 쪽)

> 서버가 아닌, **접속하는 PC**의 설정이다. 상시 실행 불필요.

### 8.1 cloudflared 설치

| OS | 설치 명령 |
|----|----------|
| **Windows** | `winget install cloudflare.cloudflared` |
| **macOS** | `brew install cloudflared` |
| **Linux** | `sudo apt install cloudflared` |

### 8.2 SSH Config 설정

`~/.ssh/config` (Windows: `C:\Users\<유저명>\.ssh\config`):

```ssh-config
# ── DB 서버 ──
Host example-db
    HostName ssh-db.example.com
    User dbuser
    ProxyCommand cloudflared access ssh --hostname %h

# ── Web 서버 ──
Host example-web
    HostName ssh-web.example.com
    User webuser
    ProxyCommand cloudflared access ssh --hostname %h

# ── 워크스테이션 1 ──
Host example-ws1
    HostName ssh-ws1.example.com
    User wsuser
    ProxyCommand cloudflared access ssh --hostname %h

# ── 워크스테이션 2 ──
Host example-ws2
    HostName ssh-ws2.example.com
    User wsuser
    ProxyCommand cloudflared access ssh --hostname %h
```

### 8.3 접속 방법

```bash
# 터미널에서
ssh example-db

# Cursor/VS Code에서 Remote SSH
# Host 이름(example-db)으로 접속하면 ProxyCommand 자동 적용
```

### 8.4 브라우저 SSH (설치 없이 접속)

클라이언트에 아무것도 설치하지 않고 웹 브라우저에서 접속하는 방법:

1. 대시보드 → **Access** → **Applications** → **Add an application** → **Self-hosted**
2. Application domain: `ssh-db.example.com`
3. Policy: Allow / Emails ending in `@example.com`
4. **Additional settings** → **Browser rendering** → `SSH` 선택
5. 저장 후 브라우저에서 `https://ssh-db.example.com` 접속

### 8.5 ProxyCommand가 필요한 이유

```
일반 SSH:   클라이언트 ──TCP:22──→ 서버      (직접 연결, 포트 필요)
Tunnel SSH: 클라이언트 ──→ cloudflared(통역) ──HTTPS──→ Cloudflare ──→ 서버

SSH와 HTTPS는 프로토콜이 다르기 때문에
클라이언트 쪽 cloudflared가 SSH ↔ HTTPS 변환을 해준다.
상시 실행이 아니라 SSH 접속 시에만 잠깐 동작한다.
```

---

## 9. 서비스 추가 (Public Hostname)

### 이미 돌고 있는 Tunnel에 서비스를 추가하는 방법

서버 쪽 작업 **없음**. 대시보드에서만 하면 된다.

1. Zero Trust 대시보드 → **Networks** → **Tunnels**
2. 해당 Tunnel 클릭 → **Public Hostnames** → **Add a public hostname**
3. 정보 입력 후 저장
4. `cloudflared` 재시작 불필요 (자동 반영)

### 서비스 타입별 설정

| 서비스 | Type | URL 예시 |
|--------|------|----------|
| SSH | `SSH` | `localhost:22` |
| 웹 앱 (HTTP) | `HTTP` | `localhost:3000` |
| 웹 앱 (HTTPS) | `HTTPS` | `localhost:443` |
| API 서버 | `HTTP` | `localhost:8000` |
| DB 관리 (pgAdmin) | `HTTP` | `localhost:5050` |
| Jupyter Notebook | `HTTP` | `localhost:8888` |
| Gradio | `HTTP` | `localhost:7860` |

---

## 10. Access 정책 설정

### 정책 유형

| 용도 | 정책 | 설명 |
|------|------|------|
| 내부 전용 | Emails ending in `@example.com` | 회사 이메일만 허용 |
| 특정인만 | Emails: `admin@example.com` | 지정 이메일만 |
| 외부 공개 (인증) | One-time PIN | 이메일 OTP 인증 |
| 완전 공개 | Bypass | 인증 없이 누구나 접근 |
| API 전용 | Service Token | 토큰 기반 인증 |

### 설정 방법

1. **Access** → **Applications** → **Add an application** → **Self-hosted**
2. Application domain 입력 (예: `ssh-db.example.com`)
3. Policy 추가:
   - Action: **Allow**
   - Include rule 설정 (이메일 도메인, 특정 이메일 등)
4. 저장

### 추천 정책 매핑

| 서브도메인 | 접근 수준 | Access 정책 |
|-----------|----------|-------------|
| `ssh-*.example.com` | 내부 전용 | @example.com |
| `admin.example.com` | 내부 전용 | @example.com |
| `db.example.com` | 최소 권한 | 특정 이메일만 |
| `demo.example.com` | 외부 공개 | OTP 또는 Bypass |
| `api.example.com` | API 전용 | Service Token |

---

## 11. 트러블슈팅

### DNS 레코드 충돌

```
Error: An A, AAAA, or CNAME record with that host already exists.
```

**해결**: Cloudflare 대시보드 → DNS → Records에서 충돌하는 레코드 삭제 후 다시 시도

### config.yml 못 찾음 (Linux)

```
Cannot determine default configuration path. No file [config.yml config.yaml]
```

**해결**: 대시보드에서 만든 Tunnel은 토큰 방식을 사용한다. config.yml이 아니라 토큰으로 서비스 등록:

```bash
sudo cloudflared service install eyJhIjoi....(토큰)
```

### cert.pem 못 찾음

```
Error locating origin cert: No file cert.pem
```

**해결**: 동일. 대시보드에서 만든 Tunnel은 CLI 인증서 불필요. 토큰 방식 사용.

### 서비스 이미 설치됨

```
cloudflared service is already installed
```

**해결**:

```bash
# Linux
sudo cloudflared service uninstall
sudo cloudflared service install eyJhIjoi....(토큰)

# macOS
sudo cloudflared service uninstall
sudo cloudflared service install eyJhIjoi....(토큰)

# Windows (관리자 PowerShell)
cloudflared service uninstall
cloudflared service install eyJhIjoi....(토큰)
```

### SSH 연결 시 Connection timed out

```
ssh: connect to host ssh-db.example.com port 22: Connection timed out
```

**원인**: ProxyCommand가 빠져있음. SSH config에 아래 추가:

```
ProxyCommand cloudflared access ssh --hostname %h
```

### 브라우저 접속 시 빈 화면

**원인**: Access Application이 생성되지 않았거나, Browser rendering이 설정되지 않음.

**해결**: Access → Applications에서 해당 도메인으로 Application 생성 → Browser rendering → SSH 선택

### 브라우저 접속 시 500 에러

**원인**: Tunnel은 연결됐지만 로컬 서비스가 응답하지 않음.

**해결**:
1. 서버에서 SSH 서비스 확인: `sudo systemctl status sshd`
2. Tunnel Public Hostname 설정의 Path 필드가 비어있는지 확인
3. Service Type과 URL이 올바른지 확인

### Tunnel 연결 상태 빠르게 확인

```bash
curl -I https://ssh-db.example.com
```

| 응답 | 의미 |
|------|------|
| 200/3xx | 정상 |
| 502 | Tunnel OK, 로컬 서비스 문제 |
| 타임아웃 | Tunnel 자체 미연결 |

---

## 12. 서브도메인 네이밍 컨벤션

### 규칙

```
{서비스}-{서버역할}.example.com
```

### 예시

| 서브도메인 | 서비스 | 서버 |
|-----------|--------|------|
| `ssh-db.example.com` | SSH | DB 서버 |
| `ssh-web.example.com` | SSH | Web 서버 |
| `ssh-ws1.example.com` | SSH | 워크스테이션 1 |
| `db.example.com` | pgAdmin | DB 서버 |
| `demo.example.com` | 웹 데모 | Web 서버 |
| `api.example.com` | API | Web 서버 |
| `pms.example.com` | PMS | DB 서버 |
| `gpu-demo.example.com` | Gradio | 워크스테이션 1 |

### Path 라우팅은 쓰지 않는다

- SSH는 HTTP가 아니라 path 자체가 불가능
- 웹 서비스도 path prefix 붙이면 CSS/JS 경로 깨짐
- 서브도메인 분리가 가장 안전하고 깔끔

---

## 부록: 빠른 시작 체크리스트

### 서버 추가 시

- [ ] 대시보드에서 Tunnel 생성 → 토큰 복사
- [ ] Public Hostname에 SSH 서비스 추가 (ssh-{역할}.example.com)
- [ ] 서버에 cloudflared 설치
- [ ] `cloudflared service install <토큰>` 실행
- [ ] `systemctl status cloudflared` (Linux) 또는 `launchctl list | grep cloudflared` (macOS) 확인
- [ ] 로그에 `Registered tunnel connection` 4개 확인

### 서비스 추가 시 (기존 서버)

- [ ] 대시보드 → 해당 Tunnel → Public Hostnames → Add
- [ ] 서브도메인, Type, URL 입력 (Path는 비움)
- [ ] 저장 (서버 재시작 불필요)
- [ ] 브라우저 또는 curl로 접속 테스트

### 클라이언트 추가 시

- [ ] cloudflared 설치
- [ ] `~/.ssh/config`에 Host + ProxyCommand 추가
- [ ] `ssh <host-alias>` 테스트
