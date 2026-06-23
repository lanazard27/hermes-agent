# 설계안: /sc claude, /sc opencode 대화형 세션 지원

## 1. 문제 요약

| 항목 | 현재 동작 | 목표 |
|------|----------|------|
| `_handle_message` | `_external_session_channels` 메시지 → early return (무시) | 메시지를 받아 runner spawn |
| `claude_session_runner.py` | `claude --print "프롬프트"` 1회 실행 후 종료 | 세션 ID 저장 → `claude --print --resume <id>` 로 재사용 |
| `opencode_session_runner.py` | `opencode run "프롬프트"` 1회 실행 후 종료 | 세션 ID 저장 → `opencode run --session <id>` 로 재사용 |
| adapter.py | runner spawn은 `/sc` 최초 1회만 | 채널에 메시지 올 때마다 runner spawn |

## 2. CLI 세션 재개 메커니즘 (검증 완료)

### Claude Code CLI
```bash
# 최초 실행: JSON 출력으로 session_id 획득
claude --print --output-format json "프롬프트"
# → {"session_id": "uuid-xxx", "result": "..."}

# 후속 실행: --resume로 세션 재개
claude --print --resume "uuid-xxx" "후속 프롬프트"
```
- `--output-format json` → JSON 출력에 `session_id` 포함
- `--resume <uuid>` → 해당 세션 컨텍스트를 이어서 실행

### OpenCode CLI
```bash
# 최초 실행: --format json으로 sessionID 획득
opencode run --format json --title "이름" "프롬프트"
# → 이벤트 스트림에 {"sessionID": "ses_xxx"} 포함

# 후속 실행: --session으로 재개
opencode run --session "ses_xxx" "후속 프롬프트"

# 또는 --continue로 마지막 세션 재개
opencode run --continue "후속 프롬프트"
```
- `--format json` → 이벤트에 `sessionID` 필드 포함
- `--session <id>` → 특정 세션 재개
- `--continue` → 마지막 세션 재개 (세션 ID 모를 때 fallback)

## 3. 수정 파일 목록 및 변경 내용

### 3-1. `adapter.py` — 세션 레지스트리 + 메시지 라우팅

#### A. `_external_session_channels` → `_external_sessions` (구조 변경)

**현재:**
```python
self._external_session_channels: set = set()  # channel_id만 저장
```

**변경:**
```python
# channel_id → ExternalSessionInfo 매핑
self._external_sessions: Dict[str, "ExternalSessionInfo"] = {}

# 동시 실행 제어: channel_id → 현재 실행 중인 subprocess
self._active_runners: Dict[str, subprocess.Popen] = {}

# 메시지 큐: runner가 실행 중일 때 후속 메시지 대기
self._pending_messages: Dict[str, List[str]] = {}
```

새 데이터 클래스:
```python
@dataclass
class ExternalSessionInfo:
    session_type: str           # "claude" | "opencode"
    channel_id: str             # Discord channel ID
    session_name: str           # 채널명
    workdir: str                # 작업 디렉토리
    cli_session_id: str | None  # Claude/OpenCode CLI의 세션 ID (최초 실행 후 저장)
    model: str | None           # opencode용 모델
    created_at: float           # 생성 시각
```

#### B. `_handle_sc_slash` 변경

```python
# AS-IS:
self._external_session_channels.add(channel_id)

# TO-BE:
from time import time
info = ExternalSessionInfo(
    session_type=session_type,
    channel_id=channel_id,
    session_name=resolved_name,
    workdir=str(Path.home()),
    cli_session_id=None,   # 첫 runner 실행 후 저장
    model="zai-coding-plan/glm-5.1" if session_type == "opencode" else None,
    created_at=time(),
)
self._external_sessions[channel_id] = info
```

#### C. `_handle_message` 변경 — early return 대신 라우팅

**현재 (5603-5612):**
```python
if msg_channel_id and msg_channel_id in self._external_session_channels:
    logger.info("Ignored message in external session channel %s", msg_channel_id)
    return
```

