# Rotating a shared HMAC key without downtime: keep two keys, not one

**Question:** Many unattended clients authenticate to a server by signing each request with an HMAC over a shared secret key. How do you rotate that key without breaking anyone — and why isn't "one key plus an expiry date" good enough?

**Answer:** Make the server accept a small *set* of keys at once — a current one and a next one — instead of a single key. The overlap lets every client switch on its own schedule, so you never need a synchronized, everyone-at-once cutover. Rotation becomes "drop the oldest key and mint a new next key," and there are always two valid keys, so there's never a moment when a client is locked out.

## The setup

A server exposes an endpoint that unattended clients (jobs, agents, scheduled scripts — no human to type a password) call. Each request is authenticated with:

```
auth = HMAC_SHA256(key, f"{method}\n{url}\n{body}\n{nonce}\n{timestamp}")
```

The server holds the same `key`, recomputes the HMAC, and compares in constant time. One shared secret, many clients.

Now you need to change that key — it leaked, it's old, or policy says rotate every N days. If the server knows exactly one key, the moment you change it every client using the old key starts failing. You'd have to update the server and every client at the same instant. That's a coordinated outage.

## Why "one key + an expiry date" is a trap

A natural instinct is to give the single key an expiry date and enforce it: after the date, reject it. This is worse, not better:

- It doesn't fix the real risk. A leaked key is fully usable right up until the date.
- It turns the expiry into a **scheduled outage**. The instant the date passes, every client using that key gets rejected — unless you've already swapped the server and all clients over, with zero overlap window.

Enforced expiry only makes rotation safe if two keys can be valid at the same time. With one key, a hard expiry is a countdown to breakage.

## The fix: the server accepts a set of keys

Store the key as a short comma-separated list and accept a request if it matches **any** entry:

```python
def verify(req, keys):          # keys = [current, next]
    plain = signed_string(req)
    for i, k in enumerate(keys):
        expected = hmac.new(k.encode(), plain.encode(), "sha256").hexdigest()
        if secrets.compare_digest(expected, req.auth):
            return True, i      # which key matched — useful below
    return False, None
```

Config holds the set:

```
UPLOAD_KEYS = Kcurrent,Knext
```

The invariant: **the server always accepts two keys** — one active (what almost everyone uses) and one standby (pre-seeded, ready). It never shrinks to one. That overlap is the whole point.

## The rotation lifecycle

Start in steady state: server = `[K1, K2]`, everyone signs with `K1`, `K2` is the standby.

1. **Distribute `K2`** to the clients. Each one overwrites its stored key `K1 → K2` whenever convenient — the server accepts both, so there's no timing pressure and nothing breaks mid-switch.
2. **Wait for drain.** Watch until no request still uses `K1`.
3. **Rotate the set** in one config change: `[K1, K2] → [K2, K3]`. This drops the old `K1` (now globally invalid) *and* mints a fresh standby `K3`, in a single step.

You're back in steady state — everyone on `K2`, `K3` standby. Repeat for the next rotation. The set never collapses to one key, so you're never a single leak or a single expiry away from an outage.

## When is it safe to drop the old key?

Step 3 is only safe once no one is using the key you're about to remove. Two ways to know:

- **Signal-based (preferred):** have `verify()` report *which* key matched (the index it returned above). Drop `K1` only when its match count hits zero. Data-driven; you drop with confidence.
- **Time-based:** give each key its own expiry date. The old key self-drops on its date — but you must have distributed the next key and migrated everyone *before* then. Simpler, but a slow client breaks at the deadline.

Note the difference from the trap earlier: an expiry date is fine here *because there's overlap*. The date drops the **older of two** valid keys, not the **only** key.

## The client stays dumb

Clients don't manage expiry or track a schedule. Each one just holds a single key and signs with it. To rotate, an operator overwrites that stored value with the new key — that overwrite *is* the client-side "switch," no timer needed. If a client somehow misses the window and its key gets retired, it simply starts getting `401`/`403`, and the fix is "push the current key to it." Validity lives entirely on the server.

## You already use this — you just don't see it

Overlapping keys during rotation is the standard pattern; platforms hide it behind two named keys or a rotate button:

| System | The overlap |
|---|---|
| Azure Storage accounts | `key1` + `key2` — two keys exist *specifically* so you rotate one while the other stays live |
| AWS IAM | two active access keys per user, for exactly this |
| JWT / JWKS | verifiers accept multiple signing keys (`kid`) during rollover |
| Django / Rails | a *list* of secrets — the current one signs, all of them verify (`SECRET_KEY_FALLBACKS`) |

So "multiple keys" isn't exotic. It's usually presented as *two named keys*, not a raw list, but it's the same idea.

## Two anti-patterns to avoid

- **Single key + enforced expiry.** Combines the pain of rotation (coordinated cutover on a deadline) with none of the safety (no overlap). It's the one combination that can't rotate without downtime.
- **Using an identifier as the credential.** If a follow-up call is "authorized" just by knowing a resource id (a task id, a request id), remember that ids are *identifiers*, not *secrets* — they ride in URLs and land in access logs, so anyone who sees a log line can act. Authorize follow-up actions with a real secret (a per-request token in the body), not with the id in the path.

## Takeaways

- Keep **two** keys valid at once (current + next). Overlap is what makes rotation downtime-free.
- Rotation = **drop the oldest, mint a new next** — one atomic config change, never "shrink to one."
- Prefer **signal-based** retirement (drop a key only when its usage hits zero) over a blind timer.
- An expiry date is only safe *with* overlap; a hard expiry on a lone key is a scheduled outage.
- Keep clients dumb: they hold one key, get a new one pushed, and 401 if stale. The server owns validity.
