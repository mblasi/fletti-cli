# fletti-cli — CLAUDE.md

Go CLI (binary name `flesh`, Makefile:2) for administering the Fletti proximity-marketplace/delivery platform through the public REST API. It is a thin, admin/ops-oriented HTTP client over `https://api.fletti.com` — it holds no business logic beyond request assembly and pretty-printing (including ASCII-art map/avatar rendering in the terminal).

- Module: `github.com/fletticorp/fletti-cli`, Go 1.13 (go.mod:1-3)
- Frameworks: cobra (commands), viper (config/token storage), go-staticmaps + nfnt/resize (ASCII maps)
- Entry point: main.go:10 — parses `-config` flag, loads `$HOME/.fletti/<name>.yml` via viper, then `command.Execute()`
- 19 Go files: `main.go`, `command/*.go` (17 files), `errors/root.go` (single `ErrorUnauthorized` sentinel, errors/root.go:5)

## Build / install

- `make install` (Makefile:9-12): builds `bin/flesh`, copies to `$GOPATH/bin/flesh`, seeds `~/.fletti/config.yml` (copy-if-absent via `cp -n`)
- `make all` = clean + golint + `go test ./...` + build (no tests exist)
- `.goreleaser.yml`: cross-builds darwin/linux/windows (386+amd64), archives bundle `config.yml`

## Config & auth

Config file `~/.fletti/config.yml` (main.go:19); shipped default contains only `api_uri: https://api.fletti.com` (config.yml:1). Missing `api_uri` falls back to the same default (root.go:44, root.go:87-94). Override file with `flesh -config <name>` (name without extension, still resolved under `~/.fletti/`). Env vars are also read (`viper.AutomaticEnv()`, main.go:15).

Auth flow — browser-based OAuth, tokens persisted into the SAME config.yml by viper:
1. `flesh login` starts a local HTTP server on port **9876** (login.go:91), opens `https://www.fletti.com/login/flesh` in the browser (login.go:108-110); the site redirects back to `localhost:9876/?access_token=...&refresh_token=...`, which are saved via `viper.WriteConfig()` (login.go:95-101).
2. Every authenticated command group uses `PersistentPreRunE: ensureAuth` (login.go:42-73): validates token with `GET /me?authorization=<token>` (login.go:76), tries refresh (`POST /oauth2/flesh/token` with `grant_type=refresh_token`, token.go:60), and finally falls back to a full browser login.
3. **Tokens are sent as a query parameter** `?authorization=<access_token>` on every call — never as an Authorization header. Keep this convention when adding commands.

Impersonation ("i15n"): admin-only. `flesh i15n <nickname>` resolves the user (`GET /users/{nick}`), fetches delegated tokens via `GET /oauth2/impersonalize/{userID}/token?authorization=...` (login.go:154), and stores them under viper keys `<nickname>.access_token` / `<nickname>.refresh_token` (login.go:198-199). Most command groups then accept `-i/--impersonalize <nickname>` (default `me`); `getToken()`/`getRefreshToken()` key off that flag (root.go:96-111). Every run logs "Running as me" or a yellow impersonation warning (root.go:79-85).

## Command catalog (command → API call)

All URLs are `<api_uri>` + path. `GET/POST/PUT` helpers live in root.go:113-191.

