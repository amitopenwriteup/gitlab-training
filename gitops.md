
## Day 1: Introduction to GitOps and FluxCD

### 1.1 Understanding GitOps Principles
- The Four Core GitOps Principles (Declarative, Versioned, Automated, Continuously Reconciled)
- GitOps vs Traditional CI/CD: A paradigm shift
- Git as the single source of truth
- Pull-based vs Push-based deployment models
- Reconciliation loops and drift detection

### 1.2 Why GitOps with FluxCD?
- FluxCD v2 vs Argo CD: Feature comparison and trade-offs
- Flux's CNCF graduation and ecosystem maturity
- Operator model and Kubernetes-native design
- Built-in multi-tenancy support
- Community, extensions, and long-term viability

### 1.3 Kubernetes Cluster Setup for GitOps
- Cluster prerequisites and RBAC considerations
- Namespace strategy for multi-team GitOps workflows
- Setting up local clusters with Kind or k3s for development
- Production-readiness checklist (OIDC, audit logging, network policies)
- Resource quotas and LimitRanges for GitOps-managed workloads

### 1.4 Installing FluxCD on Your Cluster
- Flux CLI installation and version pinning
- Bootstrapping Flux with `flux bootstrap github`
- Understanding what bootstrap deploys (CRDs, controllers, RBAC)
- Verifying Flux system health with `flux check`
- Flux upgrade strategies and maintenance windows

### 1.5 Git Repository Structure for GitOps
- Monorepo vs Polyrepo: Patterns and trade-offs
- Recommended directory layout: `clusters/`, `infrastructure/`, `apps/`
- Environment promotion strategy: dev → staging → production
- Branch strategy: long-lived environment branches vs trunk-based
- Repository access controls and protected branches

### 1.6 Manifests, Helm Charts, and Kustomize
- Raw Kubernetes manifests in GitOps workflows
- When to use Helm vs Kustomize vs raw YAML
- Helm chart versioning and release strategies
- Kustomize overlays for environment-specific configuration
- Combining Helm and Kustomize with `HelmRelease` + post-render patches

### 1.7 GitOps Best Practices
- Keeping secrets out of Git (external secrets vs sealed secrets)
- Immutable tags and avoiding `latest` in production
- Drift detection and automated reconciliation alerts
- Policy enforcement with OPA/Kyverno before merge
- Audit trails: who changed what, when, and why

---

## Day 2: Working with FluxCD

### 2.1 FluxCD Architecture and Core Components
- Source Controller: `GitRepository`, `HelmRepository`, `OCIRepository`, `Bucket`
- Kustomize Controller: `Kustomization` CRD deep-dive
- Helm Controller: `HelmRelease` lifecycle and reconciliation
- Notification Controller: Alerts, Providers, and Receivers
- Image Reflector & Automation Controllers
- How controllers communicate via the Flux object graph

### 2.2 Deploying a Sample Application Using FluxCD
- Creating a `GitRepository` source pointing to your app repo
- Defining a `Kustomization` object with health checks
- Setting `interval`, `timeout`, and `retryInterval` tuning
- Observing reconciliation with `flux get all` and `flux events`
- Forcing manual reconciliation with `flux reconcile`

### 2.3 Deploying Helm Charts with FluxCD
- Creating `HelmRepository` and `HelmRelease` resources
- Value overrides: inline values vs `valuesFrom` ConfigMaps/Secrets
- Handling Helm hooks in a GitOps context
- Chart dependency management and ordering
- Rollback behavior on failed Helm upgrades (`remediation` spec)

### 2.4 Using Kustomize for Application Deployment
- Kustomize `bases`, `overlays`, and `patches` in depth
- `commonLabels`, `namePrefix`, and `nameSuffix` at scale
- Strategic merge patches vs JSON patches
- Generating ConfigMaps and Secrets with Kustomize generators
- `dependsOn` for ordered multi-manifest deployments

### 2.5 Managing Secrets in GitOps
- Why plaintext secrets in Git are never acceptable
- **Sealed Secrets**: `kubeseal` workflow and controller setup
- **SOPS + Age/GPG**: Encrypting secrets at rest in Git
- **External Secrets Operator**: Syncing from Vault, AWS SSM, GCP Secret Manager
- Flux native support for SOPS decryption via `spec.decryption`
- Secret rotation strategies and zero-downtime updates

### 2.6 Synchronization and Rollbacks
- Understanding Flux reconciliation intervals and jitter
- Suspended resources and manual overrides (`flux suspend`)
- Automated rollback on health check failures
- Manual rollback with `flux install --export` + `kubectl apply`
- Rollback vs Roll-forward: GitOps philosophy
- Using `git revert` for auditable rollbacks

---

## Day 3: Advanced GitOps with FluxCD


### 3.2 Introduction to FluxCD Image Automation
- Image Reflector Controller: `ImageRepository` and `ImagePolicy` CRDs
- Semver policies: `^1.0`, `~2.3`, range constraints
- `ImageUpdateAutomation`: auto-committing image tag updates to Git
- Filtering images by regex and alphabetical ordering
- Restricting automation to non-production environments
- Notifications for automated image commits

