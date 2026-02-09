---
title: OpenClaw + MacStudio 원격 제어 설정 가이드
date: 2026-02-08 23:00:00 +0900
categories: [DevOps, OpenClaw]
tags: [openclaw, raspberry-pi, mac-studio, remote-control, wake-on-lan]
---

Raspberry Pi 5에 OpenClaw Gateway를 설치하고, Mac Studio를 원격 노드로 연결해서 
Telegram/Slack에서 AI 어시스턴트로 맥을 제어하는 방법을 정리했습니다.

## 시스템 구성

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   사용자 폰     │────▶│  Raspberry Pi 5 │────▶│   Mac Studio    │
│  (Telegram)     │     │   (Gateway)     │     │    (Node)       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                              │
                              ▼
                        집순이 (AI)
```

### 하드웨어

| 장치 | 역할 | IP |
|------|------|-----|
| Raspberry Pi 5 (8GB) | OpenClaw Gateway | 192.168.0.10 |
| Mac Studio | OpenClaw Node | 192.168.0.100 |

## Raspberry Pi 5 설정

### 1. OS 설치

[Raspberry Pi Imager](https://www.raspberrypi.com/software/)로 64-bit OS를 설치합니다.

> **중요:** OpenClaw은 64-bit OS가 필수입니다!
{: .prompt-warning }

### 2. Node.js 설치

OpenClaw은 Node.js 22 이상이 필요합니다.

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
```

### 3. OpenClaw 설치

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

### 온보딩 주의사항

| 프롬프트 | 선택 | 이유 |
|----------|------|------|
| Homebrew 설치? | **No** | Mac 전용 |
| Node manager? | **npm** | 이미 설치됨 |
| Claude Max 인증 | setup-token | API 키 대신 |

## MacStudio 노드 연결

### 설치

```bash
npm install -g openclaw
openclaw config set gateway.remote.url ws://192.168.0.10:18789
openclaw node install --display-name "MacStudio"
```

### Exec Approvals 설정

보안을 위해 실행 가능한 명령어를 허용 목록으로 관리합니다.

```bash
openclaw approvals allowlist add --node MacStudio "/usr/bin/git"
openclaw approvals allowlist add --node MacStudio "/opt/homebrew/bin/claude"
```

## Wake-on-LAN 설정

### MacStudio 설정

1. 시스템 설정 → 에너지
2. "네트워크 접근으로 깨우기" 활성화

### Gateway에서 WoL

```bash
sudo apt install wakeonlan
wakeonlan 9c:76:0e:48:f3:dd
```

> **FileVault 주의:** FileVault가 활성화되어 있으면 WoL 후에도 로컬에서 비밀번호 입력이 필요합니다. 원격 부팅이 필요하면 FileVault를 비활성화하세요.
{: .prompt-tip }

## Claude Code 원격 실행

```bash
openclaw nodes run --node MacStudio -- \
  claude -p "프로젝트 설명해줘" --max-turns 1
```

## 사용 예시

Telegram이나 Slack에서 AI 어시스턴트에게:

- "MacStudio에서 git pull 해줘"
- "Claude로 코드 리뷰해줘"
- "MacStudio 상태 확인해줘"

이런 식으로 자연어로 요청하면 됩니다!

## 참고 자료

- [OpenClaw 문서](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
