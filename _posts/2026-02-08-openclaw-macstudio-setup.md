---
title: OpenClaw + MacStudio 원격 제어 완벽 가이드
date: 2026-02-08 23:00:00 +0900
categories: [DevOps, OpenClaw]
tags: [openclaw, raspberry-pi, mac-studio, remote-control, wake-on-lan, telegram, slack]
---

Raspberry Pi 5에 OpenClaw Gateway를 설치하고, Mac Studio를 원격 노드로 연결해서 
Telegram/Slack에서 AI 어시스턴트로 맥을 제어하는 전체 과정을 정리했습니다.

## 시스템 구성

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   사용자 폰     │────▶│  Raspberry Pi 5 │────▶│   Mac Studio    │
│ (Telegram/Slack)│     │   (Gateway)     │     │    (Node)       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                              │
                              ▼
                        집순이 (AI)
```

### 하드웨어

| 장치 | 역할 | IP | 비고 |
|------|------|-----|-----|
| Raspberry Pi 5 (8GB) | OpenClaw Gateway | 192.168.x.10 | 64-bit OS |
| Mac Studio | OpenClaw Node | 192.168.x.100 | Apple Silicon |

---

## Part 1: Raspberry Pi 5 설정

### 1-1. 준비물

**하드웨어:**
- Raspberry Pi 5 (8GB RAM 권장, 4GB도 가능)
- USB-C 전원 (5V/5A)
- microSD 카드 (32GB 이상 권장)
- microSD 리더기
- 이더넷 케이블 (권장) 또는 Wi-Fi

**쿨링팬 (선택사항):**
- 모델: Weiyixing 3007S (30x30x7mm, 2핀)
- 연결: **Pin 4 (5V)** + **Pin 6 (GND)**

```
GPIO 헤더 (보드 위에서 봤을 때)

   3.3V  [1] [2]  5V
   SDA   [3] [4]  5V   ← 빨간 선 (+)
   SCL   [5] [6]  GND  ← 검은 선 (-)
```

### 1-2. OS 설치

1. [Raspberry Pi Imager](https://www.raspberrypi.com/software/) 다운로드 및 설치

2. Imager 실행:
   - **Device** → Raspberry Pi 5 선택
   - **OS** → Raspberry Pi OS (64-bit) 선택
   - **Storage** → microSD 카드 선택

> **중요:** 반드시 64-bit OS를 선택하세요! OpenClaw은 64-bit가 필수입니다.
{: .prompt-warning }

3. **OS Customization** (톱니바퀴 아이콘):
   - ✅ 사용자 이름/비밀번호 설정
   - ✅ Wi-Fi SSID/비밀번호 입력 (유선이면 생략 가능)
   - ✅ SSH 활성화 (headless 사용 시 필수)
   - ✅ Locale: Asia/Seoul, Keyboard: Korean

4. **Write** 클릭하여 이미지 굽기

### 1-3. 첫 부팅 및 IP 찾기

microSD를 Pi에 삽입하고 전원 연결 후, IP 주소를 찾습니다:

```bash
# 방법 1: nmap (가장 빠름)
nmap -sn 192.168.0.0/24 | grep -i raspberry

# 방법 2: bash 원라이너
for i in $(seq 1 254); do 
  ping -c 1 -W 1 192.168.0.$i &> /dev/null && echo "192.168.0.$i UP"
done

# 방법 3: 공유기 관리 페이지에서 확인
```

### 1-4. SSH 접속 및 업데이트

```bash
ssh 사용자명@192.168.x.10

# 시스템 업데이트
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

### 1-5. Node.js 설치

OpenClaw은 **Node.js 22 이상**이 필수입니다. 24 LTS를 권장합니다.

```bash
# NodeSource 저장소 추가
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -

# Node.js 설치
sudo apt install -y nodejs

# 버전 확인
node -v  # v24.x.x
npm -v
```

### 1-6. OpenClaw 설치

```bash
# 전역 설치
npm install -g openclaw@latest

# 온보딩 위저드 실행 (데몬 자동 설치 포함)
openclaw onboard --install-daemon
```

### 온보딩 중 선택 가이드

| 프롬프트 | 선택 | 이유 |
|----------|------|------|
| Homebrew 설치? | **No** | Mac 전용, Pi에선 불필요 |
| Node manager? | **npm** | NodeSource로 설치 시 npm 포함 |
| Skills 목록 | **Skip for now** | 나중에 추가 가능 |
| GOOGLE_PLACES_API_KEY | **비워두고 Enter** | 선택사항 |

