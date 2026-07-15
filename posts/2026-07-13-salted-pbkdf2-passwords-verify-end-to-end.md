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
