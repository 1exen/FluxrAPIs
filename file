<h1 align="center">
  <code>fluxr.lol</code> — Full API Reference
</h1>

<p align="center">
  Every endpoint on <strong>Fluxer</strong> (the 2019-era revival platform), pulled from the
  production Next.js bundles and verified against the live gateway.<br>
  <strong>301 endpoints</strong> · <strong>31 services</strong> · last checked <em>2026-11-13</em>
</p>

<p align="center">
  <img alt="endpoints" src="https://img.shields.io/badge/endpoints-301-9cf?style=flat-square">
  <img alt="services" src="https://img.shields.io/badge/services-31-orange?style=flat-square">
  <img alt="base" src="https://img.shields.io/badge/gateway-%2Fapisite-blue?style=flat-square">
  <img alt="auth" src="https://img.shields.io/badge/auth-cookie%20%2B%20csrf-red?style=flat-square">
</p>

---

## Contents

- [How this map was made](#how-this-map-was-made)
- [Architecture](#architecture)
- [Base URLs](#base-urls)
- [Auth & headers](#auth--headers)
- [Response conventions](#response-conventions)
- [API reference](#api-reference)
  - [users](#users) · [accountinformation](#accountinformation) · [accountsettings](#accountsettings) · [auth](#auth) · [avatar](#avatar) · [catalog](#catalog) · [thumbnails](#thumbnails) · [games](#games) · [groups](#groups) · [friends](#friends) · [inventory](#inventory) · [develop](#develop) · [itemconfiguration](#itemconfiguration) · [economy](#economy) · [trades](#trades) · [forums](#forums) · [badges](#badges) · [chat](#chat) · [privatemessages](#privatemessages) · [presence](#presence) · [premiumfeatures](#premiumfeatures) · [ads](#ads) · [billing](#billing) · [privateservers](#privateservers) · [support](#support) · [api](#api-alerts) · [locale](#locale) · [donate](#donate) · [safechat](#safechat) · [paidaccess](#paidaccess) · [rbxcdn](#rbxcdn)
- [Legacy web endpoints](#legacy-web-endpoints)
- [Game launch protocol](#game-launch-protocol)
- [Client downloads](#client-downloads)
- [Quick examples](#quick-examples)
- [Notes](#notes)

---

## How this map was made

No guessing — everything below comes from three sources:

1. **`/_next/static/OQlDqchIHbJOf0zyI7o0L/_buildManifest.js`** — leaks the rewrite table that mounts the gateway under `/apisite/*`, `/thumbs/*`, `/images/*`, `/asset/*`, `/admin/*`, `/v1/*`.
2. **Every page + shared chunk** (82 files, `webpackChunk_N_E`) — all `axios` calls (`Ay`) and the service resolver (`bO`) were parsed out and cross-checked.
3. **Live probes** against the production gateway to confirm status codes and response shapes.

> The frontend builds URLs as `apiFormat: https://www.fluxr.lol/apisite/{service}{path}`.
> So `bO("users", "/v1/users/authenticated")` → `GET https://www.fluxr.lol/apisite/users/v1/users/authenticated`.

---

## Architecture

```
                    ┌──────────────────────────────────────────────┐
                    │                fluxr.lol (Next.js)           │
 browser ──────────►│  www.fluxr.lol                               │
                    │    ├── /apisite/{service}/**  ──► API gateway│
                    │    ├── /thumbs/** /images/**  ──► CDN/thumbs │
                    │    ├── /asset/**              ──► asset host │
                    │    ├── /game/**  /universes/** ──► legacy web│
                    │    └── /admin-api/**          ──► staff tools│
                    └──────────────┬───────────────────────────────┘
                                   │
              desktop client ──────┴────── 127.0.0.1:4400/4401 (/launch)
```

---

## Base URLs

| What | URL |
|---|---|
| Website | `https://www.fluxr.lol` |
| API gateway | `https://www.fluxr.lol/apisite/{service}{path}` |
| Legacy web API | `https://www.fluxr.lol/...` (no prefix) |
| Thumbnail CDN | `https://www.fluxr.lol/images/...` |
| Legacy thumbnail route | `https://www.fluxr.lol/thumbs/asset.ashx` |
| Client download host | `https://setup.fluxr.lol` |

---

## Auth & headers

- **Cookie session.** Login sets httpOnly session cookies; requests to member endpoints must carry them. There is no public token API.
- **CSRF.** Every request sends an `x-csrf-token` header. On a `403` whose response carries a fresh `x-csrf-token`, the client stores it and retries the same request once. Copy this behaviour or mutating calls will bounce.
- **Turnstile.** Signup/login are protected by Cloudflare Turnstile (sitekey `0x4AAAAAAF_V9KGtIJ95V_DV`). Login page: `/auth/accountlogin`, Discord flows: `/auth/discord/start`, `/auth/discord/login`.
- **Unauthenticated calls** return `401` with the standard error envelope.

```json
{ "errors": [ { "code": 0, "message": "Authorization has been denied for this request." } ] }
```

---

## Response conventions

- REST-ish JSON. Collections paginate with `nextPageCursor` / `previousPageCursor` + `limit`.
- Detail endpoints return bare objects; list endpoints wrap in `{"data": [...]}`.
- Legacy (`.aspx`-era) routes use PascalCase payloads (`IsValid`, `Data`, `Items`, `Comments`).
- Thumbnail endpoints return `{"data":[{"targetId","state","imageUrl","version":"TN3"}]}` with `state` one of `Pending / Completed / Blocked / Error`.

<details>
<summary><strong>Live response samples (verified)</strong></summary>

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
{ "sorts": [ { "token": "popular", "name": "Popular", "tokenExpiryInSeconds": 86400, ... },
             { "token": "mostFavorited", ... } ] }

// GET /apisite/thumbnails/v1/users/avatar?userIds=1&size=420x420&format=png
{ "data": [ { "targetId": 1, "state": "Completed",
  "imageUrl": "https://www.fluxr.lol/images/thumbnails/...png", "version": "TN3" } ] }

// GET /apisite/api/alerts/alert-info
{ "IsVisible": true, "Text": "Welcome to Fluxer! ...", "LinkUrl": "https://www.fluxr.lol/support" }
```
</details>

---

## API reference

Legend — `id` = id number. Auth column: `P` public, `A` needs session, `S` staff only.

### users

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/users/v1/users/authenticated` | Current session user | A |
| 2 | `GET` | `/apisite/users/v1/users/{id}` | Public profile (name, description, badges, RAP) | P |
| 3 | `GET` | `/apisite/users/v1/users/{id}/status` | User status blurb | P |
| 4 | `PATCH` | `/apisite/users/v1/users/{id}/status` | Set status `{status}` | A |
| 5 | `PATCH` | `/apisite/users/v1/users/{id}/display-names` | Change display name `{newDisplayName}` | A |
| 6 | `GET` | `/apisite/users/v1/users/{id}/username-history` | Past usernames `?limit=100&cursor=` | P |
| 7 | `POST` | `/apisite/users/v1/usernames/users` | Resolve usernames → ids `{usernames:[...]}` | P |
| 8 | `GET` | `/apisite/users/v1/users/{id}/profile-background` | Profile banner info | P |
| 9 | `POST` | `/apisite/users/v1/profile-background` | Upload profile banner (multipart) | A |
| 10 | `PATCH` | `/apisite/users/v1/profile-background/position` | Banner offset `{posX,posY,pending}` | A |
| 11 | `DELETE` | `/apisite/users/v1/profile-background` | Remove banner | A |

### accountinformation

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/accountinformation/v1/birthdate` | Account birthday | A |
| 2 | `POST` | `/apisite/accountinformation/v1/birthdate` | Set `{birthMonth,birthDay,birthYear}` | A |
| 3 | `GET` | `/apisite/accountinformation/v1/gender` | Account gender | A |
| 4 | `POST` | `/apisite/accountinformation/v1/gender` | Set `{gender}` | A |
| 5 | `POST` | `/apisite/accountinformation/v1/description` | Set `{description}` | A |
| 6 | `GET` | `/apisite/accountinformation/v1/users/{id}/promotion-channels` | Social links `?alwaysReturnUrls=` | P |
| 7 | `POST` | `/apisite/accountinformation/v1/promotion-channels` | Save social links | A |
| 8 | `GET` | `/apisite/accountinformation/v1/users/{id}/roblox-badges` | Classic badge icons | P |
| 9 | `GET` / `POST` | `/apisite/accountinformation/v1/staff-icon` | Read / set `{hidden}` staff-icon visibility | A/S |

### accountsettings

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` / `POST` | `/apisite/accountsettings/v1/trade-privacy` | Who can trade with me | A |
| 2 | `GET` / `POST` | `/apisite/accountsettings/v1/trade-value` | Trade quality filter (`None/Low/Medium/High`) | A |
| 3 | `GET` / `POST` | `/apisite/accountsettings/v1/inventory-privacy` | Inventory visibility (`AllUsers`, `Friends`, `NoOne`, …) | A |
| 4 | `GET` / `POST` | `/apisite/accountsettings/v1/private-message-privacy` | DM privacy | A |
| 5 | `GET` / `POST` | `/apisite/accountsettings/v1/follow-me-privacy` | Who can follow into games | A |
| 6 | `GET` / `POST` | `/apisite/accountsettings/v1/private-server-invite-privacy` | Private-server invite privacy | A |
| 7 | `POST` | `/apisite/accountsettings/v1/avatar-page-style` | Avatar page skin `{avatarPageStyle}` | A |
| 8 | `POST` | `/apisite/accountsettings/v1/users/{id}/block` | Block user | A |
| 9 | `POST` | `/apisite/accountsettings/v1/users/{id}/unblock` | Unblock user | A |
| 10 | `GET` | `/apisite/accountsettings/v1/users/{id}/is-blocked` | `{isBlocked}` | A |

### auth

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/auth/v2/login` | `{ctype:"username",cvalue,password}` | P |
| 2 | `POST` | `/apisite/auth/v1/username` | Legacy login `{username,password}` | P |
| 3 | `GET` | `/apisite/auth/v1/usernames/validate` | Name rules `?username=&context=` | P |
| 4 | `POST` | `/apisite/auth/v2/logout` | Kill current session `{}` | A |
| 5 | `POST` | `/apisite/auth/v2/logoutfromallsessionsandreauthenticate` | Sign out everywhere | A |
| 6 | `POST` | `/apisite/auth/v2/user/passwords/change` | `{currentPassword,newPassword}` | A |
| 7 | `GET` | `/apisite/auth/v2/accounts` | Linked multi-account list | A |
| 8 | `POST` | `/apisite/auth/v2/accounts/switch` | `{userId}` | A |
| 9 | `POST` | `/apisite/auth/v2/accounts/remove` | `{userId}` | A |
| 10 | `GET` / `POST` | `/apisite/auth/v2/ban` (+ `/ban/unlock`) | Ban status / acknowledge `{understood}` | A |

### avatar

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/avatar/v1/avatar` | Own avatar state | A |
| 2 | `GET` | `/apisite/avatar/v1/users/{id}/avatar` | Anyone's avatar | P |
| 3 | `GET` | `/apisite/avatar/v1/avatar-rules` | Rig types, scale ranges, wear limits | P |
| 4 | `POST` | `/apisite/avatar/v1/avatar/set-wearing-assets` | `{assetIds:[...]}` | A |
| 5 | `POST` | `/apisite/avatar/v1/avatar/set-body-colors` | Body color map | A |
| 6 | `POST` | `/apisite/avatar/v1/avatar/set-player-avatar-type` | `{playerAvatarType:1\|2}` (R6→1, R15→2) | A |
| 7 | `POST` | `/apisite/avatar/v1/avatar/set-scales` | Scale object (see avatar-rules) | A |
| 8 | `POST` | `/apisite/avatar/v1/avatar/redraw-thumbnail` | Queue re-render | A |
| 9 | `GET` | `/apisite/avatar/v1/avatar/set-rig?rigtype=R6\|R15` | Toggle rig (fired via `fetch`, not axios) | A |
| 10 | `GET` | `/apisite/avatar/v1/users/{id}/outfits` | `?itemsPerPage=50&page=1` | P |
| 11 | `POST` | `/apisite/avatar/v1/outfits/create` | `{name}` | A |
| 12 | `POST` | `/apisite/avatar/v1/outfits/{id}/wear` | Wear outfit | A |
| 13 | `POST` | `/apisite/avatar/v1/outfits/{id}/rename` | `{name}` | A |
| 14 | `PATCH` | `/apisite/avatar/v1/outfits/{id}` | Update outfit to current look `{}` | A |
| 15 | `POST` | `/apisite/avatar/v1/outfits/{id}/delete` | Delete outfit | A |
| 16 | `GET` | `/apisite/avatar/v1/recent-items/{category}/list` | `category`: `all`, `clothing`, `bodyparts`, `avataranimations`, `accessories`, `outfits`, `gear` | A |

### catalog

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/catalog/v1/catalog/items/details` | Batch details `{items:[{itemType:"Asset",id}]}` | P |
| 2 | `GET` | `/apisite/catalog/v1/search/items` | `?category=&maxRows=&cursor=&keyword=` | P |
| 3 | `GET` | `/apisite/catalog/v1/recommendations/asset/` | `?contextAssetId=&numItems=` | P |
| 4 | `GET` | `/apisite/catalog/v1/favorites/users/{id}/assets/{id}/favorite` | Is favorited | A |
| 5 | `POST` | `/apisite/catalog/v1/favorites/users/{id}/assets/{id}/favorite` | Favorite | A |
| 6 | `DELETE` | `/apisite/catalog/v1/favorites/users/{id}/assets/{id}/favorite` | Unfavorite | A |

### thumbnails

All return `{"data":[…]}`; sizes like `150x150`, `420x420`; `format=png`.

| # | Method | Endpoint | Description |
|:-:|---|---|---|
| 1 | `GET` | `/apisite/thumbnails/v1/users/avatar?userIds=&size=&format=` | Full-body renders |
| 2 | `GET` | `/apisite/thumbnails/v1/users/avatar-headshot?userIds=&size=&format=` | Headshots |
| 3 | `GET` | `/apisite/thumbnails/v1/users/avatar-3d?userIds=&size=&format=` | 3D JSON renders |
| 4 | `GET` | `/apisite/thumbnails/v1/users/outfits?userOutfitIds=&size=&format=` | Outfit renders |
| 5 | `GET` | `/apisite/thumbnails/v1/groups/icons?groupIds=&format=&size=` | Group icons |
| 6 | `GET` | `/apisite/thumbnails/v1/assets?assetIds=&format=&size=` | Item thumbnails |
| 7 | `GET` | `/apisite/thumbnails/v1/games/icons?universeIds=&size=&format=` | Game icons |
| 8 | `GET` | `/apisite/thumbnails/v1/try-on/{assetId}` | 2D "wear on my avatar" render — poll until `state:"ready"` |
| 9 | `GET` | `/apisite/thumbnails/v1/assets/{assetId}/3d` | 3D item model — poll until `state:"ready"` |

### games

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/games/v1/games?universeIds=` | Batch universe info (csv) | P |
| 2 | `GET` | `/apisite/games/v1/games/{universeId}` | Universe detail | P |
| 3 | `GET` | `/apisite/games/v1/games/sorts?gameSortsContext=Home` | Home sorts → `sortToken`s | P |
| 4 | `GET` | `/apisite/games/v1/games/list` | `?sortToken=&maxRows=&genre=&keyword=&year=` | P |
| 5 | `GET` | `/apisite/games/v1/games/recommendations/game/{placeId}?maxRows=` | Similar games | P |
| 6 | `GET` | `/apisite/games/v1/games/multiget-place-details?placeIds=` | Batch place details | P |
| 7 | `GET` | `/apisite/games/v1/games/votes?universeIds=` | Like/dislike counts | P |
| 8 | `PATCH` | `/apisite/games/v1/games/{universeId}/user-votes` | `{vote:true\|false\|null}` | A |
| 9 | `GET` | `/apisite/games/v1/games/game-passes/{id}` | Game-pass info | P |
| 10 | `GET` | `/apisite/games/v1/games/{universeId}/social-links/list` | Universe social links | P |
| 11 | `GET` | `/apisite/games/v1/users/{id}/count` | Games created by user | P |
| 12 | `GET` | `/apisite/games/v2/games/{universeId}/media` | Media assets | P |
| 13 | `GET` | `/apisite/games/v2/groups/{groupId}/games` | `?limit=25&cursor=` | P |

### groups

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/groups/v1/groups/search` | `?keyword=&maxRows=&startIndex=&sort=members` | P |
| 2 | `GET` | `/apisite/groups/v1/groups/{id}` | Group info | P |
| 3 | `GET` | `/apisite/groups/v1/groups/{id}/membership` | Own membership | A |
| 4 | `GET` | `/apisite/groups/v1/groups/{id}/roles` | Role list | P |
| 5 | `GET` | `/apisite/groups/v1/groups/{id}/settings` | Group settings | A |
| 6 | `GET` | `/apisite/groups/v1/groups/{id}/users` | Members `?cursor=&limit=&sortOrder=&search=` | P |
| 7 | `GET` | `/apisite/groups/v1/groups/{id}/roles/{roleId}/users` | Members of a role | P |
| 8 | `GET` | `/apisite/groups/v1/groups/{id}/roles/{roleId}/permissions` | Role permissions | P |
| 9 | `GET` | `/apisite/groups/v1/groups/{id}/join-requests` | `?limit=&cursor=` (owner) | A |
| 10 | `GET` | `/apisite/groups/v1/groups/{id}/audit-log` | `?cursor=&action=&userId=&sortOrder=desc&limit=100` | S |
| 11 | `POST` | `/apisite/groups/v1/groups/create` | Create group | A |
| 12 | `POST` | `/apisite/groups/v1/groups/{id}/users` | Join group | A |
| 13 | `DELETE` | `/apisite/groups/v1/groups/{id}/users/{userId}` | Kick member | A |
| 14 | `PATCH` | `/apisite/groups/v1/groups/{id}/users/{userId}` | Change rank `{roleId}` | A |
| 15 | `POST` / `DELETE` | `/apisite/groups/v1/groups/{id}/join-requests/users/{userId}` | Accept / decline join request | A |
| 16 | `POST` | `/apisite/groups/v1/groups/{id}/claim-ownership` | Claim abandoned group | A |
| 17 | `PATCH` | `/apisite/groups/v1/groups/{id}/change-owner` | `{userId}` | A |
| 18 | `PATCH` | `/apisite/groups/v1/groups/{id}/description` | `{description}` | A |
| 19 | `PATCH` | `/apisite/groups/v1/groups/{id}/settings` | `{isApprovalRequired,areEnemiesAllowed,areGroupFundsVisible,areGroupGamesVisible}` | A |
| 20 | `PATCH` | `/apisite/groups/v1/groups/{id}/status` | `{message}` | A |
| 21 | `POST` | `/apisite/groups/v1/groups/{id}/rolesets/create` | `{name,description,rank}` | A |
| 22 | `PATCH` / `DELETE` | `/apisite/groups/v1/groups/{id}/rolesets/{roleId}` | Edit / delete role | A |
| 23 | `PATCH` | `/apisite/groups/v1/groups/{id}/roles/{roleId}/permissions` | `{permissions}` | A |
| 24 | `PATCH` | `/apisite/groups/v1/groups/icon?groupId=` | Upload group icon (multipart) | A |
| 25 | `POST` | `/apisite/groups/v1/groups/{id}/wall/posts` | `{body}` | A |
| 26 | `GET` | `/apisite/groups/v2/groups/{id}/wall/posts` | `?sortOrder=&limit=&cursor=` | P |
| 27 | `DELETE` | `/apisite/groups/v1/groups/{id}/wall/posts/{postId}` | Delete wall post | A |
| 28 | `POST` | `/apisite/groups/v1/groups/{id}/payouts` | `{PayoutType:"FixedAmount",Recipients:[...]}` | A |
| 29 | `GET` | `/apisite/groups/v1/users/{id}/groups/roles` | User's group roles | P |
| 30 | `GET` | `/apisite/groups/v1/users/{id}/groups/primary/role` | Primary group role | P |
| 31 | `POST` | `/apisite/groups/v1/user/groups/primary` | Set primary `{groupId}` | A |
| 32 | `DELETE` | `/apisite/groups/v1/user/groups/primary` | Clear primary | A |

### friends

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/friends/v1/users/{id}/friends/statuses?userIds=` | Friend state per user | A |
| 2 | `GET` | `/apisite/friends/v1/users/{id}/followers` | `?cursor=&sort=&limit=` | P |
| 3 | `GET` | `/apisite/friends/v1/users/{id}/followers/count` | Count | P |
| 4 | `GET` | `/apisite/friends/v1/users/{id}/followings` | `?cursor=&sort=&limit=` | P |
| 5 | `GET` | `/apisite/friends/v1/users/{id}/followings/count` | Count | P |
| 6 | `GET` | `/apisite/friends/v1/user/friend-requests/count` | Pending count | A |
| 7 | `GET` | `/apisite/friends/v1/my/friends/requests` | `?limit=&cursor=` | A |
| 8 | `POST` | `/apisite/friends/v1/users/{id}/request-friendship` | Send request | A |
| 9 | `POST` | `/apisite/friends/v1/users/{id}/accept-friend-request` | Accept | A |
| 10 | `POST` | `/apisite/friends/v1/users/{id}/decline-friend-request` | Decline | A |
| 11 | `POST` | `/apisite/friends/v1/users/{id}/unfriend` | Remove friend | A |
| 12 | `POST` | `/apisite/friends/v1/users/{id}/follow` | Follow | A |
| 13 | `POST` | `/apisite/friends/v1/users/{id}/unfollow` | Unfollow | A |
| 14 | `POST` | `/apisite/friends/v1/user/following-exists` | `{targetUserIds:[...]}` | A |
| 15 | `GET` | `/apisite/friends/v1/my/nicknames` | List nicknames | A |
| 16 | `PUT` / `DELETE` | `/apisite/friends/v1/users/{id}/nickname` | Set (`{nickname}`) / clear | A |

### inventory

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/inventory/v1/users/{id}/assets/collectibles` | `?cursor=&limit=&assetType=` (limiteds) | P |
| 2 | `GET` | `/apisite/inventory/v1/users/{id}/items/Asset/{assetId}` | Instances of an item | P |
| 3 | `GET` | `/apisite/inventory/v2/assets/{assetId}/owners` | `?cursor=&limit=&sortOrder=` | P |
| 4 | `DELETE` | `/apisite/inventory/v2/inventory/asset/{userAssetId}` | Delete owned instance | A |

### develop

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/develop/v1/places/{id}` | Place info | A |
| 2 | `GET` / `PATCH` | `/apisite/develop/v1/places/{id}/place-copy` | Copying on/off `{enabled}` | A |
| 3 | `PATCH` | `/apisite/develop/v1/places/{id}/roblox-place-id` | Link real place `{robloxPlaceId}` | A |
| 4 | `POST` | `/apisite/develop/v1/places/{id}/game-icons/auto-generated` | Auto icon | A |
| 5 | `GET` | `/apisite/develop/v1/universes/{id}` | Universe info | A |
| 6 | `GET` / `PATCH` | `/apisite/develop/v1/universes/{id}/access` | `{privacyType,playableDevices,universeAvatarType,emotesEnabled}` | A |
| 7 | `PATCH` | `/apisite/develop/v1/universes/{id}/max-player-count` | `{maxPlayers}` | A |
| 8 | `PATCH` | `/apisite/develop/v1/universes/{id}/set-year` | `{year}` (retro year) | A |
| 9 | `POST` | `/apisite/develop/v1/universes/{id}/start-place` | `{placeId}` | A |
| 10 | `POST` / `DELETE` | `/apisite/develop/v1/universes/{id}/social-links` (+`/{linkId}`) | Manage links `{type,url,title}` | A |
| 11 | `POST` | `/apisite/develop/v1/universes/{id}/thumbnails/auto-generated` | Auto thumbnail | A |
| 12 | `POST` | `/apisite/develop/v1/universes/{id}/thumbnails/youtube` | `{url}` | A |
| 13 | `POST` | `/apisite/develop/v1/universes/{id}/thumbnails/{id}` | Activate thumbnail | A |
| 14 | `GET` / `POST` | `/apisite/develop/v1/universes/{id}/web-permissions` | Team create `{collaborators,roles}` | A |
| 15 | `GET` | `/apisite/develop/v1/universes/{id}/web-places` | Place list | A |
| 16 | `GET` | `/apisite/develop/v1/universes/{id}/developerproducts?pageSize=25` | Developer products | A |
| 17 | `POST` | `/apisite/develop/v1/universes/{id}/developerproducts?name=&description=&priceInRobux=&iconImageAssetId=` | Create product | A |
| 18 | `POST` | `/apisite/develop/v1/universes/{id}/developerproducts/{id}/update` | `{Name,Description,IconImageAssetId,PriceInRobux}` | A |
| 19 | `GET` | `/apisite/develop/v1/assets/{id}/published-versions` | `?limit=&sortOrder=Desc&cursor=` | A |
| 20 | `POST` | `/apisite/develop/v1/assets/{id}/revert-version?assetVersionNumber=` | Roll back | A |
| 21 | `PATCH` | `/apisite/develop/v1/assets/{id}` | `{name,description,genres,isCopyingAllowed,enableComments,isForSale}` | A |
| 22 | `POST` | `/apisite/develop/v1/assets/upload-gameicon?placeId=` | Multipart icon upload | A |
| 23 | `POST` | `/apisite/develop/v1/assets/upload-thumbnail?universeId=` | Multipart thumbnail upload | A |
| 24 | `GET` | `/apisite/develop/v1/assets/genres` | Genre list | P |

### itemconfiguration

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/itemconfiguration/v1/assets/restrictions?assetIds=` | Moderation restrictions | P |
| 2 | `GET` | `/apisite/itemconfiguration/v1/creations/get-assets?assetType=&limit=&cursor=&groupId=` | Own creations | A |
| 3 | `POST` | `/apisite/itemconfiguration/v1/creations/get-asset-details` | `{assetIds:[...]}` | A |
| 4 | `POST` | `/apisite/itemconfiguration/v1/assets/{id}/update-price` | `{priceInRobux}` | A |

### economy

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/economy/v1/users/{id}/currency` | Robux balance | A |
| 2 | `GET` | `/apisite/economy/v1/groups/{id}/currency` | Group funds | A |
| 3 | `GET` | `/apisite/economy/v1/assets/{id}/resale-data` | Limited resale stats | P |
| 4 | `GET` | `/apisite/economy/v1/assets/{id}/users/{id}/resellable-copies` | Own copies | A |
| 5 | `PATCH` | `/apisite/economy/v1/assets/{id}/resellable-copies/{uaid}` | Reprice `{price}` | A |
| 6 | `POST` | `/apisite/economy/v1/purchases/products/{productId}` | Buy `{assetId,expectedPrice,expectedSellerId,userAssetId,expectedCurrency}` | A |
| 7 | `GET` | `/apisite/economy/v2/users/{id}/transactions` | `?cursor=&transactionType=` | A |
| 8 | `GET` | `/apisite/economy/v2/users/{id}/transaction-totals` | `?timeFrame=&transactionType=summary` | A |
| 9 | `GET` | `/apisite/economy/v2/groups/{id}/transactions` | Group transactions | A |
| 10 | `GET` | `/apisite/economy/v2/groups/{id}/transaction-totals` | Group summary | A |

### trades

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/trades/v1/trades/{type}?cursor=` | `type`: `inbound`, `outbound`, `completed`, `inactive` | A |
| 2 | `GET` | `/apisite/trades/v1/trades/{id}` | Trade detail | A |
| 3 | `GET` | `/apisite/trades/v1/trades/inbound/count` | Badge count | A |
| 4 | `POST` | `/apisite/trades/v1/trades/send` | `{offers:[{robux,userAssetIds,userId},…]}` | A |
| 5 | `POST` | `/apisite/trades/v1/trades/{id}/accept` | Accept | A |
| 6 | `POST` | `/apisite/trades/v1/trades/{id}/decline` | Decline | A |
| 7 | `POST` | `/apisite/trades/v1/trades/{id}/counter` | Counter-offer `{offers:[…]}` | A |

### forums

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/forums/v1/sub-category/{id}/info` | Category info | P |
| 2 | `GET` | `/apisite/forums/v1/sub-category/{id}/posts` | `?limit=&cursor=` | P |
| 3 | `POST` | `/apisite/forums/v1/sub-category/{id}/thread` | New thread `{post,subject}` | A |
| 4 | `GET` | `/apisite/forums/v1/threads/{id}/info` | Thread info | P |
| 5 | `GET` | `/apisite/forums/v1/threads/{id}/replies` | `?limit=&cursor=` | P |
| 6 | `GET` | `/apisite/forums/v1/posts/{id}/info` | Post info | P |
| 7 | `POST` | `/apisite/forums/v1/posts/{id}/reply` | `{post}` | A |
| 8 | `POST` | `/apisite/forums/v1/posts/{id}/mark-as-read` | Read receipt | A |
| 9 | `DELETE` | `/apisite/forums/v1/posts/{id}` | Delete post | A |
| 10 | `GET` | `/apisite/forums/v1/users/{id}/posts` | `?limit=&cursor=` | P |

### badges

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/badges/v1/badges/{id}` | Badge detail | P |
| 2 | `GET` | `/apisite/badges/v2/badges/{id}/basic` | Lightweight detail | P |
| 3 | `GET` | `/apisite/badges/v1/users/{id}/badges` | `?cursor=&limit=` | P |
| 4 | `GET` | `/apisite/badges/v1/universes/{id}/badges` | `?cursor=&limit=` | P |
| 5 | `DELETE` | `/apisite/badges/v1/users/badges/{id}` | Revoke own badge | A |
| 6 | `PATCH` | `/apisite/badges/v1/badges/{id}` | Toggle `{enabled}` | A |

### chat

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` / `POST` | `/apisite/chat/v2/chat-settings` | Read / save chat settings | A |
| 2 | `GET` / `POST` | `/apisite/chat/v2/chat-privacy` | `{inGameChat,websiteChat}` | A |
| 3 | `GET` | `/apisite/chat/v2/get-user-conversations` | `?pageNumber=&pageSize=` | A |
| 4 | `GET` | `/apisite/chat/v2/get-messages` | `?conversationId=&pageSize=&exclusiveStartMessageId=` | A |
| 5 | `GET` | `/apisite/chat/v2/multi-get-latest-messages?conversationIds=` | Latest per conversation | A |
| 6 | `POST` | `/apisite/chat/v2/send-message` | `{conversationId,message}` | A |
| 7 | `POST` | `/apisite/chat/v2/start-one-to-one-conversation` | `{participantUserId}` | A |
| 8 | `POST` | `/apisite/chat/v2/start-group-conversation` | `{participantUserIds,title}` | A |
| 9 | `POST` | `/apisite/chat/v2/add-to-group-conversation` | `{conversationId,participantUserIds}` | A |
| 10 | `POST` | `/apisite/chat/v2/leave-group-conversation` | `{conversationId}` | A |
| 11 | `POST` | `/apisite/chat/v2/rename-group-conversation` | `{conversationId,title}` | A |
| 12 | `POST` | `/apisite/chat/v2/mark-as-read` | `{conversationId,endMessageId}` | A |
| 13 | `POST` | `/apisite/chat/v2/update-user-typing-status` | `{conversationId,isTyping}` | A |
| 14 | `POST` | `/apisite/chat/v2/send-game-link-message` | Share a game `{conversationId,universeId}` | A |

### privatemessages

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/privatemessages/v1/messages` | `?messageTab=&pageSize=&pageNumber=` (tabs: inbox/sent/archive) | A |
| 2 | `GET` | `/apisite/privatemessages/v1/messages/unread/count` | Unread badge | A |
| 3 | `GET` | `/apisite/privatemessages/v1/announcements` | System announcements | P |
| 4 | `POST` | `/apisite/privatemessages/v1/messages/send` | `{recipientid,body,subject,replyMessageId,includePreviousMessage}` | A |
| 5 | `POST` | `/apisite/privatemessages/v1/messages/{archive\|unarchive}` | `{messageIds:[...]}` | A |
| 6 | `POST` | `/apisite/privatemessages/v1/messages/{mark-read\|mark-unread}` | `{messageIds:[...]}` | A |

### presence

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/presence/v1/presence/users` | `{userIds:[...]}` → presence per user | A |

### premiumfeatures

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/premiumfeatures/v1/users/{id}/validate-membership` | Membership check | A |

### ads

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/ads/v1/user-ads/{User\|Group}/{creatorId}` | List ads of creator | P |
| 2 | `POST` | `/apisite/ads/v1/user-ads/{User\|Group}/create?assetId={targetId}` | Multipart `name`, `files` | A |
| 3 | `POST` | `/apisite/ads/v1/user-ads/{adId}/run` | Run campaign `{robux}` | A |

### billing

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/billing/v1/promocodes/redeem` | `{code}` | A |

### privateservers

Wrapper base: `/apisite/privateservers/v1`.

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` / `PATCH` | `/apisite/privateservers/v1/places/{placeId}/settings` | `{enabled,price}` | A |
| 2 | `GET` / `POST` | `/apisite/privateservers/v1/places/{placeId}/servers` | List / create (`{name}`) | A |
| 3 | `GET` / `PATCH` / `DELETE` | `/apisite/privateservers/v1/servers/{id}` | Detail / rename `{name,friendsAllowed}` / delete | A |
| 4 | `POST` | `/apisite/privateservers/v1/servers/{id}/members` | Whitelist `{username}` | A |
| 5 | `DELETE` | `/apisite/privateservers/v1/servers/{id}/members/{userId}` | Remove member | A |
| 6 | `POST` | `/apisite/privateservers/v1/servers/{id}/link` | Generate invite `{linkCode}` | A |
| 7 | `DELETE` | `/apisite/privateservers/v1/servers/{id}/link` | Revoke invite | A |
| 8 | `POST` | `/apisite/privateservers/v1/servers/{id}/shutdown` | Kill instance | A |
| 9 | `POST` | `/apisite/privateservers/v1/servers/{id}/renew` | Extend expiry | A |
| 10 | `GET` | `/apisite/privateservers/v1/links/{linkCode}` | Resolve invite link | P |
| 11 | `GET` | `/apisite/privateservers/v1/join-script` | `?privateServerId=` or `?linkCode=` → join payload | A |

### support

Wrapper base: `/apisite/support/v1`.

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/support/v1/me` | Requester identity `{isStaff}` | A |
| 2 | `GET` | `/apisite/support/v1/categories` | Ticket categories | A |
| 3 | `GET` | `/apisite/support/v1/tickets` | `?all=true` (staff: all tickets) | A/S |
| 4 | `GET` | `/apisite/support/v1/tickets/{id}` | Ticket detail | A |
| 5 | `POST` | `/apisite/support/v1/tickets` | Open `{category,subject,message}` | A |
| 6 | `POST` | `/apisite/support/v1/tickets/{id}/reply` | `{message}` | A |
| 7 | `POST` | `/apisite/support/v1/tickets/{id}/status` | `{status}` | A |
| 8 | `POST` | `/apisite/support/v1/tickets/{id}/attachments` | Multipart upload | A |

### api (alerts)

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/api/alerts/alert-info` | Site-wide banner `{IsVisible,Text,LinkText,LinkUrl}` | P |

### locale

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `POST` | `/apisite/locale/v1/locales/set-user-supported-locale` | `{supportedLocaleCode}` | A |

### donate

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/donate/v1/settings` | Donation config for `/donate` | P |

### safechat

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` / `POST` | `/apisite/safechat/v1/settings` | Safe-chat toggle `{enabled}` | A |

### paidaccess

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/paidaccess/v1/places/{placeId}` | Paid-access info | P |
| 2 | `PATCH` | `/apisite/paidaccess/v1/places/{placeId}` | Set price `{price}` | A |
| 3 | `POST` | `/apisite/paidaccess/v1/places/{placeId}/purchase` | Buy access | A |
| 4 | `GET` | `/apisite/paidaccess/v1/places/{placeId}/group-roles` | Roles exempt from payment | A |
| 5 | `PATCH` | `/apisite/paidaccess/v1/places/{placeId}/group-roles` | `{roleIds:[...]}` | A |

### rbxcdn

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/apisite/rbxcdn/static/images/Template-Shirts-R15_07262019.png` | Clothing upload template (used by /develop) | P |

---

## Legacy web endpoints

Classic Roblox-era routes still live on the main domain. These are what the frontend hits
directly (`$_()`/`Xu()` helpers — no `/apisite` prefix).

| # | Method | Endpoint | Description | Auth |
|:-:|---|---|---|:-:|
| 1 | `GET` | `/game/get-join-script?placeId=` | `{prefix,joinScriptUrl,joinUrl}` for launching | A |
| 2 | `GET` | `/game/get-join-script-fromjobid?placeId=&jobId=` | Rejoin a running server | A |
| 3 | `GET` | `/game/get-studio-link?placeId=` | Studio launch payload | A |
| 4 | `GET` | `/game/get-temp-studio-link?placeId=` | Temporary studio link | A |
| 5 | `GET` | `/games/getgameinstancesjson?placeId=&startIndex=` | Live server instances | A |
| 6 | `GET` | `/rcc/killserver?placeId=&jobId=` | Kill one server job | S |
| 7 | `GET` | `/rcc/killallservers?placeId=` | Kill every server of a place | S |
| 8 | `POST` | `/universes/create` | `{templatePlaceIdToUse,groupId}` → new universe | A |
| 9 | `GET` | `/universes/get-universe-places?universeId=` | Places in a universe | A |
| 10 | `GET` | `/v1/gametemplates` | Create-place templates | A |
| 11 | `GET` | `/search/users/results?keyword=&maxRows=&startIndex=` | User search | P |
| 12 | `GET` | `/users/inventory/list-json?userId=&assetTypeId=&cursor=&itemsPerPage=` | Legacy inventory | P |
| 13 | `GET` | `/users/favorites/list-json?userId=&assetTypeId=&cursor=&itemsPerPage=` | Legacy favorites | P |
| 14 | `GET` | `/users/profile/robloxcollections-json?userId=` | Profile collection | P |
| 15 | `GET` | `/comments/get-json?assetId=&startIndex=&thumbnailWidth=100&thumbnailHeight=100&thumbnailFormat=PNG&cachebuster=` | Item comments | P |
| 16 | `POST` | `/comments/post` | `{text,assetId}` — rate/captcha limited | A |
| 17 | `POST` | `/asset/toggle-profile` | Pin to profile `{assetId,addToProfile}` | A |
| 18 | `GET` | `/asset/status?assetId=` | Asset moderation status | A |
| 19 | `GET` | `/asset/?id={id}` | Raw asset stream (audio, models) | A |
| 20 | `POST` | `/develop/upload` | Multipart: `name,assetType,file,groupId,description,universeId` | A |
| 21 | `POST` | `/develop/ugc-upload` | UGC flow variant | A |
| 22 | `POST` | `/develop/upload-version` | Push new version of an asset | A |
| 23 | `GET` | `/user-sponsorship/{type}` | Ad-banner HTML (`type` 1=728x90, 2=160x620, 3=300x270) | P |
| 24 | `GET` | `/v1/themes/User/{userId}` · `PATCH {themeType}` | Site theme (Light/Dark) | A |
| 25 | `GET` | `/admin-api/api/pending-counts` | Staff queue counters (moderation) | S |
| 26 | `GET` | `/markdown/{page}` | Help-article markdown | P |
| 27 | `GET` | `/thumbs/asset.ashx?assetId=&width=&height=` | Legacy thumbnail redirect | P |

---

## Game launch protocol

```
1.  GET /game/get-join-script?placeId=123
    → { "prefix": "<scheme>", "joinScriptUrl": "<auth ticket url>", ... }

2.  The web client first tries the local helper:
      POST http://127.0.0.1:4401/launch   (fallback :4400)
      body: { "uri": "<prefix><joinScriptUrl>" }

3.  If no helper answers, it clicks a hidden
      <a href="{prefix}{joinScriptUrl}">
    which hands off to the desktop client via its URI scheme.
```

Ports `4400`/`4401` are hardcoded in `_app`. Studio uses the same dance through
`get-studio-link` / `get-temp-studio-link`.

---

## Client downloads

| Client | Year | URL |
|---|---|---|
| Fluxer Studio | 2017 | `https://setup.fluxr.lol/FluxerStudio2017.zip` |
| Fluxer Studio | 2018 | `https://setup.fluxr.lol/FluxerStudio2018.zip` |
| Fluxer Studio | 2020 | `https://setup.fluxr.lol/FluxerStudio2020.zip` |
| Fluxer Studio | 2021 | `https://setup.fluxr.lol/FluxerStudio2021.zip` |
| Fluxer Player | — | hosted behind `https://fluxr.lol/downloads` |

---

## Quick examples

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

## Notes

- Counts: **274 gateway endpoints** across **31 services**, **27 legacy routes** = **301** documented calls, plus the CDN paths.
- **Not all endpoints are equal.** `P` endpoints answered `200` unauthenticated during testing;
  `A` returned `401`; `S` routes (`/rcc/*`, `/admin-api/*`) need staff and are listed for
  completeness only — don't poke them.
- Poll-based render endpoints (`try-on`, `assets/{id}/3d`) answer `Pending` until the renderer
  finishes; the site polls every 1.5 s up to 80 attempts.
- The gateway is a thin proxy — Roblox-style envelope, cursors and error codes match the
  2019-era conventions, but nothing here is affiliated with Roblox Corporation.
- Documented from public client bundles for interoperability/research. Respect fluxr.lol's
  ToS, don't automate anything abusive, don't spam their servers, and expect breaking changes
  without notice — the whole thing is one Next.js build ID away from shifting.

<p align="center">
  <sub>mapped by hand from <code>OQlDqchIHbJOf0zyI7o0L</code> · corrections welcome via PR</sub>
</p>
