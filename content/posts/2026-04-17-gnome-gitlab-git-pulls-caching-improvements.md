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

The current architecture has three components:

- **Fastly** as the user-facing CDN for `gitlab.gnome.org`, with custom VCL that intercepts `git-upload-pack` traffic, hashes the request body, converts the POST to a GET, and caches the response at edge POPs worldwide
- **OpenResty** (Nginx + LuaJIT) running as the origin server, with a Lua script that restores the original POST, checks a Valkey denylist for private repositories, and signals cacheability back to Fastly
- **Valkey + webhook** — a small [Valkey](https://valkey.io/) instance stores a denylist of private repository paths, kept in sync by a [webhook service](https://gitlab.gnome.org/GNOME/gitlab-git-cache-webhook) that listens for GitLab project visibility changes

<pre class="mermaid">
flowchart TD
    client["Git client / CI runner"]
    edge["Fastly Edge POP (nearest)"]
    shield["Fastly Shield POP (IAD)"]
    nginx["OpenResty Nginx (origin)"]
    lua["Lua: git_upload_pack.lua"]
    valkey["Valkey denylist"]
    gitlab["GitLab webservice"]
    webhook["gitlab-git-cache-webhook"]
    gitlab_events["GitLab project events"]

    client -- "POST /git-upload-pack" --> edge
    edge -- "HIT → serve from edge" --> client
    edge -- "MISS → forward to shield" --> shield
    shield -- "HIT → return to edge (edge caches)" --> edge
    shield -- "MISS → fetch from origin" --> nginx
    nginx --> lua
    lua -- "authenticated? check denylist" --> valkey
    lua -- "denied/error: keep auth, skip cache" --> gitlab
    lua -- "allowed: keep auth, signal cacheable" --> gitlab
    gitlab -- "packfile response" --> nginx
    nginx -- "X-Git-Cacheable: 1 (if allowed)" --> shield
    gitlab_events --> webhook
    webhook -- "SET/DEL git:deny:" --> valkey
</pre>

The request flow:

1. The `POST /git-upload-pack` arrives at the nearest Fastly edge POP.
2. VCL checks the body: if `Content-Length` exceeds 8 KB (the limit of what Fastly can read from `req.body`), or the body does not contain `command=fetch`, the request is passed through uncached.
3. VCL hashes the body with SHA256 to build the cache key, base64-encodes the body into `X-Git-Original-Body`, and converts the request to GET. If the request carries authentication headers (`Authorization`, `PRIVATE-TOKEN`, `Job-Token`), VCL sets `X-Git-Auth-Passthrough` to flag it — but the request still enters the cache lookup.
4. On a cache hit at the edge, the packfile is served immediately — regardless of whether the request is authenticated or not.
5. On a miss, the request routes to the IAD shield POP. If the shield has it cached, it returns the object and the edge caches it locally.
6. On a shield miss, the request reaches Nginx at the origin. Lua detects `X-Git-Original-Body` and restores the POST body. If `X-Git-Auth-Passthrough` is set, Lua checks the Valkey denylist: if the repo is private (or Valkey is unreachable), the `Authorization` header is preserved and cacheability is not signaled — the response passes through uncached. If the repo is not on the denylist, `Authorization` is preserved (internal repos need it for GitLab to return 200) and cacheability is signaled.
7. For unauthenticated requests (no passthrough flag), Lua strips `Authorization` and signals cacheability unconditionally — these are by definition accessing public repositories.
8. The response flows back through the shield and the edge. If `X-Git-Cacheable: 1` is present, both nodes cache the response. Subsequent requests — authenticated or not — for the same cache key are served directly from cache.

## The VCL layer

The `vcl_recv` snippet runs at priority 9, before the existing `enable_segmented_caching` snippet at priority 10 which would otherwise `return(pass)` for non-asset URLs:

```
# Snippet git-cache-vcl-recv : 9

# Edge: convert POST to GET, hash body, encode body in header
if (req.url ~ "/git-upload-pack$" && req.request == "POST") {
    if (std.atoi(req.http.Content-Length) > 8192) {
        return(pass);
    }

    if (req.body !~ "command=fetch") {
        return(pass);
    }

    set req.http.X-Git-Cache-Key = "v3:" digest.hash_sha256(req.body);
    set req.http.X-Git-Original-Body = digest.base64(req.body);

    # Flag authenticated requests — they still enter the cache lookup,
    # but on a miss Lua uses this to decide whether to cache the response
    if (req.http.Authorization || req.http.PRIVATE-TOKEN || req.http.Job-Token) {
        set req.http.X-Git-Auth-Passthrough = "1";
    }

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

Authenticated requests — CI runners with `Authorization: Basic <gitlab-ci-token:TOKEN>`, API clients with `PRIVATE-TOKEN` or `Job-Token` — are no longer sent straight to origin. Instead, VCL flags them with `X-Git-Auth-Passthrough` and lets them enter the cache lookup. On a cache hit, the packfile is served directly from the edge — no origin contact, no credential validation needed, because the cached object can only exist if a previous request already established that the repository is public (see [Protecting private repositories](#protecting-private-repositories)). On a cache miss, the flagged request reaches origin where Lua checks the Valkey denylist to decide whether the response should be cached.

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

Private repository traffic must never be cached — that would mean storing authenticated git content in a third-party cache and serving it to arbitrary clients. The protection relies on two independent layers.

**Layer 1: cache population is restricted.** The cache can only be populated when Lua signals cacheability via `X-Git-Cacheable: 1`. Lua only signals cacheability when the request is either unauthenticated (by definition accessing a public repo) or authenticated for a repo that is not on the Valkey denylist. For private repos, Lua does not signal cacheability, so `vcl_fetch` sets `ttl=0` and `cacheable=false` — the response is delivered but never stored.

**Layer 2: the Valkey denylist.** A [webhook service](https://gitlab.gnome.org/GNOME/gitlab-git-cache-webhook) listens for GitLab `project_create` and `project_update` system hooks. When a project's visibility is set to private (level `0`), the webhook sets a `git:deny:<path>` key in Valkey. When visibility changes to internal (level `10`) or public (level `20`), the key is removed. A periodic reconciliation job (`reconcile.py`) syncs the full denylist against the GitLab API to correct any drift from missed events.

On a cache miss for an authenticated request, Lua checks the denylist:

- **Repo is on the denylist (private):** `Authorization` is preserved, cacheability is not signaled. The request proxies to GitLab with credentials intact, GitLab validates the token, the response is returned but never cached.
- **Repo is not on the denylist (public/internal):** `Authorization` is preserved (internal repos require it for GitLab to return 200), cacheability is signaled. The response is cached for future requests.
- **Valkey is unreachable or returns an error:** treated the same as denied — `Authorization` is preserved, cacheability is not signaled. This fail-closed design means infrastructure failures result in cache misses, never in data leaks.

The denylist only needs to track private repositories, which are a small fraction of the total on GNOME's GitLab instance. A private repo's packfile can never enter the cache through two independent mechanisms: the denylist prevents Lua from signaling cacheability, and even if the denylist were somehow wrong, an unauthenticated request to a private repo returns a 401 from GitLab — which `vcl_fetch` does not cache (it only caches `200 + X-Git-Cacheable`).

## The Lua layer

With the VCL handling body hashing, the POST-to-GET conversion, and the cache lookup for all requests, the Lua script runs on cache misses that reach origin. Both authenticated and unauthenticated requests can arrive here. The script's responsibilities are:

1. Detect that the request arrived from Fastly with an encoded body (the `X-Git-Original-Body` header).
2. Decode and restore the original POST.
3. For authenticated requests, check the Valkey denylist to determine if the repository is private.
4. Signal back to Fastly whether the response is safe to cache.

```Lua
local redis_helper = require("redis_helper")

local redis_host = os.getenv("REDIS_HOST")
local redis_port = os.getenv("REDIS_PORT")

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

if ngx.req.get_headers()["X-Git-Auth-Passthrough"] then
    ngx.req.clear_header("X-Git-Auth-Passthrough")

    local uri = ngx.var.uri
    local repo_path = uri:match("^/(.+)/git%-upload%-pack$")
    if repo_path then
        repo_path = repo_path:gsub("%.git$", "")
    end

    local denied, err = redis_helper.is_denied(redis_host, redis_port, repo_path)

    if err then
        ngx.log(ngx.WARN, "git-cache: Redis error for ", repo_path, ": ", err,
                " — keeping auth, skipping cache")
    end

    if err or denied then
        return
    end

    ngx.ctx.git_cacheable = true
else
    ngx.req.clear_header("Authorization")
    ngx.ctx.git_cacheable = true
end
```

The two branches handle the authenticated and unauthenticated paths. When `X-Git-Auth-Passthrough` is present, the request came from a CI runner or API client. Lua checks the denylist: if the repo is private or Valkey is unreachable, the script returns early — `Authorization` stays on the request (so GitLab can validate it), and `git_cacheable` is never set (so the response is not cached). If the repo is not denied, `Authorization` is preserved and cacheability is signaled. The `Authorization` header is kept rather than stripped because internal repositories (visibility level `10`) require authentication for git operations — stripping it would cause GitLab to return a 401. Public repos work with or without credentials, so keeping the header is safe for both.

For unauthenticated requests (no passthrough flag), `Authorization` is stripped and cacheability is signaled unconditionally — these are by definition accessing public repositories.

The early `return` for denied or errored lookups is the fail-closed behavior. The request still proxies to GitLab (the `proxy_pass` directive in the Nginx location block runs after Lua), but without the cacheable signal, `vcl_fetch` will not store the response.

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

**Internal repositories require authentication for git operations.** An early version of the Lua script stripped `Authorization` for any repo not on the denylist, assuming that "not private" meant "accessible without credentials." Internal repositories (visibility level `10`) are not on the denylist — their content is not sensitive — but GitLab still requires authentication for git clone/fetch operations on them. Stripping credentials produced a 401 from GitLab. The fix was to preserve `Authorization` for all authenticated requests that pass the denylist check, regardless of whether the repo is public or internal. Public repos accept the header harmlessly; internal repos require it.

## How we got here

The current architecture is the result of two iterations. The sections above describe the final design; this section documents the path we took to get there.

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

This architecture worked, but it had two significant limitations that led to the current design.

### Iteration 2: Edge caching with CI runner participation

The first problem with the separate CDN service was geographic. Nginx runs in AWS us-east-1, so from Fastly's perspective the only client of the CDN was that single instance in Virginia. Every request entered through the IAD POP, which meant the CDN's edge POPs around the world were never populated. A CI runner in Europe would have its request travel from a European Fastly POP to IAD, then to Nginx, then back to Fastly IAD, and then all the way back — crossing the Atlantic twice for every cache miss.

The fix was to eliminate the separate CDN service and move all the caching logic into the `gitlab.gnome.org` Fastly service itself. The key insight was that the POST-to-GET conversion and body hashing could happen in Fastly's VCL rather than in Lua — Fastly provides `digest.hash_sha256()` and `digest.base64()` functions that operate directly on `req.body`. By doing the conversion at the CDN edge, every POP in the network became a potential cache node for git traffic.

The second problem was that the original denylist approach had two flaws. First, its error handling was fail-open: a Valkey connection error would cause the Lua script to assume the repo was public and strip credentials — the wrong default. Second, even after briefly replacing the denylist with a simple VCL auth bypass (`return(pass)` for any request with `Authorization`), CI runners were left completely uncached. GitLab CI always injects a `CI_JOB_TOKEN` into every job, and the runner authenticates with `Authorization: Basic <gitlab-ci-token:TOKEN>` regardless of whether the repository is public or private. With the auth bypass, every CI clone skipped the cache entirely — safe, but it left the biggest source of redundant traffic unserved.

The current design solves both problems. VCL flags authenticated requests with `X-Git-Auth-Passthrough` instead of bypassing the cache, letting them participate in cache lookups. On a hit, the cached packfile is served immediately. On a miss, the request reaches Lua at origin, where the flag triggers a denylist check against Valkey — the same denylist and webhook infrastructure from iteration 1, re-deployed with one critical change: fail-closed error handling. A Valkey error or missing connection causes Lua to preserve `Authorization` and skip cacheability signaling. The request still works (GitLab validates the token and serves the packfile), but the response is not cached. Infrastructure failures result in cache misses, never in data leaks.

The denylist only tracks private repositories (visibility level `0`), which are a small fraction of the total on GNOME's GitLab. Public and internal repositories pass the denylist check, and Lua signals cacheability while preserving the `Authorization` header — internal repos require it for GitLab to return 200, and public repos accept it harmlessly.

## Conclusions

The system has been running in production since April 2026 and has gone through two iterations to reach its current form. Packfiles are cached at Fastly edge POPs worldwide — a CI runner in Europe gets a cache hit served from a European POP rather than making a round trip to the US East coast. 

The moving parts are Fastly's VCL, an OpenResty Nginx instance with a ~30-line Lua script, a Valkey instance storing the private repository denylist, and a small webhook service that keeps the denylist synchronized with GitLab. Private repositories are protected by two independent layers: the Valkey denylist (which prevents cacheability signaling) and GitLab's own authentication (which rejects unauthenticated access).

If something goes wrong with the cache layer, requests fall through to GitLab directly — the same path they took before caching existed. There is no failure mode where caching breaks git operations. This also means we don't redirect any traffic to github.com anymore.

That should be all for today, stay tuned!

<script type="module">
import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
mermaid.initialize({ startOnLoad: true, theme: 'neutral' });
</script>
