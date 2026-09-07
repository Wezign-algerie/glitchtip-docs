# GlitchTip — Full Reference

Self-hosted, Sentry-API-compatible error tracking, performance monitoring,
uptime monitoring, and log ingestion. Deployed on the Wezign shared VPS,
2026-09-06. SMTP wired 2026-09-07.

This document has two halves:

- **Part 1** — exactly how *this* instance is built, configured, and operated.
- **Part 2** — what GlitchTip as a product can do, summarized from the
  official docs at https://glitchtip.com/documentation, with a link back to
  the source page for every section so you can go deeper.

---

# Part 1 — This instance

| | |
| --- | --- |
| URL | https://glitchtip.wecreativeagency.dz |
| Server | `ssh vipcar-vps-root` |
| App directory | `/opt/glitchtip` |
| Image | `glitchtip/glitchtip:6` (all-in-one mode — web, worker, and scheduler in one container) |
| Database | `postgres:18`, own container |
| Cache / queue | `valkey/valkey:9` (Redis-compatible), own container |
| TLS | Let's Encrypt via certbot, auto-renewing |
| Org owner | `anis.rasoul@wezignalgerie.com` |

## 1.1 Architecture on this box

```
Internet
   │  :443 (TLS)
   ▼
nginx (shared by ~28 other sites on this host)
   │  proxy_pass → 127.0.0.1:8000
   ▼
docker compose stack, /opt/glitchtip
   ┌─────────────────────────────────────────┐
   │  web  (glitchtip/glitchtip:6)            │
   │    SERVER_ROLE=all_in_one                │
   │    binds 127.0.0.1:8000 only             │
   │  postgres:18   — pg-data volume          │
   │  valkey:9      — cache + task queue      │
   └─────────────────────────────────────────┘
```

