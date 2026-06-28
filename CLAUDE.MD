# CLAUDE.md — support-01

Context + working agreement for Claude Code operating on **support-01**.
Read this fully before making changes. When in doubt, match how **docker-02**
is already built (see `docker-02-setup.md` if present in the repo).

---

## What these hosts are

- **docker-02** — the primary public app host (Hetzner, internet-facing, Ubuntu
  26.04). Already built: Traefik + CrowdSec + socket-proxy + Kanidm SSO + immich
  + audiobookshelf + borgmatic backups. **Kanidm lives here** at `idm.mar.cloud`
  and is the OIDC identity provider for the whole estate.
- **support-01** — *this host*. Public (Hetzner), Ubuntu **24.04**. Intended to
  run support/infra services that should NOT sit on the host they observe:
  monitoring and self-hosted git. Currently under-built and under-secured.
- **docker-01** — legacy, **being decommissioned**. Do not build new things
  here. Anything still pointing at it (e.g. a Portainer server) is going away.

Operator: single admin (Marc). Style: compose-CLI + git, no GUI orchestrators.

---

## Conventions (mirror docker-02 exactly)

1. **One stack per directory** under `/opt/stacks/<stack>/`, each with
   `compose.yaml` + `.env`. No compose definitions hidden in a tool's database
   (we just dropped Portainer specifically to escape that).
2. **Compose files are the source of truth**, versioned in git. If a running
   container has no compose on disk (Portainer-era), reconstruct it (see
   `docker-autocompose`) into `/opt/stacks/<stack>/` before touching it.
3. **Data split by access pattern** (the core rule):
   - Transactional / lock-sensitive / watched state (Postgres, SQLite, Redis,
     lock files, consume dirs) → **local block storage**. Never on a network FS.
   - Large read-mostly blobs → CIFS (Storage Box) when one applies.
   - Re-downloadable caches → local, and excluded from backups.
   - **CIFS is mounted as a Docker named volume with `cifs` `driver_opts`** —
     this is the standard here. Docker performs the mount at container start and
     fails the container if it fails (fails closed). Do **not** use host
     `systemd` mounts / `RequiresMountsFor`.
4. **Networks**: `proxy` (external, Traefik ↔ service front-ends) and
   `socket_proxy` (external, `--internal`, Docker-API isolation). Databases/caches
   stay on a stack-local internal network, never on `proxy`.
5. **Secrets** live only in each stack's `.env` (chmod `600`) and off-host escrow.
   Never inline in committed files. `.env.example` (values stripped) is committed.
6. **TLS**: Traefik v3.7 + Let's Encrypt HTTP-01. Every service gets a
   `<name>.mar.cloud` host + a DNS A/AAAA record before it'll get a cert.
7. **SSO**: prefer native OIDC pointed at Kanidm on docker-02
   (`https://idm.mar.cloud`). For apps without native OIDC, use a Traefik
   forward-auth middleware. Match existing users by **email**; keep
   **Auto-Register OFF** and local login enabled until an OIDC login is verified.
8. **Backups**: borgmatic, encrypted (repokey + passphrase), to a Hetzner Storage
   Box over SSH port 23, with the PostgreSQL hook for any DB (never back up raw
   `pgdata`). Export the Borg key off-host. Restore-test before trusting it.

---

## Current state of support-01 (as found)

```
CONTAINER       IMAGE                    NOTES
beszel          henrygd/beszel:0.18.7    monitoring hub, raw 0.0.0.0:8090, left in place pending replacement
beszel-agent    henrygd/beszel-agent     local agent (also updated to 0.18.7)
```

- portainer_agent removed (step 0 done).
- No Hetzner Cloud Firewall attached. Worth adding as general hygiene, but it has
  been public for months — this is **not** treated as an emergency.
- Beszel data dir (Portainer-managed path): `/data/compose/51/beszel_data/`.
  Contains `data.db` (PocketBase main store), `auxiliary.db`, `backups/`,
  `storage/`, and `id_ed25519[.pub]` (**agent auth keypair — do not delete or
  agents must be re-keyed**).
- Beszel may be kept or ditched — decision pending evaluation. Leave it running
  as-is. Do not front it with Traefik or wire SSO until decided.

---

## Plan for support-01 (ordered)

Do these in order. Each step should end with configs on disk under `/opt/stacks/`
and committed to git.

