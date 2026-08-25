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
