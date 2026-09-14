# brand

Source assets for the dashboards.

`mark-white.png` is the white Seed mark as supplied. The `favicon-*.png` files
are box-filtered downsamples of it, generated with alpha premultiplied so the
transparent edges do not bleed grey into a white mark.

## Why these are files and not inline data: URIs

Gatus renders `ui.favicon` into an HTML `href`, and Go's `html/template` URL
filter **rejects the `data:` scheme** — it replaces the whole attribute with the
sentinel `#ZgotmplZ`. Only `http(s):` and relative URLs survive.

`ui.logo` behaves differently: it lands inside a JavaScript string literal
(`window.config = {logo: "..."}`), where a `data:` URI is accepted. That is why
the header logo is inlined and these are not — the same value is safe in one
template context and filtered in the other.

## Where they are served from

`raw.githubusercontent.com`, via `config-public/02-favicon.yaml`. That is a
third party, but not a *new* one: Komodo already pulls this repository from
GitHub in order to deploy at all, so it adds no dependency that was not already
load-bearing. A favicon that fails to load is also purely cosmetic — it does not
affect the status page.

To make it first-party, host these on seed.com.br and change the three URLs in
`config-public/02-favicon.yaml`.
