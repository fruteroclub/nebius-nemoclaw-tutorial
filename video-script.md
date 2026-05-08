# Video Script — NemoClaw + Nebius Token Factory Tutorial (Phase 1)

**Target runtime:** 10–15 minutes  
**Format:** Screen recording with voiceover — show terminal as you speak  
**Tone:** Developer-to-developer. Honest about gotchas. No filler.

---

## [INTRO] ~1:00

[Show: blank terminal on EC2 instance]

Hey — in this video I'm going to show you how to wire up NemoClaw, NVIDIA's sandboxed agent runtime, to use Nebius Token Factory as its inference backend, with a Telegram bot as the interface.

By the end of this video you'll have a live agent running on your own EC2 instance, serving inference remotely from Nebius, and responding to messages on Telegram — no GPU required.

A couple of things upfront: NemoClaw is alpha software, so expect rough edges. I ran into a few undocumented issues during setup that aren't in NVIDIA's docs — I'll call those out as we go so you don't get stuck.

Let's get into it.

---

## [STACK OVERVIEW] ~1:30

[Show: simple diagram or just speak to camera / static slide]

Before we touch the terminal, let me explain the three layers involved — because they're easy to conflate and it'll save you a lot of confusion.

**Layer one is NemoClaw.** This is NVIDIA's tool for deploying and managing OpenClaw assistants. It handles the sandbox lifecycle — building the container image, setting up networking, managing credentials, wiring up messaging channels. Think of it as the deployment and ops layer.

**Layer two is OpenClaw.** This is the actual agent runtime that runs inside the NemoClaw sandbox. It's a messaging gateway — it listens on Telegram, Discord, or Slack, receives messages, calls your inference provider, and sends the reply back. The agent logic lives here.

**Layer three is Nebius Token Factory.** This is an OpenAI-compatible inference endpoint — same API shape as OpenAI, different models, different pricing. We're using it to serve the model that actually thinks and responds. No GPU on our end — Nebius handles all of that.

These three layers talk to each other through a local proxy inside the sandbox. Your API key never leaves the host.

---

## [PREREQUISITES] ~0:45

[Show: terminal]

For this tutorial you need:

An AWS EC2 **m7i.xlarge** — that's 4 vCPUs and about 15 gigs of RAM. Ubuntu 24.04. No GPU needed.

A **Nebius Token Factory API key** — sign up at docs.tokenfactory.nebius.com. It's free to start.

And a **Telegram bot token** from BotFather. We'll use that later.

That's it. Let's start.

---

## [STEP 1: INSTALL DOCKER] ~1:00

[Show: terminal, run commands]

NemoClaw requires Docker to be installed and running before you even run its installer. That's not mentioned in NVIDIA's quickstart — you'll just get a confusing error if you skip it.

Install Docker Engine on Ubuntu using the official guide or the DigitalOcean community guide — links in the repo. Then verify it's running:

```bash
sudo systemctl status docker
```

And make sure your user can run Docker without sudo — the NemoClaw installer runs as your regular user:

```bash
newgrp docker
docker run --rm hello-world
```

[Show: hello-world output]

You should see "Hello from Docker!" — that's your green light.

---

## [STEP 2: INSTALL NEMOCLAW] ~1:30

[Show: terminal]

Now install NemoClaw. The NVIDIA docs show this command:

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
```

Don't run that directly — it'll fail. Here's why: piping curl into bash loses the terminal's TTY, and the installer requires interactive acceptance of third-party software terms. You'll get an error that says "Interactive third-party software acceptance requires a TTY."

[Show or read the error]

The fix is to pass the acceptance flag explicitly:

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash -s -- --yes-i-accept-third-party-software
```

[Show: installer running]

Once that finishes — second thing the docs don't mention — the onboarding wizard does not launch automatically. You have to run it yourself:

```bash
nemoclaw onboard
```

---

## [STEP 3: ONBOARDING WIZARD] ~3:30

[Show: terminal running nemoclaw onboard]

The wizard runs 8 steps. I'll walk through each one.

**Steps 1 and 2** — preflight checks and gateway startup — run automatically. You'll see Docker confirmed, and a note that local GPU inference is unavailable. That's expected — we're doing remote inference.

**Step 3 — inference provider.** This is the important one.

[Show: the inference options menu]

You'll see 7 options. We want **Option 3 — Other OpenAI-compatible endpoint.** That's how we connect Nebius.

Now, I want to flag something here. My first instinct was to use one of the NVIDIA Nemotron models on Nebius — that would make a nice story: NVIDIA's agent framework, NVIDIA's model, Nebius as the infrastructure. But it doesn't work.

All three Nemotron models currently available on Nebius are reasoning models. They return their responses in a field called `reasoning_content` instead of the standard `content` field. NemoClaw's Option 3 path hardcodes `REASONING=false` — it doesn't know to look in `reasoning_content`. The result is that the smoke check during onboarding fails, and tool calls return 400 errors.

