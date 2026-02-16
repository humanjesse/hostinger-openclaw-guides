# AgentMail Webhook Setup for OpenClaw on Hostinger VPS

## Prerequisites

- Hostinger VPS with 1-click OpenClaw deployment
- SSH root access to the VPS
- An AgentMail account (agentmail.to)

## Key Architecture Notes

Hostinger's OpenClaw deployment uses a custom Express proxy (`/hostinger/server.mjs`) on port 53016 that proxies to the OpenClaw gateway on `127.0.0.1:18789`. This proxy's `express.json()` middleware consumes POST request bodies before `http-proxy-middleware` can forward them, which breaks webhook delivery. The solution is a lightweight Python webhook relay that bypasses the Express proxy entirely using `docker exec`.

```
AgentMail webhook
       ↓
nginx (TLS + rate limit + signature passthrough)
  └── /etc/nginx/openclaw.d/hooks.conf  ← this guide
       ↓
127.0.0.1:8080 (Python relay)
       ↓
docker exec → curl inside container
       ↓
127.0.0.1:18789 (OpenClaw Gateway, token auth)
```

> **Note:** `openclaw gateway restart` does not work in Docker (it calls systemctl which doesn't exist). Always use `docker restart` instead.

---

## Step 1: Identify Your Container and Config Paths

```bash
# Find your OpenClaw container name
CONTAINER_NAME=$(docker ps --filter "name=openclaw" --format '{{.Names}}')
echo "Container: $CONTAINER_NAME"

# Find your deployment directory
DEPLOY_DIR=$(ls -d /docker/openclaw-*/)
echo "Deploy dir: $DEPLOY_DIR"

# Verify config and compose files exist
ls "$DEPLOY_DIR/data/.openclaw/openclaw.json"
ls "$DEPLOY_DIR/docker-compose.yml"
ls "$DEPLOY_DIR/.env"
```

> **Important:** All commands in this guide use `$CONTAINER_NAME` and `$DEPLOY_DIR`. If you open a new terminal session, re-run the two lines above before continuing.

## Step 2: Save AgentMail API Key as an .env Variable

```bash
# Only add the key if it isn't already in .env (prevents duplicates on re-run)
grep -q '^AGENTMAIL_API_KEY=' "$DEPLOY_DIR/.env" \
  && echo "AGENTMAIL_API_KEY already set — edit $DEPLOY_DIR/.env manually to change it." \
  || echo 'AGENTMAIL_API_KEY=your_actual_key_here' >> "$DEPLOY_DIR/.env"
```

Then restart:

```bash
cd "$DEPLOY_DIR"
docker compose down
docker compose up -d
```

## Step 3: Install the AgentMail Skill in OpenClaw

Open the OpenClaw web UI and run:

```
clawdhub install agentmail
```

Or in the terminal:

```bash
docker exec -it $CONTAINER_NAME clawdhub install agentmail
```

The skill is just docs + scripts — the SDK isn't bundled. Install it separately:

```bash
docker exec $CONTAINER_NAME pip install --break-system-packages agentmail
```

## Step 4: Enable Hooks in OpenClaw Config

Generate a random token for hook authentication and save it — you'll need it again in Step 7:

```bash
HOOK_TOKEN=$(openssl rand -base64 32)
echo "Hook token: $HOOK_TOKEN"
echo "Save this value — you'll also need it in Step 7."
```

Edit the config to enable hooks **and** store the token:

```bash
python3 -c "
import json
config_path = '$DEPLOY_DIR/data/.openclaw/openclaw.json'
token = '$HOOK_TOKEN'
with open(config_path, 'r') as f:
    config = json.load(f)
config.setdefault('hooks', {})
config['hooks']['enabled'] = True
config['hooks']['path'] = '/hooks'
config['hooks']['token'] = token
with open(config_path, 'w') as f:
    json.dump(config, f, indent=2)
print('Done')
print(json.dumps(config['hooks'], indent=2))
"
```

Restart the container:

```bash
docker restart $CONTAINER_NAME
```

## Step 5: Install Nginx and Set Up SSL

```bash
apt update && apt install -y nginx certbot python3-certbot-nginx

# Find your Hostinger VPS hostname
hostname -f
# or check Hostinger panel for the .hstgr.cloud domain
```

Check for port 80 conflicts before continuing:

```bash
ss -tlnp | grep ':80 '
# If another process (not nginx) is using port 80, stop it or reconfigure it first
```

Remove the default nginx site so it doesn't conflict:

```bash
rm -f /etc/nginx/sites-enabled/default
```

This guide uses a **modular nginx config** so each OpenClaw integration (AgentMail, ElevenLabs, etc.) can be set up independently without conflicting. A shared base config owns the server block, and each integration drops a location snippet into `/etc/nginx/openclaw.d/`.

Create the snippet directory:

```bash
mkdir -p /etc/nginx/openclaw.d
```

Create the base config if it doesn't already exist (skip this block entirely if another OpenClaw guide already set it up):

```bash
if [ -f /etc/nginx/sites-available/openclaw ]; then
    echo "Base config already exists — skipping."
else
    cat > /etc/nginx/sites-available/openclaw << 'NGINX'
# Rate limit and connection limit zones — shared across all OpenClaw integrations
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;
limit_req_zone $binary_remote_addr zone=hook_limit:10m rate=2r/s;
limit_conn_zone $binary_remote_addr zone=completions_conn:10m;

server {
    listen 80;
    server_name YOUR_HOSTNAME.hstgr.cloud;

    # Security headers
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    # Reject oversized payloads
    client_max_body_size 2m;

    # Load all OpenClaw integration snippets
    include /etc/nginx/openclaw.d/*.conf;

    location / {
        return 404;
    }
}
NGINX
    ln -sf /etc/nginx/sites-available/openclaw /etc/nginx/sites-enabled/
    echo "Base config created."
fi
```

Now drop the AgentMail webhook snippet:

```bash
cat > /etc/nginx/openclaw.d/hooks.conf << 'EOF'
# AgentMail webhook relay
location /hooks/ {
    limit_req zone=hook_limit burst=10 nodelay;

    proxy_pass http://127.0.0.1:8080/hooks/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
EOF

nginx -t && systemctl reload nginx
```

Get an SSL certificate (skip if you already have one):

```bash
certbot --nginx -d YOUR_HOSTNAME.hstgr.cloud
```

Set up the firewall to restrict inbound traffic:

```bash
ufw allow 22/tcp    # SSH
ufw allow 80/tcp    # HTTP (certbot renewal)
ufw allow 443/tcp   # HTTPS
ufw --force enable
ufw status
```

## Step 6: Register the Webhook in AgentMail

Register a webhook to get the signing secret you'll need for the relay in Step 7. Use **either** the UI or the API — not both.

**Option A — AgentMail UI:**

1. Go to the AgentMail dashboard → Webhooks
2. Create a new webhook:
   - **URL:** `https://YOUR_HOSTNAME.hstgr.cloud/hooks/wake`
   - **Event:** `message.received`
3. Copy the signing secret from the response (starts with `whsec_`)

**Option B — API:**

```bash
curl -s -X POST https://api.agentmail.to/v0/webhooks \
  -H "Authorization: Bearer YOUR_AGENTMAIL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://YOUR_HOSTNAME.hstgr.cloud/hooks/wake",
    "event_types": ["message.received"]
  }' | jq .
```

Copy the signing secret from the response (starts with `whsec_`). You'll use it in the next step.

## Step 7: Create the Webhook Relay

Create the environment file with your tokens. The `HOOK_TOKEN` and `CONTAINER_NAME` variables are expanded from Steps 1 and 4. You must manually replace the `AGENTMAIL_SECRET` placeholder with the signing secret from Step 6:

```bash
cat > /etc/webhook-relay.env << EOF
OPENCLAW_TOKEN="$HOOK_TOKEN"
AGENTMAIL_SECRET="whsec_PASTE_YOUR_SIGNING_SECRET_HERE"
OPENCLAW_CONTAINER="$CONTAINER_NAME"
EOF
chmod 600 /etc/webhook-relay.env
```

> **⚠️ Important:** Edit `/etc/webhook-relay.env` and replace `whsec_PASTE_YOUR_SIGNING_SECRET_HERE` with the actual signing secret from Step 6. Without it, signature verification is skipped and any sender can trigger your relay.

Create the relay script:

```bash
cat > /usr/local/bin/webhook-relay.py << 'PYEOF'
#!/usr/bin/env python3
import hmac
import hashlib
import base64
import json
import os
import sys
import time
import subprocess
from http.server import HTTPServer, BaseHTTPRequestHandler

OPENCLAW_TOKEN = os.environ.get("OPENCLAW_TOKEN", "")
AGENTMAIL_SECRET = os.environ.get("AGENTMAIL_SECRET", "")
OPENCLAW_CONTAINER = os.environ.get("OPENCLAW_CONTAINER", "")

if not OPENCLAW_CONTAINER:
    print("[relay] FATAL: OPENCLAW_CONTAINER not set in environment", flush=True)
    sys.exit(1)

if not OPENCLAW_TOKEN:
    print("[relay] FATAL: OPENCLAW_TOKEN not set — OpenClaw will reject all requests", flush=True)
    sys.exit(1)

def verify_svix_signature(body, headers):
    if not AGENTMAIL_SECRET:
        print("[relay] WARNING: No AGENTMAIL_SECRET set — skipping verification. "
              "This is INSECURE for production use.", flush=True)
        return True

    msg_id = headers.get("Svix-Id", "")
    timestamp = headers.get("Svix-Timestamp", "")
    signature_header = headers.get("Svix-Signature", "")

    if not all([msg_id, timestamp, signature_header]):
        print("[relay] Missing Svix headers", flush=True)
        return False

    try:
        ts = int(timestamp)
        if abs(time.time() - ts) > 300:
            print(f"[relay] Timestamp too old, rejecting", flush=True)
            return False
    except ValueError:
        return False

    secret_str = AGENTMAIL_SECRET
    if secret_str.startswith("whsec_"):
        secret_str = secret_str[6:]

    try:
        secret_bytes = base64.b64decode(secret_str)
    except Exception as e:
        print(f"[relay] Failed to decode secret: {e}", flush=True)
        return False

    signed_content = f"{msg_id}.{timestamp}.{body}".encode()
    expected = base64.b64encode(
        hmac.new(secret_bytes, signed_content, hashlib.sha256).digest()
    ).decode()

    for sig in signature_header.split(" "):
        if sig.startswith("v1,"):
            if hmac.compare_digest(expected, sig[3:]):
                return True

    return False

class WebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(content_length)
        body_str = body.decode('utf-8')

        print(f"[relay] Received POST {self.path}, body length: {len(body_str)}", flush=True)

        if not verify_svix_signature(body_str, self.headers):
            print("[relay] Signature verification FAILED", flush=True)
            self.send_response(401)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(b'{"error":"invalid signature"}')
            return

        print("[relay] Signature verified OK", flush=True)

        try:
            payload = json.loads(body_str)
            data = payload.get("data", payload)
            message = data.get("message", data)

            sender = message.get("from_", ["unknown"])
            if isinstance(sender, list):
                sender = sender[0] if sender else "unknown"
            subject = message.get("subject", "No subject")
            text = message.get("text", message.get("body", ""))

            openclaw_text = f"New email from {sender}\nSubject: {subject}\n\n{text}"
        except Exception as e:
            print(f"[relay] Parse error: {e}", flush=True)
            openclaw_text = f"New email webhook received: {body_str[:500]}"

        openclaw_payload = json.dumps({
            "text": openclaw_text,
            "name": "AgentMail",
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

        print(f"[relay] OpenClaw: {result.stdout}", flush=True)
        if result.stderr:
            print(f"[relay] OpenClaw stderr: {result.stderr}", flush=True)

        self.send_response(200)
        self.send_header('Content-Type', 'application/json')
        self.end_headers()
        self.wfile.write(result.stdout.encode() if result.stdout else b'{"ok":true}')

    def log_message(self, format, *args):
        print(f"[relay] {args[0]}", flush=True)

print("[relay] Webhook relay listening on 127.0.0.1:8080", flush=True)
if not AGENTMAIL_SECRET:
    print("[relay] ⚠️  WARNING: Running without signature verification!", flush=True)
HTTPServer(('127.0.0.1', 8080), WebhookHandler).serve_forever()
PYEOF

chmod +x /usr/local/bin/webhook-relay.py
```

## Step 8: Create the Systemd Service

```bash
cat > /etc/systemd/system/webhook-relay.service << 'EOF'
[Unit]
Description=OpenClaw Webhook Relay
After=network.target docker.service

[Service]
Type=simple
EnvironmentFile=/etc/webhook-relay.env
Environment=PYTHONUNBUFFERED=1
ExecStart=/usr/bin/python3 /usr/local/bin/webhook-relay.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable webhook-relay
systemctl start webhook-relay
```

## Step 9: Test End-to-End

Send an email to your AgentMail inbox address and verify the chain:

```bash
journalctl -u webhook-relay -f
```

You should see: received → signature verified → forwarded → OpenClaw `{"ok":true,"mode":"now"}`.

---

## Security Layers

1. **SSL/TLS** — Let's Encrypt via certbot
2. **Svix signature verification** — HMAC-SHA256 with replay protection (5 min window)
3. **Bearer token auth** — OpenClaw hooks endpoint requires Authorization header
4. **Nginx rate limiting** — 2 req/s per IP with burst of 10; prevents webhook abuse and DoS
5. **Nginx path restriction** — Only registered integration paths are proxied; everything else returns 404
6. **Payload size limit** — `client_max_body_size 2m` rejects oversized requests at the edge
7. **Security headers** — `X-Content-Type-Options`, `X-Frame-Options` to prevent content sniffing and framing
8. **Localhost-only relay** — Relay binds to `127.0.0.1:8080`; not reachable from the internet even if nginx goes down
9. **Localhost-only gateway** — Port 18789 is not publicly exposed
10. **Firewall** — `ufw` restricts inbound traffic to SSH, HTTP, and HTTPS only

### Production Hardening (Optional)

For commercial or high-risk deployments, also consider:

- **IP allowlisting**: Restrict `/hooks/` to AgentMail/Svix IP ranges if known
- **Token hygiene**: Use a long random token (32+ bytes via `openssl rand -base64 32`), rotate periodically, never reuse other API keys
- **fail2ban**: Install and configure for nginx to auto-ban IPs with repeated 401s
- **Container resource limits**: Add `mem_limit` and `cpus` to docker-compose to prevent runaway containers
- **Log rotation**: Configure `journald` or `logrotate` for relay logs to prevent disk exhaustion
- **Monitoring**: Watch `journalctl -u webhook-relay` and nginx access logs for anomalies
- **docker exec alternatives**: For higher-security environments, consider a sidecar container or local HTTP bridge instead of `docker exec`

## Troubleshooting

- **`openclaw gateway restart` fails**: Use `docker restart $CONTAINER_NAME` instead (systemctl doesn't exist inside Docker)
- **Webhooks timeout through Hostinger proxy**: This is the Express body-parsing bug — use the relay
- **No relay logs in journalctl**: Ensure `PYTHONUNBUFFERED=1` is in the service file
- **Relay exits with `FATAL: OPENCLAW_TOKEN not set`**: The hook auth token from Step 4 is missing in `/etc/webhook-relay.env`. Re-generate one if needed and update both the env file and `openclaw.json`
- **Signature verification fails**: Quote values in env file if they contain special characters (`//`, `+`, etc.)
- **Container name wrong**: Run `docker ps --filter "name=openclaw" --format '{{.Names}}'` to confirm, then update `OPENCLAW_CONTAINER` in `/etc/webhook-relay.env`
- **Relay unreachable after nginx restart**: Verify relay is running with `systemctl status webhook-relay` and port is bound with `ss -tlnp | grep 8080`
- **Nginx won't reload after adding snippet**: Check `nginx -t` output — a syntax error in any `.conf` file under `openclaw.d/` will block the entire config. Verify with `ls /etc/nginx/openclaw.d/`
- **Other integration broke after this setup**: Ensure you didn't overwrite `/etc/nginx/sites-available/openclaw` — each guide should only touch its own snippet file
