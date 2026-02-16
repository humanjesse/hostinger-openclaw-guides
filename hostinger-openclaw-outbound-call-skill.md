# ElevenLabs Outbound Call Skill for OpenClaw on Hostinger VPS

## Prerequisites

- Completed the [ElevenLabs Voice Agent Setup](hostinger-openclaw-elevenlabs-setup.md) guide (OpenClaw + ElevenLabs + Twilio all working for inbound calls)
- SSH root access to the VPS
- A Twilio phone number imported into ElevenLabs (from the previous guide's Step 7)
- Your ElevenLabs API key (from [elevenlabs.io/app/settings/api-keys](https://elevenlabs.io/app/settings/api-keys))

## Key Architecture Notes

This guide builds an OpenClaw **skill** — a self-contained module that teaches the agent a new capability. When a user says "call +15551234567" in the OpenClaw chat, the agent recognizes the intent, runs a Python script, and the ElevenLabs API initiates a real phone call via Twilio. The voice agent on the call uses the same OpenClaw brain as inbound calls.

```
User: "Call +15551234567"
       ↓
OpenClaw agent (recognizes intent via SKILL.md)
       ↓
python3 skills/outbound-call/call.py +15551234567
       ↓
POST https://api.elevenlabs.io/v1/convai/twilio/outbound-call
  Headers: xi-api-key
  Body: { agent_id, agent_phone_number_id, to_number }
       ↓
ElevenLabs initiates Twilio call
       ↓
Recipient's phone rings
       ↓
Voice agent uses OpenClaw as LLM backend (same as inbound)
```

**The outbound call uses the exact same agent configuration as inbound calls** — same voice, same Custom LLM (your OpenClaw endpoint), same prompt, same tools, same knowledge base. You can optionally override the first message or pass dynamic variables per call.

## Quick Install via ClawHub

This skill is published on ClawHub at [clawhub.ai/humanjesse/outbound-call](https://clawhub.ai/humanjesse/outbound-call). If you'd rather skip the manual setup (Steps 4–6), install it directly:

```bash
docker exec -it $CONTAINER_NAME clawdhub install humanjesse/outbound-call
```

Then jump to **Step 7** to add your environment variables. The rest of this guide walks through building the skill from scratch, which is useful if you want to understand how it works or customize it.

---

## Step 1: Identify Your Container and Config Paths

```bash
# Find your OpenClaw container name
CONTAINER_NAME=$(docker ps --filter "name=openclaw" --format '{{.Names}}')
echo "Container: $CONTAINER_NAME"

# Find your deployment directory
DEPLOY_DIR=$(ls -d /docker/openclaw-*/)
echo "Deploy dir: $DEPLOY_DIR"
```

> **Important:** All commands in this guide use `$CONTAINER_NAME` and `$DEPLOY_DIR`. If you open a new terminal session, re-run the two lines above before continuing.

Verify the ElevenLabs integration from the previous guide is still working:

```bash
# Confirm the container is running
docker ps --filter "name=openclaw" --format '{{.Names}} {{.Status}}'

# Confirm the chat completions endpoint responds
GATEWAY_TOKEN=$(grep 'OPENCLAW_GATEWAY_TOKEN\|GATEWAY_TOKEN' "$DEPLOY_DIR/.env" | head -1 | cut -d= -f2-)
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H "Authorization: Bearer $GATEWAY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"openclaw:main","messages":[{"role":"user","content":"say hello"}]}' | head -c 200
echo
```

You should get a JSON response with a message. If not, revisit the [ElevenLabs setup guide](hostinger-openclaw-elevenlabs-setup.md).

## Step 2: Get Your ElevenLabs Agent ID and Phone Number ID

The outbound call API requires two IDs: your **agent ID** and the **phone number ID** (not the phone number itself — it's an internal ElevenLabs identifier assigned when you imported your Twilio number).

Get the agent ID from the ElevenLabs dashboard URL. Navigate to your agent at [elevenlabs.io/app/agents](https://elevenlabs.io/app/agents) — the URL will look like:

```
https://elevenlabs.io/app/agents/agent_xxxxxxxxxxxxxxxxxxxx/edit
```

Store it:

```bash
AGENT_ID="paste_your_agent_id_here"
echo "Agent ID: $AGENT_ID"
```

Get the phone number ID via the API:

```bash
ELEVENLABS_API_KEY="paste_your_elevenlabs_api_key_here"

curl -sS "https://api.elevenlabs.io/v1/convai/phone-numbers" \
  -H "xi-api-key: $ELEVENLABS_API_KEY" | python3 -m json.tool
```

This returns a list of phone numbers imported into ElevenLabs. Find the one matching your Twilio number and copy its `phone_number_id` field:

```json
{
  "phone_numbers": [
    {
      "phone_number_id": "pn_xxxxxxxxxxxxxxxx",
      "phone_number": "+16893199614",
      "label": "...",
      "agent_id": "agent_xxxx..."
    }
  ]
}
```

Store it:

```bash
PHONE_NUMBER_ID="paste_your_phone_number_id_here"
echo "Phone number ID: $PHONE_NUMBER_ID"
```

> **Note:** If the list is empty, your Twilio number hasn't been imported into ElevenLabs yet. Go to your agent's **Phone** deployment section in the dashboard and import it first (covered in the previous guide's Step 7).

## Step 3: Test the Outbound Call API Directly

Before building the skill, verify the API works with a manual curl call. Use a phone number you control for testing:

```bash
TEST_NUMBER="+1XXXXXXXXXX"  # Replace with your own number

curl -sS -X POST "https://api.elevenlabs.io/v1/convai/twilio/outbound-call" \
  -H "xi-api-key: $ELEVENLABS_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"agent_id\": \"$AGENT_ID\",
    \"agent_phone_number_id\": \"$PHONE_NUMBER_ID\",
    \"to_number\": \"$TEST_NUMBER\"
  }" | python3 -m json.tool
```

A successful response looks like:

```json
{
  "success": true,
  "message": "Call initiated successfully",
  "conversation_id": "conv_xxxxxxxxxxxxxxxx",
  "callSid": "CAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

Your phone should ring within a few seconds. The voice agent on the call will use your OpenClaw as the brain.

> **Important:** If you get a 422 error, double-check your `agent_phone_number_id` — this is the most common mistake. It's not the phone number itself, it's the ElevenLabs internal ID from Step 2.

## Step 4: Create the Skill Directory

```bash
SKILL_DIR="$DEPLOY_DIR/data/.openclaw/workspace/skills/outbound-call"

if [ -d "$SKILL_DIR" ]; then
    echo "Skill directory already exists — skipping."
else
    mkdir -p "$SKILL_DIR"
    echo "Created: $SKILL_DIR"
fi

ls -la "$SKILL_DIR"
```

Check what other skills are already installed for reference:

```bash
docker exec $CONTAINER_NAME ls /data/.openclaw/workspace/skills/ 2>/dev/null || echo "No skills installed yet"
```

## Step 5: Write the Call Script

Create the Python script that the OpenClaw agent will execute to trigger outbound calls:

```bash
cat > "$SKILL_DIR/call.py" << 'PYEOF'
#!/usr/bin/env python3
"""Trigger an outbound phone call via ElevenLabs Conversational AI + Twilio."""
import sys
import os
import json
import re
import urllib.request
import urllib.error

AGENT_ID = os.environ.get("ELEVENLABS_AGENT_ID", "")
API_KEY = os.environ.get("ELEVENLABS_API_KEY", "")
PHONE_NUMBER_ID = os.environ.get("ELEVENLABS_PHONE_NUMBER_ID", "")

API_URL = "https://api.elevenlabs.io/v1/convai/twilio/outbound-call"


def make_call(to_number: str, first_message: str = "", context: str = "") -> dict:
    """Place an outbound call to the given phone number.

    Args:
        to_number: Phone number in E.164 format (+1XXXXXXXXXX for US).
        first_message: Optional custom greeting for this call.
        context: Optional context to pass as a dynamic variable.
    """
    if not API_KEY:
        return {"error": "ELEVENLABS_API_KEY not set. Add it to your .env file."}
    if not AGENT_ID:
        return {"error": "ELEVENLABS_AGENT_ID not set. Add it to your .env file."}
    if not PHONE_NUMBER_ID:
        return {"error": "ELEVENLABS_PHONE_NUMBER_ID not set. Add it to your .env file."}

    # Validate E.164 format (US numbers: +1 followed by 10 digits)
    if not re.match(r'^\+1\d{10}$', to_number):
        return {"error": f"Invalid US phone number: {to_number}. Use +1XXXXXXXXXX format (E.164)."}

    payload = {
        "agent_id": AGENT_ID,
        "agent_phone_number_id": PHONE_NUMBER_ID,
        "to_number": to_number,
    }

    # Add optional overrides if provided
    client_data = {}
    if first_message:
        client_data["conversation_config_override"] = {
            "agent": {"first_message": first_message}
        }
    if context:
        client_data["dynamic_variables"] = {"call_context": context}
    if client_data:
        payload["conversation_initiation_client_data"] = client_data

    data = json.dumps(payload).encode()
    req = urllib.request.Request(API_URL, data=data, headers={
        "xi-api-key": API_KEY,
        "Content-Type": "application/json",
    })

    try:
        with urllib.request.urlopen(req) as resp:
            result = json.loads(resp.read())
            # ElevenLabs may return HTTP 200 with an error in the body
            if not result.get("success", True) or "error" in result:
                return {"error": result.get("error", result.get("message", "Unknown API error"))}
            return {
                "success": True,
                "conversation_id": result.get("conversation_id", ""),
                "call_sid": result.get("callSid", ""),
                "message": result.get("message", "Call initiated"),
            }
    except urllib.error.HTTPError as e:
        body = e.read().decode()
        return {"error": f"ElevenLabs API error (HTTP {e.code}): {body}"}
    except urllib.error.URLError as e:
        return {"error": f"Network error: {e.reason}"}


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(json.dumps({"error": "Usage: call.py <phone_number> [first_message] [context]"}))
        sys.exit(1)

    to = sys.argv[1]
    first_msg = sys.argv[2] if len(sys.argv) > 2 else ""
    ctx = sys.argv[3] if len(sys.argv) > 3 else ""

    result = make_call(to, first_msg, ctx)
    print(json.dumps(result, indent=2))
    sys.exit(0 if result.get("success") else 1)
PYEOF

chmod +x "$SKILL_DIR/call.py"
echo "Created: $SKILL_DIR/call.py"
```

## Step 6: Write the SKILL.md

The SKILL.md file tells OpenClaw when and how to use the skill. Its frontmatter declares metadata and its body is injected into the agent's context:

```bash
cat > "$SKILL_DIR/SKILL.md" << 'SKILLEOF'
---
name: outbound-call
description: Make outbound phone calls via ElevenLabs voice agent and Twilio
metadata:
  clawdbot:
    requires:
      env:
        - ELEVENLABS_API_KEY
        - ELEVENLABS_AGENT_ID
        - ELEVENLABS_PHONE_NUMBER_ID
    primaryEnv: ELEVENLABS_API_KEY
---

# Outbound Call

Place outbound phone calls using the ElevenLabs voice agent with Twilio. The voice agent on the call uses OpenClaw as its brain — same as inbound calls.

## When to use

When the user asks you to:
- Call someone or phone someone
- Make a phone call
- Dial a number
- Ring someone
- Place a call to a number

## How to use

Run the call script with a phone number in E.164 format:

```bash
python3 skills/outbound-call/call.py +1XXXXXXXXXX
```

With an optional custom first message (what the agent says when the recipient picks up):

```bash
python3 skills/outbound-call/call.py +1XXXXXXXXXX "Hi John, I'm calling about your appointment tomorrow."
```

With optional call context (passed as a dynamic variable to the agent):

```bash
python3 skills/outbound-call/call.py +1XXXXXXXXXX "Hi, this is a quick follow-up call." "Customer requested callback about billing issue #4521"
```

## Phone number format

- US numbers: +1 followed by 10 digits, e.g., +15551234567
- If the user gives a number like 555-123-4567 or (555) 123-4567, reformat it to +15551234567
- Always confirm the formatted number with the user before placing the call

## Rules

- ALWAYS confirm the phone number with the user before placing a call
- NEVER place a call without explicit user consent
- Report the result back to the user (conversation ID on success, error details on failure)
- If the call fails, explain the error and suggest fixes
SKILLEOF

echo "Created: $SKILL_DIR/SKILL.md"
```

Verify both files exist:

```bash
ls -la "$SKILL_DIR/"
```

You should see:

```
call.py
SKILL.md
```

## Step 7: Add Environment Variables

The call script reads three environment variables. Add them to the OpenClaw `.env` file:

```bash
# Add ELEVENLABS_API_KEY if not already present
grep -q '^ELEVENLABS_API_KEY=' "$DEPLOY_DIR/.env" \
  && echo "ELEVENLABS_API_KEY already set." \
  || echo "ELEVENLABS_API_KEY=$ELEVENLABS_API_KEY" >> "$DEPLOY_DIR/.env"

# Add ELEVENLABS_AGENT_ID
grep -q '^ELEVENLABS_AGENT_ID=' "$DEPLOY_DIR/.env" \
  && echo "ELEVENLABS_AGENT_ID already set." \
  || echo "ELEVENLABS_AGENT_ID=$AGENT_ID" >> "$DEPLOY_DIR/.env"

# Add ELEVENLABS_PHONE_NUMBER_ID
grep -q '^ELEVENLABS_PHONE_NUMBER_ID=' "$DEPLOY_DIR/.env" \
  && echo "ELEVENLABS_PHONE_NUMBER_ID already set." \
  || echo "ELEVENLABS_PHONE_NUMBER_ID=$PHONE_NUMBER_ID" >> "$DEPLOY_DIR/.env"
```

> **Important:** If you're using shell variables from earlier steps, verify they expanded correctly:

```bash
grep 'ELEVENLABS_' "$DEPLOY_DIR/.env"
```

You should see three lines with actual values (not empty strings). If any are blank, edit the file manually:

```bash
nano "$DEPLOY_DIR/.env"
```

## Step 8: Enable First Message Override in ElevenLabs

Enable the first message override so OpenClaw can pass a custom greeting when initiating outbound calls:

1. Go to your agent in the [ElevenLabs dashboard](https://elevenlabs.io/app/agents)
2. Navigate to the **Security** tab
3. Enable the **First message** override toggle
4. Optionally enable **System prompt** override if you want per-call prompt customization

> **Important:** Without this, outbound calls will **immediately hang up** if OpenClaw passes a custom first message. ElevenLabs rejects the override and drops the call. This is the most common issue when first testing the skill.

## Step 9: Restart and Test

Restart the container to pick up the new environment variables and skill:

```bash
cd "$DEPLOY_DIR"
docker compose down
docker compose up -d
sleep 5
```

Verify the skill files are visible inside the container:

```bash
docker exec $CONTAINER_NAME ls /data/.openclaw/workspace/skills/outbound-call/
```

Test the call script directly from inside the container to confirm environment variables are loaded:

```bash
# Dry run — check that env vars are set (this will fail with a validation error since
# the number is fake, but it confirms the script loads and env vars are available)
docker exec $CONTAINER_NAME python3 /data/.openclaw/workspace/skills/outbound-call/call.py +10000000000
```

You should see an error about the API call failing (invalid number), not about missing environment variables. If you see `ELEVENLABS_API_KEY not set`, the `.env` file isn't being loaded — verify the `env_file` directive in `docker-compose.yml`.

Now test with a real number you control:

```bash
docker exec $CONTAINER_NAME python3 /data/.openclaw/workspace/skills/outbound-call/call.py +1XXXXXXXXXX
```

Replace `+1XXXXXXXXXX` with your actual phone number. Your phone should ring within seconds.

Finally, test through the OpenClaw web UI:

1. Open the OpenClaw web interface
2. Start a new chat session
3. Type: "Can you call +1XXXXXXXXXX?" (your test number)
4. The agent should:
   - Recognize the call intent (from SKILL.md)
   - Confirm the number with you
   - Run `call.py` with the formatted number
   - Report the result (conversation ID or error)
5. Your phone rings and the voice agent uses OpenClaw as its brain

---

## Security Layers

1. **ElevenLabs API key auth** — The `xi-api-key` header authenticates all outbound call requests; without it the API rejects the call
2. **User consent enforcement** — SKILL.md instructs the agent to always confirm the number before calling; prevents accidental or unauthorized calls
3. **E.164 validation** — `call.py` validates phone number format before hitting the API; rejects malformed input
4. **Environment variable isolation** — API keys live in `.env` on the host, not hardcoded in scripts; secrets aren't committed to version control
5. **Container boundary** — The skill runs inside the OpenClaw Docker container; no direct internet access except through the ElevenLabs API call
6. **Existing infrastructure security** — All layers from the ElevenLabs setup guide still apply (TLS, gateway token auth, localhost-only ports, rate limiting, firewall)

### Production Hardening (Optional)

For commercial or high-risk deployments, also consider:

- **Call rate limiting**: Add a daily/hourly call limit in `call.py` (e.g., track calls in a local file and reject after N per hour)
- **Number allowlisting**: Restrict outbound calls to a predefined list of approved numbers
- **Call logging**: Log every outbound call (number, timestamp, conversation ID) to a file for audit trails
- **ElevenLabs usage alerts**: Set up billing alerts in ElevenLabs to catch unexpected call volume
- **Separate API key**: Use a dedicated ElevenLabs API key for outbound calls with restricted permissions, rather than your main account key

## Dynamic Variables and Per-Call Customization

The call script supports passing context to the voice agent via ElevenLabs dynamic variables. To use this:

1. Add `{{call_context}}` to your agent's system prompt in the ElevenLabs dashboard, e.g.:

   ```
   You are a helpful assistant. {{call_context}}
   ```

2. Pass context when making a call:

   ```bash
   python3 skills/outbound-call/call.py +15551234567 "" "Remind the customer about their 3pm appointment tomorrow"
   ```

The agent will see "Remind the customer about their 3pm appointment tomorrow" injected into its prompt for that specific call.

## Troubleshooting

- **`ELEVENLABS_API_KEY not set`**: The environment variable isn't reaching the container. Check `grep ELEVENLABS_ "$DEPLOY_DIR/.env"` and verify `docker-compose.yml` has `env_file: .env`. Restart with `docker compose down && docker compose up -d`.
- **`Invalid US phone number`**: The script expects E.164 format: `+1` followed by exactly 10 digits. No dashes, spaces, or parentheses. Example: `+15551234567`.
- **HTTP 422 from ElevenLabs**: Usually means `agent_phone_number_id` is wrong. Re-run the phone numbers API call from Step 2 to get the correct ID.
- **HTTP 401 from ElevenLabs**: API key is invalid or expired. Generate a new one at [elevenlabs.io/app/settings/api-keys](https://elevenlabs.io/app/settings/api-keys).
- **Call connects but agent uses default greeting**: You need to enable the "First message" override in your agent's Security tab (Step 8). Without it, the `first_message` parameter is silently ignored.
- **Phone rings but no voice / silence**: The ElevenLabs agent can't reach your OpenClaw endpoint. Verify `curl -sS https://YOUR_HOSTNAME.hstgr.cloud/v1/chat/completions` still works with the gateway token.
- **Skill not recognized by OpenClaw**: The agent doesn't use `call.py` when you ask it to make a call. Verify the SKILL.md exists at the correct path: `docker exec $CONTAINER_NAME cat /data/.openclaw/workspace/skills/outbound-call/SKILL.md`. Restart the container after adding the skill.
- **`Permission denied` running call.py**: The script needs execute permission. Run `docker exec $CONTAINER_NAME chmod +x /data/.openclaw/workspace/skills/outbound-call/call.py`.
- **Rate limits / billing concerns**: ElevenLabs bills outbound calls per minute of conversation. Check your plan limits at [elevenlabs.io/app/settings/billing](https://elevenlabs.io/app/settings/billing). Consider adding call rate limiting (see Production Hardening).
- **Container name changed after redeployment**: Run `docker ps --filter "name=openclaw" --format '{{.Names}}'` to get the current name.

## Next Steps

After making outbound calls, you'll likely want to see what was said. The **[ElevenLabs Transcript Webhook](hostinger-openclaw-elevenlabs-transcript-webhook.md)** guide sets up automatic transcript delivery — when any call ends (inbound or outbound), the full transcript appears in the OpenClaw chat.
