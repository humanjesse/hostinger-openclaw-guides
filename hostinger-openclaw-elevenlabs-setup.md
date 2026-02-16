# ElevenLabs Voice Agent Setup for OpenClaw on Hostinger VPS

## Prerequisites

- Hostinger VPS with 1-click OpenClaw deployment
- SSH root access to the VPS
- An ElevenLabs account (elevenlabs.io) with an Agents-capable plan
- (For phone calls) A Twilio account with a phone number

## Key Architecture Notes

ElevenLabs Agents handles all voice processing (speech-to-text, text-to-speech, turn-taking, phone integration). OpenClaw is the brain — it handles reasoning, memory, tools, and skills. They communicate over the standard OpenAI-compatible `/v1/chat/completions` endpoint.

```
Internet (ElevenLabs / Twilio)
       ↓
nginx (TLS + rate limit + auth passthrough)
  ├── /etc/nginx/openclaw.d/completions.conf  ← this guide
  ├── /etc/nginx/openclaw.d/hooks.conf         ← AgentMail guide
  └── /etc/nginx/openclaw.d/future.conf        ← next integration
       ↓
127.0.0.1:18789 (Docker localhost-only)
       ↓
OpenClaw Gateway (token auth)
```

**This endpoint is disabled by default in OpenClaw** and must be enabled in the config.

Hostinger's Docker deployment binds the OpenClaw gateway to loopback inside the container. To allow Docker port forwarding to reach it, the gateway must be set to bind on all interfaces (`"bind": "lan"`). Security is maintained by Docker's localhost-only port mapping and the gateway's token auth.

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

# Verify files exist
ls "$DEPLOY_DIR/docker-compose.yml"
ls "$DEPLOY_DIR/.env"
ls "$DEPLOY_DIR/data/.openclaw/openclaw.json"
```

> **Important:** All commands in this guide use `$CONTAINER_NAME` and `$DEPLOY_DIR`. If you open a new terminal session, re-run the two lines above before continuing.

## Step 2: Expose the Gateway Port via Docker

Edit the docker-compose file to expose port 18789 on localhost only. This uses a targeted insert rather than overwriting the file:

```bash
# Check if the port binding is already present
if grep -q '127.0.0.1:18789:18789' "$DEPLOY_DIR/docker-compose.yml"; then
    echo "Port 18789 already exposed — skipping."
else
    # Backup the original
    cp "$DEPLOY_DIR/docker-compose.yml" "$DEPLOY_DIR/docker-compose.yml.bak"
    echo "Backup saved to $DEPLOY_DIR/docker-compose.yml.bak"

    # Insert the localhost port binding after the existing PORT line
    sed -i '/\${PORT}:\${PORT}/a\      - "127.0.0.1:18789:18789"' "$DEPLOY_DIR/docker-compose.yml"

    echo "Port binding added. Verify the change:"
    grep -A 3 'ports:' "$DEPLOY_DIR/docker-compose.yml"
fi
```

Your `ports:` section should now look like this:

```yaml
    ports:
      - "${PORT}:${PORT}"
      - "127.0.0.1:18789:18789"
```

The `127.0.0.1:18789:18789` binding means only processes on the host (like nginx) can reach the gateway. It is not exposed to the internet.

Restart the container:

```bash
cd "$DEPLOY_DIR"
docker compose down
docker compose up -d
```

Verify the port is listening:

```bash
ss -tlnp | grep 18789
# Should show docker-proxy listening on 127.0.0.1:18789
```

## Step 3: Find Your Gateway Auth Token

The gateway auth token was set during Hostinger's initial OpenClaw deployment. You'll need it for ElevenLabs configuration and for testing the endpoint.

```bash
# Look for the token in the .env file
grep -iE 'token|secret|key' "$DEPLOY_DIR/.env"
```

Store the token for use in later steps:

```bash
GATEWAY_TOKEN="paste_your_token_value_here"
```

If no gateway token exists yet, generate one and add it:

```bash
GATEWAY_TOKEN=$(openssl rand -base64 32)
echo "OPENCLAW_GATEWAY_TOKEN=$GATEWAY_TOKEN" >> "$DEPLOY_DIR/.env"
echo "Generated gateway token: $GATEWAY_TOKEN"
```

> **Important:** Save this token — you'll enter it as the Bearer token in the ElevenLabs agent configuration (Step 6).

## Step 4: Enable the Chat Completions Endpoint

The `/v1/chat/completions` endpoint is disabled by default. Enable it, set the gateway to bind on all interfaces inside the container, and add Docker's network to trusted proxies:

```bash
docker exec $CONTAINER_NAME python3 -c "
import json
with open('/data/.openclaw/openclaw.json', 'r') as f:
    config = json.load(f)