This works fine through NVIDIA's own endpoint — Option 1 — where NemoClaw knows it's talking to a reasoning model. But through a third-party wrapper like Nebius, it doesn't have that signal. This is a known gap — we've flagged it to both teams.

So for this tutorial, we're using **DeepSeek V3.2** — a standard chat model that supports tool calls and passes the smoke check cleanly.

[Show: entering the base URL]

Enter the Nebius base URL:
```
https://api.tokenfactory.nebius.com/v1
```

Then your API key when prompted.

Then the model:
```
nvidia/Llama-3_1-Nemotron-Ultra-253B-v1
```

Wait — I just said we're using DeepSeek. In the written guide we reference the Ultra model because that's what readers should use for production quality. For this recording we're running DeepSeek V3.2 as the working model — same setup, different model ID. If you're following along, enter:

```
deepseek-ai/DeepSeek-V3.2
```

[Show: wizard validating the endpoint]

The wizard sends a live inference request to validate. You'll see a note about the Responses API streaming not being supported — Nebius uses the Chat Completions API instead. That's fine, NemoClaw handles it automatically.

**Step 4 — sandbox name.** Give your sandbox a name. I'll use `price-intel`.

**Step 5 — web search.** We don't need it — our agent will call MercadoLibre's API directly. Enter N.

**Step 6 — messaging channels.** This is where you add Telegram.

[Show: channel toggle UI]

Toggle Telegram on, then enter your BotFather token when prompted. You'll also be asked whether the bot should reply to all messages or only when @mentioned in groups. Choose @mentioned — it's cleaner for group chats.

One heads-up: this step is easy to accidentally skip. If you do, you can add the channel later with `nemoclaw <name> channels add telegram`.

**Step 7 — sandbox build.** This takes about 6 minutes the first time. NemoClaw is building a Docker image with your config baked in — model, provider, channel tokens, network policies. Subsequent builds are faster because layers are cached.

**Step 8 — policy presets.** Select Balanced, then toggle on `telegram` and `github`. The agent needs outbound access to both.

[Show: onboarding complete output]

Onboarding complete. Your sandbox is running.

---

## [STEP 4: FIX THE TELEGRAM BUG] ~1:00

[Show: terminal]

Before you test Telegram, there's one more thing — and this one bit me hard during setup, so I want to save you the trouble.

The NemoClaw wizard writes an invalid value into the OpenClaw config for the Telegram group policy. It writes `"mentions"` but OpenClaw only accepts `"open"`, `"disabled"`, or `"allowlist"`. The gateway process reads this config on startup, finds an invalid value, and exits immediately — silently. Your CLI inference still works because it bypasses the gateway, but Telegram is completely dead.

Connect to your sandbox:

```bash
nemoclaw price-intel connect
```

Then run this fix:

```bash
sed -i 's/"groupPolicy": "mentions"/"groupPolicy": "allowlist"/' ~/.openclaw/openclaw.json
nohup openclaw gateway > /tmp/gateway.log 2>&1 &
```

[Show: commands running]

This patches the config and starts the gateway in the background. Important: this fix doesn't survive a sandbox rebuild. If you ever re-run onboard, you'll need to re-apply it. We've filed this as a bug with the NemoClaw team.

---

## [STEP 5: VERIFY] ~1:00

[Show: terminal inside sandbox]

Now let's verify everything is working. Still inside the sandbox, run:

```bash
openclaw agent --agent main -m "hello"
```

Note: the NVIDIA quickstart shows a `--local` flag here. Drop it — that flag is blocked inside NemoClaw sandboxes. It bypasses the security layer and the sandbox rejects it.

[Show: agent responding]

You'll see a response from the agent — that's DeepSeek V3.2 on Nebius Token Factory, served through the NemoClaw gateway, responding to your message.

Now go to Telegram and DM your bot.

[Show: Telegram DM with bot responding]

There it is. Live inference, Telegram interface, no GPU, running on a standard EC2 instance.

---

## [WRAP UP + WHAT'S NEXT] ~0:45

[Show: GitHub repo]

Everything we just did is documented step by step in the guide in this repo — including all the gotchas, workarounds, and a full issues tracker with bug reports for both teams.

In the next video we'll build the actual Price Intel Assistant — the agent that takes a product query, hits MercadoLibre's public API for live prices, and comes back with a structured recommendation. That's where the agent code lives.

Links are in the description. If you run into issues, the issues tracker in the repo covers everything we found. See you in the next one.

---

## Timing estimate

| Section | Time |
|---------|------|
| Intro | ~1:00 |
| Stack overview | ~1:30 |
| Prerequisites | ~0:45 |
| Install Docker | ~1:00 |
| Install NemoClaw | ~1:30 |
| Onboarding wizard | ~3:30 |
| Fix Telegram bug | ~1:00 |
| Verify | ~1:00 |
| Wrap up | ~0:45 |
| **Total** | **~12:00** |
