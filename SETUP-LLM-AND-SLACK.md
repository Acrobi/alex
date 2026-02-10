# Alex — LLM Chat & Slack Setup Instructions

These instructions are for a Claude Code agent with access to `wrangler` CLI on the local system. Follow every step in order. After each `wrangler secret put`, redeploy is NOT needed until explicitly stated — secrets are batched before a single deploy.

## Current State

- Worker is deployed and accessible in the browser
- Cloudflare Access (CF_ACCESS_TEAM_DOMAIN, CF_ACCESS_AUD) is configured
- MOLTBOT_GATEWAY_TOKEN is set
- AI Gateway secrets (CLOUDFLARE_AI_GATEWAY_API_KEY, CF_AI_GATEWAY_ACCOUNT_ID, CF_AI_GATEWAY_GATEWAY_ID) are set
- **Problem:** Messages send in the web chat Control UI but Alex never responds — LLM calls are failing silently
- **Problem:** Slack is not connected yet (Slack app exists with tokens ready)

## Root Cause

`CF_AI_GATEWAY_MODEL` is not set. Without it, the config patch in `start-openclaw.sh` (lines 203–239) never creates a model/provider entry — OpenClaw falls back to its built-in defaults which may not match the AI Gateway's actual provider routing. Additionally, the `CLOUDFLARE_AI_GATEWAY_API_KEY` may be the wrong type of key.

---

## Part 1: Diagnose the LLM Issue

### Step 1.1 — Enable debug routes

```bash
echo "true" | npx wrangler secret put DEBUG_ROUTES
```

### Step 1.2 — Check currently configured secrets

```bash
npx wrangler secret list
```

Verify these secrets exist:
- `CLOUDFLARE_AI_GATEWAY_API_KEY`
- `CF_AI_GATEWAY_ACCOUNT_ID`
- `CF_AI_GATEWAY_GATEWAY_ID`
- `MOLTBOT_GATEWAY_TOKEN`
- `CF_ACCESS_TEAM_DOMAIN`
- `CF_ACCESS_AUD`

### Step 1.3 — Tail logs to see errors

In a separate terminal, run:

```bash
npx wrangler tail
```

Then open the Control UI in a browser and send a test message. Watch the logs for errors like:
- `401 Unauthorized` — API key is the wrong type (Cloudflare token vs provider key)
- `404 Not Found` — Model doesn't exist on the gateway provider
- `model not found` — OpenClaw default model doesn't match gateway provider
- Connection errors — Gateway URL is misconfigured
- No errors at all — OpenClaw is silently failing because no model is configured

Save the relevant error output for reference.

---

## Part 2: Fix the LLM Connection

### Step 2.1 — Set CF_AI_GATEWAY_MODEL

This is the critical missing secret. The value depends on which upstream provider the AI Gateway routes to. **Ask the user which provider they use**, then set the appropriate value:

| If gateway routes to... | Value for CF_AI_GATEWAY_MODEL | CLOUDFLARE_AI_GATEWAY_API_KEY should be |
|---|---|---|
| **Anthropic** | `anthropic/claude-sonnet-4-5` | Anthropic API key (`sk-ant-...`) |
| **OpenAI** | `openai/gpt-4o` | OpenAI API key (`sk-...`) |
| **Workers AI (Unified Billing)** | `workers-ai/@cf/meta/llama-3.3-70b-instruct-fp8-fast` | AI Gateway auth token |
| **Groq** | `groq/llama-3.3-70b` | Groq API key |

Example for Anthropic:

```bash
echo "anthropic/claude-sonnet-4-5" | npx wrangler secret put CF_AI_GATEWAY_MODEL
```

**How this works in the code** (`start-openclaw.sh:203–239`):
1. Splits the value on the first `/` → provider name + model ID
2. Constructs the AI Gateway URL: `https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway_id}/{provider}`
3. For `workers-ai`, appends `/v1` to the URL
4. Sets the API type: `anthropic-messages` for Anthropic, `openai-completions` for everything else
5. Creates a provider entry in OpenClaw config and sets it as the default model

