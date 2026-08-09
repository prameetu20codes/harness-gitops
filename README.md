# Harness GitOps Demo — multi-project, multi-app

A hands-on GitOps setup to learn how a **single account-level agent** serves **multiple Argo Projects**,
how those map to **downstream Harness Projects**, and how apps **communicate across namespaces/projects**.

Everything here is generated locally. You push `manifests-repo/` to Git, then wire the rest up in the Harness UI.

## What gets built

| Argo Project      | Harness Project | Application(s)                | Namespace ("place") | Talks to                        |
|-------------------|-----------------|-------------------------------|---------------------|---------------------------------|
| `web-project`     | `web`           | `web-dev`, `web-prod`         | `web-dev`, `web-prod` | `api` tier (`/api/` proxy)      |
| `api-project`     | `api`           | `api-dev`, `api-prod`         | `api-dev`, `api-prod` | — (backend)                     |
| `platform-project`| `platform`      | `platform-gateway`            | `platform`          | `web` + `api` tiers             |
| `shared-project`  | `platform`      | `shared-redis`                | `platform`          | shared infra (used by others)   |

> Note the last two rows: **two Argo Projects map to one Harness Project (`platform`)**. That is intentional —
> it shows how a single agent partitions many Argo Projects onto Harness's project/RBAC model.

## Topology

```text
                         Account-level GitOps Agent (in cluster, namespace: gitops)
                         ├── App Controller / Repo Server / Redis / ApplicationSet
                         │
   Git repo (this)  ───► reconciled into destinations below
                         │
   web-project ──► web-dev (ns web-dev) ──/api/──► api.api-dev
                └► web-prod (ns web-prod) ─/api/──► api.api-prod
   api-project ──► api-dev (ns api-dev)
                └► api-prod (ns api-prod)
   platform-project ──► platform-gateway (ns platform) ──/web/──► web.web-prod
                                                        └─/api/──► api.api-prod
   shared-project   ──► shared-redis (ns platform)   (redis.platform.svc)

   web-project  ─┐
   api-project  ─┼─ mapped to Harness Projects: web, api, platform (platform x2)
   platform-proj ┘
```

Communication is plain Kubernetes Service DNS: `<service>.<namespace>.svc.cluster.local`.

## Repo layout

```text
manifests-repo/        # PUSH THIS TO GIT — the GitOps source of truth
  api/                 # nginx serving a JSON payload (backend)
    base/ overlays/{dev,prod}
  web/                 # nginx frontend, reverse-proxies /api/ to the api tier
    base/ overlays/{dev,prod}
  platform-gateway/    # nginx gateway routing /web/ and /api/
    base/
  shared-config/       # redis (shared backing service)
    base/
argocd/                # REFERENCE for what to create in the Harness UI
  appprojects/         # the 4 Argo Projects (AppProjects)
  applications/        # the 6 Applications
```

## Before you start — edit these placeholders

1. In every file under `argocd/`, replace `https://github.com/YOUR_ORG/harness-gitops-demo` with your repo URL.
2. In `argocd/appprojects/*.yaml`, `metadata.namespace: gitops` should match the namespace where your agent is installed.
3. Destination `https://kubernetes.default.svc` = the same cluster the agent runs in (in-cluster). To deploy to a
   different cluster ("another place"), register that cluster and use its server URL instead.

## Step-by-step in the Harness UI

Do these in order — dependencies flow top to bottom.

1. **Create the GitOps Agent (account level).**
   `Account Settings → GitOps → Agents → New GitOps Agent`. Choose Argo, install into namespace `gitops`,
   apply the generated YAML in your cluster, and wait for **Healthy / Connected** (like the panel in your screenshot).

2. **Create the downstream Harness Projects.** Create `web`, `api`, and `platform` under your org.

3. **Register the Repository.** Under the agent’s GitOps settings → `Repositories`, add your Git repo URL
   (the one holding `manifests-repo/`). Use HTTPS + token or SSH key.

4. **Register Cluster(s).** Under `Clusters`, add the in-cluster destination (`https://kubernetes.default.svc`).
   Add extra clusters here if you want apps in other physical "places".

5. **Create the 4 Argo Projects.** Under the agent → `Argo Projects`, create `web-project`, `api-project`,
   `platform-project`, `shared-project`. Use `argocd/appprojects/*.yaml` as the source of truth for
   `sourceRepos`, `destinations`, and resource whitelists.

6. **Map Argo Projects → Harness Projects.** On the agent overview (the "Mapped Harness Project" table in your
   screenshot), map:
   - `web-project` → `web`
   - `api-project` → `api`
   - `platform-project` → `platform`
   - `shared-project` → `platform`
   Leave **Auto Create** off for manual control (turn it on if you want new Argo Projects to auto-associate).

7. **Create the 6 Applications.** Under each mapped Harness Project → `GitOps → Applications → New Application`.
   Use `argocd/applications/*.yaml` for the repo path, target revision, project, and destination namespace.
   Recommended sync order (respects communication deps):
   1. `shared-redis`  2. `api-dev`, `api-prod`  3. `web-dev`, `web-prod`  4. `platform-gateway`
   Sync policy is automated with `CreateNamespace=true`, so namespaces are created for you.

## Verify communication

After all apps are Synced + Healthy:

```bash
# web -> api (cross-namespace proxy). Should return the api JSON payload.
kubectl -n web-dev port-forward svc/web 8080:80
curl -s localhost:8080/api/         # -> {"service":"api",...}
curl -s localhost:8080/             # -> web frontend HTML

# platform gateway -> web and api (cross-project routing)
kubectl -n platform port-forward svc/platform-gateway 8081:80
curl -s localhost:8081/             # -> gateway landing text
curl -s localhost:8081/web/         # -> web frontend HTML (from web-prod)
curl -s localhost:8081/api/         # -> api JSON (from api-prod)

# shared redis reachable in the platform namespace
kubectl -n platform get svc redis
```

## Try these experiments

- **Guardrails:** remove `api-prod` from `api-project`'s `destinations`, re-sync `api-prod` — the sync is denied
  because the Argo Project no longer permits that destination. This is what an AppProject enforces.
- **RBAC / visibility:** give a user access only to Harness project `api`. They see only the `api-*` applications,
  not `web-*` — that is the mapping doing the scoping.
- **Another place:** register a second cluster, add it to a project's destinations, and point an overlay at it.
