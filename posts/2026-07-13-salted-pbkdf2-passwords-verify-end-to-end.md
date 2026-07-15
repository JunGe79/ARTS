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

## Transport encryption vs. the hash

Don't confuse the login transport encryption with the hash. When a user logs in, the client (or web server) wraps the typed password in a reversible cipher for transit. Server-side, the validation code decrypts it back to the plaintext and only then runs PBKDF2. So the value that actually goes through PBKDF2 is always the raw plaintext password — the reversible transport layer is transparent and doesn't affect the hash.