### -1. Backups (borgmatic) — DONE
- borgmatic 2.1.6, daily at 01:00, repo at `support-01.borg` on `u312159-sub10`.
- Borg key exported — store in password manager with passphrase.
- Old `/data/compose/51/` can now be removed (beszel migrated + backed up).

### 0. Cleanup — DONE
- Portainer agent removed.
- Beszel + beszel-agent updated to 0.18.7 in place for evaluation.

### 1. Edge: Traefik + CrowdSec + socket-proxy — DONE
- socket-proxy v0.4.2, CrowdSec v1.7.8, Traefik v3.7.5, bouncer plugin v1.6.0.
- Dashboard at `traefik.support-01.cloud.mar.cloud` (basic-auth, traefikadmin).
- Kanidm CA cert still needed at `traefik/dynamic/kanidm-ca.crt` (SSO step).

### 2. Uptime Kuma — DONE
- Running at `uptime.mar.cloud`, image pinned to `2.4.0`, SQLite backend.
- No native OIDC — SSO via forward-auth wired in step 3.
- Beszel fate (keep alongside or ditch) decided after evaluating both.

### 3. SSO
- Prefer native OIDC pointed at Kanidm on docker-02 (`https://idm.mar.cloud`):
  create the client on docker-02 with `kanidm system oauth2 create`, add redirect
  URLs for the `<app>.mar.cloud` host, match existing users by **email**, keep
  Auto-Register OFF, keep local login until verified.
- For apps lacking native OIDC, build/reuse a Traefik forward-auth middleware
  against Kanidm.

### 4. Self-host git (Forgejo)
- Deploy **Forgejo** at `git.mar.cloud` under `/opt/stacks/forgejo/`. DB local
  (SQLite fine for single-user, or Postgres local), repos on local disk. Behind
  Traefik + LE, OIDC via Kanidm.
- Becomes the off-host git remote for the `/opt/stacks` config repos; repoint
  origins here once live. Optional later: CI runners for the app pipelines.

### 5. OS upgrade — DEFERRED
- **Do NOT** `do-release-upgrade` to 26.04 yet. The LTS→LTS path only opens at
  **26.04.1 (~Aug 6 2026)**; before that it requires the `-d` dev flag, which is
  not safe on a public production host. 24.04 is supported to 2029 — no pressure.
- When the time comes: snapshot first, read release notes for the stack (26.04
  ships kernel 7.0, sudo-rs replacing sudo, Wayland-only, PQC defaults), schedule
  a window. docker-02 is on 26.04 because it was a **fresh install**, not an
  in-place upgrade.

---

## Hard rules / footguns (learned on docker-02)

- **Verify, don't assume, current versions** of images/plugins before pinning.
  Several builds broke on stale version guesses (Traefik tag, CrowdSec bouncer
  plugin version, borgmatic `init`→`repo-create`).
- **`$` escaping**: in compose `.env`, every literal `$` must be doubled (`$$`);
  in Traefik's **file** provider, use a single `$`.
- **Cross-provider middleware** references need an explicit `@file` / `@docker`
  suffix or Traefik reports "middleware does not exist".
- **Plugins** (e.g. CrowdSec bouncer) must be declared in Traefik **static**
  config before any middleware references them.
- **Phantom mount / phantom account**: containers starting before a network mount
  exists write to an empty local dir — always mount CIFS as a Docker `cifs` named
  volume (fails closed), never a plain bind to a host mountpoint. OIDC with
  Auto-Register ON + a non-matching email silently creates an empty new account.
- **Never commit secrets/state.** Verify the staged list before the first commit.
  Highest-stakes files to keep out: any `*.key`/`cakey.pem`, `*.htpasswd`,
  `acme.json`, `id_ed25519`, database files (`*.db`, `pgdata/`), `.env`.
- **`container_name`** on every service for predictable `docker exec`/log targets.
- **Set a stable `hostname`** on borgmatic (it drives Borg archive names).

---

## Definition of done (per stack)

1. `compose.yaml` + `.env` under `/opt/stacks/<stack>/`, no raw published admin
   ports, behind Traefik with TLS.
2. Secrets only in `.env` (600); `.env.example` committed.
3. SSO wired (native OIDC or forward-auth) where the app supports it.
4. Backed up by borgmatic (DB via hook, raw data dir excluded) if it has state.
5. Committed to git and pushed to the off-host remote.
