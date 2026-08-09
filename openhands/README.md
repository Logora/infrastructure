# OpenHands (Agent Canvas) on Kubernetes

This folder contains everything needed to deploy **Agent Canvas** — the
self-hosted, single-tenant OpenHands deployment (web UI + agent-server +
automations in one all-in-one image) — on Kubernetes, protected in front by
**oauth2-proxy** authenticating against a **GitHub organization**.

This guide is intentionally **cloud-agnostic**: it works on any Kubernetes
cluster (managed or not) that has an nginx ingress controller, cert-manager
and a block StorageClass. Cluster-specific values are shown as examples to
adapt, not to copy blindly.

## Architecture

```
Internet
   │  https://agent.logora.com
   ▼
nginx Ingress (TLS via cert-manager)
   │  auth_url / auth_signin (auth_request)
   ▼
oauth2-proxy ── GitHub OAuth (org "Logora") → only org members pass
   │
   ▼
Agent Canvas (StatefulSet, PVC for persistent state)
```

- **Agent Canvas**: official Helm chart, deployed as a `StatefulSet` (UI +
  agent-server + automations) with one `PersistentVolumeClaim`.
- **oauth2-proxy**: GitHub OAuth gateway; only members of the GitHub org
  `Logora` can reach the UI.
- **Dedicated node**: both workloads are pinned to a dedicated, tainted
  node so agent code execution does not compete with (or get evicted by)
  your other applications.

## Prerequisites

- Kubernetes **1.24+**
- **Helm 3.x** and `kubectl` configured against the target cluster
- **nginx ingress controller** installed
- **cert-manager** installed (automatic TLS)
- A **ReadWriteOnce block StorageClass** for the PVC (update
  `persistence.storageClassName` in `values.yaml` — OVH example:
  `csi-cinder-high-speed`)
- A **GitHub OAuth App** (see below)
- An **OpenRouter API key**

## Setup

### 1. GitHub OAuth App

1. GitHub → **Settings → Developer settings → OAuth Apps → New OAuth App**.
2. Set:
   - **Homepage URL**: `https://agent.logora.com`
   - **Authorization callback URL**: `https://agent.logora.com/oauth2/callback`
3. Copy the **Client ID** and **Client Secret** into your local `.env`
   (see `.env.example`).
4. Membership is restricted to the **`Logora`** org via `--github-org=Logora`
   in `oauth2-proxy/deployment.yaml`. Adjust if you want a different org or
   team.

### 2. Namespace and secrets

```bash
kubectl create namespace openhands
```

Create the secrets from a `.env` file (copy `.env.example` first and fill
it in):

```bash
# LLM credentials (OpenRouter)
kubectl create secret generic openhands-llm \
  --namespace openhands --from-env-file=.env

# GitHub OAuth credentials for oauth2-proxy
kubectl -n openhands create secret generic openhands-github-oauth \
  --from-literal=OAUTH2_PROXY_CLIENT_ID="$(grep OAUTH2_PROXY_CLIENT_ID .env | cut -d= -f2)" \
  --from-literal=OAUTH2_PROXY_CLIENT_SECRET="$(grep OAUTH2_PROXY_CLIENT_SECRET .env | cut -d= -f2)" \
  --from-literal=OAUTH2_PROXY_COOKIE_SECRET="$(grep OAUTH2_PROXY_COOKIE_SECRET .env | cut -d= -f2)"
```

> All secrets can instead be created out of band (Vault, SealedSecrets,
> your GitOps operator) and simply referenced by name — the manifests only
> read them via `secretKeyRef`.

### 3. Dedicated node

Pin everything to a dedicated node so agent workloads are isolated:

```bash
# Label + taint the target node (replace <node-name>)
kubectl label node <node-name> node-role.openhands.infra=true
kubectl taint nodes <node-name> openhands=true:NoSchedule
```

