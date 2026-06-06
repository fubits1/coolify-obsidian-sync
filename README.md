# Obsidian Self-hosted LiveSync — Coolify Deployment

CouchDB backend for the [Self-hosted LiveSync](https://github.com/vrtmrz/obsidian-livesync) Obsidian plugin, packaged for one-click deployment via [Coolify](https://coolify.io).

> **Note**: This is **not** the proprietary Obsidian Sync service. Obsidian Sync (the paid product from Obsidian.md) has no self-hostable engine. `Self-hosted LiveSync` is the community plugin that replicates a vault to a CouchDB you control.

## What gets deployed

| Service | Image | Purpose |
|---|---|---|
| `couchdb` | `couchdb:3.4` | Document store + replication engine |
| `couchdb-init` | `curlimages/curl:8.10.1` | One-shot: single-node cluster setup, CORS, size limits, vault DB creation |

Persistent volumes: `couchdb-data`, `couchdb-config`.

## Coolify magic env vars

Coolify (v4.0.0-beta.411+) supports auto-generated env vars in Docker Compose stacks. This compose uses three:

| Var | Type | What Coolify does |
|---|---|---|
| `SERVICE_FQDN_COUCHDB_5984` | FQDN+port | Allocates a subdomain on the configured wildcard, injects Traefik labels routing `https://<fqdn>/` → container `:5984`, provisions Let's Encrypt cert |
| `SERVICE_USER_COUCHDB` | User | 16-char random string, persisted across redeploys |
| `SERVICE_PASSWORD_COUCHDB` | Password | Symbol-free random password, persisted across redeploys |

The naming convention is `SERVICE_<TYPE>_<IDENTIFIER>[_PORT]`. The `<IDENTIFIER>` (`COUCHDB`) is reused — every magic var with the same identifier resolves to the same generated value across services in the stack.

### How the compose references them

```yaml
environment:
  - SERVICE_FQDN_COUCHDB_5984              # marker only; value injected by Coolify
  - COUCHDB_USER=${SERVICE_USER_COUCHDB}
  - COUCHDB_PASSWORD=${SERVICE_PASSWORD_COUCHDB}
```

A bare `SERVICE_FQDN_X_PORT` entry (no `=value`) is the Coolify signal to wire Traefik to that port. The `expose: ["5984"]` in the service is what Coolify routes _to_.

### Things Coolify does NOT need

- No `ports:` host-binding (Traefik fronts everything).
- No reverse-proxy config (Caddy/Traefik/nginx) — Coolify generates it.
- No TLS termination (handled at the Traefik edge).

## Deploy on Coolify

1. **New Resource → Docker Compose Empty** (or **Public Repository** if you host this in git).
2. Paste/point at `docker-compose.yml`.
3. Coolify auto-detects the three `SERVICE_*` vars and shows them on the **Environment Variables** tab. Leave them blank; Coolify generates them on first deploy.
4. Optional: set `COUCHDB_DB` (defaults to `obsidian`).
5. On the **Domains** tab, confirm the auto-allocated subdomain or replace with your own.
6. **Deploy**. The `couchdb-init` container exits 0 once setup is applied; CouchDB stays running.
7. Copy the FQDN + generated user/password from the Coolify env tab into the Obsidian plugin (see [Client setup](#client-setup-obsidian-plugin)).

### Persistence note

Generated `SERVICE_USER_COUCHDB` / `SERVICE_PASSWORD_COUCHDB` are stored by Coolify and reused on redeploy. The CouchDB volumes survive redeploys. **Do not regenerate the password** in Coolify after first init — CouchDB has already hashed it into the admin DB; you would have to wipe the volume.

## Local validation with OrbStack

The `docker-compose.local.yml` override adds a host port mapping so you can poke CouchDB directly.

```bash
cp .env.local.example .env.local
docker compose -f docker-compose.yml -f docker-compose.local.yml --env-file .env.local up -d
```

Wait for `couchdb-init` to exit (`docker compose ps`), then validate:

```bash
# 1. CouchDB is up
curl -sf http://localhost:5984/_up                                # {"status":"ok"}

# 2. Single-node cluster finished
curl -su admin:changeme-local-only http://localhost:5984/_cluster_setup
                                                                  # {"state":"cluster_finished"}

# 3. CORS configured for Obsidian
curl -su admin:changeme-local-only \
  http://localhost:5984/_node/_local/_config/cors                 # origins includes app://obsidian.md

# 4. Vault DB exists
curl -su admin:changeme-local-only http://localhost:5984/obsidian # {"db_name":"obsidian",...}

# 5. Init container exited cleanly
docker compose logs couchdb-init | tail -5                        # "==> done"
```

Tear down:

```bash
docker compose -f docker-compose.yml -f docker-compose.local.yml down -v   # -v wipes volumes
```

## Client setup (Obsidian plugin)

1. Obsidian → **Settings → Community plugins → Browse → "Self-hosted LiveSync" → Install + Enable**.
2. Open the plugin settings → **Remote database configuration**:
   - **URI**: `https://<your-fqdn>` (Coolify) or `http://localhost:5984` (local)
   - **Username**: value of `SERVICE_USER_COUCHDB`
   - **Password**: value of `SERVICE_PASSWORD_COUCHDB`
   - **Database name**: `obsidian` (or whatever you set `COUCHDB_DB` to)
3. **Test database connection** → should report all green checks.
4. Run the **wizard** for first-vault setup. Pick the device that has the canonical vault as the source.

## CouchDB config applied by `couchdb-init`

Mirrors the upstream [`couchdb-init.sh`](https://github.com/vrtmrz/obsidian-livesync/blob/main/utils/couchdb/couchdb-init.sh) but inlined to avoid a runtime fetch:

| Section | Key | Value |
|---|---|---|
| `chttpd` | `require_valid_user` | `true` |
| `chttpd_auth` | `require_valid_user` | `true` |
| `httpd` | `WWW-Authenticate` | `Basic realm="couchdb"` |
| `httpd` / `chttpd` | `enable_cors` | `true` |
| `chttpd` | `max_http_request_size` | `4294967296` (4 GB) |
| `couchdb` | `max_document_size` | `50000000` (50 MB) |
| `cors` | `credentials` | `true` |
| `cors` | `origins` | `app://obsidian.md,capacitor://localhost,http://localhost` |
| `cors` | `methods` | `GET,PUT,POST,HEAD,DELETE` |
| `cors` | `headers` | `accept,authorization,content-type,origin,referer,x-csrf-token` |

## Troubleshooting

- **`couchdb-init` exits 22 / 401**: the `SERVICE_USER_COUCHDB` / `SERVICE_PASSWORD_COUCHDB` Coolify regenerated after a volume already had a different admin baked in. Either restore the old values in Coolify or wipe the `couchdb-data` volume.
- **Obsidian client reports CORS error**: confirm `cors/origins` includes `app://obsidian.md` (desktop) and `capacitor://localhost` (mobile). Re-run the init container: `docker compose run --rm couchdb-init`.
- **Mobile sync fails over LTE only**: TLS cert chain issue. Confirm Coolify's Traefik served a full chain at the FQDN (`curl -vI https://<fqdn>`).

## Sources

- [Coolify — Environment Variables docs](https://coolify.io/docs/knowledge-base/environment-variables)
- [Coolify — Docker Compose build pack](https://coolify.io/docs/knowledge-base/docker/compose)
- [obsidian-livesync — setup_own_server.md](https://github.com/vrtmrz/obsidian-livesync/blob/main/docs/setup_own_server.md)
- [obsidian-livesync — couchdb-init.sh](https://github.com/vrtmrz/obsidian-livesync/blob/main/utils/couchdb/couchdb-init.sh)
