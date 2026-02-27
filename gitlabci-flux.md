
GitLab CI/CD + FluxCD
GitOps Project  —  Complete Setup Guide
Module 3

This guide covers the complete setup of a GitOps pipeline using GitLab CI/CD and FluxCD. Two repositories are used: my-app for application code and fleet-infra for Kubernetes manifests.

Architecture Overview

Developer pushes code
  │
  ▼
GitLab CI Pipeline
  ├── Stage 1: docker build + push  →  GitLab Container Registry
  └── Stage 2: update image tag in fleet-infra  →  git push
  │
  ▼
Flux detects commit (polls every 1m)
  │
  ▼
Kubernetes rolls out new image

Repository	Purpose
my-app	Source code, Dockerfile, .gitlab-ci.yml
fleet-infra	Kubernetes manifests, Flux config, image automation

Section 1 — Source Files

1.1  main.py
📄  my-app/main.py
from http.server import HTTPServer, BaseHTTPRequestHandler
 
class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'Hello from v1.0.0!')
 
HTTPServer(('', 8080), Handler).serve_forever()

1.2  Dockerfile
📄  my-app/Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY main.py .
EXPOSE 8080
CMD ["python", "main.py"]

1.3  .gitlab-ci.yml
Two-stage pipeline: builds and pushes the Docker image, then updates the image tag in fleet-infra so Flux detects the change.

📄  my-app/.gitlab-ci.yml
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

Variable	Description
MANIFEST_TOKEN	GitLab PAT with write_repository scope
CI_REGISTRY_IMAGE	Auto-provided: registry.gitlab.com/<group>/<project>
CI_COMMIT_SHA	Auto-provided: full commit SHA
CI_PROJECT_NAMESPACE	Auto-provided: GitLab namespace

1.4  deployment.yaml
📄  fleet-infra/apps/base/my-app/deployment.yaml
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

1.5  Flux Kustomization
📄  fleet-infra/clusters/my-cluster/apps/my-app/kustomization.yaml
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

1.6  image-automation.yaml
📄  fleet-infra/clusters/my-cluster/apps/image-automation.yaml
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

Section 2 — Step-by-Step Setup

export GITLAB_USER=your-gitlab-username
export GITLAB_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx

1	Create Both GitLab Repositories

# Create two blank projects in GitLab UI:
# https://gitlab.com/$GITLAB_USER/my-app
# https://gitlab.com/$GITLAB_USER/fleet-infra

2	Push the Application Repository

cd my-app
git init && git add .
git commit -m 'feat: initial application'
git remote add origin https://gitlab.com/$GITLAB_USER/my-app
git push -u origin main

3	Bootstrap Flux into fleet-infra

flux bootstrap gitlab \
  --owner=$GITLAB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --token-auth
 
# Enter your GITLAB_TOKEN when prompted

4	Push fleet-infra Manifests

git clone https://gitlab.com/$GITLAB_USER/fleet-infra
cd fleet-infra
 
# Create folder structure
mkdir -p clusters/my-cluster/apps/my-app
mkdir -p apps/base/my-app
 
# Create the cluster-level kustomization.yaml
cat > clusters/my-cluster/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - apps
EOF
 
# Create the apps-level kustomization.yaml
cat > clusters/my-cluster/apps/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - my-app
EOF
 
# Copy manifest files from Section 1, then replace placeholder:
sed -i "s/GITLAB_USER/$GITLAB_USER/g" \
  apps/base/my-app/deployment.yaml \
  clusters/my-cluster/apps/image-automation.yaml
 
git add .
git commit -m 'feat: add my-app manifests'
git push

5	Create Kubernetes Secrets

# Git auth for Flux to read fleet-infra
flux create secret git gitlab-fleet-auth \
  --namespace=flux-system \
  --url=https://gitlab.com/$GITLAB_USER/fleet-infra \
  --username=git \
  --password=$GITLAB_TOKEN
 
# Registry auth for Flux image scanning (flux-system namespace)
kubectl create secret docker-registry gitlab-regcred \
  --namespace=flux-system \
  --docker-server=registry.gitlab.com \
  --docker-username=$GITLAB_USER \
  --docker-password=$GITLAB_TOKEN \
  --docker-email=you@example.com
 
# Registry auth for pods to pull images (my-app namespace)
kubectl create secret docker-registry gitlab-regcred \
  --namespace=my-app \
  --docker-server=registry.gitlab.com \
  --docker-username=$GITLAB_USER \
  --docker-password=$GITLAB_TOKEN \
  --docker-email=you@example.com

6	Add CI/CD Variable in GitLab

GitLab → my-app → Settings → CI/CD → Variables → Add variable

Key	Value / Notes
MANIFEST_TOKEN	Your PAT with write_repository scope  |  Protected: Yes  |  Masked: Yes

7	Trigger the Full GitOps Loop

cd my-app
sed -i 's/Hello from v1.0.0/Hello from v2.0.0/' main.py
git add main.py
git commit -m 'feat: update greeting to v2.0.0'
git push origin main

8	Monitor and Verify

# Watch Flux reconcile
flux get kustomizations --watch
 
# Force immediate reconciliation
flux reconcile source git flux-system
flux reconcile kustomization my-app
 
# Verify pod image
kubectl describe pod -n my-app -l app=my-app | grep Image:
 
# Test the app
kubectl port-forward svc/my-app 8080:8080 -n my-app &
curl http://localhost:8080
# → Hello from v2.0.0!
 
# Confirm Flux Bot committed to fleet-infra
cd fleet-infra && git pull && git log --oneline -5