# Enable the chat completions HTTP endpoint
config.setdefault('gateway', {}).setdefault('http', {}).setdefault('endpoints', {})['chatCompletions'] = {'enabled': True}

# Bind to all interfaces inside the container so Docker port forwarding works
config['gateway']['bind'] = 'lan'

# Trust Docker's default bridge network and localhost for proxy headers
# If you use a custom Docker network, check your subnet: docker network inspect bridge
config['gateway']['trustedProxies'] = ['127.0.0.1/32', '172.17.0.0/16']

with open('/data/.openclaw/openclaw.json', 'w') as f:
    json.dump(config, f, indent=2)

print('Done')
print('  chatCompletions:', config['gateway']['http']['endpoints']['chatCompletions'])
print('  bind:', config['gateway']['bind'])
print('  trustedProxies:', config['gateway']['trustedProxies'])
"
```

> **Note on `trustedProxies`:** The default Docker bridge subnet is `172.17.0.0/16`. If you use a custom Docker network, find your actual subnet with `docker network inspect bridge | grep Subnet` and adjust accordingly.

Restart and verify:

```bash
docker restart $CONTAINER_NAME
sleep 5

# Test the endpoint directly
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H "Authorization: Bearer $GATEWAY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"openclaw:main","messages":[{"role":"user","content":"say hello"}]}'
```

You should get back a JSON response with `"content":"Hello."` or similar.

## Step 5: Set Up Nginx with SSL

Install nginx and certbot if not already present:

```bash
apt update && apt install -y nginx certbot python3-certbot-nginx
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

This guide uses a **modular nginx config** so each OpenClaw integration (ElevenLabs, AgentMail, etc.) can be set up independently without conflicting. A shared base config owns the server block, and each integration drops a location snippet into `/etc/nginx/openclaw.d/`.

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

Now drop the ElevenLabs completions snippet:

```bash
cat > /etc/nginx/openclaw.d/completions.conf << 'EOF'
# ElevenLabs voice agent endpoint
location /v1/chat/completions {
    limit_req zone=api_limit burst=20 nodelay;
    limit_conn completions_conn 10;

    # Recommended: Restrict to ElevenLabs egress IPs
    # Check ElevenLabs documentation for current IP ranges, then uncomment and update:
    # allow 1.2.3.0/24;     # ElevenLabs range (replace with actual)
    # deny all;

    proxy_pass http://127.0.0.1:18789/v1/chat/completions;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 120s;
    proxy_buffering off;
}
EOF

nginx -t && systemctl reload nginx
```

> **Recommended: IP Allowlisting.** The `/v1/chat/completions` endpoint is open to the internet by default. Uncomment the `allow`/`deny` lines in `completions.conf` and add the ElevenLabs egress IP ranges from their documentation to restrict access to only their servers.

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

Test the public endpoint:

```bash
curl -sS https://YOUR_HOSTNAME.hstgr.cloud/v1/chat/completions \
  -H "Authorization: Bearer $GATEWAY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"openclaw:main","messages":[{"role":"user","content":"say hello"}]}'
```

## Step 6: Create the ElevenLabs Agent