### 1-7. Claude Max 인증 (setup-token)

Claude Max 구독자는 API 키 대신 **setup-token** 방식을 사용합니다.

```bash
# 별도 터미널 (또는 다른 컴퓨터)에서 토큰 생성
claude setup-token

# 출력된 토큰을 온보딩 위저드에 붙여넣기
```

> setup-token은 일회용이며, 입력 후 즉시 인증됩니다.
{: .prompt-info }

### 1-8. 설치 확인

온보딩 완료 후 **Hatch TUI**에 진입하면 성공입니다.

```bash
# 서비스 상태 확인
systemctl --user status openclaw

# Gateway 상태 확인
openclaw gateway status
```

---

## Part 2: 메시징 채널 연결

### 2-1. Telegram 연결

**Step 1: BotFather에서 봇 생성**

1. Telegram에서 **@BotFather** 검색하여 대화 시작
2. `/newbot` 명령어 전송
3. 봇 이름 입력 (예: "집순이")
4. 봇 username 입력 (예: "jipsuni_bot") - 반드시 `_bot`으로 끝나야 함
5. **봇 토큰** 복사 (예: `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`)

**Step 2: OpenClaw에 토큰 등록**

Hatch TUI에서 채널 설정으로 진입하거나, 설정 파일 직접 수정:

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_BOT_TOKEN",
      "dmPolicy": "pairing"
    }
  }
}
```

**Step 3: 페어링**

```bash
# 1. Telegram에서 봇에게 /start 전송
# 2. 페어링 코드 확인 (예: ABC123)
# 3. Gateway에서 승인
openclaw pairing approve telegram ABC123
```

### 2-2. Slack 연결

**Step 1: Slack App 생성**

1. [Slack API](https://api.slack.com/apps) 접속
2. **Create New App** → **From scratch**
3. App 이름: "OpenClaw" (또는 원하는 이름)
4. Workspace 선택

**Step 2: OAuth & Permissions 설정**

Bot Token Scopes 추가:
- `chat:write`
- `im:history`
- `im:read`
- `im:write`
- `users:read`

**Step 3: App 설치 및 토큰 복사**

1. **Install to Workspace** 클릭
2. **Bot User OAuth Token** 복사 (`xoxb-...`)

**Step 4: OpenClaw 설정**

```json
{
  "channels": {
    "slack": {
      "enabled": true,
      "botToken": "xoxb-YOUR-TOKEN",
      "appToken": "xapp-YOUR-APP-TOKEN"
    }
  }
}
```

**Step 5: 페어링**

```bash
# Slack DM에서 봇에게 메시지 전송 → 페어링 코드 확인
openclaw pairing approve slack <CODE>
```

---

## Part 3: MacStudio 노드 연결

### 3-1. MacStudio에서 OpenClaw 설치

```bash
# OpenClaw 설치
npm install -g openclaw

# Gateway 주소 설정
openclaw config set gateway.remote.url ws://192.168.x.10:18789

# 노드 서비스 설치
openclaw node install --display-name "MacStudio"
```

### 3-2. 연결 확인 (Gateway에서)

```bash
openclaw nodes status
```

정상 연결 시:
```json
{
  "displayName": "MacStudio",
  "platform": "darwin",
  "connected": true,
  "caps": ["browser", "system"]
}
```

### 3-3. Exec Approvals 설정

보안을 위해 노드에서 실행 가능한 명령어를 허용 목록으로 관리합니다.

```bash
# Gateway에서 실행
openclaw approvals allowlist add --node MacStudio "/usr/bin/git"
openclaw approvals allowlist add --node MacStudio "/opt/homebrew/bin/claude"
openclaw approvals allowlist add --node MacStudio "/bin/ls"
openclaw approvals allowlist add --node MacStudio "/bin/cat"
openclaw approvals allowlist add --node MacStudio "/sbin/ifconfig"

