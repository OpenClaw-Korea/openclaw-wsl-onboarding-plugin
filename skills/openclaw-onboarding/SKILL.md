---
name: openclaw-onboarding
description: Install and onboard OpenClaw inside WSL2 from zero to a running Gateway with an open Control UI dashboard. Use this skill whenever the user wants to install OpenClaw, set up OpenClaw, onboard OpenClaw, reinstall OpenClaw, get the OpenClaw dashboard running, configure an OpenClaw agent with an API key, or mentions phrases like "openclaw setup", "openclaw first time", "openclaw from scratch", "openclaw gateway not starting", or "openclaw dashboard port 18789". Trigger even when the user doesn't explicitly name every step — e.g., "I want to try openclaw" or "help me get openclaw running in wsl" should invoke this.
---

# OpenClaw Onboarding (WSL2)

Take the user from nothing to a running OpenClaw Gateway with the Control UI dashboard reachable at `http://127.0.0.1:18789/`, all inside WSL2. At the end, append one JSONL line summarizing the run to `~/.claude/logs/openclaw-onboarding.log` so runs can be audited later.

Windows-native OpenClaw has known friction (mDNS conflicts with WSL instances, Scheduled Task permissions, slow Bonjour probing). This skill deliberately targets WSL only.

## Before you start

Collect what you need **without pulling the API key into the chat transcript**. The key must never appear in chat, in your tool call arguments, or in any tool output. It goes straight from a file on the user's disk into `openclaw onboard` via a shell substitution.

### Ask the structured questions with AskUserQuestion

Prefer the `AskUserQuestion` tool (Claude Code form input) so the user gets a clickable menu instead of free typing:

1. **Provider** — ask with options like:
   - `Anthropic` (Claude)
   - `OpenAI` (Recommended)
   - `Google` (Gemini)
   - Other (the user can type their provider name; match it to the corresponding `--<provider>-api-key` flag by running `openclaw onboard --help` if uncertain)
2. **Model choice** — ask:
   - `Keep provider default` (Recommended)
   - `Use a cheaper/smaller model` (then ask the user to type the exact model ID, e.g. `openai/gpt-5.4-mini`. Do not invent IDs — a typo only surfaces as a provider 404 at first chat.)

### Get the API key via a file, not the chat

Target path: the user's Windows **Downloads** folder. Downloads is the safest drop zone because it is:
- Not localized (unlike Desktop → 바탕화면/Escritorio/Bureau/…) so the path is stable across locales.
- Not typically synced by OneDrive by default, so the file won't be replicated to the cloud.
- Easy to reach from both File Explorer and any browser "Save as" dialog.

Do **not** ask the user to save to WSL home (`\\wsl.localhost\...`) or to Desktop — both create friction for non-technical users.

Tell the user, in plain language:

1. Open your Downloads folder in File Explorer (or press `Win+E` → Downloads).
2. Create a new plain text file named exactly `api.txt`.
3. Paste your API key as the **only** contents. One line, no quotes, no prefix.
4. Save and close. Tell me when it's done.

From WSL, detect the current Windows user and verify the file exists:

```bash
wsl -e bash -c 'WIN_USER=$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d "\r\n"); FILE="/mnt/c/Users/$WIN_USER/Downloads/api.txt"; test -s "$FILE" && echo "ok: $FILE" || echo "missing: $FILE"'
```

If missing, ask the user to double-check the filename (must be `api.txt`, not `api.txt.txt` — Windows hides known extensions by default), or whether they saved somewhere else. Do **not** ever `cat` the file; printing its contents would pull the key into your tool output and into the conversation transcript.

Remember that Windows path as `$API_FILE` for the next steps. You will pass it via shell substitution only — the key's bytes never leave the shell into your context.

Also capture the start time in seconds (`date +%s`) so you can compute `duration_s` for the final log line.

## Execution rules

- All commands go through `wsl -e bash -lc "..."` (or `-c` for one-shot). The host shell does not matter — pwsh, cmd, git-bash all work because `wsl.exe` is a Windows binary.
- Verify each step actually succeeded; don't assume success from the absence of output. Several OpenClaw subcommands print errors to stderr while exiting 0.
- Track every non-trivial event (existing install reset, PATH modified, stale gateway killed, mDNS conflicts noticed, etc.) in a running list. These become the `warnings` field in the final log line and part of what you relay to the user.

## Flow

### 1. Prerequisites

```bash
wsl -e bash -lc "node --version"
```

Node 24 is recommended, 22.14+ supported. If Node is missing, stop — do not try to install Node silently; the user's distro may have a preferred version manager. Tell the user to install Node in WSL first and write a `failure` log line with `phase: "prereq"`.

### 2. Install OpenClaw

Global npm install inside WSL. The PowerShell `irm | iex` installer is for native Windows and is not appropriate here.

