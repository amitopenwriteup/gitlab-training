# FluxCD Multi-Tenancy & RBAC Lab Guide
> Namespace Isolation, RBAC Configuration, and Tenant Onboarding with FluxCD GitOps

**3 Labs · 25+ Commands · Production-Ready**

---

## Table of Contents

- [Concepts — Multi-Tenancy in FluxCD](#concepts--multi-tenancy-in-fluxcd)
- [Lab 1 — Namespace-Based Isolation](#lab-1--namespace-based-isolation)
- [Lab 2 — RBAC Configuration for Flux](#lab-2--rbac-configuration-for-flux)
- [Lab 3 — Service Accounts and Permissions](#lab-3--service-accounts-and-permissions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Concepts — Multi-Tenancy in FluxCD

### What is Multi-Tenancy?

Multi-tenancy in Kubernetes means multiple teams (tenants) share the same cluster while being isolated from each other. FluxCD enforces this at the GitOps layer — each tenant has their own namespace, and Flux ensures a tenant can only deploy to their own namespace and cannot read or modify another tenant's resources.

### Your Current Repo Structure

```
flux-gitops/
└── clusters/
    └── production/
        ├── apps/                        ← workload manifests live here
        └── flux-system/
            ├── gotk-components.yaml     ← Flux controller deployments (generated)
            ├── gotk-sync.yaml           ← GitRepository + Kustomization for this repo
            └── kustomization.yaml       ← root kustomize manifest
```

### Target Structure After This Lab

```
flux-gitops/
└── clusters/
    └── production/
        ├── apps/
        ├── flux-system/
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml
        └── tenants/                     ← NEW: one folder per tenant
            ├── tenant-a/
            │   ├── namespace.yaml
            │   ├── serviceaccount.yaml
            │   ├── rbac.yaml
            │   ├── gitrepository.yaml
            │   ├── sync.yaml
            │   └── kustomization.yaml
            └── tenant-b/
                ├── namespace.yaml
                ├── serviceaccount.yaml
                ├── rbac.yaml
                ├── gitrepository.yaml
                ├── sync.yaml
                └── kustomization.yaml
```

### The Two-Layer Model

```
Platform Team (cluster-admin)
└── Manages: flux-system, CRDs, cluster-wide policies
    └── Provisions: one namespace + one GitRepository per tenant

Tenant Team (namespace-scoped)
└── Manages: their own namespace only
    └── Deploys: apps, secrets, configmaps within their namespace
```

### Key Concepts

**Impersonation** — `spec.serviceAccountName` in a Flux Kustomization tells the controller: *apply these resources as this ServiceAccount, not as cluster-admin.* The ServiceAccount's Role limits what can be created and where.

**`--no-cross-namespace-refs`** — A controller flag that prevents a Kustomization in one namespace from referencing a GitRepository in a different namespace. This stops tenant-a from consuming tenant-b's sources.

**Tenant isolation levels:**

| Level | Mechanism | Prevents |
|---|---|---|
| Namespace | Kubernetes namespaces | Resource access across tenants |
| RBAC | Role + RoleBinding | Deploying outside own namespace |
| Source | `--no-cross-namespace-refs` | Cross-tenant Git source references |

---

## Lab 1 — Namespace-Based Isolation

**Objective:** Patch the Flux controllers to enforce isolation, then create namespaces for two tenants inside your existing repo.

**Estimated Time:** 30 minutes

### Step 1 — Patch the Flux Controllers

Edit your existing `clusters/production/flux-system/kustomization.yaml` to add controller patches.

First, create the patch file:

```bash
cat > clusters/production/flux-system/kustomize-controller-patch.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kustomize-controller
  namespace: flux-system
spec:
  template:
    spec:
      containers:
        - name: manager
          args:
            - --events-addr=http://notification-controller.flux-system.svc.cluster.local./
            - --watch-all-namespaces=true
            - --log-level=info
            - --log-encoding=json
            - --enable-leader-election
            - --no-cross-namespace-refs=true
            - --default-service-account=default
EOF
```

Now update your existing `clusters/production/flux-system/kustomization.yaml` to reference the patch:

```yaml
# clusters/production/flux-system/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - path: kustomize-controller-patch.yaml
    target:
      kind: Deployment
      name: kustomize-controller
```

---

### Step 2 — Create the Tenants Directory

```bash
# From the root of your flux-gitops repo
mkdir -p clusters/production/tenants/tenant-a
mkdir -p clusters/production/tenants/tenant-b
```

---

### Step 3 — Create Tenant Namespaces

```bash
# Namespace for tenant-a
cat > clusters/production/tenants/tenant-a/namespace.yaml <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    toolkit.fluxcd.io/tenant: tenant-a
    environment: production
EOF

# Namespace for tenant-b
cat > clusters/production/tenants/tenant-b/namespace.yaml <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-b
  labels:
    toolkit.fluxcd.io/tenant: tenant-b
    environment: production
EOF
```

---

### Step 4 — Add a Kustomization Resource List per Tenant

```bash
# tenant-a resource list
cat > clusters/production/tenants/tenant-a/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - serviceaccount.yaml
  - rbac.yaml
  - gitrepository.yaml
  - sync.yaml
EOF

# tenant-b resource list
cat > clusters/production/tenants/tenant-b/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - serviceaccount.yaml
  - rbac.yaml
  - gitrepository.yaml
  - sync.yaml
EOF
```

---

### Step 5 — Reference Tenants from the Root Kustomization

Update your existing `clusters/production/flux-system/gotk-sync.yaml` OR add a new top-level Kustomization that includes the tenants path.

Create a tenants Flux Kustomization so Flux watches the tenants folder:

```bash
cat > clusters/production/tenants-sync.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: tenants
  namespace: flux-system
spec:
  interval: 5m
  path: ./clusters/production/tenants
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system          # reuses the same GitRepository Flux bootstrap created
    namespace: flux-system
EOF
```

---

### Step 6 — Commit and Verify

```bash
git add clusters/production/
git commit -m 'feat: add tenant namespace structure and controller patches'
git push

# Reconcile
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source

# Verify namespaces exist
kubectl get namespaces | grep tenant

# Expected:
# tenant-a   Active   1m
# tenant-b   Active   1m

# Confirm labels
kubectl get namespace tenant-a --show-labels
```

### ✅ Lab 1 Checklist

- [x] `kustomize-controller-patch.yaml` created and referenced in `flux-system/kustomization.yaml`
- [x] `clusters/production/tenants/tenant-a/` and `tenant-b/` directories created
- [x] Namespace YAML files committed for both tenants
- [x] `tenants-sync.yaml` added so Flux watches the tenants folder
- [x] Both namespaces confirmed with `kubectl get namespaces`

---

## Lab 2 — RBAC Configuration for Flux

**Objective:** Create Roles and RoleBindings that restrict each tenant's Flux reconciler to their own namespace only.

**Estimated Time:** 30 minutes

### How RBAC Scoping Works

```
Flux kustomize-controller
  └── reads Kustomization (spec.serviceAccountName = tenant-a-reconciler)
      └── impersonates ServiceAccount/tenant-a-reconciler in namespace tenant-a
          └── bound to Role/tenant-a-role (namespace: tenant-a only)
              └── can only create/update/delete resources in tenant-a
```

---

### Step 1 — Create the Role and RoleBinding for Tenant A

```bash
cat > clusters/production/tenants/tenant-a/rbac.yaml <<EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-a-role
  namespace: tenant-a
rules:
  # Core resources
  - apiGroups: [""]
    resources:
      - configmaps
      - persistentvolumeclaims
      - pods
      - secrets
      - serviceaccounts
      - services
    verbs: ["*"]
  # Deployments
  - apiGroups: ["apps"]
    resources:
      - daemonsets
      - deployments
      - replicasets
      - statefulsets
    verbs: ["*"]
  # Autoscaling
  - apiGroups: ["autoscaling"]
    resources:
      - horizontalpodautoscalers
    verbs: ["*"]
  # Jobs
  - apiGroups: ["batch"]
    resources:
      - cronjobs
      - jobs
    verbs: ["*"]
  # Ingress
  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-a-rolebinding
  namespace: tenant-a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: tenant-a-role
subjects:
  - kind: ServiceAccount
    name: tenant-a-reconciler
    namespace: tenant-a
EOF
```

---

### Step 2 — Create the Role and RoleBinding for Tenant B

```bash
cat > clusters/production/tenants/tenant-b/rbac.yaml <<EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-b-role
  namespace: tenant-b
rules:
  - apiGroups: [""]
    resources:
      - configmaps
      - persistentvolumeclaims
      - pods
      - secrets
      - serviceaccounts
      - services
    verbs: ["*"]
  - apiGroups: ["apps"]
    resources:
      - daemonsets
      - deployments
      - replicasets
      - statefulsets
    verbs: ["*"]
  - apiGroups: ["autoscaling"]
    resources:
      - horizontalpodautoscalers
    verbs: ["*"]
  - apiGroups: ["batch"]
    resources:
      - cronjobs
      - jobs
    verbs: ["*"]
  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-b-rolebinding
  namespace: tenant-b
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: tenant-b-role
subjects:
  - kind: ServiceAccount
    name: tenant-b-reconciler
    namespace: tenant-b
EOF
```

---

### Step 3 — Commit and Verify RBAC

```bash
git add clusters/production/tenants/
git commit -m 'feat: add RBAC roles and bindings for tenant-a and tenant-b'
git push

flux reconcile kustomization tenants --with-source

# List roles in each namespace
kubectl get role,rolebinding -n tenant-a
kubectl get role,rolebinding -n tenant-b

# Verify tenant-a-reconciler CAN deploy in tenant-a
kubectl auth can-i create deployments \
  --namespace=tenant-a \
  --as=system:serviceaccount:tenant-a:tenant-a-reconciler
# Expected: yes

# Verify tenant-a-reconciler CANNOT deploy in tenant-b
kubectl auth can-i create deployments \
  --namespace=tenant-b \
  --as=system:serviceaccount:tenant-a:tenant-a-reconciler
# Expected: no
```

### ✅ Lab 2 Checklist

- [x] `rbac.yaml` created for both tenants with namespace-scoped Role
- [x] RoleBinding links Role to each tenant's ServiceAccount
- [x] `kubectl auth can-i` confirms access within own namespace
- [x] Cross-namespace access confirmed as denied

---

## Lab 3 — Service Accounts and Permissions

**Objective:** Create ServiceAccounts per tenant, add GitRepository sources, and wire Flux Kustomizations with `spec.serviceAccountName` to enforce impersonation.

**Estimated Time:** 30 minutes

### Step 1 — Create ServiceAccounts

```bash
# ServiceAccount for tenant-a
cat > clusters/production/tenants/tenant-a/serviceaccount.yaml <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-a-reconciler
  namespace: tenant-a
  labels:
    toolkit.fluxcd.io/tenant: tenant-a
EOF

# ServiceAccount for tenant-b
cat > clusters/production/tenants/tenant-b/serviceaccount.yaml <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-b-reconciler
  namespace: tenant-b
  labels:
    toolkit.fluxcd.io/tenant: tenant-b
EOF
```

---

### Step 2 — Create GitRepository Sources per Tenant Namespace

Each tenant's `GitRepository` lives in their own namespace. Because `--no-cross-namespace-refs=true` is set, a Kustomization in `tenant-a` can only reference a GitRepository also in `tenant-a`.

```bash
# GitRepository for tenant-a — points to their own app repo
cat > clusters/production/tenants/tenant-a/gitrepository.yaml <<EOF
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: tenant-a
  namespace: tenant-a
spec:
  interval: 1m
  url: https://gitlab.com/your-org/tenant-a-gitops
  branch: main
  secretRef:
    name: tenant-a-gitlab-token
EOF

# GitRepository for tenant-b
cat > clusters/production/tenants/tenant-b/gitrepository.yaml <<EOF
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: tenant-b
  namespace: tenant-b
spec:
  interval: 1m
  url: https://gitlab.com/your-org/tenant-b-gitops
  branch: main
  secretRef:
    name: tenant-b-gitlab-token
EOF
```

---

### Step 3 — Create Flux Kustomizations with Impersonation

`spec.serviceAccountName` is what enforces the permission boundary — Flux applies the tenant's manifests **as** the tenant's ServiceAccount, not as cluster-admin.

```bash
# Flux Kustomization for tenant-a
cat > clusters/production/tenants/tenant-a/sync.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: tenant-a
  namespace: tenant-a
spec:
  serviceAccountName: tenant-a-reconciler   # Impersonate this SA on every apply
  interval: 5m
  path: ./apps                              # Path inside tenant-a-gitops repo
  prune: true
  sourceRef:
    kind: GitRepository
    name: tenant-a
    namespace: tenant-a                     # Must match GitRepository namespace
EOF

# Flux Kustomization for tenant-b
cat > clusters/production/tenants/tenant-b/sync.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: tenant-b
  namespace: tenant-b
spec:
  serviceAccountName: tenant-b-reconciler
  interval: 5m
  path: ./apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: tenant-b
    namespace: tenant-b
EOF
```

---

### Step 4 — Create GitLab Auth Secrets (Imperative)

These secrets hold each tenant's GitLab deploy token. They are created with `kubectl` — **not committed to Git** — because they contain credentials.

```bash
# Create deploy token secret for tenant-a
kubectl create secret generic tenant-a-gitlab-token \
  --namespace=tenant-a \
  --from-literal=username=tenant-a-deploy \
  --from-literal=password=<TENANT_A_DEPLOY_TOKEN>

# Create deploy token secret for tenant-b
kubectl create secret generic tenant-b-gitlab-token \
  --namespace=tenant-b \
  --from-literal=username=tenant-b-deploy \
  --from-literal=password=<TENANT_B_DEPLOY_TOKEN>

# Verify secrets exist
kubectl get secret tenant-a-gitlab-token -n tenant-a
kubectl get secret tenant-b-gitlab-token -n tenant-b
```

> **GitLab deploy token location:** Project → Settings → Repository → Deploy tokens → Create deploy token with `read_repository` scope.

---

### Step 5 — Commit Everything and Verify

```bash
git add clusters/production/tenants/
git commit -m 'feat: add ServiceAccounts, GitRepositories and Kustomizations for tenants'
git push

# Reconcile
flux reconcile kustomization tenants --with-source

# Watch all kustomizations across namespaces
flux get kustomizations -A

# Expected output:
# NAMESPACE   NAME      REVISION      SUSPENDED  READY  MESSAGE
# flux-system tenants   main/abc123   False      True   Applied revision: main/abc123
# tenant-a    tenant-a  main/def456   False      True   Applied revision: main/def456
# tenant-b    tenant-b  main/ghi789   False      True   Applied revision: main/ghi789

# Verify GitRepository sources are syncing
flux get sources git -A

# Check impersonation is set correctly
kubectl get kustomization tenant-a -n tenant-a \
  -o jsonpath='{.spec.serviceAccountName}'
# Expected: tenant-a-reconciler

# Final isolation check
kubectl auth can-i create deployments \
  --namespace=tenant-a \
  --as=system:serviceaccount:tenant-a:tenant-a-reconciler
# Expected: yes

kubectl auth can-i create deployments \
  --namespace=tenant-b \
  --as=system:serviceaccount:tenant-a:tenant-a-reconciler
# Expected: no
```

---

### Step 6 — Tenant App Repo Structure

Each tenant's GitOps repo (`tenant-a-gitops`) must have the path Flux is watching (`./apps`):

```
tenant-a-gitops/
└── apps/
    ├── kustomization.yaml
    ├── deployment.yaml
    └── service.yaml
```

`apps/kustomization.yaml` inside the tenant repo:

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

`apps/deployment.yaml` inside the tenant repo:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: tenant-a          # Must match tenant's own namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:1.25
          ports:
            - containerPort: 80
```

> ⚠️ If a tenant tries to deploy into a different namespace, the Role will deny it and Flux will report a reconciliation error in `flux get kustomizations`.

### ✅ Lab 3 Checklist

- [x] ServiceAccounts created in respective tenant namespaces
- [x] GitRepository sources created per tenant namespace
- [x] Flux Kustomizations use `spec.serviceAccountName` for impersonation
- [x] GitLab deploy token secrets created imperatively (not in Git)
- [x] All tenant Kustomizations showing `Ready: True`
- [x] Tenant app repo has `./apps/` path with a `kustomization.yaml`

---

## Quick Reference Cheat Sheet

| Task | Command |
|---|---|
| Reconcile platform repo | `flux reconcile kustomization flux-system --with-source` |
| Reconcile tenants folder | `flux reconcile kustomization tenants --with-source` |
| Watch all kustomizations | `flux get kustomizations -A` |
| Watch all git sources | `flux get sources git -A` |
| Test tenant RBAC (allow) | `kubectl auth can-i create deployments --namespace=tenant-a --as=system:serviceaccount:tenant-a:tenant-a-reconciler` |
| Test tenant RBAC (deny) | `kubectl auth can-i create deployments --namespace=tenant-b --as=system:serviceaccount:tenant-a:tenant-a-reconciler` |
| View tenant events | `flux events --for Kustomization/tenant-a -n tenant-a` |
| List roles per namespace | `kubectl get role,rolebinding -n tenant-a` |
| Watch controller logs | `kubectl logs -n flux-system -l app=kustomize-controller -f` |
| Describe ServiceAccount | `kubectl describe serviceaccount tenant-a-reconciler -n tenant-a` |

---

### Multi-Tenancy Security Checklist

- [ ] `--no-cross-namespace-refs=true` patched onto kustomize-controller
- [ ] Every tenant Kustomization has `spec.serviceAccountName` set
- [ ] Every ServiceAccount has a Role scoped to its own namespace only
- [ ] No ClusterRoleBindings granted to tenant ServiceAccounts
- [ ] GitLab deploy tokens are namespace-scoped secrets, not committed to Git
- [ ] `prune: true` set on all tenant Kustomizations
- [ ] Cross-namespace deny tested with `kubectl auth can-i`
