# AGENTS.md

Guidance for AI coding agents (Claude Code, Codex, Cursor, Copilot, Gemini CLI, etc.) working in this repository.

## Commands

This repo uses **pnpm** (lockfile: `pnpm-lock.yaml`). Use pnpm for every dependency change so the lockfile stays authoritative.

```bash
pnpm install         # install dependencies
pnpm start           # run the MCP server (stdio transport)
pnpm run check       # syntax-check src/index.js without running it
pnpm add <pkg>       # add a dependency (never hand-edit package.json versions)
```

There is no test suite and no linter configured. The only verification step is `pnpm run check`.

To run the installed `bin` (`itad-mcp` in `package.json`) from the repo: `pnpm exec itad-mcp`.

npm still works for users (`npm install`, `npm start`), and `package-lock.json` is kept for them. If you change dependencies, regenerate both lockfiles: `pnpm install`, then `npm install --package-lock-only`.

## Hard rule: never write to stdout

The server speaks MCP over **stdio**, so stdout carries the protocol stream. Any stray write to stdout (`console.log`, a library banner, etc.) corrupts the transport and breaks the client connection.

- Use `console.error` for diagnostics.
- Keep `quiet: true` on the `dotenvConfig(...)` call: dotenv 17+ logs to stdout by default (fixed in commit `5c351f0`).
- When adding a dependency, check that it does not print to stdout on import or init.

## Architecture

This is a single-file MCP server (`src/index.js`) that wraps the entire IsThereAnyDeal (ITAD) REST API as MCP tools — one tool per API operation (65 defined, from ITAD API spec **v2.11.0**). 57 are registered by default; the 8 `/internal/*` tools are registered only when `ITAD_ENABLE_INTERNAL_TOOLS` is truthy (see below). All server code lives in that one file; the only other content is docs (`README.md`, `docs/CALLING_GUIDE.md`).

### Core data flow

```
MCP Client → tool call → ensureRequiredValues() → applyAuth() → itadRequest() → ITAD REST API
```

1. **Tool registration** — The `ENDPOINTS` array drives everything. Each entry has `{ tool, method, path, summary, auth }`. `ACTIVE_ENDPOINTS` filters out `/internal/*` paths unless `ITAD_ENABLE_INTERNAL_TOOLS` is set, and a `for` loop at the bottom registers those tools using `buildToolInputSchema()` and `buildToolDescription()`.

2. **Input schema** — All tools share the same five fields: `pathParams`, `query`, `headers`, `body`, `oauthToken`. `key_or_oauth` tools add a sixth, `useOAuth`. Required fields are enforced at runtime (not via Zod `.required()`) by `ensureRequiredValues()`, using three lookup maps (`REQUIRED_QUERY_PARAMS`, `REQUIRED_HEADER_PARAMS`, `REQUIRED_PATH_PARAMS`) and the set `BODY_REQUIRED_TOOLS`.

3. **Auth** — `applyAuth()` sends the API key as the `ITAD-API-Key` header (never the `key` query param, so it stays out of URLs) or an `Authorization: Bearer` header (oauth). Credentials are read from env vars at request time, not at startup.

4. **HTTP** — `itadRequest()` uses `fetch` for all methods except GET/HEAD with a body (which WHATWG fetch rejects). That edge case uses `node:http`/`node:https` directly; it serves `internal_exfgls_v1` (body required) and `deals_v2` (body optional). On a 429 the error message includes ITAD's `Retry-After` value.

5. **MCP Resources** — Two resources are registered for agent discoverability:
   - `itad://guide/calling` — Markdown human-readable calling guide
   - `itad://endpoints/index.json` — Machine-readable metadata for every registered tool

### Auth types

| `auth` value | Behavior |
|---|---|
| `"key"` | API Key required; injected as `?key=` query param |
| `"key_or_oauth"` | API Key by default; OAuth when `useOAuth: true` or `oauthToken` is passed, or when no key is set. Used by `deals_v2` / `deals_post_v2` (OAuth enables user filters) |
| `"oauth"` | OAuth Bearer token required; `oauthToken` input param takes priority over env var |
| `"none"` | No auth |

### Adding or modifying a tool

All changes flow from the `ENDPOINTS` array plus the four constraint collections. To add a new endpoint:
1. Add an entry to `ENDPOINTS`
2. If it has required params, add to the relevant map (`REQUIRED_QUERY_PARAMS`, etc.)
3. If the body is required, add the tool name to `BODY_REQUIRED_TOOLS`

No other registration code is needed — the loop handles the rest. If the tool count changes, update the counts in this file, `README.md` (including its generated tool list) and `docs/CALLING_GUIDE.md`. The calling-guide resource and the server version (`package.json`) are computed at runtime.

When syncing with a newer ITAD spec, diff `dist/openapi.json` from <https://github.com/isthereanydeal/api> against `ENDPOINTS` (method, path, `operationId` = tool name with `-` → `_`, security, required params), and read its `CHANGELOG.md`.

### Special cases

- `games_info_v2`: `query.id` must be a single string, not an array (enforced explicitly in `ensureRequiredValues`)
- `internal_exfgls_v1`: GET request with a required body — uses raw `node:https` to bypass fetch's restriction
- Query arrays are serialized as comma-joined strings (e.g. `?id=a,b,c`), not repeated params (matches the spec's `explode: false` arrays)
- `/internal/*` tools are opt-in: ITAD documents them as available only for its own services. Keep them gated; the README lists the risks

## Environment variables

Copy `.env.example` to `.env` and populate as needed. `.env` is loaded from the repo root (resolved relative to `src/index.js`, not the cwd); credentials are looked up at request time. Never commit `.env`.

- `ITAD_API_KEY` (recommended) or `ISTHEREANYDEAL_API_KEY` / `APIKEY` / `apikey` — for key-auth endpoints
- `ITAD_OAUTH_TOKEN` or `ISTHEREANYDEAL_OAUTH_TOKEN` — for oauth endpoints (and the deals tools in OAuth mode)
- `ITAD_ENABLE_INTERNAL_TOOLS` — `true`/`1`/`yes`/`on` registers the 8 internal tools. Read once at startup; off by default

## MCP client config example

```json
{
  "mcpServers": {
    "isthereanydeal": {
      "command": "node",
      "args": ["/path/to/itad-mcp/src/index.js"],
      "env": {
        "ITAD_API_KEY": "your_api_key"
      }
    }
  }
}
```
