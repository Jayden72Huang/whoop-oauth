<p align="center">
  <img src="./assets/readme/hero.svg" width="100%" alt="whoop-oauth — long-term Whoop access for your AI agent">
</p>

**English | [简体中文](./README.zh-CN.md)**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
![Battle-tested](https://img.shields.io/badge/Battle--tested-Oct%202026-green)

## What this is

A single `SKILL.md` that teaches an AI agent how to connect to Whoop via OAuth2 — and *stay* connected — so the agent can read recovery, sleep, workouts, and energy data long-term. Agent-agnostic: written for Meta Muse, but OpenAI GPTs / Claude builders can substitute their own OAuth flow (Muse-specific parts are marked `[Muse]` in the skill).

## Battle-tested, not theorized

This skill is the distilled record of connecting Whoop **three times in real-world sessions in October 2026**, including two full failure-and-fix cycles:

- **Attempt #1** died with a 401 — traced to `token_endpoint_auth_method` defaulting to `client_secret_basic`; switching to `client_secret_post` fixed it.
- **Attempt #2** worked for about an hour, then died — the access token had no refresh token because `offline` wasn't in the scope (the Whoop dashboard has no checkbox for it; it must be hand-added to the auth URL).
- **Attempt #3** verified the complete fix end-to-end: authorization took ~2 minutes, and auto-refresh was confirmed still working **80+ minutes after issuance**.

## Why this exists

The official Whoop docs don't explain either of these, and both bite on first contact:

1. **`token_endpoint_auth_method` must be `client_secret_post`.** The OAuth default, `client_secret_basic`, returns a bare 401 against Whoop's token endpoint. The client secret goes in the POST body, not the `Authorization` header.
2. **You must hand-add `offline` to the authorization URL's scope.** The Whoop Developer Dashboard has no `offline` checkbox. Without it, the token you get lives ~1 hour and has *no refresh token* — your connection dies silently an hour later.

Bonus gotcha worth knowing: Whoop's app form doesn't validate the client secret. A typo won't show up until your token exchange returns 401.

## Quickstart

**1. Create the app.** In the [Whoop Developer Dashboard](https://developer.whoop.com/), create an app and set the redirect URI to your agent's OAuth callback. For Muse: `https://agent.meta.ai/api/hatch/oauth/callback`. You'll need a public privacy-policy page — it's an app requirement.

**2. Authorize with the exact parameters.** Build the auth URL with your client ID, redirect URI, and all seven scopes — `offline` included:

```text
https://api.prod.whoop.com/oauth/oauth2/auth
  ?client_id=YOUR_CLIENT_ID
  &redirect_uri=YOUR_REDIRECT_URI
  &response_type=code
  &scope=offline+read:recovery+read:cycles+read:sleep+read:workout+read:profile+read:body_measurement
```

**3. User authorizes.** This takes ~2 minutes. The user grants access and your callback receives a `code`.

**4. Exchange the code for tokens.** Note: `client_secret` goes in the POST **body** (`client_secret_post`) — not the header:

```bash
curl -X POST https://api.prod.whoop.com/oauth/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "code=YOUR_CODE" \
  -d "redirect_uri=YOUR_REDIRECT_URI" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET"
```

**5. Verify auto-refresh.** Wait **70+ minutes**, then call an API endpoint again. If it works without re-authorizing, refresh is alive and your connection is long-term. No webhook is needed for personal use.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| 401 on token exchange | Secret in `Authorization` header (`client_secret_basic`) | Move `client_id`/`client_secret` into the POST body |
| Works, then dies after ~1 hour | `offline` wasn't in the auth URL scope → no refresh token | Re-authorize with `offline` added to `scope` |
| 401 right after creating the app | Typo in the client secret — the dashboard form never validates it | Re-check the secret, re-register the app if needed |
| 404 on `/sleep`, `/workout`, or `/recovery` | No data in the queried range | Not an error — the band usually just wasn't worn at night |

## After you're connected

API base: `https://api.prod.whoop.com`

| Endpoint | What it gives you |
|---|---|
| `/user/profile/basic` | User profile |
| `/user/body_measurement` | Height, weight, max heart rate |
| `/cycle` | Physiological days (sleep-to-sleep — may cross calendar days); strain + total energy per cycle |
| `/recovery` | Recovery score, HRV, resting HR |
| `/sleep` | Sleep performance, stages, efficiency |
| `/workout` | Logged activities with strain and energy |

Collection endpoints paginate at **limit 25** with `nextToken`. Empty recovery/sleep/workout collections almost always mean the band wasn't worn at night — treat "no data" as "no data," not an error.

## Limits & notes

- The skill only reads data; it never writes to Whoop or changes user settings.
- **Never commit client secrets** — the skill routes all credentials through the agent's secure OAuth flow, never through chat or files.
- Token lifetime is ~1 hour; the skill relies on the `offline` scope's refresh token, not on re-authorization.
- Muse-specific setup steps are marked `[Muse]` in the skill; other agents substitute their platform's OAuth flow.

## License

[MIT](./LICENSE) — do what you want with it.
