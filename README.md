# openclaw-wsl-onboarding-plugin

Claude Code용 플러그인입니다. WSL2 안에서 [OpenClaw](https://openclaw.ai)를 **npm 설치부터 Gateway 실행 + Control UI 대시보드 오픈까지** 한 번에 끝내줍니다.

---

## Why — 왜 만들었나

OpenClaw 처음 깔아보면 다음 지점에서 자주 막힙니다.

- **네이티브 Windows 설치 경로의 마찰**: Gateway를 서비스로 등록하려면 Scheduled Task 권한이 필요하고, 다른 OpenClaw 인스턴스(WSL쪽 등)와 Bonjour/mDNS 이름 충돌로 `(2)`, `(3)`이 붙으며 로그가 도배됩니다.
- **비대화형 onboarding의 숨겨진 플래그들**: `--accept-risk`가 빠지면 즉시 거부되고, `--skip-daemon`을 안 쓰면 서비스 설치 단계에서 실패하며, `--skip-health`가 없으면 아직 뜨지도 않은 Gateway를 health check하러 가다 터집니다.
- **API 키 저장 모드의 함정**: `--secret-input-mode ref`로 온보딩하면 TUI가 실행될 때마다 `No API key found for provider "<provider>". Auth store: ~/.openclaw/agents/main/agent/auth-profiles.json` 에러로 실패합니다. env var가 해당 쉘에 export돼있어야만 동작하기 때문입니다.
- **PATH가 스크립트에서만 안 먹히는 문제**: Ubuntu 기본 `~/.bashrc`는 파일 첫 부분에서 비대화형 쉘이면 즉시 `return`하기 때문에, 그 뒤에 추가한 PATH export가 `bash -lc`로 들어온 명령에는 적용되지 않습니다. 결과적으로 `openclaw: command not found`가 스크립트에서만 납니다.
- **설정 키 이름의 비자명함**: mDNS를 끄고 싶어서 `gateway.advertise.mdns = false`라고 쓰면 `Unrecognized key`로 튕깁니다. 실제 키는 `gateway.discovery.mdns.mode`이고 값은 `off | minimal | full`입니다.

이 플러그인은 위 지점들을 **한 번 겪은 사람이 정답만 추려서** 스크립트 가능한 형태로 묶은 것입니다. 동료가 OpenClaw를 새 WSL에 처음 깔 때 같은 함정을 다시 밟지 않게 하는 것이 목적입니다.

---

## What — 무엇이 들어있나

```
.
├── .claude-plugin/plugin.json            # 플러그인 매니페스트
├── skills/openclaw-onboarding/SKILL.md   # 대화 레이어 (provider/key 수집, 결과 전달)
├── agents/openclaw-onboarder.md          # 실제 설치/온보딩을 수행하는 subagent
└── hooks/hooks.json                      # SubagentStart/Stop 훅 → 실행 로그
```

**역할 분담**

| 컴포넌트 | 역할 |
|---|---|
| **Skill** (`openclaw-onboarding`) | 사용자와 대화하며 provider/key/(선택)모델을 모음. API 키가 채팅에 노출됐다면 경고. 이후 subagent에 self-contained brief를 넘김. 돌아온 결과를 사용자에게 전달. |
| **Subagent** (`openclaw-onboarder`) | 독립된 컨텍스트에서 WSL 진입 → Node 확인 → `npm install -g openclaw` → `~/.profile` PATH 보정 → `openclaw onboard --non-interactive --accept-risk --<provider>-api-key '<KEY>' --skip-daemon --skip-channels --skip-health --secret-input-mode plaintext` → (선택) 기본 모델 교체 → `nohup openclaw gateway run`으로 백그라운드 기동 → `127.0.0.1:18789` 리스닝 확인 → 토큰 추출. 사용자와 직접 대화하지 않고 구조화된 리포트만 리턴. |
| **Hooks** | `SubagentStart`와 `SubagentStop`을 subagent 이름 `openclaw-onboarder`로 매치해서, 시작/종료 타임스탬프와 `session_id`를 `~/.claude/logs/openclaw-onboarding.log`에 append. 온보딩 재현 감사 용도. |

**설계상 한계(의도된 제약)**

- **WSL2 전용**: 네이티브 Windows 경로는 의도적으로 다루지 않습니다. 그쪽은 함정이 많아 유지보수 비용이 훨씬 큽니다.
- **로컬 루프백 싱글 유저**: Gateway는 `127.0.0.1:18789`에 바인드합니다. LAN 노출은 이 플러그인의 범위 밖입니다.
- **API 키 평문 저장**: 키는 `~/.openclaw/agents/main/agent/auth-profiles.json`에 파일 권한 600으로 평문 저장됩니다. `ref` 모드의 런타임 env 의존 문제를 피하기 위한 의도적 선택입니다. 공유 머신에는 권장하지 않습니다.

---

## How — 어떻게 쓰나

### 설치

Claude Code 세션에서:

```
/plugin install https://github.com/OpenClaw-Korea/openclaw-wsl-onboarding-plugin
```

또는 로컬 클론 후 경로로 설치:

```bash
git clone https://github.com/OpenClaw-Korea/openclaw-wsl-onboarding-plugin \
  ~/.claude/local-plugins/openclaw-onboarding
```
```
/plugin install ~/.claude/local-plugins/openclaw-onboarding
```

### 사전 조건

호스트(Windows):

- Windows 10 이상 + WSL2 활성화
- WSL에 Ubuntu(또는 호환 배포판) 설치

WSL 내부:

- Node 24 권장 (22.14+ 허용)
- `npm`, `python3`, `jq` — Ubuntu 기본 셋에 포함됨

### 실행

Claude Code 대화 중에 아래 비슷하게 말하면 스킬이 트리거됩니다.

- "openclaw WSL에 깔아줘"
- "openclaw 처음부터 설정"
- "install openclaw"
- "get the openclaw dashboard running"

스킬이 provider와 API 키를 묻고, 원하는 모델 ID(선택)까지 받은 뒤 subagent에 일을 넘깁니다. subagent는 WSL 안에서 전체 플로우를 돌린 뒤 다음을 리포트합니다.

- 대시보드 URL (보통 `http://127.0.0.1:18789/`)
- Gateway 인증 토큰
- 사용된 primary 모델
- 경고 사항(기존 설치 리셋, PATH 수정, 좀비 Gateway 정리 등)
- 다음 액션(대시보드 열기 / `openclaw tui` 실행)

### 로그 확인

```bash
wsl -e bash -c "tail -f ~/.claude/logs/openclaw-onboarding.log"
```

예시:

```
[2026-04-19T13:45:02Z] start session=abc123 cwd=/home/user
[2026-04-19T13:46:37Z] end   session=abc123
```

---

## 보안 권고

- 가능하면 API 키를 Claude Code 채팅에 직접 붙여넣지 마세요. 채팅 전사에 남습니다. 이미 붙여넣었다면 온보딩이 끝나는 즉시 provider의 키 관리 페이지에서 **rotate**하세요.
- 이 플러그인은 키를 외부로 전송하지 않습니다. WSL 안에서 `openclaw onboard`를 로컬로 호출할 뿐이며, 키는 OpenClaw가 자체 파일(`~/.openclaw/agents/main/agent/auth-profiles.json`)에 저장합니다.
- Gateway는 loopback 바인드가 기본입니다. `gateway.bind`를 `lan` 또는 `auto`로 바꾸기 전에 해당 네트워크의 접근 주체를 반드시 점검하고 강력한 토큰을 설정하세요.

---

## 라이선스

MIT
