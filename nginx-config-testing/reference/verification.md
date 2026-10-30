# How the nginx-config-testing skill was verified

Evidence behind the claims in [SKILL.md](../SKILL.md). Moved out of it so that it is not read on every load.

- The config and the byte counts (518,150 to 119,242 for a 520 kB Angular bundle) come from a run against the `nginx:alpine` image on Docker Desktop on 2026-10-08 (the only commit of this skill).
- Re-run on 2026-10-29 with `nginx:alpine` = nginx 1.31.8 (Docker Engine 28.0.1), using the config above with a generated 200 kB script, a favicon, an `index.html` and a second nginx as the API:
  - `nginx -t` passes; the official image's template step replaced only `${API_UPSTREAM}` and kept `$uri`.
  - The hashed file answered `Content-Encoding: gzip` and `Cache-Control: public, max-age=31536000, immutable` (148,715 bytes compressed against 200,000 plain, for random text); `favicon.ico` had no `Cache-Control`; `index.html` and the deep link `/employees/7` answered 200 with `no-cache`; a missing hashed file answered 404; an unhashed file under `/assets/` had no cache header with the Angular pattern.
  - The unquoted `{8}` regex failed `nginx -t` (`unknown directive "8}\.js$"`); a location with its own `add_header` dropped the server-level header.
  - Upstream keepalive, 200 requests: an `upstream` block with `keepalive 16` and HTTP/1.1 made 16 connections; a bare `proxy_pass http://host:port` made 200; an `upstream` block with no `keepalive` also made 16, which confirms nginx 1.31 pools by default when an `upstream` block exists.
- Not re-run: the byte counts of the real Angular bundle, gzip of proxied API JSON, the `curl %{size_download}` oddity under Git Bash, a template written by a code generator.
