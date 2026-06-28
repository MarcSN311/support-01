# Task — deploy ntfy on support-01 (public, locked down, full Android)
 
Follow `CLAUDE.md` conventions. Goal: ntfy that **just works on Android over normal
internet** (no WireGuard/backbone dependency) and whose **web UI works** for
config/operation. Lock it down hard. Publishing (ingress) is effectively local —
only support-01 services hold a write token — but the server itself is public so
the phone can subscribe and receive push anywhere.
 
## Exposure
- Public via **Traefik + Let's Encrypt** at `ntfy.mar.cloud` (mirror docker-02's
  Traefik pattern). On the `proxy` network, no raw published ports. Add a DNS
  A/AAAA record for `ntfy.mar.cloud` before expecting a cert.
- `server.yml`: `base-url: "https://ntfy.mar.cloud"` (must match exactly — the
  app and push wakeup depend on it).
## Auth (mandatory — ntfy is open by default)
- `auth-default-access: "deny-all"`.
- Create with `docker exec ntfy ntfy user/access/token`:
  - an **admin** user (for the web UI / management),
  - a **read**-scoped user + token on topic `alerts` → for the phone,
  - a **write**-scoped user + token on `alerts` → for support-01 publishers.
- **Do NOT** put Traefik forward-auth / OIDC in front of ntfy — it would break the
  app's token auth and push. ntfy uses its own accounts/tokens; that's correct here.
- Set ntfy visitor rate limits in `server.yml` (it's public now).
## Android push on GrapheneOS (NO Google Play Services — no FCM/Firebase)
- The phone is **GrapheneOS without Play Services**, so Firebase/FCM is not
  available. Do **NOT** set `upstream-base-url` — it would do nothing here.
- Use the **F-Droid build** of the ntfy app (it is FCM-free and uses Instant
  Delivery). Add server `https://ntfy.mar.cloud`, log in with the **read** token,
  subscribe to `alerts`, and ensure **Instant Delivery** is on for that
  subscription. The app holds a persistent outbound connection to the public
  server, so push arrives over mobile data/Wi‑Fi with no Google and no VPN.
- Trade-off (expected on GrapheneOS): Instant Delivery runs a foreground service
  with a persistent notification and uses more battery than FCM would. There is no
  FCM alternative without Play Services — this is the correct path. Note it in the
  stack README.
- Because delivery relies on the phone's persistent connection to the **public**
  server, keep ntfy public over TLS (do not move it WG-only).
## Ingress = local
- Uptime Kuma is **containerised**, so it reaches ntfy over a **shared docker
  network**: put ntfy and Uptime Kuma both on a `monitoring` network
  (`docker network create monitoring`, declared `external: true` in both stacks)
  and publish to `http://ntfy:80`. `http://127.0.0.1` will NOT work across
  container netns. The public POST endpoint stays write-token-gated; no other host
  gets a write token yet.
## Wire Uptime Kuma
- Settings → Notifications → ntfy: server URL `http://ntfy:80` (shared docker
  network), topic `alerts`, the **write** token. Assign to monitors, send a test →
  confirm it lands on the Android app.
## Done when
- `https://ntfy.mar.cloud` serves over TLS via Traefik; no raw ports.
- `auth-default-access: deny-all`; separate read (phone) and write (publishers) tokens.
- Android app receives a Uptime Kuma test alert over normal internet (no WG).
- Web UI reachable and usable with the admin account.
- `/opt/stacks/ntfy/{compose.yaml,config/server.yml,README,.env.example}` committed;
  `.env`, auth db, cache excluded from git.

