# gatus

Deployment configuration for [Gatus](https://github.com/TwiN/gatus) on
`seed-vm-services-eastus`. **Not a fork** — this repository holds only the
Compose file and the monitor definitions.

Deployed by Komodo as the stack `gatus`. It replaced Uptime Kuma on 2026-09-11.

| | |
| --- | --- |
| Public | <https://uptime.seedaps.com> — Cloudflare Tunnel |
| VNet | `http://172.17.0.4:3001` — `netsh portproxy` |
| Image | `ghcr.io/twin/gatus:v5.36.0` |

## One instance, internal audience

A single board at <https://uptime.seedaps.com>, behind HTTP basic auth and
reached through the Cloudflare Tunnel. There is no public status page.

There was one — an unauthenticated `gatus-public` stack on port 3002 at
`status.seedaps.com`, with `compose.public.yaml` and a `config-public/`
directory. It was **retired on 2026-09-14**. `git log` has the whole thing if it
is ever wanted back; `git revert` of that commit restores the compose file, the
config directory and the symlink in one step.

Its DNS record was deliberately **kept** while the tunnel ingress rule was
removed, so the hostname returns the tunnel's clean `404` instead of falling
through to the `*.seedaps.com` wildcard and landing on an unrelated Azure App
Service. That is a standing rule here, not a one-off — see ADR-0009 in the
infrastructure documentation.

## Groups

| Group | What it covers |
| --- | --- |
| `seed` | 33 tenants on the shared multi-tenant application |
| `coortex` | Coortex production environments |
| `coortex dev` | their development counterparts |
| `infrastructure` | n8n, SigNoz, the Azure SQL gateway |
| `internal` | services reached directly on the Docker host, plus the admin portal |

## Write conditions against the body, not the status

The single most important rule in this repository.

The `seedaps.com` zone carries a wildcard, `*.seedaps.com` pointing at an
unrelated Azure App Service. **Every name under it resolves and answers `200`,**
whether or not the tunnel routes it — so `[STATUS] == 200` proves nothing.

This is not theoretical, and it has bitten twice.

On 2026-09-11 a check for `status.seedaps.com` passed a status-only condition
**before its DNS record or ingress rule existed**. Two invented hostnames
returned the same `200`, and the same `401 {"Message": ""}` on `/api/health`, as
the real ones.

On 2026-09-14 it turned out **`*.coortex.com` is a wildcard too**, and its
`/health` answers `Healthy` for *any* hostname —
`zzz-nope-7712.coortex.com/health` included. Checks that asserted
`[BODY] == Healthy` there looked rigorous and proved only that the shared
platform was alive. The coortex endpoints now check the **root** and assert on
the page title, because an unconfigured host 404s there.

For the `seed` tenants the discriminator is inverted: every hostname under
`seedaps.com` renders a login page, but an **unconfigured** one shows
`Endere&#231;o de empresa inexistente.`, so the condition is
`[BODY] != pat(*inexistente*)`. Verified across all 33 tenants and four invented
controls.

So:

```yaml
conditions:
  - "[STATUS] == 200"
  - "[BODY] == Healthy"        # or [BODY].status == UP for Gatus's own /health
```

The body is the part the wildcard cannot fake. A check that asserts only on the
status code is a green light wired to nothing.

## Adding or changing a monitor

Edit a file under `config/`, commit, push. Komodo redeploys, and Gatus reloads
its configuration without a restart.

`GATUS_CONFIG_PATH=/config`, so **every** `*.yaml` in that directory is merged:
maps are deep-merged and lists are appended. Add a new file rather than growing
one, if that reads better.

Conditions must be HTTP-level. Do not use `tcp://` checks against anything
reached through `netsh portproxy` — portproxy answers the TCP handshake before
dialling the backend, so such a check reports up permanently even when nothing
is listening.

## Two things that will bite

**Pin the image.** On this project `:latest` tracks **master**, not the newest
release — `:stable` is the tag that means what most people expect. Always pin an
explicit `vX.Y.Z`.

**There is no healthcheck, and there cannot be one.** The image is `FROM
scratch`: a static binary, a CA bundle, a default config. No shell, no `wget`.
Any `healthcheck:` fails to exec and reports unhealthy forever. Gatus watches its
own `/health` instead.

## Branding

`config/01-ui.yaml` holds the whole `ui:` block — title, headings, the Seed APS
logo and the theme. Everything under `ui:` is kept in that one file so nothing
depends on how Gatus deep-merges a *map* across files; only distinct top-level
keys are relied on.

The logo is inlined as a `data:` URI rather than hot-linked. It is 17 KB, so the
cost is ~23 KB of base64 in the file, and in exchange the status page has no
third-party dependency on every load, leaks no visitor IPs to `seed.com.br`, and
still renders correctly if `seed.com.br` is itself the thing that is down.

Three things to know before editing `custom-css`:

- **Gatus is shadcn-themed.** Every colour resolves through HSL custom
  properties — `.bg-background{background-color:hsl(var(--background))}` — so
  re-theming means overriding ~20 variables, not chasing utility classes.
- **`/css/custom.css` is linked _before_ `/css/app.css`.** A plain `:root`
  (specificity 0,1,0) ties with app.css's own `:root` and loses on source order.
  The selectors here are doubled (`:root:root`, 0,2,0) for that reason.
- **There is no theme toggle in the UI.** The visitor's `prefers-color-scheme`
  alone decides. The Seed wordmark is solid white, so the same palette is
  applied to `:root` *and* `.dark` — every visitor gets the same legible page.

The logo also ships inside a 48x48 box (`w-12 h-12`) with `object-contain`,
which renders a 1080x509 wordmark at 48x23. The CSS releases that box.

## Secrets

`GATUS_BASIC_AUTH_B64` — base64 of a bcrypt hash (cost 9), protecting the dashboard's
status data. It is supplied by the Komodo stack environment and is **not** in
this repository. Badge and raw uptime/response-time endpoints are unauthenticated
by design in Gatus, so treat endpoint *names* as public.

## Storage

SQLite in the named Docker volume `gatus-data`, mounted at `/data`. It holds
history only — the configuration is this repository, so losing the volume costs
graphs, not monitors.
