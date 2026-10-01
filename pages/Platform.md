- Where each item of the extended stack list goes: event bus, NATS, cookies, sessions, JWT, deployments, pod load balancing, NGINX ingress, minikube, Google Cloud, git repositories, CI/CD. Companion to the architecture overview and [[Services]].
- | Item | Where it lives | Wave |
  | --- | --- | --- |
  | Event bus | NATS JetStream, one stream of `likho.*` subjects | 0 |
  | NATS | the event bus itself (replaces Kafka in the plan) | 0 |
  | Cookies, session | browser ↔ likho-api login | 1 |
  | JWT | likho-api → every other service | 1 |
  | Git repositories | one per service in the GitHub organisation | 0 |
  | CI | GitHub Actions per repo, shared workflows | 0 |
  | Deployment, pod load balancer | Kubernetes Deployment + Service per service | 2 |
  | minikube | local Kubernetes | 2 |
  | Ingress, NGINX | NGINX gateway in front of everything | 0 (Compose), 2 (Kubernetes) |
  | Google Cloud | GKE cluster, the production target | 3 |
  | CD | Argo CD syncing the cluster from git | 3 |
- ## 1. Event bus: NATS JetStream (one bus, not two)
	- The plan first said Kafka; the list now also says NATS. Both do the same job, so only one runs. **Recommendation: NATS JetStream.**
	- | | NATS JetStream | Kafka / Redpanda |
	  | --- | --- | --- |
	  | Footprint | one ~20 MB binary, ~50 MB RAM | ~1 GB RAM for a dev broker |
	  | Long jobs (a 20-minute call takes ~15 minutes to transcribe) | built in: the worker acknowledges when done and reports "in progress" meanwhile; a crashed worker's job is redelivered | needs poll-interval tuning or pause/resume to avoid rebalances |
	  | Work queue + fan-out | both, per consumer | fan-out natural, work queue by partition count |
	  | Extras this project uses | request/reply, key-value store, leaf nodes (an office machine can serve a cloud cluster) | large connector ecosystem, very high throughput |
	  | Fits this machine (15.8 GB) | yes | costs a gigabyte for no benefit at this volume |
	- Kafka earns its place at millions of events a day or when its connectors are needed. Likho handles a few calls a minute. Because every event is a CloudEvent published through one small `events` module per language, moving to Kafka later changes that module, not the services.
	- How it is used:
	- Stream `LIKHO` stores `likho.>`; `LIKHO_LIVE` stores the live transcript lines for one day.
	- Each service has a **durable consumer** per subject it cares about. Several instances of a service share the consumer, so each message goes to one of them.
	- `likho.transcription.requested` is the job queue: explicit ack after the transcript is stored, "in progress" every 30 s, three deliveries, then the message moves to `likho.dead` and the job is marked failed with the reason.
	- Publishing sets `Nats-Msg-Id` to the event id, so a retry is not stored twice. Consumers are idempotent on that id.
	- Note: the old "NATS Streaming Server" (STAN) used in many tutorials is retired; JetStream is its replacement and is part of the normal NATS server.
- ## 2. Cookies, sessions and JWT
	- Two different problems, two tools.
	- **Browser → likho-api: session cookie.**
	- Login (email + password in Wave 1, Keycloak single sign-on in Wave 3) creates a session row; the browser gets `likho_session`, a random id, as a cookie that is `HttpOnly`, `Secure`, `SameSite=Lax`, path `/`.
	- The session itself (user, workspace, role, expiry) lives in Redis with a PostgreSQL record, so it can be listed and revoked ("log out everywhere").
	- Nothing is kept in `localStorage`; JavaScript cannot read the cookie, which is what protects it from script injection. `SameSite` plus an origin check on every mutation covers cross-site request forgery.
	- Lifetime: 7 days, refreshed while in use.
	- **likho-api → other services: short-lived JWT.**
	- For each incoming request likho-api mints an internal token: 5 minutes, signed with EdDSA, claims `sub` (user), `ws` (workspace), `role`, `scope`.
	- It travels as gRPC metadata (`authorization: Bearer …`). Services verify it with the public key from `likho-api/.well-known/jwks.json` — no call back, no shared secret.
	- Events carry the actor (`userid`, `workspace`) as CloudEvent attributes instead of a token, because an event outlives any token.
	- The browser never sees this JWT.
	- **Scripts and the CLI: API keys.** Created in Settings, stored hashed, sent as `Authorization: Bearer lk_…` to `/api/v1`, exchanged by likho-api for the same internal JWT.
	- **Audio playback:** likho-api asks likho-media for a presigned URL valid for a few minutes, so the audio element can seek with range requests without carrying credentials.
	- A NestJS auth module provides the session, cookie and JWKS pieces in likho-api, reusing the lockout, permission catalogue and audit-log design of an existing internal application. Moving to Keycloak later replaces only the login step; cookie and internal JWT stay as they are.
