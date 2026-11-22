# itad-mcp-lobster

An MCP (Model Context Protocol) server for the IsThereAnyDeal API.

## Design Principles

- No generic "call any API" tool
- **Each ITAD API operation maps to a dedicated MCP tool**
- Full coverage of all 65 operations in the official OpenAPI spec (v2.11.0): 57 tools are registered by default, and the 8 internal tools are opt-in (see [Internal endpoints](#internal-endpoints-opt-in))
- API key is read from an environment variable and sent as the `ITAD-API-Key` header, never in the request URL

## Features

### Input Structure (All Tools)

Every tool uses the following input fields (populate only what the endpoint requires):

- `pathParams`: Path parameter object (e.g. `{ "shopId": 61 }`)
- `query`: Query parameter object (e.g. `{ "id": "uuid", "country": "US" }`)
- `headers`: Request headers object (e.g. `{ "ITAD-Profile": "xxx" }`)
- `body`: Request body (object or array)
- `oauthToken`: OAuth token (OAuth endpoints and the deals tools; optional, falls back to the env var)
- `useOAuth`: `deals_v2` / `deals_post_v2` only. Set `true` to authenticate with OAuth instead of the API key, which enables user filters such as "in my Waitlist"

Each tool validates required parameters automatically for its corresponding endpoint.

### Documentation

- **[Full tool list](./docs/CALLING_GUIDE.md#6-full-tool-list-with-required-parameters)**: all 65 tools with method, path, auth and required parameters
- [docs/CALLING_GUIDE.md](./docs/CALLING_GUIDE.md): full calling documentation (auth, safety annotations, response format, examples, common errors)

The server also exposes two MCP Resources so agents that can't read repository files still have access to calling information:

- `itad://guide/calling`: Markdown calling guide (human-readable)
- `itad://endpoints/index.json`: Machine-readable metadata for every registered tool

## Requirements

- Node.js 18+
- [pnpm](https://pnpm.io/installation) (recommended) or npm (bundled with Node.js)
- IsThereAnyDeal API Key (register at <https://isthereanydeal.com/apps/my/>)

## Installation

```bash
git clone https://github.com/jlobster/itad-mcp-lobster.git
cd itad-mcp-lobster
pnpm install
pwd   # copy this path — you'll need it for the MCP client config
```

Using npm instead? Replace `pnpm install` with `npm install`.

## Credentials

### Creating an app on IsThereAnyDeal

Both the API key and OAuth client ID come from the same app registration at <https://isthereanydeal.com/apps/my/>.

When creating the app:

- **App Name** — required; any name you like (e.g. `itad-mcp-lobster`)
- **Description** — optional
- **OAuth — app type** — select **"This app will NOT store OAuth secret securely"** (a local tool can't keep a client secret; the PKCE flow below doesn't need one)
- **Redirect URI** — set to `http://localhost`

After saving, the page shows your **Client ID** (used in the OAuth authorization URL below) and an **API Key**.

### API Key (price/deal endpoints)

Most read-only endpoints (prices, deals, search, game info) require an API key. Copy the API Key shown on your app's page.

### OAuth Token (account endpoints)

Endpoints that access your account (waitlist, collection, ignore list, notes, notifications, profiles, webhooks, user info) require an OAuth token. The deals tools can also use one, for filters such as "in my Waitlist".

ITAD supports the **authorization code flow with PKCE**. Steps:

1. Create an app at <https://isthereanydeal.com/apps/my/> as described above (or use an existing one).
2. Generate a PKCE verifier and challenge (Node.js is already installed for this server):
   ```bash
   node -e "const c=require('crypto');const v=c.randomBytes(32).toString('base64url');console.log('verifier: '+v);console.log('challenge: '+c.createHash('sha256').update(v).digest('base64url'))"
   ```
3. Open this URL in your browser, substituting your Client ID and the **challenge**:
   ```
   https://isthereanydeal.com/oauth/authorize/?client_id=YOUR_CLIENT_ID&response_type=code&redirect_uri=http://localhost&code_challenge=YOUR_CHALLENGE&code_challenge_method=S256&state=itad-mcp-lobster&scope=user_info%20notifications%20notes_read%20notes_write%20wait_read%20wait_write%20coll_read%20coll_write%20ignored_read%20ignored_write%20profiles%20webhooks
   ```
4. Authorize the app. ITAD redirects to `http://localhost/?code=...&state=itad-mcp-lobster`. The page won't load, but the URL bar holds the **code**. Copy it right away, because codes expire quickly.
5. Exchange the code for tokens, substituting your Client ID, the code and the **verifier**:
   ```bash
   curl -X POST https://isthereanydeal.com/oauth/token/      -d grant_type=authorization_code      -d client_id=YOUR_CLIENT_ID      -d redirect_uri=http://localhost      -d code=YOUR_CODE      -d code_verifier=YOUR_VERIFIER
   ```
   PowerShell (one line):
   ```powershell
   curl.exe -X POST https://isthereanydeal.com/oauth/token/ -d grant_type=authorization_code -d client_id=YOUR_CLIENT_ID -d redirect_uri=http://localhost -d code=YOUR_CODE -d code_verifier=YOUR_VERIFIER
   ```
   The JSON response contains `access_token` (put it in `ITAD_OAUTH_TOKEN`), `expires_in` and a `refresh_token`. Keep the refresh token private.
6. When the access token expires, get a new one without repeating the browser step:
   ```bash
   curl -X POST https://isthereanydeal.com/oauth/token/      -d grant_type=refresh_token      -d client_id=YOUR_CLIENT_ID      -d refresh_token=YOUR_REFRESH_TOKEN
   ```
   PowerShell (one line):
   ```powershell
   curl.exe -X POST https://isthereanydeal.com/oauth/token/ -d grant_type=refresh_token -d client_id=YOUR_CLIENT_ID -d refresh_token=YOUR_REFRESH_TOKEN
   ```

**Scopes.** The URL above requests every scope the API uses, so all account tools work: `user_info`, `notifications`, `notes_read`/`notes_write`, `wait_read`/`wait_write`, `coll_read`/`coll_write`, `ignored_read`/`ignored_write`, `profiles` and `webhooks`. To grant less, remove scopes from the list; the tools that need them will then fail with an authorization error. ITAD notes that some endpoints need explicit approval. If the authorize page rejects a scope, drop that scope and retry.

> **Implicit flow (discouraged).** Older guides use `response_type=token`, which puts the token straight into the redirect URL with no exchange step. ITAD says its "usage is discouraged and support may be removed in the future". It also gives you no refresh token. It still works for now, if you need it:
>
> ```
> https://isthereanydeal.com/oauth/authorize/?client_id=YOUR_CLIENT_ID&response_type=token&redirect_uri=http://localhost&scope=user_info%20notifications%20notes_read%20notes_write%20wait_read%20wait_write%20coll_read%20coll_write%20ignored_read%20ignored_write%20profiles%20webhooks
> ```
>
> After you authorize, the token is in the URL bar after `#access_token=`. Copy it into `ITAD_OAUTH_TOKEN`.

### Setting credentials via .env

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env   # macOS / Linux
copy .env.example .env  # Windows
```

`.env`:
```
ITAD_API_KEY=your_isthereanydeal_api_key
ITAD_OAUTH_TOKEN=your_oauth_token
```

The server loads this file automatically — no credentials needed in your MCP client config. `ITAD_OAUTH_TOKEN` is only needed if you use account endpoints.

Restrict the file to your user account only:

```bash
# macOS / Linux
chmod 600 .env

# Windows (PowerShell)
icacls .env /inheritance:r /grant:r "${env:USERNAME}:(R)"
```

## Starting the Server

```bash
pnpm start
```

With npm: `npm start`. Either one runs `node src/index.js`.

> This package is named `itad-mcp-lobster`. Don't use `npx itad-mcp`: that npm name belongs to a different, unrelated project, so npx would download and run that package instead. `itad-mcp-lobster` isn't published to npm, so point your MCP client at `src/index.js` as shown below.

The server communicates with MCP clients over stdio. You normally don't start it by hand: your MCP client launches it using the config below.

## MCP Client Configuration Example

```json
{
  "mcpServers": {
    "isthereanydeal": {
      "command": "node",
      "args": ["/path/to/itad-mcp-lobster/src/index.js"]
    }
  }
}
```

The server loads credentials from the `.env` file in the project root automatically — no `env` block needed. If you prefer to set credentials as OS-level user environment variables instead of a `.env` file, those work too and take precedence.

## Security

- **Tool safety annotations.** Every tool declares MCP annotations: `readOnlyHint`, `destructiveHint`, `idempotentHint` and `openWorldHint`. Clients that honor them can ask before risky calls. Of the 65 tools, 43 are read-only (35 of them registered by default, plus the 8 opt-in internal tools). 8 make additive account changes, such as adding to the waitlist. 14 are destructive: every delete, plus updates that overwrite data, profile syncs and `webhooks_add_v1`. Write and destructive tools also carry a warning in their description. The full breakdown is in [docs/CALLING_GUIDE.md](./docs/CALLING_GUIDE.md#3-tool-safety-annotations).
- **Prompt injection.** Tool results contain third-party text: game titles, giveaway descriptions, your own notes. Text like that could try to steer the agent into calling a write tool. The riskiest is `webhooks_add_v1`, which would send your ITAD notifications to whatever URL it's given. If your client lets you, require approval for non-read-only tools. Don't give an agent your OAuth token if it only needs prices and deals.
- **Credentials stay out of URLs.** The API key is sent as the `ITAD-API-Key` header. Any `key` query param a caller passes is dropped.
- **Prefer env vars for the OAuth token.** Tools accept an `oauthToken` argument, but anything passed that way is visible in the conversation transcript and client logs. Set `ITAD_OAUTH_TOKEN` instead.
- **Request timeout.** Each call to ITAD is aborted after 30 seconds by default. To change it, set `ITAD_REQUEST_TIMEOUT_MS`.

## Internal endpoints (opt-in)

The ITAD spec includes 8 `/internal/*` endpoints. ITAD's docs say internal APIs "are available only for our own services" and are "documented here for the sake of completeness". This server therefore **does not register them by default**. To expose them, set this in `.env` or your MCP client's `env` block, then restart the server:

```
ITAD_ENABLE_INTERNAL_TOOLS=true
```

`1`, `yes` and `on` also work. The value is read once, at startup.

**What gets exposed** (all use your API key):

| Tool | Endpoint | Data |
|---|---|---|
| `internal_earlyaccess_v1` | GET /internal/early-access/v1 | Early Access games |
| `internal_exfgls_v1` | GET /internal/exfgls/v1 | Games excluded from Steam Family Sharing (a GET with a JSON body) |
| `internal_hltb_v1` | GET /internal/hltb/v1 | HowLongToBeat times for a Steam `appid` |
| `internal_players_v1` | GET /internal/players/v1 | Player-count statistics for a Steam `appid` |
| `internal_rates_v1` | GET /internal/rates/v1 | Currency conversion rates |
| `internal_reviews_v1` | GET /internal/reviews/v1 | Review scores for a Steam `appid` |
| `internal_twitchstream_v1` | GET /internal/twitch/stream/v1 | Current stream of a Twitch `channel` |
| `internal_wsgf_v1` | GET /internal/wsgf/v1 | WSGF (widescreen support) overview for a Steam `appid` |

**Potential issues and risks:**

- **They will probably fail.** Expect authorization errors with a normal API key, because ITAD reserves these endpoints for its own services. They're only worth enabling if ITAD has approved your app for them.
- **No stability promise.** Internal endpoints can change shape or disappear at any time without a version bump. Responses may differ from the spec.
- **Terms of service.** ITAD's terms let it revoke API access "at any point without notice", and warn that "attempts to work around the limits will cause ban". Calling endpoints reserved for ITAD's own services without approval risks your API key and account.
- **Shared rate limit.** Every call, including failed ones, counts against your app's limit (1000 requests per 5 minutes for verified accounts).
- **More tools for the agent.** Eight extra tools enlarge the tool list your AI client loads and may tempt the agent to call them instead of the supported endpoints.

## Tool List

The server defines 65 tools, one per ITAD API operation: 57 are registered by default and the 8 internal tools are opt-in (see [Internal endpoints](#internal-endpoints-opt-in)). The full list, with method, path, auth and required parameters for each tool, is in [docs/CALLING_GUIDE.md](./docs/CALLING_GUIDE.md#6-full-tool-list-with-required-parameters). A running server also lists every registered tool in the `itad://endpoints/index.json` resource.
