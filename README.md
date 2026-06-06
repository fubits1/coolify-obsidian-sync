# Obsidian Self-hosted LiveSync on Coolify

CouchDB backend for the [Self-hosted LiveSync](https://github.com/vrtmrz/obsidian-livesync) Obsidian plugin, packaged for deployment via [Coolify](https://coolify.io). Derived from the upstream Coolify service template proposed in open PR [coollabsio/coolify#9822](https://github.com/coollabsio/coolify/pull/9822).

This is not Obsidian's first-party paid Sync product (no self-hostable engine exists for that). Self-hosted LiveSync is the community plugin that replicates a vault to a CouchDB you control.

## The fundamental wall: Coolify hides the Domains UI for CouchDB

Verified in Coolify source:

- [`bootstrap/helpers/constants.php`](https://github.com/coollabsio/coolify/blob/main/bootstrap/helpers/constants.php) declares `const DATABASE_DOCKER_IMAGES = [..., 'couchdb', ...]`. The string `couchdb` is hard-coded as a database image.
- [`resources/views/livewire/project/application/general.blade.php`](https://github.com/coollabsio/coolify/blob/main/resources/views/livewire/project/application/general.blade.php) at line 73 renders the per-service Domains/URL field only when `!isDatabaseImage($service.image)`. For `couchdb:3.4.2` that returns false, so the field is hidden.

Consequence: deploying this compose via Coolify Docker Compose Build Pack (Public Repository or Docker Compose Empty) on any current Coolify release surfaces no URL/Domains field for the `couchdb` service in the General tab, regardless of compose contents. Coolify deliberately suppresses HTTP routing UI for images it classifies as databases.

Three viable paths follow from this. Pick one.

### Path A: Wait for PR #9822 to land in Coolify's catalog

PR [coollabsio/coolify#9822](https://github.com/coollabsio/coolify/pull/9822) adds `obsidian-livesync` to Coolify's built-in service catalog under `templates/compose/`. Catalog deploys go through a separate code path that does not call `isDatabaseImage`. When (if) it merges, deploying "Obsidian LiveSync" from Coolify's catalog dropdown works end-to-end with zero manual UI input. Currently the PR has unresolved Copilot AI review comments; no merge timeline.

### Path B: Front CouchDB with a Caddy (or nginx) reverse proxy

Add a second service (Caddy or nginx) to the compose. The proxy is not a database image, so Coolify renders its Domains field and auto-allocates an FQDN through the standard magic-var path (e.g. `SERVICE_FQDN_CADDY_80`). The proxy terminates HTTPS at the Coolify-served FQDN and forwards to `http://couchdb:5984` on the internal compose network. CouchDB never gets a direct Traefik label, never gets exposed externally, and the user does no manual Coolify configuration. Trade-off: extra service in the stack, plus a small Caddyfile or nginx.conf checked into the repo.

### Path C: Current compose, explicit Traefik labels, manual `FQDN` env var

`docker-compose.yaml` in this repo declares Traefik labels directly so Coolify's parser doesn't need to inject them, and references `${FQDN}` as a plain env var the user sets manually in Coolify's Environment Variables tab. Works on any Coolify ≥ v4.0.0-beta.411. Trade-off: one manual env var per deploy.

## Compose alignment with PR #9822

`docker-compose.yaml` mirrors the upstream PR's `templates/compose/obsidian-livesync.yaml` at head sha `e2dbf78e496b6fd6ca36a84381df6c5351b46fa6` byte-for-byte (modulo a trailing newline). That includes:

- All catalog header comments (`# documentation:` ... `# port: 5984`).
- The `SERVICE_FQDN_COUCHDB_5984` magic var marker.
- `SERVICE_USER_COUCHDB` and `SERVICE_PASSWORD_64_COUCHDB` magic vars for credentials.
- `MAX_DOCUMENT_SIZE` and `MAX_HTTP_REQUEST_SIZE` overrides with PR defaults.
- The `volumes: - type: bind ... content: |` extension that inlines `local.ini`.
- The healthcheck verbatim.

`local.ini` is also shipped as a real file at the repo root, mirroring the PR's inline INI byte-for-byte. This is for local OrbStack validation (which does not honour Coolify's `content:` extension) and as a fallback Coolify can bind-mount if the `content:` extension is ever stripped.

This alignment means: when PR #9822 lands in Coolify's catalog, deploying from the catalog dropdown will run this exact YAML through Coolify's catalog code path, which handles `isDatabaseImage` correctly (catalog deploys bypass the Domains-UI hide) and processes `SERVICE_FQDN_*` magic vars end-to-end. Path A becomes the trivial recommendation.

Until that merges, deploying this compose via the Docker Compose build pack hits the database-image wall above. The compose stays canonical so a future catalog merge requires zero migration work.

## Path B (reverse proxy) details

Not implemented in this repo yet. Sketch:

```yaml
services:
  caddy:
    image: caddy:2
    environment:
      - SERVICE_FQDN_CADDY_80
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    depends_on:
      - couchdb
  couchdb:
    image: couchdb:3.4.2
    # ... unchanged from canonical PR shape, minus SERVICE_FQDN_COUCHDB_5984 ...

volumes:
  caddy-data:
  caddy-config:
  couchdb-data:
```

`Caddyfile`:

```
{$CADDY_FQDN} {
    reverse_proxy couchdb:5984
}
```

The `SERVICE_FQDN_CADDY_80` magic var works because `caddy` is not in `DATABASE_DOCKER_IMAGES`. Coolify renders Caddy's Domains UI, auto-allocates an FQDN under the wildcard, terminates TLS at Caddy, forwards to `couchdb:5984` on the internal compose network. CouchDB stays private.

This route requires a separate `docker-compose.path-b.yaml` (or similar) to keep `docker-compose.yaml` aligned with PR #9822. Open an issue or ask if you want it generated.

## Local validation with OrbStack

The local compose does not use Coolify's `content:` extension. It bind-mounts the on-disk `./local.ini` directly so plain Docker can parse it.

## Local validation with OrbStack

```bash
cp .env.local.example .env.local
# Edit .env.local if you want different creds.
docker compose -f docker-compose.local.yaml --env-file .env.local up -d
docker compose -f docker-compose.local.yaml --env-file .env.local ps   # wait for (healthy)
```

Validation (substitute the password from `.env.local`):

```bash
USER=admin; PW=changeme-local-only-at-least-rotate-before-prod

# 1. CouchDB up + auth working
curl -fsS -u "$USER:$PW" http://localhost:5984/_up
# {"status":"ok","seeds":{}}

# 2. CORS configured for Obsidian
curl -fsS -u "$USER:$PW" http://localhost:5984/_node/_local/_config/cors
# origins includes app://obsidian.md, capacitor://localhost

# 3. require_valid_user + max_http_request_size applied
curl -fsS -u "$USER:$PW" http://localhost:5984/_node/_local/_config/chttpd
# {"require_valid_user":"true","max_http_request_size":"67108864",...}

# 4. CORS preflight from Obsidian desktop origin
curl -sI -X OPTIONS \
  -H "Origin: app://obsidian.md" \
  -H "Access-Control-Request-Method: GET" \
  http://localhost:5984/
# HTTP/1.1 204  +  Access-Control-Allow-Origin: app://obsidian.md
```

Teardown:

```bash
docker compose -f docker-compose.local.yaml --env-file .env.local down -v
```

## Create the vault database

The canonical template does not auto-create the vault database. You have three options:

Use the plugin wizard (recommended for first-time setup). The LiveSync plugin offers to create the database during the Remote Database wizard.

Use the Fauxton UI for manual setup: open `https://<fqdn>/_utils/`, log in, select Create Database, name it `obsidian` or any vault id, and select Non-partitioned.

Use curl: `curl -fsS -u "$USER:$PW" -X PUT https://<fqdn>/obsidian`.

## Obsidian plugin setup

### Install

Obsidian > Settings > Community plugins > Browse > "Self-hosted LiveSync" > Install and Enable.

### Pick a setup method

The plugin offers three methods:

1. Using setup URIs (recommended once you have a working primary device; see the Setup URI flow section)
2. Minimal setup (the wizard prompts only for bare essentials)
3. Fully manual setup (all settings exposed at once)

For a first device, use Minimal or Manual.

### Remote configuration

In the Remote Database wizard or settings:

| Field | Value |
|---|---|
| Remote type | CouchDB |
| URI | Production: `https://<your-coolify-fqdn>`. Local desktop on the same Mac: `http://localhost:5984`. Local mobile or LAN clients: see below. |
| Username | `SERVICE_USER_COUCHDB` value |
| Password | `SERVICE_PASSWORD_64_COUCHDB` value |
| Database name | `obsidian` (or your chosen vault id) |

Click Test Database Connection. All fields should pass.

Local URI per client type:

| Client | URI |
|---|---|
| Desktop Obsidian on the Mac running OrbStack | `http://localhost:5984` |
| Desktop or mobile Obsidian on the same LAN | `http://couchdb.obsidian.orb.local:5984` (OrbStack mDNS, resolves on macOS host and LAN devices on the same network) or `http://<mac-lan-ip>:5984` |
| Mobile Obsidian over LTE / different network | Expose via Cloudflare Tunnel, ngrok, or Tailscale Funnel. Plain HTTP over the internet will fail CORS preflight from `capacitor://localhost`; use HTTPS. |

Coolify production deploys handle this for you: Traefik fronts the FQDN with a Let's Encrypt cert, so the URI is just `https://<allocated>.<wildcard>` from any device.

### End-to-End Encryption

Enable in plugin settings under End-to-End Encryption and set a vault passphrase. This encrypts every note before it leaves your device and the server stores only ciphertext. The passphrase is not the same as the Setup URI passphrase (see the Setup URI flow section). Save both somewhere outside Obsidian. The same E2EE passphrase must be entered on every device that joins this vault.

### Obfuscate Properties

Optional setting in the same dialog. Hides file paths, sizes, and creation/modification dates from the server for extra privacy, at the cost of a small amount of metadata fidelity.

## Setup URI flow (multi-device)

The Setup URI is an `obsidian://setuplivesync?settings=<encrypted-blob>` URL that bundles every LiveSync setting (server URL, user/pass, E2EE passphrase, db name). Generate it once on a working device, share it to others, and skip manual setup.

### Device A: generate

After your primary device is fully configured and synced, run the Command palette and execute `Copy settings as a new setup URI`, or go to plugin Settings and click `Copy the current settings to a Setup URI`.

When prompted, enter a URI passphrase (encrypts the URI itself, distinct from the vault E2EE passphrase). Save it somewhere outside Obsidian.

The URI lands in your clipboard:

```
obsidian://setuplivesync?settings=<encrypted>
```

### Device B: import

Run Command palette and execute `Use the copied setup URI` (formerly `Open setup URI`), paste the URI, enter the URI passphrase, and pick `Set it up as secondary or subsequent device`.

Sync starts automatically.

### Generate URI without opening Obsidian (CLI)

Requires [Deno](https://deno.com/):

```bash
deno run -A https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/flyio/generate_setupuri.ts \
  --hostname=https://<your-coolify-fqdn> \
  --database=obsidian \
  --username=<SERVICE_USER_COUCHDB> \
  --password=<SERVICE_PASSWORD_64_COUCHDB> \
  --passphrase=<E2EE-vault-passphrase>
```

Outputs the `obsidian://setuplivesync?...` URI and an adjective-noun-number URI passphrase.

## CouchDB configuration applied

The bind-mounted `local.ini` writes the following at boot:

| Section | Key | Value |
|---|---|---|
| `couchdb` | `single_node` | `true` |
| `couchdb` | `max_document_size` | `52428800` (50 MB; edit `local.ini` to tune) |
| `chttpd` | `require_valid_user` | `true` |
| `chttpd` | `max_http_request_size` | `67108864` (64 MB; edit `local.ini` to tune) |
| `chttpd_auth` | `require_valid_user` | `true` |
| `chttpd_auth` | `authentication_redirect` | `/_utils/session.html` |
| `httpd` | `WWW-Authenticate` | `Basic realm="couchdb"` |
| `httpd` | `enable_cors` | `true` |
| `cors` | `origins` | `app://obsidian.md,capacitor://localhost,http://localhost` |
| `cors` | `credentials` | `true` |
| `cors` | `headers` | `accept, authorization, content-type, origin, referer` |
| `cors` | `methods` | `GET, PUT, POST, HEAD, DELETE` |
| `cors` | `max_age` | `3600` |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Two links in Coolify Links panel, one missing scheme | `SERVICE_FQDN_X` returns domain-only (per [Coolify env-var docs](https://coolify.io/docs/knowledge-base/environment-variables)); Coolify renders both URL and FQDN forms | Use the `https://` one |
| 503 "no available server" from FQDN; `docker inspect` shows no `traefik.*` labels | Coolify parser bailed on a non-standard compose field (e.g. `volumes: ... content:`) and skipped Traefik-label injection | See "Why this deviates from upstream Coolify PR #9822" at the top of this README |
| Manually adding `SERVICE_FQDN_COUCHDB_5984` in Coolify Env tab gets removed on save | Same root cause: parser failure prevents Coolify from registering the magic var | Same fix |
| `_up` returns 401 from anywhere except the healthcheck | `require_valid_user=true` is set; every endpoint needs Basic Auth | Pass `-u user:pass` to curl |
| Obsidian client reports CORS error | Wrong scheme or origin not in allowlist | Confirm URI uses `https://` and `cors/origins` includes `app://obsidian.md` (desktop) and `capacitor://localhost` (iOS/Android) |
| Mobile sync hangs on LTE | Incomplete TLS chain at FQDN | `curl -vI https://<fqdn>/` from a mobile-like client; ensure Coolify served full chain |
| Lost admin password | CouchDB hashed it into `_users` on the volume | Restore the original Coolify-generated value, or `docker compose down -v` (wipes vault) and start over |
| OrbStack: init script can't reach `couchdb:5984` (if you re-add a sidecar) | OrbStack injects `HTTP_PROXY=proxyproxy.orb.internal` into every container | Add `NO_PROXY=*` to the sidecar service |

## Sources

### Coolify upstream. Obsidian-LiveSync service template (in-flight, not yet merged)

- [Core PR · coollabsio/coolify#9822](https://github.com/coollabsio/coolify/pull/9822). adds `templates/compose/obsidian-livesync.yaml` and SVG logo. Author `StaiMerr`. Currently open, blocked on Copilot AI review comments.
- [Canonical template YAML (PR head sha `e2dbf78`)](https://github.com/coollabsio/coolify/blob/e2dbf78e496b6fd6ca36a84381df6c5351b46fa6/templates/compose/obsidian-livesync.yaml). what this repo's `docker-compose.yaml` mirrors verbatim.
- [Docs PR · coollabsio/coolify-docs#616](https://github.com/coollabsio/coolify-docs/pull/616). adds `content/docs/services/obsidian-livesync.mdx`. Replaces closed PRs [#610](https://github.com/coollabsio/coolify-docs/pull/610) and [#614](https://github.com/coollabsio/coolify-docs/pull/614).
- [Service page MDX (raw)](https://raw.githubusercontent.com/coollabsio/coolify-docs/f302374a63f1a6d20265b87dffc74cadace3e549/content/docs/services/obsidian-livesync.mdx). once merged, will publish at `coolify.io/docs/services/obsidian-livesync`.

### Coolify general reference

- [Service template contribution guide](https://coolify.io/docs/get-started/contribute/service). header-comment spec (`documentation`, `slogan`, `category`, `tags`, `logo`, `port`), env-var patterns (`${VAR:?}` required vs `${VAR:-default}` optional), file/logo naming rules, 1000-star repo prerequisite.
- [Environment Variables docs](https://coolify.io/docs/knowledge-base/environment-variables). `SERVICE_FQDN_*`, `SERVICE_URL_*`, `SERVICE_USER_*`, `SERVICE_PASSWORD_*`, `SERVICE_PASSWORD_64_*`, `SERVICE_BASE64_*` magic-var syntax.
- [Docker Compose build pack](https://coolify.io/docs/knowledge-base/docker/compose). how Coolify reads `docker-compose.yaml`, the `volumes: content:` inline-file extension, Caddy/Traefik routing model.

### Obsidian LiveSync upstream

- [`docs/quick_setup.md`](https://github.com/vrtmrz/obsidian-livesync/blob/main/docs/quick_setup.md). the three setup methods (URI, minimal, manual), wizard flow.
- [`docs/setup_own_server.md`](https://github.com/vrtmrz/obsidian-livesync/blob/main/docs/setup_own_server.md). official CouchDB setup reference linked from the Coolify service template's `# documentation:` header.
- [`utils/flyio/generate_setupuri.ts`](https://github.com/vrtmrz/obsidian-livesync/blob/main/utils/flyio/generate_setupuri.ts). Deno helper for generating an `obsidian://setuplivesync?settings=…` URI from the CLI.
- [Plugin repo](https://github.com/vrtmrz/obsidian-livesync). issues, releases, plugin source.

### CouchDB

- [CouchDB 3.4 documentation](https://docs.couchdb.org/en/3.4.0/). reference for every `local.ini` section and key applied here.