**변경:**
```python
if msg_channel_id and msg_channel_id in self._external_sessions:
    # 봇 자신의 메시지는 무시
    if message.author.id == self._bot_user_id:
        return
    await self._dispatch_external_session_message(msg_channel_id, message)
    return
```

#### D. 새 메서드: `_dispatch_external_session_message`

```python
async def _dispatch_external_session_message(
    self, channel_id: str, message: "DiscordMessage"
) -> None:
    """외부 세션 채널에 메시지가 오면 runner를 spawn하거나 큐잉."""
    info = self._external_sessions.get(channel_id)
    if info is None:
        return

    prompt = message.content.strip()
    if not prompt:
        return

    # 동시 실행 체크
    active = self._active_runners.get(channel_id)
    if active is not None and active.poll() is None:
        # runner가 아직 실행 중 → 큐잉
        self._pending_messages.setdefault(channel_id, []).append(prompt)
        await self._send_discord_message(
            channel_id,
            f"⏳ 이전 요청 처리 중... 메시지가 큐에 추가되었습니다. (대기: {len(self._pending_messages[channel_id])}건)"
        )
        return

    # runner spawn
    await self._spawn_runner(info, prompt)
```

#### E. 새 메서드: `_spawn_runner`

```python
async def _spawn_runner(self, info: ExternalSessionInfo, prompt: str) -> None:
    """세션 정보와 프롬프트로 runner subprocess를 실행."""
    runner_path, cmd = self._build_runner_command(info, prompt)
    try:
        proc = subprocess.Popen(
            cmd,
            start_new_session=True,
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
        )
        self._active_runners[info.channel_id] = proc
        logger.info(
            "[ext-session] Spawned runner for channel=%s type=%s pid=%d",
            info.channel_id, info.session_type, proc.pid,
        )
        # 완료 감지를 위한 백그라운드 태스크
        asyncio.ensure_future(
            self._watch_runner_completion(info.channel_id)
        )
    except Exception as exc:
        logger.error("[ext-session] Failed to spawn runner: %s", exc)
        await self._send_discord_message(
            info.channel_id,
            f"⚠️ Runner 실행 실패: {exc}"
        )
```

#### F. 새 메서드: `_build_runner_command`

```python
def _build_runner_command(
    self, info: ExternalSessionInfo, prompt: str
) -> tuple[str, list[str]]:
    """세션 타입에 따라 runner 명령어를 빌드."""
    if info.session_type == "claude":
        runner_path = str(Path(__file__).parent / "claude_session_runner.py")
        cmd = [
            sys.executable, runner_path,
            "--thread-id", info.channel_id,
            "--bot-token", self.config.token,
            "--discord-api", "https://discord.com/api/v10",
            "--prompt", prompt,
            "--session-name", info.session_name,
        ]
        if info.cli_session_id:
            cmd += ["--resume", info.cli_session_id]
        return runner_path, cmd

    elif info.session_type == "opencode":
        runner_path = str(Path(__file__).parent / "opencode_session_runner.py")
        cmd = [
            sys.executable, runner_path,
            "--thread-id", info.channel_id,
            "--bot-token", self.config.token,
            "--discord-api", "https://discord.com/api/v10",
            "--prompt", prompt,
            "--session-name", info.session_name,
        ]
        if info.model:
            cmd += ["--model", info.model]
        if info.cli_session_id:
            cmd += ["--session-id", info.cli_session_id]
        return runner_path, cmd
```

#### G. 새 메서드: `_watch_runner_completion`

```python
async def _watch_runner_completion(self, channel_id: str) -> None:
    """Runner subprocess 완료를 감지하고 세션 ID를 업데이트."""
    proc = self._active_runners.get(channel_id)
    if proc is None:
        return

    # 폴링 (10초 간격)
    while proc.poll() is None:
        await asyncio.sleep(10)

    rc = proc.returncode
    logger.info("[ext-session] Runner completed: channel=%s rc=%d", channel_id, rc)

    # 세션 ID 파일에서 읽기 (runner가 작성)
    info = self._external_sessions.get(channel_id)
    if info and info.cli_session_id is None:
        session_id = self._read_saved_session_id(channel_id, info.session_type)
        if session_id:
            info.cli_session_id = session_id
            logger.info("[ext-session] Saved CLI session ID: %s → %s", channel_id, session_id)

    # active runner에서 제거
    self._active_runners.pop(channel_id, None)

    # 대기 중인 메시지가 있으면 다음 실행
    pending = self._pending_messages.get(channel_id, [])
    if pending:
        next_prompt = pending.pop(0)
        if not pending:
            del self._pending_messages[channel_id]
        await self._spawn_runner(info, next_prompt)
```

