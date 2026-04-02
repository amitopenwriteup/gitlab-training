# Flux CD: The Comprehensive Guide
### Intermediate to Advanced GitOps with Flux v2

> **Version:** Flux v2.8+ | **Audience:** Intermediate–Advanced Kubernetes Engineers
> **Source:** Based on official FluxCD documentation at https://fluxcd.io/flux/guides/

---

## Table of Contents

1. [Introduction to Flux v2 and GitOps Principles](#1-introduction-to-flux-v2-and-gitops-principles)
2. [Architecture: The GitOps Toolkit](#2-architecture-the-gitops-toolkit)
3. [Bootstrap and Installation Deep Dive](#3-bootstrap-and-installation-deep-dive)
4. [Repository Structure Strategies](#4-repository-structure-strategies)
5. [The Kustomize Controller](#5-the-kustomize-controller)
6. [Managing Helm Releases with Flux](#6-managing-helm-releases-with-flux)
7. [Secrets Management: SOPS and Sealed Secrets](#7-secrets-management-sops-and-sealed-secrets)
8. [Webhook Receivers and Event-Driven Reconciliation](#8-webhook-receivers-and-event-driven-reconciliation)
9. [Image Automation: Automated Container Updates](#9-image-automation-automated-container-updates)
10. [Sortable Image Tags for Automation](#10-sortable-image-tags-for-automation)
11. [Multi-Tenancy and Access Control](#11-multi-tenancy-and-access-control)
12. [Monitoring, Alerting and Observability](#12-monitoring-alerting-and-observability)
13. [OCI Artifacts and the Source Controller](#13-oci-artifacts-and-the-source-controller)
14. [Advanced Kustomization Patterns](#14-advanced-kustomization-patterns)
15. [Progressive Delivery with Flagger](#15-progressive-delivery-with-flagger)
16. [Troubleshooting and Debugging](#16-troubleshooting-and-debugging)
17. [Security Best Practices](#17-security-best-practices)
18. [Reference: Essential CLI Commands](#18-reference-essential-cli-commands)

---

## 1. Introduction to Flux v2 and GitOps Principles

### What is GitOps?

GitOps is an operational model where Git is the single source of truth for declarative infrastructure and application state. The core principles are:

- **Declarative:** All desired system state is expressed declaratively in Git
- **Versioned and immutable:** Desired state is stored in a versioned system that enforces immutability
- **Pulled automatically:** Software agents automatically pull the desired state from the source
- **Continuously reconciled:** Software agents continuously observe actual system state and attempt to apply the desired state

This model fundamentally shifts how deployments happen: instead of pushing changes into a cluster imperatively (via `kubectl apply`), a reconciliation agent inside the cluster continuously pulls configuration from Git and converges the cluster to the declared state.

### Why Flux v2?

Flux v2 (the current generation) is a complete rewrite over Flux v1, designed from the ground up as a set of composable, extensible Kubernetes controllers using controller-runtime patterns. Key advantages over its predecessor include:

- **Controller-per-concern architecture:** Each GitOps function is a separate, independently upgradeable controller
- **Multi-tenancy:** Native namespace-level isolation with RBAC-aware reconciliation
- **Multi-source support:** Git, Helm repositories, OCI registries, S3-compatible buckets
- **Extensibility:** The GitOps Toolkit APIs can be consumed by custom operators
- **Drift detection:** Continuous detection and correction of manual changes in the cluster

Flux is a graduated CNCF project and is actively maintained, with a strong enterprise adoption track record.

### The Reconciliation Loop

The fundamental operation of Flux is the reconciliation loop. Each controller watches its respective custom resources and periodically (or on-demand) compares the desired state from the source with the actual state in the cluster. When a drift is detected, the controller applies the necessary changes.

```
   Git Repository
        │
        │  clone / pull
        ▼
  Source Controller ──── stores artifact ────▶ In-cluster artifact cache
        │
        │  artifact ready event
        ▼
  Kustomize / Helm Controller
        │
        │  compare desired ↔ actual
        ▼
  Apply / Upgrade ──────────────────────────▶ Kubernetes API Server
        │
        │  record status
        ▼
  Conditions / Events / Metrics
```

---

## 2. Architecture: The GitOps Toolkit

Flux v2 is built on the GitOps Toolkit, a set of composable APIs and controllers. Understanding each component is essential for advanced operations.

### Source Controller

The Source Controller manages the acquisition of artifacts from external sources and makes them available to other controllers. It handles:

- **GitRepository:** Clones a Git repository and exposes a tarball artifact
- **HelmRepository:** Indexes a Helm repository (HTTP/S or OCI) and exposes chart metadata
- **HelmChart:** Fetches and packages a Helm chart from a repository or Git source
- **OCIRepository:** Pulls an OCI image (artifact) and exposes it as a tarball
- **Bucket:** Syncs an S3-compatible bucket and exposes its contents

The Source Controller implements content-addressable storage: artifacts are only re-fetched when the underlying content changes. This makes it efficient even with short polling intervals.

### Kustomize Controller

The Kustomize Controller applies Kubernetes manifests from a source artifact using Kustomize. It handles:

- Running `kustomize build` on a path within a source artifact
- Applying the resulting manifests with server-side apply
- Health checking deployed resources using status conditions
- Garbage collecting resources that were removed from the source
- Decrypting SOPS-encrypted files at apply time

### Helm Controller

The Helm Controller manages Helm releases declaratively. It handles:

- Installing, upgrading, and rolling back Helm releases
- Testing Helm releases via `helm test`
- Detecting and correcting drift (with optional strict drift detection)
- Cross-namespace source references

### Notification Controller

The Notification Controller handles inbound and outbound event processing:

- **Receivers:** Accept webhook payloads from Git forges (GitHub, GitLab, etc.) to trigger immediate reconciliation
- **Providers:** Send alerts to Slack, Teams, PagerDuty, Opsgenie, etc. when reconciliation events occur
- **Alerts:** Define which events from which sources are routed to which providers

### Image Reflector and Image Automation Controllers

These two controllers work together to close the automation loop for container images:

- **Image Reflector Controller:** Scans container registries and reflects discovered tags into `ImageRepository` and `ImagePolicy` custom resources
- **Image Automation Controller:** Updates YAML files in Git based on the latest image tag matching a policy

### Controller Interaction Model

```
                     ┌─────────────────────────────┐
                     │       Git Repository         │
                     │   (single source of truth)   │
                     └──────────────┬──────────────┘
                                    │ poll / webhook
                                    ▼
                     ┌─────────────────────────────┐
                     │      Source Controller       │
                     │  GitRepository  HelmRepo     │
                     │  OCIRepository  Bucket       │
                     └──────┬──────────────┬────────┘
                            │              │
                  artifact  │              │ artifact
                            ▼              ▼
              ┌─────────────────┐  ┌──────────────────┐
              │  Kustomize Ctrl │  │   Helm Controller │
              │  Kustomization  │  │   HelmRelease     │
              └─────────────────┘  └──────────────────┘
                            │              │
                            ▼              ▼
                     ┌─────────────────────────────┐
                     │     Kubernetes API Server    │
                     └─────────────────────────────┘
                                    │
                                    ▼
              ┌────────────────────────────────────────┐
              │         Notification Controller         │
              │   Receivers  ──  Providers  ──  Alerts  │
              └────────────────────────────────────────┘
```

---

## 3. Bootstrap and Installation Deep Dive

Bootstrap is the process of installing Flux into a cluster and configuring it to manage itself from a Git repository. This is the preferred installation method because Flux's own configuration becomes subject to GitOps reconciliation.

### Prerequisites

```bash
# Verify cluster connectivity
kubectl cluster-info

# Check Flux prerequisites (checks Kubernetes version, CRD support, etc.)
flux check --pre

# Export your Git provider token
export GITHUB_TOKEN=<your-pat>
export GITHUB_USER=<your-username>
```

### Bootstrapping with GitHub

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

What this does:
1. Creates the GitHub repository if it doesn't exist
2. Generates Flux component manifests
3. Pushes those manifests to the repository at the specified path
4. Applies the manifests to the cluster
5. Flux then reconciles itself, reading from the repository

### Bootstrapping with GitLab

```bash
flux bootstrap gitlab \
  --owner=<group-or-username> \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/my-cluster \
  --token-auth
```

### Bootstrapping with a Generic Git Server

For servers without a supported provider-specific flag (e.g., Gitea, Bitbucket Server, Azure DevOps with custom domains):

```bash
flux bootstrap git \
  --url=ssh://git@<host>/<org>/fleet-infra \
  --branch=main \
  --path=clusters/my-cluster
```

### The Bootstrap Output Structure

After bootstrap, the repository will contain:

```
clusters/
└── my-cluster/
    └── flux-system/
        ├── gotk-components.yaml    # All Flux CRDs and controllers
        ├── gotk-sync.yaml          # GitRepository + Kustomization pointing to this repo
        └── kustomization.yaml      # Kustomize config for the above
```

The `gotk-sync.yaml` is the key file — it creates a `GitRepository` pointing to the bootstrap repo, and a `Kustomization` that reads from `clusters/my-cluster`. This means any file you add to that path will be automatically reconciled.

### Customizing Bootstrap

To add extra components at bootstrap time (e.g., the image automation controllers, which are optional):

```bash
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

### Post-Bootstrap Verification

```bash
# Check all Flux components are running
flux check

# Watch all Flux resources
flux get all

# Check the bootstrapped GitRepository
flux get sources git

# Check the bootstrapped Kustomization
flux get kustomizations
```

---

## 4. Repository Structure Strategies

Choosing the right repository structure is one of the most impactful architectural decisions for a GitOps implementation. There is no single right answer; the best choice depends on team size, security requirements, and deployment complexity.

### Strategy 1: Monorepo

A monorepo stores all Kubernetes manifests — for every environment and every application — in a single Git repository. All team members can read and write to this repository.

**Canonical Directory Layout (Kustomize Overlays):**

```
fleet-infra/
├── apps/
│   ├── base/
│   │   ├── podinfo/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── nginx/
│   │       └── ...
│   ├── staging/
│   │   ├── kustomization.yaml        # patches staging-specific values
│   │   └── podinfo-values.yaml
│   └── production/
│       ├── kustomization.yaml        # patches prod-specific values
│       └── podinfo-values.yaml
├── infrastructure/
│   ├── base/
│   │   ├── cert-manager/
│   │   ├── ingress-nginx/
│   │   └── redis/
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
└── clusters/
    ├── staging/
    │   ├── flux-system/              # Flux bootstrap files
    │   ├── infrastructure.yaml       # Kustomization: deploys infrastructure/staging
    │   └── apps.yaml                 # Kustomization: deploys apps/staging
    └── production/
        ├── flux-system/
        ├── infrastructure.yaml       # depends on: [infrastructure-controllers]
        └── apps.yaml                 # depends on: [infrastructure]
```

The `clusters/staging/infrastructure.yaml` Kustomization:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/staging
  prune: true
  wait: true
  timeout: 5m
```

The `clusters/staging/apps.yaml` Kustomization with dependency ordering:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: infrastructure       # apps will not reconcile until infrastructure is healthy
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging
  prune: true
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: podinfo
      namespace: default
```

**When to use a monorepo:** Small to medium teams, single platform team, simpler access control requirements.

**Tradeoffs:**
- All team members can read the production config (including secrets if not encrypted)
- All environment changes are visible in one diff
- Simpler tooling — one repository to clone, one set of PR workflows

### Strategy 2: Repo Per Environment

Similar to the monorepo but with a separate repository for each target environment (staging, production). Access to the production repository is restricted to a subset of engineers.

**Repository layout:**

```
fleet-infra-staging/      # accessible by all engineers
├── apps/
├── infrastructure/
└── clusters/staging/

fleet-infra-production/   # restricted access
├── apps/
├── infrastructure/
└── clusters/production/
```

**When to use per-environment repos:**
- Regulatory or compliance requirements demand access restrictions on production configuration
- Large teams where unintentional production changes are a risk
- Auditable production-only change history

**Tradeoffs:**
- Promoting infrastructure changes across environments requires copying between repositories (more work than a single PR)
- PR review scope is smaller per repo, which reduces error surface in production reviews
- More operational overhead (two repositories to maintain)

### Strategy 3: Repo Per Team

The platform team owns a central repository that bootstraps the cluster and onboards dev teams. Each dev team owns their own repository with their application manifests.

**Platform team repository:**

```
fleet-infra/                        # platform team repo
├── clusters/
│   ├── production/
│   │   ├── flux-system/
│   │   ├── team1.yaml              # Kustomization pointing to team1's repo
│   │   └── team2.yaml
│   └── staging/
│       ├── flux-system/
│       ├── team1.yaml
│       └── team2.yaml
├── infrastructure/
│   ├── base/
│   ├── production/
│   └── staging/
└── teams/
    ├── team1/
    │   └── rbac.yaml               # RBAC permissions for team1's namespace
    └── team2/
        └── rbac.yaml
```

The `team1.yaml` entry in the platform repo creates a GitRepository and Kustomization for team1:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: team1
  namespace: team1
spec:
  interval: 1m
  url: https://github.com/org/team1-apps
  ref:
    branch: main
  secretRef:
    name: team1-git-auth
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team1-apps
  namespace: team1
spec:
  serviceAccountName: kustomize-controller  # scoped service account
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: team1
  path: ./deploy/production
  prune: true
  targetNamespace: team1
```

**Team1 application repository:**

```
team1-apps/
├── src/                    # application source code
└── deploy/
    ├── base/
    ├── staging/
    └── production/
```

**When to use per-team repos:**
- Platform-as-a-service model with clear team boundaries
- Dev teams need autonomy over their own delivery pipeline
- Different release cadences per team
- Regulatory isolation requirements between tenants

### Strategy 4: Repo Per App

Each application lives in its own repository, containing both source code and deployment manifests. The platform config repo holds "pointers" to each app repo using Flux `GitRepository` resources.

**App repository structure (plain manifests):**

```
my-app/
├── src/
│   └── main.py
├── Dockerfile
└── deploy/
    └── manifests/
        ├── deployment.yaml
        ├── service.yaml
        └── kustomization.yaml
```

**Pointer from the config repo (semver pinning):**

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/my-app
  ref:
    semver: ">=1.0.0 <2.0.0"    # only reconcile tagged releases in this semver range
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./deploy/manifests
  prune: true
  patches:
    - patch: |
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-app
        spec:
          replicas: 3
      target:
        kind: Deployment
        name: my-app
```

**App repository structure (Helm chart):**

```
my-app/
├── src/
└── chart/
    ├── Chart.yaml
    ├── templates/
    ├── values.yaml
    └── values-production.yaml
```

---

## 5. The Kustomize Controller

The Kustomize Controller is the core reconciliation engine for plain manifests. It extends standard Kustomize with additional capabilities designed for GitOps workflows.

### The Kustomization Resource

A `Kustomization` custom resource defines a set of Kubernetes manifests to be reconciled:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 5m                    # reconcile every 5 minutes even without changes
  retryInterval: 2m               # retry on transient failures after 2 minutes
  timeout: 3m                     # fail if reconciliation takes longer than 3 minutes
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging/podinfo
  prune: true                     # delete resources removed from the source
  wait: true                      # block until all health checks pass
  force: false                    # do not recreate immutable resources on conflict
  targetNamespace: podinfo        # override namespace for all resources
  serviceAccountName: podinfo-sa  # impersonate this SA for apply operations
```

### Dependencies Between Kustomizations

The `dependsOn` field allows you to enforce ordering between Kustomizations. A Kustomization with dependencies will not begin reconciling until all listed dependencies are in a `Ready` state:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: infrastructure-controllers    # wait for CRDs to be installed first
    - name: infrastructure-configs        # wait for configmaps/secrets
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/production
  prune: true
```

### Health Checks

Beyond Kubernetes conditions, you can define custom health checks for resources that Flux should wait for:

```yaml
spec:
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: podinfo
      namespace: podinfo
    - apiVersion: helm.toolkit.fluxcd.io/v2
      kind: HelmRelease
      name: redis
      namespace: redis
```

For `wait: true`, Flux uses the health check logic from kstatus to determine readiness. This supports all standard Kubernetes resources and many custom ones.

### Kustomize Patches in Kustomizations

Flux's `Kustomization` supports inline strategic merge patches and JSON6902 patches without needing a separate `kustomization.yaml`:

```yaml
spec:
  patches:
    # Strategic merge patch — increase replicas in production
    - patch: |
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: podinfo
        spec:
          replicas: 5
      target:
        kind: Deployment
        name: podinfo

    # JSON6902 patch — add a resource limit
    - patch: |
        - op: add
          path: /spec/template/spec/containers/0/resources
          value:
            limits:
              memory: 256Mi
              cpu: 500m
      target:
        kind: Deployment
        labelSelector: "app=podinfo"
```

### Substitution Variables

Flux supports variable substitution using `postBuild.substitute` and `postBuild.substituteFrom`. This allows you to parameterize manifests without duplicating them:

```yaml
spec:
  postBuild:
    substitute:
      cluster_region: "us-east-1"
      cluster_env: "production"
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars           # key-value pairs from a ConfigMap
      - kind: Secret
        name: cluster-secrets        # sensitive substitutions from a Secret
        optional: true
```

In your manifests, reference variables with `${VAR_NAME}`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    cluster.region: ${cluster_region}
    cluster.env: ${cluster_env}
spec:
  template:
    spec:
      containers:
        - name: my-app
          env:
            - name: CLUSTER_ENV
              value: ${cluster_env}
```

### Garbage Collection (Prune)

With `prune: true`, Flux tracks all objects applied from a given Kustomization using a `kustomize.toolkit.fluxcd.io/name` label. When a resource is removed from the source, Flux will delete it from the cluster on the next reconciliation.

**Caution:** `prune: true` is powerful but dangerous if misconfigured. Always test changes in staging before promoting to production.

### Server-Side Apply

Flux uses Kubernetes Server-Side Apply (SSA) to avoid conflicts with other controllers (like HPA, KEDA, or mutation webhooks). SSA means each field has a defined "manager" — Flux owns the fields it declared, and other controllers can own additional fields without conflict.

---

## 6. Managing Helm Releases with Flux

The Helm Controller provides a declarative, GitOps-native way to manage Helm releases. Rather than running `helm upgrade` imperatively, you describe the desired Helm release state in a `HelmRelease` resource.

### Defining a HelmRepository Source

Before creating a `HelmRelease`, you need a `HelmRepository` source:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 12h           # refresh the index every 12 hours
  url: https://charts.bitnami.com/bitnami
```

For OCI Helm repositories (e.g., GHCR, ECR, GCR):

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 5m
  type: oci
  url: oci://ghcr.io/stefanprodan/charts
```

For private repositories requiring authentication:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: private-charts
  namespace: flux-system
spec:
  interval: 10m
  url: https://charts.example.com
  secretRef:
    name: helm-repo-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: helm-repo-auth
  namespace: flux-system
type: Opaque
stringData:
  username: my-username
  password: my-password
```

### Creating a HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: default
spec:
  interval: 5m
  chart:
    spec:
      chart: podinfo
      version: ">=6.0.0 <7.0.0"   # semver constraint — auto-upgrades within this range
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
      interval: 1m                  # check for new chart versions every minute
  values:
    replicaCount: 2
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
    service:
      type: ClusterIP
      port: 9898
```

### Values Overrides and Merging

Flux supports multiple levels of values override, applied in order (last wins):

```yaml
spec:
  values:
    # inline values — highest priority
    replicaCount: 3

  valuesFrom:
    # values from a ConfigMap — merged before inline values
    - kind: ConfigMap
      name: podinfo-values
      valuesKey: values.yaml      # default key is "values.yaml"

    # values from a Secret — for sensitive overrides
    - kind: Secret
      name: podinfo-secret-values
      valuesKey: prod-values.yaml
      optional: true              # don't fail if the secret doesn't exist
```

### Install and Upgrade Remediation

Flux can automatically remediate failed Helm operations:

```yaml
spec:
  install:
    remediation:
      retries: 3                  # retry install up to 3 times
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true  # rollback on the last failed upgrade attempt
    cleanupOnFail: true           # delete new resources on upgrade failure
  rollback:
    timeout: 5m
    cleanupOnFail: true
```

### Drift Detection and Correction

By default, Flux only reconciles HelmReleases when the `HelmRelease` object or the chart/values change. To enable continuous drift detection (detecting and correcting manual `helm` or `kubectl` changes):

Enable it cluster-wide at bootstrap:

```bash
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --feature-gates=HelmAllowCrossNamespaceRef=true \
  ...
```

Or configure it per-release with `driftDetection`:

```yaml
spec:
  driftDetection:
    mode: enabled        # "enabled" | "warn" | "disabled"
    ignore:
      - paths:
          - /spec/replicas    # ignore replica count (managed by HPA)
        target:
          kind: Deployment
```

### Multi-Environment Helm Values Strategy

A common pattern is to have base values and environment-specific overrides using a Kustomize overlay:

**Base HelmRelease (apps/base/podinfo/helmrelease.yaml):**

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
spec:
  interval: 5m
  chart:
    spec:
      chart: podinfo
      version: "6.x"
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  values:
    replicaCount: 1
```

**Staging overlay (apps/staging/podinfo/kustomization.yaml):**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/podinfo
patches:
  - patch: |
      - op: replace
        path: /spec/values/replicaCount
        value: 2
    target:
      kind: HelmRelease
      name: podinfo
```

**Production overlay (apps/production/podinfo/kustomization.yaml):**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/podinfo
patches:
  - patch: |
      - op: replace
        path: /spec/values/replicaCount
        value: 5
    target:
      kind: HelmRelease
      name: podinfo
```

### Cross-Namespace References

HelmReleases can reference sources from other namespaces (requires the `HelmAllowCrossNamespaceRef` feature gate):

```yaml
spec:
  chart:
    spec:
      chart: podinfo
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system      # source is in flux-system, release in default
```

### Testing Helm Releases

Flux can run `helm test` after each install or upgrade:

```yaml
spec:
  test:
    enable: true
    ignoreFailures: false    # fail the HelmRelease if tests fail
```

---

## 7. Secrets Management: SOPS and Sealed Secrets

One of the most important considerations in GitOps is how to store secrets. Since all configuration is in Git, secrets must never be stored in plaintext. Flux natively supports two primary approaches: Mozilla SOPS and Bitnami Sealed Secrets.

### Approach 1: Mozilla SOPS

SOPS (Secrets OPerationS) is a tool that encrypts the values (not the keys) of YAML files. Flux's Kustomize Controller has built-in support for decrypting SOPS-encrypted files at reconciliation time.

SOPS supports multiple key providers: Age, PGP, AWS KMS, GCP KMS, Azure Key Vault, and HashiCorp Vault.

#### Setting Up SOPS with Age (Recommended)

Age is the recommended modern encryption tool for SOPS. It is simple, fast, and has no complex key infrastructure.

**Step 1: Generate an Age key pair:**

```bash
# Install age
brew install age    # macOS
# or: apt install age

# Generate key pair
age-keygen -o age.agekey

# Output looks like:
# Public key: age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xjx35dp2sqlfgpl
# AGE-SECRET-KEY-1...
```

**Step 2: Create a Kubernetes secret with the private key:**

```bash
cat age.agekey |
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

**Step 3: Configure the Kustomize Controller to use SOPS:**

Add a `.sops.yaml` file at the root of your repository:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: >-
      age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xjx35dp2sqlfgpl
```

**Step 4: Encrypt a secret file:**

```bash
# Create a plaintext secret
cat > secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
stringData:
  api-key: "my-super-secret-value"
  db-password: "another-secret"
EOF

# Encrypt it using SOPS
sops --encrypt --age age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xjx35dp2sqlfgpl \
  --encrypted-regex '^(data|stringData)$' \
  secret.yaml > secret.enc.yaml

# The encrypted file can be safely committed to Git
git add secret.enc.yaml
git commit -m "feat: add encrypted secret"
```

The encrypted file looks like:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
stringData:
  api-key: ENC[AES256_GCM,data:abc123...,type:str]
  db-password: ENC[AES256_GCM,data:xyz789...,type:str]
sops:
  kms: []
  age:
    - recipient: age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xjx35dp2sqlfgpl
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        ...
        -----END AGE ENCRYPTED FILE-----
  ...
```

**Step 5: Configure the Kustomization to use SOPS decryption:**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age    # the Kubernetes secret containing the Age private key
```

#### SOPS with AWS KMS

For cloud environments, KMS-based SOPS avoids storing any private key material in Kubernetes. The controller's pod identity (IRSA/Workload Identity) is used to access the KMS key.

```yaml
# .sops.yaml for AWS KMS
creation_rules:
  - path_regex: clusters/production/.*
    kms: arn:aws:kms:us-east-1:123456789012:key/mrk-abc123
  - path_regex: clusters/staging/.*
    kms: arn:aws:kms:us-east-1:123456789012:key/mrk-def456
```

Configure the Kustomize Controller with IRSA (AWS IAM Roles for Service Accounts):

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  decryption:
    provider: sops
    # No secretRef needed — controller uses pod's IAM role
```

### Approach 2: Sealed Secrets

Sealed Secrets uses asymmetric cryptography where a controller running in the cluster holds the private key, and users encrypt secrets using the corresponding public key. The encrypted `SealedSecret` resource can be committed to Git.

#### Installing the Sealed Secrets Controller

Via Flux HelmRelease:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: sealed-secrets
  namespace: flux-system
spec:
  interval: 1h
  url: https://bitnami-labs.github.io/sealed-secrets
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: sealed-secrets-controller
  namespace: flux-system
spec:
  interval: 1h
  releaseName: sealed-secrets-controller
  chart:
    spec:
      chart: sealed-secrets
      version: ">=2.7.0"
      sourceRef:
        kind: HelmRepository
        name: sealed-secrets
        namespace: flux-system
      interval: 1h
```

#### Creating a SealedSecret

```bash
# Install kubeseal CLI
brew install kubeseal

# Fetch the public key from the controller
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=flux-system \
  > pub-sealed-secrets.pem

# Create and encrypt a secret
kubectl create secret generic my-secret \
  --namespace=default \
  --from-literal=api-key=my-super-secret \
  --dry-run=client \
  -o yaml | \
  kubeseal --format=yaml \
  --cert=pub-sealed-secrets.pem \
  > my-sealed-secret.yaml

# Commit the SealedSecret (safe to store in Git)
git add my-sealed-secret.yaml
git commit -m "feat: add sealed secret"
```

The `SealedSecret` is decrypted automatically by the controller when applied to the cluster, creating the underlying `Secret`.

#### Key Rotation with Sealed Secrets

The Sealed Secrets controller automatically rotates its encryption key every 30 days by default, keeping older keys to decrypt existing secrets. To manually trigger rotation:

```bash
# Force key rotation
kubectl label secret -n flux-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  sealedsecrets.bitnami.com/sealed-secrets-key=compromised

# Restart controller to generate a new key
kubectl rollout restart deployment/sealed-secrets-controller -n flux-system
```

### Choosing Between SOPS and Sealed Secrets

| Dimension | SOPS | Sealed Secrets |
|---|---|---|
| Key management | External (Age, KMS, PGP) | In-cluster (controller holds key) |
| Cloud KMS support | Native (AWS, GCP, Azure) | Via external key sources |
| Offline decryption | Yes (with private key) | No (requires cluster) |
| Multi-cluster | Separate keys per cluster | Separate controllers per cluster |
| Key rotation | Manual re-encryption | Automatic (30-day default) |
| Secret scope | Any YAML file value | Kubernetes Secret only |
| Air-gapped clusters | Excellent (Age) | Good (controller is self-contained) |

---

## 8. Webhook Receivers and Event-Driven Reconciliation

By default, Flux polls sources on a configurable interval (typically 1–10 minutes). For faster feedback loops — especially in active development — webhook receivers allow external systems to trigger immediate reconciliation.

### How Receivers Work

The Notification Controller exposes an HTTP endpoint. When an external system (GitHub, GitLab, DockerHub) sends a webhook to this endpoint, the controller triggers an immediate reconciliation of the associated Flux resources.

### Setting Up the Receiver Endpoint

**Step 1: Expose the Notification Controller:**

```bash
# Using a LoadBalancer service
kubectl -n flux-system apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: receiver
  namespace: flux-system
spec:
  type: LoadBalancer
  selector:
    app: notification-controller
  ports:
    - name: http
      port: 80
      targetPort: 9292
EOF

# Or using an Ingress
```

**Step 2: Create a webhook secret token:**

```bash
TOKEN=$(head -c 12 /dev/urandom | shasum | cut -d ' ' -f1)
kubectl -n flux-system create secret generic webhook-token \
  --from-literal=token=$TOKEN
echo "Token: $TOKEN"   # Save this, you'll need it when registering the webhook
```

**Step 3: Create a Receiver resource:**

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1
kind: Receiver
metadata:
  name: github-receiver
  namespace: flux-system
spec:
  type: github
  events:
    - "ping"
    - "push"
  secretRef:
    name: webhook-token
  resources:
    - apiVersion: source.toolkit.fluxcd.io/v1
      kind: GitRepository
      name: flux-system
    - apiVersion: kustomize.toolkit.fluxcd.io/v1
      kind: Kustomization
      name: apps
```

**Step 4: Get the webhook URL:**

```bash
kubectl -n flux-system get receiver github-receiver -o jsonpath='{.status.webhookPath}'
# Output: /hook/sha256sum...

# Full URL: https://<receiver-service-ip>/hook/sha256sum...
```

**Step 5: Register the webhook in GitHub:**

1. Navigate to your repository → Settings → Webhooks → Add webhook
2. Payload URL: `https://<your-receiver-endpoint>/hook/<hash>`
3. Content type: `application/json`
4. Secret: the token from Step 2
5. Events: Select "Just the push event" (or "Send me everything")

### Supported Webhook Types

Flux supports the following receiver types out of the box:

| Type | Source | Events |
|---|---|---|
| `github` | GitHub | push, ping |
| `gitlab` | GitLab | push, tag push, merge request |
| `bitbucket` | Bitbucket | push |
| `harbor` | Harbor | push (image) |
| `dockerhub` | Docker Hub | push (image) |
| `quay` | Quay.io | push (image) |
| `gcr` | Google Container Registry | push (image) |
| `nexus` | Nexus | push (image) |
| `generic` | Any HTTP client | custom |
| `generic-hmac` | Any HTTP client (HMAC-verified) | custom |

### Generic Receivers

For systems not in the supported list, use the `generic` or `generic-hmac` type:

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1
kind: Receiver
metadata:
  name: jenkins-receiver
  namespace: flux-system
spec:
  type: generic-hmac
  secretRef:
    name: webhook-token
  resources:
    - apiVersion: source.toolkit.fluxcd.io/v1
      kind: GitRepository
      name: flux-system
```

Your Jenkins pipeline then sends a POST to the receiver URL with the HMAC signature:

```bash
# In Jenkins pipeline
curl -X POST \
  -H "Content-Type: application/json" \
  -H "X-Signature: sha256=$(echo -n '{}' | openssl dgst -sha256 -hmac $TOKEN | awk '{print $2}')" \
  https://<receiver-url>/hook/<hash> \
  -d '{}'
```

---

## 9. Image Automation: Automated Container Updates

One of Flux's most powerful features is automated container image updates. When a new image is pushed to a container registry, Flux can automatically update the image tag in Git and trigger a deployment — closing the entire GitOps loop.

This requires two optional controllers: the **Image Reflector Controller** and the **Image Automation Controller**.

### Enable Image Automation Controllers

At bootstrap:

```bash
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  ...
```

Or apply the components manually:

```bash
flux install --components-extra=image-reflector-controller,image-automation-controller
```

### Step 1: Create an ImageRepository

An `ImageRepository` tells Flux which container registry to scan:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  image: ghcr.io/stefanprodan/podinfo
  interval: 1m          # scan the registry every minute
```

For private registries:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app
  interval: 1m
  secretRef:
    name: ecr-credentials    # Docker config secret with registry credentials
```

### Step 2: Create an ImagePolicy

An `ImagePolicy` defines which image tags to select based on a policy rule:

**Semver policy (most common for releases):**

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: podinfo
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: podinfo
  policy:
    semver:
      range: ">=5.0.0 <6.0.0"    # select latest in this semver range
```

**Alphabetical policy (for sortable timestamp-based tags):**

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: my-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: my-app
  filterTags:
    pattern: '^main-[a-f0-9]+-(?P<ts>[0-9]+)'
    extract: '$ts'
  policy:
    alphabetical:
      order: asc          # select the "latest" by alphabetical sort (ascending)
```

**Numerical policy:**

```yaml
spec:
  policy:
    numerical:
      order: asc
```

### Step 3: Annotate Manifests for Automated Updates

Add marker comments to your Kubernetes manifests to indicate which field Flux should update:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
spec:
  template:
    spec:
      containers:
        - name: podinfo
          image: ghcr.io/stefanprodan/podinfo:5.0.0  # {"$imagepolicy": "flux-system:podinfo"}
```

For Helm `values.yaml`:

```yaml
# values.yaml
image:
  repository: ghcr.io/stefanprodan/podinfo
  tag: "5.0.0"  # {"$imagepolicy": "flux-system:podinfo:tag"}
```

The `$imagepolicy` marker format is:
- `namespace:name` — updates the entire `image:tag` field
- `namespace:name:tag` — updates only the tag component
- `namespace:name:name` — updates only the repository component

### Step 4: Create an ImageUpdateAutomation

The `ImageUpdateAutomation` resource tells Flux to commit changes back to Git:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: |
        Automated image update

        Automation name: {{ .AutomationObject }}

        Files:
        {{ range $filename, $_ := .Updated.Files -}}
        - {{ $filename }}
        {{ end -}}

        Objects:
        {{ range $resource, $_ := .Updated.Objects -}}
        - {{ $resource.Kind }} {{ $resource.Name }}
        {{ end -}}

        Images:
        {{ range .Updated.Images -}}
        - {{.}}
        {{ end -}}
    push:
      branch: main      # push directly to main
  update:
    strategy: Setters   # use the marker-comment strategy
    path: ./apps        # only scan this directory for markers
```

### Push to a Separate Branch for Review

For production environments, you may want automated image updates to create a pull request rather than pushing directly to `main`:

```yaml
spec:
  git:
    push:
      branch: image-updates     # push to a branch instead of main
```

This allows a human to review and merge the PR before the update goes to production.

### Verify Image Automation Status

```bash
# Check the ImageRepository scanning status
flux get images repository podinfo

# Check the latest selected image
flux get images policy podinfo

# Check the automation status and last commit
flux get images update flux-system

# Watch all image resources
flux get images all
```

---

## 10. Sortable Image Tags for Automation

For image automation to work effectively, container image tags must be sortable in a meaningful way — so that Flux can determine which tag represents the "latest" version.

### Why Sortable Tags Matter

If you're using random tags (e.g., Git commit SHAs like `a1b2c3d4`) without additional structure, Flux cannot determine temporal order. You need to encode ordering information into the tag itself.

### Pattern 1: Timestamp-Based Tags (Recommended)

Encode a timestamp in the tag so that lexicographic ordering matches temporal ordering. The simplest format: `BRANCH-SHA-TIMESTAMP`.

**GitHub Actions CI pipeline:**

```yaml
name: Build and Push

on:
  push:
    branches: [main]

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set image metadata
        id: meta
        run: |
          echo "DATE=$(date -u +'%Y%m%d%H%M%S')" >> $GITHUB_OUTPUT
          echo "SHA=${GITHUB_SHA::8}" >> $GITHUB_OUTPUT

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:main-${{ steps.meta.outputs.SHA }}-${{ steps.meta.outputs.DATE }}
```

This produces tags like `main-a1b2c3d4-20240115143022`.

**Corresponding ImagePolicy:**

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: my-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: my-app
  filterTags:
    pattern: '^main-[a-f0-9]+-(?P<ts>[0-9]+)$'
    extract: '$ts'
  policy:
    alphabetical:
      order: asc     # the highest timestamp is the latest
```

### Pattern 2: Semver Tags (For Releases)

For release workflows, use proper semantic versioning (`v1.2.3`):

```bash
# In your release pipeline
docker tag my-app:latest my-app:v1.2.3
docker push my-app:v1.2.3
```

**Corresponding ImagePolicy:**

```yaml
spec:
  policy:
    semver:
      range: ">=1.0.0 <2.0.0"
```

### Pattern 3: Branch-Based Tags

Different branches (main, develop, feature) push to separate tag prefixes, allowing separate image policies per environment:

```bash
# main branch → production images
docker build -t my-app:main-${SHA}-${TIMESTAMP}

# develop branch → staging images
docker build -t my-app:develop-${SHA}-${TIMESTAMP}
```

**Staging policy (develop branch images):**

```yaml
spec:
  filterTags:
    pattern: '^develop-[a-f0-9]+-(?P<ts>[0-9]+)$'
    extract: '$ts'
  policy:
    alphabetical:
      order: asc
```

**Production policy (main branch images):**

```yaml
spec:
  filterTags:
    pattern: '^main-[a-f0-9]+-(?P<ts>[0-9]+)$'
    extract: '$ts'
  policy:
    alphabetical:
      order: asc
```

---

## 11. Multi-Tenancy and Access Control

In a shared Kubernetes cluster, multiple teams (tenants) need isolation from each other. Flux provides several mechanisms to enforce this.

### Multi-Tenancy Lockdown

Flux's multi-tenancy lockdown (`--default-service-account`) prevents tenant Kustomizations from using service accounts with elevated privileges.

Enable multi-tenancy lockdown at bootstrap:

```bash
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --default-service-account=kustomize-controller \
  ...
```

Or configure it via a `FluxInstance` (Flux Operator) or post-bootstrap patch.

### Namespace Isolation

The fundamental unit of isolation is the Kubernetes namespace. Each tenant gets their own namespace:

```bash
flux create tenant team1 \
  --with-namespace=team1 \
  --export > teams/team1/rbac.yaml
```

This generates:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team1
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kustomize-controller
  namespace: team1
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kustomize-controller
  namespace: team1
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kustomize-controller
  namespace: team1
subjects:
  - kind: ServiceAccount
    name: kustomize-controller
    namespace: team1
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kustomize-controller
```

### Configuring Tenant Kustomizations

The platform team creates the GitRepository and Kustomization for each tenant, scoped to the tenant's namespace:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: team1
  namespace: team1       # scoped to team1 namespace
spec:
  interval: 1m
  url: https://github.com/org/team1-apps
  ref:
    branch: main
  secretRef:
    name: team1-deploy-key
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team1-apps
  namespace: team1       # scoped to team1 namespace
spec:
  serviceAccountName: kustomize-controller   # uses the restricted team1 service account
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: team1
  path: ./deploy/production
  prune: true
  targetNamespace: team1
```

Because `serviceAccountName` is set to the restricted service account, team1's manifests can only affect resources in the `team1` namespace. They cannot create `ClusterRole`, `PersistentVolume`, or other cluster-scoped resources.

### Preventing Privilege Escalation

To prevent tenants from creating resources that would escalate their privileges (e.g., ClusterRoleBinding), configure Flux with multi-tenancy restrictions:

In the controller arguments (set via bootstrap patch):

```yaml
# flux-system/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - patch: |
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --default-service-account=kustomize-controller
    target:
      kind: Deployment
      name: kustomize-controller
```

### Cross-Namespace Source References

By default, sources (GitRepository, HelmRepository) must be in the same namespace as the consumer. Cross-namespace references require explicit opt-in:

```bash
# Allow cross-namespace source references (use carefully)
flux bootstrap github \
  --feature-gates=AllowCrossNamespaceRef=true \
  ...
```

For Helm:

```bash
flux bootstrap github \
  --feature-gates=HelmAllowCrossNamespaceRef=true \
  ...
```

---

## 12. Monitoring, Alerting and Observability

### Prometheus Metrics

All Flux controllers expose Prometheus metrics at `:8080/metrics`. The key metrics to monitor are:

| Metric | Description |
|---|---|
| `gotk_reconcile_duration_seconds` | Histogram of reconciliation duration |
| `gotk_reconcile_total` | Total reconciliations, labeled by name/namespace/kind/success |
| `gotk_suspend_status` | Whether a resource is suspended |
| `controller_runtime_reconcile_errors_total` | Total reconciliation errors |
| `workqueue_depth` | Current depth of the reconciliation work queue |

### Setting Up Flux Monitoring with Prometheus and Grafana

Deploy the kube-prometheus-stack, then apply the Flux dashboards and ServiceMonitors:

```bash
# Install kube-prometheus-stack
flux create helmrelease kube-prometheus-stack \
  --namespace monitoring \
  --source HelmRepository/prometheus-community \
  --chart kube-prometheus-stack \
  --chart-version ">=45.0.0" \
  --values prometheus-values.yaml

# Apply Flux monitoring resources from the official repository
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flux2-monitoring-example/main/monitoring/
```

The official Flux monitoring repository includes:
- Grafana dashboards for each controller
- PrometheusRules for alerting
- ServiceMonitors to scrape metrics

### Configuring Alerts

Flux's Notification Controller can send alerts to many platforms when reconciliation events occur.

**Step 1: Create a Provider:**

```yaml
# Slack provider
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: "#flux-alerts"
  secretRef:
    name: slack-webhook-url
---
apiVersion: v1
kind: Secret
metadata:
  name: slack-webhook-url
  namespace: flux-system
stringData:
  address: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

Other supported providers include: `msteams`, `pagerduty`, `opsgenie`, `datadog`, `discord`, `github`, `gitlab`, `bitbucket`, `googlechat`, `telegram`, `lark`, `matrix`, `azureeventhub`, `sentry`, `alertmanager`, and `generic`.

**Step 2: Create an Alert:**

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-call-webapp
  namespace: flux-system
spec:
  summary: "Flux reconciliation alert"
  providerRef:
    name: slack
  eventSeverity: error        # only alert on errors (not info)
  eventSources:
    - kind: GitRepository
      name: "*"               # wildcard — all GitRepositories
    - kind: Kustomization
      name: "*"
    - kind: HelmRelease
      name: "*"
  exclusionList:
    - ".*health check failed after 1 attempt.*"    # suppress transient noise
```

**Step 3: Create an Info-Level Alert (for deployments):**

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: deployment-audit
  namespace: flux-system
spec:
  summary: "Deployment notification"
  providerRef:
    name: slack
  eventSeverity: info
  eventSources:
    - kind: Kustomization
      name: apps             # only alert on the apps Kustomization
    - kind: HelmRelease
      name: "*"
      namespace: default
```

### GitHub Commit Status Updates

Flux can post commit status checks back to GitHub, providing visibility directly in PRs:

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: github-status
  namespace: flux-system
spec:
  type: github
  address: https://github.com/org/fleet-infra
  secretRef:
    name: github-token
---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: github-commit-status
  namespace: flux-system
spec:
  providerRef:
    name: github-status
  eventSeverity: info
  eventSources:
    - kind: Kustomization
      name: apps
```

### Flux Events

Flux emits Kubernetes events for all reconciliation activity. These are visible with:

```bash
# Watch events from Flux controllers
flux events

# Watch events for a specific resource
flux events --for Kustomization/apps

# Show only error events
flux events --for HelmRelease/podinfo

# Stream events live
kubectl get events -n flux-system --watch
```

### Structured Logging

Flux controllers output structured JSON logs. Query them with:

```bash
# Tail logs from all Flux controllers
flux logs --all-namespaces

# Tail logs from a specific controller
flux logs --level=error --source GitRepository/flux-system

# Follow kustomize controller logs
kubectl -n flux-system logs -f deploy/kustomize-controller
```

---

## 13. OCI Artifacts and the Source Controller

Flux supports OCI (Open Container Initiative) as a source for both Helm charts and raw Kubernetes manifests. This enables a fully OCI-native GitOps workflow where artifacts are stored in a container registry rather than (or in addition to) Git.

### Pushing Manifests as OCI Artifacts

Use the Flux CLI to package and push Kubernetes manifests to an OCI registry:

```bash
# Build and push a Kustomize directory as OCI artifact
flux push artifact oci://ghcr.io/org/manifests/app:$(git rev-parse --short HEAD) \
  --path="./apps/production" \
  --source="$(git config --get remote.origin.url)" \
  --revision="$(git branch --show-current)@sha1:$(git rev-parse HEAD)"

# Tag as latest
flux tag artifact oci://ghcr.io/org/manifests/app:$(git rev-parse --short HEAD) \
  --tag latest
```

In a GitHub Actions pipeline:

```yaml
- name: Push OCI artifact
  run: |
    flux push artifact oci://ghcr.io/${{ github.repository_owner }}/manifests/app:${{ github.sha }} \
      --path="./apps/production" \
      --source="${{ github.repositoryUrl }}" \
      --revision="${{ github.ref_name }}@sha1:${{ github.sha }}"
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Creating an OCIRepository Source

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: app-manifests
  namespace: flux-system
spec:
  interval: 1m
  url: oci://ghcr.io/org/manifests/app
  ref:
    tag: latest           # or use digest: sha256:abc123 for immutability
  verify:
    provider: cosign      # verify the OCI artifact signature with Cosign
    secretRef:
      name: cosign-pub-key
```

### Artifact Verification with Cosign

Cosign allows you to cryptographically sign and verify OCI artifacts, preventing supply chain attacks:

**Sign the artifact (in CI):**

```bash
# Install cosign
cosign sign --key cosign.key oci://ghcr.io/org/manifests/app:$(git rev-parse --short HEAD)
```

**Verify in Flux:**

```yaml
spec:
  verify:
    provider: cosign
    secretRef:
      name: cosign-pub-key
---
apiVersion: v1
kind: Secret
metadata:
  name: cosign-pub-key
  namespace: flux-system
stringData:
  cosign.pub: |
    -----BEGIN PUBLIC KEY-----
    ...
    -----END PUBLIC KEY-----
```

### Using OCIRepository with Kustomize

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: app-from-oci
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: OCIRepository
    name: app-manifests
  path: ./              # root of the OCI artifact
  prune: true
```

### OCI Helm Charts

Helm v3.8+ supports OCI registries natively. Flux can reference OCI Helm charts directly:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: podinfo-oci
  namespace: flux-system
spec:
  interval: 5m
  type: oci
  url: oci://ghcr.io/stefanprodan/charts
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: default
spec:
  interval: 5m
  chart:
    spec:
      chart: podinfo
      version: ">=6.0.0"
      sourceRef:
        kind: HelmRepository
        name: podinfo-oci
        namespace: flux-system
```

---

## 14. Advanced Kustomization Patterns

### Component-Based Structure

Kustomize components allow you to create reusable, opt-in configuration fragments:

```
infrastructure/
└── base/
    └── ingress-nginx/
        ├── kustomization.yaml       # base installation
        └── components/
            ├── monitoring/          # optional: add Prometheus annotations
            │   ├── kustomization.yaml
            │   └── service-monitor.yaml
            └── high-availability/  # optional: increase replicas, add PodDisruptionBudget
                ├── kustomization.yaml
                └── ha-patch.yaml
```

**Production kustomization.yaml using components:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/ingress-nginx
components:
  - ../../base/ingress-nginx/components/monitoring
  - ../../base/ingress-nginx/components/high-availability
```

### Flux Kustomization with Remote Base

Reference a base from a remote Git repository:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - https://github.com/fluxcd/flux2-kustomize-helm-example//apps/base/podinfo?ref=v1.0.0
patches:
  - patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: podinfo
      spec:
        replicas: 3
    target:
      kind: Deployment
      name: podinfo
```

### Advanced Health Checks with CEL

Flux v2.3+ supports CEL (Common Expression Language) for custom health check expressions:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: podinfo
      namespace: default
  healthCheckExprs:
    - apiVersion: apps/v1
      kind: Deployment
      # Custom expression: healthy only when all replicas are ready and not just "available"
      current: "status.readyReplicas == status.replicas && status.replicas > 0"
      inProgress: "status.readyReplicas < status.replicas"
      failed: "status.replicas == 0"
```

### Dependency Chaining Across Clusters

In a multi-cluster setup, you can chain Kustomizations across clusters using Remote Kustomizations:

```yaml
# On the management cluster — points to a workload cluster's Kustomization
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: workload-cluster-apps
  namespace: fleet-system
spec:
  kubeConfig:
    secretRef:
      name: workload-cluster-kubeconfig    # kubeconfig for the target cluster
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./clusters/workload/apps
  prune: true
```

### Impersonation

To further restrict what a Kustomization can do, use `serviceAccountName` to impersonate a scoped service account:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: dev-team-apps
  namespace: dev-team
spec:
  serviceAccountName: dev-team-reconciler   # this SA has limited RBAC
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: dev-team-apps
  path: ./deploy
  prune: true
  targetNamespace: dev-team
```

---

## 15. Progressive Delivery with Flagger

While Flux handles GitOps-based deployment, Flagger (a companion project) adds progressive delivery capabilities: canary releases, A/B testing, and blue/green deployments.

### How Flagger and Flux Work Together

Flux deploys the `Canary` CRD (Flagger's resource). Flagger watches for changes to the underlying Deployment and automatically manages the traffic shifting:

```
Flux reconciles   →   Canary resource updated   →   Flagger takes over
Deployment image         (new image detected)           (shifts traffic
updated in Git                                           gradually)
```

### Installing Flagger with Flux

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: flagger
  namespace: flux-system
spec:
  interval: 1h
  url: https://flagger.app
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: flagger
  namespace: flux-system
spec:
  interval: 1h
  chart:
    spec:
      chart: flagger
      version: ">=1.30.0"
      sourceRef:
        kind: HelmRepository
        name: flagger
        namespace: flux-system
  values:
    meshProvider: nginx          # or: istio, linkerd, appmesh
    metricsServer: http://prometheus:9090
```

### Defining a Canary Release

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    port: 9898
    targetPort: 9898
  analysis:
    interval: 30s           # analyze every 30 seconds
    threshold: 5            # max 5 failed analyses before rollback
    maxWeight: 50           # maximum traffic weight for canary (50%)
    stepWeight: 10          # increment by 10% per step
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99           # must have ≥99% success rate
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500          # must be ≤500ms p99 latency
        interval: 30s
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 30s
        metadata:
          type: bash
          cmd: "curl -sd 'anon' http://podinfo-canary:9898/token | grep token"
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://podinfo-canary:9898/"
```

When Flux updates the Deployment image tag, Flagger detects the change and begins a canary analysis, gradually shifting traffic from the stable version to the canary. If metrics thresholds are breached, Flagger automatically rolls back.

---

## 16. Troubleshooting and Debugging

### Quick Diagnosis Commands

```bash
# Check overall Flux health
flux check

# Get status of all Flux resources
flux get all --all-namespaces

# Get all failed resources
flux get all --status-selector ready=false --all-namespaces

# Describe a specific resource
flux describe kustomization apps
flux describe helmrelease podinfo

# Check events for a resource
flux events --for Kustomization/apps
flux events --for HelmRelease/podinfo --watch
```

### Forcing Reconciliation

When you want Flux to reconcile immediately without waiting for the next interval:

```bash
# Force reconciliation of a GitRepository
flux reconcile source git flux-system

# Force reconciliation of a Kustomization
flux reconcile kustomization apps

# Force reconciliation of a HelmRelease
flux reconcile helmrelease podinfo

# Force reconciliation with a dependency chain
flux reconcile kustomization apps --with-source
```

### Diffing Against the Cluster

See what changes Flux would apply without actually applying them:

```bash
# Diff a Kustomization against the current cluster state
flux diff kustomization apps

# Diff with a local path
flux diff kustomization apps --path ./apps/production
```

### Debugging HelmRelease Failures

```bash
# Describe the HelmRelease to see error conditions
flux describe helmrelease podinfo

# Debug using the flux debug command (shows computed values)
flux debug helmrelease podinfo

# Check Helm controller logs for detail
kubectl -n flux-system logs -f deploy/helm-controller | grep podinfo

# Check the actual Helm release state
helm -n default history podinfo
helm -n default get values podinfo
```

### Common Error Patterns and Resolutions

**Error: `context deadline exceeded`**

The reconciliation timed out. Increase the `timeout` field or investigate why resources are taking so long to become ready:

```yaml
spec:
  timeout: 10m    # increase from default 5m
```

**Error: `unable to replace resource`**

An immutable field was changed. Use `force: true` to allow recreation:

```yaml
spec:
  force: true
```

**Error: `Health check failed after X attempts`**

A deployed resource is not reaching a ready state. Check the resource directly:

```bash
kubectl describe pod -n default -l app=podinfo
kubectl logs -n default -l app=podinfo --tail=50
```

**Error: `Secret not found` (SOPS)**

The SOPS private key secret is missing or misconfigured:

```bash
kubectl -n flux-system get secret sops-age
kubectl -n flux-system describe kustomization apps | grep decryption
```

**Error: `chart not found` or `version not found`**

The HelmRepository hasn't been indexed or the chart version doesn't exist:

```bash
flux reconcile source helm bitnami
flux get sources helm bitnami
```

### Suspending and Resuming

When performing manual maintenance or investigating an issue, you can pause reconciliation:

```bash
# Suspend a Kustomization (stops all reconciliation)
flux suspend kustomization apps

# Make manual changes...

# Resume reconciliation
flux resume kustomization apps

# Suspend everything (useful for cluster maintenance)
flux suspend kustomization --all
flux suspend helmrelease --all

# Resume everything
flux resume kustomization --all
flux resume helmrelease --all
```

### Viewing the Dependency Tree

```bash
# Show the full tree of resources managed by a Kustomization
flux tree kustomization apps

# Show OCI artifact trees
flux tree artifact oci://ghcr.io/org/manifests/app:latest
```

---

## 17. Security Best Practices

### Principle of Least Privilege for Flux Controllers

By default, Flux's kustomize-controller runs with cluster-admin equivalent permissions (it needs to apply any resource type). For multi-tenant environments, restrict this:

```yaml
# Create a limited ClusterRole for Flux
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: flux-limited
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["services", "configmaps", "serviceaccounts"]
    verbs: ["*"]
  # Add other resource types your fleet needs
```

### Network Policy for Flux Controllers

Restrict egress from Flux controllers to only required destinations:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: source-controller
  namespace: flux-system
spec:
  podSelector:
    matchLabels:
      app: source-controller
  policyTypes:
    - Egress
  egress:
    - ports:
        - port: 443   # HTTPS to Git/Helm registries
        - port: 22    # SSH to Git repositories
      to:
        - ipBlock:
            cidr: 0.0.0.0/0
```

### Workload Identity for Cloud KMS

Avoid storing cloud credentials in Kubernetes Secrets. Instead, use workload identity:

**AWS (IRSA):**

```bash
# Annotate the Flux service account with the IAM role ARN
kubectl annotate serviceaccount kustomize-controller \
  -n flux-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/flux-kustomize-controller
```

**GCP (Workload Identity):**

```bash
kubectl annotate serviceaccount kustomize-controller \
  -n flux-system \
  iam.gke.io/gcp-service-account=flux@my-project.iam.gserviceaccount.com
```

**Azure (Managed Identity):**

```bash
kubectl annotate serviceaccount kustomize-controller \
  -n flux-system \
  azure.workload.identity/client-id=<MANAGED_IDENTITY_CLIENT_ID>
```

### Artifact Verification

Always verify the integrity of artifacts in production. For Git sources:

```yaml
spec:
  verification:
    mode: head                # verify the HEAD commit
    secretRef:
      name: git-signing-key  # PGP public key or SSH signing key
```

For OCI sources (with Cosign):

```yaml
spec:
  verify:
    provider: cosign
    secretRef:
      name: cosign-pub-key
```

### Audit Logging

Enable Kubernetes audit logging to capture all API server interactions made by Flux controllers. Flux's service accounts are identifiable by name (`kustomize-controller`, `helm-controller`, etc.), making it easy to filter Flux-specific operations in audit logs.

### GitOps Security Checklist

| Item | Description |
|---|---|
| Encrypted secrets | All secrets in Git are encrypted with SOPS or sealed |
| RBAC per tenant | Each tenant has a scoped service account |
| Network policies | Flux controllers have egress restrictions |
| Artifact verification | Git commits and OCI images are signed and verified |
| Branch protection | `main`/`production` branches require PR reviews |
| Audit trail | All Flux commits are attributed (bot user, signed) |
| Key rotation | SOPS/Sealed Secrets keys are rotated regularly |
| Workload identity | Cloud KMS accessed via pod identity, not static keys |
| Suspend capability | Operators know how to pause Flux in an emergency |

---

## 18. Reference: Essential CLI Commands

### Installation and Bootstrap

```bash
# Check prerequisites
flux check --pre

# Bootstrap with GitHub
flux bootstrap github \
  --owner=<user> \
  --repository=<repo> \
  --branch=main \
  --path=clusters/my-cluster \
  --personal

# Bootstrap with extra components
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=<user> --repository=<repo> --branch=main --path=clusters/my-cluster --personal

# Upgrade Flux controllers to latest
flux install --version=latest

# Uninstall Flux (keeps CRDs and custom resources)
flux uninstall --keep-namespace
```

### Source Management

```bash
# Create a GitRepository source
flux create source git my-app \
  --url=https://github.com/org/my-app \
  --branch=main \
  --interval=1m

# Create a HelmRepository source
flux create source helm bitnami \
  --url=https://charts.bitnami.com/bitnami \
  --interval=12h

# Create an OCI repository source
flux create source oci my-app \
  --url=oci://ghcr.io/org/manifests/my-app \
  --tag=latest \
  --interval=1m

# List all sources
flux get sources all

# Force sync a source
flux reconcile source git flux-system
```

### Kustomization Management

```bash
# Create a Kustomization
flux create kustomization my-app \
  --source=GitRepository/flux-system \
  --path=./apps/production \
  --prune=true \
  --interval=5m

# Get Kustomization status
flux get kustomizations

# Force reconcile
flux reconcile kustomization apps --with-source

# Diff
flux diff kustomization apps

# Suspend / Resume
flux suspend kustomization apps
flux resume kustomization apps

# Delete (removes managed resources if prune=true)
flux delete kustomization my-app
```

### HelmRelease Management

```bash
# Create a HelmRelease
flux create helmrelease podinfo \
  --source=HelmRepository/podinfo \
  --chart=podinfo \
  --chart-version="6.x" \
  --namespace=default \
  --interval=5m

# Get HelmRelease status
flux get helmreleases --all-namespaces

# Force reconcile
flux reconcile helmrelease podinfo

# Debug values and chart metadata
flux debug helmrelease podinfo

# Suspend/Resume
flux suspend helmrelease podinfo
flux resume helmrelease podinfo
```

### Image Automation

```bash
# Create an ImageRepository
flux create image repository my-app \
  --image=ghcr.io/org/my-app \
  --interval=1m

# Create an ImagePolicy (semver)
flux create image policy my-app \
  --image-ref=my-app \
  --select-semver=">=1.0.0 <2.0.0"

# Create an ImagePolicy (latest tag by timestamp)
flux create image policy my-app \
  --image-ref=my-app \
  --filter-regex='^main-[a-f0-9]+-(?P<ts>[0-9]+)$' \
  --filter-extract='$ts' \
  --select-alphanum-asc

# Create an ImageUpdateAutomation
flux create image update flux-system \
  --git-repo-ref=flux-system \
  --git-repo-path=./apps \
  --checkout-branch=main \
  --push-branch=main \
  --author-name=fluxcdbot \
  --author-email=fluxcdbot@users.noreply.github.com \
  --commit-template='Automated image update to {{range .Updated.Images}}{{println .}}{{end}}' \
  --interval=1m

# Check all image automation resources
flux get images all
```

### Notification and Alerting

```bash
# Create a Slack provider
flux create alert-provider slack \
  --type=slack \
  --channel='#flux-alerts' \
  --secret-ref=slack-webhook-url

# Create an alert
flux create alert on-call-webapp \
  --event-severity=error \
  --event-source=GitRepository/* \
  --event-source=Kustomization/* \
  --provider-ref=slack

# Create a GitHub receiver
flux create receiver github-receiver \
  --type=github \
  --event=push \
  --secret-ref=webhook-token \
  --resource=GitRepository/flux-system

# Get receiver webhook URL
kubectl -n flux-system get receiver github-receiver \
  -o jsonpath='{.status.webhookPath}'
```

### OCI Artifact Operations

```bash
# Push manifests as OCI artifact
flux push artifact oci://ghcr.io/org/manifests:$(git rev-parse --short HEAD) \
  --path="./apps/production" \
  --source="$(git config --get remote.origin.url)" \
  --revision="main@sha1:$(git rev-parse HEAD)"

# Tag an artifact
flux tag artifact oci://ghcr.io/org/manifests:abc1234 --tag latest

# List artifacts
flux list artifacts oci://ghcr.io/org/manifests

# Diff artifact against cluster
flux diff artifact oci://ghcr.io/org/manifests:latest
```

### General Operations

```bash
# Export all Flux resources to YAML
flux export source git --all
flux export kustomization --all
flux export helmrelease --all

# View all Flux events (live stream)
flux events --watch

# View logs from all controllers
flux logs --all-namespaces --level=error

# Check Flux version
flux version

# Get resource tree
flux tree kustomization apps

# Trace a resource (find which Kustomization manages it)
flux trace deployment podinfo --namespace default

# Run pre-upgrade checks
flux check
```

---

## Appendix A: API Version Reference

| Resource | API Version | Controller |
|---|---|---|
| `GitRepository` | `source.toolkit.fluxcd.io/v1` | Source |
| `HelmRepository` | `source.toolkit.fluxcd.io/v1` | Source |
| `HelmChart` | `source.toolkit.fluxcd.io/v1` | Source |
| `OCIRepository` | `source.toolkit.fluxcd.io/v1beta2` | Source |
| `Bucket` | `source.toolkit.fluxcd.io/v1` | Source |
| `Kustomization` | `kustomize.toolkit.fluxcd.io/v1` | Kustomize |
| `HelmRelease` | `helm.toolkit.fluxcd.io/v2` | Helm |
| `Alert` | `notification.toolkit.fluxcd.io/v1beta3` | Notification |
| `Provider` | `notification.toolkit.fluxcd.io/v1beta3` | Notification |
| `Receiver` | `notification.toolkit.fluxcd.io/v1` | Notification |
| `ImageRepository` | `image.toolkit.fluxcd.io/v1beta2` | Image Reflector |
| `ImagePolicy` | `image.toolkit.fluxcd.io/v1beta2` | Image Reflector |
| `ImageUpdateAutomation` | `image.toolkit.fluxcd.io/v1beta2` | Image Automation |

---

## Appendix B: Environment Variable and Label Reference

### Standard Labels Applied by Flux

| Label | Value | Description |
|---|---|---|
| `kustomize.toolkit.fluxcd.io/name` | `<kustomization-name>` | Applied to all resources managed by a Kustomization |
| `kustomize.toolkit.fluxcd.io/namespace` | `<namespace>` | Applied to all resources managed by a Kustomization |
| `helm.toolkit.fluxcd.io/name` | `<helmrelease-name>` | Applied to all resources managed by a HelmRelease |
| `helm.toolkit.fluxcd.io/namespace` | `<namespace>` | Applied to all resources managed by a HelmRelease |

### Common Status Conditions

| Condition | Reason | Description |
|---|---|---|
| `Ready=True` | `ReconciliationSucceeded` | Last reconciliation completed successfully |
| `Ready=False` | `BuildFailed` | Kustomize build failed |
| `Ready=False` | `HealthCheckFailed` | Resource health checks failed |
| `Ready=False` | `ArtifactFailed` | Could not retrieve source artifact |
| `Ready=False` | `UpgradeFailed` | Helm upgrade failed |
| `Stalled=True` | `ReconciliationFailed` | Too many retries, halted until source changes |
| `Suspended=True` | `Suspended` | Resource is manually suspended |

---

## Appendix C: Quick Reference — Reconciliation Interval Recommendations

| Source Type | Recommended Interval | Notes |
|---|---|---|
| GitRepository (active dev) | `1m` | Fast feedback; use webhooks to avoid polling overhead |
| GitRepository (stable) | `5m–10m` | Lower load on Git server |
| HelmRepository | `10m–12h` | Depends on how often charts update |
| OCIRepository | `1m` | Use digest pinning for stability |
| Bucket | `5m` | Higher interval for cost-sensitive S3 usage |
| ImageRepository | `1m–5m` | Balance freshness vs. registry rate limits |
| Kustomization | `5m–10m` | Source polling drives actual refresh |
| HelmRelease | `5m–30m` | Mostly driven by source changes |

---

*Flux CD Comprehensive Guide — Based on Flux v2.8 official documentation*
*Reference: https://fluxcd.io/flux/guides/ | CNCF Graduated Project*
