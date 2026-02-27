Here is your content converted into a clean **Markdown (.md) format** ready to save as:

`canary-deployments-with-flagger.md`

---

# Canary Deployments with Flagger

Implement automated canary releases and blue-green deployments using **Flagger** and **FluxCD**.

---

## Prerequisites

* Kubernetes cluster with FluxCD bootstrapped (`kind + flux bootstrap gitlab`)
* NGINX Ingress Controller installed via Flux (see Exercise 0)
* `fleet-infra` repository cloned locally
* `GITLAB_TOKEN` and `GITLAB_USER` exported in your shell

---

# Exercise 0: Install NGINX Ingress Controller via FluxCD

Before deploying Flagger, you must have the NGINX Ingress Controller running in your cluster.

We install it the GitOps way — using a HelmRelease managed by Flux committed to your `fleet-infra` repository.

---

## 0.1 Add NGINX Ingress HelmRepository Source

Create the HelmRepository pointing to the ingress-nginx Helm chart registry.

```yaml
# clusters/my-cluster/infrastructure/sources/nginx-ingress.yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 1h
  url: https://kubernetes.github.io/ingress-nginx
```

### Git Commit & Flux Reconcile

```bash
cd fleet-infra
mkdir -p clusters/my-cluster/infrastructure/sources
git add clusters/my-cluster/infrastructure/sources/nginx-ingress.yaml
git commit -m "feat: add ingress-nginx HelmRepository source"
git push

flux reconcile source git flux-system
flux get sources helm
```

Expected:

```
ingress-nginx   https://...   False   True   stored artifact
```

---

## 0.2 Create Namespace + Install NGINX via HelmRelease

```yaml
# clusters/my-cluster/infrastructure/controllers/nginx-ingress-namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
```

```yaml
# clusters/my-cluster/infrastructure/controllers/nginx-ingress.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  releaseName: ingress-nginx
  interval: 12h
  install:
    createNamespace: true
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  chart:
    spec:
      chart: ingress-nginx
      version: "4.x"
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
  values:
    controller:
      replicaCount: 1
      service:
        type: LoadBalancer
      metrics:
        enabled: true
        serviceMonitor:
          enabled: false
      config:
        enable-modsecurity: "false"
        use-forwarded-headers: "true"
```

### Git Commit & Verify

```bash
mkdir -p clusters/my-cluster/infrastructure/controllers
git add clusters/my-cluster/infrastructure/controllers/nginx-ingress-namespace.yaml
git add clusters/my-cluster/infrastructure/controllers/nginx-ingress.yaml
git commit -m "feat: add ingress-nginx namespace and HelmRelease"
git push

flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
flux get helmreleases -n ingress-nginx --watch
```

Verify:

```bash
kubectl get ns ingress-nginx
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

---

# Exercise 1: Install Flagger via FluxCD

Install Flagger using a HelmRelease managed by Flux.

---

## 1.1 Add Flagger HelmRepository Source

```yaml
# clusters/my-cluster/infrastructure/controllers/ns1.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: flagger-system
```

```yaml
# clusters/my-cluster/infrastructure/sources/flagger.yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: flagger
  namespace: flagger-system
spec:
  interval: 1h
  type: oci
  url: oci://ghcr.io/fluxcd/charts
```

### Commit & Verify

```bash
git add .
git commit -m "feat: add flagger HelmRepository source"
git push

flux reconcile source git flux-system
flux get sources helm
```

---

## 1.2 Install Flagger (NGINX Provider)

```yaml
# clusters/my-cluster/infrastructure/controllers/flagger.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: flagger
  namespace: flagger-system
spec:
  releaseName: flagger
  interval: 12h
  install:
    createNamespace: true
    crds: CreateReplace
    remediation:
      retries: 3
  upgrade:
    crds: CreateReplace
    remediation:
      retries: 3
  chart:
    spec:
      chart: flagger
      version: "1.x"
      sourceRef:
        kind: HelmRepository
        name: flagger
        namespace: flagger-system
  values:
    meshProvider: nginx
```

```bash
git add clusters/my-cluster/infrastructure/controllers/flagger.yaml
git commit -m "feat: add flagger HelmRelease (nginx provider)"
git push

flux reconcile source git flux-system
flux reconcile helmrelease flagger -n flagger-system
flux get helmreleases -n flagger-system --watch

kubectl get pods -n flagger-system
```

---

# Exercise 2: Configure a Canary Resource

---

## 2.1 Deploy Sample Application

```yaml
#clusters/my-cluster/apps/team-alpha/nse.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
# clusters/my-cluster/apps/team-alpha/podinfo-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: team-alpha
  labels:
    app: podinfo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfo
        image: ghcr.io/stefanprodan/podinfo:6.5.0 # {"$imagepolicy": "flux-system:podinfo"}
        ports:
        - containerPort: 9898
        readinessProbe:
          httpGet:
            path: /readyz
            port: 9898
          initialDelaySeconds: 5
```

> **Image Policy Marker**
> The comment `# {"$imagepolicy": "flux-system:podinfo"}` enables Flux image automation.

```bash
mkdir -p clusters/my-cluster/apps/team-alpha
git add clusters/my-cluster/apps/team-alpha/podinfo-deployment.yaml
git commit -m "feat: add podinfo deployment for canary testing"
git push

flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source

kubectl get pods -n team-alpha
```

---

## 2.2 Create Flagger Canary Resource

```yaml
# clusters/my-cluster/apps/team-alpha/podinfo-canary.yaml
---
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: team-alpha
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  ingressRef:
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    name: podinfo
  service:
    port: 80
    targetPort: 9898
  analysis:
    interval: 1m
    iterations: 10
    threshold: 5
```

```yaml
# clusters/my-cluster/apps/team-alpha/podinfo-ingress.yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo
  namespace: team-alpha
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: podinfo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: podinfo
            port:
              number: 80
```

```bash
git add clusters/my-cluster/apps/team-alpha/podinfo-canary.yaml
git commit -m "feat: add podinfo Canary resource for progressive delivery"
git push

flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source

kubectl get canary -n team-alpha
kubectl describe canary podinfo -n team-alpha | tail -20
```

---

## Manual Testing (Non-GitOps — For Testing Only)

```bash
kubectl set image deployment/podinfo podinfo=ghcr.io/stefanprodan/podinfo:6.5.1 -n team-alpha

kubectl get events -n team-alpha --field-selector reason=Synced -w

kubectl logs -n flagger-system deploy/flagger -f | grep podinfo
```

---

# 0.3 Add Kustomization Entry for NGINX Ingress

```yaml
# clusters/my-cluster/infrastructure/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - sources/flagger.yaml
  - sources/nginx-ingress.yaml
  - controllers/nginx-ingress-namespace.yaml
  - controllers/ns1.yaml
  - controllers/nginx-ingress.yaml
  - controllers/flagger.yaml
```

```bash
git add clusters/my-cluster/infrastructure/kustomization.yaml
git commit -m "chore: register nginx-ingress namespace + controller in kustomization"
git push

flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source

kubectl get pods -n ingress-nginx
kubectl get pods -n flagger-system
```

Both should show **Running** before proceeding.

---

If you'd like, I can also:

* Generate a downloadable `.md` file
* Generate a PDF lab guide
* Convert this into a student-friendly training manual
* Create a diagram explaining the Canary flow visually
