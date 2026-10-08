# Red Hat Developer Hub — Microsoft Entra ID — ArgoCD/Kustomize

A Red Hat Developer Hub instance (Backstage CR) at
**https://rhdhazure.apps.ocprod.303.tube**, with:

- Sign-in through **Microsoft Entra ID** (built-in `microsoft` auth provider)
- Entra users/groups imported into the catalog by the **MS Graph** dynamic plugin
- A **dedicated PostgreSQL 16** StatefulSet (operator-managed local DB disabled)
- The **Kubernetes** plugin wired to the OpenShift cluster that hosts the
  instance, authenticating as its own in-cluster ServiceAccount
- Everything deployable by **ArgoCD** from Kustomize directories

**Prerequisite:** the Red Hat Developer Hub Operator (1.7 or later) is already
installed on the cluster. This repo only creates the instance, not the operator.

```
.
├── argocd/                     # ArgoCD Application (bootstrap)
│   └── rhdh-azure.application.yaml      -> path: overlays/ocprod
├── base/
│   ├── postgres/               # StatefulSet + Services (PVC 10Gi)
│   └── rhdh/
│       ├── backstage.yaml      # Backstage CR (rhdh.redhat.com/v1alpha4)
│       ├── kubernetes-rbac.yaml # SA + read-only ClusterRole for the K8s plugin
│       └── dynamic-plugins.yaml
└── overlays/ocprod/
    ├── kustomization.yaml      # namespace, route host patch
    ├── app-config.yaml         # baseUrl, DB, Entra auth, MS Graph provider
    └── secrets/*.example.yaml  # templates only (git-ignored for real values)
```

Sync order: namespace (-1) → PostgreSQL (0) → ServiceAccount/RBAC (1) →
Backstage CR (2).

---

## 1. Register the app in Microsoft Entra ID

Azure portal → **Microsoft Entra ID → App registrations → New registration**

| Setting | Value |
|---|---|
| Supported account types | Single tenant |
| Redirect URI (platform **Web**) | `https://rhdhazure.apps.ocprod.303.tube/api/auth/microsoft/handler/frame` |

Then:

1. **Certificates & secrets** → New client secret → copy the *Value*.
2. **API permissions** → Microsoft Graph:
   - *Delegated*: `openid`, `offline_access`, `profile`, `email`, `User.Read` (sign-in)
   - *Application*: `User.Read.All`, `GroupMember.Read.All` (catalog ingestion)
   - **Grant admin consent**.
3. Note the **Directory (tenant) ID** and **Application (client) ID** from Overview.

## 2. Create the secrets

Secrets are deliberately kept out of Git. Either create them directly:

```bash
oc new-project rhdh-azure   # or let ArgoCD create it first

oc -n rhdh-azure create secret generic rhdh-azure-secrets \
  --from-literal=BACKEND_SECRET="$(openssl rand -base64 32)" \
  --from-literal=AZURE_TENANT_ID=<tenant-id> \
  --from-literal=AZURE_CLIENT_ID=<client-id> \
  --from-literal=AZURE_CLIENT_SECRET=<client-secret> \
  --from-literal=AZURE_DEVOPS_CLIENT_ID=<devops-sp-client-id> \
  --from-literal=AZURE_DEVOPS_CLIENT_SECRET=<devops-sp-client-secret> \
  --from-literal=AZURE_DEVOPS_ORG=<devops-org-name>

oc -n rhdh-azure create secret generic rhdh-postgres-secret \
  --from-literal=POSTGRES_HOST=rhdh-postgres \
  --from-literal=POSTGRES_PORT=5432 \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -base64 24)"
```

…or, to keep it fully GitOps, seal them and reference the sealed files in
`overlays/ocprod/kustomization.yaml`:

```bash
cp overlays/ocprod/secrets/rhdh-azure-secrets.example.yaml /tmp/s.yaml  # fill in values
kubeseal -o yaml < /tmp/s.yaml > overlays/ocprod/secrets/rhdh-azure-secrets.sealed.yaml
```

(External Secrets Operator with Azure Key Vault works equally well — just produce
Secrets with the same names and keys.)

> Do not change `POSTGRES_PASSWORD` after first start: the PostgreSQL image only
> applies it at init. Rotate with `ALTER USER postgres PASSWORD ...` first.

## 3. Deploy with ArgoCD

1. Push this directory to your Git repo and set `repoURL` in
   `argocd/rhdh-azure.application.yaml`.
2. Bootstrap:

```bash
oc apply -k argocd/
```

The OpenShift GitOps controller needs permission to manage the `rhdh-azure`
namespace and `backstages.rhdh.redhat.com`. The namespace is labelled
`argocd.argoproj.io/managed-by=openshift-gitops`; if your controller still
can't manage `Backstage` resources, grant its ServiceAccount a role for them.

## 4. Kubernetes plugin

