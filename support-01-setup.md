# support-01 — Docker Host Build & Operations

Hetzner-hosted Docker server running support and infra services for the mar.cloud
estate. Hosts monitoring (Uptime Kuma), push notifications (ntfy), and the
self-hosted git server (Forgejo). Acts as the off-host remote for config repos
that live on docker-02.

Edge layer mirrors docker-02 exactly: Traefik v3 + CrowdSec + Docker socket proxy.
SSO via Kanidm on docker-02 (`idm.mar.cloud`).

> **Secrets:** every password/key in this document is a placeholder.
> Real values live only in each stack's `.env` (chmod `600`) and off-host escrow.
> Nothing sensitive is committed here.

---

## 1. Host conventions

| Item | Value |
|------|-------|
| Hostname | `support-01` |
| Provider | Hetzner, public internet-facing |
| OS | Ubuntu 24.04 (supported to 2029 — do not upgrade to 26.04 until 26.04.1, ~Aug 2026) |
| Docker | Docker CE from the official repo (`get.docker.com`), Compose v2 plugin |
| Compose stacks | `/opt/stacks/<stack>/` — one dir per stack, each with `compose.yaml` + `.env` |
| Local state | on local block storage, under each stack dir |
| Hardware | 2 vCPU, 3.7 GB RAM, 38 GB disk, no swap |

### Directory layout

```
/opt/stacks/
├── socket-proxy/      # Docker socket isolation (shared dependency)
├── traefik/           # reverse proxy + TLS + middlewares
├── crowdsec/          # IPS, reads traefik access logs
├── uptime-kuma/       # uptime monitoring
├── ntfy/              # push notifications (GrapheneOS / F-Droid)
├── forgejo/           # self-hosted git
├── beszel/            # server resource monitoring hub + agent
└── borgmatic/         # encrypted backups to Hetzner Storage Box
```

---

## 2. Networks and volumes

Created once, referenced as `external` by all stacks.

```bash
docker network create proxy
docker network create --internal socket_proxy
docker network create monitoring
docker network create --internal beszel
docker volume create traefik-logs
```

| Network | Scope | Members | Purpose |
|---------|-------|---------|---------|
| `proxy` | external | traefik, crowdsec, all app front-ends | Traefik ↔ services |
| `socket_proxy` | external, **internal (no egress)** | dockerproxy, traefik, uptime-kuma | isolate Docker API access |
| `monitoring` | external | uptime-kuma, ntfy, borgmatic | internal container-to-container notification path |
| `beszel` | external, **internal** | beszel hub, beszel-agent | hub ↔ agent isolation |

| Volume | Purpose |
|--------|---------|
| `traefik-logs` | shared access log between traefik (writer) and crowdsec (reader) |

---

## 3. Stack: socket-proxy

Tecnativa Docker socket proxy — exposes a read-only subset of the Docker API
over TCP so Traefik and Uptime Kuma never touch `/var/run/docker.sock` directly.

**Image:** `tecnativa/docker-socket-proxy:v0.4.2`

Allowed API endpoints: `CONTAINERS`, `NETWORKS`, `SERVICES`, `TASKS`, `EVENTS`,
`PING`, `VERSION`. `POST=0` (read-only).

```bash
docker compose -f /opt/stacks/socket-proxy/compose.yaml up -d
```

---

## 4. Stack: traefik + crowdsec

Edge layer, mirroring docker-02.

**Images:** `traefik:v3.7.5`, `crowdsecurity/crowdsec:v1.7.8`  
**Plugin:** `crowdsec-bouncer-traefik-plugin v1.6.0`

### Setup (first time)

