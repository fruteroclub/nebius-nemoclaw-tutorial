# Guide Draft — NemoClaw + Nebius Token Factory Tutorial

> Working draft. Steps are added as verified on the reference host (EC2 m7i.xlarge, Ubuntu 24.04). Do not publish until end-to-end run is complete.

---

## Model compatibility with NemoClaw + Nebius Token Factory

> ⚠️ **Flag for Nebius team:** This section documents a compatibility gap discovered during tutorial development. Please review before publishing.

NemoClaw's **Option 3 (Other OpenAI-compatible endpoint)** path hardcodes `NEMOCLAW_REASONING=false` in the sandbox build. It cannot detect whether the model behind the endpoint is a reasoning model — it always expects standard `choices[0].message.content` responses.

**All three NVIDIA Nemotron models currently available on Nebius Token Factory are reasoning models.** They return responses in `reasoning_content` instead of `content`, which causes two failures:

1. **NemoClaw's smoke check fails** — the check sends a constrained prompt with a 32-token budget. Reasoning models consume all tokens internally and return empty `content`, causing onboarding to abort.
2. **OpenClaw tool calls return 400** — OpenClaw's agentic runtime relies on tool calls. Nemotron reasoning models on Nebius do not support tool calls via the OpenAI-compatible wrapper.

**Root cause:** NemoClaw is designed to use NVIDIA reasoning models through NVIDIA's own inference endpoint (Option 1 — `build.nvidia.com`), where it sets `NEMOCLAW_REASONING=true` and handles the response format correctly. When Nemotron is served through a third-party OpenAI-compatible wrapper (Option 3), NemoClaw has no signal to switch modes.

**Affected models on Nebius Token Factory (as of 2026-05-06):**
- `nvidia/Llama-3_1-Nemotron-Ultra-253B-v1`
- `nvidia/nemotron-3-super-120b-a12b`
- `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B`

**Working models on Nebius Token Factory with NemoClaw Option 3:**
- `deepseek-ai/DeepSeek-V3.2` ✓ (used in this tutorial)
- `meta-llama/Llama-3.3-70B-Instruct` ✓
- `NousResearch/Hermes-4-70B` ✓
- Qwen3 Instruct variants (non-Thinking) — likely ✓

**Recommendation for Nebius team:** expose a non-reasoning inference mode for Nemotron models via the OpenAI-compatible endpoint, or document this limitation in the Token Factory + NemoClaw integration guide.

### Known issue: NemoClaw wizard writes invalid Telegram groupPolicy

> ⚠️ **Flag for NVIDIA/NemoClaw team.**

During `nemoclaw onboard`, the prompt "Reply only when @mentioned? [Y/n]" writes `"groupPolicy": "mentions"` into the sandbox config. OpenClaw does not accept `"mentions"` — valid values are `"open"`, `"disabled"`, and `"allowlist"`. The gateway process validates the config on startup and exits immediately on finding an invalid value, silently killing the Telegram bridge.

**Symptoms:** CLI inference works (embedded mode bypasses the gateway), but Telegram is unresponsive.

**Workaround** — patch the config and start the gateway manually after connecting to the sandbox:

```bash
sed -i 's/"groupPolicy": "mentions"/"groupPolicy": "allowlist"/' ~/.openclaw/openclaw.json
nohup openclaw gateway > /tmp/gateway.log 2>&1 &
```

> ⚠️ This patch does not survive a sandbox rebuild. If you rebuild, re-apply it.

**Permanent fix:** requires a change in NemoClaw's wizard or `generate-openclaw-config.py` to map the "mentions" answer to `"allowlist"` before baking it into the sandbox image.

---

## Prerequisites

Before starting, you need:

- An AWS EC2 **m7i.xlarge** instance (4 vCPU, 15.3 GiB RAM) running **Ubuntu 24.04 LTS**
- A **Nebius Token Factory** API key — sign up at https://docs.tokenfactory.nebius.com/
- A messaging bot token (Telegram recommended — create one via BotFather) **or** plan to use the browser UI at `localhost:18790`

---

## Step 1 — Install Docker Engine

NemoClaw requires Docker to be installed and running before its installer is invoked. Docker is not bundled — you must install it first.

Follow the official Docker Engine install for Ubuntu:
https://docs.docker.com/engine/install/ubuntu/

Or use DigitalOcean's community guide (tested on this setup, works on 24.04):
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-22-04

After installation, verify Docker is running:

```bash
sudo systemctl status docker
```

Then confirm your user can run Docker without `sudo` (apply group change):

```bash
newgrp docker
docker run --rm hello-world
```

Expected output ends with `Hello from Docker!` — if you see that, Docker is ready.

> **Verified:** Docker 29.4.2 on Ubuntu 24.04 LTS / EC2 m7i.xlarge — 2026-05-06

---

## Step 2 — Install NemoClaw

The NVIDIA docs show the installer as:

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
```

> ⚠️ **Undocumented gotcha:** piping directly into `bash` loses the TTY, causing the installer to fail with `[ERROR] Interactive third-party software acceptance requires a TTY.` The NVIDIA quickstart does not mention this. Pass the acceptance flag explicitly:

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash -s -- --yes-i-accept-third-party-software
```