`plugin-kubernetes` (frontend) and `plugin-kubernetes-backend` are both
preinstalled in the RHDH image and listed, disabled, in the catalog index's
`dynamic-plugins.default.yaml`; `base/rhdh/dynamic-plugins.yaml` enables them by
their local dist paths and mounts `EntityKubernetesContent` into
`entity.page.kubernetes/cards`. No OCI artifact is needed, unlike the Azure
DevOps plugins.

Nothing else is required to reach the hosting cluster — no kubeconfig, no API
URL, no token to create or rotate:

- `base/rhdh/kubernetes-rbac.yaml` creates the `rhdh-kubernetes` ServiceAccount
  and binds it to the read-only `rhdh-kubernetes-reader` ClusterRole (workloads,
  routes, pod logs, pod metrics — cluster-wide, because the `multiTenant`
  service locator searches every namespace).
- The Backstage CR patches the Deployment to run as that ServiceAccount with
  `automountServiceAccountToken: true`. The operator leaves automount off by
  default, which is why the patch is needed.
- `app-config.yaml` sets `authProvider: serviceAccount` and **no**
  `serviceAccountToken`. That is the backend's in-cluster mode: it takes the API
  server and CA from the pod environment and re-reads
  `/var/run/secrets/kubernetes.io/serviceaccount/token` per request, so rotation
  of the projected token is handled for free.

The ClusterRole is cluster-scoped: a second Developer Hub instance on the same
cluster must reuse it (adding its ServiceAccount as a subject) or use its own
name.

### Making the tab appear on an entity

The tab renders only for entities that declare which workloads are theirs:

```yaml
metadata:
  annotations:
    backstage.io/kubernetes-id: my-service        # matches the label below
    # or: backstage.io/kubernetes-namespace: my-namespace
    # or: backstage.io/kubernetes-label-selector: app=my-service,env=prod
```

with workloads labelled to match:

```bash
oc -n my-namespace label deployment/my-service \
  backstage.io/kubernetes-id=my-service
```

### Other clusters

To add a remote cluster, append another entry under
`kubernetes.clusterLocatorMethods[0].clusters`. In-cluster mode does not apply
there, so it needs an explicit `url`, a `serviceAccountToken` (from a Secret via
`extraEnvs`, e.g. `${K8S_TOKEN_DEV}`) and `caData`.

## 5. Verify

```bash
oc -n rhdh-azure get backstage,statefulset,pods,route
oc -n rhdh-azure logs deploy/backstage-developer-hub-azure -c backstage-backend \
  | grep -i -E "msgraph|microsoft|error"
```

Kubernetes plugin:

```bash
# Pod runs as the ServiceAccount, with its token mounted
oc -n rhdh-azure get deploy/backstage-developer-hub-azure \
  -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'

# That ServiceAccount really can read workloads cluster-wide
oc auth can-i list deployments -A \
  --as=system:serviceaccount:rhdh-azure:rhdh-kubernetes

# Clusters the backend resolved from app-config
oc -n rhdh-azure logs deploy/backstage-developer-hub-azure -c backstage-backend \
  | grep -i -E "kubernetes|cluster"
```

Open https://rhdhazure.apps.ocprod.303.tube and choose **Sign in using Microsoft**.

First sign-in may fail with *"user not found"* until the first MS Graph sync
(starts 15 s after boot, then hourly) has imported your user. Check the catalog
for `kind:User` entities if it persists — the user must pass the `user.filter`
and the resolvers match on Entra object ID, then email, then UPN local part.

## Customising

| What | Where |
|---|---|
| Hostname | `overlays/ocprod/kustomization.yaml` (patch) **and** `app-config.yaml` (`baseUrl`, `cors`) |
| Which users/groups are imported | `catalog.providers.microsoftGraphOrg.default.user/group.filter` |
| DB size / storage class | `base/postgres/statefulset.yaml` → `volumeClaimTemplates` |
| More plugins | `base/rhdh/dynamic-plugins.yaml` |
| Kubernetes clusters / console link | `app-config.yaml` → `kubernetes.clusterLocatorMethods` |
| What the Kubernetes plugin may read | `base/rhdh/kubernetes-rbac.yaml` → ClusterRole rules |
| RBAC | Enable the `permission` block + an RBAC policy ConfigMap; Entra groups appear as `group:default/<name>` |
| Another environment | Copy `overlays/ocprod` to a new overlay and change host/app-config |

### Version notes

- Written for **RHDH 1.10**. The CR uses `rhdh.redhat.com/v1alpha4`, which
  1.7+ operators serve. Newer operators also serve `v1alpha5`; no field changes are needed here.
- In the upcoming RHDH **2.x** line, auth providers become dynamic plugins. When
  upgrading, add `ref://backstage-plugin-auth-backend-module-microsoft-provider`
  and change the MS Graph entry to `ref://backstage-plugin-catalog-backend-module-msgraph`
  in `dynamic-plugins.yaml`; the app-config stays the same.
