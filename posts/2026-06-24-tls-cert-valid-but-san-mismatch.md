# A "Valid" Certificate That Still Broke Our Uploads

**Question:** A server-to-server HTTPS upload kept failing with a TLS
error, but the browser said the cert was valid. How?

**Answer:** "Valid" has three parts: trusted chain, good dates, **and**
the hostname is listed in the cert's **SAN**. Our wildcard didn't cover
the hostname, so clients correctly rejected it.

## The error

```
java.security.cert.CertificateException:
  No subject alternative DNS name matching node-102.api.example.com found
```

## The cert

```
Subject : CN=*.example.com
SAN     : *.example.com, example.com
```

Trusted, not expired — but `node-102.api.example.com` is not in the SAN.

## Wildcards cover ONE label only

`*.example.com` replaces exactly one label between dots.

| Hostname                   | Covered? |
|----------------------------|----------|
| `api.example.com`          | yes      |
| `node-102.api.example.com` | **no** — two labels |

Clients match against the SAN, not the Subject CN. CN matching has been
dead for years.

## Check the served cert

```bash
echo | openssl s_client -connect node-102.api.example.com:443 \
       -servername node-102.api.example.com 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName -dates
```

A chain check like .NET's `X509Certificate2.Verify()` returns **true**
here — it tests trust + expiry, not the hostname. Hostname matching is
a separate TLS step. That's why "the cert verifies fine" misleads you.

## Fix

Reissue the cert with the real hostname in the SAN — either the explicit
name or a deeper wildcard like `*.api.example.com`. Re-bind :443 to the
new thumbprint; installing the cert alone isn't enough.

## Takeaways

- Three independent conditions: chain, dates, **SAN matches hostname**.
- Read the SAN, not the Subject CN.
- One wildcard = one label. `*.example.com` does not cover
  `a.b.example.com`.
- TLS-call fails but cert "looks fine"? Fetch the served cert and diff
  its SAN against the exact name you dialed.