### 3.3 Implementing Blue-Green Deployments
- Blue-green concepts in a Kubernetes-native context
- Using `Service` selectors to switch traffic between deployments
- Orchestrating blue-green with Kustomize overlays and Flux `Kustomization`
- Health check gates before traffic switch
- Automated rollback on failed health probes
- Integration with external load balancers and ingress controllers

### 3.4 Implementing Canary Deployments
- Canary release theory: traffic splitting and metrics-based promotion
- Manual canary with weighted `HTTPRoute` or Nginx ingress annotations
- Automated canary with **Flagger** (see Section 3.7 for deep-dive)
- Defining success criteria: latency, error rate, custom metrics
- Pausing and resuming canary rollouts
- Wiring canary status back to Flux health checks

### 3.5 Preparing a Sample Application for GitOps
- Containerising the app with reproducible Docker builds
- Tagging strategy: Git SHA + semantic version for traceability
- Writing production-ready Kubernetes manifests (probes, limits, PDBs)
- Structuring Helm values for environment promotion
- Pre-flight validation with `kubeval`, `kubeconform`, and `helm lint`
- Adding Flux `HelmRelease` health checks for automated gates

### 3.6 Connecting FluxCD to GitHub
- GitHub PAT vs GitHub App authentication (recommended for production)
- Creating a GitHub App and configuring Flux bootstrap
- `Receiver` and webhook-driven reconciliation for instant sync
- HMAC secret validation for webhook security
- Org-level repo access and fine-grained permissions
- Handling rate limits and GitHub API quotas

### 3.7 Setting Up the CI Pipeline Using GitHub Actions
- Separation of concerns: CI builds artifacts, CD (Flux) deploys them
- CI workflow: lint → test → build → push image → update Git tag
- Using `peter-evans/create-pull-request` to open promotion PRs
- Policy gates: require CI green before Flux can reconcile
- Caching layers and build matrix for faster pipelines
- Signing container images with `cosign` for supply-chain security

### 3.8 Using FluxCD for Continuous Deployment (CD)
- End-to-end GitOps CD pipeline walkthrough
- Promotion workflow: dev → staging → production via Git PRs
- Automated staging promotion on CI success
- Manual approval gates for production (GitHub Environments + required reviewers)
- Observability: correlating Git commits to running workloads
- SLO monitoring and alerting for deployed workloads

---

## Module 4: Progressive Delivery with Flagger

### 4.1 What is Flagger and Why It Matters
- Flagger as a Kubernetes operator for progressive delivery
- Relationship between Flagger and Flux: complementary, not competing
- Supported deployment strategies: Canary, Blue/Green, A/B Testing, Mirror
- Supported service meshes and ingress controllers:
  - Istio, Linkerd, AWS App Mesh, Open Service Mesh
  - Nginx, Traefik, Gloo, Contour, Gateway API
- Flagger's analysis loop: metric scraping → decision → promote/rollback

### 4.2 Installing Flagger with FluxCD
- Adding Flagger's `HelmRepository` source in Flux
- Deploying Flagger via `HelmRelease` with Prometheus integration
- Installing the Flagger load testing addon (`flagger-loadtester`)
- Configuring Flagger to watch specific namespaces
- Verifying Flagger readiness: CRDs, webhook, and RBAC

### 4.3 Core Flagger Concepts
- The `Canary` CRD: anatomy and key fields
- `spec.targetRef`: pointing Flagger at a `Deployment` or `DaemonSet`
- `spec.service`: virtual service, port, and traffic policy configuration
- `spec.analysis`: interval, threshold, iterations, and max weight
- `spec.metrics`: built-in vs custom metric templates
- `spec.webhooks`: load testing, acceptance tests, manual approval gates

### 4.4 Canary Releases with Flagger
- Creating your first `Canary` resource end-to-end
- How Flagger automatically creates primary and canary deployments
- Traffic shifting: from 0% → 10% → 50% → 100% with configurable steps
- Real-time status with `kubectl describe canary` and Flagger events
- Metric analysis during promotion: request success rate and latency P99
- Triggering a canary by updating the deployment image tag

### 4.6 Blue-Green Deployments with Flagger
- Configuring `spec.analysis.iterations` for blue-green (vs canary step weight)
- Flagger-managed blue-green: no manual `Service` selector switching
- Running acceptance tests via webhooks before cutover
- Zero-downtime traffic switch using Kubernetes `Service` updates
- Automated rollback on webhook failure or metric breach
- Differences from manual blue-green in Section 3.3

### 4.10 Flagger in a Full GitOps Pipeline
- End-to-end flow: GitHub PR → CI build → image push → Flux detects tag → Flagger promotes
- Flux `ImageUpdateAutomation` triggering Flagger canary automatically
- Using Flux `Alert` to notify teams when Flagger promotes or rolls back
- Gating production promotion: Flagger webhook calling an approval service
- Observability stack: Prometheus + Grafana + Flagger dashboard
- Full reference architecture diagram and component interaction map

---