```bash
# 1. Copy and populate .env
cp traefik/.env.example traefik/.env && chmod 600 traefik/.env
# Set ACME_EMAIL=<your email>

# 2. Generate dashboard htpasswd (bcrypt)
docker run --rm -it httpd:alpine htpasswd -nB traefikadmin \
  | grep '^traefikadmin:' > traefik/dynamic/users.htpasswd
chmod 600 traefik/dynamic/users.htpasswd

# 3. Copy middlewares example and fill bouncer key (after crowdsec is up)
cp traefik/dynamic/middlewares.yml.example traefik/dynamic/middlewares.yml

# 4. Start crowdsec first
docker compose -f /opt/stacks/crowdsec/compose.yaml up -d

# 5. Generate bouncer key and put it in middlewares.yml
docker exec crowdsec cscli bouncers add traefik-bouncer
# edit traefik/dynamic/middlewares.yml → crowdsecLapiKey

# 6. Start traefik
docker compose -f /opt/stacks/traefik/compose.yaml up -d
```

### Dynamic config files (not committed)

| File | Purpose |
|------|---------|
| `traefik/dynamic/middlewares.yml` | secure-headers, rate-limit, crowdsec bouncer (contains bouncer key) |
| `traefik/dynamic/users.htpasswd` | basic-auth for dashboard |
| `traefik/dynamic/kanidm-ca.crt` | Kanidm CA cert for forward-auth (SSO step) |
| `traefik/acme/acme.json` | Let's Encrypt certificates (created by Traefik) |

### DNS convention

All services use `<service>.mar.cloud` or `<service>.support-01.cloud.mar.cloud`.
Traefik dashboard: `traefik.support-01.cloud.mar.cloud`.
DNS entries are CNAMEs pointing at `support-01.cloud.mar.cloud`.

### Global middlewares (applied to all websecure traffic)

- `secure-headers@file` — HSTS, XSS protection, referrer policy, frame options
- `crowdsec@file` — CrowdSec bouncer in stream mode
- `rate-limit@file` — 100 req/s average, burst 200

---

## 5. Stack: uptime-kuma

Uptime monitoring at `uptime.mar.cloud`.

**Image:** `louislam/uptime-kuma:2.4.0`  
**DB:** SQLite (chosen at first-run setup wizard)  
**Networks:** `proxy` (Traefik), `socket_proxy` (Docker monitor), `monitoring` (ntfy notifications)  
**Docker host:** `DOCKER_HOST=tcp://dockerproxy:2375` (read-only socket proxy)

```bash
docker compose -f /opt/stacks/uptime-kuma/compose.yaml up -d
```

### Notifications

Uptime Kuma sends alerts to ntfy over the shared `monitoring` network:
- Settings → Notifications → ntfy
- Server URL: `http://ntfy:80` (internal, not the public URL)
- Topic: `alerts`
- Token: write token from ntfy (see §6)

---

## 6. Stack: ntfy

Push notification server at `ntfy.mar.cloud`. Designed for GrapheneOS without
Google Play Services — uses the F-Droid ntfy app with Instant Delivery
(persistent connection, no FCM).

**Image:** `binwiederhier/ntfy:v2.25.0`  
**Networks:** `proxy` (Traefik), `monitoring` (internal publishers)

### Key config (`config/server.yml`)

- `base-url: https://ntfy.mar.cloud` — must match exactly for the Android app
- `auth-default-access: deny-all` — no anonymous access
- No `upstream-base-url` — FCM is not available on GrapheneOS

### First-time user setup

```bash
docker exec -it ntfy ntfy user add --role=admin admin
docker exec -it ntfy ntfy user add --role=user ntfy-reader
docker exec -it ntfy ntfy user add --role=user ntfy-writer
docker exec ntfy ntfy access ntfy-reader alerts read
docker exec ntfy ntfy access ntfy-writer alerts write
docker exec ntfy ntfy token add ntfy-reader   # → phone
docker exec ntfy ntfy token add ntfy-writer   # → Uptime Kuma, borgmatic
```

The writer token (`NTFY_TOKEN`) is shared by all internal publishers (Uptime Kuma, borgmatic).
Do **not** put Traefik forward-auth/OIDC in front of ntfy — it breaks the app's token auth.