| Command | API call | Notes |
|---|---|---|
| `login` | browser OAuth (see above) | login.go:38 |
| `i15n <nickname>` | GET `/users/{nick}`; GET `/oauth2/impersonalize/{id}/token` | login.go:176 |
| `token refresh` | POST `/oauth2/flesh/token` (grant_type=refresh_token) | token.go:48; honors `-i` |
| `token show` | none (prints stored token) | token.go:89 |
| `ping` | GET `/ping` | ping.go:21; unauthenticated |
| `who am i` | GET `/me` | prints only `general_profile` (who.go:52) |
| `users me` | GET `/me` | users.go:90 |
| `users roles` | GET `/roles` | users.go:122 |
| `users show <nick>` | GET `/users/{nick}` | users.go:136 |
| `users lkl <nick>` | GET `/users/{nick}` | renders `last_known_location` as ASCII static map (users.go:150-194) |
| `users avatar <nick>` | GET `/users/{nick}`; GET `/photos/{id}/avatar` | base64 JPEG → ASCII art (users.go:196) |
| `users new` | POST `/users` body `{"all": false}` | "list new users" — yes, a POST (users.go:256) |
| `account show` | GET `/billing/account` | account.go:31 |
| `clouds show` | GET `/clouds` | account clouds (fleets/tenants) the user belongs to (clouds.go:31) |
| `drones available [zones]` | GET `/drones?zones=<z>` | drones.go:60 |
| `drones show <id>` | GET `/drones/{id}` | drones.go:77 |
| `drones on <id>` / `off <id>` | PUT `/drones/{id}/on` / `/off` | drones.go:89-111; see PUT gotcha below |
| `ecommerce me/purchases/sales/token` | GET `/ecommerce/{me,purchases,sales,token}` | ecommerce.go:59-111 |
| `maps geocode <address>` | GET `/geocode?address=`; GET `/zones/{lat}/{lng}` | prints JSON + ASCII map; `--width/--height` flags (maps.go:22-23) |
| `offers list` | GET `/offers` | offers.go:51 |
| `offers detail <offerID>` | **not implemented — empty TODO** | offers.go:64-67 |
| `offers new <requestID> <nickname\|all>` | GET `/request/{id}/availability` then POST `/offers` `{request_id, shippers}` | offers.go:69-124; `all` offers to every available shipper |
| `requests last` (`-s/--since`, default `1d`) | GET `/clouds/requests/last?since=` | requests.go:126 |
| `requests list` | GET `/requests` | requests.go:138 |
| `requests show <id>` | GET `/request/{id}` | requests.go:150 |
| `requests offers <id>` | GET `/request/{id}/offers` | requests.go:162 |
| `requests detail <id>` | GET `/request/{id}` + GET `/request/{id}/remaining` | condensed view; status labels in Spanish (Pendiente…Abortado, requests.go:205-219) |
| `requests price <site> <origin> <dest> [vehicle] [assistants] [floors]` | GET `/geocode` ×2; GET `/route?points=lat,lng,lat,lng`; GET `/price?site_id=&weight=&items=&sections=&vehicle=&distance=&assistants=&floors=` | requests.go:243-289 |
| `requests availability <id>` | GET `/request/{id}/availability` | prints shipper nickname/commitment/reputation (requests.go:313) |
| `requests new <desc> <origin> <dest> <sender me\|cell> <receiver me\|cell> [vehicle]` | GET `/geocode` ×2; GET `/me`; GET `/route?vehicle=&points=`; POST `/request` | builds a single-section request (start/end, dropins/dropoffs); SLA = route `commitment` + 1800 s (requests.go:418) |
| `requests schedule <cron> <desc> <origin> <dest> <sender> <receiver> [vehicle]` | same lookups; POST `/schedule/request` (adds `cron_expression`) | requests.go:423-525 |
| `scheduling list` | GET `/schedule/requests` | scheduling.go:31 |
| `shippers availability` | POST `/shippers` `{}` | flattens `available_shippers` per zone (shippers.go:41) |
| `shippers requests <status>` | GET `/clouds/shipper/requests/{status}` | shippers.go:85 |
| `route availability <origin> <dest> <vehicle>` | GET `/geocode` ×2; GET `/route/availability?route=lat,lng,lat,lng&vehicle=<int>` | route.go:31 |
| `wallet customer` | GET `/wallet/customer` | wallet.go:41 |
| `wallet cards` | GET `/wallet/cards` | wallet.go:55 |

Vehicle mapping (requests.go:13-21, requests.go:291-311): enum `VehicleCategory` 0=bici, 1=car, 2=van, 3=truck, 4=drone. String aliases: `bici`, `moto`→bici(!), `auto`→car, `miniflete`→van, `camion`→truck, `drone`. Unknown strings silently map to bici (zero value). Note `/route` takes the **string** (`?vehicle=bici`), while `/price` and `/route/availability` take the **int** enum.

Note: despite the group name, this CLI does not *create* clouds/drones — it lists/inspects them and toggles drone power. Data creation is limited to requests, scheduled requests, and offers.

## Typical workflows

- First-time: `make install` → `flesh login` (browser) → `flesh who am i`
- Ops smoke test: `flesh ping` (hits `/ping`, no auth)
- E2E delivery seeding (admin): `flesh requests new "caja" "Av. Corrientes 1234" "Callao 100" me 5491155555555 bici` → `flesh requests availability <id>` → `flesh offers new <id> all` → `flesh requests detail <id>`
- Act as another user: `flesh i15n pepito` then any command with `-i pepito`
- Recurring shipment: `flesh requests schedule "0 9 * * 1" ...`

## Gotchas / bugs / TODOs

- **`offers detail` is an unimplemented TODO stub** (offers.go:64-67) — the only explicit TODO in the repo.
- **`PUT()` actually issues an HTTP POST** (`http.Post` at root.go:172) — so `drones on/off` sends POST, not PUT. If the API ever enforces methods this breaks.
- Access tokens travel in URL query strings (logged by proxies/servers). Design decision of the API, not fixable CLI-side alone.
- `login` blocks forever if the browser flow never redirects to `localhost:9876`; the callback server is plain HTTP, hardcoded port.
- Heavy unchecked type assertions on JSON (e.g. requests.go:197-201, users.go:165) — malformed/empty API responses panic instead of erroring.
- `refreshToken()` (token.go:48) doesn't check HTTP status; a failed refresh silently writes `nil` tokens into config.
- README.md:14 points at `github.com/fletti/fletti-cli` but the module is `fletticorp/fletti-cli`; README:55 references API docs at `https://api.fletti.com/docs/index.html` (swagger).
- `maps geocode` map tiles are fetched client-side by go-staticmaps (OSM tile servers), not through the Fletti API.
- drones.go:19 short description says "clouds subcommands" (copy-paste), and `who` requires auth via group but its README-visible parent is undocumented; cosmetic only.
- No tests anywhere (`go test ./...` is a no-op).

## Contribution workflow

All work follows the fork flow — never commit to `fletticorp` directly:
1. Start every change as a feature branch on the `mblasi/fletti-cli` fork (branch from `master`).
2. Open a PR on the fork and merge it there.
3. Finish with a PR from the fork's `master` to `fletticorp/fletti-cli` `master`.