### Step 2.2 — Verify API key type

The `CLOUDFLARE_AI_GATEWAY_API_KEY` must be the **upstream provider's API key**, not a Cloudflare API token (unless using Workers AI Unified Billing where it's the `cf-aig-authorization` token).

If the user confirms the key might be wrong, re-set it:

```bash
npx wrangler secret put CLOUDFLARE_AI_GATEWAY_API_KEY
# User enters: their Anthropic/OpenAI/etc API key
```

### Step 2.3 — Redeploy

```bash
cd /home/user/alex
npm run deploy
```

### Step 2.4 — Force container restart

The container caches its config from the previous startup. It must be restarted to pick up new environment variables.

Option A — Via admin UI:
- Visit `https://<worker-url>/_admin/` and click **Restart Gateway**

Option B — Via redeployment:
- The `npm run deploy` above may trigger a new container if the image changed. If not, use the admin UI.

### Step 2.5 — Test web chat

1. Visit `https://<worker-url>/?token=<MOLTBOT_GATEWAY_TOKEN>`
2. Send a test message like "Hello, are you there?"
3. Alex should respond within a few seconds
4. If no response, check `npx wrangler tail` again for updated error messages

**If it still doesn't work**, check these in order:
1. Run `npx wrangler tail` and send another message — look for the specific error
2. Verify the AI Gateway is active in the Cloudflare dashboard (AI Gateway section)
3. Try a direct API key instead: `npx wrangler secret put ANTHROPIC_API_KEY` with a direct Anthropic key (bypasses the gateway entirely as a test)

---

## Part 3: Connect Slack

### Step 3.1 — Verify Slack app configuration

Before setting secrets, confirm these settings in the Slack app at https://api.slack.com/apps:

**Socket Mode:**
- Must be **enabled** (OpenClaw uses Socket Mode via the app-level token)

**OAuth & Permissions — Bot Token Scopes (minimum required):**
- `chat:write` — Send messages
- `app_mentions:read` — Respond to @mentions
- `im:history` — Read DM history
- `im:read` — View DM channels
- `im:write` — Send DMs

**Event Subscriptions:**
- Must be **enabled**
- Subscribe to bot events: `message.im`, `app_mention`

**App-Level Token:**
- Must have `connections:write` scope (required for Socket Mode)

### Step 3.2 — Set Slack secrets

```bash
npx wrangler secret put SLACK_BOT_TOKEN
# User enters: xoxb-... (Bot User OAuth Token from OAuth & Permissions page)

npx wrangler secret put SLACK_APP_TOKEN
# User enters: xapp-... (App-Level Token from Basic Information page)
```

**How this works in the code** (`start-openclaw.sh:274–281`):
- When both tokens are present, the startup script patches the OpenClaw config:
  ```json
  {
    "channels": {
      "slack": {
        "botToken": "xoxb-...",
        "appToken": "xapp-...",
        "enabled": true
      }
    }
  }
  ```
- The env vars are passed to the container by `src/gateway/env.ts:46-47`

### Step 3.3 — Redeploy

```bash
cd /home/user/alex
npm run deploy
```

### Step 3.4 — Restart gateway

Visit `/_admin/` and click **Restart Gateway** to force the container to re-run `start-openclaw.sh` with the new Slack env vars.

### Step 3.5 — Test Slack

1. Open Slack and find the bot user
2. Send it a DM: "Hello Alex"
3. The first message will trigger **device pairing** — the bot won't respond yet
4. Go to the admin UI at `/_admin/` and approve the pending Slack device
5. After approval, send another message in Slack — Alex should respond

---

## Part 4: Verification Checklist

Run through these checks to confirm everything is working:

