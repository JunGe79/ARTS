# App registration + certificate: authenticating a fleet to Azure without shipping secrets

**Question:** I have a fleet of machines — say thousands of edge servers — that each need to authenticate to an Azure service (a Function, a storage account). I don't want to generate a password for each one and ship it out, because a shipped secret can be stolen in transit, on disk, or from wherever the cloud stores it. How does the "app registration + certificate" approach work, and — the part that confused me most — *who* actually sets each one up?

**Answer:** Each machine generates its **own** key pair locally, keeps the **private key** on the box, and only its **public certificate** is registered on a per-machine **app registration** in your Entra (Azure AD) directory. To log in, the machine signs a tiny JWT with its private key; Azure verifies it against the registered public cert and hands back an access token. Nothing secret ever travels the wire. The catch is **provisioning**: creating the app registration, attaching the cert, and granting it permissions is a *privileged directory operation*, so a **central automation** does it — never the machine itself.

## What an app registration actually is

It's an **identity for a program**, living in Azure's directory (Entra ID / Azure AD) — the software equivalent of a user account.

- A **user account** is an ID card for a *person*.
- An **app registration** is an ID card for a *program*, so software can log in and be granted permissions.

It has an **Application (client) ID** (its "username"), a home **tenant**, one or more **credentials** (a secret or a **certificate** — its "password/key"), and **permissions** (RBAC). When created, Azure also makes a matching **service principal** — the object that actually holds the permissions. Treat "app registration + service principal" as one identity.

## Certificate vs. client secret

Both prove "I am this app," but they leak very differently:

- **Client secret** = a password. Azure must **store** it to compare, and you must **ship** it to the machine. Two copies of a thing that grants impersonation, and it travels over the wire at every login.
- **Certificate** = a key pair. You keep the **private key**; Azure only ever holds the **public certificate**. You prove identity by **signing**, so only a *signature* crosses the wire — never the key. A breach of Azure's directory yields only public keys, which forge nothing.

For a fleet where you don't want to distribute secrets, the certificate is the clear choice.

## The certificate flow (OAuth2 client credentials + JWT assertion)

This is the standard flow; here's what actually happens on the wire. The machine builds a short **JWT** ("client assertion") in three parts:

**1. Header** — says *how* it's signed and *which* cert:
```json
{ "alg": "RS256", "typ": "JWT", "x5t": "<base64url of the cert's SHA-1 thumbprint>" }
```
`x5t` is the pointer that tells Azure which registered public cert to check.

**2. Payload (claims)** — *who* and *for how long*:
```json
{ "aud": "https://login.microsoftonline.com/{tenant}/oauth2/token",
  "sub": "<client-id>", "iss": "<client-id>",
  "exp": now + 3600, "nbf": now }
```
`sub` = `iss` = the app asserting its own identity; valid for one hour.

**3. Signature** — the proof:
```
RS256_sign( privateKey, base64url(header) + "." + base64url(payload) )
```
Signed **with the private key**, locally. Only its holder could produce this.

Then it POSTs to Azure's token endpoint:
```
client_id={id}&scope={resource}&grant_type=client_credentials
&client_assertion=<header>.<payload>.<signature>
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
```

Azure looks up the app by `client_id`, finds the registered cert matching `x5t`, **verifies the signature** with its public key, and returns a short-lived **access token**. The machine then calls the real resource with `Authorization: Bearer <token>`; the resource just trusts Azure and authorizes via RBAC. **The resource stores no secret.**

## Who generates the key pair? The machine.

This is the part that makes the whole thing safe, and it's easy to get backwards:

- The **machine generates its own key pair** — a **self-signed** cert is fine (no Certificate Authority, nobody buys anything).
- The **private key stays on the machine** and never moves.
- Only the **public certificate** goes *up* to your cloud.

The *wrong* way is to have the cloud generate the key pair and ship the **private key** (a `.pfx`) down to each machine — that's the exact "distribute a secret to thousands of boxes" problem you were trying to avoid. Generate on the box, and no private material ever crosses the wire in either direction.

## Who registers the app registration? A privileged automation — and here's the subscription-vs-tenant trap

The machine **cannot** register itself, and shouldn't. Two facts explain why:

- **App registrations live in a *tenant* (the directory), not a *subscription* (the resource/billing container).** People conflate these constantly. Creating an app registration is a **directory** operation; it has nothing to do with which subscription your storage sits in.
- Only an identity with **Microsoft Graph** rights (`Application.ReadWrite`) in *your* tenant can create one. Your edge machines have no such rights (and no reason to touch Azure directly).

So a **central cloud automation** does it. Note it needs privileges in **two different planes**:

| Step | Plane | Permission |
|---|---|---|
| Create the app registration + attach the public cert | **Directory (tenant)** | Graph `Application.ReadWrite` |
| Grant the new identity access to your resource (RBAC) | **Resource (subscription)** | `roleAssignments/write` (Owner / User Access Administrator) |

That's a powerful combination — the ability to **mint identities** *and* **hand out permissions**. You do **not** want to bolt that onto every service; keep it in **one audited automation**, and have everything else *call* it.

## The end-to-end flow

```
MACHINE                                   YOUR CLOUD
1. proves it's node X  ────(bootstrap)──► verify (existing trust)
2. generates keypair, keeps PRIVATE key,
   sends PUBLIC cert  ─────────────────►  3. privileged automation → Azure (Graph):
                                             • create app registration (in your tenant)
                                             • attach the public cert
                                             • assign RBAC on your resource
   ◄──(client ID + tenant ID)────────────
4. from now on: sign a JWT with the
   private key → get an Azure token ─────► Azure validates (public cert), issues token
5. call the resource with the token
```

Notice step 1: there's still a **bootstrap** — some first-contact trust that proves the machine is really node X before you register its cert. Certificates don't remove that; they just make sure no *private* material is ever transmitted afterward.

## Takeaways

- An **app registration** is an identity for a program, living in a **tenant** (directory), not a subscription.
- **Certificate beats client secret**: the private key never leaves the machine; only a *signature* is sent, and the cloud stores only public keys.
- Auth is the OAuth2 **client-credentials** grant with a **JWT assertion** signed by the private key; Azure verifies with the registered cert and issues a token.
- **The machine generates its own key pair** and keeps the private key — never have the cloud generate it and ship the private key down.
- **The machine never registers itself.** A **central, privileged automation** creates the app registration (Graph, tenant plane) and grants RBAC (subscription plane) — keep that power in one audited place.
- You still need a **bootstrap** trust for the very first cert registration; the certificate model doesn't eliminate it, it just stops secrets from crossing the wire.
