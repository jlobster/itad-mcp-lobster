# MCP Calling Guide (IsThereAnyDeal)

This document covers practical calling information for the **65 API tools** in this repository (ITAD API spec v2.11.0). 57 are registered by default; the 8 `internal_*` tools are registered only when `ITAD_ENABLE_INTERNAL_TOOLS=true` (see the README's "Internal endpoints" section).

## 1. Input Structure (All Tools)

All tools share the same input schema:

```json
{
  "pathParams": {},
  "query": {},
  "headers": {},
  "body": null,
  "oauthToken": "",
  "useOAuth": false
}
```

Field descriptions:

- `pathParams`: Path parameters, e.g. `shopId` in `/lookup/id/shop/{shopId}/v1`
- `query`: Query string parameters. A `key` param is always dropped; the API key is sent as a header
- `headers`: Request headers (e.g. `ITAD-Profile`)
- `body`: Request body (object or array)
- `oauthToken`: OAuth token (`oauth` and `key_or_oauth` endpoints). Prefer the environment variable: a token passed here is visible in the conversation transcript and client logs
- `useOAuth`: `key_or_oauth` endpoints only (`deals_v2`, `deals_post_v2`). `true` = authenticate with OAuth instead of the API key

> Only populate the fields required by the endpoint — unused fields can be omitted.

---

## 2. Authentication Rules

- `auth = key`: API Key required (sent automatically as the `ITAD-API-Key` header from the env var, never in the URL)
- `auth = key_or_oauth`: API Key by default. OAuth is used when `useOAuth: true` or `oauthToken` is passed, or when no API key is set. OAuth enables user filters such as in-Waitlist / in-Collection (scopes `wait_read`, `coll_read`)
- `auth = oauth`: OAuth token required (`oauthToken` param takes priority, then env var)
- `auth = none`: No authentication required

API Key environment variables (any one):

- `ITAD_API_KEY` (recommended)
- `ISTHEREANYDEAL_API_KEY`
- `APIKEY`
- `apikey`

OAuth environment variables (any one):

- `ITAD_OAUTH_TOKEN`
- `ISTHEREANYDEAL_OAUTH_TOKEN`

---

## 3. Tool Safety Annotations

Every tool carries MCP annotations that clients can use to ask for confirmation:

- **Read-only** (`readOnlyHint: true`, 43 tools): all GETs, plus the query-style POSTs (`deals_post_v2`, `games_prices_v3`, `games_overview_v2`, `games_historylow_v1`, `games_storelow_v2`, `games_subscriptions_v1`, and the three `lookup_*` tools)
- **Write** (`readOnlyHint: false`, `destructiveHint: false`, 8 tools): additive changes, e.g. `waitlist_games_v1_put`, `collection_games_v1_put`, `collection_copies_v1_post`, `notifications_read_v1_put`
- **Destructive** (`destructiveHint: true`, 14 tools): every DELETE, plus `collection_copies_v1_patch`, `collection_groups_v1_patch`, `user_notes_v1_put`, `profiles_sync_collection_v1_put`, `profiles_sync_waitlist_v1_put` and `webhooks_add_v1`, which sends the user's notifications to an external URL

All tools have `openWorldHint: true`, since they call the ITAD API. Counts include the 8 internal tools, which are all read-only: with the default registration, 35 tools are read-only.

## 4. Response and Error Format

On success, MCP returns:

```json
{
  "content": [
    { "type": "text", "text": "<JSON string>" }
  ]
}
```

On failure, MCP returns:

```json
{
  "isError": true,
  "content": [
    { "type": "text", "text": "<error message>" }
  ]
}
```

---

## 5. Example Calls

### 5.1 Query Bundles Containing a Game

tool: `games_bundles_v2`  
endpoint: `GET /games/bundles/v2`  
auth: `key`  
required: `query.id`

```json
{
  "query": {
    "id": "01849783-6a26-7147-ab32-71804ca47e8e",
    "country": "US",
    "expired": false
  }
}
```

### 5.2 Search for a Game

tool: `games_search_v1`  
endpoint: `GET /games/search/v1`  
auth: `key`  
required: `query.title`

```json
{
  "query": {
    "title": "Baldur's Gate 3",
    "results": 10
  }
}
```

### 5.3 Batch Price Query

tool: `games_prices_v3`  
endpoint: `POST /games/prices/v3`  
auth: `key`  
required: `body`

```json
{
  "query": {
    "country": "US",
    "deals": true,
    "shops": [61, 35]
  },
  "body": [
    "01849783-6a26-7147-ab32-71804ca47e8e",
    "01849782-1017-7389-8de4-c97c587fd7e3"
  ]
}
```

### 5.4 Reverse Shop ID Lookup (with path parameter)

tool: `lookup_gid_shopid_v1`  
endpoint: `POST /lookup/id/shop/{shopId}/v1`  
auth: `key`  
required: `pathParams.shopId`, `body`

```json
{
  "pathParams": {
    "shopId": 61
  },
  "body": [
    "app/620",
    "app/730"
  ]
}
```

### 5.5 OAuth Endpoint Example (Get User Info)

tool: `user_info_v2`  
endpoint: `GET /user/info/v2`  
auth: `oauth`

```json
{}
```

The token comes from `ITAD_OAUTH_TOKEN`. Passing `"oauthToken": "..."` also works, but it puts the token in the transcript, so reserve it for a deliberate one-off account switch.

### 5.6 Query Game Info (`games_info_v2`)

tool: `games_info_v2`  
endpoint: `GET /games/info/v2`  
auth: `key`  
required: `query.id`

```json
{
  "query": {
    "id": "01849783-6a26-7147-ab32-71804ca47e8e"
  }
}
```

> Note: `games_info_v2` `query.id` only supports a single string, not an array. For multiple games, call it once per ID.

---

## 6. Full Tool List with Required Parameters

### Bundles / Deals / Games / Lookup / Service / Stats / Internal / Unstable (API Key, API Key or OAuth, none)

| tool | method | path | auth | required |
|---|---|---|---|---|
| `bundles_v1` | GET | `/bundles/v1` | key | - |
| `deals_post_v2` | POST | `/deals/v2` | key_or_oauth | - |
| `deals_v2` | GET | `/deals/v2` | key_or_oauth | - |
| `games_bundles_v2` | GET | `/games/bundles/v2` | key | query:`id` |
| `games_giveaways_v1` | GET | `/games/giveaways/v1` | key | query:`id` |
| `games_history_v2` | GET | `/games/history/v2` | key | query:`id` |
| `games_historylow_v1` | POST | `/games/historylow/v1` | key | body |
| `games_info_v2` | GET | `/games/info/v2` | key | query:`id` |
| `games_lookup_v1` | GET | `/games/lookup/v1` | key | - |
| `games_overview_v2` | POST | `/games/overview/v2` | key | body |
| `games_prices_v3` | POST | `/games/prices/v3` | key | body |
| `games_search_v1` | GET | `/games/search/v1` | key | query:`title` |
| `games_storelow_v2` | POST | `/games/storelow/v2` | key | body |
| `games_subscriptions_v1` | POST | `/games/subs/v1` | key | body |
| `giveaways_v1` | GET | `/giveaways/v1` | key | - |
| `lookup_gid_shopid_v1` | POST | `/lookup/id/shop/{shopId}/v1` | key | path:`shopId`, body |
| `lookup_gid_title_v1` | POST | `/lookup/id/title/v1` | key | body |
| `lookup_shopid_gid_v1` | POST | `/lookup/shop/{shopId}/id/v1` | key | path:`shopId`, body |
| `service_shops_map_v1` | GET | `/service/shops/map/v1` | none | - |
| `service_shops_v1` | GET | `/service/shops/v1` | none | - |
| `stats_most_collected_v1` | GET | `/stats/most-collected/v1` | key | - |
| `stats_most_popular_v1` | GET | `/stats/most-popular/v1` | key | - |
| `stats_most_waitlisted_v1` | GET | `/stats/most-waitlisted/v1` | key | - |
| `stats_waitlist_v1` | GET | `/stats/waitlist/v1` | key | query:`id` |
| `internal_earlyaccess_v1` ¹ | GET | `/internal/early-access/v1` | key | - |
| `internal_exfgls_v1` ¹ | GET | `/internal/exfgls/v1` | key | body |
| `internal_hltb_v1` ¹ | GET | `/internal/hltb/v1` | key | query:`appid` |
| `internal_players_v1` ¹ | GET | `/internal/players/v1` | key | query:`appid` |
| `internal_rates_v1` ¹ | GET | `/internal/rates/v1` | key | - |
| `internal_reviews_v1` ¹ | GET | `/internal/reviews/v1` | key | query:`appid` |
| `internal_twitchstream_v1` ¹ | GET | `/internal/twitch/stream/v1` | key | query:`channel` |
| `internal_wsgf_v1` ¹ | GET | `/internal/wsgf/v1` | key | query:`appid` |
| `unstable_games_dots_v1` | GET | `/unstable/games/dots/v1` | key | - |
| `unstable_games_list_v1` | GET | `/unstable/games/list/v1` | key | - |

¹ Registered only when `ITAD_ENABLE_INTERNAL_TOOLS=true`. ITAD documents `/internal/*` as available only for its own services, so expect authorization errors.

### Collection / Ignored / Waitlist / Notes / Notifications / Profiles / User / Webhooks (OAuth)

| tool | method | path | auth | required |
|---|---|---|---|---|
| `collection_copies_v1_delete` | DELETE | `/collection/copies/v1` | oauth | body |
| `collection_copies_v1_get` | GET | `/collection/copies/v1` | oauth | - |
| `collection_copies_v1_patch` | PATCH | `/collection/copies/v1` | oauth | body |
| `collection_copies_v1_post` | POST | `/collection/copies/v1` | oauth | body |
| `collection_games_v1_delete` | DELETE | `/collection/games/v1` | oauth | body |
| `collection_games_v1_get` | GET | `/collection/games/v1` | oauth | - |
| `collection_games_v1_put` | PUT | `/collection/games/v1` | oauth | body |
| `collection_groups_v1_delete` | DELETE | `/collection/groups/v1` | oauth | body |
| `collection_groups_v1_get` | GET | `/collection/groups/v1` | oauth | - |
| `collection_groups_v1_patch` | PATCH | `/collection/groups/v1` | oauth | body |
| `collection_groups_v1_post` | POST | `/collection/groups/v1` | oauth | body |
| `ignored_games_v1_delete` | DELETE | `/ignored/games/v1` | oauth | body |
| `ignored_games_v1_get` | GET | `/ignored/games/v1` | oauth | - |
| `ignored_games_v1_put` | PUT | `/ignored/games/v1` | oauth | body |
| `notifications_list_v1_get` | GET | `/notifications/v1` | oauth | - |
| `notifications_read_all_v1_put` | PUT | `/notifications/read/all/v1` | oauth | - |
| `notifications_read_v1_put` | PUT | `/notifications/read/v1` | oauth | query:`id` |
| `notifications_waitlist_v1_get` | GET | `/notifications/waitlist/v1` | oauth | query:`id` |
| `profiles_link_v1_delete` | DELETE | `/profiles/link/v1` | oauth | headers:`ITAD-Profile` |
| `profiles_link_v1_put` | PUT | `/profiles/link/v1` | oauth | body |
| `profiles_sync_collection_v1_put` | PUT | `/profiles/sync/collection/v1` | oauth | headers:`ITAD-Profile`, body |
| `profiles_sync_waitlist_v1_put` | PUT | `/profiles/sync/waitlist/v1` | oauth | headers:`ITAD-Profile`, body |
| `user_info_v2` | GET | `/user/info/v2` | oauth | - |
| `user_notes_v1_delete` | DELETE | `/user/notes/v1` | oauth | body |
| `user_notes_v1_get` | GET | `/user/notes/v1` | oauth | - |
| `user_notes_v1_put` | PUT | `/user/notes/v1` | oauth | body |
| `waitlist_games_v1_delete` | DELETE | `/waitlist/games/v1` | oauth | body |
| `waitlist_games_v1_get` | GET | `/waitlist/games/v1` | oauth | - |
| `waitlist_games_v1_put` | PUT | `/waitlist/games/v1` | oauth | body |
| `webhooks_add_v1` | PUT | `/webhooks/v1` | oauth | body |
| `webhooks_delete_v1` | DELETE | `/webhooks/v1` | oauth | body |

---

## 7. Common Errors

- `missing required query/path/header parameter`: Supply the missing field per the docs above
- `requires a request body`: This endpoint must receive a body
- `requires an API Key`: Set `ITAD_API_KEY` or one of the other API key env vars
- `requires an OAuth Token`: Pass `oauthToken` or set an OAuth env var
- `requires an API Key or an OAuth Token`: a `key_or_oauth` tool found neither credential
- `ITAD API request failed (429 ...): Rate limited by ITAD; retry after N seconds`: wait for the `Retry-After` time given. The documented limit is 1000 requests per 5-minute window, so cache results where possible
- `ITAD API request timed out after N ms`: ITAD didn't answer in time. Retry, or raise `ITAD_REQUEST_TIMEOUT_MS`
- `ITAD API request failed (4xx/5xx)`: Upstream error or invalid parameters — check the response text