> ⚠️ **Undocumented gotcha:** the installer does not automatically launch the onboarding wizard. After the script completes, run it manually:

```bash
nemoclaw onboard
```

> **Verified:** NemoClaw installed successfully on Ubuntu 24.04 LTS / EC2 m7i.xlarge — 2026-05-06

---

## Step 3 — Run the onboarding wizard with Nebius Token Factory

The wizard runs 8 steps. Here is the full sequence with the choices made for this tutorial:

**[1/8] Preflight checks** — runs automatically. Verifies Docker, DNS, container runtime, and available RAM. Expected output includes `✓ Docker is running` and `ⓘ Local NIM unavailable — no GPU detected` (normal for remote inference).

**[2/8] Starting OpenShell gateway** — runs automatically.

**[3/8] Configuring inference** — choose your inference provider:

```
Inference options:
  1) NVIDIA Endpoints
  2) OpenAI
  3) Other OpenAI-compatible endpoint  ← choose this for Nebius
  4) Anthropic
  5) Other Anthropic-compatible endpoint
  6) Google Gemini
  7) Install Ollama (Linux)
```

Select **3**, then enter:

- **Base URL:** `https://api.tokenfactory.nebius.com/v1`
- **API key:** your Nebius Token Factory API key
- **Model:** `deepseek-ai/DeepSeek-V3.2`

> ⚠️ **Why not Nemotron?** All three NVIDIA Nemotron models on Nebius Token Factory are reasoning models — they return responses in `reasoning_content` instead of `content`. NemoClaw's Option 3 path expects standard `content` responses and has no way to detect reasoning models from third-party wrappers. This causes the smoke check to fail and tool calls to return 400. Use `deepseek-ai/DeepSeek-V3.2` or another non-reasoning model. See the [compatibility notes](#model-compatibility-with-nemoclaw--nebius-token-factory) for the full breakdown.

> ℹ️ The wizard sends a live inference probe to validate your endpoint. You'll see:
> `ℹ Responses API streaming is missing required events: response.output_text.delta. Falling back to chat completions API.`
> This is expected — Nebius uses the Chat Completions API. NemoClaw will use `openai-completions` mode.

**[4/8] Sandbox name** — enter a name for your sandbox:

```
Sandbox name: price-intel
```

**[5/8] Web search** — enter `N` (the agent calls MercadoLibre's API directly).

**[6/8] Messaging channels** — toggle your desired channel. For Telegram, have your BotFather token ready.

> ⚠️ **UI gotcha:** it's easy to accidentally skip this step. If you skip it, you can add channels later with `nemoclaw price-intel policy-add`.

**[7/8] Creating sandbox** — the first build takes ~6 minutes. The image is ~608 MB compressed.

**[8/8] Policy presets** — select **Balanced**, then enable:
- `telegram` (for the messaging channel)
- `github` (for pushing to the tutorial repo)

Leave all other Balanced defaults enabled.

When onboarding completes, you'll see:

```
Sandbox      price-intel (Landlock + seccomp + netns)
Model        deepseek-ai/DeepSeek-V3.2 (Other OpenAI-compatible endpoint)

Run:         nemoclaw price-intel connect
Status:      nemoclaw price-intel status
Logs:        nemoclaw price-intel logs --follow
```

> **Verified:** Full onboarding completed on Ubuntu 24.04 LTS / EC2 m7i.xlarge — 2026-05-06

---

## Step 4 — Fix Telegram and verify inference

Connect to the sandbox:

```bash
nemoclaw price-intel connect
```

**Apply the Telegram gateway fix** — do this before testing anything. The NemoClaw wizard writes an invalid `groupPolicy` value (`"mentions"`) that crashes the gateway on startup, silently killing the Telegram bridge. CLI inference still works because it bypasses the gateway, but Telegram won't respond until this is patched:

```bash
sed -i 's/"groupPolicy": "mentions"/"groupPolicy": "allowlist"/' ~/.openclaw/openclaw.json
nohup openclaw gateway > /tmp/gateway.log 2>&1 &
```

> ⚠️ This fix does not survive a sandbox rebuild. Re-apply it every time you reconnect after a rebuild. See [ISSUE-001](issues.md#issue-001--telegram-wizard-writes-invalid-grouppolicy-value) for the full details and status.

Now send a test message:

```bash
openclaw agent --agent main -m "hello"
```

> ⚠️ **Undocumented gotcha:** the NVIDIA quickstart shows `openclaw agent --agent main --local -m "hello"` but the `--local` flag is blocked inside NemoClaw sandboxes — it bypasses the gateway's security protections. Drop `--local`.

Expected output:

```
🦞 OpenClaw 2026.4.24 — ...

│
◇
Hello! How can I assist you today?
```

The `EnvHttpProxyAgent is experimental` warnings are harmless — Node's proxy agent used for the `inference.local` routing.

If you see a response from the agent, Nebius Token Factory is serving inference correctly through the NemoClaw gateway. ✓

> **Verified:** 2026-05-06

To exit the sandbox:

```bash
/exit   # exit the chat
exit    # return to the host shell
```

---

## Step 5 — Build the Price Intel Assistant agent

_[ to be filled in during Phase 2 ]_

---

## Step 6 — Run the agent end-to-end

_[ to be filled in during Phase 3 ]_
