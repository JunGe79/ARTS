# A "Valid" Certificate That Still Broke Our Uploads

**Question:** A server-to-server HTTPS upload kept failing with a TLS
error, but when I opened the same site in a browser the certificate
looked perfectly valid — trusted issuer, not expired. How can the cert
be "valid" and still fail?

**Answer:** A certificate being *trusted and unexpired* is not the same
as it being *valid for the name you dialed*. The TLS client also checks
that the hostname you connected to is listed in the certificate's **SAN**
(Subject Alternative Name). Our cert was a wildcard that did not cover the
hostname, so every client correctly rejected it — even though the cert
itself was genuine.

## The error

Two services (one Java, one .NET) both talked to the same host. Both
failed at the same place:

```
java.security.cert.CertificateException:
  No subject alternative DNS name matching node-102.api.example.com found
```

The .NET side showed the same root cause in a different shape: the HTTP
call threw during the TLS handshake, returned no response, and the upload
was logged as failed.

## What a certificate actually promises

A certificate has two name-bearing fields:

- **Subject CN (Common Name)** — e.g. `CN=*.example.com`. This is a
  human-readable label. Modern clients (browsers, Java, .NET) **do not**
  use the CN for hostname matching anymore.
- **SAN (Subject Alternative Name)** — the real list of hostnames the
  cert is valid for. This is the **only** field clients check.

Our cert:

```
Subject : CN=*.example.com
SAN     : *.example.com, example.com   <-- only these two names
Issuer  : (a public CA, trusted)
Validity: not expired
```

So the chain was trusted and the dates were fine. The problem was the SAN
list.

## A wildcard matches only ONE label

`*.example.com` replaces exactly **one** label (one piece between dots).
It does not cross a dot.

| Hostname you connect to        | Covered by `*.example.com`? | Why |
|--------------------------------|-----------------------------|-----|
| `example.com`                  | yes                         | exact match (2nd SAN entry) |
| `www.example.com`              | yes                         | `*` = `www` (one label) |
| `api.example.com`              | yes                         | `*` = `api` (one label) |
| `node-102.api.example.com`     | **no**                      | two labels (`node-102` + `api`) |

The host we actually connected to — `node-102.api.example.com` — sits
**two** labels below `example.com`. A single-level wildcard cannot reach
it. So the client checks the SAN, finds no match, and aborts the
handshake. That is exactly what the error message says.

## Why "looks valid in a browser" fooled us

Two reasons:

1. A browser shows the cert as valid based on **issuer trust + expiry**.
   Those were both fine. The hostname check only fails for the specific
   name that isn't covered — if you happened to open the cert by a name
   the wildcard *does* cover, you'd see no warning at all.
2. "Trusted and unexpired" feels like "valid," but TLS validity has three
   independent parts: **(a)** chain trust, **(b)** validity dates, and
   **(c)** the hostname is in the SAN. All three must pass. We had (a) and
   (b); we failed (c).

## How to check the cert yourself

Pull the actual certificate the server is serving and read its SAN. You
do not need to trust it to read it.

Linux / macOS:

```bash
echo | openssl s_client -connect node-102.api.example.com:443 \
       -servername node-102.api.example.com 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName -dates
```

Windows (PowerShell), reading the cert bound to a local IIS site:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
  Select-Object Subject, NotAfter,
    @{ n = 'SAN'; e = { $_.DnsNameList -join ', ' } }
```

Look at the **SAN** line, not the Subject. If the hostname you connect to
isn't in that list, that's your bug.

> Note: a chain-validity check (such as .NET's `X509Certificate2.Verify()`)
> returns **true** here, because it only tests trust + expiry + revocation.
> It does **not** test the hostname. Hostname matching is a separate step
> the TLS client does at connection time. This is why a "the cert verifies
> fine" check can lull you into looking in the wrong place.

## The fix

Two options:

1. **Fix the certificate (correct fix).** Issue a cert whose SAN includes
   the real hostname — either the exact name `node-102.api.example.com`,
   or a wildcard at the right level, `*.api.example.com`. Install it and
   re-bind the HTTPS port to the new cert. (Installing the new cert is not
   enough on its own — the server keeps using the old binding until you
   point the port at the new thumbprint.)
2. **Change the name you connect by (interim).** If reissuing a cert is
   slow, point the client at a hostname the current cert already covers
   (a single-label name directly under `example.com`) and make DNS resolve
   it to the same server. This sidesteps the mismatch without a new cert,
   but the clean fix is still the correct SAN.

## Takeaways

- "Valid certificate" has three separate conditions: trusted chain, good
  dates, **and** the hostname is in the SAN. Don't stop at the first two.
- Clients match on **SAN**, not on the Subject CN. Read the SAN.
- A wildcard covers **one** label only. `*.example.com` does not cover
  `a.b.example.com`. For deeper names you need a deeper wildcard
  (`*.b.example.com`) or the explicit hostname.
- When a TLS call fails but the cert "looks fine," fetch the served cert
  and compare its SAN against the exact name your client dialed.