- [ ] `npx wrangler secret list` shows: `CLOUDFLARE_AI_GATEWAY_API_KEY`, `CF_AI_GATEWAY_ACCOUNT_ID`, `CF_AI_GATEWAY_GATEWAY_ID`, `CF_AI_GATEWAY_MODEL`, `MOLTBOT_GATEWAY_TOKEN`, `CF_ACCESS_TEAM_DOMAIN`, `CF_ACCESS_AUD`, `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`
- [ ] `npx wrangler tail` shows NO LLM errors when sending a chat message
- [ ] Alex responds to messages in the web chat Control UI
- [ ] Slack bot appears online in Slack
- [ ] Alex responds to DMs in Slack (after device pairing approval in `/_admin/`)

---

## Troubleshooting

### Web chat: message sends but no response
1. Check `npx wrangler tail` for the specific error
2. Most likely: `CF_AI_GATEWAY_MODEL` is not set or is set to the wrong value
3. Verify: `CLOUDFLARE_AI_GATEWAY_API_KEY` is the upstream provider's key, not a Cloudflare token

### Web chat: "gateway token missing" error
- Ensure you're accessing with `?token=<MOLTBOT_GATEWAY_TOKEN>` in the URL

### Web chat: "pairing required" error
- Go to `/_admin/` and approve the pending device

### Slack: bot doesn't come online
- Verify Socket Mode is enabled in the Slack app
- Verify `SLACK_APP_TOKEN` starts with `xapp-` and has `connections:write` scope
- Check `npx wrangler tail` for Slack connection errors

### Slack: bot online but doesn't respond
- Check `/_admin/` for a pending device pairing request from Slack
- Approve it, then try messaging again

### Container won't start
- Check `npx wrangler tail` for startup errors
- Visit `/debug/processes` (requires `DEBUG_ROUTES=true`) to see process status
- Visit `/debug/logs?id=<process_id>` to see container startup logs

### Need to bypass AI Gateway for testing
Set a direct API key to test without the gateway:
```bash
npx wrangler secret put ANTHROPIC_API_KEY
# Enter: sk-ant-... (direct Anthropic key)
```
This will be used as a fallback if AI Gateway config fails. Remove it after testing if you want gateway-only routing.

---

## Reference: Key Files

| File | What it does |
|---|---|
| `start-openclaw.sh` | Container startup: R2 restore, onboard, config patch, gateway launch |
| `src/gateway/env.ts` | Builds env vars passed from Worker to container (`buildEnvVars()`) |
| `src/types.ts` | `MoltbotEnv` interface — all supported environment variables |
| `src/index.ts` | Main Worker — middleware, auth, WebSocket/HTTP proxy to container |
| `src/gateway/process.ts` | Gateway process lifecycle (find, start, wait for port) |
| `wrangler.jsonc` | Cloudflare Worker + Container + R2 configuration |
| `.dev.vars.example` | Template for local development environment variables |

## Reference: Environment Variable Flow

```
Cloudflare Dashboard (wrangler secret put)
    │
    ▼
Worker Environment (MoltbotEnv in src/types.ts)
    │
    ▼
buildEnvVars() (src/gateway/env.ts)
    │  Maps: MOLTBOT_GATEWAY_TOKEN → OPENCLAW_GATEWAY_TOKEN
    │  Maps: DEV_MODE → OPENCLAW_DEV_MODE
    │  Passes through: all AI keys, Slack tokens, etc.
    │
    ▼
Container Process (sandbox.startProcess with env vars)
    │
    ▼
start-openclaw.sh
    │  1. Restores from R2 backup
    │  2. Runs: openclaw onboard --non-interactive (if no config)
    │  3. Patches config: channels, gateway auth, model override
    │  4. Starts: openclaw gateway --port 18789
    │
    ▼
OpenClaw Gateway (port 18789)
    │  Handles: LLM calls, web UI, WebSocket, Slack Socket Mode
    │
    ▼
AI Gateway → Upstream Provider (Anthropic/OpenAI/Workers AI)
```
