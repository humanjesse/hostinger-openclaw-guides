# ElevenLabs Transcript Webhook for OpenClaw on Hostinger VPS

## Prerequisites

- Completed the [ElevenLabs Voice Agent Setup](hostinger-openclaw-elevenlabs-setup.md) guide (OpenClaw + ElevenLabs + Twilio working)
- SSH root access to the VPS
- Your ElevenLabs API key (from [elevenlabs.io/app/settings/api-keys](https://elevenlabs.io/app/settings/api-keys))

## Key Architecture Notes

When an ElevenLabs voice call ends (inbound or outbound), ElevenLabs can fire a **post-call webhook** containing the full conversation transcript, call metadata, and optional analysis results. This guide sets up a relay that catches those webhooks and delivers the transcript into OpenClaw via `/hooks/agent` — each call gets its own isolated session (keyed by conversation ID), and a summary is automatically surfaced in the main session via heartbeat.

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
127.0.0.1:18789 /hooks/agent (OpenClaw Gateway, token auth)
       ↓
Isolated session per call + summary in main session
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

The transcript relay delivers data via OpenClaw's `/hooks/agent` endpoint, which must be enabled with token authentication and the `allowRequestSessionKey` setting. If you've already followed the [AgentMail guide](hostinger-openclaw-agentmail-setup.md), hooks are already enabled — check and reuse the existing token:

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

If hooks are already enabled, retrieve the existing token and ensure `allowRequestSessionKey` is set:

```bash
# Get the token from the config
HOOK_TOKEN=$(docker exec $CONTAINER_NAME python3 -c "
import json
with open('/data/.openclaw/openclaw.json', 'r') as f:
    print(json.load(f).get('hooks', {}).get('token', ''))
")
echo "Hook token: ${HOOK_TOKEN:0:8}..."

# Enable allowRequestSessionKey (required for /hooks/agent with custom session keys)
docker exec $CONTAINER_NAME python3 -c "
import json
config_path = '/data/.openclaw/openclaw.json'
with open(config_path, 'r') as f:
    config = json.load(f)
if not config.get('hooks', {}).get('allowRequestSessionKey'):
    config['hooks']['allowRequestSessionKey'] = True
    with open(config_path, 'w') as f:
        json.dump(config, f, indent=2)
    print('allowRequestSessionKey enabled — restart container.')
else:
    print('allowRequestSessionKey already enabled.')
"
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
config['hooks']['allowRequestSessionKey'] = True
with open(config_path, 'w') as f:
    json.dump(config, f, indent=2)
print('Hooks enabled.')
"

docker restart $CONTAINER_NAME
```

## Step 3: Create the Webhook in ElevenLabs and Get the HMAC Secret

ElevenLabs uses **HMAC-SHA256** to sign webhook payloads — there is no option for a simple shared secret. You must create the webhook in the dashboard first to receive the signing secret.

1. Go to the [ElevenLabs dashboard](https://elevenlabs.io/app/agents)
2. Navigate to **Webhooks** (under workspace or agent settings)
3. Click **Create Webhook**:
   - **URL:** `https://YOUR_HOSTNAME.hstgr.cloud/webhooks/elevenlabs/transcript` (replace `YOUR_HOSTNAME` with your actual VPS hostname, e.g. `srv1370452`)
   - **Events:** Select **Post-call transcription**
   - **Authentication:** HMAC (this is the only option)
4. Click **Create** — ElevenLabs will generate and display an HMAC secret (starts with `wsec_`)
5. **Copy the secret immediately** — you won't be able to see it again

> **Note:** ElevenLabs offers three types of post-call webhooks: **Transcription** (full transcript + analysis), **Audio** (MP3 recording), and **Call initiation failure**. For this guide, select **Post-call transcription**.

Save the secret in a shell variable for use in the next steps:

```bash
WEBHOOK_SECRET="wsec_YOUR_SECRET_HERE"
```

Then assign the webhook to your agent: open your agent's settings, find the **Post-call webhook** dropdown, and select the webhook you just created.

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

> **Important:** Verify the values expanded correctly: `cat /etc/elevenlabs-transcript-relay.env`. All three should have real values — `WEBHOOK_SECRET` should start with `wsec_`.

Create the relay script:

```bash
cat > /usr/local/bin/elevenlabs-transcript-relay.py << 'PYEOF'
#!/usr/bin/env python3
"""Relay ElevenLabs post-call transcript webhooks to OpenClaw via /hooks/agent."""
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
    """Verify the webhook signature using ElevenLabs HMAC-SHA256."""
    if not WEBHOOK_SECRET:
        print("[transcript] WARNING: No WEBHOOK_SECRET set — skipping verification. "
              "This is INSECURE for production use.", flush=True)
        return True

    sig_header = headers.get("ElevenLabs-Signature", "")
    if not sig_header:
        sig_header = headers.get("elevenlabs-signature", "")
    if not sig_header:
        print("[transcript] No ElevenLabs-Signature header found", flush=True)
        return False

    # Parse header: format is "t=<timestamp>,v0=<signature>"
    parts = {}
    for item in sig_header.split(","):
        if "=" in item:
            key, val = item.split("=", 1)
            parts[key.strip()] = val.strip()

    timestamp = parts.get("t", "")
    signature = parts.get("v0", "")

    if not timestamp or not signature:
        print(f"[transcript] Malformed signature header: {sig_header}", flush=True)
        return False

    # Compute expected: HMAC-SHA256(secret_without_prefix, "timestamp.body")
    secret = WEBHOOK_SECRET
    if secret.startswith("wsec_"):
        secret = secret[5:]

    signed_payload = f"{timestamp}.{body}".encode()
    expected = hmac.new(
        secret.encode(), signed_payload, hashlib.sha256
    ).hexdigest()

    if hmac.compare_digest(expected, signature):
        return True

    # Fallback: try with full secret including wsec_ prefix
    expected_full = hmac.new(
        WEBHOOK_SECRET.encode(), signed_payload, hashlib.sha256
    ).hexdigest()

    if hmac.compare_digest(expected_full, signature):
        return True

    print("[transcript] Signature mismatch", flush=True)
    return False


def format_transcript(payload):
    """Extract and format the transcript from the webhook payload."""
    # ElevenLabs wraps data: {"type": "post_call_transcription", "data": {...}}
    data = payload.get("data", payload)

    conversation_id = data.get("conversation_id", "unknown")
    status = data.get("status", "unknown")

    # Build transcript text from turn array
    transcript = data.get("transcript", [])
    lines = []
    for turn in transcript:
        role = turn.get("role", "unknown").capitalize()
        message = turn.get("message", "")
        if message:
            lines.append(f"{role}: {message}")

    transcript_text = "\n".join(lines) if lines else "(no transcript available)"

    # Extract analysis if present
    analysis = data.get("analysis", {})
    analysis_text = ""
    if analysis:
        summary = analysis.get("transcript_summary", "")
        if summary:
            analysis_text += f"\nSummary: {summary}"
        call_successful = analysis.get("call_successful", "")
        if call_successful:
            analysis_text += f"\nCall outcome: {call_successful}"
        eval_result = analysis.get("evaluation_criteria_results", {})
        data_results = analysis.get("data_collection_results", {})
        if eval_result:
            analysis_text += f"\nEvaluation: {json.dumps(eval_result)}"
        if data_results:
            analysis_text += f"\nCollected data: {json.dumps(data_results)}"

    # Extract metadata
    metadata = data.get("metadata", {})
    duration = metadata.get("call_duration_secs", "")
    duration_text = f", duration: {duration}s" if duration else ""

    header = f"Call ended (conversation: {conversation_id}, status: {status}{duration_text})"
    if analysis_text:
        header += analysis_text

    return conversation_id, f"{header}\n\nTranscript:\n{transcript_text}"


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
            conversation_id, openclaw_text = format_transcript(payload)
        except Exception as e:
            print(f"[transcript] Parse error: {e}", flush=True)
            conversation_id = "unknown"
            openclaw_text = f"Call transcript webhook received (parse error): {body_str[:500]}"

        # Use /hooks/agent with a stable sessionKey per conversation
        openclaw_payload = json.dumps({
            "message": openclaw_text,
            "name": "ElevenLabs",
            "sessionKey": f"hook:elevenlabs:{conversation_id}",
            "wakeMode": "now"
        })

        result = subprocess.run([
            'docker', 'exec', OPENCLAW_CONTAINER,
            'curl', '-s', '-X', 'POST', 'http://127.0.0.1:18789/hooks/agent',
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
print("[transcript] Using /hooks/agent for isolated session + main session summary", flush=True)
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

## Step 7: Test End-to-End

Watch the relay logs in one terminal:

```bash
journalctl -u elevenlabs-transcript-relay -f
```

In another terminal (or from the OpenClaw web UI), trigger a call. If you have the [outbound call skill](hostinger-openclaw-outbound-call-skill.md) installed, ask OpenClaw to call your test number. Otherwise, call your Twilio number from any phone.

When the call ends, you should see in the relay logs:

```
[transcript] Received POST /webhooks/elevenlabs/transcript, body length: XXXX
[transcript] Webhook verified OK
[transcript] OpenClaw: {"ok":true}
```

The transcript is delivered to an isolated session keyed by the conversation ID. The main session will receive a summary automatically via heartbeat. In the OpenClaw web UI, you should see a message like:

```
Call ended (conversation: conv_xxxx, status: done, duration: 45s)
Summary: The caller tested the transcript feature.
Call outcome: success

Transcript:
Agent: Hello! How can I help you today?
User: Hi, just testing the transcript feature.
Agent: Great, the transcript webhook is working!
```

> **Note:** There may be a delay of 10–30 seconds between the call ending and the webhook firing. ElevenLabs processes the transcript and runs any configured analysis before sending the webhook.

---

## Security Layers

1. **SSL/TLS** — Let's Encrypt via certbot
2. **HMAC-SHA256 signature verification** — ElevenLabs signs each payload with a shared secret; the relay verifies the `ElevenLabs-Signature` header before processing
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

- **Webhook not firing after call ends**: Verify the webhook URL is correct in the ElevenLabs dashboard. Check that you selected **Post-call transcription** (not Audio or Failure). Ensure the webhook is assigned to your agent in the agent's post-call webhook setting. There may be a 10–30 second delay while ElevenLabs processes the transcript.
- **Relay not receiving requests**: Check `systemctl status elevenlabs-transcript-relay` and `ss -tlnp | grep 8081`. Verify the nginx snippet loaded: `nginx -t && systemctl reload nginx`. Check `curl -sS http://127.0.0.1:8081/webhooks/elevenlabs/transcript -X POST -d '{}'` returns a response (even if 401).
- **401 from relay (signature mismatch)**: The HMAC secret doesn't match. Verify the `WEBHOOK_SECRET` in `/etc/elevenlabs-transcript-relay.env` is the exact `wsec_...` value ElevenLabs gave you in Step 3. If you lost the secret, delete the webhook in the ElevenLabs dashboard and create a new one to get a fresh secret. After updating the env file, restart the relay: `systemctl restart elevenlabs-transcript-relay`.
- **Transcript shows "(no transcript available)"**: The call may still be in `processing` status. The webhook should only fire after processing completes, but if it fires early the transcript array may be empty.
- **OpenClaw doesn't show the transcript**: Check that hooks are enabled in `openclaw.json` and the `OPENCLAW_TOKEN` in the env file matches the token in the config. Check relay logs for `OpenClaw: {"ok":true}`. The transcript is delivered to an isolated session via `/hooks/agent` — the main session receives a summary automatically via heartbeat, which may take a moment to appear. Try refreshing the web UI.
- **"sessionKey is disabled" error in relay logs**: Set `hooks.allowRequestSessionKey = true` in `openclaw.json` and restart the container. See Step 2 for the exact command.
- **Relay exits with FATAL error**: `OPENCLAW_TOKEN` or `OPENCLAW_CONTAINER` not set in `/etc/elevenlabs-transcript-relay.env`. Verify with `cat /etc/elevenlabs-transcript-relay.env`.
- **Nginx won't reload after adding snippet**: Check `nginx -t` output. A syntax error in any `.conf` file under `openclaw.d/` blocks the entire config.
- **Other integrations broke**: Ensure you didn't overwrite `/etc/nginx/sites-available/openclaw` — each guide only touches its own snippet file.
- **No relay logs in journalctl**: Ensure `PYTHONUNBUFFERED=1` is in the service file (prevents Python from buffering output).