- ## 3. Kubernetes: deployments and pod load balancing
	- Each service gets the same four objects from one shared Helm chart (`likho-service`):
	- ```
	  Deployment   replicas: N, rolling update, liveness /healthz, readiness /readyz, resources
	  Service      ClusterIP — one stable name (likho-api:4000) that spreads connections over the pods
	  HPA / KEDA   scale on CPU (api, web) or on queue depth (transcription, via the NATS scaler)
	  ConfigMap + Secret   environment variables; secrets from Google Secret Manager in the cloud
	  ```
	- "Pod load balancer" in practice:
	- **Inside the cluster** the Service does it: a client connects to `likho-language:5030` and Kubernetes picks a healthy pod.
	- **gRPC needs one extra step.** gRPC keeps a single long connection, so all calls would land on one pod. The gRPC Services are therefore headless (`clusterIP: None`) and the clients use `dns:///…` with round-robin, which balances per call.
	- **From outside** one `LoadBalancer` Service in front of the gateway gets the public IP (a Google Cloud load balancer on GKE, `minikube tunnel` locally).
	- **Workers are not load-balanced at all** — transcription pods pull jobs from NATS, so adding pods is the scaling.
	- Stateful pieces (PostgreSQL, MongoDB, Redis, NATS, Meilisearch) run as StatefulSets from their official Helm charts on minikube; in the cloud the databases move to managed services where that is cheaper to operate.
- ## 4. Ingress with NGINX
	- **Compose (now):** a plain `nginx` container with one config file: `/` → web, `/graphql`, `/api`, `/events` → likho-api (`proxy_buffering off` on `/events` so live lines are not held back), `/media` → likho-media (large uploads, range requests).
	- **Kubernetes (Wave 2):** the well-known community controller `ingress-nginx` — the one in most tutorials and in minikube's `ingress` addon — was retired by the Kubernetes project: announced November 2025, maintenance ended March 2026. New clusters should not start on it. The supported NGINX route is the **Gateway API** with **NGINX Gateway Fabric**: one `Gateway` and an `HTTPRoute` per service, kept in `likho-infra/k8s/gateway/`.
	- The same `HTTPRoute` files work on GKE's own Gateway classes, so the choice between NGINX and Google's load balancer in the cloud can be made in Wave 3 without rewriting routes.
	- To check when Wave 2 starts: current NGINX Gateway Fabric version and its minikube install steps.
- ## 5. minikube and Skaffold (local Kubernetes, Wave 2)
	- ```powershell
	  winget install Kubernetes.minikube
	  winget install Helm.Helm
	  winget install GoogleContainerTools.Skaffold
	  minikube start --driver=docker --cpus=4 --memory=4g --profile likho
	  skaffold dev            # from likho-infra: builds the repos next to it, deploys, reloads on change
	  minikube tunnel -p likho   # gives the gateway an address; open http://likho.localhost
	  ```
	- `kubectl` 1.36 is already installed. minikube takes 4 GB, so on this machine it replaces the Compose stack while it runs, and the Whisper worker stays on the host, connected to the cluster's NATS through a forwarded port. Compose remains the everyday way to work on one service; minikube is for testing the charts and routes before the cloud.
- ## 6. Google Cloud (Wave 3)
	- | Need | Google Cloud service |
	  | --- | --- |
	  | Cluster | **GKE Autopilot** (Google runs the nodes; billed per pod) |
	  | Images | GHCR stays the registry; Artifact Registry as a mirror if pulls need to be faster |
	  | PostgreSQL | Cloud SQL, one instance with a database per service |
	  | Redis | Memorystore |
	  | Object storage | Cloud Storage through its S3-compatible API (no code change in the services) |
	  | MongoDB, NATS, Meilisearch | in the cluster from their Helm charts (or MongoDB Atlas on Google Cloud) |
	  | Secrets | Secret Manager, mounted into pods |
	  | Public entry | Gateway API → Google load balancer, managed TLS certificate, Cloud DNS |
	  | Deploy identity | Workload Identity Federation: GitHub Actions gets short-lived access, no stored keys |
	- Transcription in the cloud is the cost question: Whisper needs either many CPU cores or a GPU node. Two options to price before Wave 3:
	- a GPU node pool that scales to zero when the queue is empty (KEDA), or
	  logseq.order-list-type:: number
	- **hybrid**: the cluster runs everything except transcription, and the office machine (or a rented GPU box) connects as a NATS leaf node and takes the jobs.
	  logseq.order-list-type:: number
	- No cloud resources are created before a written cost estimate is agreed.
- ## 7. Git repositories and CI/CD
	- **Repositories** — one per service in the organisation (list in PLAN.md §4), plus `likho-contracts`, `likho-infra`, `likho-docs`, `.github`. `main` is protected by an organisation ruleset: pull request, green CI, no force push.
	- **CI (every repo, every push and pull request)** — one short `ci.yml` calling the shared workflow for its language in likho-infra:
	- ```
	  lint → type check → unit tests → integration tests (Testcontainers: real Postgres / NATS / Mongo)
	       → build image → scan image (Trivy) → on main: push ghcr.io/<org>/<repo>:<git sha>
	  ```
	- `likho-contracts` adds `buf lint` and `buf breaking`. release-please opens the release pull request (version + changelog) from Conventional Commits; merging it tags `vX.Y.Z` and pushes the image with that tag.
	- **CD** — git is the source of truth for what runs:
	- ```
	  service repo  ──CI──▶  image :sha  ──▶  pull request to likho-infra (values/<service>.yaml: new tag)
	  likho-infra   ──merge──▶  Argo CD in the cluster sees the change and rolls the Deployment
	  ```
	- Staging: the tag bump merges automatically when CI is green.
	- Production: the same pull request against the production values, merged by a person.
	- Rollback: revert the commit; Argo CD rolls back.
	- Until Wave 3 there is no cluster to deploy to; CD ends at "image published".