# 현재 허용 목록 확인
openclaw approvals list --node MacStudio
```

---

## Part 4: Wake-on-LAN 설정

### 4-1. MacStudio 설정

1. **시스템 설정** → **에너지**
2. **"네트워크 접근으로 깨우기"** ✅ 활성화

### 4-2. MAC 주소 확인

```bash
ifconfig en0 | grep ether
# 결과: ether xx:xx:xx:xx:xx:xx
```

### 4-3. Gateway에서 WoL 도구 설치

```bash
sudo apt install wakeonlan
```

### 4-4. 깨우기 명령

```bash
wakeonlan xx:xx:xx:xx:xx:xx
```

### 4-5. FileVault와 WoL

| FileVault 상태 | WoL 후 동작 |
|---------------|-------------|
| **ON** | 컴퓨터는 깨어나지만 로컬에서 비밀번호 입력 필요 (원격 불가) |
| **OFF** | 자동 로그인 → LaunchAgent 시작 → 노드 연결 (원격 가능) |

> FileVault를 끄면 WoL + 자동 로그인이 가능해져서 완전한 원격 제어가 가능합니다.
> 단, 디스크 암호화가 해제되므로 도난 시 데이터 노출 위험이 있습니다.
{: .prompt-tip }

---

## Part 5: SSH 백업 설정

노드 연결이 안 될 때 SSH로 백업 접근이 가능합니다.

### 5-1. MacStudio에서 SSH 활성화

1. **시스템 설정** → **일반** → **공유**
2. **"원격 로그인"** ✅ 활성화

### 5-2. SSH 키 설정 (선택)

```bash
# Gateway에서 SSH 키 생성
ssh-keygen -t ed25519

# MacStudio에 공개키 복사
ssh-copy-id hyangmin@192.168.x.100
```

### 5-3. 연결 테스트

```bash
ssh hyangmin@192.168.x.100 "echo SSH OK"
```

### SSH vs Node 비교

| 기능 | SSH | Node |
|------|-----|------|
| 명령어 실행 | ✅ | ✅ |
| 화면 잠금에서 작동 | ✅ | ❌ |
| Claude Code 실행 | ❌ (환경 문제) | ✅ |
| browser.proxy | ❌ | ✅ |

---

## Part 6: Claude Code 원격 실행

### Node로 실행 (권장)

```bash
# Gateway에서 OpenClaw 통해 실행
openclaw nodes run --node MacStudio -- \
  claude -p "프로젝트 설명해줘" --max-turns 1
```

### 프로젝트 디렉토리에서 실행

```bash
# 집순이에게 요청 (Telegram/Slack에서)
"MacStudio에서 ~/Downloads/github/myoffice 프로젝트 분석해줘"
```

---

## Part 7: 트러블슈팅

### 문제: 노드 연결 안 됨

```bash
# 상태 확인
openclaw nodes status

# MacStudio에서 노드 재시작
openclaw node restart
```

### 문제: Exec 권한 거부

```bash
# 명령어 허용 목록에 추가
openclaw approvals allowlist add --node MacStudio "/path/to/command"
```

### 문제: 두 개의 노드 프로세스

```bash
# 프로세스 확인
ps aux | grep openclaw

# LaunchAgent만 사용 (LaunchDaemon 제거)
sudo launchctl unload /Library/LaunchDaemons/ai.openclaw.node.plist
sudo rm /Library/LaunchDaemons/ai.openclaw.node.plist
```

### 문제: Claude Code file descriptor 에러

```bash
# 임시 해결
sudo launchctl limit maxfiles 2147483646 2147483646
```

---

## 최종 운영 방식

### 권장 설정

1. **MacStudio 잠자기 활성화** + **FileVault OFF**
   - 평소에는 잠자기 상태 (전력 절약)
   - 필요시 WoL로 깨우기
   - 자동 로그인 → 노드 자동 연결

2. **라즈베리파이 상시 가동**
   - 저전력 (5W)
   - Telegram/Slack 메시지 수신
   - 맥 없이도 기본 응답 가능

### 사용 예시

```
사용자: "MacStudio에서 myoffice git pull 해줘"
집순이: [WoL로 맥 깨우기 → 노드로 git pull 실행]

사용자: "Claude로 코드 리뷰해줘"  
집순이: [노드로 Claude Code 실행]

사용자: "MacStudio 상태 확인해줘"
집순이: [nodes status 확인 → 결과 보고]
```

---

## 참고 자료

- [OpenClaw 문서](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [Raspberry Pi 시작 가이드](https://www.raspberrypi.com/documentation/computers/getting-started.html)
- [Claude Code](https://claude.ai/code)

---

*이 문서는 2026년 2월 8일 설정 과정을 기록한 것입니다.* 🏠
