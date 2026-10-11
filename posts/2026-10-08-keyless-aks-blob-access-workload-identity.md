# Keyless read-only blob access for someone else's AKS pods (Azure Workload Identity)

**Question:** Another team runs pods in their own AKS cluster and needs **read-only** access to blob containers in *my* storage account. I don't want to hand them a storage account key or a client secret. How do I give their pods access to my blobs with **no shared secret at all**?

**Answer:** Use **Azure AD Workload Identity**. Their cluster signs a short-lived OIDC token for each pod; you create **one user-assigned managed identity (UAMI)** and add **federated credentials** that tell Entra "trust tokens from *that* cluster for *these* service accounts." Then you grant the UAMI `Storage Blob Data Reader` on your account. Nothing secret ever changes hands — the pods prove who they are with a cluster-signed JWT, and Entra verifies it against the cluster's public keys.

## How workload identity works (the mental model)

Three parties, no shared secret:

```
[Their AKS cluster]                [Microsoft Entra ID]              [Your storage account]
  pod uses ServiceAccount            verifies the JWT against           Storage Blob Data Reader
  cluster signs a JWT  ───token───▶  the cluster's public keys  ──▶    granted to the UAMI
  (iss, sub, aud claims)             (matched by a federated cred)      → read-only blob access
```

1. The AKS cluster acts as an **OIDC issuer**. It signs a JWT for each pod and publishes its *public* keys at a well-known URL.
2. You create a **UAMI** (a managed identity — an Entra service principal with no password you ever see).
3. You add a **federated credential** to the UAMI: a trust rule matching three claims in the pod's JWT — `iss` (which cluster), `sub` (which Kubernetes service account), `aud` (a fixed audience constant).
4. At runtime the pod presents its JWT; Entra validates the signature against the cluster's published keys and checks it matches a federated credential; if so it issues a real Azure access token for the UAMI.
5. The UAMI has `Storage Blob Data Reader` on your account, so the token can read blobs — and only read.

The key insight: **the cluster's signing key stays in the cluster, and only its public keys are ever exposed.** There's nothing for you to store, rotate, or leak.

## The three pieces of the JWT trust rule

A federated credential matches exactly three things. Get these from the *other* team — they come from their cluster, not yours.

| Claim | What it is | Example (placeholder) |
|---|---|---|
| **issuer** (`iss`) | The cluster's OIDC issuer URL | `https://<region>.oic.prod-aks.azure.com/<tenant>/<cluster-id>/` |
| **subject** (`sub`) | `system:serviceaccount:<namespace>:<serviceaccount>` | `system:serviceaccount:team-app:team-app` |
| **audience** (`aud`) | A fixed well-known constant | `api://AzureADTokenExchange` |

The subject is Kubernetes' own identifier for the pod's service account, stamped into the token. The audience is always `api://AzureADTokenExchange` for workload-identity federation — don't overthink it.

## One identity or many?

The team here had **two** namespaces/service accounts that needed the *same* access. The rule of thumb:

> **One UAMI per distinct access profile, not per service account.**

Both service accounts want the same thing (read-only on the same account), so they **share one UAMI** with **two federated credentials** (one per subject). That means **one role assignment** covers both. A single UAMI holds up to 20 federated credentials, so this scales fine.

If the two needed *different* access (say, one read-only and one read-write), that'd be two UAMIs.

## Phase 1 — create the identity and the trust rules

You (as a Contributor on the resource group) can do all of this. Run it against your subscription.

```bash
# 1. Create the user-assigned managed identity
az identity create \
  -g my-resource-group \
  -n my-aks-reader-identity \
  --subscription <subscription-id>
# → note the output: principalId (= object id) and clientId
```

The output gives you two ids you'll need later:
- **`principalId`** (the object id) → used in Phase 2 for the role assignment.
- **`clientId`** → handed to the team for Phase 3 to annotate their service accounts.

```bash
# 2. Federated credential for the first service account
az identity federated-credential create \
  -g my-resource-group \
  --identity-name my-aks-reader-identity \
  -n fed-team-app \
  --issuer 'https://<region>.oic.prod-aks.azure.com/<tenant>/<cluster-id>/' \
  --subject 'system:serviceaccount:team-app:team-app' \
  --audience 'api://AzureADTokenExchange'

# 3. A second federated credential — same identity, different subject
az identity federated-credential create \
  -g my-resource-group \
  --identity-name my-aks-reader-identity \
  -n fed-team-worker \
  --issuer 'https://<region>.oic.prod-aks.azure.com/<tenant>/<cluster-id>/' \
  --subject 'system:serviceaccount:team-worker:team-worker' \
  --audience 'api://AzureADTokenExchange'
```