### Android setup (GrapheneOS, F-Droid ntfy app)

1. Install ntfy from F-Droid (FCM-free build)
2. Add server `https://ntfy.mar.cloud`, log in with the **reader token**
3. Subscribe to topic `alerts`
4. Enable **Instant Delivery** for that subscription

Trade-off: Instant Delivery runs a persistent foreground service with a
notification and uses more battery than FCM. This is unavoidable without
Google Play Services.

---

## 7. Stack: forgejo

Self-hosted git at `git.mar.cloud`. Single-admin, SQLite, SSH on port 222.

**Image:** `codeberg.org/forgejo/forgejo:15-rootless` (runs as uid 1000)  
**DB:** SQLite at `./data/data/forgejo.db`  
**SSH:** port 222 on the host → 2222 in container. Clone URL: `ssh://git@git.mar.cloud:222/<user>/<repo>.git`

### First-time setup

```bash
mkdir -p /opt/stacks/forgejo/data
chown -R 1000:1000 /opt/stacks/forgejo/data
cp forgejo/.env.example forgejo/.env && chmod 600 forgejo/.env
# Set SECRET_KEY=$(openssl rand -hex 32)

docker compose -f /opt/stacks/forgejo/compose.yaml up -d

# Create admin user
docker exec forgejo forgejo admin user create \
  --username marc --email git@marcsn.de --admin --random-password
```

### Kanidm OIDC (SSO)

Auth source configured via CLI (kanidm side run on docker-02):

```bash
# On docker-02:
kanidm system oauth2 create forgejo "Forgejo" https://git.mar.cloud
kanidm system oauth2 add-redirect-url forgejo https://git.mar.cloud/user/oauth2/kanidm/callback
kanidm group create forgejo_users
kanidm group add-members forgejo_users marc
kanidm system oauth2 update-scope-map forgejo forgejo_users openid email profile
kanidm system oauth2 prefer-short-username forgejo   # avoids @ in username
kanidm system oauth2 show-basic-secret forgejo       # → CLIENT_SECRET

# On support-01:
docker exec forgejo forgejo admin auth add-oauth \
  --name kanidm \
  --provider openidConnect \
  --key forgejo \
  --secret <CLIENT_SECRET> \
  --auto-discover-url "https://idm.mar.cloud/oauth2/openid/forgejo/.well-known/openid-configuration" \
  --scopes "openid email profile"
```

Key settings in compose:
- `ALLOW_ONLY_EXTERNAL_REGISTRATION=true` — new accounts via Kanidm only
- `DISABLE_REGISTRATION=false` — required for external registration to work
- `ENABLE_OPENID_SIGNIN=false` / `ENABLE_OPENID_SIGNUP=false` — hides the generic OIDC button

The client secret lives in `kanidm.db` on docker-02 and is recoverable at any
time with `kanidm system oauth2 show-basic-secret forgejo`.

---

## 8. Stack: beszel

Server resource monitoring at `beszel.mar.cloud`. Hub + local agent pair.

**Images:** `henrygd/beszel:latest`, `henrygd/beszel-agent:latest` (pinned to 0.18.7 on upgrade)  
**DB:** PocketBase SQLite (`./data/data.db`)  
**Networks:** hub on `proxy` + `beszel`; agent on `beszel` only (no public access)

```bash
cp beszel/.env.example beszel/.env && chmod 600 beszel/.env
# Set BESZEL_AGENT_KEY from the hub UI (Settings → Add system → copy ed25519 public key)
docker compose -f /opt/stacks/beszel/compose.yaml up -d
```

In the hub UI, add the agent with host `beszel-agent` and port `45876`.  
The agent auth key (`BESZEL_AGENT_KEY`) is the hub's public key, visible in the hub
Settings panel. Without it the agent rejects connections.

---

## 9. Stack: borgmatic

Encrypted daily backups to Hetzner Storage Box over SSH.