#### H. 세션 ID 읽기 헬퍼

```python
def _read_saved_session_id(self, channel_id: str, session_type: str) -> str | None:
    """Runner가 저장한 세션 ID 파일에서 읽기."""
    if session_type == "claude":
        path = Path.home() / ".hermes" / f"claude-session-{channel_id}.id"
    else:
        path = Path.home() / ".hermes" / f"opencode-session-{channel_id}.id"
    try:
        if path.exists():
            sid = path.read_text().strip()
            return sid if sid else None
    except Exception as e:
        logger.warning("[ext-session] Failed to read session ID: %s", e)
    return None
```

#### I. Discord 메시지 전송 헬퍼

```python
async def _send_discord_message(self, channel_id: str, content: str) -> None:
    """간단한 Discord 메시지 전송."""
    try:
        channel = self._bot.get_channel(int(channel_id))
        if channel:
            await channel.send(content)
    except Exception as e:
        logger.warning("[ext-session] Failed to send message: %s", e)
```

---

### 3-2. `claude_session_runner.py` 변경

#### A. 새 argparse 인자

```python
parser.add_argument("--resume", default=None,
                    help="Claude CLI session ID to resume")
```

#### B. 세션 ID 저장 로직 추가

```python
def main():
    # ... 기존 인자 파싱 ...

    session_id_path = Path.home() / ".hermes" / f"claude-session-{args.thread_id}.id"
    session_id_path.parent.mkdir(parents=True, exist_ok=True)

    # ── 1. Post "started" message ──
    is_resume = bool(args.resume)
    resume_label = " (이어서)" if is_resume else ""
    post_message(
        args.discord_api, args.thread_id, args.bot_token,
        (
            f"[CLAUDE-SESSION] 🔧 **Claude Code 세션 시작{resume_label}**\n"
            f"세션: `{args.session_name}`\n"
            f"{'재개: `' + args.resume + '`' if is_resume else '새 세션'}\n"
            f"프롬프트 처리 중..."
        ),
    )

    # ── 2. Run Claude Code CLI ──
    cmd = [
        "claude",
        "--print",
        "--output-format", "json",
        "--dangerously-skip-permissions",
        "--max-turns", "120",
    ]
    if args.resume:
        cmd += ["--resume", args.resume]
    cmd.append(args.prompt)

    # ... 실행 ...

    # ── 3. Parse JSON output & extract session_id ──
    cli_session_id = None
    try:
        result_json = json.loads(stdout_text)
        cli_session_id = result_json.get("session_id")
        # 실제 텍스트 결과 추출
        actual_text = result_json.get("result", stdout_text)
    except (json.JSONDecodeError, AttributeError):
        actual_text = stdout_text

    # 세션 ID 저장 (adapter.py가 읽어감)
    if cli_session_id:
        session_id_path.write_text(cli_session_id, encoding="utf-8")
        logger.info("Saved session ID: %s", cli_session_id)

    # ── 4. Post result ──
    display_text = actual_text if cli_session_id else stdout_text
    result_summary = display_text[:1800]
    if len(display_text) > 1800:
        result_summary += "\n...(truncated)"

    session_info = f"\n세션 ID: `{cli_session_id}`" if cli_session_id else ""
    status_emoji = "✅" if result.returncode == 0 else "⚠️"
    result_msg = (
        f"[CLAUDE-SESSION] {status_emoji} **완료** (exit: {result.returncode}){session_info}\n\n"
        f"<result>\n{result_summary}\n</result>"
    )
    post_message(args.discord_api, args.thread_id, args.bot_token, result_msg)

    # ── 5. Write log ──
    log_content = (
        f"Session: {args.session_name}\n"
        f"CLI Session ID: {cli_session_id}\n"
        f"Thread: {args.thread_id}\n"
        f"Resume: {args.resume}\n"
        f"Exit code: {result.returncode}\n"
        f"Prompt: {args.prompt[:500]}\n\n"
        f"--- stdout ---\n{result.stdout}\n\n"
        f"--- stderr ---\n{result.stderr}\n"
    )
    log_path.write_text(log_content, encoding="utf-8")
```

