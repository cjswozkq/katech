# ControlDesk MCP 설치 가이드 (빈 Windows PC 기준)

참고: [dSPACE MCP Hub](https://github.com/dspace-group/dSPACE-MCP-Hub), [ControlDeskMCP](https://github.com/dspace-group/ControlDeskMCP) (v0.1.0 기준)

## 0. 구조 이해

```
[Claude Desktop / Claude Code / VS Code Copilot]  ← MCP 클라이언트
          │ stdin/stdout (stdio)
          ▼
[ControlDeskMCP (.exe 또는 .cmd → uv → Python)]  ← 클라이언트가 자식 프로세스로 실행
          │ COM 자동화
          ▼
[dSPACE ControlDesk]
```

- 서버는 **ControlDesk가 설치된 같은 Windows PC**에서 돌아야 함 (COM 사용). 네트워크 포트 안 엶.
- 서버를 직접 "켜둘" 필요 없음. 클라이언트가 알아서 실행함.

## 1. 전제 조건

| 항목 | 내용 |
|---|---|
| OS | Windows (64-bit) |
| ControlDesk | 2025-A 이상 설치 + **라이선스 활성화** (이 PC: **2025-A**) |
| PowerShell | Windows 기본 포함 |

ControlDesk COM이 살아있는지 먼저 확인 (PowerShell):

```powershell
$app = New-Object -ComObject ControlDeskNG.Application
$app.GetType().FullName
```

에러가 나면 MCP 문제가 아니라 ControlDesk 설치/라이선스 문제. 먼저 해결.

## 2. 서버 설치 — 방법 A 또는 B 중 하나

### 방법 A: 릴리스 exe (가장 간단, Python/uv/Git 불필요)

1. `C:\mcp\ControlDeskMCP` 폴더 생성
2. https://github.com/dspace-group/ControlDeskMCP/releases 에서 최신 릴리스의
   `ControlDeskMCP.exe`, `ControlDeskMCP.exe.sha256` 다운로드 → 위 폴더에 저장
3. 해시 검증:
   ```powershell
   cd C:\mcp\ControlDeskMCP
   (Get-FileHash .\ControlDeskMCP.exe -Algorithm SHA256).Hash
   Get-Content .\ControlDeskMCP.exe.sha256
   ```
   두 값이 같으면 OK. (다운로드 파일이 차단되면 파일 우클릭 → 속성 → "차단 해제")
4. 동작 확인:
   ```powershell
   .\ControlDeskMCP.exe --version
   .\ControlDeskMCP.exe --list-tools
   ```

→ 이후 설정에서 실행 경로는 `C:\mcp\ControlDeskMCP\ControlDeskMCP.exe`

### 방법 B: 소스 + uv (MCP Hub 공식 방식)

1. **Git 설치** (PowerShell):
   ```powershell
   winget install --id Git.Git -e
   ```
2. **uv 설치**:
   ```powershell
   powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
   ```
   (또는 `winget install --id astral-sh.uv -e`)
3. **PowerShell 창을 닫고 새로 열기** (PATH 반영). 확인:
   ```powershell
   git --version
   uv --version
   ```
4. **Python** — 따로 설치 안 해도 `uv sync`가 64-bit Python 3.11+를 자동으로 받아옴.
   직접 설치하려면 `uv python install 3.12`.
5. **클론 + 의존성 설치**:
   ```powershell
   mkdir C:\mcp
   cd C:\mcp
   git clone https://github.com/dspace-group/ControlDeskMCP.git
   cd ControlDeskMCP
   uv sync
   ```
6. 동작 확인 (ControlDesk 없이도 동작):
   ```powershell
   .\ControlDeskMCP.cmd --version
   .\ControlDeskMCP.cmd --list-tools
   ```

→ 이후 설정에서 실행 경로는 `C:\mcp\ControlDeskMCP\ControlDeskMCP.cmd`

> 업데이트: `cd C:\mcp\ControlDeskMCP; git pull; uv sync`

## 3. MCP 클라이언트 연결 — 쓰는 것 하나만

아래 예시는 **방법 B(.cmd)** 기준. 방법 A면 `.cmd` 경로를 `.exe` 경로로 바꾸고,
`cmd /c` 래퍼 없이 exe를 바로 `command`에 넣어도 됨.

> `.cmd` 파일은 `cmd /c`로 감싸서 실행하는 게 안전함 (Node 기반 클라이언트가 .cmd를 직접 spawn하지 못하는 경우가 있음).

### 3-1. Claude Desktop

1. https://claude.ai/download 에서 설치 후 로그인
2. 설정 파일 열기 (없으면 생성됨):
   ```powershell
   notepad "$env:APPDATA\Claude\claude_desktop_config.json"
   ```
   (앱에서 Settings → Developer → Edit Config 로도 열 수 있음)
3. 내용:
   ```json
   {
     "mcpServers": {
       "controldesk": {
         "command": "cmd",
         "args": ["/c", "C:\\mcp\\ControlDeskMCP\\ControlDeskMCP.cmd"],
         "env": {
           "CONTROLDESK_VERSION": "2025-A"
         }
       }
     }
   }
   ```
   exe 사용 시:
   ```json
   {
     "mcpServers": {
       "controldesk": {
         "command": "C:\\mcp\\ControlDeskMCP\\ControlDeskMCP.exe",
         "args": [],
         "env": { "CONTROLDESK_VERSION": "2025-A" }
       }
     }
   }
   ```
   - JSON 경로는 `\\` 두 번. 마지막 콤마 금지.
   - 이 PC는 ControlDesk **2025-A** → `"2025-A"` (하이픈 필수! `2025A`는 형식 오류로 서버가 안 뜸).
     ControlDesk가 하나만 설치돼 있으면 `env` 블록 통째로 빼도 자동 감지됨.
4. Claude Desktop **완전 종료**(트레이 아이콘 → Quit) 후 재실행
5. 채팅 입력창의 도구(🔨/설정) 아이콘에 `controldesk`가 보이면 성공

### 3-2. Claude Code (CLI)

1. 설치 (PowerShell, Git for Windows 필요 → 2-B의 1번):
   ```powershell
   irm https://claude.ai/install.ps1 | iex
   ```
2. 서버 등록 (모든 프로젝트에서 쓰려면 `--scope user`):
   ```powershell
   claude mcp add controldesk --scope user -e CONTROLDESK_VERSION=2025-A -- cmd /c C:\mcp\ControlDeskMCP\ControlDeskMCP.cmd
   ```
   exe 사용 시:
   ```powershell
   claude mcp add controldesk --scope user -e CONTROLDESK_VERSION=2025-A -- C:\mcp\ControlDeskMCP\ControlDeskMCP.exe
   ```
3. 확인: `claude mcp list` → `controldesk: ... ✓ Connected`
   `claude` 실행 후 `/mcp` 로도 상태 확인 가능

### 3-3. VS Code + GitHub Copilot

작업 폴더에 `.vscode/mcp.json`:

```json
{
  "servers": {
    "ControlDeskMCP": {
      "type": "stdio",
      "command": "C:\\mcp\\ControlDeskMCP\\ControlDeskMCP.cmd",
      "env": { "CONTROLDESK_VERSION": "2025-A" }
    }
  }
}
```

파일에 뜨는 **Start** 버튼 클릭 → Copilot Chat을 **Agent 모드**로 전환 → 도구 목록에서 확인.

## 4. 연결 테스트

클라이언트 채팅에 순서대로:

```
controldesk://server/info 리소스를 읽어서 보여줘.
```
(ControlDesk를 건드리지 않음 → 서버 연결만 확인)

```
ControlDesk에 연결하고 사용 가능한 플랫폼 보여줘.
```
(`controldesk_app_start_or_attach` → ControlDesk 실행/연결)

```
최근 ControlDesk 프로젝트 목록 보여주고 가장 최근 걸 열어줘.
```

일반적인 작업 순서: 앱 연결 → 프로젝트/실험 열기 → 플랫폼 연결 → 측정/캘리브레이션/변수 작업
→ 측정·레코더 정지 → 플랫폼 연결 해제 → (필요 시) 앱 종료

## 5. 선택 환경변수

`env`에 넣어서 사용:

| 변수 | 용도 | 예 |
|---|---|---|
| `CONTROLDESK_VERSION` | 특정 ControlDesk 버전 지정 (없으면 최신). 반드시 `YYYY-L` 형식 | `2025-A` |
| `COM_LAUNCH_TIMEOUT_MS` | ControlDesk 기동 대기시간 (느린 PC) | `60000` |
| `COM_TIMEOUT_MS` | COM 호출 타임아웃 | `30000` |
| `LOG_LEVEL` | 로그 레벨 | `DEBUG` |

## 6. 트러블슈팅

| 증상 | 조치 |
|---|---|
| `uv` 인식 안 됨 | 터미널/클라이언트 재시작. 그래도 안 되면 로그아웃·재로그인 (PATH 반영) |
| 클라이언트에 서버가 안 보임 | JSON 문법(콤마, `\\`) 확인 → 클라이언트 완전 재시작 → 경로에서 `--version` 직접 실행 |
| 클라이언트에서만 실패, 터미널에선 OK | 클라이언트가 PATH에서 uv를 못 찾는 경우. 방법 A(exe)로 바꾸는 게 제일 확실 |
| ControlDesk not found | 라이선스·설치 확인, `CONTROLDESK_VERSION` 명시, `COM_LAUNCH_TIMEOUT_MS` 증가 |
| COM 호출 거부/멈춤 | ControlDesk에 떠 있는 모달 대화상자 닫고 재시도 |
| 타임아웃 / ControlDesk 종료됨 | `controldesk_app_get_logs` 로 로그 확인 후 `controldesk_app_start_or_attach` 로 재연결 |
| Python 버전/비트 오류 | 64-bit Python 3.11+ 필요 (`uv python install 3.12`) |

Claude Desktop 로그: `%APPDATA%\Claude\logs\mcp-server-controldesk.log`
