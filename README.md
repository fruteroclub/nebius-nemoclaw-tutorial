# NemoClaw + Nebius Token Factory Tutorial

Run a sandboxed AI agent on **NemoClaw** (NVIDIA's OpenClaw runtime) backed by **Nebius Token Factory** inference — with a Telegram interface and a working demo that fetches and reasons about real product prices.

---

## What you'll build

A **Price Intel Assistant** — a Telegram bot that takes a product query, fetches live prices from MercadoLibre's public API, and returns a structured price-action recommendation. Inference runs remotely on Nebius Token Factory using `deepseek-ai/DeepSeek-V3.2`.

---

## Stack

| Layer | Technology |
|-------|-----------|
| Agent sandbox + lifecycle | [NemoClaw](https://docs.nvidia.com/nemoclaw/latest/) (NVIDIA, alpha) |
| Agent runtime | [OpenClaw](https://docs.openclaw.ai) (NVIDIA) |
| Inference | [Nebius Token Factory](https://docs.tokenfactory.nebius.com) — OpenAI-compatible endpoint |
| Model | `deepseek-ai/DeepSeek-V3.2` |
| Messaging channel | Telegram |
| Data source | MercadoLibre public search API |
| Host | AWS EC2 `m7i.xlarge` — Ubuntu 24.04 LTS (no GPU required) |

---

## Prerequisites

- AWS EC2 **m7i.xlarge** (4 vCPU, 15.3 GiB RAM) running Ubuntu 24.04 LTS
- [Nebius Token Factory](https://docs.tokenfactory.nebius.com) API key
- Telegram bot token from [@BotFather](https://t.me/BotFather)
- Node.js 22.16+ and npm 10+

---

## Guide

→ [Full step-by-step guide](guide-draft.md)

Covers: Docker install, NemoClaw install + onboarding wizard, Nebius Token Factory configuration, Telegram setup, agent verification, and known issues.

---

## Known issues

→ [issues.md](issues.md)

Documents bugs discovered during development, with workarounds and recommendations for the NemoClaw and Nebius teams.

---

## Quick start (after completing the guide)

```bash
# Connect to your sandbox
nemoclaw <your-sandbox-name> connect

# Fix the Telegram gateway bug (required after every onboard)
sed -i 's/"groupPolicy": "mentions"/"groupPolicy": "allowlist"/' ~/.openclaw/openclaw.json
nohup openclaw gateway > /tmp/gateway.log 2>&1 &

# Test inference
openclaw agent --agent main -m "hello"
```

Then DM your Telegram bot to start chatting.

---

## License

MIT