```bash
wsl -e bash -lc "npm install -g openclaw"
```

Confirm the binary exists:

```bash
wsl -e bash -c "~/.npm-global/bin/openclaw --version"
```

If the user's npm prefix is elsewhere, adapt with `npm config get prefix` + `/bin`. Some setups have the global bin already on the default PATH and don't need step 3.

### 3. Fix PATH for non-interactive shells

Ubuntu's `~/.bashrc` starts with a guard that returns immediately for non-interactive shells:

```
case $- in
    *i*) ;;
      *) return;;
esac
```

A PATH export placed later in `.bashrc` therefore never applies to scripted invocations like `bash -lc "openclaw ..."`. Login-shell PATH belongs in `~/.profile`.

Append only if missing:

```bash
wsl -e bash -c "grep -q 'npm-global/bin' ~/.profile && echo ok || printf '\n# npm global packages\nif [ -d \"\$HOME/.npm-global/bin\" ] ; then\n    PATH=\"\$HOME/.npm-global/bin:\$PATH\"\nfi\n' >> ~/.profile"
```

Verify from a login shell:

```bash
wsl -e bash -lc "openclaw --version"
```

If this still fails, resolve the prefix and adjust before going further. Later steps rely on `openclaw` resolving in `bash -lc`. Record the result (`appended` vs `skipped`) for the log's `steps.path_fix` field.

### 4. Non-interactive onboarding

Flag choices and why they matter:

- `--non-interactive --accept-risk` — both required; the CLI refuses without `--accept-risk`.
- `--skip-daemon` — skip Gateway service install. On WSL there is no good analog and a plain foreground-style process is what we want.
- `--skip-channels` — Telegram/WhatsApp/etc. can be added later.
- `--skip-health` — without `--install-daemon` the health probe fails because no Gateway is running yet; we start it ourselves in step 6.
- `--secret-input-mode plaintext` — key is written into `~/.openclaw/agents/main/agent/auth-profiles.json`. `ref` mode stores a reference to an env var and makes the TUI error with `No API key found for provider "<provider>"` whenever the env var isn't exported in that shell. Plaintext avoids that footgun for a local single-user box. File perms are 600 but the key is unencrypted — remind the user at the end.

The provider flag is `--<provider>-api-key <key>`. For example `--openai-api-key`, `--anthropic-api-key`, `--google-api-key`, `--openrouter-api-key`, `--xai-api-key`. Run `openclaw onboard --help` when in doubt.

The key comes from `api.txt` via shell substitution — `$(cat "$API_FILE")` is evaluated inside the WSL shell, so the key's bytes stay on the user's machine and never enter your tool output or the conversation transcript.

```bash
wsl -e bash -lc 'WIN_USER=$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d "\r\n"); API_FILE="/mnt/c/Users/$WIN_USER/Downloads/api.txt"; openclaw onboard --non-interactive --accept-risk --<provider>-api-key "$(cat "$API_FILE")" --skip-daemon --skip-channels --skip-health --secret-input-mode plaintext'
```

Verify the auth profile has the key:

```bash
wsl -e bash -c "test -s ~/.openclaw/agents/main/agent/auth-profiles.json && echo ok"
```

Once the auth profile is confirmed, delete `api.txt` so the key does not linger on disk in two places:

```bash
wsl -e bash -lc 'WIN_USER=$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d "\r\n"); rm -f "/mnt/c/Users/$WIN_USER/Downloads/api.txt" && echo "api.txt removed"'
```

Add `"api.txt removed from Downloads"` to warnings so the user sees it in the final report.

### 5. (Optional) switch the default model

Only if the user specified a model. Onboarding sets a provider-appropriate default (e.g. `openai/gpt-5.4`). To change it, edit `~/.openclaw/openclaw.json` at `agents.defaults.model.primary` and `agents.defaults.models`:

```bash
wsl -e bash -lc "python3 - <<'PY'
import json, os
p = os.path.expanduser('~/.openclaw/openclaw.json')
c = json.load(open(p))
c.setdefault('agents',{}).setdefault('defaults',{})
mid = '<MODEL_ID>'
c['agents']['defaults']['models'] = {mid: {'alias': 'GPT'}}
c['agents']['defaults']['model'] = {'primary': mid}
json.dump(c, open(p, 'w'), indent=2)
print('model:', mid)
PY"
```

Run `openclaw doctor` to confirm the config validates. Doctor exits 0 on valid config even when it prints warnings.

### 6. Start the Gateway detached

If a Gateway is already running from a previous attempt, kill only that process — leave any TUI the user may have in another pane alone:

```bash
wsl -e bash -lc "pkill -9 -f openclaw-gateway 2>/dev/null; sleep 1"
```

If anything was killed, add `"stale gateway killed"` to warnings.

Start detached so nothing ties up a WSL terminal:

```bash
wsl -e bash -lc "nohup openclaw gateway run > ~/.openclaw/gateway.log 2>&1 < /dev/null & disown"
```

Poll for the listener. Node takes a few seconds to bind and an empty `gateway.log` combined with a bound port is normal — the process renames itself to `openclaw-gateway` and may not write to that log:

```bash
wsl -e bash -lc "for i in 1 2 3 4 5 6 7 8 9 10; do ss -tlnp 2>/dev/null | grep -q ':18789 ' && echo LISTENING && exit 0; sleep 1; done; echo NOT_LISTENING; tail -40 ~/.openclaw/gateway.log 2>/dev/null; ls -t ~/.openclaw/logs/*.json* 2>/dev/null | head -1 | xargs -r tail -40"
```

If `NOT_LISTENING`, write a `failure` log line with `phase: "gateway"` and the tailed error. Do not silently retry.

### 7. Read the gateway token

```bash
wsl -e bash -c "grep -oP '\"token\":\\s*\"\\K[^\"]+' ~/.openclaw/openclaw.json | head -1"
```

That's the Gateway auth token. The dashboard URL is `http://127.0.0.1:18789/`. On Windows + WSL2 with default networking, the user can open that URL in their Windows browser directly; localhost forwards. On mirrored networking, suggest `wslview http://127.0.0.1:18789/` inside WSL, or use the IP from `wsl hostname -I`.

## Write the log line

Compute `duration_s = now - start_time`. Then append exactly one JSONL line to `~/.claude/logs/openclaw-onboarding.log` summarizing this run. This log is the primary audit surface — write it even on failure.

Success example:

```json
{"ts":"2026-04-19T14:36:35Z","status":"success","duration_s":380,"model":"openai/gpt-5.4-mini","steps":{"install":"ok","path_fix":"appended","onboard":"ok","gateway":"listening"},"warnings":["stale gateway killed"]}
```

Failure example:

```json
{"ts":"2026-04-19T15:13:10Z","status":"failure","duration_s":67,"phase":"onboard","error":"--xai-api-key flag not recognized; run openclaw onboard --help","warnings":[]}
```

Write it like this (adjust `<JSONL>` to the single-line JSON object you built):

```bash
wsl -e bash -c "mkdir -p ~/.claude/logs && echo '<JSONL>' >> ~/.claude/logs/openclaw-onboarding.log"
```

Do not include the API key in the log. Keep `warnings` a plain array of short strings — messages like `"stale gateway killed"`, `"PATH appended to ~/.profile"`, `"mDNS name conflict detected"`, `"existing openclaw.json backed up to .bak"`.

## Tell the user what they got

Present in chat:

- **Dashboard URL**: `http://127.0.0.1:18789/` — openable in their Windows browser.
- **Gateway auth token** — they paste it into the dashboard's auth prompt.
- **Alternative**: `openclaw tui` inside WSL for an in-terminal chat UI.
- **Security reminders**:
  - API key is now stored plaintext in `~/.openclaw/agents/main/agent/auth-profiles.json` (perms 600 but unencrypted). The `api.txt` file the user created in Downloads has been deleted.
  - The key never entered this chat transcript — it went from Downloads/api.txt straight into `openclaw onboard` via a shell substitution. So no rotation is needed on that basis.
  - Gateway binds to loopback by default. Do not change `gateway.bind` to `lan` or `auto` without understanding the implications.
- **Warnings** captured during the run, in plain language.

## Troubleshooting

If the user runs into an issue after onboarding (or you hit one mid-flow), these are the failure modes we've confirmed:

- `openclaw: command not found` in a script but works in the user's terminal → PATH problem, redo step 3.
- `Unrecognized key: X` on a config edit → schema mismatch. Run `openclaw doctor --fix`, or find the right key by grepping the installed package's `runtime-schema-*.js` for the nearest match. Don't guess.
- `--<x>-api-key cannot be used with --secret-input-mode ref unless <X>_API_KEY is set in env` → switch to `--secret-input-mode plaintext` (what this skill uses), or export the env var before invoking.
- Gateway starts but `127.0.0.1:18789` never listens → usually a stale process still bound. `pkill -9 -f openclaw-gateway`, then restart.
- TUI shows `No API key found for provider "<provider>"` even though onboarding succeeded → config ended up in `ref` mode. Re-run step 4 with `--secret-input-mode plaintext`.
- Bonjour/mDNS name-conflict spam in gateway logs (`[bonjour] gateway name conflict resolved; newName="... (2)"`) → another OpenClaw (e.g., a leftover native-Windows instance) is advertising the same name. Set `gateway.discovery.mdns.mode` to `"off"` in `~/.openclaw/openclaw.json` (values: `off | minimal | full`). The correct key is `gateway.discovery.mdns.mode`; `gateway.advertise.mdns` is rejected as "Unrecognized key". A clean WSL-only install does not need this workaround.
