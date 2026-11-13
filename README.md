<h1 align="center">
  <code>fluxr.lol</code> — The Complete API Reference
</h1>

<h3 align="center">
  Every endpoint on <strong>Fluxer</strong> — the 2019-era revival platform —<br>
  extracted from the production Next.js bundles &amp; verified against the live gateway.
</h3>

<p align="center">
  <img alt="endpoints" src="https://img.shields.io/badge/endpoints-302-2ea44f?style=for-the-badge">
  <img alt="services" src="https://img.shields.io/badge/services-31-f0883e?style=for-the-badge">
  <img alt="gateway" src="https://img.shields.io/badge/gateway-%2Fapisite-316dca?style=for-the-badge">
  <img alt="auth" src="https://img.shields.io/badge/auth-cookie%20%2B%20csrf-d1242f?style=for-the-badge">
  <img alt="unofficial" src="https://img.shields.io/badge/status-unofficial-8250df?style=for-the-badge">
</p>

> [!NOTE]
> Everything in this document was recovered from **three** places: the leaked rewrite table in
> `_buildManifest.js`, all **82 webpack chunks** of the production frontend, and live probes
> against `www.fluxr.lol`. Nothing is guessed. This is a community document — not affiliated
> with, endorsed by, or connected to fluxr.lol or Roblox Corporation in any way.

---

## 📦 Contents

