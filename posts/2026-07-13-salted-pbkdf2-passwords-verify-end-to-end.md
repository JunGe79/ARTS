# How Login Passwords Are Salted and Hashed — and How to Verify It End-to-End

**Question:** A lot of enterprise apps store login passwords the same way. What is that common scheme, and how do you actually check that it works — both from the web front end and from the database back end?

**Answer:** A very common pattern is a **salted PBKDF2-HMAC-SHA256 hash** with a short version tag in front. Each user gets a random salt; the password is run through PBKDF2 (many rounds of HMAC-SHA256) with that salt; the result is stored as text. Because it's a standard algorithm, you can reproduce it in a few lines of code and verify it two ways: recompute the hash on the back end and compare it to what's stored, or round-trip a real login through the front-end API and confirm you get a token.

This post explains the scheme in plain English and shows both verification methods.

## What gets stored

For each user, the database keeps two pieces of text:

| Column | What it holds |
|---|---|
| `salt` | a random value, unique per user (often 64 hex characters) |
| `password` | the hash — a short **version tag** followed by the hex of the hash |

Two important things:

- The **salt is not secret**. It sits right next to the hash. Its only job is to make sure two people who pick the same password don't end up with the same stored hash.
- The stored `password` is a **one-way hash**, not encryption. You can check a guess against it, but you can't turn it back into the original password.

A stored value might look like `7a1b2c...` — a single leading character (`7`) that says "this is version 7 of our hashing scheme," followed by 64 hex characters (32 bytes) of hash.

## The algorithm, step by step

When a password is set:

1. **Make a salt.** Generate 32 random bytes and write them as a 64-character hex string. Store that string in the `salt` column.

2. **Run PBKDF2.** PBKDF2 is a deliberately *slow* function. Slowness is the point — it makes brute-force guessing expensive. It takes:
   - the **password** (its raw UTF-8 bytes) as the HMAC key,
   - the **salt** as the data,
   - a fixed **iteration count** (1000 is common),
   - and a fixed **output size** (32 bytes).

   Internally it does 1000 rounds of HMAC-SHA256, each round feeding the previous result back in, and XORs them together into the final 32 bytes. That's exactly the textbook PBKDF2 recipe (RFC 8018 / NIST SP 800-132).

3. **Store it.** Write the version tag plus the hex of those 32 bytes into the `password` column.

To check a login later: read the user's salt, run the exact same PBKDF2 on the password the user typed, and compare the result to the stored hash. Equal means correct.

### One subtle detail that trips people up

The salt is stored as a 64-character **hex string**. When it's fed into PBKDF2, many implementations use the **literal characters of that string** (64 ASCII bytes), *not* the 32 raw bytes you'd get by decoding the hex. If you reproduce the hash and decode the salt "helpfully" back to bytes first, you'll get a different, wrong answer. Feed the salt in exactly as it's stored — as text.

## Reproduce it in code

The whole thing is one line with a standard library:

```python
import hashlib

def make_hash(password: str, salt: str) -> str:
    # salt is used as its literal text bytes, NOT hex-decoded
    digest = hashlib.pbkdf2_hmac("sha256", password.encode("utf-8"),
                                 salt.encode("ascii"), 1000, 32).hex()
    return "7" + digest   # "7" = the version tag used by this scheme
```

That's it. No custom crypto — the standard `pbkdf2_hmac` does all 1000 rounds and the XOR-combine for you.

## Verify it on the back end

The cleanest proof is to **recompute a known account's hash and compare it to what's stored.** Pick a user whose password you already know:

```python
salt   = "e6c1e6...b3fe"          # from the user's salt column
stored = "70edc7...b124"          # from the user's password column
print(make_hash("KnownPassword1", salt) == stored)   # -> True
```

If it prints `True`, your understanding of the algorithm is correct — byte for byte. This is exactly what the login code does internally, so a match here means a match at login time.

Once you trust the recipe, you can also **set a password with a single update** (handy for test setups), because you control both inputs:

```sql
-- compute make_hash(newpassword, <that user's salt>) first, then:
UPDATE Users
SET password = '7b9ec1...7764'   -- the computed hash
WHERE id = 70;
```

Keep the existing salt (the hash was computed against it) and keep the version tag — some schemes enforce it with a trigger and reject a value that doesn't start with the right character.

## Verify it from the web front end

The back-end check proves the math. The **front-end check proves the whole stack** — transport, decoding, and comparison all wired together. Just call the login API and see if you get a session token:

```bash
curl -sk -X POST https://app.example.com/api/login \
  -H 'Content-Type: application/json' \
  --data '{"username":"alice","password":"S25vd25QYXNzd29yZDE="}'
# 200 + a token  -> success
# 401 / "incorrect" -> the hash or salt doesn't line up
```

A couple of practical notes:

- Many APIs expect the password **base64-encoded** in the JSON (that `S25v...` above), not in the clear. The server decodes it, then hashes.
- If the username contains a domain-style prefix like `domain\alice`, remember JSON needs the backslash escaped (`domain\\alice`). The easiest way to dodge shell-and-JSON quoting headaches is to put the body in a file and send `--data @login.json`.
- Watch the response, not just the status line. A token means the salted hash matched. An "incorrect" message means the stored hash and the typed password don't agree under the algorithm above — usually a salt-encoding or version-tag mistake.

## Why it's built this way

- **Salt per user** defeats precomputed "rainbow table" attacks and hides the fact that two users share a password.
- **Slow KDF (1000+ iterations)** makes each guess costly, so leaked hashes are far harder to crack than a plain SHA-256 would be.
- **A version tag** lets the app upgrade the scheme later (more iterations, a different function) without breaking old accounts — the tag tells the verifier which recipe to use.
- **One-way by design** means even the app can't recover a forgotten password. It can only overwrite it.

## Takeaways

- The common scheme is **`versiontag + hex(PBKDF2-HMAC-SHA256(password, salt, 1000, 32))`**, with a random per-user salt stored alongside.
- Feed the salt into PBKDF2 **as its stored text**, not as decoded bytes — the most common reproduction bug.
- **Verify on the back end** by recomputing a known account's hash and comparing — if it matches, you've got the algorithm exactly right.
- **Verify on the front end** by round-tripping a real login through the API and checking for a token — that tests transport, decode, and compare together.
- It's a standard, one-way, deliberately-slow hash — you can check and overwrite passwords, but never read them back.
