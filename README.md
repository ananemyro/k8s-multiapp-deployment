# Deploying a Multi‑App Stack on Kubernetes: Three Approaches (kubectl, Helm, Helmfile)

> **Stack used in examples**: PostgreSQL, Java/Spring backend (`kanban-app`), Angular frontend (`kanban-ui`), and Adminer.

This repo summarizes three common ways to deploy the same multi‑service application to Kubernetes:

1. **kubectl + raw YAML** (most explicit; best for learning)
2. **Helm** (templated, DRY manifests; reusable per app)
3. **Helmfile** (environment‑level orchestration of many Helm releases)

It also includes notes on Ingress, Services, PVCs, and ConfigMaps, and calls out what changed since the original 2020 tutorials (some resource names/CRDs/ingress annotations evolved).

---

## Why three approaches?

- **kubectl** teaches the building blocks (Deployment, Service, Ingress, PVC, ConfigMap) and how they relate.
- **Helm** removes repetition via templates and per‑app `values.yaml`.
- **Helmfile** lets you describe the *entire* cluster state (apps + ingress controller) in one file and sync in one command.

---

## Architecture (applies to all three)

- **Deployments** manage Pods (replicas, rolling updates).
- **Services (ClusterIP)** provide stable DNS for Pods (load balance across replicas).
- **Ingress** routes external traffic (host/path) to Services (requires an Ingress Controller).
- **PVC** persists Postgres data.
- **ConfigMap** centralizes shared configuration (DB creds, URLs, etc.).

### Example internal wiring

- UI → `http://kanban-app:8080` (Service DNS)
- Adminer → connects to Postgres via `postgres` Service name
- Ingress → maps `adminer.k8s.local` and `kanban.k8s.local` → Services

> For local dev, map hostnames in `/etc/hosts` to `127.0.0.1` and enable an Ingress Controller (Minikube addon or NGINX Helm chart).

---

## Approach 1 — kubectl (raw YAML)

**When to use:** learning, quick prototypes, ultra‑fine control.

**Pros**

- Maximum transparency and control.
- No extra tooling beyond `kubectl`.

**Cons**

- Many files to maintain (duplicated values).
- Changing one env var may require edits in several manifests.

**Quickstart**

```bash
minikube start
minikube addons enable ingress
kubectl apply -f k8s/
```

**What you’ll define**

- `Deployment` for each component (Adminer, Postgres, backend, UI)
- `Service` (ClusterIP) for each
- `ConfigMap` with shared env
- `PersistentVolumeClaim` for Postgres
- `Ingress` with host/path rules

---

## Approach 2 — Helm

**When to use:** repeated deployments; DRY templates; parameterized installs.

**Pros**

- Templates eliminate repetition (one chart → many apps).
- Per‑app overrides via `values.yaml`.
- Versioned releases with `helm list/upgrade/rollback`.

**Cons**

- Some template learning curve.
- Still one `helm install/upgrade` per app.

**Quickstart**

```bash
# Example: install Postgres and three stateless apps + ingress rules
helm install -f values/kanban-postgres.yaml postgres ./charts/postgres
helm install -f values/adminer.yaml         adminer  ./charts/app
helm install -f values/kanban-app.yaml      backend  ./charts/app
helm install -f values/kanban-ui.yaml       ui       ./charts/app
helm install -f values/ingress.yaml         ingress  ./charts/ingress
```

**Chart roles**

- `charts/postgres` → Deployment, PVC, ConfigMap, Service
- `charts/app` → generic Deployment + Service for stateless apps
- `charts/ingress` → just routing rules (keep controller separate)

---

## Approach 3 — Helmfile

**When to use:** multi‑service stacks, multiple environments, GitOps‑style workflows.

**Pros**

- One place (`helmfile.yaml`) to describe all releases (apps + ingress controller).
- `helmfile sync` applies drifted/changed state atomically.
- Clear separation: charts vs values vs orchestration.

**Cons**

- Adds another tool (but simplifies day‑2 operations greatly).

**Quickstart**

```bash
helmfile repos   # add chart repos
helmfile sync    # install/upgrade all releases (apps + nginx-ingress + routing)
```

**Recommended split**

- Install NGINX Ingress Controller as its **own Helm release**.
- Keep `charts/ingress` strictly for routing rules (Hosts/Paths → Services).

---

## Ingress options compared

| Approach | How Controller is Provided                                                  | Why This Choice                                 |
| -------- | --------------------------------------------------------------------------- | ----------------------------------------------- |
| kubectl  | Minikube addon (built‑in)                                                   | Simple for local dev                            |
| Helm     | (Option A) dependency in a parent chart; **or** (Option B) separate release | (B) preferred for clearer ownership             |
| Helmfile | Separate Helm release + routing chart                                       | Clean separation and full orchestration control |

---

## What changed since 2020

- Ingress classes/annotations evolved; prefer `kubernetes.io/ingress.class: nginx` or `ingressClassName` where supported.
- Some chart names/locations moved; use `helmfile repos` or `helm repo add` to pin sources.
- Container images and tags updated; keep them in `values/*.yaml`.

---

## Attribution

This repo summarizes and re‑implements a multi‑part tutorial series (updated for 2025 APIs/tooling) and cites the following sources:

- “Deployment of multiple apps on Kubernetes cluster — walkthrough”: https://wkrzywiec.medium.com/deployment-of-multiple-apps-on-kubernetes-cluster-walkthrough-e05d37ed63d1
- “How to deploy application on Kubernetes with Helm”: https://wkrzywiec.medium.com/how-to-deploy-application-on-kubernetes-with-helm-39f545ad33b8
- “How to declaratively run Helm charts using Helmfile”: https://medium.com/swlh/how-to-declaratively-run-helm-charts-using-helmfile-ac78572e6088

