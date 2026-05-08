# Issues Tracker — nemoclaw-openclaw-tutorial

Bugs and gaps discovered during tutorial development. Flag to the relevant teams before or alongside publishing.

---

## NVIDIA / NemoClaw

### ISSUE-001 — Telegram wizard writes invalid groupPolicy value
**Severity:** High — silently breaks Telegram bridge  
**Discovered:** 2026-05-06  
**Status:** Filed — [NVIDIA/NemoClaw#3274](https://github.com/NVIDIA/NemoClaw/issues/3274)  

**Description:** During `nemoclaw onboard`, the prompt "Reply only when @mentioned? [Y/n]" writes `"groupPolicy": "mentions"` into the OpenClaw config. OpenClaw only accepts `"open"`, `"disabled"`, or `"allowlist"`. The gateway process validates config on startup and exits immediately on finding the invalid value, killing the Telegram bridge with no clear error message.

**Symptoms:** CLI inference works; Telegram is unresponsive; `openclaw agent` logs "gateway closed (1006 abnormal closure)".

**Workaround:**
```bash
sed -i 's/"groupPolicy": "mentions"/"groupPolicy": "allowlist"/' ~/.openclaw/openclaw.json
nohup openclaw gateway > /tmp/gateway.log 2>&1 &
```

**Expected fix:** Map `"mentions"` → `"allowlist"` in the wizard or in `generate-openclaw-config.py`.

---

### ISSUE-002 — Install script fails when piped through bash (TTY error)
**Severity:** Medium — blocks installation, no clear guidance in docs  
**Discovered:** 2026-05-06  
**Status:** Filed — [NVIDIA/NemoClaw#3275](https://github.com/NVIDIA/NemoClaw/issues/3275)  

**Description:** The documented install command `curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash` fails with `[ERROR] Interactive third-party software acceptance requires a TTY.` Piping curl into bash loses the TTY. The quickstart does not mention this or the workaround.

**Workaround:**
```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash -s -- --yes-i-accept-third-party-software
```

**Expected fix:** Document the flag in the quickstart, or detect the missing TTY and print the workaround automatically.

---

### ISSUE-003 — Installer does not auto-launch onboarding wizard
**Severity:** Low — minor UX friction  
**Discovered:** 2026-05-06  
**Status:** Filed — [NVIDIA/NemoClaw#3276](https://github.com/NVIDIA/NemoClaw/issues/3276)  

**Description:** After the install script completes, the onboarding wizard does not launch automatically. Users must run `nemoclaw onboard` manually. The quickstart implies the wizard runs as part of the install.

**Expected fix:** Either auto-launch `nemoclaw onboard` at the end of the install script, or add a clear post-install prompt.

---

### ISSUE-004 — Quickstart documents `--local` flag that is blocked inside sandboxes
**Severity:** Medium — causes confusing errors for users following the docs  
**Discovered:** 2026-05-06  
**Status:** Filed — [NVIDIA/NemoClaw#3277](https://github.com/NVIDIA/NemoClaw/issues/3277)  

**Description:** The NemoClaw quickstart shows `openclaw agent --agent main --local -m "hello"` as the verification command. The `--local` flag is explicitly blocked inside NemoClaw sandboxes with an error message explaining it bypasses security protections. The quickstart should use `openclaw agent --agent main -m "hello"` instead.

**Expected fix:** Remove `--local` from the quickstart example.

---

### ISSUE-005 — Credential not re-prompted in resume mode after credential reset
**Severity:** Medium — leaves sandbox in broken state after reset  
**Discovered:** 2026-05-06  
**Status:** Filed — [NVIDIA/NemoClaw#3278](https://github.com/NVIDIA/NemoClaw/issues/3278)  

**Description:** After running `nemoclaw credentials reset compatible-endpoint`, running `nemoclaw onboard --resume` skips the inference provider setup step and does not re-prompt for the API key. The sandbox rebuild then fails with an authentication error because the credential is missing.

**Workaround:** Run `nemoclaw onboard` (full wizard, not resume) after a credential reset.

**Expected fix:** Resume mode should detect missing credentials and re-prompt for them.

---

## Nebius Token Factory

### ISSUE-006 — All Nebius Nemotron models are incompatible with NemoClaw Option 3
**Severity:** High — blocks the natural NVIDIA-on-Nebius integration path  
**Discovered:** 2026-05-06  
**Status:** Filed — [NVIDIA/NemoClaw#3279](https://github.com/NVIDIA/NemoClaw/issues/3279) + [nebius/api#211](https://github.com/nebius/api/issues/211)  

**Description:** All three NVIDIA Nemotron models available on Nebius Token Factory (`Llama-3_1-Nemotron-Ultra-253B-v1`, `nemotron-3-super-120b-a12b`, `NVIDIA-Nemotron-3-Nano-30B-A3B`) are reasoning models. They return responses in `reasoning_content` with empty `content`. NemoClaw's Option 3 (OpenAI-compatible endpoint) hardcodes `NEMOCLAW_REASONING=false` and checks `choices[0].message.content`, causing:

1. Smoke check failure during onboarding (32-token budget exhausted by reasoning, `content` is null/empty)
2. 400 errors when OpenClaw attempts tool calls (not supported by reasoning models via the OpenAI-compatible wrapper)

**Root cause:** NemoClaw is designed to use Nemotron through NVIDIA's own endpoint (Option 1), where it sets `NEMOCLAW_REASONING=true`. Third-party OpenAI-compatible wrappers (Option 3) receive no reasoning-mode signal.

**Workaround:** Use a non-reasoning model — `deepseek-ai/DeepSeek-V3.2`, `meta-llama/Llama-3.3-70B-Instruct`, or `NousResearch/Hermes-4-70B`.

**Expected fix (Nebius):** Expose a non-reasoning inference mode for Nemotron models via the OpenAI-compatible endpoint, or document the incompatibility clearly in the NemoClaw integration guide.

**Expected fix (NVIDIA):** Allow `NEMOCLAW_REASONING` to be set as a runtime flag during Option 3 onboarding, or detect reasoning-model responses and handle them gracefully.

---

*Last updated: 2026-05-08*
