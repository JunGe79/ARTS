# Setting Up a Large Web-App Dev Environment, Part 1: The Toolchain and First Build

**Question:** You've joined a project with a big enterprise web application — a React single-page app on the front, a Java/Tomcat server on the back — and you need to run the whole thing on your own machine. Where do you start, and why are there so many tools?

**Answer:** You install two separate toolchains (Node/Yarn for the React side, JDK/Gradle for the Java side), pull the source, run the installs, build once, and point the app at a backend with a config file. This post covers that first half — the mental model, the prerequisites, and getting to a first successful build. Later posts cover running it and debugging.

## The mental model: it's two apps, not one

The thing you're setting up is really **two programs that talk to each other**:

- **The frontend** — a React single-page app (the buttons, tables, forms you see). It runs in the browser.
- **The backend** — a Java web application running on a server (Tomcat). It holds the data, does the real work, and *serves* the frontend to the browser.

That's why you need two completely different toolchains: **Node/Yarn** build the React side, and **JDK/Gradle** build the Java side. Keep that split in your head and the rest makes sense.

## 0. The 60-second overview

Roughly, the whole setup is:

```
get the source
install the JS build tools        (at the repo ROOT — easy to miss)
build the app                     (may hit a checksum snag)
edit a config file                (point it at a backend)
compile the Java                  (once)
run it                            (embedded Tomcat)
open it in a browser
```

Each step below earns its place — the annotations are the things that cost real time the first time around.

## 1. Prerequisites — the tools, and why each one exists

### Git (+ Git LFS)

Source control. Git LFS ("Large File Storage") handles big binary assets; some repos' hooks call `git-lfs` even when your change doesn't touch a large file, so install it up front to avoid a surprise failure at push time.

### JDK 21 + Gradle

The backend is Java, so you need a **JDK** (Java Development Kit) to compile it. **Gradle** is the *build tool* for Java — it downloads dependencies, compiles the source, runs tests, and packages the result. Think of Gradle as "npm + webpack, but for Java."

You usually don't install Gradle globally — projects ship a `gradlew` **wrapper** script that downloads the exact Gradle version the project pins, so everyone builds the same way. So you run `./gradlew build`, not a globally-installed `gradle`.

### Node.js (via nvm) + Corepack + the package registry

The frontend is JavaScript, so you need **Node.js**. Install it with **nvm** (Node Version Manager) so you can keep several Node versions and switch between them:

```
nvm install 22
```

Two things people mix up:

- **nvm vs npm** — *nvm* manages *which version of Node* you have; *npm* (bundled with Node) manages *which libraries* your project uses. Analogy: nvm ≈ pyenv, npm ≈ pip.
- **Corepack** — a tool bundled with Node that hands you the right **Yarn** version per project automatically (each project pins one in its `packageManager` field). Enable it once:

```
corepack enable
```

That's why you don't `npm i -g yarn` — Corepack does that job.

If your company hosts an **internal package registry** (a private mirror of npm), point both package managers at it:

```
yarn config set npmRegistryServer https://registry.example.com
npm  set registry https://registry.example.com
```

That's where the internal-only packages live (and it caches public ones too).

### The dev launcher + a bundled Tomcat

Finally, install the CLI your project uses to build/run locally, plus a bundled Tomcat so you don't need a separate server install:

```
npm i -g <dev-launcher> <embedded-tomcat>
```

(`-g` = global; these land on your PATH everywhere.)

**Global vs local — a common mix-up:** most of the above (nvm, corepack, npm registry, `npm -g`) are **machine/user-wide**, not per-folder. The one exception is `yarn config set` — by default it writes to the `.yarnrc.yml` in your *current folder*; add `-H` to make it global.

## 2. Get the source

Clone your fork, add the upstream remote:

```
git clone https://git.example.com/<you>/web.git
cd web
git remote add upstream https://git.example.com/eng/web.git
```

(A `git worktree` is handy for a second checkout on another branch without a second clone — a topic for a later post.)

## 3. The root install — the step everyone skips

Before building anything, run the JS install **at the repo root**:

```
yarn install
```

Why: the top-level build script itself has dependencies (small helper libraries). Skip this and the build dies immediately with `Cannot find module '...'`. Remember — dependencies aren't stored in git (`node_modules/` is gitignored); `yarn install` is what *downloads* them. It's the JS equivalent of `pip install -r requirements.txt`.

In a monorepo there are **several** `package.json` files, each needing its own install — the root one (for the build scripts) is the easy one to forget.

## 4. Build it — and the checksum gotcha

Run the project's build command, something like:

```
node build.js <target>
```

This installs the frontend's dependencies and builds the React app.

**The gotcha:** if you use an internal package registry, the first build may fail with checksum errors:

```
YN0018: acorn@...  The remote archive doesn't match the expected checksum
YN0018: tapable@... The remote archive doesn't match the expected checksum
```

This is *not* a broken checkout. An internal mirror often re-packs tarballs slightly differently than the public npm ones the lockfile was built against, so the hashes differ — and a strict (`--immutable`) install refuses to continue.

Fix: do a one-time install that's allowed to update the local checksums, then re-run the build:

```
cd <frontend-folder>
YARN_CHECKSUM_BEHAVIOR=update yarn install
```

Don't commit the changed lockfile — those checksum edits are local-only (CI uses the original public-npm checksums).

