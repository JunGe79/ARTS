# Both Azure Managed Identities Need RBAC Roles

**Question:** In Azure, a system-assigned managed identity (SAMI) clearly needs an RBAC role assignment to read a Key Vault secret — but a user-assigned managed identity (UAMI) seems to work without one. Does UAMI not need RBAC roles?

**Answer:** Both SAMI and UAMI need RBAC role assignments to reach a protected resource. The difference is *when and where* you assign the role, and whether the identity (and its roles) can be reused. A UAMI that "works without a role" almost always already has the role — assigned to the identity itself (often earlier, or at a broader scope), not to the app you just attached it to.

## The real difference: lifecycle and reuse, not whether roles are required

| | SAMI | UAMI |
|---|---|---|
| Identity creation | Auto-created with the Azure resource | Created on its own, independently |
| Lifecycle | Tied to the resource | Independent of any resource |
| **RBAC role assignment** | **Required** | **Required** |
| Reuse across resources | No (one per resource) | Yes (attach to many) |
| Role assignment persistence | Lost when the resource/identity is deleted | Kept even if a consuming resource is deleted |
| Attaching to a resource | Automatic (just enable it) | Explicit (you attach it) |

The row that matters: **RBAC is required either way.** Everything else is about lifecycle and reuse.

## Why UAMI looks like it skips the role

Take an Azure Function reading a secret from Key Vault.

**With SAMI** — the identity is born with the app, so you do the role assignment right there:

1. Enable SAMI on Function App A → Azure creates a new service principal.
2. Assign **Key Vault Secrets User** to that identity.
3. App A can read secrets.

**With UAMI** — the identity exists first, and you give it the role once:

1. Create UAMI X.
2. Assign **Key Vault Secrets User** to UAMI X.
3. Attach UAMI X to Function App A.
4. Attach the *same* UAMI X to Function App B.

Now **both** apps read Key Vault with no extra role assignment — because the role belongs to **the identity (X)**, not to the app. The second app "works without a role" only because the identity it borrowed already had one.

## The two operations people conflate

Azure has two *separate* permission operations, and mixing them up is what makes UAMI look like magic:

1. **Attach a UAMI to a resource** — lets that resource *use* the identity. Whoever does the attach needs **Managed Identity Operator** (or equivalent) on the UAMI.
2. **Assign an RBAC role to a UAMI** — lets the identity *access a target* (Key Vault, Storage, Service Bus, …).

Attaching is not authorizing. Attaching says "App B may act as X." The role says "X may read this Key Vault." You need both, and they are done at different times by possibly different people.

## So if SAMI needs a role and UAMI doesn't…

The likely explanation isn't that UAMI is exempt — it's that **the UAMI already has the permission**, possibly:

- assigned directly to the identity earlier, or
- inherited from a role assigned at the **resource group** or **subscription** scope.

Check the UAMI's own role assignments *and* the inherited ones (RG/subscription) before concluding it needs nothing.

## Takeaways

- Both SAMI and UAMI **require** RBAC role assignments — neither is exempt.
- With SAMI the role is tied to one resource's identity; with UAMI the role lives on a reusable identity you attach to many resources.
- "Attach an identity" and "assign a role to an identity" are **different** operations (Managed Identity Operator vs an RBAC role assignment).
- If a UAMI seems to need no role, it already has one — check direct and inherited (RG/subscription) assignments.
