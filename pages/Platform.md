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
	- Built in Wave 2 as its own repository, [likho-deploy](https://github.com/likho-ai/likho-deploy): two Helm charts and Skaffold. `charts/likho` renders the same objects for every service from one template, driven by values:
	- ```
	  Deployment   replicas, rolling update (Recreate where a volume is attached), liveness /healthz, readiness /readyz, resources
	  Service      one stable name - likho-api:4000, likho-media:5010 - the names the .env files already use
	  ConfigMap    <service>-env: the repository's .env.<environment>, synced into environments/<environment>/env.yaml
	  Secret       <service>-secrets: the .env.<environment>.local file, applied straight to the cluster by scripts/secrets.py
	  ```
	- `charts/likho-stack` runs the backing services (PostgreSQL, MongoDB, Redis, NATS JetStream, SeaweedFS, Meilisearch) as StatefulSets on persistent volumes, with the database users, the streams and the buckets created from one Secret on the first start. Managed services replace a part by disabling it and pointing the service's `.env.<environment>.local` at the managed address.
	- "Pod load balancer" in practice:
	- **Inside the cluster** the Service does it: a client connects to `likho-language:5030` and Kubernetes picks a healthy pod.
	- **gRPC needs one extra step.** gRPC keeps a single long connection, so all calls would land on one pod. The gRPC Services (media, transcription, language) are therefore headless (`clusterIP: None`): a client that resolves the name gets the pods' own addresses. The clients still connect with plain `host:port`; switching them to `dns:///` with round-robin is the step to take when a gRPC service first runs more than one replica.
	- **From outside** the cluster's Gateway sends the hostname to `likho-gateway` (section 4); without one, the gateway Service becomes a `LoadBalancer`.
	- **Workers are not load-balanced at all** - transcription pods pull jobs from NATS, so adding pods is the scaling. A stopping transcription pod gets fifteen minutes to finish the job in hand.
	- Not built yet: HPA / KEDA scaling on queue depth, PodDisruptionBudgets, NetworkPolicies, the observability profile.
- ## 4. Ingress with NGINX
	- **Compose (now):** a plain `nginx` container with one config file: `/` → web, `/graphql`, `/api`, `/events` → likho-api (`proxy_buffering off` on `/events` so live lines are not held back), `/media` → likho-media (large uploads, range requests), `/mfe/…` → the web apps.
	- **Kubernetes (built):** the same nginx runs inside the cluster as `likho-gateway`, with the cluster's Services as upstreams (looked up at request time, so it serves while a service is being replaced). It keeps the routes and the upload and streaming settings in one place for every environment.
	- The public hostname is an `HTTPRoute` (Gateway API) from the cluster's `Gateway` to `likho-gateway`. The community controller `ingress-nginx` was retired by the Kubernetes project (announced November 2025, maintenance ended March 2026), so no `Ingress` object is used. The Gateway itself - NGINX Gateway Fabric on our own clusters, GKE's Gateway in the cloud - and its TLS certificate belong to the cluster, not to the chart; the same route works on both.
- ## 5. minikube and Skaffold (local Kubernetes)
	- ```powershell
	  cd D:\likho\likho-deploy
	  .\scripts\local.ps1 up     # start minikube (docker driver, 4 CPUs, 5 GB), make the Secrets, install the backing
	                              # services (Helm), build the seven images inside the cluster and deploy the product
	                              # (Skaffold), forward the gateway to http://localhost:8080, rebuild what changes
	  ```
	- The local cluster runs the **staging configuration** (the staging `env.yaml` and the `.env.staging.local` secrets) with `http://localhost:8080` as its address and the development admin login, so what is rehearsed on the machine is what staging gets. Skaffold tags images by content: a rebuild of unchanged sources is a no-op. The backing services are a Helm release Skaffold never touches, so a redeploy never restarts a database or the event bus. The Compose stack remains the everyday way to work on one service; minikube is for the charts and the routes. Proven on 3 October 2026: the web app's browser test passes against the local cluster end to end (upload, transcription in the cluster, both layers, playback, delete).
	- Releases: a change of the image tag in `environments/<environment>/values.yaml`, committed, then `scripts/deploy.ps1 staging` (or `production`): both charts as Helm releases; a rollback is `helm rollback` or the same change the other way. One setting exists only for a cluster: likho-media's `INTERNAL_URL`, the address the transcription worker downloads originals from, since the public address is not reachable from inside.
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
