# Sealed Secrets + FluxCD Lab Guide
> Encrypt Kubernetes Secrets with Bitnami Sealed Secrets and GitOps Automation via Flux

**3 Labs · 15+ Commands · Production-Ready**

---

## Table of Contents

- [Overview](#overview)
- [Lab 1 — Install Sealed Secrets](#lab-1--install-sealed-secrets)
- [Lab 2 — Seal & Commit Secrets](#lab-2--seal--commit-secrets)
- [Lab 3 — Flux GitOps Integration](#lab-3--flux-gitops-integration)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Overview

### How Sealed Secrets Works

```
Developer                 Git Repository           Kubernetes Cluster
─────────────            ────────────────         ──────────────────
Plain Secret YAML   →    SealedSecret YAML   →    Decrypted Secret
(never committed)        (safe to commit)         (applied by controller)
```

Sealed Secrets uses **asymmetric cryptography**:

- The **public key** (fetched from the cluster) encrypts secrets — safe to share.
- The **private key** (managed by the controller in-cluster) decrypts secrets — never leaves the cluster.
- A `SealedSecret` is a Kubernetes CRD. The controller watches for it, decrypts it, and produces a standard `Secret`.

### Sealed Secrets vs SOPS

| Feature | Sealed Secrets | SOPS + age |
|---|---|---|
| Key management | In-cluster controller | External (age key / KMS) |
| Encryption tool | `kubeseal` CLI | `sops` CLI |
| Output format | `SealedSecret` CRD | Encrypted YAML |
| Cluster dependency | Yes (needs controller for sealing) | No (encrypt offline) |
| Key rotation | Controller handles | Manual re-encryption |

---

## Lab 1 — Install Sealed Secrets

**Objective:** Deploy the Sealed Secrets controller to your cluster and install the `kubeseal` CLI.

**Estimated Time:** 20 minutes

### Step 1 — Deploy the Controller via Flux

Add the Sealed Secrets Helm release to your `fleet-infra` repository so Flux manages the controller.

```bash
# Create the directory structure for infrastructure components
mkdir -p clusters/my-cluster/infrastructure

# Create the HelmRepository source
cat > clusters/my-cluster/infrastructure/sealed-secrets-helmrepo.yaml <<EOF
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: sealed-secrets
  namespace: flux-system
spec:
  interval: 1h
  url: https://bitnami-labs.github.io/sealed-secrets
EOF

# Create the HelmRelease to deploy the controller
cat > clusters/my-cluster/infrastructure/sealed-secrets-release.yaml <<EOF
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: sealed-secrets
  namespace: flux-system
spec:
  interval: 1h
  chart:
    spec:
      chart: sealed-secrets
      version: '>=2.0.0'
      sourceRef:
        kind: HelmRepository
        name: sealed-secrets
        namespace: flux-system
  install:
    crds: Create
  upgrade:
    crds: CreateReplace
  values:
    fullnameOverride: sealed-secrets-controller
EOF
```

> **Why deploy via Flux?** This keeps the controller itself under GitOps management — version pinning, upgrades, and rollbacks all happen through Git.

---

### Step 2 — Commit and Push

```bash
git add clusters/my-cluster/infrastructure/
git commit -m 'feat: add Sealed Secrets controller via Flux HelmRelease'
git push

# Force reconciliation
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
```

---

### Step 3 — Verify the Controller is Running

```bash
# Watch the HelmRelease
flux get helmreleases -n flux-system

# Expected output:
# NAME             REVISION  SUSPENDED  READY  MESSAGE
# sealed-secrets   2.x.x     False      True   Release reconciliation succeeded

# Confirm the controller pod is running
kubectl get pods -n flux-system -l app.kubernetes.io/name=sealed-secrets

# Confirm the CRD was installed
kubectl get crd sealedsecrets.bitnami.com
```

---

### Step 4 — Install the `kubeseal` CLI

The `kubeseal` CLI fetches the controller's public key and encrypts secrets locally.

**Linux (x86_64):**

```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest \
  | grep tag_name | cut -d '"' -f 4 | sed 's/v//')

curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"

tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
install -m 755 kubeseal /usr/local/bin/kubeseal

kubeseal --version
```

---

### Step 5 — Fetch and Store the Public Key

Fetch the public certificate for offline encryption (useful in CI/CD pipelines).

```bash
# Fetch the public key from the running controller
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=flux-system \
  > pub-sealed-secrets.pem

# View the certificate
cat pub-sealed-secrets.pem

# Commit the public cert to Git — it is safe to share
git add pub-sealed-secrets.pem
git commit -m 'feat: add Sealed Secrets public certificate'
git push
```

> **Why commit the public cert?** It allows offline sealing and CI/CD pipelines to encrypt secrets without direct cluster access.

### ✅ Lab 1 Checklist

- [x] Sealed Secrets controller deployed via Flux HelmRelease
- [x] Controller pod running in `flux-system` namespace
- [x] `kubeseal` CLI installed and verified
- [x] Public certificate fetched and committed to Git

---

## Lab 2 — Seal & Commit Secrets

**Objective:** Create a plain Kubernetes Secret, encrypt it with `kubeseal`, and safely push the `SealedSecret` to Git.

**Estimated Time:** 25 minutes

### Step 1 — Create the Directory Structure

```bash
mkdir -p clusters/my-cluster/secrets
```

---

### Step 2 — Create a Plain Secret Manifest

Write the plaintext secret — this file will **never** be committed to Git.

```bash
cat > /tmp/db-credentials-plain.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
stringData:
  DATABASE_URL: postgresql://admin:super-secret-password@db.example.com:5432/mydb
  DATABASE_PASSWORD: super-secret-password
  API_KEY: sk-prod-abc123xyz789
EOF
```

> ⚠️ **Save to `/tmp/`** — not inside your Git repo. This file must never be committed.

---

### Step 3 — Seal the Secret with `kubeseal`

```bash
# Seal using the public cert (offline — no cluster access needed)
kubeseal \
  --cert pub-sealed-secrets.pem \
  --format yaml \
  < /tmp/db-credentials-plain.yaml \
  > clusters/my-cluster/secrets/db-credentials-sealed.yaml

# Inspect the output
cat clusters/my-cluster/secrets/db-credentials-sealed.yaml
```

The sealed output looks like this:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: default
spec:
  encryptedData:
    DATABASE_URL: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    DATABASE_PASSWORD: AgCIom5GqMHQnPkIW84fmBQmHpR5D...
    API_KEY: AgBDK2qHjy3CdTn7n6LmnMD0G1HjnkZD...
  template:
    metadata:
      name: db-credentials
      namespace: default
    type: Opaque
```

> ✅ The `SealedSecret` is safe to commit. The encrypted values can only be decrypted by the controller's private key inside your cluster.

---

### Step 4 — Understanding Scoping

`kubeseal` supports three encryption scopes. Choose based on your security requirements.

| Scope | Description | Flag |
|---|---|---|
| `strict` (default) | Bound to secret name **and** namespace — cannot be renamed or moved | `--scope strict` |
| `namespace-wide` | Bound to namespace only — can be renamed within the same namespace | `--scope namespace-wide` |
| `cluster-wide` | No binding — can be used in any namespace | `--scope cluster-wide` |

```bash
# Example: namespace-wide scope
kubeseal \
  --cert pub-sealed-secrets.pem \
  --scope namespace-wide \
  --format yaml \
  < /tmp/db-credentials-plain.yaml \
  > clusters/my-cluster/secrets/db-credentials-sealed.yaml
```

---

### Step 5 — Commit and Push

```bash
git add clusters/my-cluster/secrets/db-credentials-sealed.yaml
git commit -m 'feat: add sealed db-credentials secret'
git push

# Clean up the plaintext file
rm /tmp/db-credentials-plain.yaml
```

---

### Step 6 — Update a Sealed Secret

To update a value, re-create the plain secret with new values and re-seal it.

```bash
# Create updated plaintext in /tmp
cat > /tmp/db-credentials-updated.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
stringData:
  DATABASE_URL: postgresql://admin:NEW-password@db.example.com:5432/mydb
  DATABASE_PASSWORD: NEW-password
  API_KEY: sk-prod-newkey456
EOF

# Re-seal and overwrite the committed file
kubeseal \
  --cert pub-sealed-secrets.pem \
  --format yaml \
  < /tmp/db-credentials-updated.yaml \
  > clusters/my-cluster/secrets/db-credentials-sealed.yaml

git add clusters/my-cluster/secrets/db-credentials-sealed.yaml
git commit -m 'chore: rotate db-credentials secret'
git push

rm /tmp/db-credentials-updated.yaml
```

### ✅ Lab 2 Checklist

- [x] Plain secret created in `/tmp/` (never in Git)
- [x] Secret sealed with `kubeseal` using the public certificate
- [x] `SealedSecret` YAML committed to fleet-infra
- [x] Plaintext file deleted after sealing

---

## Lab 3 — Flux GitOps Integration

**Objective:** Configure a Flux `Kustomization` to automatically apply `SealedSecret` resources from Git, triggering the controller to decrypt and create the live `Secret`.

**Estimated Time:** 30 minutes

### How the Full Flow Works

```
Git Push                 Flux                    Sealed Secrets Controller
─────────                ────                    ──────────────────────────
SealedSecret YAML  →  source-controller    →  Detects new SealedSecret CRD
                        pulls from Git          Decrypts with private key
                    →  kustomize-controller →   Creates plain Secret in cluster
                        applies YAML            Secret available to Pods
```

---

### Step 1 — Create a Kustomization for Secrets

```bash
cat > clusters/my-cluster/secrets-kustomization.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: secrets
  namespace: flux-system
spec:
  interval: 5m
  path: ./clusters/my-cluster/secrets
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  healthChecks:
    - apiVersion: v1
      kind: Secret
      name: db-credentials
      namespace: default
EOF
```

> **Note:** Unlike SOPS integration, Sealed Secrets requires **no** `decryption` block in the Kustomization. The `SealedSecret` controller handles decryption automatically as a CRD controller — Flux simply applies the manifest.

---

### Step 2 — Commit and Push

```bash
git add clusters/my-cluster/secrets-kustomization.yaml
git commit -m 'feat: add secrets Kustomization for Sealed Secrets'
git push

# Force immediate reconciliation
flux reconcile kustomization flux-system --with-source
```

---

### Step 3 — Watch the Reconciliation

```bash
# Watch all Kustomizations
flux get kustomizations --watch

# Expected output:
# NAME     REVISION        SUSPENDED  READY  MESSAGE
# secrets  main/abc123     False      True   Applied revision: main/abc123

# Confirm the SealedSecret was created
kubectl get sealedsecret db-credentials -n default

# Confirm the controller decrypted it into a plain Secret
kubectl get secret db-credentials -n default

# Verify the decrypted value
kubectl get secret db-credentials -n default \
  -o jsonpath='{.data.DATABASE_PASSWORD}' | base64 -d
```

---

### Step 4 — Inspect the Controller Logs

```bash
# View Sealed Secrets controller logs
kubectl logs -n flux-system \
  -l app.kubernetes.io/name=sealed-secrets \
  -f | grep -i "unsealed\|error"

# Expected log line on success:
# {"event":"Unsealed","namespace":"default","name":"db-credentials"}
```

---

### Step 5 — Key Rotation (Production Practice)

The controller automatically generates a new sealing key every **30 days**. Old keys are retained so existing `SealedSecrets` can still be decrypted.

```bash
# List all sealing keys in the cluster
kubectl get secrets -n flux-system -l sealedsecrets.bitnami.com/sealed-secrets-key

# Manually trigger key rotation (if needed)
kubectl delete secret -n flux-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active

# After rotation, re-fetch the new public cert
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=flux-system \
  > pub-sealed-secrets.pem

git add pub-sealed-secrets.pem
git commit -m 'chore: rotate Sealed Secrets public cert'
git push
```

> ⚠️ **After key rotation**, re-seal all existing secrets with the new certificate to ensure they can be decrypted if old keys are ever removed.

### ✅ Lab 3 Checklist

- [x] Kustomization created for the secrets path (no `decryption` block needed)
- [x] Flux successfully applied the `SealedSecret` manifest
- [x] Controller decrypted the `SealedSecret` into a plain `Secret`
- [x] Controller logs confirm successful unsealing
- [x] Key rotation strategy understood

---

## Quick Reference Cheat Sheet

| Task | Command |
|---|---|
| Install kubeseal | `curl -OL https://github.com/bitnami-labs/sealed-secrets/releases/...` |
| Fetch public cert | `kubeseal --fetch-cert --controller-name=sealed-secrets-controller --controller-namespace=flux-system > pub.pem` |
| Seal a secret | `kubeseal --cert pub-sealed-secrets.pem --format yaml < plain.yaml > sealed.yaml` |
| Seal (namespace-wide) | `kubeseal --cert pub-sealed-secrets.pem --scope namespace-wide --format yaml < plain.yaml > sealed.yaml` |
| Apply a SealedSecret | `kubectl apply -f sealed.yaml` |
| Verify decryption | `kubectl get secret <name> -n <ns> -o jsonpath='{.data.<key>}' \| base64 -d` |
| List sealing keys | `kubectl get secrets -n flux-system -l sealedsecrets.bitnami.com/sealed-secrets-key` |
| Force Flux reconcile | `flux reconcile kustomization flux-system --with-source` |
| Watch kustomizations | `flux get kustomizations --watch` |
| Controller logs | `kubectl logs -n flux-system -l app.kubernetes.io/name=sealed-secrets -f` |

---

> **Security Reminder:** Never commit plaintext `Secret` YAML to Git. Only commit `SealedSecret` resources. Store your plain YAML in `/tmp/` and delete it immediately after sealing.