*(You'll also see a wall of peer-dependency **warnings** during the build. Those are noise. Only a line that says "Failed with errors" is the actual failure — read that, not the warnings.)*

## 5. Point it at a backend — the config file

The frontend has no data of its own; it forwards API calls to a real backend. A local build usually reads a config file that tells it **which backend** and flips a few "I'm a local/dev instance" switches. Conceptually:

```
# where the backend lives
baseUrl            = https://backend.example.com/api/

# make the app actually READ this file (see below)
configFileOverride = true

# dev-mode switches
devMode            = true                     # take the local-dev code paths
reviewAppMode      = true                     # treat this as a preview instance, not production
disableSSO         = true                     # skip the auto-login handshake; use plain login
forceHttps         = false                    # we only have a plain-HTTP port locally
corsAllowlist      = http://localhost:3000    # let the dev server's page call the backend
```

A few of these are worth understanding, because you'll meet them again:

- **Point-at-the-backend (`baseUrl`)** — a local frontend forwards its API calls to a real backend server; this line chooses which one.
- **`configFileOverride` — the "actually read my file" switch.** This is the one people miss. Normally the app *doesn't* trust this local file for most settings — it fetches the authoritative configuration from the backend and **overrides your file values with them.** So you can edit `baseUrl` all you like and nothing happens. `configFileOverride = true` flips that precedence: now **your local file wins**, and your hand-edited values actually take effect. Without it, the rest of the file is basically decorative.
- **Dev / preview mode** — production builds enforce the full login/security machinery; the dev switches relax that so a local instance can boot and log in normally.
- **Disable SSO** — many enterprise apps auto-log-you-in via a single-sign-on handshake when components are co-installed. Locally that handshake has nothing to talk to, so you turn it off and use a plain username/password login.
- **CORS allowlist** — if you run the React app on a dev server (say `localhost:3000`) that calls a backend on a *different* address, the browser blocks it by default. The allowlist tells the backend, "trust pages coming from `localhost:3000`." (More on CORS in a later post — it's the single most common browser error you'll hit.)

### Why "dev mode" is sometimes mandatory, not optional

It's tempting to read `devMode = true` as just "turn on convenient dev behavior." Often it's more than that — it can be a **runtime-compatibility shim** without which the app won't even boot locally, and it's worth knowing the shape of this because you'll hit the class of problem again.

Here's the trap. Your **local** server is usually a lightweight, bundled one — and it's frequently an **older version** than the full server the code is built against. Java's servlet API (the thing that handles requests and cookies) is versioned, and newer versions add new methods. If the code calls a method that only exists in the *newer* servlet version, it compiles fine (it was built against the new API) but at runtime on the *older* local server that method **doesn't exist** — and the JVM throws a `NoSuchMethodError` on the first request. The app looks dead.

`devMode = true` exists partly to **skip those newer-only code paths** on the local server. A common example: code that stamps a tracking **cookie** using a brand-new cookie API — dev mode routes around it so the old local server never hits the missing method.

Which is a good excuse to make sure you understand cookies, since they sit right at this seam.

### Sidebar: what a cookie actually is

A **cookie** is a small piece of text the site tells your browser to save, and the browser **sends it back automatically on every future request to that site.** That's the whole trick — HTTP has no memory of its own, so cookies are how a site "remembers" you between clicks.

When you log in:

1. You POST your username and password.
2. The server verifies them and replies with `Set-Cookie: session=abc123…`.
3. Your browser stores `session=abc123`.
4. On every later request the browser adds `Cookie: session=abc123`; the server looks it up, sees "already authenticated," and lets you straight in — **no password re-sent.** That random value is a *session token*, not your password.

Why you sometimes stay logged in after closing the browser, and sometimes don't:

| Cookie type | Has an expiry? | Survives closing the browser? |
|---|---|---|
| **Session cookie** (`session=abc`) | No | **No** — deleted on browser close |
| **Persistent cookie** (`session=abc; Max-Age=…`) | Yes | **Yes** — kept until the expiry date |

So "I didn't have to log in again" means your browser **still held a valid, unexpired cookie** and sent it automatically — and it survived the close only because it was a *persistent* ("remember me") cookie. Two caveats: even a persistent cookie dies at its expiry, and the **server can still reject it** (idle timeout, logout elsewhere, a redeploy that cleared sessions) — in which case you're bounced to login even though the cookie is still sitting in your browser.

That "tracking cookie" the dev-mode shim skips is exactly this machinery — a `Set-Cookie` the server tries to write on every request, using a cookie method the local server is too old to have.

## Takeaways

- The app is **two programs**: a React frontend and a Java/Tomcat backend — which is why there are **two toolchains** (Node/Yarn and JDK/Gradle).
- **nvm** picks your Node version; **npm/Yarn** install libraries; **Corepack** hands you the right Yarn per project.
- Run the **root `yarn install`** — the top-level build script needs it, and it's the easy one to miss.
- An **internal registry** causes first-build **checksum errors** — fix with a one-time checksum-update install; don't commit the lockfile change.
- A local build **points at a real backend** via a config file, plus a few **dev-mode switches** (dev/preview mode, disable SSO, a CORS allowlist for the dev-server port).
- Set the **"actually read my file" override** (`configFileOverride`) or the app ignores your edits and pulls config from the backend instead.
- **Dev mode can be mandatory, not optional** — it skips newer-only code paths (e.g. a new cookie API) that would `NoSuchMethodError` on the older lightweight local server.

Next up (**Part 2**): compiling the Java, running the whole thing on embedded Tomcat, and the two ways to run the frontend — the fast dev server vs. the full deployed build.
