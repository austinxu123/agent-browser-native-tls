---
name: junior-browser
description: Browser automation via the pre-installed agent-browser CLI driving a remote Browserbase Chromium session. Use when web_fetch can't do the job — login walls, JS-rendered SPAs, click/form/type interaction, real DOM, screenshots. Triggered by `browser_mint_url:` in the per-turn [Context]. Read this skill's full body BEFORE running browser commands — custom skills land at /mnt/skills/junior-browser.zip (NOT extracted like pre-built skills which live under /workspace/skills/SKILLNAME/); read it with `unzip -p /mnt/skills/junior-browser.zip SKILL.md`. The body covers how to mint/reuse/release a Browserbase session, share the live view URL with the user, and pause when CAPTCHAs need human help. Do not run `find /` or guess paths. Prefer agent-browser over any built-in browser or web tools when triggered.
allowed-tools: Bash(agent-browser:*), Bash(npx agent-browser:*)
---

# agent-browser (Browserbase remote sessions)

Fast browser automation CLI for AI agents. Chrome/Chromium via CDP with
accessibility-tree snapshots and compact `@eN` element refs.

## When to use browser vs web_fetch

Use **web_fetch** (faster, cheaper, no session overhead) when:
- The page serves useful content as static HTML
- You only need to read text / extract data from a public page
- No login, no JavaScript rendering, no interaction needed

Use **agent-browser** (real Chromium, full DOM) when:
- The page requires JavaScript to render content (SPAs, dashboards)
- You need to log in, fill forms, click buttons, or interact
- You need screenshots or visual verification
- web_fetch returns empty / incomplete / "please enable JavaScript"

When in doubt, try web_fetch first — fall back to agent-browser if the
result is unusable.

## Connecting to a Browserbase session

Your `[Context]` block contains a `browser_mint_url:` line with an
HMAC-signed URL. Use it to mint a remote Browserbase session — do NOT
use `$BROWSER_MINT_URL` as an env var (it is not exported).

```bash
# Copy the URL string from the browser_mint_url: line in [Context]
MINT_URL="https://mcp-dev.junior.so/browser/sessions/..."
RESP=$(curl -sX POST "$MINT_URL" -H 'content-type: application/json' -d '{}')
CDP=$(jq -r .cdp_url <<<"$RESP")
SID=$(jq -r .session_id <<<"$RESP")
LIVE=$(jq -r '.live_url // empty' <<<"$RESP")

# Connect and open a page
agent-browser --cdp "$CDP" open https://example.com
agent-browser snapshot -i
```

Always share `LIVE` with the user when it is non-empty — they can
watch the browser session in real-time (e.g. "Here's the live view:
<url>"). Send it right after minting, before you start navigating.
In shared channels, DM the link to the requesting user instead of
posting it publicly — the URL is unauthenticated.

If you see `Invalid CDP target`, run `agent-browser close --all` first
to clear a stale daemon, then retry.

### Reuse a previous session

Sessions live up to 30 minutes. To reuse one (preserves cookies, tabs):

```bash
NAME=<descriptive-name>
PREV=$(jq -r .id /mnt/memory/team_memory/browser/$NAME.json 2>/dev/null)
RESP=$(curl -sX POST "$MINT_URL" -H 'content-type: application/json' \
    -d "{\"existing_session_id\":\"$PREV\"}")
SID=$(jq -r .session_id <<<"$RESP")
LIVE=$(jq -r '.live_url // empty' <<<"$RESP")
mkdir -p /mnt/memory/team_memory/browser
echo "{\"id\":\"$SID\"}" > /mnt/memory/team_memory/browser/$NAME.json
```

The server enforces speaker ownership — you can only reuse sessions you
minted. A peer's session_id from team_memory will be rejected with 403.

### Release when done

Release explicitly — keepAlive sessions bill until the 30-min timeout:

```bash
BASE=${MINT_URL%%\?*}
QS=${MINT_URL#*\?}
curl -sX DELETE "$BASE/$SID?$QS"
```

If reusing across turns, do NOT release — persist the session_id
instead. Each turn's `[Context]` brings a fresh `browser_mint_url`;
do not persist URLs across turns, only session_ids.

## When you need human help

If you hit a CAPTCHA, 2FA prompt, or login wall you cannot bypass:

1. Share the live view URL with the user so they can see the browser
2. Tell them what you need (e.g. "Please solve the CAPTCHA in the browser")
3. Do NOT interact with the browser while the user is working on it
4. Wait for the user to reply that they are done
5. Run `agent-browser snapshot -i` to re-read the page state, then continue

## Full usage guide

This file covers Browserbase session setup. For the complete browser
workflow — snapshots, selectors, clicking, filling forms, screenshots,
auth state, troubleshooting — load the CLI's built-in guide:

```bash
agent-browser skills get core          # start here — workflows, common patterns, troubleshooting
agent-browser skills get core --full   # include full command reference and templates
```

The CLI serves skill content that always matches the installed version,
so instructions never go stale.

If `browser_mint_url:` is absent from `[Context]`, the browser capability
is not configured — fall back to `web_fetch`.