Three deliberate departures from GlitchTip's own sample `compose.yml`
(https://glitchtip.com/documentation/install), all because this box hosts
~19 other unrelated apps on a single CPU core:

1. **`127.0.0.1:8000:8000`, not `8000:8000`.** The bare form in the sample
   would publish GlitchTip straight to the internet, bypassing nginx and ufw
   entirely, because Docker writes its own iptables rules. Everything must go
   through the reverse proxy.
2. **A generated Postgres password**, not the sample's
   `POSTGRES_HOST_AUTH_METHOD: trust`.
3. **`mem_limit` on every service** (postgres 512m, valkey 192m, web 1g) so a
   leak in GlitchTip can't starve the rest of the box.

## 1.2 Files on the server

| File | Purpose |
| --- | --- |
| `/opt/glitchtip/compose.yml` | Service definitions, non-secret config |
| `/opt/glitchtip/.env` | Secrets: `SECRET_KEY`, `POSTGRES_PASSWORD`, `EMAIL_URL`. Mode 600, root-owned |
| `/etc/nginx/sites-available/glitchtip.wecreativeagency.dz` | Reverse proxy vhost |
| `/etc/letsencrypt/live/glitchtip.wecreativeagency.dz/` | TLS certificate |

Read the secrets:

```bash
ssh vipcar-vps-root 'cat /opt/glitchtip/.env'
```

## 1.3 Viewing logs

```bash
ssh vipcar-vps-root
cd /opt/glitchtip

docker compose logs web --tail 100        # the app — this is almost always the one you want
docker compose logs web -f                # follow live, Ctrl+C to stop
docker compose logs web --since 30m       # time-bounded instead of line-bounded
docker compose logs web --tail 300 | grep -iE "error|traceback|exception|critical"

docker compose logs postgres --tail 100
docker compose logs valkey --tail 100
docker compose logs --tail 100            # all three interleaved
```

One-liner without an interactive shell:

```bash
ssh vipcar-vps-root 'cd /opt/glitchtip && docker compose logs web --tail 100'
```

## 1.4 Health and status

```bash
docker compose ps
docker inspect -f '{{.Name}} restarts={{.RestartCount}} state={{.State.Status}}' $(docker compose ps -q)
docker stats --no-stream
curl -s http://127.0.0.1:8000/_health/      # bypasses nginx entirely
```

## 1.5 Managing the stack

```bash
cd /opt/glitchtip

docker compose stop
docker compose up -d              # start
docker compose up -d web          # recreate just the app after an env/compose change — NOT `restart`, which won't pick up new env vars

docker compose pull && docker compose up -d   # upgrade; migrations run automatically on boot
```

## 1.6 Email / SMTP

Live, verified with a real test send (`./manage.py sendtestemail`). Outbound
mail authenticates as `noreply@wecreativeagency.dz` against
`ssl4.icosnethosting.com:465` (implicit TLS).

Three non-obvious things, all of which will bite again if this is ever
rebuilt:

1. **Host must be `ssl4.icosnethosting.com`, not `mail.wecreativeagency.dz`.**
   Both names resolve to the same machine (`197.140.11.9`), but the TLS
   certificate there is a wildcard for `*.icosnethosting.com` only.
   GlitchTip 6 runs Python 3.14 and verifies certificates, so the `mail.`
   name fails with `CERTIFICATE_VERIFY_FAILED: Hostname mismatch`. The bare
   domain `wecreativeagency.dz` isn't usable at all — it resolves to this VPS,
   not the mail server.
2. **The mailbox password contains `#`, percent-encoded as `%23`** in
   `EMAIL_URL` (it's a URL — an unescaped `#` starts a fragment and silently
   truncates everything after it). `@` in the username is `%40`.
3. **Port 465 is implicit TLS** → `smtp+ssl://` scheme. (587 would be
   STARTTLS → `smtp+tls://`; 587 also works on this mail server if ever
   needed.)

Change it:

```bash
# edit EMAIL_URL in /opt/glitchtip/.env, then:
cd /opt/glitchtip
docker compose up -d web
docker compose exec -T web ./manage.py sendtestemail you@example.com
```

Exit 0, no traceback = the SMTP server accepted the message. Sanity-check the
parsed value without ever printing the password itself:

```bash
docker compose exec -T web ./manage.py shell -c \
  "from django.conf import settings as s; print(s.EMAIL_HOST, s.EMAIL_PORT, len(s.EMAIL_HOST_PASSWORD))"
```

Password length should read **17**. A length of 1 means the `#` truncated it.

## 1.7 Accounts and registration

`ENABLE_USER_REGISTRATION=False` and `ENABLE_ORGANIZATION_CREATION=False` —
set deliberately at deploy time so this instance is not open to the public.
Only these users exist on it:

| id | email | role |
| --- | --- | --- |
| 1 | anis.rasoul@wezignalgerie.com | org owner (first self-signup) |
| 2 | development@wezignalgerie.com | created 2026-09-07 |
| 3 | chakib.kerbadj@wezignalgerie.com | created 2026-09-07 |

**Important, undocumented-upstream behavior we hit in practice:** with
registration disabled, GlitchTip's org-invite flow refuses any email that
doesn't already belong to a registered user — the UI says *"only existing
members may be invited."* There's no self-service way around this while
registration stays locked. The workaround used here: create the user record
directly (Django shell, `get_user_model().objects.create_user(...)`, no
usable password), mark their email verified via
`allauth.account.models.EmailAddress`, then invite them from the org members
page. The new user sets their own password via **Forgot password** on the
login screen — nobody's password passes through a chat or a script.

To add another person under this same closed-registration model, tell me
their email and I'll repeat that process, then you invite them from the UI.

Alternative if you ever want this fully self-service: enable
`ENABLE_USER_REGISTRATION`, which opens `/register` to anyone with the link.
Not recommended while the instance has no other access control in front of
it.

## 1.8 Networking and security

- App container reachable only on `127.0.0.1:8000` — nginx is the only path in.
- TLS: Let's Encrypt, issued 2026-09-06, expires 2026-12-05,
  `certbot.timer` active, renewal dry-run passes.
- **Always `nginx -t` before `systemctl reload nginx`** — this nginx serves
  ~28 other sites; a syntax error takes all of them down.
- Per-container `mem_limit`s protect the rest of the box from a GlitchTip leak.

## 1.9 Troubleshooting

| Symptom | Check |
| --- | --- |
| Site won't load | `docker compose ps`, then `curl http://127.0.0.1:8000/_health/` on the server |
| 502 from nginx | App still booting (20–30s after a restart on this single-core box) or crashed — `docker compose logs web --tail 50` |
| Email not arriving | `docker compose exec -T web ./manage.py sendtestemail you@example.com`. Cert error → hostname mismatch (§1.6.1). Auth error → check `%23` encoding (§1.6.2) |
| "only existing members may be invited" | Registration is locked (§1.7) — the invited email needs an account created first |
| Certificate expiring | `certbot renew --cert-name glitchtip.wecreativeagency.dz --dry-run` |
| Config change had no effect | `docker compose up -d web`, not `restart` — env/compose changes only apply on container recreation |
| Out of disk/memory | `docker system df`, `docker stats --no-stream`, `df -h /` |

---

# Part 2 — What GlitchTip can do

Summarized from https://glitchtip.com/documentation. Every section links to
its source page — read that page directly for anything this summary
compresses too far.

## 2.1 Getting started: org → team → project

Source: https://glitchtip.com/documentation/getting-started

- **Organization** — the top-level container. Represents your company or a
  client; you can have several. (This instance's org is owned by
  `anis.rasoul@wezignalgerie.com`.)
- **Team** — controls *who gets notified*. Any org member can view and modify
  any project, but only members of a project's team receive alert
  notifications for it. Create the team before finishing project setup.
- **Project** — one endpoint for receiving error/event data from one
  app/service. Split projects by app, by environment (staging vs prod), or
  however makes sense for your alerting.

### Creating a project (UI steps)

1. Log in, pick (or create) an organization.
2. **Projects → Create Project.**
3. Choose a platform — GlitchTip shows platform-specific SDK setup
   instructions based on this choice.
4. Assign it to a team, so someone actually gets notified.
5. Land on the project's **Issues** page — this page shows the **DSN**
   (Data Source Name, the per-project URL your app's SDK sends events to)
   plus copy-pasteable SDK install instructions for the platform you picked.
6. Trigger a test error from your app to confirm events arrive.

## 2.2 Error tracking

Source: https://glitchtip.com/documentation/error-tracking

Your app's SDK reports exceptions to the project's DSN; GlitchTip groups them
into **issues**, notifies the project's team, and shows stack traces, tags,
and event history per issue.

- **Alerts** — per-project, configurable by error frequency. Default is email
  to the project's team members; you can also add a **webhook URL** (a
  generic webhook — this is how Slack/Discord/Teams integrations are
  typically wired in practice, by pointing the webhook URL at that service's
  incoming-webhook endpoint, even though the integrations page below doesn't
  name them explicitly).
- **Source maps** — for JavaScript, upload source maps so minified stack
  traces become readable. Done via the GlitchTip CLI (§2.6): inject debug IDs
  into your build output, then upload the maps tied to a release.

## 2.3 Performance monitoring

Source: https://glitchtip.com/documentation/performance

Captures **transactions** (top-level operations, e.g. an HTTP request) and
their child **spans** (e.g. a DB query or downstream API call inside that
request), grouped by endpoint so you can spot slow paths and regressions.

Enable it by setting `tracesSampleRate` (0.0–1.0) in your Sentry SDK's
`init()` call — this is the fraction of transactions captured. Keep it low in
production (the FAQ, §2.8, suggests `0.01` as a starting point) to limit
overhead and event volume.

## 2.4 Uptime monitoring

Source: https://glitchtip.com/documentation/uptime-monitoring

Polls your app on an interval you set and flags it Down when it doesn't
respond as expected. Four monitor types:

| Type | How it works |
| --- | --- |
| Ping | HTTP HEAD request — the common default |
| GET | HTTP GET, checked against an expected status code |
| POST | HTTP POST, checked against an expected status code |
| Heartbeat | Reversed — *your app* pings a URL GlitchTip gives you; silence = Down |

Link a monitor to a project with alerting configured to actually get
notified on a status change.

## 2.5 Logs

Source: https://glitchtip.com/documentation/logs

Ingests structured application logs through the same SDK/endpoint as error
events, and correlates a log entry with its transaction when it carries a
`trace_id`.

- Controlled by `GLITCHTIP_ENABLE_LOGS` (on by default).
- SDK-side: `enable_logs=True` (Python) / `enableLogs: true` (JS), or route
  standard logging output through the SDK automatically. OpenTelemetry users
  should use sentry-sdk's OTel logs integration rather than sending logs
  directly.
- Viewed on a project's **Logs** page: filter by severity (trace → fatal),
  service name, environment, or full-text search.
- Retention: `GLITCHTIP_LOG_RETENTION_DAYS` (default 90) total,
  `GLITCHTIP_LOG_HOT_DAYS` (default 7) in Postgres before optional archival to
  Parquet/S3 via DuckDB cold storage (`GLITCHTIP_ENABLE_DUCKDB`, off on this
  instance).

## 2.6 CLI tool

Source: https://glitchtip.com/documentation/cli

For source maps, releases, and scripted operations.

```bash
# install
curl -fsSL https://glitchtip.com/install.sh | sh
# or: cargo install glitchtip-cli

# source maps
glitchtip-cli sourcemaps inject ./dist
glitchtip-cli sourcemaps upload ./dist --release 1.0.0 --org my-org --project my-project

# releases
glitchtip-cli releases new 1.0.0 --org my-org --project my-project
glitchtip-cli releases finalize 1.0.0 --org my-org --project my-project
glitchtip-cli releases set-commits 1.0.0 --auto --org my-org --project my-project
glitchtip-cli releases list --org my-org --project my-project
glitchtip-cli releases delete 1.0.0 --org my-org --project my-project
```

Also supports debug-file uploads, deployment tracking, uptime monitor
management, issue resolution, and log queries from the command line.

## 2.7 Integrations

Source: https://glitchtip.com/documentation/integrations

Two integrations are explicitly documented; beyond these, GlitchTip's claim
is broad Sentry-API compatibility — *"anything that works with Sentry should
also work with GlitchTip."*

**GitLab Error Tracking:** GitLab project → Settings → Monitor → Error
Tracking → enable → provider "Sentry" → enter this instance's URL. Needs a
GlitchTip **Auth Token** (Profile → Auth Tokens); tokens can be scoped
read-only, and grant access to any project the issuing user can see.

**Grafana data source:** Grafana → Configuration → Data sources → add →
search "Sentry" → enter the GlitchTip URL (no trailing slash) and the org
name (one data source per org) → needs a read-only Auth Token → Save & Test.

## 2.8 MCP server (AI assistant access)

Source: https://glitchtip.com/documentation/mcp

A built-in MCP server exposes 17 tools so an AI assistant (Claude Desktop,
Claude Code, or any MCP-compatible client) can browse issues, read stack
traces and event data, resolve issues, inspect performance data (including
spotting N+1 query patterns), search logs with trace correlation, and check
alert/uptime config — directly against this GlitchTip instance.

- Enable: `GLITCHTIP_ENABLE_MCP=True` (currently `False` on this instance).
- Endpoint: `https://glitchtip.wecreativeagency.dz/mcp`
- Auth: OAuth 2.0 (auto-discovered by supporting clients) or a static API token.

## 2.9 SDKs

Source: https://glitchtip.com/sdkdocs

GlitchTip uses the Sentry SDK ecosystem unchanged — install the Sentry SDK
for your platform, point it at this instance's project DSN instead of
Sentry's, and it works. Broad platform coverage:

- **JS/web:** JavaScript, React, Vue, Angular, Next.js
- **Node:** Node.js, Express, Koa, Connect
- **Python:** Django, Flask, FastAPI, Celery, AWS Lambda, ASGI/WSGI, and more
- **Mobile:** Android, iOS (Objective-C, Swift), React Native, Flutter
- **.NET:** ASP.NET Core, C#
- **JVM:** Java, Log4j 1/2, Logback, java.util.logging
- **Other:** Go, Ruby/Rails, PHP/Laravel/Symfony/Drupal, Elixir, Rust,
  Electron, Native C/C++

Any platform not listed: use the generic Sentry SDK setup guide — GlitchTip's
Sentry-protocol compatibility means it should still work.

## 2.10 Reducing event volume / cost

Source: https://glitchtip.com/documentation/frequently-asked-questions

- Performance events: lower `tracesSampleRate` (e.g. `0.01` = 1% of
  transactions captured).
- Error events: lower `sampleRate` (e.g. `0.5` = 50% of errors captured).
- Custom sampler functions are available in the SDKs for finer-grained control.

Not directly relevant to a self-hosted instance (no metered billing here),
but relevant if this instance's Postgres starts growing faster than
expected — dialing down sampling on noisy/high-traffic projects is the lever.

## 2.11 Privacy / GDPR

Source: https://glitchtip.com/documentation/frequently-asked-questions

Stated policies from GlitchTip's own hosted offering (useful context, though
this is a self-hosted instance so *we* control the data, not GlitchTip Inc.):
no use of customer data to train AI models, data processing agreements
available on request, minimal cookie use. Self-hosting this instance means
none of your event data ever reaches GlitchTip's own servers at all — it's
entirely inside `/opt/glitchtip`'s Postgres volume on this VPS.

