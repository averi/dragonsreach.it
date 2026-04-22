---
title: "GNOME GitLab Git traffic caching"
date: 2026-04-17T10:00:00-04:00
type: post
url: /2026/04/17/gnome-gitlab-git-pulls-caching-improvements/
categories:
  - Planet GNOME
  - Syseng
tags:
  - planet-gnome
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
- [The VCL layer](#the-vcl-layer)
- [The POST-to-GET conversion](#the-post-to-get-conversion)
- [Protecting private repositories](#protecting-private-repositories)
- [The Lua layer](#the-lua-layer)
- [Debugging the rollout](#debugging-the-rollout)
- [How we got here](#how-we-got-here)
- [Conclusions](#conclusions)

## Introduction

One of the most visible signs that GNOME's infrastructure has grown over the years is the amount of CI traffic that flows through [gitlab.gnome.org](https://gitlab.gnome.org) on any given day. Hundreds of pipelines run in parallel, most of them starting with a `git clone` or `git fetch` of the same repository, often at the same commit. All that traffic was landing directly on GitLab's webservice pods, generating redundant load for work that was essentially identical.

GNOME's infrastructure runs on AWS, which generously provides credits to the project. Even so, data transfer is one of the largest cost drivers we face, and we have to operate within a defined budget regardless of those credits. The bandwidth costs associated with this Git traffic grew significant enough that for a period of time we redirected unauthenticated HTTPS Git pulls to our GitHub mirrors as a short-term cost mitigation. That measure bought us some breathing room, but it was never meant to be permanent: sending users to a third-party platform for what is essentially a core infrastructure operation is not a position we wanted to stay in. The goal was always to find a proper solution on our own infrastructure.

This post documents the caching layer we built to address that problem. The solution sits between the client and GitLab, intercepts Git fetch traffic, and routes it through Fastly's CDN so that repeated fetches of the same content are served from cache rather than generating a fresh pack every time. The design went through several iterations — this post presents the final architecture first, then walks through [how we got here](#how-we-got-here) for readers interested in the evolution.

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

The current architecture has two components:

- **Fastly** as the user-facing CDN for `gitlab.gnome.org`, with custom VCL that intercepts `git-upload-pack` traffic, hashes the request body, converts the POST to a GET, and caches the response at edge POPs worldwide
- **OpenResty** (Nginx + LuaJIT) running as the origin server, with a minimal Lua script that restores the original POST and signals cacheability back to Fastly

<pre class="mermaid">
flowchart TD
    client["Git client / CI runner"]
    edge["Fastly Edge POP (nearest)"]
    shield["Fastly Shield POP (IAD)"]
    nginx["OpenResty Nginx (origin)"]
    lua["Lua: git_upload_pack.lua"]
    gitlab["GitLab webservice"]

    client -- "POST /git-upload-pack" --> edge
    edge -- "authenticated? → return(pass)" --> nginx
    edge -- "HIT → serve from edge" --> client
    edge -- "MISS → forward to shield" --> shield
    shield -- "HIT → return to edge (edge caches)" --> edge
    shield -- "MISS → fetch from origin" --> nginx
    nginx --> lua
    lua -- "restore POST, proxy" --> gitlab
    gitlab -- "packfile response" --> nginx
    nginx -- "X-Git-Cacheable: 1" --> shield
</pre>

The request flow:

1. The `POST /git-upload-pack` arrives at the nearest Fastly edge POP.
2. VCL checks for authentication headers (`Authorization`, `PRIVATE-TOKEN`, `Job-Token`). If present, the request is sent directly to origin with credentials intact — private repos and CI runner clones never enter the cache path.
3. VCL checks the body: if `Content-Length` exceeds 8 KB (the limit of what Fastly can read from `req.body`), or the body does not contain `command=fetch`, the request is passed through uncached.
4. For cacheable requests, VCL hashes the body with SHA256 to build the cache key, base64-encodes the body into `X-Git-Original-Body`, converts the request to GET, and does `return(lookup)`.
5. On a cache hit at the edge, the packfile is served immediately.
6. On a miss, the request routes to the IAD shield POP. If the shield has it cached, it returns the object and the edge caches it locally.
7. On a shield miss, the request reaches Nginx at the origin. Lua detects `X-Git-Original-Body`, restores the POST body, and proxies to GitLab.
8. The response flows back through the shield (which caches it) and the edge (which also caches it). Subsequent requests from the same region are served directly from the edge.

## The VCL layer

The `vcl_recv` snippet runs at priority 9, before the existing `enable_segmented_caching` snippet at priority 10 which would otherwise `return(pass)` for non-asset URLs:

```
# Snippet git-cache-vcl-recv : 9

# Edge: convert POST to GET, hash body, encode body in header
if (req.url ~ "/git-upload-pack$" && req.request == "POST") {
    # Authenticated requests bypass cache entirely (CI runners, private repos)
    if (req.http.Authorization || req.http.PRIVATE-TOKEN || req.http.Job-Token) {
        return(pass);
    }

    if (std.atoi(req.http.Content-Length) > 8192) {
        return(pass);
    }

    if (req.body !~ "command=fetch") {
        return(pass);
    }

    set req.http.X-Git-Cache-Key = "v3:" digest.hash_sha256(req.body);
    set req.http.X-Git-Original-Body = digest.base64(req.body);
    set req.request = "GET";

    set req.backend = F_Host_1;
    if (req.restarts == 0) {
        set req.backend = fastly.try_select_shield(ssl_shield_iad_va_us, F_Host_1);
    }
    return(lookup);
}

# Shield: request already converted to GET by the edge
if (req.http.X-Git-Cache-Key) {
    set req.backend = F_Host_1;
    return(lookup);
}
```

The auth check at the top is the first guard. GitLab CI runners authenticate with `Authorization: Basic <gitlab-ci-token:TOKEN>`, API clients use `PRIVATE-TOKEN` or `Job-Token`. Any request carrying these headers is sent straight to origin with credentials intact — it never enters the cache path, never has its body encoded, and never touches the Lua script. This is how private repositories are protected (see [Protecting private repositories](#protecting-private-repositories)).

The `command=fetch` filter means only Git protocol v2 fetch commands are cached. The `ls-refs` command is excluded because its request body is essentially static — caching it with a long TTL would serve stale ref listings after a push. Fetch bodies encode exactly the SHAs the client wants and already has, making them safe to cache indefinitely.

The `v3:` prefix is a cache version string. Bumping it invalidates all existing cache entries without touching Fastly's purge API.

The second `if` block handles the shield. When a cache miss at the edge forwards the request to the shield POP, the shield runs `vcl_recv` again. At that point the request is already a GET (the edge converted it), so the first block's `req.request == "POST"` check will not match. Without the second block, the request would fall through to the `enable_segmented_caching` snippet, which returns `pass` for any URL that is not an artifact or archive — effectively preventing the shield from ever caching git traffic.

The `vcl_hash` snippet overrides the default URL-based hash when a cache key is present:

```
# Snippet git-cache-vcl-hash : 10
if (req.http.X-Git-Cache-Key) {
    set req.hash += req.http.X-Git-Cache-Key;
    return(hash);
}
```

The `vcl_fetch` snippet caches 200 responses that carry the `X-Git-Cacheable` signal from Nginx:

```
# Snippet git-cache-vcl-fetch : 100
if (req.http.X-Git-Cache-Key) {
    if (beresp.status == 200 && beresp.http.X-Git-Cacheable == "1") {
        set beresp.http.Surrogate-Key = "git-cache " regsub(req.url.path, "/git-upload-pack$", "");

        set beresp.cacheable = true;
        set beresp.ttl = 30d;
        set beresp.http.X-Git-Cache-Key = req.http.X-Git-Cache-Key;

        unset beresp.http.Cache-Control;
        unset beresp.http.Pragma;
        unset beresp.http.Expires;
        unset beresp.http.Set-Cookie;

        return(deliver);
    }

    set beresp.ttl = 0s;
    set beresp.cacheable = false;
    return(deliver);
}
```

The `Surrogate-Key` line tags each cached object with both a global `git-cache` key and the repository path. This enables targeted purging — a single repository's cache can be flushed with `fastly purge --key "/GNOME/glib"`, or all git cache at once with `fastly purge --key "git-cache"`.

The 30-day TTL is deliberately long. Git pack data is content-addressed: a pack for a given set of `want`/`have` lines will always be the same. As long as the objects exist in the repository, the cached pack is valid. The only case where a cached pack could be wrong is if objects were deleted (force-push that drops history, for instance), which is rare and, on GNOME's GitLab, made even rarer by the [Gitaly custom hooks](https://gitlab.gnome.org/GNOME/gitaly-custom-hooks) we run to prevent force-pushes and history rewrites on protected namespaces. In those cases the cache version prefix would force a key change rather than relying on TTL expiry.

The `X-Git-Cacheable` header is intentionally **not** unset in `vcl_fetch`. This is important for the shielding architecture: when the shield caches the object, the stored headers include `X-Git-Cacheable: 1`. When the edge later fetches this object from the shield, the edge's own `vcl_fetch` sees the header and knows it is safe to cache locally. If `vcl_fetch` stripped the header, the edge would never cache — every request would be a local miss that has to travel back to the shield.

The cleanup happens in `vcl_deliver`, which runs last before the response reaches the client:

```
# Snippet git-cache-vcl-deliver : 100
if (req.http.X-Git-Cache-Key) {
    set resp.http.X-Git-Cache-Status = if(fastly_info.state ~ "HIT(?:-|\z)", "HIT", "MISS");
    unset resp.http.X-Git-Original-Body;

    if (!req.http.Fastly-FF) {
        unset resp.http.X-Git-Cacheable;
        unset resp.http.X-Git-Cache-Key;
    }
}
```

The `Fastly-FF` check distinguishes between inter-POP traffic (shield-to-edge) and the final client response. `Fastly-FF` is set when the request comes from another Fastly node. On the shield, where the request came from the edge, internal headers like `X-Git-Cacheable` and `X-Git-Cache-Key` are preserved — the edge's `vcl_fetch` needs them. On the edge, where the request came from the actual client, those headers are stripped from the final response. Only `X-Git-Cache-Status` is exposed to clients for observability.

## The POST-to-GET conversion

This is probably the most unusual part of the design. Fastly's consistent hashing and shield routing only works for GET requests. POST requests always go straight to origin. Fastly does provide a way to force POST responses into the cache — by returning `pass` in `vcl_recv` and setting `beresp.cacheable` in `vcl_fetch` — but it is a blunt instrument: there is no consistent hashing, no shield collapsing, and no guarantee that two nodes in the same POP will ever share the cached result.

By converting the POST to a GET in VCL, encoding the body in a header (`X-Git-Original-Body`), and using a body-derived SHA256 as the cache key, we get consistent hashing and shield-level request collapsing for free. The VCL uses the `X-Git-Cache-Key` header (not the URL or method) as the cache key, so the GET conversion is invisible to the caching logic.

Fastly's shield feature routes cache misses through a designated shield node before going to origin. When two different edge nodes both get a MISS for the same cache key simultaneously, the shield node collapses them into a single origin request. This is important because without it, a burst of CI jobs fetching the same commit would all miss, all go to origin in parallel, and GitLab would end up generating the same pack multiple times.

## Protecting private repositories

Private repository traffic must never enter the cache — that would mean sending authenticated git content through a third-party cache. The VCL handles this with a single check at the top of `vcl_recv`, before any body processing:

```
if (req.http.Authorization || req.http.PRIVATE-TOKEN || req.http.Job-Token) {
    return(pass);
}
```

Authenticated requests (CI runners, API clients, private repo clones) are sent directly to GitLab with credentials intact, completely bypassing the cache path. Unauthenticated requests are, by definition, accessing public repositories — the only kind that should be cached.

This approach follows the same trust model GitLab itself uses: credentials are the boundary between private and public. It requires no external state, cannot drift out of sync, and has no failure modes beyond Fastly itself.

An earlier iteration used a Valkey (Redis) denylist to track private repositories and a webhook service to keep it synchronized with GitLab — see [How we got here](#how-we-got-here) for why that was replaced.

## The Lua layer

With the VCL handling body hashing, the POST-to-GET conversion, and the auth bypass for private repos, the Lua script's role is reduced to the bare minimum. Every request that reaches Lua is guaranteed to be an unauthenticated clone of a public repository — the VCL already filtered out everything else. The script's only responsibilities are:

1. Detect that the request arrived from Fastly with an encoded body (the `X-Git-Original-Body` header).
2. Decode and restore the original POST.
3. Signal back to Fastly that the response is safe to cache.

```Lua
local encoded_body = ngx.req.get_headers()["X-Git-Original-Body"]
if not encoded_body then
    return
end

local body = ngx.decode_base64(encoded_body)
ngx.req.read_body()
ngx.req.set_method(ngx.HTTP_POST)
ngx.req.set_body_data(body)
ngx.req.set_header("Content-Length", tostring(#body))
ngx.req.clear_header("X-Git-Original-Body")

ngx.req.clear_header("Authorization")
ngx.ctx.git_cacheable = true
```

The `ngx.ctx.git_cacheable` flag is picked up by the `header_filter_by_lua_block` in the Nginx configuration, which translates it into the `X-Git-Cacheable: 1` response header that `vcl_fetch` checks:

```Nginx
location ~ /git-upload-pack$ {
    client_body_buffer_size 5m;
    client_max_body_size 5m;

    access_by_lua_file /etc/nginx/lua/git_upload_pack.lua;

    header_filter_by_lua_block {
        if ngx.ctx.git_cacheable then
            ngx.header["X-Git-Cacheable"] = "1"
        end
    }

    proxy_pass http://gitlab-webservice;
    ...
}
```

## Debugging the rollout

The rollout surfaced a few issues worth documenting for anyone building a similar setup on Fastly.

**Shielding introduces a second `vcl_recv` execution.** When the edge forwards a cache miss to the shield, the shield runs the entire VCL pipeline from scratch. The POST-to-GET conversion in `vcl_recv` checks for `req.request == "POST"`, but on the shield the request is already a GET. Without the fallback `if (req.http.X-Git-Cache-Key)` block, the shield's `vcl_recv` would fall through to the segmented caching snippet and `return(pass)` — making the shield unable to cache anything.

**Response headers must survive the shield-to-edge hop.** `vcl_fetch` and `vcl_deliver` both run on each node independently. If `vcl_fetch` on the shield strips a header after caching the object, the stored object will not have that header. When the edge fetches from the shield, the edge's `vcl_fetch` will not see it. The solution is to only strip internal headers in `vcl_deliver` on the final client response, using `Fastly-FF` to distinguish inter-POP traffic from client traffic.

**Fastly's `req.body` is limited to 8 KB.** VCL can only inspect the first 8192 bytes of a request body. For the vast majority of git fetch negotiations — especially shallow clones and CI pipelines fetching recent commits — the body is well under this limit. Requests with larger bodies (deep fetches with many `have` lines) fall through to `return(pass)` and are handled directly by GitLab without caching. This is an acceptable tradeoff: those large-body requests are typically unique negotiations that would not benefit from caching anyway.

**Git protocol v1 clients are not cached.** The VCL filters on `command=fetch`, which is a Git protocol v2 construct. Protocol v1 uses a different body format (`want`/`have` lines without the `command=` prefix). Since protocol v2 has been the default since git 2.26 (March 2020), the vast majority of traffic benefits from caching. Protocol v1 clients still work correctly — they simply bypass the cache.

**Authenticated requests must bypass cache before body processing.** The initial edge VCL converted all `git-upload-pack` POSTs to cacheable GETs, including authenticated requests from CI runners. The Lua denylist was supposed to catch private repos, but CI runners authenticate with `Authorization: Basic <gitlab-ci-token:TOKEN>` — a header the Lua script unconditionally stripped for any repo not on the denylist. This broke private repository CI builds with 401 errors. The fix was adding the auth header check as the very first guard in `vcl_recv`, before any body hashing or request conversion. This also made the entire denylist infrastructure unnecessary, since the auth boundary naturally separates private from public traffic.

## How we got here

The current architecture is the result of three iterations. The sections above describe the final design; this section documents the path we took to get there.

### Iteration 1: Separate CDN service with Lua-driven caching

The first version used a separate Fastly CDN service (`cdn.gitlab.gnome.org`) as the cache layer, with Nginx doing most of the heavy lifting in Lua:

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

In this design, the Lua script did everything: read the POST body, SHA256-hash it to build a cache key, check a Valkey denylist to exclude private repositories, convert the POST to a GET, encode the body in a header, and perform an internal redirect to a `/cdn-origin` location that proxied to the CDN. On a cache miss, the CDN would fetch from `gitlab.gnome.org` directly (the "second pass"), where Lua would detect the origin fetch, decode the body, restore the POST, and proxy to GitLab.

Private repositories were protected by a denylist stored in Valkey. A small FastAPI webhook service ([gitlab-git-cache-webhook](https://gitlab.gnome.org/GNOME/gitlab-git-cache-webhook)) listened for GitLab system hooks on `project_create` and `project_update` events, maintaining `git:deny:<path>` keys for private repositories (visibility level `0`). Internal repositories (level `10`) were treated the same as public (level `20`) since they are accessible to any authenticated user on the instance.

The Lua script for this design was substantially more complex:

```Lua
local resty_sha256 = require("resty.sha256")
local resty_str = require("resty.string")
local redis_helper = require("redis_helper")

local redis_host = os.getenv("REDIS_HOST") or "localhost"
local redis_port = os.getenv("REDIS_PORT") or "6379"

-- Second pass: request arriving from CDN origin fetch.
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

And on the first pass, it handled hashing, denylist checks, and the CDN redirect:

```Lua
if not body:find("command=fetch", 1, true) then
    ngx.header["X-Git-Cache-Status"] = "BYPASS"
    return
end

local sha256 = resty_sha256:new()
sha256:update(body)
local body_hash = resty_str.to_hex(sha256:final())
local cache_key = "v2:" .. repo_path .. ":" .. body_hash

local denied, err = redis_helper.is_denied(redis_host, redis_port, repo_path)
if denied then return end

ngx.req.clear_header("Authorization")
ngx.req.set_header("X-Git-Original-Body", ngx.encode_base64(body))
ngx.req.set_method(ngx.HTTP_GET)
ngx.req.set_body_data("")
return ngx.exec("/cdn-origin" .. uri)
```

The CDN's VCL was relatively simple — it used `X-Git-Cache-Key` for the hash, routed through a shield, and cached 200 responses for 30 days.

This architecture worked, but it had a significant limitation.

### Iteration 2: Moving caching to the edge

The problem with the separate CDN service was that Nginx runs in AWS us-east-1. From Fastly's perspective, the only client of the CDN service was that single Nginx instance in Virginia. Every request entered the CDN through the IAD (Ashland, Virginia) POP, which meant the CDN's edge POPs around the world were never used. The shield node in IAD cached the objects, but the edge POPs never got a chance to build up their own local caches.

A CI runner in Europe would have its request travel from a European Fastly POP to IAD (the `gitlab.gnome.org` service), then to Nginx in AWS, then back to Fastly IAD (the CDN service), and then all the way back. Every single request for a cached object still had to cross the Atlantic twice.

The fix was to eliminate the separate CDN service entirely and move all the caching logic into the `gitlab.gnome.org` Fastly service itself. The key insight was that the POST-to-GET conversion and body hashing could happen in Fastly's VCL rather than in Lua — Fastly provides `digest.hash_sha256()` and `digest.base64()` functions that operate directly on `req.body`. By doing the conversion at the CDN edge, every POP in the network became a potential cache node for git traffic.

This iteration still used the Valkey denylist and webhook to protect private repositories, with Lua checking the denylist and signaling cacheability via `X-Git-Cacheable`.

### Iteration 3: VCL auth bypass, denylist removed

The denylist approach had a fundamental flaw that surfaced once all `git-upload-pack` traffic flowed through the VCL cache path: authenticated requests from CI runners cloning private repositories were being converted to cacheable GETs. The Lua script would strip their `Authorization` header (if the repo was not on the denylist, or if the denylist was incomplete), and GitLab would reject the request with a 401.

The fix was adding the auth header check as the very first guard in `vcl_recv` — three lines of VCL that made the entire denylist infrastructure unnecessary. Authenticated requests go straight to origin. Unauthenticated requests are, by definition, public. The auth header is the correct boundary, and it requires no external state.

With this change, the Valkey instance, the `redis_helper.lua` module, and the `gitlab-git-cache-webhook` service were all decommissioned. The Lua script went from ~50 lines with Redis dependencies to 12 lines with no external dependencies.

## Conclusions

The system has been running in production since April 2026. Packfiles are cached at Fastly edge POPs worldwide — a CI runner in Europe gets a cache hit served from a European POP rather than making a round trip to the US East coast. The Lua script is twelve lines. The only moving parts are Fastly's VCL and Nginx.

The cache hit rate on fetch traffic has been consistently high (over 80%). If something goes wrong with the cache layer, requests fall through to GitLab directly — the same path they took before caching existed. There is no failure mode where caching breaks git operations. This also means we don't redirect any traffic to github.com anymore.

That should be all for today, stay tuned!

<script type="module">
import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
mermaid.initialize({ startOnLoad: true, theme: 'neutral' });
</script>
