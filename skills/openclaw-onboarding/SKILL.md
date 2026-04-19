---
name: openclaw-onboarding
description: Install and onboard OpenClaw inside WSL2 from zero to a running Gateway with an open Control UI dashboard. Use this skill whenever the user wants to install OpenClaw, set up OpenClaw, onboard OpenClaw, reinstall OpenClaw, get the OpenClaw dashboard running, configure an OpenClaw agent with an API key, or mentions phrases like "openclaw setup", "openclaw first time", "openclaw from scratch", "openclaw gateway not starting", or "openclaw dashboard port 18789". Trigger even when the user doesn't explicitly name every step — e.g., "I want to try openclaw" or "help me get openclaw running in wsl" should invoke this.
---

# OpenClaw Onboarding (conversational wrapper)

Your job in this skill is the **conversational layer** around an onboarding flow that is actually executed by the `openclaw-onboarder` subagent. You gather the inputs the subagent needs, hand them off, and relay the result. Do not run the install / config / gateway commands yourself — the subagent is the one with the execution context and the SubagentStart/Stop hooks that log each run.

## Step 1 — Collect inputs from the user

Before invoking the subagent, make sure you have:

1. **API provider** — one of Anthropic, OpenAI, Google, OpenRouter, xAI, Moonshot, Together, or any other provider listed under `openclaw onboard --help`. If the user hasn't named one, ask.
2. **API key** — for that provider.
3. **(Optional) preferred model ID** — e.g. `openai/gpt-5.4-mini`. If omitted, the onboarder uses the provider default. Do not invent a model ID; if the user says "mini" without a full ID and you're unsure of the canonical form, ask rather than guess, because an invalid model only fails at first chat as a provider 404.

If the user pastes the API key directly into the chat, warn them once that the key is now in the transcript and recommend rotating it at the provider's key management page after onboarding completes. Do not refuse to proceed — just warn and continue.

## Step 2 — Delegate to the onboarder subagent

Invoke the `openclaw-onboarder` subagent via the Agent tool with `subagent_type: "openclaw-onboarder"`. The prompt you pass must be a self-contained brief — the subagent has no access to this conversation's history and cannot ask follow-up questions mid-run.

Include in the brief:

- The provider and API key (so the subagent can run `openclaw onboard --<provider>-api-key '<KEY>'`).
- The chosen model ID if the user specified one.
- A reminder that this is a WSL2-only flow (skip Windows-native logic).
- An instruction to return, at the end, the dashboard URL, the gateway auth token, and any warnings the subagent encountered (unusual PATH, mDNS conflicts, existing install it had to reset, etc.) so you can relay them faithfully.

The `SubagentStart` and `SubagentStop` hooks in this plugin will record the run to `~/.claude/logs/openclaw-onboarding.log`; you do not need to do any logging yourself.

## Step 3 — Relay the result to the user

When the subagent returns, present to the user:

- **Dashboard URL** (normally `http://127.0.0.1:18789/`) — tell them they can open it in their Windows browser; WSL2 default networking forwards loopback.
- **Gateway auth token** — they'll paste it into the dashboard's auth prompt.
- **Alternative: TUI** — mention `openclaw tui` inside WSL for an in-terminal chat UI.
- **Security reminders** — API key is stored plaintext in `~/.openclaw/agents/main/agent/auth-profiles.json` (perms 600 but plaintext); if the key was pasted here, rotate it now.
- **Any warnings** the subagent surfaced.

If the subagent reports a failure, summarize what went wrong and what the user can try — do not silently re-run the same flow. Common failure modes to recognize in its output:

- "command not found" from a non-interactive shell → PATH problem; `~/.profile` fix may be needed.
- `Unrecognized key: X` on a config edit → schema mismatch; don't hand-guess config keys.
- TUI "No API key found for provider" → `--secret-input-mode` was `ref` instead of `plaintext`.
- `127.0.0.1:18789` never listens after start → usually a stale process; kill with `pkill -9 -f openclaw-gateway` before retrying.
- Bonjour/mDNS name-conflict spam in gateway logs → another OpenClaw (e.g., a leftover native-Windows instance) is advertising the same name. Fix by setting `gateway.discovery.mdns.mode` to `"off"` in `~/.openclaw/openclaw.json`. The correct key is `gateway.discovery.mdns.mode` (values `off | minimal | full`); `gateway.advertise.mdns` is rejected by the schema. A clean WSL-only install does not need this workaround.