## 2.12 Not covered here

- **API docs** — full REST API reference: https://app.glitchtip.com/api/docs
  (GlitchTip is Sentry-API-compatible, so most Sentry API tooling/clients work
  against it too).
- **Hosted architecture** — how GlitchTip Inc. runs their own SaaS (DOKS,
  CloudNativePG, etc.) — not applicable to this self-hosted setup, included
  in the docs index for completeness only.
- **Contributing / self-hosting install** — already applied in Part 1;
  original install guide: https://glitchtip.com/documentation/install

---

## Quick links

| Topic | URL |
| --- | --- |
| Docs home | https://glitchtip.com/documentation |
| Getting started | https://glitchtip.com/documentation/getting-started |
| Error tracking | https://glitchtip.com/documentation/error-tracking |
| Performance | https://glitchtip.com/documentation/performance |
| Uptime monitoring | https://glitchtip.com/documentation/uptime-monitoring |
| Logs | https://glitchtip.com/documentation/logs |
| Integrations | https://glitchtip.com/documentation/integrations |
| MCP server | https://glitchtip.com/documentation/mcp |
| CLI | https://glitchtip.com/documentation/cli |
| SDK list | https://glitchtip.com/sdkdocs |
| Self-hosting / install | https://glitchtip.com/documentation/install |
| FAQ | https://glitchtip.com/documentation/frequently-asked-questions |
| REST API reference | https://app.glitchtip.com/api/docs |
