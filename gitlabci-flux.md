# GitLab CI/CD + FluxCD — Complete GitOps Setup Guide (Module 3)

This guide explains how to build a **production-style GitOps pipeline** using:

* GitLab (CI/CD + Container Registry)
* FluxCD (GitOps Operator)
* Kubernetes (Cluster Runtime)

We use **two repositories**:

| Repository    | Purpose                                            |
| ------------- | -------------------------------------------------- |
| `my-app`      | Application source code + Dockerfile + CI pipeline |
| `fleet-infra` | Kubernetes manifests + Flux configuration          |

---

# 🏗 Architecture Overview

```
Developer pushes code
      │
      ▼
GitLab CI Pipeline
  ├── Stage 1: docker build + push → GitLab Container Registry
  └── Stage 2: update image tag in fleet-infra → git push
      │
      ▼
Flux polls Git (every 1m)
      │
      ▼
Kubernetes rolls out new image
```

---

# 📁 Section 1 — Source Files

---

## 1.1 `main.py`

📄 `my-app/main.py`

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'Hello from v1.0.0!')

HTTPServer(('', 8080), Handler).serve_forever()
```

---

## 1.2 `Dockerfile`

📄 `my-app/Dockerfile`

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY main.py .
EXPOSE 8080
CMD ["python", "main.py"]
```

---

## 1.3 `.gitlab-ci.yml`

Two-stage pipeline:

* Build & push Docker image
* Update manifest repo so Flux detects change

📄 `my-app/.gitlab-ci.yml`

```yaml
stages:
  - build
  - update-manifest

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build-image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE .
    - docker push $IMAGE
    - |
        if [ -n "$CI_COMMIT_TAG" ]; then
          docker tag $IMAGE $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
          docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
        fi
  only:
    - main

update-manifest:
  stage: update-manifest
  image: ubuntu
  only:
    - main
  before_script:
    - apt-get update -y && apt-get install -y git
  script:
    - |
        git config --global user.email 'ci@myorg.com'
        git config --global user.name 'GitLab CI'
        git clone https://oauth2:${MANIFEST_TOKEN}@gitlab.com/amitopenwriteup/fleet-infra1 fleet
        sed -i "s|image:.*|image: ${IMAGE}|g" fleet/apps/base/my-app/deployment.yaml
        cd fleet
        git add .
        git commit -m "chore: update image to ${CI_COMMIT_SHA}"
        git push
```

---

### CI Variables Explained

| Variable               | Description                              |
| ---------------------- | ---------------------------------------- |
| `MANIFEST_TOKEN`       | GitLab PAT with `write_repository` scope |
| `CI_REGISTRY_IMAGE`    | Auto: registry.gitlab.com/group/project  |
| `CI_COMMIT_SHA`        | Auto: full commit SHA                    |
| `CI_PROJECT_NAMESPACE` | Auto: GitLab namespace                   |

---

## 1.4 `deployment.yaml`

📄 `fleet-infra/apps/base/my-app/deployment.yaml`

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: my-app

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      imagePullSecrets:
        - name: gitlab-regcred
      containers:
        - name: my-app
          image: registry.gitlab.com/GITLAB_USER/my-app:latest # {"$imagepolicy": "flux-system:my-app"}
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"

---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

---

## 1.5 Flux Kustomization

📄 `fleet-infra/clusters/my-cluster/apps/my-app/flux-ks.yaml

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/base/my-app
  prune: true
  targetNamespace: my-app
```

---
#fleet-infra/clusters/my-cluster/apps/my-app/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1

kind: Kustomization

resources:

  - flux-ks.yaml


## 1.6 Image Automation

📄 `fleet-infra/clusters/my-cluster/apps/image-automation.yaml`

```yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  image: registry.gitlab.com/GITLAB_USER/my-app
  interval: 1m
  secretRef:
    name: gitlab-regcred

---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: my-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: my-app
  policy:
    alphabetical:
      order: asc

---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: my-app
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
        email: fluxbot@gitlab.com
        name: Flux Bot
      messageTemplate: 'chore: update my-app to {{range .Updated.Images}}{{.NewTag}}{{end}}'
    push:
      branch: main
  update:
    path: ./apps/base/my-app
    strategy: Setters
```