---

### 3-3. `opencode_session_runner.py` 변경

#### A. 새 argparse 인자

```python
parser.add_argument("--session-id", default=None, dest="resume_session_id",
                    help="OpenCode session ID to resume")
```

#### B. 세션 ID 저장 로직 추가

```python
def main():
    # ... 기존 인자 파싱 ...

    session_id_path = Path.home() / ".hermes" / f"opencode-session-{args.thread_id}.id"
    session_id_path.parent.mkdir(parents=True, exist_ok=True)

    # ── 1. Post "started" message ──
    is_resume = bool(args.resume_session_id)
    resume_label = " (이어서)" if is_resume else ""
    post_message(
        args.discord_api, args.thread_id, args.bot_token,
        (
            f"[OPENCODE-SESSION] 🚀 **OpenCode 세션 시작{resume_label}**\n"
            f"세션: `{args.session_name}`\n"
            f"{'재개: `' + args.resume_session_id + '`' if is_resume else '새 세션'}\n"
            f"프롬프트 처리 중..."
        ),
    )

    # ── 2. Run OpenCode CLI ──
    cmd = [
        "opencode", "run",
        "--format", "json",
        "--title", args.session_name,
    ]
    if args.model:
        cmd += ["--model", args.model]
    if args.resume_session_id:
        cmd += ["--session", args.resume_session_id]
    cmd.append(args.prompt)

    # ... 실행 (subprocess.run) ...

    # ── 3. Parse JSON event stream & extract sessionID ──
    cli_session_id = None
    actual_text = ""
    try:
        for line in stdout_text.splitlines():
            line = line.strip()
            if not line:
                continue
            event = json.loads(line)
            if "sessionID" in event and not cli_session_id:
                cli_session_id = event["sessionID"]
            if event.get("type") == "result":
                r = event.get("result", {})
                if isinstance(r, dict):
                    actual_text += r.get("text", "")
                else:
                    actual_text += str(r)
    except (json.JSONDecodeError, AttributeError):
        actual_text = stdout_text

    if not actual_text:
        actual_text = stdout_text or "(no output)"

    # 세션 ID 저장
    if cli_session_id:
        session_id_path.write_text(cli_session_id, encoding="utf-8")
        logger.info("Saved session ID: %s", cli_session_id)

    # ── 4. Post result ──
    display_text = actual_text
    result_summary = display_text[:1800]
    if len(display_text) > 1800:
        result_summary += "\n...(truncated)"

    session_info = f"\n세션 ID: `{cli_session_id}`" if cli_session_id else ""
    status_emoji = "✅" if result.returncode == 0 else "⚠️"
    result_msg = (
        f"[OPENCODE-SESSION] {status_emoji} **완료** (exit: {result.returncode}){session_info}\n\n"
        f"<result>\n{result_summary}\n</result>"
    )
    post_message(args.discord_api, args.thread_id, args.bot_token, result_msg)

    # ── 5. Write log ──
    log_content = (
        f"Session: {args.session_name}\n"
        f"CLI Session ID: {cli_session_id}\n"
        f"Thread: {args.thread_id}\n"
        f"Resume Session: {args.resume_session_id}\n"
        f"Exit code: {result.returncode}\n"
        f"Prompt: {args.prompt[:500]}\n\n"
        f"--- stdout ---\n{result.stdout}\n\n"
        f"--- stderr ---\n{result.stderr}\n"
    )
    log_path.write_text(log_content, encoding="utf-8")
```

---

## 4. 세션 ID 저장 방식

### 파일 기반 (채택)

