# Test the real function, not a copy of it

**Question:** How do you write a unit test for a C++ function that is (a) a *private* class method and (b) does real I/O near the top — a database fetch, a cloud call — when the test has to run on a build server with no database and no cloud?

**Answer:** Don't copy the function's logic into your test. Compile a tiny test shim *into the library itself*, guarded by a build-time macro so it never ships. The shim calls the **real** function. A separate driver program `dlopen`s the freshly built library and calls the shim. To reach a private, I/O-bound function, add two build-time-only edits: a guarded public accessor, and a hook that swaps the DB fetch for a value you feed in as text. Then prove it is a real test: run it on code **without** the fix (it must FAIL) and **with** the fix (it must PASS).

## The trap: a test that always passes

The obvious way to test a hard-to-reach function is to copy its body into the test file and run the copy. This feels fine and it is a lie.

If you copy the logic, your test exercises *your copy*, not the shipped code. The day someone breaks the real function, your test stays green, because your copy still does the right thing. A test that cannot fail when the product breaks is worse than no test — it gives false confidence.

So the rule is simple and non-negotiable: **the test must call the real, compiled function.** Everything below is about how to reach it.

## Why the function is hard to reach

Two things get in the way:

1. **It's a private method.** You can't `dlsym` a mangled C++ member and call it — it needs a `this` and fully-built argument objects. A free function in your test can't touch a `private:` member either.
2. **It does I/O near the top.** The first thing it does is fetch its input from a database. On a build server there is no database, so the function bails before it reaches the logic you care about.

A plain "link the library and call it" approach dies on both counts.

## The shape: shim inside the library + driver outside

Each test case has two pieces.

**The harness** — a small shim compiled *into* the library, behind a macro:

```cpp
#ifdef LIB_UT
extern "C" int cvtest_build_config(const char* input_xml,
                                   char* out, int outlen)
{
    // build the arguments the real function needs, then CALL IT
    // (not a copy of it), and report what it produced.
}
#endif
```

Because the shim lives inside the library, it can see the library's private members and reach the real function. It exposes a plain-C name (`cvtest_build_config`) that a driver can find with `dlsym`.

**The driver** — a standalone program that loads the freshly built library and calls the shim:

```cpp
void* h = dlopen("./libfoo.so", RTLD_NOW | RTLD_GLOBAL);
auto fn = (int(*)(const char*, char*, int))dlsym(h, "cvtest_build_config");
int n = fn(test_input_xml, buf, sizeof buf);
// assert on n / buf, print PASS or FAIL
```

The driver `dlopen`s the exact `.so` you just built from its output folder, so you are always testing this build, not some installed copy.

## Reaching a private, I/O-bound function

Two more build-time-only edits get you past the two blockers. Both are staged by the build script and reverted after the run — they never live in the committed source.

**1. A guarded public accessor** so the shim can call the private method:

```cpp
// in the class header, build-time only
#ifdef LIB_UT
public:
    int cvtestBuildConfig(const char* xml, char* out, int len);
#endif
```

A member method *can* call a private sibling, so the shim goes through this accessor.

**2. A DB-bypass hook** so the function stops fetching from a database and takes your input instead. Find the fetch and, under the macro, feed the test XML in its place:

```cpp
#ifdef LIB_UT
    if (g_test_input) {                 // set by the shim
        config.unserialize(g_test_input);
        return SUCCESS;                 // skip the real DB fetch
    }
#endif
```

Now the real function runs, top to bottom, with an input you control — no database, no cloud.

## Release safety: it must never ship

The whole thing hangs on one rule: the macro is added **only at build time**, never committed.

```
# the build script appends this to the compile flags, then reverts
CFLAGS += -DLIB_UT
```

A normal build never defines `LIB_UT`, so every `#ifdef LIB_UT` block compiles to nothing. Verify it with one command:

```bash
nm -D libfoo.so | grep cvtest_    # release build: zero lines
```

Zero exported `cvtest_` symbols means the shim is gone from the shipped binary. This is why the test code can live in the repo without any risk to production — the guarded blocks are inert unless someone builds with the flag on, and the flag is only ever on during a test run.

## The part that makes it a real test

Here is the step people skip. A test framework is only a *regression guard* if it fails when the product is broken. So prove it, both directions, on the same build tree:

| Tree state | Result |
|---|---|
| Product code **without** the fix | **FAIL** |
| Same tree **with** the fix applied | **PASS** |

Concretely, the function under test was supposed to add two default tags to every node it configures. On a tree without that change:

```
TC1 with user tags  -> count=2   FAIL (expected 4: user tags + 2 defaults)
TC2 no user tags    -> count=0   FAIL (expected 2 defaults)
==== OVERALL FAIL ====
```

Apply the one-block fix to the real source and rebuild — nothing else changes — and the *same* test flips:

```
TC1 with user tags  -> count=4   PASS
TC2 no user tags    -> count=2   PASS
==== OVERALL PASS ====
```

The only variable between the two runs was the product code. That is the proof the test is wired to the real function and not to a copy. If it had passed in both runs, the test would be worthless — and you would never have known.

## One entry point, so it can gate CI

Wrap each test case in a `build_and_run.sh` (stage the shim, build with the macro, compile the driver, run, restore the tree), and put one `run_all.sh` on top that discovers every case and exits non-zero if any fails. Now a single command gives a suite result a CI job can gate on:

```
###### SUMMARY: 1 passed, 0 failed ######
```

## Takeaways

- **Never copy the function's logic into the test.** A copy passes even when the product is broken. Call the real, compiled function.
- **Put the test shim inside the library, behind a build-time macro.** From inside, it can reach private members; the macro keeps it out of release builds.
- **Load it from a driver with `dlopen`/`dlsym`**, so you test the exact binary you just built.
- **For a private, I/O-bound function**, add two guarded, build-time-only edits: a public accessor, and a hook that replaces the DB/cloud fetch with an input you feed in.
- **Prove release-safety** with `nm -D | grep` returning nothing.
- **Prove it's a real guard**: it must FAIL without the fix and PASS with it. Run both. A test you have only ever seen pass has told you nothing.