---

# 🚀 Section 2 — Step-by-Step Setup

---

## Step 1 — Export Variables

```bash
export GITLAB_USER=your-gitlab-username
export GITLAB_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
```

---

## Step 2 — Create GitLab Repositories

Create two blank projects:

```
https://gitlab.com/$GITLAB_USER/my-app
https://gitlab.com/$GITLAB_USER/fleet-infra
```

---

## Step 3 — Push Application Repository

```bash
cd my-app
git init && git add .
git commit -m 'feat: initial application'
git remote add origin https://gitlab.com/$GITLAB_USER/my-app
git push -u origin main
```

---

## Step 4 — Bootstrap Flux

```bash
flux bootstrap gitlab \
  --owner=$GITLAB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --token-auth
```

Enter your `GITLAB_TOKEN` when prompted.

---

## Step 5 — Push Infrastructure Manifests

```bash
git clone https://gitlab.com/$GITLAB_USER/fleet-infra
cd fleet-infra

mkdir -p clusters/my-cluster/apps/my-app
mkdir -p apps/base/my-app
```

Create cluster kustomizations and copy manifests.

Replace placeholder:

```bash
sed -i "s/GITLAB_USER/$GITLAB_USER/g" \
  apps/base/my-app/deployment.yaml \
  clusters/my-cluster/apps/image-automation.yaml
```

Commit & push:

```bash
git add .
git commit -m 'feat: add my-app manifests'
git push
```

---

## Step 6 — Create Kubernetes Secrets

### Git Auth for Flux

```bash
flux create secret git gitlab-fleet-auth \
  --namespace=flux-system \
  --url=https://gitlab.com/$GITLAB_USER/fleet-infra \
  --username=git \
  --password=$GITLAB_TOKEN
```

### Registry Secret (flux-system)

```bash
kubectl create secret docker-registry gitlab-regcred \
  --namespace=flux-system \
  --docker-server=registry.gitlab.com \
  --docker-username=$GITLAB_USER \
  --docker-password=$GITLAB_TOKEN \
  --docker-email=you@example.com
```

### Registry Secret (my-app namespace)

```bash
kubectl create secret docker-registry gitlab-regcred \
  --namespace=my-app \
  --docker-server=registry.gitlab.com \
  --docker-username=$GITLAB_USER \
  --docker-password=$GITLAB_TOKEN \
  --docker-email=you@example.com
```

---

## Step 7 — Add GitLab CI Variable

`my-app → Settings → CI/CD → Variables`

| Key              | Value                             |
| ---------------- | --------------------------------- |
| `MANIFEST_TOKEN` | PAT with `write_repository` scope |

Protected: Yes
Masked: Yes

---

## Step 8 — Trigger GitOps Flow

```bash
cd my-app
sed -i 's/Hello from v1.0.0/Hello from v2.0.0/' main.py
git add main.py
git commit -m 'feat: update greeting to v2.0.0'
git push origin main
```

---

## Step 9 — Monitor & Verify

```bash
flux get kustomizations --watch
flux reconcile source git flux-system
flux reconcile kustomization my-app
```

Check running image:

```bash
kubectl describe pod -n my-app -l app=my-app | grep Image:
```

Test app:

```bash
kubectl port-forward svc/my-app 8080:8080 -n my-app &
curl http://localhost:8080
```

Expected:

```
Hello from v2.0.0!
```

Confirm Flux Bot commit:

```bash
cd fleet-infra
git pull
git log --oneline -5
```

---

# ✅ Final Result

You now have a **fully automated GitOps pipeline**:

* Code push → CI builds image
* CI updates Git
* Flux detects change
* Kubernetes deploys new version

---

If you want, I can next create:

* 📊 Architecture diagram (visual)
* 🏗 Production-grade structure (dev/stage/prod)
* 🔐 Secure enterprise setup version
* 🎓 Trainer version explanation (step-by-step teaching mode)

Just tell me which version you want.
