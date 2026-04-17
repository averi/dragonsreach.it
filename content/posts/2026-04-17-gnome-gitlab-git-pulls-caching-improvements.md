---
title: "GNOME GitLab Git traffic caching"
date: 2026-04-17T10:00:00-04:00
type: post
url: /2026/04/17/gnome-gitlab-git-pulls-caching-improvements/
categories:
  - Syseng
tags:
  - syseng
  - gitlab
  - nginx
  - fastly
  - caching

---
## Table of Contents
- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [The problem](#the-problem)
- [Architecture overview](#architecture-overview)
- [The Nginx and Lua layer](#the-nginx-and-lua-layer)
- [Building the cache key](#building-the-cache-key)
- [The POST-to-GET conversion](#the-post-to-get-conversion)
- [Protecting private repositories](#protecting-private-repositories)
- [The Fastly VCL](#the-fastly-vcl)
- [Conclusions](#conclusions)

## Introduction

One of the most visible signs that GNOME's infrastructure has grown over the years is the amount of CI traffic that flows through [gitlab.gnome.org](https://gitlab.gnome.org) on any given day. Hundreds of pipelines run in parallel, most of them starting with a `git clone` or `git fetch` of the same repository, often at the same commit. All that traffic was landing directly on GitLab's webservice pods, generating redundant load for work that was essentially identical.

GNOME's infrastructure runs on AWS, which generously provides credits to the project. Even so, data transfer is one of the largest cost drivers we face, and we have to operate within a defined budget regardless of those credits. The bandwidth costs associated with this Git traffic grew significant enough that for a period of time we redirected unauthenticated HTTPS Git pulls to our GitHub mirrors as a short-term cost mitigation. That measure bought us some breathing room, but it was never meant to be permanent: sending users to a third-party platform for what is essentially a core infrastructure operation is not a position we wanted to stay in. The goal was always to find a proper solution on our own infrastructure.

This post documents the caching layer we built to address that problem. The solution sits between the client and GitLab, intercepts Git fetch traffic, and routes it through Fastly's CDN so that repeated fetches of the same content are served from cache rather than generating a fresh pack every time.

## The problem

The Git smart HTTP protocol uses two endpoints: `info/refs` for capability advertisement and ref discovery, and `git-upload-pack` for the actual pack generation. The second one is the expensive one. When a CI job runs `git fetch origin main`, GitLab has to compute and send the entire pack for that fetch negotiation. If ten jobs run the same fetch within a short window, GitLab does that work ten times.

The tricky part is that `git-upload-pack` is a `POST` request with a binary body that encodes what the client already has (`have` lines) and what it wants (`want` lines). Traditional HTTP caches ignore POST bodies entirely. Building a cache that actually understands those bodies and deduplicates identical fetches requires some work at the edge.

For a fresh clone the body contains only `want` lines — one per ref the client is requesting:

```
0032want 7d20e995c3c98644eb1c58a136628b12e9f00a78
0032want 93e944c9f728a4b9da506e622592e4e3688a805c
0032want ef2cbad5843a607236b45e5f50fa4318e0580e04
...
```

For an incremental fetch the body is a mix of `want` lines (what the client needs) and `have` lines (commits the client already has locally), which the server uses to compute the smallest possible packfile delta:

```
00a4want 51a117587524cbdd59e43567e6cbd5a76e6a39ff
0000
0032have 8282cff4b31dce12e100d4d6c78d30b1f4689dd3
0032have be83e3dae8265fdc4c91f11d5778b20ceb4e2479
0032have 7d46abdf9c5a3f119f645c8de6d87efffe3889b8
...
```

The leading four hex characters on each line are the pkt-line length prefix. The server walks back through history from the wanted commits until it finds a common ancestor with the `have` set, then packages everything in between into a packfile. Two CI jobs running the same pipeline at the same commit will produce byte-for-byte identical request bodies and therefore identical responses — exactly the property a cache can help with.

## Architecture overview

The overall setup involves four components:

- **OpenResty** (Nginx + LuaJIT) running as a reverse proxy in front of GitLab's webservice
- **Fastly** acting as the CDN, with custom VCL to handle the non-standard caching behaviour
- **Valkey** (a Redis-compatible store) holding the denylist of private repositories
- **gitlab-git-cache-webhook**, a small Python/FastAPI service that keeps the denylist in sync with GitLab

<pre class="mermaid">
flowchart TD
    client["Git client / CI runner"]
    gitlab_gnome["gitlab.gnome.org (Nginx reverse proxy)"]
    nginx["OpenResty Nginx"]
    lua["Lua: git_upload_pack.lua"]
    cdn_origin["/cdn-origin internal location"]
    fastly_cdn["Fastly CDN"]
    origin["gitlab.gnome.org via its origin (second pass)"]
    gitlab["GitLab webservice"]
    valkey["Valkey denylist"]
    webhook["gitlab-git-cache-webhook"]
    gitlab_events["GitLab project events"]

    client --> gitlab_gnome
    gitlab_gnome --> nginx
    nginx --> lua
    lua -- "check denylist" --> valkey
    lua -- "private repo: BYPASS" --> gitlab
    lua -- "public/internal: internal redirect" --> cdn_origin
    cdn_origin --> fastly_cdn
    fastly_cdn -- "HIT" --> cdn_origin
    fastly_cdn -- "MISS: origin fetch" --> origin
    origin --> gitlab
    gitlab_events --> webhook
    webhook -- "SET/DEL git:deny:" --> valkey
</pre>

The request path for a public or internal repository looks like this:

1. The Git client runs `git fetch` or `git clone`. Git's smart HTTP protocol translates this into two HTTP requests: a `GET /Namespace/Project.git/info/refs?service=git-upload-pack` for ref discovery, followed by a `POST /Namespace/Project.git/git-upload-pack` carrying the negotiation body. It is that second request — the expensive pack-generating one — that the cache targets.
2. It arrives at `gitlab.gnome.org`'s Nginx server, which acts as the reverse proxy in front of GitLab's webservice.
3. The `git-upload-pack` location runs a Lua script that parses the repo path, reads the request body, and SHA256-hashes it. The hash is the foundation of the cache key: because the body encodes the exact set of `want` and `have` SHAs the client is negotiating, two jobs fetching the same commit from the same repository will produce byte-for-byte identical bodies and therefore the same hash — making the cached packfile safe to reuse.
4. Lua checks Valkey: is this repo in the denylist? If yes, the request is proxied directly to GitLab with no caching.
5. For public/internal repos, Lua strips the `Authorization` header, builds a cache key, converts the `POST` to a `GET`, and does an internal redirect to `/cdn-origin`. The POST-to-GET conversion is necessary because Fastly does not apply consistent hashing to POST requests — each of the hundreds of nodes within a POP maintains its own independent cache storage, so the same POST request hitting different nodes will always be a miss. By converting to a GET, Fastly's consistent hashing kicks in and routes requests with the same cache key to the same node, which means the cache is actually shared across all concurrent jobs hitting that POP.
6. The `/cdn-origin` location proxies to the Fastly git cache CDN with the `X-Git-Cache-Key` header set.
7. Fastly's VCL sees the key and does a cache lookup. On a HIT it returns the cached pack. On a MISS it fetches from `gitlab.gnome.org` directly via its origin (bypassing the CDN to avoid a loop) — the same Nginx instance — and caches the response for 30 days.
8. On that second pass (origin fetch), Nginx detects the `X-Git-Cache-Internal` header, decodes the original POST body from `X-Git-Original-Body`, restores the request method, and proxies to GitLab.

## The Nginx and Lua layer

The Nginx configuration exposes two relevant locations. The first is the internal one used for the CDN proxy leg:

```Nginx
location ^~ /cdn-origin/ {
    internal;
    rewrite ^/cdn-origin(/.*)$ $1 break;
    proxy_pass $cdn_upstream;
    proxy_ssl_server_name on;
    proxy_ssl_name <cdn-hostname>;
    proxy_set_header Host <cdn-hostname>;
    proxy_set_header Accept-Encoding "";
    proxy_http_version 1.1;
    proxy_buffering on;
    proxy_request_buffering off;
    proxy_connect_timeout 10s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;

    header_filter_by_lua_block {
        ngx.header["X-Git-Cache-Key"] = ngx.req.get_headers()["X-Git-Cache-Key"]
        ngx.header["X-Git-Body-Hash"] = ngx.req.get_headers()["X-Git-Body-Hash"]

        local xcache = ngx.header["X-Cache"] or ""
        if xcache:find("HIT") then
            ngx.header["X-Git-Cache-Status"] = "HIT"
        else
            ngx.header["X-Git-Cache-Status"] = "MISS"
        end
    }
}
```

The `header_filter_by_lua_block` here is doing something specific: it reads `X-Cache` from the response Fastly returns and translates it into a clean `X-Git-Cache-Status` header for observability. The `X-Git-Cache-Key` and `X-Git-Body-Hash` are also passed through so that callers can see what cache entry was involved.

The second location is `git-upload-pack` itself, which delegates all the logic to a Lua file:

```Nginx
location ~ /git-upload-pack$ {
    client_body_buffer_size 1m;
    client_max_body_size 5m;

    access_by_lua_file /etc/nginx/lua/git_upload_pack.lua;

    header_filter_by_lua_block {
        local key = ngx.req.get_headers()["X-Git-Cache-Key"]
        if key then
            ngx.header["X-Git-Cache-Key"] = key
        end
    }

    proxy_pass http://gitlab-webservice;
    proxy_http_version 1.1;
    proxy_set_header Host gitlab.gnome.org;
    proxy_set_header X-Real-IP $http_fastly_client_ip;
    proxy_set_header X-Forwarded-For $http_fastly_client_ip;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-Port 443;
    proxy_set_header X-Forwarded-Ssl on;
    proxy_set_header Connection "";
    proxy_buffering off;
    proxy_request_buffering off;
    proxy_connect_timeout 10s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
}
```

The `access_by_lua_file` directive runs before the request is proxied. If the Lua script calls `ngx.exec("/cdn-origin" .. uri)`, Nginx performs an internal redirect to the CDN location and the `proxy_pass` to GitLab is never reached. If the script returns normally (for private repos or non-fetch commands), the request falls through to the `proxy_pass`.

## Building the cache key

The full Lua script that runs in `access_by_lua_file` handles both passes of the request. The first pass (client → nginx) does the heavy lifting:

```Lua
local resty_sha256 = require("resty.sha256")
local resty_str = require("resty.string")
local redis_helper = require("redis_helper")

local redis_host = os.getenv("REDIS_HOST") or "localhost"
local redis_port = os.getenv("REDIS_PORT") or "6379"

-- Second pass: request arriving from CDN origin fetch.
-- Decode the original POST body from the header and restore the method.
if ngx.req.get_headers()["X-Git-Cache-Internal"] then
    local encoded_body = ngx.req.get_headers()["X-Git-Original-Body"]
    if encoded_body then
        ngx.req.read_body()
        local body = ngx.decode_base64(encoded_body)
        ngx.req.set_method(ngx.HTTP_POST)
        ngx.req.set_body_data(body)
        ngx.req.set_header("Content-Length", tostring(#body))
        ngx.req.clear_header("X-Git-Original-Body")
    end
    return
end
```

The second-pass guard is at the top of the script. When Fastly's origin fetch arrives, it will carry `X-Git-Cache-Internal: 1`. The script detects that, reconstructs the POST body from the base64-encoded header, restores the `POST` method, and returns — allowing Nginx to proxy the real request to GitLab.

For the first pass, the script parses the repo path from the URI, reads and buffers the full request body, and computes a SHA256 over it:

```Lua
-- Only cache "fetch" commands; ls-refs responses are small, fast, and
-- become stale on every push (the body hash is constant so a long TTL
-- would serve outdated ref listings).
if not body:find("command=fetch", 1, true) then
    ngx.header["X-Git-Cache-Status"] = "BYPASS"
    return
end

-- Hash the body
local sha256 = resty_sha256:new()
sha256:update(body)
local body_hash = resty_str.to_hex(sha256:final())

-- Build cache key: cache_versioning + repo path + body hash
local cache_key = "v2:" .. repo_path .. ":" .. body_hash
```

A few things worth noting here. The `ls-refs` command is explicitly excluded from caching. The reason is that `ls-refs` is used to list references and its request body is essentially static (just a capability advertisement). If we cached it with a 30-day TTL, a push to the repository would not invalidate the cache — the key would be the same — and clients would get stale ref listings. Fetch bodies, on the other hand, encode exactly the SHAs the client wants and already has. The same set of `want`/`have` lines always maps to the same pack, which makes them safe to cache for a long time.

The `v2:` prefix is a cache version string. It makes it straightforward to invalidate all existing cache entries if we ever need to change the key scheme, without touching Fastly's purge API.

## The POST-to-GET conversion

This is probably the most unusual part of the design:

```Lua
-- Carry the POST body as a base64 header and convert to GET so that
-- Fastly's intra-POP consistent hashing routes identical cache keys
-- to the same server (Fastly only does this for GET, not POST).
ngx.req.set_header("X-Git-Original-Body", ngx.encode_base64(body))
ngx.req.set_method(ngx.HTTP_GET)
ngx.req.set_body_data("")

return ngx.exec("/cdn-origin" .. uri)
```

Fastly's shield feature routes cache misses through a designated intra-POP "shield" node before going to origin. When two different edge nodes both get a MISS for the same cache key simultaneously, the shield node collapses them into a single origin request. This is important for us because without it, a burst of CI jobs fetching the same commit would all miss, all go to origin in parallel, and GitLab would end up generating the same pack multiple times anyway.

The catch is that Fastly's consistent hashing and shield routing only works for GET requests. POST requests always go straight to origin. By converting the POST to a GET and encoding the body in a header, we get shield-level request collapsing for free.

The VCL on the Fastly side uses the `X-Git-Cache-Key` header (not the URL or method) as the cache key, so the GET conversion is invisible to the caching logic.

## Protecting private repositories

We cannot route private repository traffic through an external CDN — that would mean sending authenticated git content to a third-party cache. The way we prevent this is a denylist stored in Valkey. Before doing anything else, the Lua script checks whether the repository is listed there:

```Lua
local denied, err = redis_helper.is_denied(redis_host, redis_port, repo_path)

if err then
    ngx.log(ngx.ERR, "git-cache: Redis error for ", repo_path, ": ", err,
            " — cannot verify project visibility, bypassing CDN")
    ngx.header["X-Git-Cache-Status"] = "BYPASS"
    return
end

if denied then
    ngx.header["X-Git-Cache-Status"] = "BYPASS"
    ngx.header["X-Git-Body-Hash"] = body_hash:sub(1, 12)
    return
end

-- Public/internal repo: strip credentials before routing through CDN
ngx.req.clear_header("Authorization")
```

If Valkey is unreachable, the script logs an error and bypasses the CDN entirely, treating the repository as if it were private. This is the safe default: the cost of a Redis failure is slightly increased load on GitLab, not the risk of routing private repository content through an external cache. In practice, Valkey runs alongside Nginx on the same node, so true availability failures are uncommon.

The denylist is maintained by [gitlab-git-cache-webhook](https://gitlab.gnome.org/GNOME/gitlab-git-cache-webhook), a small FastAPI service. It listens for GitLab system hooks on `project_create` and `project_update` events:

```Python
HANDLED_EVENTS = {"project_create", "project_update"}

@router.post("/webhook")
async def webhook(request: Request, ...) -> Response:
    ...
    event = body.get("event_name", "")
    if event not in HANDLED_EVENTS:
        return Response(status_code=204)

    project = body.get("project", {})
    path = project.get("path_with_namespace", "")
    visibility_level = project.get("visibility_level")

    if visibility_level == 0:
        await deny_repo(path)
    else:
        removed = await allow_repo(path)
    return Response(status_code=204)
```

GitLab's `visibility_level` is `0` for private, `10` for internal, and `20` for public. Internal repositories are intentionally treated the same as public ones here: they are accessible to any authenticated user on the instance, so routing them through the CDN is acceptable. Only truly private repositories go into the denylist.

The key format in Valkey is `git:deny:<path_with_namespace>`. The Lua `redis_helper` module does an `EXISTS` check on that key. The webhook service also ships a reconciliation command (`python -m app.reconcile`) that does a full resync of all private repositories via the GitLab API, which is useful to run on first deployment or after any extended Valkey downtime.

## The Fastly VCL

On the Fastly side, three VCL subroutines carry the relevant logic. In `vcl_recv`:

```VCL
if (req.url ~ "/info/refs") {
    return(pass);
}
if (req.http.X-Git-Cache-Key) {
    set req.backend = F_Host_1;
    if (req.restarts == 0) {
        set req.backend = fastly.try_select_shield(ssl_shield_iad_va_us, F_Host_1);
    }
    return(lookup);
}
```

`/info/refs` is always passed through uncached — it is the capability advertisement step and caching it would cause problems with protocol negotiation. Requests carrying `X-Git-Cache-Key` get an explicit `lookup` directive and are routed through the shield. Everything else falls through to Fastly's default behaviour.

In `vcl_hash`, the cache key overrides the default URL-based key:

```VCL
if (req.http.X-Git-Cache-Key) {
    set req.hash += req.http.X-Git-Cache-Key;
    return(hash);
}
```

And in `vcl_fetch`, responses are marked cacheable when they come back with a 200 and a non-empty body:

```VCL
if (req.http.X-Git-Cache-Key && beresp.status == 200) {
    if (beresp.http.Content-Length == "0") {
        set beresp.ttl = 0s;
        set beresp.cacheable = false;
        return(deliver);
    }

    set beresp.cacheable = true;
    set beresp.ttl = 30d;
    set beresp.http.X-Git-Cache-Key = req.http.X-Git-Cache-Key;

    unset beresp.http.Cache-Control;
    unset beresp.http.Pragma;
    unset beresp.http.Expires;
    unset beresp.http.Set-Cookie;

    return(deliver);
}
```

The 30-day TTL is deliberately long. Git pack data is content-addressed: a pack for a given set of `want`/`have` lines will always be the same. As long as the objects exist in the repository, the cached pack is valid. The only case where a cached pack could be wrong is if objects were deleted (force-push that drops history, for instance), which is rare and would be handled by the cache version prefix forcing a key change rather than by expiry.

Empty responses (`Content-Length: 0`) are explicitly not cached. GitLab can return an empty body in edge cases and caching that would break all subsequent fetches for that key.

## Conclusions

The system has been running in production for a while now and the cache hit rate on fetch traffic has been consistently high (over 80%) for repositories that are cloned frequently by CI. The design deliberately keeps things simple: there is no custom invalidation logic and no TTL management beyond what Fastly handles natively. If something goes wrong with the cache layer, the worst case is that requests fall back to BYPASS and GitLab handles them directly, which is how things worked before.

That should be all for today, stay tuned!

<script type="module">
import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
mermaid.initialize({ startOnLoad: true, theme: 'neutral' });
</script>