1. Go to [elevenlabs.io/app/agents](https://elevenlabs.io/app/agents)
2. Create a new agent using the **Blank template**
3. Configure the **Agent** tab:
   - **System prompt**: Customize for your use case
   - **First message**: e.g., "Hello! How can I help you today?"
   - **Voice**: Pick from the voice library
   - **Language**: Set as needed
4. Scroll down to **LLM** settings:
   - Select **Chat Completions** from the dropdown
   - **Server URL**: `https://YOUR_HOSTNAME.hstgr.cloud/v1/chat/completions`
   - **Model ID**: `openclaw:main`
   - **API Key**: Select **Bearer token** and enter your gateway auth token (from Step 3)
   - **Reasoning Effort**: Set to **None** (important for low-latency voice)
5. Click **Preview** (top right) to test the agent
6. Try both text and **voice mode** (toggle in top right of preview panel)

## Step 7: Deploy — Inbound Phone Calls

Connect a Twilio number so people can call the agent:

1. In the ElevenLabs agent settings, go to the **Phone** deployment section
2. Import your Twilio credentials (Account SID + Auth Token)
3. Import or assign a Twilio phone number
4. Link the phone number to your agent

Calls to that number will now route to the ElevenLabs voice agent, which talks to your OpenClaw.

## Step 8: Deploy — Outbound Phone Calls

ElevenLabs supports outbound calling via Twilio:

1. Use the **Batch calls** feature in the ElevenLabs dashboard
2. Or trigger calls programmatically via the ElevenLabs API

Both use the same Twilio integration configured in Step 7.

## Step 9: Deploy — Website Widget

Embed the agent on any website with two lines of HTML:

```html
<elevenlabs-convai agent-id="YOUR_AGENT_ID"></elevenlabs-convai>
<script src="https://unpkg.com/@elevenlabs/convai-widget-embed" async type="text/javascript"></script>
```

Replace `YOUR_AGENT_ID` with the ID from the ElevenLabs dashboard.

## Step 10: Deploy — WhatsApp

ElevenLabs has a native WhatsApp integration in the deploy section of the agent settings. Follow the prompts in the dashboard to connect a WhatsApp Business number.

---

## Security Layers

1. **SSL/TLS** — Let's Encrypt via certbot
2. **Gateway token auth** — Bearer token required on all `/v1/chat/completions` requests
3. **Localhost-only port** — Docker exposes 18789 only on 127.0.0.1; not reachable from the internet
4. **Nginx rate limiting** — 5 req/s per IP with burst of 20; prevents DoS and cost amplification
5. **Nginx connection limiting** — 10 concurrent connections per IP; prevents connection exhaustion from streaming
6. **Nginx path restriction** — Only registered integration paths are proxied; everything else returns 404
7. **Payload size limit** — `client_max_body_size 2m` rejects oversized requests at the edge
8. **Security headers** — `X-Content-Type-Options`, `X-Frame-Options` to prevent content sniffing and framing
9. **LAN bind inside container** — Gateway binds to all interfaces inside Docker, but Docker's port mapping restricts external access
10. **Firewall** — `ufw` restricts inbound traffic to SSH, HTTP, and HTTPS only
11. **IP allowlisting** — Recommended: restrict `/v1/chat/completions` to ElevenLabs egress IPs

### Production Hardening (Optional)

For commercial or high-risk deployments, also consider:

- **Token hygiene**: Use a long random token (32+ bytes via `openssl rand -base64 32`), rotate periodically, never reuse other API keys
- **fail2ban**: Install and configure for nginx to auto-ban IPs with repeated 401s
- **Container resource limits**: Add `mem_limit` and `cpus` to docker-compose to prevent runaway containers
- **Log rotation**: Configure `journald` or `logrotate` to prevent disk exhaustion
- **Monitoring**: Watch `journalctl` and nginx access logs for anomalies

## Troubleshooting

- **`Connection reset by peer` on curl**: The completions endpoint is disabled by default. Verify `gateway.http.endpoints.chatCompletions.enabled` is `true` in `openclaw.json`.
- **`502 Bad Gateway` through nginx**: Port 18789 isn't exposed in docker-compose, or the container hasn't restarted after the config change. Check `ss -tlnp | grep 18789`.
- **`Empty reply from server`**: The gateway is binding to loopback inside the container. Set `gateway.bind` to `"lan"` and add Docker's bridge network (`172.17.0.0/16`) to `gateway.trustedProxies`. Check your actual subnet with `docker network inspect bridge | grep Subnet`.
- **ElevenLabs agent not responding**: Check that the Server URL, Model ID (`openclaw:main`), and Bearer token are correct in the LLM settings.
- **Slow voice responses**: Set Reasoning Effort to **None** in ElevenLabs. Consider using a faster model in OpenClaw (e.g., GPT-4.1 or Gemini Flash).
- **Container name changed after redeployment**: Run `docker ps --filter "name=openclaw" --format '{{.Names}}'` to get the current name.
- **`openclaw gateway restart` fails**: Use `docker restart $CONTAINER_NAME` instead (systemctl doesn't exist inside Docker).
- **Nginx won't reload after adding snippet**: Check `nginx -t` output — a syntax error in any `.conf` file under `openclaw.d/` will block the entire config. Verify with `ls /etc/nginx/openclaw.d/`
- **Other integration broke after this setup**: Ensure you didn't overwrite `/etc/nginx/sites-available/openclaw` — each guide should only touch its own snippet file
- **Docker-compose backup**: If the `sed` edit in Step 2 produced unexpected results, restore from `$DEPLOY_DIR/docker-compose.yml.bak`