```
~/.hermes/claude-session-{channel_id}.id   ← claude CLI session_id (UUID)
~/.hermes/opencode-session-{channel_id}.id ← opencode CLI sessionID (ses_xxx)
```

**선택 이유:**
- **영속성**: gateway 재시작 후에도 복구 가능
- **단순성**: 파일 1개 = 세션 1개, race condition 없음
- **호환성**: 기존 log 파일(`claude-session-{id}.log`)과 동일한 패턴

### 복구 시나리오

gateway 재시작 시:
1. `_external_sessions` dict는 메모리에서 손실됨
2. 하지만 Discord 채널은 유지됨
3. 복구 방법: `_external_session_channels` 복원 로직을 `_handle_message`에 추가

```python
# _handle_message에서 채널이 알려진 external session이 아니면
# 파일 시스템에서 세션 ID 파일이 있는지 확인하여 복구
if msg_channel_id not in self._external_sessions:
    for stype in ("claude", "opencode"):
        path = Path.home() / ".hermes" / f"{stype}-session-{msg_channel_id}.id"
        if path.exists():
            session_id = path.read_text().strip()
            self._external_sessions[msg_channel_id] = ExternalSessionInfo(
                session_type=stype,
                channel_id=msg_channel_id,
                session_name=f"recovered-{msg_channel_id[:8]}",
                workdir=str(Path.home()),
                cli_session_id=session_id,
                model="zai-coding-plan/glm-5.1" if stype == "opencode" else None,
                created_at=time(),
            )
            break
```

## 5. 메시지 수신 → Runner Spawn 흐름

```
[Discord 사용자 메시지]
       │
       ▼
_handle_message()
       │
       ├─ channel_id in _external_sessions?
       │     │
       │     ├─ YES → _dispatch_external_session_message()
       │     │              │
       │     │              ├─ 빈 메시지? → return
       │     │              │
       │     │              ├─ active runner 있음?
       │     │              │     ├─ YES → _pending_messages에 큐잉
       │     │              │     │        "⏳ 이전 요청 처리 중..." 안내
       │     │              │     └─ NO  → _spawn_runner()
       │     │              │
       │     │              └─ return
       │     │
       │     └─ NO → 파일 시스템에서 복구 시도 → 성공 시 위와 동일
       │
       └─ NO → 기존 Hermes 응답 흐름 (변경 없음)

_spawn_runner()
       │
       ├─ _build_runner_command() → 세션 타입별 명령어 생성
       │     ├─ claude: --resume <id> (있으면)
       │     └─ opencode: --session <id> (있으면)
       │
       ├─ subprocess.Popen() → detached process
       │
       └─ _watch_runner_completion() (async 백그라운드)
              │
              ├─ 10초 폴링으로 완료 대기
              │
              ├─ 완료 시:
              │     ├─ .id 파일에서 세션 ID 읽어 _external_sessions 업데이트
              │     ├─ _active_runners에서 제거
              │     └─ _pending_messages에 대기 메시지 있으면 → 다시 _spawn_runner()
              │
              └─ 에러 시에도 정리 수행
```

## 6. 동시 실행 제어

### 전략: 채널당 1개 runner + 큐잉

```
_active_runners: Dict[str, subprocess.Popen]
_pending_messages: Dict[str, List[str]]
```

| 시나리오 | 동작 |
|----------|------|
| runner 없음 + 메시지 1개 | 즉시 spawn |
| runner 실행 중 + 메시지 1개 | 큐잉 + 안내 메시지 |
| runner 실행 중 + 메시지 3개 | 모두 큐잉 (순서 보장) |
| runner 완료 + 큐 2개 | 첫 번째 메시지로 spawn, 나머지 큐 유지 |

### 대안 고려사항

| 대안 | 장점 | 단점 | 채택? |
|------|------|------|-------|
| **큐잉 (채택)** | 순서 보장, 모든 메시지 처리 | 지연 가능 | ✅ |
| 거부 (busy 에러) | 구현 간단 | 사용자 경험 나쁨 | ❌ |
| 채널당 N개 동시 실행 | 처리량 증가 | 세션 꼬임 위험 | ❌ |
| 메시지 병합 (하나의 프롬프트로) | 효율적 | 의도 파악 어려움 | ❌ |

