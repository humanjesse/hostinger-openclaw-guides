# ElevenLabs Transcript Webhook for OpenClaw on Hostinger VPS

## Prerequisites

- Completed the [ElevenLabs Voice Agent Setup](hostinger-openclaw-elevenlabs-setup.md) guide (OpenClaw + ElevenLabs + Twilio working)
- SSH root access to the VPS
- Your ElevenLabs API key (from [elevenlabs.io/app/settings/api-keys](https://elevenlabs.io/app/settings/api-keys))

## Key Architecture Notes

When an ElevenLabs voice call ends (inbound or outbound), ElevenLabs can fire a **post-call webhook** containing the full conversation transcript, call metadata, and optional analysis results. This guide sets up a relay that catches those webhooks and delivers the transcript into the OpenClaw chat via `/hooks/wake` — the same pattern used by the [AgentMail webhook guide](hostinger-openclaw-agentmail-setup.md).

```
Call ends (inbound or outbound)
       ↓
ElevenLabs post-call webhook
       ↓
nginx (TLS + rate limit)
  └── /etc/nginx/openclaw.d/transcripts.conf
       ↓
127.0.0.1:8081 (Python transcript relay)
       ↓
docker exec → curl inside container
       ↓
127.0.0.1:18789 /hooks/wake (OpenClaw Gateway, token auth)
       ↓
OpenClaw chat: "Call ended. Transcript: ..."
```

This works for **both inbound and outbound calls** — any call handled by your ElevenLabs agent will have its transcript delivered to OpenClaw automatically.

---

## Step 1: Identify Your Container and Config Paths

```bash
# Find your OpenClaw container name
CONTAINER_NAME=$(docker ps --filter "name=openclaw" --format '{{.Names}}')
echo "Container: $CONTAINER_NAME"

# Find your deployment directory
DEPLOY_DIR=$(ls -d /docker/openclaw-*/)
echo "Deploy dir: $DEPLOY_DIR"

# Find your VPS hostname
hostname -f
# Example output: srv1370452.hstgr.cloud
```

> **Important:** All commands in this guide use `$CONTAINER_NAME` and `$DEPLOY_DIR`. If you open a new terminal session, re-run the two lines above before continuing.

## Step 2: Ensure Hooks Are Enabled in OpenClaw

The transcript relay delivers data via OpenClaw's `/hooks/wake` endpoint, which must be enabled with token authentication. If you've already followed the [AgentMail guide](hostinger-openclaw-agentmail-setup.md), hooks are already enabled — check and reuse the existing token:

```bash
# Check if hooks are already configured
docker exec $CONTAINER_NAME python3 -c "
import json
with open('/data/.openclaw/openclaw.json', 'r') as f:
    config = json.load(f)
hooks = config.get('hooks', {})
if hooks.get('enabled'):
    print(f'Hooks already enabled. Token: {hooks.get(\"token\", \"(not set)\")[:8]}...')
else:
    print('Hooks not enabled yet.')
"
```

If hooks are already enabled, retrieve the existing token:

```bash
# Get the token from the config
HOOK_TOKEN=$(docker exec $CONTAINER_NAME python3 -c "
import json
with open('/data/.openclaw/openclaw.json', 'r') as f:
    print(json.load(f).get('hooks', {}).get('token', ''))
")
echo "Hook token: ${HOOK_TOKEN:0:8}..."
```

If hooks are **not** enabled, set them up now:

```bash
HOOK_TOKEN=$(openssl rand -base64 32)
echo "Hook token: $HOOK_TOKEN"
echo "Save this value."

docker exec $CONTAINER_NAME python3 -c "
import json
config_path = '/data/.openclaw/openclaw.json'
with open(config_path, 'r') as f:
    config = json.load(f)
config.setdefault('hooks', {})
config['hooks']['enabled'] = True
config['hooks']['path'] = '/hooks'
config['hooks']['token'] = '$HOOK_TOKEN'
with open(config_path, 'w') as f:
    json.dump(config, f, indent=2)
print('Hooks enabled.')
"

docker restart $CONTAINER_NAME
```

## Step 3: Generate a Webhook Secret

Create a shared secret for authenticating incoming webhooks from ElevenLabs. This prevents unauthorized parties from injecting fake transcripts:

```bash
WEBHOOK_SECRET=$(openssl rand -hex 32)
echo "Webhook secret: $WEBHOOK_SECRET"
echo "Save this value — you'll enter it in the ElevenLabs dashboard (Step 7)."
```

## Step 4: Create the Nginx Snippet

Add a location block for the transcript webhook endpoint. This is a separate snippet from the AgentMail hooks:

```bash
cat > /etc/nginx/openclaw.d/transcripts.conf << 'EOF'
# ElevenLabs post-call transcript webhook
location /webhooks/elevenlabs/transcript {
    limit_req zone=hook_limit burst=10 nodelay;

    proxy_pass http://127.0.0.1:8081/webhooks/elevenlabs/transcript;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
EOF

nginx -t && systemctl reload nginx
```

## Step 5: Create the Transcript Relay

Create the environment file:

```bash
cat > /etc/elevenlabs-transcript-relay.env << EOF
OPENCLAW_TOKEN="$HOOK_TOKEN"
OPENCLAW_CONTAINER="$CONTAINER_NAME"
WEBHOOK_SECRET="$WEBHOOK_SECRET"
EOF
chmod 600 /etc/elevenlabs-transcript-relay.env
```

> **Important:** Verify the values expanded correctly: `cat /etc/elevenlabs-transcript-relay.env`. All three should have real values.

Create the relay script:

```bash
cat > /usr/local/bin/elevenlabs-transcript-relay.py << 'PYEOF'
#!/usr/bin/env python3
"""Relay ElevenLabs post-call transcript webhooks to OpenClaw."""
import hmac
import hashlib
import json
import os
import sys
import subprocess
from http.server import HTTPServer, BaseHTTPRequestHandler

OPENCLAW_TOKEN = os.environ.get("OPENCLAW_TOKEN", "")
OPENCLAW_CONTAINER = os.environ.get("OPENCLAW_CONTAINER", "")
WEBHOOK_SECRET = os.environ.get("WEBHOOK_SECRET", "")

if not OPENCLAW_CONTAINER:
    print("[transcript] FATAL: OPENCLAW_CONTAINER not set", flush=True)
    sys.exit(1)

if not OPENCLAW_TOKEN:
    print("[transcript] FATAL: OPENCLAW_TOKEN not set", flush=True)
    sys.exit(1)


def verify_webhook(body, headers):
    """Verify the webhook came from ElevenLabs using the shared secret."""
    if not WEBHOOK_SECRET:
        print("[transcript] WARNING: No WEBHOOK_SECRET set — skipping verification. "
              "This is INSECURE for production use.", flush=True)
        return True

    # ElevenLabs sends the secret in a custom header
    provided = headers.get("X-Webhook-Secret", "")
    if not provided:
        # Also check Authorization header as fallback
        auth = headers.get("Authorization", "")
        if auth.startswith("Bearer "):
            provided = auth[7:]

    if not provided:
        print("[transcript] No secret in request headers", flush=True)
        return False

    return hmac.compare_digest(provided, WEBHOOK_SECRET)


def format_transcript(payload):
    """Extract and format the transcript from the webhook payload."""
    conversation_id = payload.get("conversation_id", "unknown")
    agent_name = payload.get("agent_name", "Voice Agent")
    status = payload.get("status", "unknown")

    # Build transcript text from turn array
    transcript = payload.get("transcript", [])
    lines = []
    for turn in transcript:
        role = turn.get("role", "unknown").capitalize()
        message = turn.get("message", "")
        if message:
            lines.append(f"{role}: {message}")

    transcript_text = "\n".join(lines) if lines else "(no transcript available)"

    # Extract analysis if present
    analysis = payload.get("analysis", {})
    analysis_text = ""
    if analysis:
        eval_result = analysis.get("evaluation_criteria_results", {})
        data_results = analysis.get("data_collection_results", {})
        if eval_result:
            analysis_text += f"\nAnalysis: {json.dumps(eval_result)}"
        if data_results:
            analysis_text += f"\nCollected data: {json.dumps(data_results)}"

    # Format the message for OpenClaw
    summary = f"Call ended (conversation: {conversation_id}, status: {status})"
    if analysis_text:
        summary += analysis_text

    return f"{summary}\n\nTranscript:\n{transcript_text}"


class TranscriptHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(content_length)
        body_str = body.decode('utf-8')

        print(f"[transcript] Received POST {self.path}, body length: {len(body_str)}", flush=True)

        if not verify_webhook(body_str, self.headers):
            print("[transcript] Webhook verification FAILED", flush=True)
            self.send_response(401)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(b'{"error":"unauthorized"}')
            return

        print("[transcript] Webhook verified OK", flush=True)

        try:
            payload = json.loads(body_str)
            openclaw_text = format_transcript(payload)
        except Exception as e:
            print(f"[transcript] Parse error: {e}", flush=True)
            openclaw_text = f"Call transcript webhook received (parse error): {body_str[:500]}"

        openclaw_payload = json.dumps({
            "text": openclaw_text,
            "name": "ElevenLabs",
            "mode": "now"
        })

        # Bypass Express proxy — curl directly inside the container
        result = subprocess.run([
            'docker', 'exec', OPENCLAW_CONTAINER,
            'curl', '-s', '-X', 'POST', 'http://127.0.0.1:18789/hooks/wake',
            '-H', f'Authorization: Bearer {OPENCLAW_TOKEN}',
            '-H', 'Content-Type: application/json',
            '-d', openclaw_payload
        ], capture_output=True, text=True, timeout=30)

        print(f"[transcript] OpenClaw: {result.stdout}", flush=True)
        if result.stderr:
            print(f"[transcript] OpenClaw stderr: {result.stderr}", flush=True)

        self.send_response(200)
        self.send_header('Content-Type', 'application/json')
        self.end_headers()
        self.wfile.write(result.stdout.encode() if result.stdout else b'{"ok":true}')

    def log_message(self, format, *args):
        print(f"[transcript] {args[0]}", flush=True)


print("[transcript] ElevenLabs transcript relay listening on 127.0.0.1:8081", flush=True)
if not WEBHOOK_SECRET:
    print("[transcript] WARNING: Running without webhook verification!", flush=True)
HTTPServer(('127.0.0.1', 8081), TranscriptHandler).serve_forever()
PYEOF

chmod +x /usr/local/bin/elevenlabs-transcript-relay.py
```

## Step 6: Create the Systemd Service

```bash
cat > /etc/systemd/system/elevenlabs-transcript-relay.service << 'EOF'
[Unit]
Description=ElevenLabs Transcript Relay for OpenClaw
After=network.target docker.service

[Service]
Type=simple
EnvironmentFile=/etc/elevenlabs-transcript-relay.env
Environment=PYTHONUNBUFFERED=1
ExecStart=/usr/bin/python3 /usr/local/bin/elevenlabs-transcript-relay.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable elevenlabs-transcript-relay
systemctl start elevenlabs-transcript-relay
```

Verify the relay is running:

```bash
systemctl status elevenlabs-transcript-relay
ss -tlnp | grep 8081
# Should show python3 listening on 127.0.0.1:8081
```

## Step 7: Configure the Webhook in ElevenLabs

1. Go to the [ElevenLabs dashboard](https://elevenlabs.io/app/agents)
2. Open your agent's settings
3. Navigate to the **Webhooks** section (under Agent settings or workspace settings)
4. Add a **Transcription webhook**:
   - **URL:** `https://YOUR_HOSTNAME.hstgr.cloud/webhooks/elevenlabs/transcript`
   - **Secret / Authentication header:** Enter the webhook secret from Step 3

Replace `YOUR_HOSTNAME` with your actual VPS hostname (e.g., `srv1370452`).

> **Note:** ElevenLabs offers three types of post-call webhooks: **Transcription** (full transcript + analysis), **Audio** (MP3 recording), and **Call initiation failure**. For this guide, configure the **Transcription** webhook.

## Step 8: Test End-to-End

Watch the relay logs in one terminal:

```bash
journalctl -u elevenlabs-transcript-relay -f
```

In another terminal (or from the OpenClaw web UI), trigger a call. If you have the [outbound call skill](hostinger-openclaw-outbound-call-skill.md) installed, ask OpenClaw to call your test number. Otherwise, call your Twilio number from any phone.

When the call ends, you should see in the relay logs:

```
[transcript] Received POST /webhooks/elevenlabs/transcript, body length: XXXX
[transcript] Webhook verified OK
[transcript] OpenClaw: {"ok":true,"mode":"now"}
```

And in the OpenClaw web UI, the agent should receive a message like:

```
Call ended (conversation: conv_xxxx, status: done)

Transcript:
Agent: Hello! How can I help you today?
User: Hi, just testing the transcript feature.
Agent: Great, the transcript webhook is working!
```

> **Note:** There may be a delay of 10–30 seconds between the call ending and the webhook firing. ElevenLabs processes the transcript and runs any configured analysis before sending the webhook.

---

## Security Layers

1. **SSL/TLS** — Let's Encrypt via certbot
2. **Webhook secret verification** — Shared secret in request headers; rejects unauthorized senders
3. **Nginx rate limiting** — 2 req/s per IP with burst of 10 via `hook_limit` zone
4. **Nginx path restriction** — Only registered integration paths are proxied; everything else returns 404
5. **Payload size limit** — `client_max_body_size 2m` rejects oversized requests at the edge
6. **Localhost-only relay** — Relay binds to `127.0.0.1:8081`; not reachable from the internet
7. **Localhost-only gateway** — Port 18789 is not publicly exposed
8. **Bearer token auth** — OpenClaw hooks endpoint requires Authorization header
9. **Firewall** — `ufw` restricts inbound traffic to SSH, HTTP, and HTTPS only

### Production Hardening (Optional)

For commercial or high-risk deployments, also consider:

- **IP allowlisting**: Restrict `/webhooks/elevenlabs/` to ElevenLabs egress IP ranges in the nginx snippet
- **Transcript redaction**: Scrub sensitive data (credit card numbers, SSNs) from the transcript before forwarding to OpenClaw
- **Log rotation**: Configure `journald` or `logrotate` for relay logs to prevent disk exhaustion
- **Monitoring**: Watch `journalctl -u elevenlabs-transcript-relay` and nginx access logs for anomalies

## Troubleshooting

- **Webhook not firing after call ends**: Verify the webhook URL is correct in the ElevenLabs dashboard. Check that you configured a **Transcription** webhook (not Audio or Failure). There may be a 10–30 second delay while ElevenLabs processes the transcript.
- **Relay not receiving requests**: Check `systemctl status elevenlabs-transcript-relay` and `ss -tlnp | grep 8081`. Verify the nginx snippet loaded: `nginx -t && systemctl reload nginx`. Check `curl -sS http://127.0.0.1:8081/webhooks/elevenlabs/transcript -X POST -d '{}'` returns a response (even if 401).
- **401 from relay**: The webhook secret doesn't match. Verify the secret in `/etc/elevenlabs-transcript-relay.env` matches what you entered in the ElevenLabs dashboard.
- **Transcript shows "(no transcript available)"**: The call may still be in `processing` status. The webhook should only fire after processing completes, but if it fires early the transcript array may be empty.
- **OpenClaw doesn't show the transcript**: Check that hooks are enabled in `openclaw.json` and the `OPENCLAW_TOKEN` in the env file matches the token in the config. Check relay logs for `OpenClaw: {"ok":true}`.
- **Relay exits with FATAL error**: `OPENCLAW_TOKEN` or `OPENCLAW_CONTAINER` not set in `/etc/elevenlabs-transcript-relay.env`. Verify with `cat /etc/elevenlabs-transcript-relay.env`.
- **Nginx won't reload after adding snippet**: Check `nginx -t` output. A syntax error in any `.conf` file under `openclaw.d/` blocks the entire config.
- **Other integrations broke**: Ensure you didn't overwrite `/etc/nginx/sites-available/openclaw` — each guide only touches its own snippet file.
- **No relay logs in journalctl**: Ensure `PYTHONUNBUFFERED=1` is in the service file (prevents Python from buffering output).
