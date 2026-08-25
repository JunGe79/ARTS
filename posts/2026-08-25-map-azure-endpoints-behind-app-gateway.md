# Mapping an Azure Function and Blob Storage behind one hostname on Application Gateway

**Question:** I have an Azure Function App and an Azure Blob Storage account, each on its own hostname (`*.azurewebsites.net` and `*.blob.core.windows.net`). My clients — or their firewalls — only allow one host: the site that's already fronted by our Azure Application Gateway. Can I serve both the function and the blob under that single host, and what breaks when I try?

**Answer:** Yes. Add two path-based routes on the Application Gateway — for example `/svc/api/*` to the function and `/svc/blob/*` to the storage account — each forcing the backend's own `Host` header on the origin. The blob leg is easy, because a SAS is signed over the request *path*, not the host, so you just have to preserve the query string. The function leg is the one that bites: if your app authenticates by signing the full request URL, rewriting the host and path at the gateway makes that signature fail. You fix it by having the gateway forward the original host and path in headers, and reconstructing the signed URL from those.

## First, is the host behind App Gateway or Front Door?

They're configured very differently, so confirm which one you have before designing anything. A two-command probe tells you:

```
$ dig +short portal.example.com
20.96.6.235                 # a direct, regional A record  -> Application Gateway
                            # a CNAME to *.azurefd.net       -> Front Door

$ curl -sSI https://portal.example.com/
set-cookie: ...ApplicationGatewayAffinity=...   # App Gateway session-affinity cookie
                                                # (Front Door adds an X-Azure-Ref header instead)
```

An `ApplicationGatewayAffinity` / `ApplicationGatewayAffinityCORS` cookie plus a plain regional A record means **Application Gateway**. A CNAME to `*.azurefd.net` and an `X-Azure-Ref` response header mean **Front Door**. The rest of this post is Application Gateway; Front Door does the same job with rule sets and origin groups.

## The goal: one host, two paths

```
https://portal.example.com/svc/api/*   ->  myfunc.azurewebsites.net        (Function App)
https://portal.example.com/svc/blob/*  ->  mystorage.blob.core.windows.net  (Blob)
```

Everything else on `portal.example.com` keeps routing to the existing backend, untouched.

## The App Gateway pieces

**Backend pools** — one per origin: the function FQDN and the blob FQDN.

**Backend HTTP settings: override the Host header.** This is the number-one thing people miss. Both Azure Functions and Azure Storage route and validate by the `Host` header. If the gateway forwards the incoming host (`portal.example.com`), the origin returns **400** because it doesn't own that name. Set "Override with new host name" to the backend's own FQDN.

**Health probes: match the origin's real status.** Default probes expect 200-399, but:

- Blob storage returns **400** at the account root (`/`).
- A POST-only function route returns **405** to the probe's GET.

So widen the probe's acceptable-status match to include those, or the pool goes Unhealthy and every request 502s.

**Path map + URL rewrite.** Route `/svc/api/*` and `/svc/blob/*` to the two pools, and rewrite the path to strip the prefix so the origin sees what it expects:

```
/svc/api/upload             ->  /api/upload
/svc/blob/<container>/<blob> ->  /<container>/<blob>
```

In App Gateway that's a rewrite condition on the `uri_path` server variable with a capture group, e.g. match `^/svc(/api/.*)$` and rewrite the path to `{var_uri_path_1}`.

## Why the blob leg is easy: a SAS signs the path, not the host

An Azure Storage shared access signature is computed over a *canonicalized resource* — `/blob/<account>/<container>/<blob>` — derived from the account name and path. It does **not** include the `Host` header. So a SAS minted for `mystorage` stays valid whether the request arrives as `mystorage.blob.core.windows.net/...` or `portal.example.com/svc/blob/...`, as long as:

- the gateway forwards to the `mystorage` origin with the Host override above, and
- the rewrite leaves the path as exactly `/<container>/<blob>` and **preserves the `?sig=...` query verbatim**.

Change the path or drop a single query character and you get `403 AuthenticationFailed`. Otherwise, no code change at all.

## Why the function leg is hard: signing the full URL

Many APIs authenticate a request by HMAC-signing a string that includes the full URL:

```
plain = method + "\n" + request_url + "\n" + body + "\n" + nonce + "\n" + timestamp
auth  = HMAC_SHA256(secret, plain)
```

The client signs the URL it *called* (`portal.example.com/svc/api/upload`); the server rebuilds the string from the URL it *received*. Once the gateway rewrites host and path, the server sees `myfunc.azurewebsites.net/api/upload` — a different string — and every request fails the signature check.

Two ways to fix it:

**Option A — reconstruct the client URL on the server (least disruptive).** Have the gateway inject the original host and path as headers *before* the rewrite:

```
X-Forwarded-Proto: https
X-Forwarded-Host:  portal.example.com
X-Original-Uri:    /svc/api/upload
```

Then change the server's verify routine to rebuild the signed URL from those headers when they're present, falling back to the received URL when they're not. The client signs the public URL; both sides now agree.

**Option B — sign only the path and query, not the scheme and host.** Cleaner and proxy-agnostic forever, but it's a protocol change the client and server have to ship together.

Either way, the lesson is the same: **a reverse proxy is invisible to TLS but not to anything that signs the URL.** If your auth binds to the host or path, the proxy has to hand the original back.

## If a WAF sits in front

Two things to check for the upload path:

- **Body size / file-upload limit.** Large uploads — even chunked block uploads — can hit the WAF's max request-body or file-upload cap. Raise it, or exclude the upload path from body inspection.
- **The SAS query string** contains `sig=`, base64 and `=` characters that SQL-injection or base64 managed rules sometimes flag. Add a path exclusion for the blob route.

## Takeaways

- Confirm the fronting product first: an `ApplicationGatewayAffinity` cookie means App Gateway; an `X-Azure-Ref` header or a `*.azurefd.net` CNAME means Front Door.
- Always override the Host header to the backend FQDN — Storage and Functions route by Host and return 400 otherwise.
- Match the health probe to the origin's real status (blob root returns 400, a POST-only route returns 405) or the pool 502s.
- A blob SAS is signed over the path, not the host, so you can front it freely — just preserve the query string exactly.
- Anything that signs the full URL (HMAC-style auth) breaks behind a rewriting proxy. Forward the original host and path and reconstruct, or sign only the relative path.
- Behind a WAF, exempt the upload path's body and its SAS query.