> **Persistence caveat**: a taint/label applied with `kubectl` is lost if
> the node is later **replaced** by your provider (maintenance, resize,
> autoscaling). To make it survive, define the taint/label at the **node
> pool template** level if your provider supports it (e.g. OVH
> `template.metadata` on the node pool, or your cloud's equivalent). A node
> **reboot** alone does not lose a kubectl-applied taint (it lives in etcd).
>
> Keep `min_nodes` = `max_nodes` on the pool if you don't want the node
> scaled down/replaced unnecessarily.

### 4. Install Agent Canvas

There is **no published Helm repo** — the chart lives in the OpenHands
source tree. Clone it and install from the local path:

```bash
git clone https://github.com/OpenHands/OpenHands.git
cd OpenHands

helm install agent-canvas ./helm/agent-canvas \
  --namespace openhands \
  --values /path/to/openhands/values.yaml
```

Upgrade later with:

```bash
helm upgrade agent-canvas ./helm/agent-canvas \
  --namespace openhands --values /path/to/openhands/values.yaml
```

### 5. Deploy oauth2-proxy

```bash
kubectl apply -f oauth2-proxy/service.yaml
kubectl apply -f oauth2-proxy/deployment.yaml
kubectl apply -f oauth2-proxy/ingress.yaml
```

## Verification

```bash
kubectl -n openhands rollout status statefulset/agent-canvas
kubectl -n openhands get pvc,pod,svc,ingress
kubectl -n openhands get pods -l app=openhands-oauth2-proxy

# Confirm the dedicated node scheduling
kubectl -n openhands get pods -o wide
```

Browse to `https://agent.logora.com` — you should be redirected to GitHub
login and only members of the `Logora` org get through.

## LLM model

The LLM is configured in the **Agent Canvas UI** (Settings → LLM), which
stores it durably on the PVC (`~/.openhands`) and survives restarts/upgrades.

Suggested starter (OpenRouter):

- **Provider**: OpenRouter (OpenAI-compatible / LiteLLM)
- **Base URL**: `https://openrouter.ai/api/v1`
- **Model**: `openrouter/deepseek/deepseek-v4-flash-0731` (OpenRouter model
  ID for DeepSeek v4 Flash)
- **API key**: an OpenRouter key `sk-or-…`

For heavier agentic coding you can switch to a stronger model, e.g.
`openrouter/anthropic/claude-opus-5` (Claude Opus 5), in the UI at any time.
OpenRouter exposes every model under the `openrouter/<id>` prefix.

To configure purely via environment instead of the UI, uncomment the
`config.extraEnv` block at the top of `values.yaml` (it references the
`openhands-llm` secret) and run the `helm upgrade` above.

## GitHub integration & PR automation

### GitHub token (required)

OpenHands (OSS) authenticates to GitHub via a Personal Access Token. Add it
in the UI: **Settings → Integrations → GitHub Token**.

1. Create a **fine-grained PAT** (Developer settings → Personal access
   tokens → Fine-grained tokens), owned by the `Logora` org.
2. Scope it to the repositories you want.
3. Permissions (all "Read and write" unless noted):
   - Contents (create branches / commits / push)
   - Pull requests (open/manage PRs)
   - Issues (comment on tickets)
   - Metadata (read-only, mandatory)
4. If the org enforces it, click **Enable SSO** next to the org.
5. Paste the token in **Settings → Integrations → GitHub Token** and save.

OpenHands exports this as `GITHUB_TOKEN` into the agent shell, so `git push`,
`gh pr`, comments and PRs all work from the workspace.

### Auto-PR on @mention (issues / PRs)

Trigger: when the bot is @mentioned in a comment on an **issue or a
pull-request**, the agent works on it and opens a PR. The most robust
self-contained approach for OSS is a **repository GitHub Actions workflow**
(runs where the code lives, no need to expose a webhook to Agent Canvas).

A ready-to-adapt template is provided in
[`github-action/auto-pr-on-mention.yml`](./github-action/auto-pr-on-mention.yml).
To enable it in a repository:

1. Copy the file to `<repo>/.github/workflows/auto-pr-on-mention.yml`.
2. Make sure the workflow's `@mention` (e.g. `@openhands` / `/openhands`)
   matches how your team summons the agent.
3. Complete the agent invocation step in the workflow (the placeholder) to
   actually run the agent against the referenced issue/PR and open a PR.
4. If the repository uses a service account bot to push, create a token with
   `repo` scope and store it as an Actions secret instead of relying on the
   default `GITHUB_TOKEN`.

The token from [Settings → Integrations → GitHub Token](#github-token-required)
is what the agent itself uses inside Agent Canvas to push/PR from its
conversations.

## Uninstall

```bash
# Remove the Helm release (PVC is retained by StatefulSet — delete explicitly to reset)
helm uninstall agent-canvas -n openhands
kubectl -n openhands delete pvc -l app.kubernetes.io/instance=agent-canvas

# Remove oauth2-proxy
kubectl delete -f oauth2-proxy/ingress.yaml
kubectl delete -f oauth2-proxy/deployment.yaml
kubectl delete -f oauth2-proxy/service.yaml

# Delete secrets + namespace
kubectl -n openhands delete secret openhands-llm openhands-github-oauth
kubectl delete namespace openhands

# Remove the node taint/label if no longer needed
kubectl taint nodes <node-name> openhands=true:NoSchedule-
kubectl label node <node-name> node-role.openhands.infra-
```

## Troubleshooting

- **WebSocket disconnects / UI drops mid-turn**: the ingress closes idle
  streams. Ensure the read/send timeout annotations (`proxy-read-timeout`,
  `proxy-send-timeout`) are set to at least `3600`.
- **Auth loop / 401**: check the GitHub OAuth App callback URL matches
  exactly, and that the requesting user is a member of the `Logora` org.
  Check oauth2-proxy logs: `kubectl -n openhands logs -l app=openhands-oauth2-proxy`.
- **Pod stuck Pending**: `kubectl -n openhands describe pod <pod>` — usually
  a missing/unschedulable StorageClass or the dedicated node not having the
  expected taint/label (see the persistence caveat above).
- **OOMKill / restarts**: check `kubectl -n openhands top pods` and increase
  `resources.limits.memory` in `values.yaml`.

## Security notes

- Agent Canvas (open-source) is **unauthenticated, single-tenant**: all
  agents share one pod/filesystem. **oauth2-proxy** provides the access
  control, but the pod itself is still shared state — treat the namespace
  as trusted.
- The agent server can read/write the pod filesystem and run shell commands.
  Do not grant `rbac` unless you explicitly want the agent to mutate the
  cluster, and scope it to specific namespaces only.
- OpenHands' own "native GitHub auth" (SSO/RBAC, per-user isolation) is a
  commercial (**Enterprise**) feature, not part of this open-source chart.
