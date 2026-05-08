# NemoClaw + Nebius Token Factory

Run a sandboxed AI agent on **NemoClaw** (NVIDIA's OpenClaw runtime) backed by **Nebius Token Factory** inference, with a Telegram interface. No GPU required.

This tutorial walks you through standing up a **Price Intel Assistant** — a Telegram bot that fetches live product prices from MercadoLibre and returns a structured recommendation, powered by `deepseek-ai/DeepSeek-V3.2` on Nebius.

---

## Stack

| Layer | Technology |
|-------|-----------|
| Agent sandbox + lifecycle | [NemoClaw](https://docs.nvidia.com/nemoclaw/latest/) (NVIDIA, alpha) |
| Agent runtime | [OpenClaw](https://docs.openclaw.ai) (NVIDIA) |
| Inference | [Nebius Token Factory](https://docs.tokenfactory.nebius.com) — OpenAI-compatible |
| Model | `deepseek-ai/DeepSeek-V3.2` |
| Channel | Telegram |
| Data | MercadoLibre public search API |
| Host | AWS EC2 `m7i.xlarge` — Ubuntu 24.04 LTS |

---

## Prerequisites

- AWS EC2 **m7i.xlarge** (4 vCPU, 15 GiB RAM), Ubuntu 24.04 LTS
- [Nebius Token Factory](https://docs.tokenfactory.nebius.com) API key
- Telegram bot token from [@BotFather](https://t.me/BotFather)
- Node.js 22.16+ and npm 10+

---

## Setup

### 1 — Install Docker

NemoClaw requires Docker running before its installer starts — it's not bundled.

```bash
# Install Docker Engine (Ubuntu 24.04)
# Follow: https://docs.docker.com/engine/install/ubuntu/

# Add your user to the docker group and verify
newgrp docker
docker run --rm hello-world
```

> `Hello from Docker!` means you're good to go.

---

### 2 — Install NemoClaw

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash -s -- --yes-i-accept-third-party-software
```

> **Note:** the standard `curl ... | bash` command in NVIDIA's docs fails with a TTY error when piped. Use the flag above. The installer also does not auto-launch the wizard — run it manually after install.

```bash
nemoclaw onboard
```

---

### 3 — Run the onboarding wizard

The wizard has 8 steps. Key choices:

**Inference provider → Option 3 (Other OpenAI-compatible endpoint)**

```
Base URL:  https://api.tokenfactory.nebius.com/v1
API key:   <your Nebius Token Factory key>
Model:     deepseek-ai/DeepSeek-V3.2
```

> **Note:** Nebius serves all Nemotron models as reasoning-only, which is currently incompatible with NemoClaw's Option 3 path. Use `deepseek-ai/DeepSeek-V3.2` or another standard chat model. See [issues.md](issues.md) for the full breakdown.

**Sandbox name:** pick a name, e.g. `price-intel`

**Web search:** `N`

**Messaging channels:** toggle Telegram, enter your BotFather token

**Policy presets:** Balanced + enable `telegram` and `github`

The first sandbox build takes ~6 minutes.

---

### 4 — Fix Telegram and verify

Connect to your sandbox:

```bash
nemoclaw price-intel connect
```

Apply this fix before anything else — the NemoClaw wizard writes an invalid Telegram config value that silently crashes the gateway:

```bash
sed -i 's/"groupPolicy": "mentions"/"groupPolicy": "allowlist"/' ~/.openclaw/openclaw.json
nohup openclaw gateway > /tmp/gateway.log 2>&1 &
```

> **Note:** this fix doesn't survive a sandbox rebuild — re-apply it each time. Bug filed with NVIDIA ([ISSUE-001](issues.md#issue-001--telegram-wizard-writes-invalid-grouppolicy-value)).

Test inference:

```bash
openclaw agent --agent main -m "hello"
```

> Drop `--local` if you see it in older docs — that flag is blocked inside NemoClaw sandboxes.

You should get a response. Then DM your Telegram bot — if it replies, the full stack is working.

---

## Files

- [`guide.md`](guide.md) — full step-by-step guide with all context and explanations
- [`issues.md`](issues.md) — bugs found during development, filed with NVIDIA and Nebius

---

## License

MIT