**Image:** `ghcr.io/borgmatic-collective/borgmatic:2.1.6`  
**Repo:** `ssh://u312159-sub10@u312159-sub10.your-storagebox.de:23/./support-01.borg`  
**Encryption:** repokey (key stored in repo) + passphrase  
**Schedule:** daily at 01:00 Europe/Berlin (cron inside container)  
**Networks:** `monitoring` (to reach ntfy for notifications)

### First-time setup

```bash
cp borgmatic/.env.example borgmatic/.env && chmod 600 borgmatic/.env
# Set BORG_PASSPHRASE and NTFY_TOKEN

# Install SSH key on storage box
ssh-copy-id -p 23 -i borgmatic/ssh/id_ed25519.pub u312159-sub10@u312159-sub10.your-storagebox.de

docker compose -f /opt/stacks/borgmatic/compose.yaml up -d

# Initialise the repo (first time only)
docker exec borgmatic borgmatic repo-create --encryption repokey

# Export the key — store in password manager alongside the passphrase
docker exec borgmatic borg key export \
  ssh://u312159-sub10@u312159-sub10.your-storagebox.de:23/./support-01.borg

# Run first backup and verify
docker exec borgmatic borgmatic create --stats -v 1
docker exec borgmatic borgmatic list
```

### What is backed up

`/opt/stacks` (read-only bind mount at `/source/stacks`), excluding:

- `*/borg-cache`, `*/borg-config` — Borg working dirs
- `*/crowdsec/data` — runtime data, recreatable from hub
- `*/beszel/data/backups` — PocketBase auto-backups (redundant)
- `*/ntfy/data/cache` — ephemeral message cache

SQLite databases (Forgejo, Uptime Kuma, ntfy auth, beszel) are backed up as
live files — safe because they use WAL mode. There are no PostgreSQL databases
on this host.

### Notifications

borgmatic uses its native ntfy integration to send to the `alerts` topic:
- **start** — `⏳ Backup started` (min priority, for diagnostics)
- **finish** — `✅ Backup complete` (default priority)
- **fail** — `🚨 Backup FAILED` (high priority)

Requires `NTFY_TOKEN` (writer token) in `.env` and borgmatic on the `monitoring` network.

### Runbook: manual backup / restore

```bash
# Force a run now
docker exec borgmatic borgmatic create --stats -v 1

# List archives
docker exec borgmatic borgmatic list

# List files in an archive
docker exec borgmatic borgmatic list --archive support-01-2026-06-28T21:08:58.635857

# Restore a single file
docker exec borgmatic borgmatic extract \
  --archive support-01-2026-06-28T21:08:58.635857 \
  --path source/stacks/forgejo/borgmatic.d/config.yaml

# Check repo integrity
docker exec borgmatic borgmatic check
```

---

## 10. Footguns

- **`prefer-short-username`** must be set on kanidm OAuth2 clients or Forgejo
  rejects the username (Kanidm returns `user@domain` by default; `@` is invalid
  in Forgejo usernames).
- **Rootless container ownership**: Forgejo's data dir must be owned by uid 1000
  before first start (`chown -R 1000:1000 ./data`), or the container silently
  fails to write.
- **INSTALL_LOCK**: Forgejo's web installer conflicts with the env-to-ini
  entrypoint — the installer saves `app.ini`, then the entrypoint overwrites it
  on restart. Skip the installer entirely: set `INSTALL_LOCK=true` and create
  the admin user via CLI.
- **CrowdSec bouncer key** is in `traefik/dynamic/middlewares.yml` (not
  committed). Recoverable with `docker exec crowdsec cscli bouncers list`.
- **`$$` in compose labels**: literal `$` in Traefik label values must be
  doubled to `$$` — compose interpolates single `$` as a variable.
- **Cross-provider middleware references** need `@file` / `@docker` suffix or
  Traefik reports "middleware does not exist".
- **Plugins** must be declared in Traefik static config (command args) before
  any middleware references them.
