# Backend (central AI proxy)

`server.js` is a zero-dependency Node server that serves the app **and** proxies AI calls
to Anthropic with a key held on the server — so end users never need their own API key or
credits. You pay for AI centrally.

## Run it

```bash
cp .env.example .env          # then put your funded key in .env
node server.js                # → http://localhost:4178
```

Or inline:

```bash
ANTHROPIC_API_KEY=sk-ant-... node server.js
```

The console prints `AI proxy: ON` when a key is loaded, `NO KEY SET` otherwise.

## How the frontend picks it up

On boot the app probes `GET /api/health`. If it returns `{ai:true}`, `Bridge.askClaude`
routes every AI call to `POST /api/ai` on this server — the browser never sees the key.

Fallback order (highest first): **Cowork host → this server proxy → user's own browser key → demo.**
So with no key set, the app still runs and degrades gracefully.

## Endpoints

- `GET /api/health` → `{ai: <bool>, web: <bool>, model, fastModel, deepModel}` — capability probe; reports the three-tier model map and whether web search is available.
- `POST /api/ai` `{prompt, blocks, fast, deep, web, cachePrefix}` → `{text}` — forwards to Anthropic. `deep`→**Claude Opus 4.8** (profile, fit, interview prep, all narrative generation), `fast`→Haiku 4.5 (classification/refreshers), else Sonnet 4.6 (extraction); `web`→enables Anthropic's `web_search` tool (used to find live job postings when Gmail has no JD) and pins to a search-capable model. `cachePrefix` is sent as an ephemeral cache block. Per-call token usage + cost is logged to the console (note: web searches bill ~$10/1k searches on top of tokens). Relays upstream errors.

## Before exposing this publicly — important

- **Gate `/api/ai` behind auth.** As written it has only a naive per-IP rate limit (20/min);
  anyone who can reach the URL can spend your credits. Tie it to the user's Google-authenticated
  session (or any login) before putting it on the internet.
- **Google (Gmail/Calendar) is still client-side OAuth**, not proxied here. To remove the
  per-user Google Cloud setup too, move the OAuth flow server-side (one project you own,
  users just "Sign in with Google"). That's the next step if you want zero user setup.
- Keep `.env` out of version control (already in `.gitignore`).