- [How this map was made](#-how-this-map-was-made)
- [Architecture](#-architecture)
- [Base URLs](#-base-urls)
- [Auth & headers](#-auth--headers)
- [When a call fails](#-when-a-call-fails)
- [Response conventions](#-response-conventions)
- [API reference](#-api-reference)
  - [users](#users) · [accountinformation](#accountinformation) · [accountsettings](#accountsettings) · [auth](#auth) · [avatar](#avatar) · [catalog](#catalog) · [thumbnails](#thumbnails) · [games](#games) · [groups](#groups) · [friends](#friends) · [inventory](#inventory) · [develop](#develop) · [itemconfiguration](#itemconfiguration) · [economy](#economy) · [trades](#trades) · [forums](#forums) · [badges](#badges) · [chat](#chat) · [privatemessages](#privatemessages) · [presence](#presence) · [premiumfeatures](#premiumfeatures) · [ads](#ads) · [billing](#billing) · [privateservers](#privateservers) · [support](#support) · [api](#api-alerts) · [locale](#locale) · [donate](#donate) · [safechat](#safechat) · [paidaccess](#paidaccess) · [rbxcdn](#rbxcdn)
- [Legacy web endpoints](#-legacy-web-endpoints)
- [Game launch protocol](#-game-launch-protocol)
- [Client downloads](#-client-downloads)
- [Quick examples](#-quick-examples)
- [Notes & gotchas](#-notes--gotchas)

---

## 🔍 How this map was made

1. **`/_next/static/OQlDqchIHbJOf0zyI7o0L/_buildManifest.js`**
   Leaks the Next.js rewrite table that mounts the API gateway under `/apisite/*`,
   `/thumbs/*`, `/images/*`, `/asset/*`, `/admin/*` and `/v1/*`.
2. **Every page + shared chunk** (`webpackChunk_N_E`, 82 files)
   All `axios` wrapper calls and the service resolver were parsed out, then cross-checked
   between chunks that alias the same helpers.
3. **Live probes** against the production gateway to confirm status codes, auth
   requirements and real response shapes.

> [!IMPORTANT]
> The frontend builds URLs from one config value:
> `apiFormat: "https://www.fluxr.lol/apisite/{service}{path}"`
>
> So `bO("users", "/v1/users/authenticated")` becomes
> `GET https://www.fluxr.lol/apisite/users/v1/users/authenticated`

---

## 🏗 Architecture

```text
                    ┌──────────────────────────────────────────────┐
                    │                fluxr.lol (Next.js)           │
 browser ──────────►│  www.fluxr.lol                               │
                    │    ├── /apisite/{service}/**  ──► API gateway│
                    │    ├── /thumbs/** /images/**  ──► CDN/thumbs │
                    │    ├── /asset/**              ──► asset host │
                    │    ├── /game/** /universes/** ──► legacy web │
                    │    └── /admin-api/**          ──► staff tools│
                    └──────────────┬───────────────────────────────┘
                                   │
              desktop client ──────┴────── 127.0.0.1:4400/4401 (/launch)
```

---

## 🌐 Base URLs

| What | URL |
|---|---|
| Website | `https://www.fluxr.lol` |
| API gateway | `https://www.fluxr.lol/apisite/{service}{path}` |
| Legacy web API | `https://www.fluxr.lol/...` (no prefix) |
| Thumbnail CDN | `https://www.fluxr.lol/images/...` |
| Legacy thumbnails | `https://www.fluxr.lol/thumbs/asset.ashx` |
| Client downloads | `https://setup.fluxr.lol` |

---

## 🔐 Auth & headers

- **Cookie session.** Login sets httpOnly session cookies; member endpoints must carry
  them. There is no public token API.
- **CSRF.** Every request sends an `x-csrf-token` header (empty on first call).

> [!IMPORTANT]
> **The CSRF dance:** on a `403` whose response headers carry a fresh `x-csrf-token`,
> the client stores it and retries the *same request once*. If you script against the
> gateway, replicate this — otherwise every mutating call will bounce.

> [!WARNING]
> **Cloudflare Turnstile** protects signup and login (sitekey `0x4AAAAAAF_V9KGtIJ95V_DV`).
> Login page: `/auth/accountlogin` · Discord flows: `/auth/discord/start`,
> `/auth/discord/login`. One Discord account may verify only one Fluxer account.

Unauthenticated calls return the standard error envelope:

```json
{ "errors": [ { "code": 0, "message": "Authorization has been denied for this request." } ] }
```

---

## 🚨 When a call fails

Every endpoint is tagged with its auth level, so a weird response almost always
tells you exactly what's missing. Decode it here:

| Response | Why it happened | What to do |
|---|---|---|
| `401` — `Authorization has been denied for this request.` | Hit a 🔐 **Session** endpoint without valid cookies | `POST /apisite/auth/v2/login`, keep the session cookies, resend |
| `403` **with** a fresh `x-csrf-token` response header | CSRF token missing or expired | Read the token out of that header, replay the **same request once** with it — this is normal, not an error |
| `403` — captcha / Turnstile rejection | Action is bot-gated (login, signup, comments, uploads) | Only works through the real web flow; don't fight it |
| `404` on an endpoint that should exist | Build ID changed, paths moved | Re-dump `_buildManifest.js` and re-check the service prefix |
| `405` | Right path, wrong verb | Compare with the **Method** column |
| `429` or error codes like `FloodedGlobally` / `FloodedPerAsset` | Rate-limited (comments, DMs, forum posts) | Back off. Codes the client actually handles: `VerifyEmail`, `RequiresCaptcha`, `UserTooNew`, `FloodedGlobally`, `FloodedPerAsset`, `Moderated` |
| `500` — `Not logged in` | 🛡️ **Staff** routes answer `500` instead of `401` for outsiders | Stop. That response *is* the answer |

> [!IMPORTANT]
> The fastest way to tell **what kind of auth you're dealing with**: fire the request
> with no cookies at all.
> • `200` → it was 🔓 **Public** ·
> • `401` + the JSON error envelope → it was 🔐 **Session** ·
> • `500` + `Not logged in` → it's 🛡️ **Staff**.

---

## 📨 Response conventions

- REST-ish JSON. Collections paginate with `nextPageCursor` / `previousPageCursor` + `limit`.
- Detail endpoints return bare objects; list endpoints wrap in `{"data": [...]}`.
- Legacy (`.aspx`-era) routes use PascalCase payloads — `IsValid`, `Data`, `Items`, `Comments`.
- Thumbnail endpoints return `{"data":[{"targetId","state","imageUrl","version":"TN3"}]}`
  where `state` is `Pending` / `Completed` / `Blocked` / `Error`.

<details>
<summary>🧪 Live response samples — verified against production</summary>

```jsonc
// GET /apisite/users/v1/users/1
{
  "description": "Welcome to the Fluxr profile! ...",
  "created": "2026-10-10T20:22:22.874471Z",
  "isBanned": false,
  "hasVerifiedBadge": true,
  "id": 1,
  "name": "FLUXR",
  "displayName": "FLUXER",
  "isStaff": false,
  "inventory_rap": 0
}

// GET /apisite/games/v1/games/sorts?gameSortsContext=Home
{ "sorts": [ { "token": "popular", "name": "Popular", "tokenExpiryInSeconds": 86400 },
             { "token": "mostFavorited", "..." : "..." } ] }

// GET /apisite/thumbnails/v1/users/avatar?userIds=1&size=420x420&format=png
{ "data": [ { "targetId": 1, "state": "Completed",
  "imageUrl": "https://www.fluxr.lol/images/thumbnails/...png", "version": "TN3" } ] }

// GET /apisite/api/alerts/alert-info
{ "IsVisible": true, "Text": "Welcome to Fluxer! ...",
  "LinkUrl": "https://www.fluxr.lol/support" }
```
</details>

---

## 📚 API reference

> [!TIP]
> **Auth legend — every row in every table is tagged with its auth level:**
> 🔓 **Public** — confirmed working with no cookies at all ·
> 🔐 **Session** — needs logged-in cookies **+** the CSRF loop (see below) ·
> 🛡️ **Staff** — moderator/admin only, everyone else gets rejected.
>
> A cell like `GET` / `POST` means both methods exist on the same path.
> Call failed? Jump to **[🚨 When a call fails](#-when-a-call-fails)** to decode the status code.

### users

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/users/v1/users/authenticated` | Current session user | 🔐 Session |
| 2 | `GET` | `/apisite/users/v1/users/{id}` | Public profile (name, description, badges, RAP) | 🔓 Public |
| 3 | `GET` | `/apisite/users/v1/users/{id}/status` | User status blurb | 🔓 Public |
| 4 | `PATCH` | `/apisite/users/v1/users/{id}/status` | Set status `{status}` | 🔐 Session |
| 5 | `PATCH` | `/apisite/users/v1/users/{id}/display-names` | Change display name `{newDisplayName}` | 🔐 Session |
| 6 | `GET` | `/apisite/users/v1/users/{id}/username-history` | Past usernames `?limit=100&cursor=` | 🔓 Public |
| 7 | `POST` | `/apisite/users/v1/usernames/users` | Resolve usernames → ids `{usernames:[...]}` | 🔓 Public |
| 8 | `GET` | `/apisite/users/v1/users/{id}/profile-background` | Profile banner info | 🔓 Public |
| 9 | `POST` | `/apisite/users/v1/profile-background` | Upload profile banner (multipart) | 🔐 Session |
| 10 | `PATCH` | `/apisite/users/v1/profile-background/position` | Banner offset `{posX,posY,pending}` | 🔐 Session |
| 11 | `DELETE` | `/apisite/users/v1/profile-background` | Remove banner | 🔐 Session |

### accountinformation

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/accountinformation/v1/birthdate` | Account birthday | 🔐 Session |
| 2 | `POST` | `/apisite/accountinformation/v1/birthdate` | Set `{birthMonth,birthDay,birthYear}` | 🔐 Session |
| 3 | `GET` | `/apisite/accountinformation/v1/gender` | Account gender | 🔐 Session |
| 4 | `POST` | `/apisite/accountinformation/v1/gender` | Set `{gender}` | 🔐 Session |
| 5 | `POST` | `/apisite/accountinformation/v1/description` | Set `{description}` | 🔐 Session |
| 6 | `GET` | `/apisite/accountinformation/v1/users/{id}/promotion-channels` | Social links `?alwaysReturnUrls=` | 🔓 Public |
| 7 | `POST` | `/apisite/accountinformation/v1/promotion-channels` | Save social links | 🔐 Session |
| 8 | `GET` | `/apisite/accountinformation/v1/users/{id}/roblox-badges` | Classic badge icons | 🔓 Public |
| 9 | `GET` / `POST` | `/apisite/accountinformation/v1/staff-icon` | Read / set `{hidden}` staff-icon visibility | 🔐/🛡️ Staff |

### accountsettings

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` / `POST` | `/apisite/accountsettings/v1/trade-privacy` | Who can trade with me | 🔐 Session |
| 2 | `GET` / `POST` | `/apisite/accountsettings/v1/trade-value` | Trade quality filter (`None/Low/Medium/High`) | 🔐 Session |
| 3 | `GET` / `POST` | `/apisite/accountsettings/v1/inventory-privacy` | Inventory visibility (`AllUsers`, `Friends`, `NoOne`, …) | 🔐 Session |
| 4 | `GET` / `POST` | `/apisite/accountsettings/v1/private-message-privacy` | DM privacy | 🔐 Session |
| 5 | `GET` / `POST` | `/apisite/accountsettings/v1/follow-me-privacy` | Who can follow into games | 🔐 Session |
| 6 | `GET` / `POST` | `/apisite/accountsettings/v1/private-server-invite-privacy` | Private-server invite privacy | 🔐 Session |
| 7 | `POST` | `/apisite/accountsettings/v1/avatar-page-style` | Avatar page skin `{avatarPageStyle}` | 🔐 Session |
| 8 | `POST` | `/apisite/accountsettings/v1/users/{id}/block` | Block user | 🔐 Session |
| 9 | `POST` | `/apisite/accountsettings/v1/users/{id}/unblock` | Unblock user | 🔐 Session |
| 10 | `GET` | `/apisite/accountsettings/v1/users/{id}/is-blocked` | `{isBlocked}` | 🔐 Session |

### auth

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/auth/v2/login` | `{ctype:"username",cvalue,password}` | 🔓 Public |
| 2 | `POST` | `/apisite/auth/v1/username` | Legacy login `{username,password}` | 🔓 Public |
| 3 | `GET` | `/apisite/auth/v1/usernames/validate` | Name rules `?username=&context=` | 🔓 Public |
| 4 | `POST` | `/apisite/auth/v2/logout` | Kill current session `{}` | 🔐 Session |
| 5 | `POST` | `/apisite/auth/v2/logoutfromallsessionsandreauthenticate` | Sign out everywhere | 🔐 Session |
| 6 | `POST` | `/apisite/auth/v2/user/passwords/change` | `{currentPassword,newPassword}` | 🔐 Session |
| 7 | `GET` | `/apisite/auth/v2/accounts` | Linked multi-account list | 🔐 Session |
| 8 | `POST` | `/apisite/auth/v2/accounts/switch` | `{userId}` | 🔐 Session |
| 9 | `POST` | `/apisite/auth/v2/accounts/remove` | `{userId}` | 🔐 Session |
| 10 | `GET` / `POST` | `/apisite/auth/v2/ban` (+ `/ban/unlock`) | Ban status / acknowledge `{understood}` | 🔐 Session |

### avatar

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/avatar/v1/avatar` | Own avatar state | 🔐 Session |
| 2 | `GET` | `/apisite/avatar/v1/users/{id}/avatar` | Anyone's avatar | 🔓 Public |
| 3 | `GET` | `/apisite/avatar/v1/avatar-rules` | Rig types, scale ranges, wear limits | 🔓 Public |
| 4 | `POST` | `/apisite/avatar/v1/avatar/set-wearing-assets` | `{assetIds:[...]}` | 🔐 Session |
| 5 | `POST` | `/apisite/avatar/v1/avatar/set-body-colors` | Body color map | 🔐 Session |
| 6 | `POST` | `/apisite/avatar/v1/avatar/set-player-avatar-type` | `{playerAvatarType:1\|2}` (R6→1, R15→2) | 🔐 Session |
| 7 | `POST` | `/apisite/avatar/v1/avatar/set-scales` | Scale object (see avatar-rules) | 🔐 Session |
| 8 | `POST` | `/apisite/avatar/v1/avatar/redraw-thumbnail` | Queue re-render | 🔐 Session |
| 9 | `GET` | `/apisite/avatar/v1/avatar/set-rig?rigtype=R6\|R15` | Toggle rig (fired via `fetch`, not axios) | 🔐 Session |
| 10 | `GET` | `/apisite/avatar/v1/users/{id}/outfits` | `?itemsPerPage=50&page=1` | 🔓 Public |
| 11 | `POST` | `/apisite/avatar/v1/outfits/create` | `{name}` | 🔐 Session |
| 12 | `POST` | `/apisite/avatar/v1/outfits/{id}/wear` | Wear outfit | 🔐 Session |
| 13 | `POST` | `/apisite/avatar/v1/outfits/{id}/rename` | `{name}` | 🔐 Session |
| 14 | `PATCH` | `/apisite/avatar/v1/outfits/{id}` | Update outfit to current look `{}` | 🔐 Session |
| 15 | `POST` | `/apisite/avatar/v1/outfits/{id}/delete` | Delete outfit | 🔐 Session |
| 16 | `GET` | `/apisite/avatar/v1/recent-items/{category}/list` | `category`: `all`, `clothing`, `bodyparts`, `avataranimations`, `accessories`, `outfits`, `gear` | 🔐 Session |

### catalog

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/catalog/v1/catalog/items/details` | Batch details `{items:[{itemType:"Asset",id}]}` | 🔓 Public |
| 2 | `GET` | `/apisite/catalog/v1/search/items` | `?category=&maxRows=&cursor=&keyword=` | 🔓 Public |
| 3 | `GET` | `/apisite/catalog/v1/recommendations/asset/` | `?contextAssetId=&numItems=` | 🔓 Public |
| 4 | `GET` | `/apisite/catalog/v1/favorites/users/{id}/assets/{id}/favorite` | Is favorited | 🔐 Session |
| 5 | `POST` | `/apisite/catalog/v1/favorites/users/{id}/assets/{id}/favorite` | Favorite | 🔐 Session |
| 6 | `DELETE` | `/apisite/catalog/v1/favorites/users/{id}/assets/{id}/favorite` | Unfavorite | 🔐 Session |

### thumbnails

All return `{"data":[…]}`; sizes like `150x150`, `420x420`; `format=png`.

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/thumbnails/v1/users/avatar?userIds=&size=&format=` | Full-body renders | 🔓 Public |
| 2 | `GET` | `/apisite/thumbnails/v1/users/avatar-headshot?userIds=&size=&format=` | Headshots | 🔓 Public |
| 3 | `GET` | `/apisite/thumbnails/v1/users/avatar-3d?userIds=&size=&format=` | 3D JSON renders | 🔓 Public |
| 4 | `GET` | `/apisite/thumbnails/v1/users/outfits?userOutfitIds=&size=&format=` | Outfit renders | 🔓 Public |
| 5 | `GET` | `/apisite/thumbnails/v1/groups/icons?groupIds=&format=&size=` | Group icons | 🔓 Public |
| 6 | `GET` | `/apisite/thumbnails/v1/assets?assetIds=&format=&size=` | Item thumbnails | 🔓 Public |
| 7 | `GET` | `/apisite/thumbnails/v1/games/icons?universeIds=&size=&format=` | Game icons | 🔓 Public |
| 8 | `GET` | `/apisite/thumbnails/v1/try-on/{assetId}` | 2D "wear on my avatar" render | 🔐 Session |
| 9 | `GET` | `/apisite/thumbnails/v1/assets/{assetId}/3d` | 3D item model | 🔐 Session |

> [!TIP]
> The render endpoints (#8, #9) answer with `state: "Pending"` until the renderer
> finishes. The website polls every **1.5 s** and gives up after **80 attempts** —
> do the same and cache the result.

### games

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/games/v1/games?universeIds=` | Batch universe info (csv) | 🔓 Public |
| 2 | `GET` | `/apisite/games/v1/games/{universeId}` | Universe detail | 🔓 Public |
| 3 | `GET` | `/apisite/games/v1/games/sorts?gameSortsContext=Home` | Home sorts → `sortToken`s | 🔓 Public |
| 4 | `GET` | `/apisite/games/v1/games/list` | `?sortToken=&maxRows=&genre=&keyword=&year=` | 🔓 Public |
| 5 | `GET` | `/apisite/games/v1/games/recommendations/game/{placeId}?maxRows=` | Similar games | 🔓 Public |
| 6 | `GET` | `/apisite/games/v1/games/multiget-place-details?placeIds=` | Batch place details | 🔓 Public |
| 7 | `GET` | `/apisite/games/v1/games/votes?universeIds=` | Like/dislike counts | 🔓 Public |
| 8 | `PATCH` | `/apisite/games/v1/games/{universeId}/user-votes` | `{vote:true\|false\|null}` | 🔐 Session |
| 9 | `GET` | `/apisite/games/v1/games/game-passes/{id}` | Game-pass info | 🔓 Public |
| 10 | `GET` | `/apisite/games/v1/games/{universeId}/social-links/list` | Universe social links | 🔓 Public |
| 11 | `GET` | `/apisite/games/v1/users/{id}/count` | Games created by user | 🔓 Public |
| 12 | `GET` | `/apisite/games/v2/games/{universeId}/media` | Media assets | 🔓 Public |
| 13 | `GET` | `/apisite/games/v2/groups/{groupId}/games` | `?limit=25&cursor=` | 🔓 Public |

### groups

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/groups/v1/groups/search` | `?keyword=&maxRows=&startIndex=&sort=members` | 🔓 Public |
| 2 | `GET` | `/apisite/groups/v1/groups/{id}` | Group info | 🔓 Public |
| 3 | `GET` | `/apisite/groups/v1/groups/{id}/membership` | Own membership | 🔐 Session |
| 4 | `GET` | `/apisite/groups/v1/groups/{id}/roles` | Role list | 🔓 Public |
| 5 | `GET` | `/apisite/groups/v1/groups/{id}/settings` | Group settings | 🔐 Session |
| 6 | `GET` | `/apisite/groups/v1/groups/{id}/users` | Members `?cursor=&limit=&sortOrder=&search=` | 🔓 Public |
| 7 | `GET` | `/apisite/groups/v1/groups/{id}/roles/{roleId}/users` | Members of a role | 🔓 Public |
| 8 | `GET` | `/apisite/groups/v1/groups/{id}/roles/{roleId}/permissions` | Role permissions | 🔓 Public |
| 9 | `GET` | `/apisite/groups/v1/groups/{id}/join-requests` | `?limit=&cursor=` (owner) | 🔐 Session |
| 10 | `GET` | `/apisite/groups/v1/groups/{id}/audit-log` | `?cursor=&action=&userId=&sortOrder=desc&limit=100` | 🛡️ Staff |
| 11 | `POST` | `/apisite/groups/v1/groups/create` | Create group | 🔐 Session |
| 12 | `POST` | `/apisite/groups/v1/groups/{id}/users` | Join group | 🔐 Session |
| 13 | `DELETE` | `/apisite/groups/v1/groups/{id}/users/{userId}` | Kick member | 🔐 Session |
| 14 | `PATCH` | `/apisite/groups/v1/groups/{id}/users/{userId}` | Change rank `{roleId}` | 🔐 Session |
| 15 | `POST` / `DELETE` | `/apisite/groups/v1/groups/{id}/join-requests/users/{userId}` | Accept / decline join request | 🔐 Session |
| 16 | `POST` | `/apisite/groups/v1/groups/{id}/claim-ownership` | Claim abandoned group | 🔐 Session |
| 17 | `PATCH` | `/apisite/groups/v1/groups/{id}/change-owner` | `{userId}` | 🔐 Session |
| 18 | `PATCH` | `/apisite/groups/v1/groups/{id}/description` | `{description}` | 🔐 Session |
| 19 | `PATCH` | `/apisite/groups/v1/groups/{id}/settings` | `{isApprovalRequired,areEnemiesAllowed,areGroupFundsVisible,areGroupGamesVisible}` | 🔐 Session |
| 20 | `PATCH` | `/apisite/groups/v1/groups/{id}/status` | `{message}` | 🔐 Session |
| 21 | `POST` | `/apisite/groups/v1/groups/{id}/rolesets/create` | `{name,description,rank}` | 🔐 Session |
| 22 | `PATCH` / `DELETE` | `/apisite/groups/v1/groups/{id}/rolesets/{roleId}` | Edit / delete role | 🔐 Session |
| 23 | `PATCH` | `/apisite/groups/v1/groups/{id}/roles/{roleId}/permissions` | `{permissions}` | 🔐 Session |
| 24 | `PATCH` | `/apisite/groups/v1/groups/icon?groupId=` | Upload group icon (multipart) | 🔐 Session |
| 25 | `POST` | `/apisite/groups/v1/groups/{id}/wall/posts` | `{body}` | 🔐 Session |
| 26 | `GET` | `/apisite/groups/v2/groups/{id}/wall/posts` | `?sortOrder=&limit=&cursor=` | 🔓 Public |
| 27 | `DELETE` | `/apisite/groups/v1/groups/{id}/wall/posts/{postId}` | Delete wall post | 🔐 Session |
| 28 | `POST` | `/apisite/groups/v1/groups/{id}/payouts` | `{PayoutType:"FixedAmount",Recipients:[...]}` | 🔐 Session |
| 29 | `GET` | `/apisite/groups/v1/users/{id}/groups/roles` | User's group roles | 🔓 Public |
| 30 | `GET` | `/apisite/groups/v1/users/{id}/groups/primary/role` | Primary group role | 🔓 Public |
| 31 | `POST` | `/apisite/groups/v1/user/groups/primary` | Set primary `{groupId}` | 🔐 Session |
| 32 | `DELETE` | `/apisite/groups/v1/user/groups/primary` | Clear primary | 🔐 Session |

### friends

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/friends/v1/users/{id}/friends/statuses?userIds=` | Friend state per user | 🔐 Session |
| 2 | `GET` | `/apisite/friends/v1/users/{id}/followers` | `?cursor=&sort=&limit=` | 🔓 Public |
| 3 | `GET` | `/apisite/friends/v1/users/{id}/followers/count` | Count | 🔓 Public |
| 4 | `GET` | `/apisite/friends/v1/users/{id}/followings` | `?cursor=&sort=&limit=` | 🔓 Public |
| 5 | `GET` | `/apisite/friends/v1/users/{id}/followings/count` | Count | 🔓 Public |
| 6 | `GET` | `/apisite/friends/v1/user/friend-requests/count` | Pending count | 🔐 Session |
| 7 | `GET` | `/apisite/friends/v1/my/friends/requests` | `?limit=&cursor=` | 🔐 Session |
| 8 | `POST` | `/apisite/friends/v1/users/{id}/request-friendship` | Send request | 🔐 Session |
| 9 | `POST` | `/apisite/friends/v1/users/{id}/accept-friend-request` | Accept | 🔐 Session |
| 10 | `POST` | `/apisite/friends/v1/users/{id}/decline-friend-request` | Decline | 🔐 Session |
| 11 | `POST` | `/apisite/friends/v1/users/{id}/unfriend` | Remove friend | 🔐 Session |
| 12 | `POST` | `/apisite/friends/v1/users/{id}/follow` | Follow | 🔐 Session |
| 13 | `POST` | `/apisite/friends/v1/users/{id}/unfollow` | Unfollow | 🔐 Session |
| 14 | `POST` | `/apisite/friends/v1/user/following-exists` | `{targetUserIds:[...]}` | 🔐 Session |
| 15 | `GET` | `/apisite/friends/v1/my/nicknames` | List nicknames | 🔐 Session |
| 16 | `PUT` / `DELETE` | `/apisite/friends/v1/users/{id}/nickname` | Set (`{nickname}`) / clear | 🔐 Session |

### inventory

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/inventory/v1/users/{id}/assets/collectibles` | `?cursor=&limit=&assetType=` (limiteds) | 🔓 Public |
| 2 | `GET` | `/apisite/inventory/v1/users/{id}/items/Asset/{assetId}` | Instances of an item | 🔓 Public |
| 3 | `GET` | `/apisite/inventory/v2/assets/{assetId}/owners` | `?cursor=&limit=&sortOrder=` | 🔓 Public |
| 4 | `DELETE` | `/apisite/inventory/v2/inventory/asset/{userAssetId}` | Delete owned instance | 🔐 Session |

### develop

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/develop/v1/places/{id}` | Place info | 🔐 Session |
| 2 | `GET` / `PATCH` | `/apisite/develop/v1/places/{id}/place-copy` | Copying on/off `{enabled}` | 🔐 Session |
| 3 | `PATCH` | `/apisite/develop/v1/places/{id}/roblox-place-id` | Link real place `{robloxPlaceId}` | 🔐 Session |
| 4 | `POST` | `/apisite/develop/v1/places/{id}/game-icons/auto-generated` | Auto icon | 🔐 Session |
| 5 | `GET` | `/apisite/develop/v1/universes/{id}` | Universe info | 🔐 Session |
| 6 | `GET` / `PATCH` | `/apisite/develop/v1/universes/{id}/access` | `{privacyType,playableDevices,universeAvatarType,emotesEnabled}` | 🔐 Session |
| 7 | `PATCH` | `/apisite/develop/v1/universes/{id}/max-player-count` | `{maxPlayers}` | 🔐 Session |
| 8 | `PATCH` | `/apisite/develop/v1/universes/{id}/set-year` | `{year}` (retro year) | 🔐 Session |
| 9 | `POST` | `/apisite/develop/v1/universes/{id}/start-place` | `{placeId}` | 🔐 Session |
| 10 | `POST` / `DELETE` | `/apisite/develop/v1/universes/{id}/social-links` (+`/{linkId}`) | Manage links `{type,url,title}` | 🔐 Session |
| 11 | `POST` | `/apisite/develop/v1/universes/{id}/thumbnails/auto-generated` | Auto thumbnail | 🔐 Session |
| 12 | `POST` | `/apisite/develop/v1/universes/{id}/thumbnails/youtube` | `{url}` | 🔐 Session |
| 13 | `POST` | `/apisite/develop/v1/universes/{id}/thumbnails/{id}` | Activate thumbnail | 🔐 Session |
| 14 | `GET` / `POST` | `/apisite/develop/v1/universes/{id}/web-permissions` | Team create `{collaborators,roles}` | 🔐 Session |
| 15 | `GET` | `/apisite/develop/v1/universes/{id}/web-places` | Place list | 🔐 Session |
| 16 | `GET` | `/apisite/develop/v1/universes/{id}/developerproducts?pageSize=25` | Developer products | 🔐 Session |
| 17 | `POST` | `/apisite/develop/v1/universes/{id}/developerproducts?name=&description=&priceInRobux=&iconImageAssetId=` | Create product | 🔐 Session |
| 18 | `POST` | `/apisite/develop/v1/universes/{id}/developerproducts/{id}/update` | `{Name,Description,IconImageAssetId,PriceInRobux}` | 🔐 Session |
| 19 | `GET` | `/apisite/develop/v1/assets/{id}/published-versions` | `?limit=&sortOrder=Desc&cursor=` | 🔐 Session |
| 20 | `POST` | `/apisite/develop/v1/assets/{id}/revert-version?assetVersionNumber=` | Roll back | 🔐 Session |
| 21 | `PATCH` | `/apisite/develop/v1/assets/{id}` | `{name,description,genres,isCopyingAllowed,enableComments,isForSale}` | 🔐 Session |
| 22 | `POST` | `/apisite/develop/v1/assets/upload-gameicon?placeId=` | Multipart icon upload | 🔐 Session |
| 23 | `POST` | `/apisite/develop/v1/assets/upload-thumbnail?universeId=` | Multipart thumbnail upload | 🔐 Session |
| 24 | `GET` | `/apisite/develop/v1/assets/genres` | Genre list | 🔓 Public |

### itemconfiguration

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/itemconfiguration/v1/assets/restrictions?assetIds=` | Moderation restrictions | 🔓 Public |
| 2 | `GET` | `/apisite/itemconfiguration/v1/creations/get-assets?assetType=&limit=&cursor=&groupId=` | Own creations | 🔐 Session |
| 3 | `POST` | `/apisite/itemconfiguration/v1/creations/get-asset-details` | `{assetIds:[...]}` | 🔐 Session |
| 4 | `POST` | `/apisite/itemconfiguration/v1/assets/{id}/update-price` | `{priceInRobux}` | 🔐 Session |

### economy

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/economy/v1/users/{id}/currency` | Robux balance | 🔐 Session |
| 2 | `GET` | `/apisite/economy/v1/groups/{id}/currency` | Group funds | 🔐 Session |
| 3 | `GET` | `/apisite/economy/v1/assets/{id}/resale-data` | Limited resale stats | 🔓 Public |
| 4 | `GET` | `/apisite/economy/v1/assets/{id}/users/{id}/resellable-copies` | Own copies | 🔐 Session |
| 5 | `PATCH` | `/apisite/economy/v1/assets/{id}/resellable-copies/{uaid}` | Reprice `{price}` | 🔐 Session |
| 6 | `POST` | `/apisite/economy/v1/purchases/products/{productId}` | Buy `{assetId,expectedPrice,expectedSellerId,userAssetId,expectedCurrency}` | 🔐 Session |
| 7 | `GET` | `/apisite/economy/v2/users/{id}/transactions` | `?cursor=&transactionType=` | 🔐 Session |
| 8 | `GET` | `/apisite/economy/v2/users/{id}/transaction-totals` | `?timeFrame=&transactionType=summary` | 🔐 Session |
| 9 | `GET` | `/apisite/economy/v2/groups/{id}/transactions` | Group transactions | 🔐 Session |
| 10 | `GET` | `/apisite/economy/v2/groups/{id}/transaction-totals` | Group summary | 🔐 Session |

### trades

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/trades/v1/trades/{type}?cursor=` | `type`: `inbound`, `outbound`, `completed`, `inactive` | 🔐 Session |
| 2 | `GET` | `/apisite/trades/v1/trades/{id}` | Trade detail | 🔐 Session |
| 3 | `GET` | `/apisite/trades/v1/trades/inbound/count` | Badge count | 🔐 Session |
| 4 | `POST` | `/apisite/trades/v1/trades/send` | `{offers:[{robux,userAssetIds,userId},…]}` | 🔐 Session |
| 5 | `POST` | `/apisite/trades/v1/trades/{id}/accept` | Accept | 🔐 Session |
| 6 | `POST` | `/apisite/trades/v1/trades/{id}/decline` | Decline | 🔐 Session |
| 7 | `POST` | `/apisite/trades/v1/trades/{id}/counter` | Counter-offer `{offers:[…]}` | 🔐 Session |

### forums

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/forums/v1/sub-category/{id}/info` | Category info | 🔓 Public |
| 2 | `GET` | `/apisite/forums/v1/sub-category/{id}/posts` | `?limit=&cursor=` | 🔓 Public |
| 3 | `POST` | `/apisite/forums/v1/sub-category/{id}/thread` | New thread `{post,subject}` | 🔐 Session |
| 4 | `GET` | `/apisite/forums/v1/threads/{id}/info` | Thread info | 🔓 Public |
| 5 | `GET` | `/apisite/forums/v1/threads/{id}/replies` | `?limit=&cursor=` | 🔓 Public |
| 6 | `GET` | `/apisite/forums/v1/posts/{id}/info` | Post info | 🔓 Public |
| 7 | `POST` | `/apisite/forums/v1/posts/{id}/reply` | `{post}` | 🔐 Session |
| 8 | `POST` | `/apisite/forums/v1/posts/{id}/mark-as-read` | Read receipt | 🔐 Session |
| 9 | `DELETE` | `/apisite/forums/v1/posts/{id}` | Delete post | 🔐 Session |
| 10 | `GET` | `/apisite/forums/v1/users/{id}/posts` | `?limit=&cursor=` | 🔓 Public |

### badges

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/badges/v1/badges/{id}` | Badge detail | 🔓 Public |
| 2 | `GET` | `/apisite/badges/v2/badges/{id}/basic` | Lightweight detail | 🔓 Public |
| 3 | `GET` | `/apisite/badges/v1/users/{id}/badges` | `?cursor=&limit=` | 🔓 Public |
| 4 | `GET` | `/apisite/badges/v1/universes/{id}/badges` | `?cursor=&limit=` | 🔓 Public |
| 5 | `DELETE` | `/apisite/badges/v1/users/badges/{id}` | Revoke own badge | 🔐 Session |
| 6 | `PATCH` | `/apisite/badges/v1/badges/{id}` | Toggle `{enabled}` | 🔐 Session |

### chat

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` / `POST` | `/apisite/chat/v2/chat-settings` | Read / save chat settings | 🔐 Session |
| 2 | `GET` / `POST` | `/apisite/chat/v2/chat-privacy` | `{inGameChat,websiteChat}` | 🔐 Session |
| 3 | `GET` | `/apisite/chat/v2/get-user-conversations` | `?pageNumber=&pageSize=` | 🔐 Session |
| 4 | `GET` | `/apisite/chat/v2/get-messages` | `?conversationId=&pageSize=&exclusiveStartMessageId=` | 🔐 Session |
| 5 | `GET` | `/apisite/chat/v2/multi-get-latest-messages?conversationIds=` | Latest per conversation | 🔐 Session |
| 6 | `POST` | `/apisite/chat/v2/send-message` | `{conversationId,message}` | 🔐 Session |
| 7 | `POST` | `/apisite/chat/v2/start-one-to-one-conversation` | `{participantUserId}` | 🔐 Session |
| 8 | `POST` | `/apisite/chat/v2/start-group-conversation` | `{participantUserIds,title}` | 🔐 Session |
| 9 | `POST` | `/apisite/chat/v2/add-to-group-conversation` | `{conversationId,participantUserIds}` | 🔐 Session |
| 10 | `POST` | `/apisite/chat/v2/leave-group-conversation` | `{conversationId}` | 🔐 Session |
| 11 | `POST` | `/apisite/chat/v2/rename-group-conversation` | `{conversationId,title}` | 🔐 Session |
| 12 | `POST` | `/apisite/chat/v2/mark-as-read` | `{conversationId,endMessageId}` | 🔐 Session |
| 13 | `POST` | `/apisite/chat/v2/update-user-typing-status` | `{conversationId,isTyping}` | 🔐 Session |
| 14 | `POST` | `/apisite/chat/v2/send-game-link-message` | Share a game `{conversationId,universeId}` | 🔐 Session |

### privatemessages

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/privatemessages/v1/messages` | `?messageTab=&pageSize=&pageNumber=` (tabs: inbox/sent/archive) | 🔐 Session |
| 2 | `GET` | `/apisite/privatemessages/v1/messages/unread/count` | Unread badge | 🔐 Session |
| 3 | `GET` | `/apisite/privatemessages/v1/announcements` | System announcements | 🔓 Public |
| 4 | `POST` | `/apisite/privatemessages/v1/messages/send` | `{recipientid,body,subject,replyMessageId,includePreviousMessage}` | 🔐 Session |
| 5 | `POST` | `/apisite/privatemessages/v1/messages/{archive\|unarchive}` | `{messageIds:[...]}` | 🔐 Session |
| 6 | `POST` | `/apisite/privatemessages/v1/messages/{mark-read\|mark-unread}` | `{messageIds:[...]}` | 🔐 Session |

### presence

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/presence/v1/presence/users` | `{userIds:[...]}` → presence per user | 🔐 Session |

### premiumfeatures

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/premiumfeatures/v1/users/{id}/validate-membership` | Membership check | 🔐 Session |

### ads

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/ads/v1/user-ads/{User\|Group}/{creatorId}` | List ads of creator | 🔓 Public |
| 2 | `POST` | `/apisite/ads/v1/user-ads/{User\|Group}/create?assetId={targetId}` | Multipart `name`, `files` | 🔐 Session |
| 3 | `POST` | `/apisite/ads/v1/user-ads/{adId}/run` | Run campaign `{robux}` | 🔐 Session |

### billing

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/billing/v1/promocodes/redeem` | `{code}` | 🔐 Session |

### privateservers

Wrapper base: `/apisite/privateservers/v1`.

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` / `PATCH` | `/apisite/privateservers/v1/places/{placeId}/settings` | `{enabled,price}` | 🔐 Session |
| 2 | `GET` / `POST` | `/apisite/privateservers/v1/places/{placeId}/servers` | List / create (`{name}`) | 🔐 Session |
| 3 | `GET` / `PATCH` / `DELETE` | `/apisite/privateservers/v1/servers/{id}` | Detail / rename `{name,friendsAllowed}` / delete | 🔐 Session |
| 4 | `POST` | `/apisite/privateservers/v1/servers/{id}/members` | Whitelist `{username}` | 🔐 Session |
| 5 | `DELETE` | `/apisite/privateservers/v1/servers/{id}/members/{userId}` | Remove member | 🔐 Session |
| 6 | `POST` | `/apisite/privateservers/v1/servers/{id}/link` | Generate invite `{linkCode}` | 🔐 Session |
| 7 | `DELETE` | `/apisite/privateservers/v1/servers/{id}/link` | Revoke invite | 🔐 Session |
| 8 | `POST` | `/apisite/privateservers/v1/servers/{id}/shutdown` | Kill instance | 🔐 Session |
| 9 | `POST` | `/apisite/privateservers/v1/servers/{id}/renew` | Extend expiry | 🔐 Session |
| 10 | `GET` | `/apisite/privateservers/v1/links/{linkCode}` | Resolve invite link | 🔓 Public |
| 11 | `GET` | `/apisite/privateservers/v1/join-script` | `?privateServerId=` or `?linkCode=` → join payload | 🔐 Session |

### support

Wrapper base: `/apisite/support/v1`.

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/support/v1/me` | Requester identity `{isStaff}` | 🔐 Session |
| 2 | `GET` | `/apisite/support/v1/categories` | Ticket categories | 🔐 Session |
| 3 | `GET` | `/apisite/support/v1/tickets` | `?all=true` (staff: all tickets) | 🔐/🛡️ Staff |
| 4 | `GET` | `/apisite/support/v1/tickets/{id}` | Ticket detail | 🔐 Session |
| 5 | `POST` | `/apisite/support/v1/tickets` | Open `{category,subject,message}` | 🔐 Session |
| 6 | `POST` | `/apisite/support/v1/tickets/{id}/reply` | `{message}` | 🔐 Session |
| 7 | `POST` | `/apisite/support/v1/tickets/{id}/status` | `{status}` | 🔐 Session |
| 8 | `POST` | `/apisite/support/v1/tickets/{id}/attachments` | Multipart upload | 🔐 Session |

### api (alerts)

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/api/alerts/alert-info` | Site-wide banner `{IsVisible,Text,LinkText,LinkUrl}` | 🔓 Public |

### locale

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/locale/v1/locales/set-user-supported-locale` | `{supportedLocaleCode}` | 🔐 Session |

### donate

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/donate/v1/settings` | Donation config for `/donate` | 🔓 Public |

### safechat

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` / `POST` | `/apisite/safechat/v1/settings` | Safe-chat toggle `{enabled}` | 🔐 Session |

### paidaccess

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/paidaccess/v1/places/{placeId}` | Paid-access info | 🔓 Public |
| 2 | `PATCH` | `/apisite/paidaccess/v1/places/{placeId}` | Set price `{price}` | 🔐 Session |
| 3 | `POST` | `/apisite/paidaccess/v1/places/{placeId}/purchase` | Buy access | 🔐 Session |
| 4 | `GET` | `/apisite/paidaccess/v1/places/{placeId}/group-roles` | Roles exempt from payment | 🔐 Session |
| 5 | `PATCH` | `/apisite/paidaccess/v1/places/{placeId}/group-roles` | `{roleIds:[...]}` | 🔐 Session |

### rbxcdn

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/rbxcdn/static/images/Template-Shirts-R15_07262019.png` | Clothing upload template (used by /develop) | 🔓 Public |

---

## 🕸 Legacy web endpoints

Classic Roblox-era routes still live on the main domain. These are what the frontend hits
directly — no `/apisite` prefix.

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/game/get-join-script?placeId=` | `{prefix,joinScriptUrl,joinUrl}` for launching | 🔐 Session |
| 2 | `GET` | `/game/get-join-script-fromjobid?placeId=&jobId=` | Rejoin a running server | 🔐 Session |
| 3 | `GET` | `/game/get-studio-link?placeId=` | Studio launch payload | 🔐 Session |
| 4 | `GET` | `/game/get-temp-studio-link?placeId=` | Temporary studio link | 🔐 Session |
| 5 | `GET` | `/games/getgameinstancesjson?placeId=&startIndex=` | Live server instances | 🔐 Session |
| 6 | `GET` | `/rcc/killserver?placeId=&jobId=` | Kill one server job | 🛡️ Staff |
| 7 | `GET` | `/rcc/killallservers?placeId=` | Kill every server of a place | 🛡️ Staff |
| 8 | `POST` | `/universes/create` | `{templatePlaceIdToUse,groupId}` → new universe | 🔐 Session |
| 9 | `GET` | `/universes/get-universe-places?universeId=` | Places in a universe | 🔐 Session |
| 10 | `GET` | `/v1/gametemplates` | Create-place templates | 🔐 Session |
| 11 | `GET` | `/search/users/results?keyword=&maxRows=&startIndex=` | User search | 🔓 Public |
| 12 | `GET` | `/users/inventory/list-json?userId=&assetTypeId=&cursor=&itemsPerPage=` | Legacy inventory | 🔓 Public |
| 13 | `GET` | `/users/favorites/list-json?userId=&assetTypeId=&cursor=&itemsPerPage=` | Legacy favorites | 🔓 Public |
| 14 | `GET` | `/users/profile/robloxcollections-json?userId=` | Profile collection | 🔓 Public |
| 15 | `GET` | `/comments/get-json?assetId=&startIndex=&thumbnailWidth=100&thumbnailHeight=100&thumbnailFormat=PNG&cachebuster=` | Item comments | 🔓 Public |
| 16 | `POST` | `/comments/post` | `{text,assetId}` — rate/captcha limited | 🔐 Session |
| 17 | `POST` | `/asset/toggle-profile` | Pin to profile `{assetId,addToProfile}` | 🔐 Session |
| 18 | `GET` | `/asset/status?assetId=` | Asset moderation status | 🔐 Session |
| 19 | `GET` | `/asset/?id={id}` | Raw asset stream (audio, models) | 🔐 Session |
| 20 | `POST` | `/develop/upload` | Multipart: `name,assetType,file,groupId,description,universeId` | 🔐 Session |
| 21 | `POST` | `/develop/ugc-upload` | UGC flow variant | 🔐 Session |
| 22 | `POST` | `/develop/upload-version` | Push new version of an asset | 🔐 Session |
| 23 | `GET` | `/user-sponsorship/{type}` | Ad-banner HTML (`type` 1=728x90, 2=160x620, 3=300x270) | 🔓 Public |
| 24 | `GET` / `PATCH` | `/v1/themes/User/{userId}` | Site theme (`{themeType}` = Light/Dark) | 🔐 Session |
| 25 | `GET` | `/admin-api/api/pending-counts` | Staff queue counters (moderation) | 🛡️ Staff |
| 26 | `GET` | `/markdown/{page}` | Help-article markdown | 🔓 Public |
| 27 | `GET` | `/thumbs/asset.ashx?assetId=&width=&height=` | Legacy thumbnail redirect | 🔓 Public |

---

## 🎮 Game launch protocol

```text
1.  GET /game/get-join-script?placeId=123
    → { "prefix": "<scheme>", "joinScriptUrl": "<auth ticket url>", ... }

2.  The web client first tries the local helper:
      POST http://127.0.0.1:4401/launch   (fallback :4400)
      body: { "uri": "<prefix><joinScriptUrl>" }

3.  If no helper answers, it clicks a hidden
      <a href="{prefix}{joinScriptUrl}">
    which hands off to the desktop client via its URI scheme.
```

> [!TIP]
> Ports `4400` / `4401` are hardcoded in the `_app` bundle. Studio uses the same
> handshake through `get-studio-link` / `get-temp-studio-link`.

---

## 💿 Client downloads

| Client | Year | URL |
|---|---|---|
| Fluxer Studio | 2017 | `https://setup.fluxr.lol/FluxerStudio2017.zip` |
| Fluxer Studio | 2018 | `https://setup.fluxr.lol/FluxerStudio2018.zip` |
| Fluxer Studio | 2020 | `https://setup.fluxr.lol/FluxerStudio2020.zip` |
| Fluxer Studio | 2021 | `https://setup.fluxr.lol/FluxerStudio2021.zip` |
| Fluxer Player | — | hosted behind `https://fluxr.lol/downloads` |

---

## ⚡ Quick examples

> [!TIP]
> Start with the 🔓 **Public** endpoints — they need nothing but `curl`. Anything marked
> 🔐 **Session** needs logged-in cookies *and* the CSRF retry loop from the auth section.
> Anything marked 🛡️ **Staff** is off-limits, full stop.

```bash
# public profile
curl https://www.fluxr.lol/apisite/users/v1/users/1

# avatar renders for ids 1,2,3
curl "https://www.fluxr.lol/apisite/thumbnails/v1/users/avatar?userIds=1,2,3&size=420x420&format=png"

# game sorts → sort tokens → game list
curl "https://www.fluxr.lol/apisite/games/v1/games/sorts?gameSortsContext=Home"
curl "https://www.fluxr.lol/apisite/games/v1/games/list?sortToken=popular&maxRows=10&genre=0"

# search users (legacy route, works unauthenticated)
curl "https://www.fluxr.lol/search/users/results?keyword=flux&maxRows=10&startIndex=0"

# authenticated call — needs session cookie + csrf retry logic
curl -b cookies.txt -H "x-csrf-token: <token>" \
  "https://www.fluxr.lol/apisite/users/v1/users/authenticated"
```

---

## 🧻 Notes & gotchas

> [!CAUTION]
> The 🛡️ **Staff**-marked routes (`/rcc/killserver`, `/rcc/killallservers`, `/admin-api/*`,
> group audit logs) are staff-only. They are listed for completeness because they're
> visible in the public bundles — **do not probe or automate them**.

> [!WARNING]
> - The whole platform is **one Next.js build ID away from breaking** — paths can shift
>   without notice on any deploy.
> - Commenting, uploading and ads are rate-limited and captcha-gated server-side.
> - Respect fluxr.lol's Terms of Use (`/tos`). Don't spam, scrape aggressively, or
>   automate anything abusive — the operators already had to lock signup behind Discord
>   verification because of malicious activity.

<details>
<summary>📊 Coverage summary</summary>

| Bucket | Count |
|---|:-:|
| Gateway endpoints (`/apisite/*`, 31 services) | 274 |
| Legacy web routes | 28 |
| **Total documented calls** | **302** |

Extracted from build `OQlDqchIHbJOf0zyI7o0L` · verified 2026-11-13.
</details>

<p align="center">
  <sub>mapped by hand from the public client bundles · corrections welcome via PR</sub>
</p>
