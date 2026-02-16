# Hostinger OpenClaw Integration Guides

Setup guides for connecting external services to [OpenClaw](https://github.com/openclaw) on a Hostinger VPS (1-click deployment).

## Guides

- **[AgentMail Webhook Setup](hostinger-openclaw-agentmail-setup.md)** — Receive emails via AgentMail webhooks and relay them to OpenClaw
- **[ElevenLabs Voice Agent Setup](hostinger-openclaw-elevenlabs-setup.md)** — Connect ElevenLabs voice agents (phone, web widget, WhatsApp) to OpenClaw as the LLM backend
- **[ElevenLabs Outbound Call Skill](hostinger-openclaw-outbound-call-skill.md)** — Build an OpenClaw skill that initiates outbound phone calls via ElevenLabs + Twilio ([ClawHub](https://clawhub.ai/humanjesse/outbound-call))
- **[ElevenLabs Transcript Webhook](hostinger-openclaw-elevenlabs-transcript-webhook.md)** — Automatically deliver call transcripts to OpenClaw when voice calls end

## Architecture

All guides use a shared modular nginx configuration. A base server block loads integration-specific snippets from `/etc/nginx/openclaw.d/`:

```
nginx (TLS + rate limiting)
  ├── openclaw.d/hooks.conf         ← AgentMail webhook relay
  ├── openclaw.d/completions.conf   ← ElevenLabs chat completions
  ├── openclaw.d/transcripts.conf   ← ElevenLabs transcript webhook
  └── openclaw.d/future.conf        ← add your own

skills/
  └── outbound-call/
        ├── SKILL.md      ← agent instructions
        └── call.py       ← ElevenLabs API wrapper

relay services (localhost only)
  ├── 127.0.0.1:8080      ← AgentMail webhook relay
  └── 127.0.0.1:8081      ← ElevenLabs transcript relay
```

Guides can be followed in any order. If the shared base nginx config already exists from a previous guide, it will be detected and skipped automatically.

## Prerequisites

- Hostinger VPS with the 1-click OpenClaw deployment
- SSH root access
- Accounts for the relevant services (AgentMail, ElevenLabs, Twilio)
