# AWS and Azure Cloud Identity: The Same Five Pieces, Different Names

**Question:** I know AWS IAM — access keys, roles, `AssumeRole`, instance profiles, the metadata endpoint. When I move the same app to Azure, what maps to what? And why does a "role" work on a cloud VM but not on my on-prem server?

**Answer:** Azure has the same five building blocks as AWS, just under different names, with one genuine design difference: Azure attaches permissions by *assignment*, not by a role you *assume*. The portability rule is identical on both clouds — a **static key is a secret you can carry anywhere**, while a **role / managed identity is bound to a cloud VM**, because it is handed out by the instance metadata endpoint that only exists inside the cloud.

## The mapping

| AWS | Azure |
|---|---|
| IAM user access key (`backup-user`) | Service principal + client secret (app registration: app/client ID + secret), or a storage account key |
| IAM role + its policies (`DataAdminRole`) | An identity + RBAC role assignments — Azure attaches permissions via role assignments, not a "role you assume" |
| STS `AssumeRole` → temporary creds (session token) | Entra (Azure AD) OAuth2 access token (~1h bearer token) |
| Instance profile on the EC2 (`WorkerRole`) | Managed identity attached to the VM — system-assigned, or user-assigned (UAMI) |
| IMDS `169.254.169.254` (creds endpoint) | Azure IMDS — same IP `169.254.169.254`, token path `/metadata/identity/oauth2/token` |

Five rows, and every row is the same idea with a different label — except row two, which hides a real difference.

## The one real difference: "assume" vs "assign"

On **AWS**, a role is something you *assume*. You start as identity A (say `backup-user`), call `sts:AssumeRole`, and AWS hands you back a *new, temporary* identity B with its own short-lived credentials. For the next hour you are literally a different principal (`assumed-role/DataAdminRole/...`). Permissions live on the role; you step into the role to get them.

On **Azure**, there is no "assume." You attach a **managed identity** to the VM, and you attach **RBAC role assignments** to that identity. The identity doesn't change — it is granted whatever its role assignments say, directly. So "Azure role" and "AWS role" are false friends: an Azure *role* is a named bundle of permissions you *assign* to an identity (closer to an AWS *policy*), while the AWS *role* is the assumable identity itself.

Keep this straight and the rest of the mapping falls into place:

- AWS *policy* ≈ Azure *role definition* (a permission set).
- AWS *role* (assumable) ≈ Azure *managed identity* + its *role assignments*.
- AWS `AssumeRole` temp creds ≈ Azure *OAuth2 access token*.

## Why a key runs anywhere but a role needs a cloud VM

This is the part that bites people, and it's identical on both clouds.

A **static key** — an AWS access key, or an Azure service-principal secret / storage key — is just a pair of strings. Any machine that holds them can sign API calls: your laptop, an on-prem server, a VM in a *different* cloud, anywhere. It carries its own secret, so it needs nothing from the environment it runs on. It only needs network egress to the cloud API and enough permissions on the key.

A **role / managed identity** has **no stored secret**. The code doesn't hold a credential at all — it asks the **instance metadata service (IMDS)** at `169.254.169.254` for one. That endpoint only exists *inside* the cloud, on the VM the identity is attached to. So:

- On an **EC2 / Azure VM**, IMDS answers → the SDK gets credentials → the role/identity works.
- On an **on-prem box** (or any non-cloud machine), there is no IMDS → the SDK waits, then times out with something like *"unable to reach the credentials endpoint."*

That's the whole reason a role-based setup forces you to run on a cloud VM, while a key-based setup can run on your on-prem server untouched. It's not a permissions problem — it's a *where does the credential come from* problem.

| | Where it can run |
|---|---|
| **Static key** (access key / SP secret) | Anywhere — on-prem, other cloud, the cloud itself. No IMDS needed. |
| **Role / managed identity** | Only on a cloud VM — no stored secret; it must read its identity from IMDS. |

## The two-step dance is the same on both

When you *do* run on a cloud VM, the secret-free flow has the same shape on AWS and Azure:

**AWS:**
1. The VM boots with an instance profile (`WorkerRole`). Its boot identity comes from IMDS.
2. That identity calls `sts:AssumeRole` to step into `DataAdminRole` (which holds the real permissions, e.g. disk reads).
3. It gets temporary session creds and calls the API as the admin role.

**Azure:**
1. The VM boots with a managed identity attached. Its token comes from IMDS (`/metadata/identity/oauth2/token`).
2. It requests an OAuth2 access token for the target resource (ARM, Storage, …).
3. It gets a ~1h bearer token and calls the API with it.

Same three beats — *boot identity from IMDS → exchange for scoped credentials → call the API* — just AWS does step 2 as an explicit `AssumeRole`, and Azure does it as a token request against an identity that already carries its role assignments.

## Takeaways

- Azure's identity stack maps one-to-one onto AWS's: SP+secret ≈ access key, managed identity ≈ instance profile, OAuth2 token ≈ `AssumeRole` session creds, IMDS is even the *same IP* (`169.254.169.254`).
- "Role" means different things: an **AWS role** is an assumable identity; an **Azure role** is a permission bundle you *assign*. The AWS role maps to an Azure **managed identity + role assignments**.
- A **static key is portable** — it carries its own secret, so it runs on-prem or anywhere.
- A **role / managed identity is location-bound** — it has no secret and must fetch one from IMDS, which only exists on a cloud VM.
- A "credentials endpoint timed out" error on-prem isn't a permissions bug — it's a role trying to reach IMDS where there is none. Use a key there, or move the workload onto a cloud VM.
