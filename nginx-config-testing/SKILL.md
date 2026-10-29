---
name: nginx-config-testing
description: Use when editing an nginx.conf or default.conf.template, a Dockerfile that serves a built Angular or Vite/React front end with nginx, or a code-generator template that writes one, and when you must prove the config with the real nginx image before shipping. Trigger words - gzip, hashed file caching, no-cache index.html, upstream keepalive, envsubst, template substitution.
---

# nginx for a built SPA: write it, then run it

## The config that worked (compression, caching, keep-alive)

```nginx
upstream api_backend { server ${API_UPSTREAM}; keepalive 16; }     # host:port only, not a URL

server {
    listen 80;
    root /usr/share/nginx/html;                                    # root at server level, not inside "location /"

    gzip on; gzip_vary on; gzip_proxied any; gzip_comp_level 5; gzip_min_length 1024;
    gzip_types application/javascript text/javascript text/css application/json image/svg+xml text/plain application/xml;

    location ~ "-[A-Z0-9]{8}\.(?:js|css|woff2?)$" {                # Angular: 8 character hash in the name
        add_header Cache-Control "public, max-age=31536000, immutable";
        try_files $uri =404;
    }
    # Vite/React instead:  location /assets/ { same two lines }
    location = /index.html { add_header Cache-Control "no-cache"; }
    location /api/ {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
    }
    location / { try_files $uri $uri/ /index.html; }
}
```

## Rules and traps

- **Immutable caching only for files whose name changes with their content.** A pattern on extension alone (`png|ico|svg`) also catches `favicon.ico` and anything copied from `public/`, which keep their names and would be cached for a year. Angular: `-HASH8.js/css/woff2`. Vite: everything under `/assets/`.
- **A regex `location` containing `{8}` must be quoted** (`location ~ "...{8}\.js$"`); unquoted, nginx reads `{` as the start of a block and `nginx -t` fails. Keep the backslash in `\.`.
- **`location = /index.html` also covers the SPA fallback**: `try_files ... /index.html` is an internal redirect that re-matches locations, so deep links get `no-cache` too. `text/html` is compressed by `gzip on` without listing it.
- **`add_header` inside a location replaces the server-level ones**; repeat any header you still need there.
- **`try_files $uri =404` on the hashed-file location** turns a missing hashed file into a 404 instead of serving index.html as JavaScript.
- **Keep-alive to the API needs an `upstream` block with `keepalive`** (plus HTTP/1.1 and an empty `Connection` header) when the nginx is old. nginx 1.31 enables upstream keepalive by default, so a connection-count test cannot tell the two apart on the current `nginx:alpine`; the directives are still right for older images. If `proxy_pass` is a full URL such as the `https://host:port` Aspire injects (`${services__timeentryapi__https__0}`), there is no upstream block, so only the header half is possible; splitting host and port out of the URL is the real fix.
- **The official image substitutes only environment variables** in `/etc/nginx/templates/*.template` (output goes to `/etc/nginx/conf.d/`), so nginx's own `$uri`, `$host` are safe. The template is included inside `http {}`, so `upstream` and `server` may sit at its top level.
- gzip also compresses the proxied API's JSON when the browser sends `Accept-Encoding: gzip`; an API that already sets `Content-Encoding` is not compressed twice.

## Prove it before shipping (Docker Desktop, Git Bash)

```bash
export MSYS_NO_PATHCONV=1                       # else Git Bash rewrites /etc/... in docker arguments
docker run -d --name t -p 18090:80 -e API_UPSTREAM=api:8080 \
  -v "$(cygpath -w $PWD/dist)":/usr/share/nginx/html:ro \
  -v "$(cygpath -w $PWD/nginx.conf.template)":/etc/nginx/templates/default.conf.template:ro nginx:alpine
docker exec t nginx -t
curl -sI -H 'Accept-Encoding: gzip' http://localhost:18090/main-XXXXXXXX.js     # Content-Encoding + Cache-Control
curl -s -H 'Accept-Encoding: gzip' URL | wc -c ; curl -s URL | wc -c           # compressed vs plain bytes
```

- Compare the old and the new config side by side on two ports with the same `dist`; the byte counts are the evidence (518,150 to 119,242 for a 520 kB Angular bundle).
- Check four URLs: a hashed file, an unhashed one (`favicon.ico`, expect no Cache-Control), `index.html`, and a router path such as `/employees/7` (expect 200 with `no-cache`).
- To test the proxy half without the real API, put a second `nginx:alpine` on a user-defined network serving a JSON file under `/srv/api/`, and give the front end `API_UPSTREAM=thatname:8080`.
- Counting upstream connections: `stub_status` on the API side (`location /status { stub_status; }`); the third number on line 3 is total accepts. Use `127.0.0.1`, not `localhost`, in `wget` inside the container. Upstream connections are pooled **per worker process** (18 workers here), so 10 requests show 10 new connections even with `keepalive`: send about 200 requests and expect roughly one connection per worker (16 measured) against 200 without a pool.
- Pick ports above 18000; Windows reserves ranges (8082 was refused with "forbidden by its access permissions").
- `curl -D - -o /dev/null -w '%{size_download}'` can print 0 under Git Bash; pipe the body into `wc -c` instead.

## In a code generator template

A template line may not hold two hyphens in a row (CodeGenNew `TemplateDashTests`), so write comments with one hyphen or none. Assert the directives in a test over the generated text, and extract the generated file between its `@@@FILE` markers to run it through the real nginx as above.

## How this was verified

- The config and the byte counts (518,150 to 119,242 for a 520 kB Angular bundle) come from a run against the `nginx:alpine` image on Docker Desktop on 2026-10-08 (the only commit of this skill).
- Re-run on 2026-10-29 with `nginx:alpine` = nginx 1.31.8 (Docker Engine 28.0.1), using the config above with a generated 200 kB script, a favicon, an `index.html` and a second nginx as the API:
  - `nginx -t` passes; the official image's template step replaced only `${API_UPSTREAM}` and kept `$uri`.
  - The hashed file answered `Content-Encoding: gzip` and `Cache-Control: public, max-age=31536000, immutable` (148,715 bytes compressed against 200,000 plain, for random text); `favicon.ico` had no `Cache-Control`; `index.html` and the deep link `/employees/7` answered 200 with `no-cache`; a missing hashed file answered 404; an unhashed file under `/assets/` had no cache header with the Angular pattern.
  - The unquoted `{8}` regex failed `nginx -t` (`unknown directive "8}\.js$"`); a location with its own `add_header` dropped the server-level header.
  - Upstream keepalive, 200 requests: an `upstream` block with `keepalive 16` and HTTP/1.1 made 16 connections; a bare `proxy_pass http://host:port` made 200; an `upstream` block with no `keepalive` also made 16, which confirms nginx 1.31 pools by default when an `upstream` block exists.
- Not re-run: the byte counts of the real Angular bundle, gzip of proxied API JSON, the `curl %{size_download}` oddity under Git Bash, a template written by a code generator.

## What this does not prove

- That the config is right for another nginx version or image (only 1.31.8 was tested), or behind a CDN or an ingress.
- That it is safe to ship: it has no TLS, security headers or rate limits.
- That the caching rules suit a build that does not hash its file names the way Angular (8 characters) or Vite (`/assets/`) does.
