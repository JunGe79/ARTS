# Session Cookie Missing 'Secure' — a One-Line Tomcat Fix, and Why It Wasn't the Real Fix

**Question:** An external security scanner flagged one of our web nodes:
`JSESSIONID` is set without the `Secure` attribute. A sibling node running the
same app was clean. What is actually wrong, and how do you close it properly?

**Answer:** The flagged node still served the app over **plain HTTP on port 80**.
Tomcat only adds `Secure` to the session cookie when the request arrives over
TLS, so every HTTP hit minted an insecure cookie. Forcing `<secure>true</secure>`
in `web.xml` closes the scanner finding in one line — but the real fix is to stop
serving the app over plaintext at all (close or redirect port 80), because
`Secure` is a browser-side rule, not server access control.

## The finding

The scanner reported, for `http://logs.example.com/`:

```
JSESSIONID=AAAAAAAA...AAAA; Path=/; HttpOnly
```

`HttpOnly` is there; **`Secure` is missing**. Same finding on a second URL that,
on closer look, resolved to the *same* machine (one name was a CNAME of the
other). A twin node serving the identical app was **not** flagged.

## Reproduce it — the whole diagnosis is two curls

The cleanest tell is the same request over each scheme:

```bash
curl -sI http://logs.example.com/    # JSESSIONID=...; Path=/; HttpOnly          <- no Secure
curl -sIk https://logs.example.com/  # JSESSIONID=...; Path=/; Secure; HttpOnly  <- correct
```

Same server, same app, only the scheme differs — and only the difference matters
because **port 80 is open**. The clean twin node had **no port-80 listener** at
all (HTTPS only), which is exactly why it was never flagged.

## Root cause

Two config facts on the flagged node's Tomcat:

1. `server.xml` had a plain-HTTP connector:
   ```xml
   <Connector port="80" protocol="HTTP/1.1" redirectPort="443" .../>
   ```
   `redirectPort="443"` looks like a redirect, but it isn't one — Tomcat only
   uses it when a resource is marked `CONFIDENTIAL` via a `<security-constraint>`.
   With no such constraint, port 80 just serves the app in the clear.

2. `web.xml` forced nothing:
   ```xml
   <session-config>
     <session-timeout>30</session-timeout>
   </session-config>
   ```
   No `<cookie-config>`, so `JSESSIONID` gets `Secure` **only** from Tomcat's
   default `request.isSecure()` — i.e. only over the HTTPS connector. Over HTTP,
   no `Secure`.

On the box, an `HttpWebRequest` to `http://127.0.0.1/` with redirects disabled
confirmed it: `Status: 200`, empty `Location`, and the insecure cookie. So HTTP
was serving real content, **not** redirecting.

## Fix 1 — force `Secure` in web.xml (closes the finding)

```xml
<session-config>
  <session-timeout>30</session-timeout>
  <cookie-config>
    <http-only>true</http-only>
    <secure>true</secure>
  </cookie-config>
</session-config>
```

Restart Tomcat, and the HTTP response now carries `JSESSIONID=...; Secure;
HttpOnly`. The scanner is happy.

**Schema trap that will fail startup:** the flags must be **inside**
`<cookie-config>`, and inside `<session-config>` the order is fixed —
`<session-timeout>` first, then `<cookie-config>`. Putting `<secure>` directly
under `<session-config>` is invalid and Tomcat refuses to start. Validate before
you bounce the service (`[xml](Get-Content web.xml)` in PowerShell just checks
well-formedness; element order still has to be right for the servlet schema).

## Why `Secure` alone is not the real fix

`Secure` is enforced by the **client**, not the server. A browser won't send a
`Secure` cookie over HTTP — good, that's the actual threat the scanner cares
about. But a non-browser client will happily do it:

```bash
curl -H "Cookie: JSESSIONID=<stolen-id>" http://logs.example.com/
```

The server reads the cookie and honors the session — it never rejects an
"insecure" cookie, because `Secure` was only ever an instruction to browsers.
So Fix 1 protects real users and closes the scorecard item, but the app is still
reachable, and replayable, over plaintext.

## Fix 2 — close the plaintext port (the actual hardening)

Match the clean twin node: comment out the port-80 connector in `server.xml`.

```xml
<!-- port 80 disabled -->
<!--<Connector port="80" protocol="HTTP/1.1" redirectPort="443" .../>-->
```

Restart Tomcat. Now `http://` is refused, `https://` is untouched:

```
curl -sI  http://logs.example.com/    -> Connection refused
curl -sIk https://logs.example.com/   -> 200, JSESSIONID=...; Secure; HttpOnly
```

No plaintext endpoint means no insecure cookie to leak and nothing to replay a
stolen cookie against. (If real clients still use `http://`, redirect port 80 to
443 instead of closing it — but check first; a bare `redirectPort` won't do that
on its own.)

## Gotchas that cost me time

- **`redirectPort` is not a redirect.** It only kicks in under a `CONFIDENTIAL`
  transport-guarantee. On its own, port 80 serves content.
- **Same-box, two findings.** A CNAME meant "two" scanner entries were one
  machine. Resolve names before you go chasing two servers.
- **Edit XML as UTF-8 with no BOM.** Writing the file back with a BOM can upset
  parsers. In PowerShell: `[IO.File]::WriteAllText($p,$s,(New-Object
  Text.UTF8Encoding($false)))`.
- **RDP paste silently mangles long PowerShell lines.** A wrapped
  `$(Get-Date -f 'yyyy-MM-dd')` or a trailing `# comment` becomes a broken
  statement, yet a later validation can still print "OK". After any scripted
  edit, diff against a backup (`Compare-Object`) to prove only the intended
  lines changed.
- **Config can regress on app upgrades.** If the platform regenerates
  `server.xml`/`web.xml`, port 80 or the cookie flag can come back. Note it so
  it gets re-applied.

## Takeaways

- Missing-`Secure`-on-`JSESSIONID` almost always means the app answers on plain
  HTTP; Tomcat only marks the cookie `Secure` over TLS.
- `<secure>true</secure>` in `web.xml` is the one-line finding-closer. It's real
  protection for browsers, but it is **not** server-side access control.
- The durable fix is to not serve the app over plaintext — close port 80 or make
  it a real redirect to 443 (and add HSTS).
- Use a clean sibling node as your reference for "what compliant looks like."
- Diff every scripted config edit against a backup; "it validated" is not "it did
  what I meant."
