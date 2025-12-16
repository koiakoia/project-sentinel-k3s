# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Common commands

Bootstrap namespace and required node labels (from README):

```bash
kubectl create namespace sentinel --dry-run=client -o yaml | kubectl apply -f -
# Label hardware capabilities (adjust node names for your cluster)
kubectl label node node-gpu special.gpu=nvidia --overwrite
kubectl label node node-compute special.cpu=avx512 --overwrite
```

Apply or update all Kubernetes resources in this repo:

```bash
kubectl apply -n sentinel -f manifests/
```

Inspect rollout and pod health:

```bash
kubectl -n sentinel get deploy,svc,pods,pvc
kubectl -n sentinel rollout status deploy/open-webui
kubectl -n sentinel logs deploy/open-webui --tail=200
kubectl -n sentinel logs deploy/litellm --tail=200
kubectl -n sentinel logs deploy/ollama --tail=200
kubectl -n sentinel logs deploy/paperless --tail=200
```

Run the bridge CronJob immediately (one-off sync run):

```bash
kubectl -n sentinel create job --from=cronjob/sentinel-bridge-job manual-$(date +%s)
```

Access the UIs (NodePort services):

```bash
# Open WebUI (default NodePort)
# http://<node-ip>:30080
# Paperless-ngx web (default NodePort)
# http://<node-ip>:30081
# Alternatively, port-forward locally:
kubectl -n sentinel port-forward svc/open-webui-service 8080:80
kubectl -n sentinel port-forward svc/paperless-web 8081:80
```

Ollama model management (ensure models used by config are present):

```bash
# Exec into ollama and pull models referenced by litellm/open-webui
kubectl -n sentinel exec -it deploy/ollama -- bash -lc "ollama pull llama3.1 && ollama pull nomic-embed-text"
```

Storage and provisioner checks (manifests expect RWX ‘nfs-client’):

```bash
kubectl get storageclass
# Expect a class named: nfs-client
```

Notes on tests and linting: this repo is deployment manifests only; there is no application build or unit test workflow here. Operational validation is done via `kubectl` commands above.

## Architecture and structure

High level

- This repo is an opinionated K3s deployment in namespace `sentinel` that wires together:
  - GPU-backed local LLM inference (Deployment `ollama`, nodeSelector `special.gpu=nvidia`, `runtimeClassName: nvidia`).
  - A routing/proxy layer (Deployment `litellm`) that exposes an OpenAI-compatible endpoint and routes calls to either:
    - Local Ollama (`ollama/llama3.1`) via ClusterIP `ollama:11434`, or
    - A cloud model (Gemini) using an API key.
  - A user UI and RAG brain (Deployment `open-webui`) configured to talk to `litellm` for chat and to `ollama` for embeddings (`nomic-embed-text`).
  - A document system (Deployments `paperless`, `paperless-db`, `paperless-redis`).
  - Persistent volumes via an RWX storage class `nfs-client` (PVCs: `ollama-pvc`, `paperless-data-pvc`, `paperless-media-pvc`, `webui-pvc`).
  - A scheduled bridge (CronJob `sentinel-bridge-job`) that fetches documents from Paperless and ingests them into Open WebUI’s Knowledge Base.

Configuration layout

- Manifests live under `manifests/` and are grouped by concern. Files often contain a `kind: List` with multiple resources; numeric prefixes (`01-…`, `02-…`) provide an intended apply order.
  - `01-configmaps-and-cronjob.yaml` creates:
    - `ConfigMap litellm-config` (model routing for LiteLLM).
    - `ConfigMap sentinel-bridge-script` (Python sync script used by the CronJob).
    - `CronJob sentinel-bridge-job` (runs on AVX-512 labeled nodes; mounts the script CM and executes it in a `python:3.11-slim` container).
  - `02-deployments.yaml` defines Deployments for `ollama` (GPU), `paperless`, `paperless-db` (Postgres with init fix for permissions), `paperless-redis`, `litellm`, and `open-webui`.
  - `03-services.yaml` exposes ClusterIP and NodePort Services (`open-webui-service:30080`, `paperless-web:30081`).
  - `04-storage.yaml` provisions PVCs bound to `nfs-client`.

Request/data flows

- Chat: user → `open-webui` → `litellm` → model route:
  - If model name maps to `ollama/*`, requests go to local `ollama` (GPU).
  - If model name maps to a cloud model (Gemini), requests go out using the configured API key.
- RAG embeddings: `open-webui` → `ollama` (embeddings model `nomic-embed-text`). Ensure the model is pulled into the `ollama` volume.
- Document ingest: `paperless` stores documents; the CronJob downloads each doc and uploads it to `open-webui` APIs, then links files into a Knowledge Base (created on demand if missing).

Cluster prerequisites and constraints

- Node labels are required for scheduling: `special.gpu=nvidia` for the GPU node (Ollama); `special.cpu=avx512` for the high-perf CPU node (Paperless and the bridge job).
- GPU runtime: Deployments assume `runtimeClassName: nvidia`; the cluster must have the NVIDIA device plugin and a working NVIDIA Container Runtime; otherwise `ollama` will not start or will run without GPU.
- Storage: An RWX storage class named `nfs-client` must exist (PVCs bind to it). Verify with `kubectl get storageclass`.

Security hygiene (repo-specific)

- Several credentials are inlined in manifests/ConfigMaps. Replace them with Kubernetes Secrets and reference via `envFrom`/`secretKeyRef`.
  - Example (create Secrets without committing values):

    ```bash
    kubectl -n sentinel create secret generic litellm-secrets \
      --from-literal=GEMINI_API_KEY={{GEMINI_API_KEY}}

    kubectl -n sentinel create secret generic webui-bridge-secrets \
      --from-literal=PAPERLESS_TOKEN={{PAPERLESS_TOKEN}} \
      --from-literal=WEBUI_KEY={{WEBUI_JWT}}
    ```

  - Update Deployments/CronJob to consume these via env/volume as appropriate.

Operational tips

- Roll restarts cleanly when rotating config/models:

  ```bash
  kubectl -n sentinel rollout restart deploy/litellm deploy/open-webui deploy/ollama
  ```

- Dry-run apply to validate manifests client-side before pushing changes:

  ```bash
  kubectl apply -n sentinel --dry-run=client -f manifests/
  ```
