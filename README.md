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