### 타임아웃 보호

```python
# _watch_runner_completion에 타임아웃 추가
async def _watch_runner_completion(self, channel_id: str) -> None:
    proc = self._active_runners.get(channel_id)
    if proc is None:
        return

    max_wait = 3600  # 1시간
    start = time.time()
    while proc.poll() is None:
        if time.time() - start > max_wait:
            proc.kill()
            await self._send_discord_message(
                channel_id,
                "⚠️ Runner 타임아웃 (1시간) — 프로세스를 종료했습니다."
            )
            break
        await asyncio.sleep(10)

    # 정리 로직...
```

## 7. 에러 처리

### Runner 실행 실패

```python
# _spawn_runner의 except 블록
except Exception as exc:
    logger.error("[ext-session] Failed to spawn runner: %s", exc)
    await self._send_discord_message(
        info.channel_id,
        f"⚠️ Runner 실행 실패: {exc}"
    )
    # active_runners에서 제거 (일관성)
    self._active_runners.pop(info.channel_id, None)
```

### CLI 세션 ID 파싱 실패

```python
# JSON 파싱 실패 → 일반 텍스트로 fallback
try:
    result_json = json.loads(stdout_text)
    cli_session_id = result_json.get("session_id")
except (json.JSONDecodeError, AttributeError):
    cli_session_id = None
    # 세션 ID 없으면 다음 실행 시 새 세션 생성 (--resume 없이)
```

### 파일 읽기/쓰기 실패

```python
# .id 파일 쓰기 실패 → 경고만, 동작에는 영향 없음
try:
    session_id_path.write_text(cli_session_id, encoding="utf-8")
except Exception as e:
    logger.warning("Failed to save session ID: %s", e)
```

### OpenCode 서버 오류

```python
# opencode run 반환값이 에러인 경우
# JSON 이벤트에서 error 타입 감지
for line in stdout_text.splitlines():
    event = json.loads(line.strip())
    if event.get("type") == "error":
        error_msg = event.get("error", {}).get("data", {}).get("message", "Unknown error")
        # Discord에 에러 메시지 전송
        post_message(discord_api, thread_id, bot_token,
            f"⚠️ **OpenCode 오류**: {error_msg}")
```

### Gateway 재시작 후 복구

- `_active_runners`는 메모리에서 손실
- runner 프로세스는 `start_new_session=True`로 계속 실행 중
- 완료 후 .id 파일은 정상적으로 작성됨
- 다음 메시지 수신 시 파일 시스템에서 복구 → 정상 동작

## 8. 요약: 변경 파일 & 핵심 diff

| 파일 | 변경 유형 | 주요 변경 |
|------|----------|----------|
| `adapter.py` | 수정 | `_external_sessions` 구조체, `_handle_message` 라우팅, `_spawn_runner`, `_watch_runner_completion`, `_dispatch_external_session_message`, `_build_runner_command`, 파일 기반 복구 |
| `claude_session_runner.py` | 수정 | `--resume` 인자, `--output-format json`, 세션 ID 파싱/저장 |
| `opencode_session_runner.py` | 수정 | `--session-id` 인자, `--format json`, 이벤트 스트림 파싱, 세션 ID 저장 |

**신규 파일**: 없음 (모두 기존 파일 수정)

## 9. 구현 순서 (권장)

1. **`claude_session_runner.py`**: `--resume` 인자 + `--output-format json` + 세션 ID 저장
2. **`opencode_session_runner.py`**: `--session-id` 인자 + `--format json` + 세션 ID 저장
3. **`adapter.py`**: `ExternalSessionInfo` + `_external_sessions` + `_dispatch_external_session_message` + `_spawn_runner` + `_watch_runner_completion`
4. **`adapter.py`**: `_handle_message`에서 기존 early return → `_dispatch_external_session_message` 호출로 변경
5. **`adapter.py`**: 파일 기반 복구 로직 추가
6. **테스트**: Discord에서 `/sc claude` → 메시지 전송 → 후속 메시지 → 세션 이어짐 확인