Verify both landed:

```bash
az identity federated-credential list \
  -g my-resource-group --identity-name my-aks-reader-identity \
  --query "[].{name:name, subject:subject}" -o table
```

**At this point the identity exists but can touch nothing.** A federated credential is only the *authentication* half — it says "I'll trust these pods." It grants zero *authorization* until Phase 2.

## Phase 2 — grant the role (needs a privileged admin)

A Contributor **cannot** create role assignments — that needs **Owner** or **User Access Administrator**. So this one command goes to whoever has that:

```bash
az role assignment create \
  --assignee-object-id <principalId-from-phase-1> \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/my-resource-group/providers/Microsoft.Storage/storageAccounts/mystorageacct"
```

Two details that save time:
- Use **`--assignee-object-id` + `--assignee-principal-type ServicePrincipal`**, not `--assignee`. It skips a Microsoft Graph lookup that can transiently fail on replication lag right after you create the identity.
- **`--assignee-object-id` requires an explicit `--scope`** — the `--resource-group` shorthand is rejected with that flag.

### Why account scope, not container scope?

Ideally you'd scope `Storage Blob Data Reader` to just the containers they need. But RBAC can't wildcard — if there are thousands of containers (and new ones appear constantly), you'd need thousands of assignments. So **account scope** is the pragmatic choice. It's still read-only: any write returns `403`. The blast radius is "can read any blob in this account," which for a logs/data account is acceptable.

## Phase 3 — the team wires it up on their side

Hand the team the **`clientId`** from Phase 1. They:

1. Annotate each **ServiceAccount** with the client id:
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: team-app
     namespace: team-app
     annotations:
       azure.workload.identity/client-id: <clientId-from-phase-1>
   ```
2. Label each **pod template** so the mutating webhook injects the token:
   ```yaml
   metadata:
     labels:
       azure.workload.identity/use: "true"
   ```

That label triggers Azure's webhook to inject the projected token file plus the env vars (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_FEDERATED_TOKEN_FILE`, `AZURE_AUTHORITY_HOST`) that `DefaultAzureCredential()` reads automatically. Their cluster also needs `--enable-oidc-issuer --enable-workload-identity` turned on — usually already done if they're using workload identity at all.

## Does blobfuse change anything?

The team wanted to **mount** the blobs with blobfuse rather than call the SDK. Good news: **no change to the plan.** Modern blobfuse2 (v2.x) and the Azure Blob CSI driver both support workload identity natively — same keyless auth, no account key. The only client-side differences:

- blobfuse mounts **one container per mount** (not the whole account), so account-scope Data Reader lets them mount any container they name.
- Mount **read-only** (`-o ro`) to match the role.

## The division of labor (who runs what)

This is the part worth remembering — the work splits by *privilege*:

| Phase | Who | Why |
|---|---|---|
| 1 — create UAMI + federated creds | **You** (Contributor) | Contributor can create identities and federated credentials. |
| 2 — role assignment | **An Owner / User Access Administrator** | Role assignments need elevated rights. |
| 3 — annotate SAs + label pods | **The other team** | It's their cluster. They only need the `clientId`. |

## Takeaways

- **Workload identity means no shared secret** — the pods prove themselves with a cluster-signed JWT; you store nothing, rotate nothing, leak nothing.
- A **federated credential** is just a trust rule matching three JWT claims: issuer (which cluster), subject (`system:serviceaccount:<ns>:<sa>`), and the fixed audience `api://AzureADTokenExchange`.
- **One UAMI per access profile, not per service account** — same access → share the identity + one role assignment, with one federated credential per subject.
- A federated credential is **authentication only**; it grants nothing until you add an RBAC role (`Storage Blob Data Reader` for read-only).
- **Account scope** is the realistic choice when there are too many containers to enumerate — it's still read-only (writes `403`).
- **blobfuse / the Blob CSI driver support workload identity too** — no account key, one container per mount, mount `-o ro`.
- Work splits by privilege: **Contributor** makes the identity, an **Owner/UAA** grants the role, the **team** wires up their pods with just the `clientId`.
