# Homelab deployment — Twenty on k3s

Deployment guide for running this fork on Nelson's homelab k3s cluster at
`https://twenty.ker4s.com`. The upstream repo's CLAUDE.md covers the app
itself; this file is specific to the homelab target.

## Cluster facts

- **kubeconfig**: `~/.kube/k3s.yaml`
- **Nodes**: workbench (amd64, control-plane, 192.168.1.100), spark1/spark2 (arm64 workers)
- **All Twenty pods are pinned to workbench** via `nodeSelector` — the image is
  amd64-only and workbench is the only amd64 node
- **Storage class**: `local-path` only — PVCs pin pods to the first-bound node
- **Ingress class**: `traefik` (also `tailscale` available)
- **cert-manager**: installed; `letsencrypt-cloudflare` ClusterIssuer uses DNS-01
  so certs issue even when the host isn't publicly reachable
- **Image registry**: `registry.ker4s.com:30095` — pinned to workbench, valid
  Let's Encrypt cert, no auth. Pod has `/data/registry` hostPath on workbench
- **DNS pattern**: `*.ker4s.com` resolves to workbench's Tailscale IP
  `100.99.121.42` (not the LAN IP). Zone in Cloudflare; records managed via API

## Files

- `k8s/homelab/values.yaml` — Helm values override (image, ingress, resources,
  nodeSelector, bundled PG/Redis)
- `k8s/homelab/README.md` — shorter install/uninstall reference
- `packages/twenty-docker/helm/twenty/` — the upstream community chart.
  **Disclaimer at the top: not maintained by the Twenty core team.** We've
  patched the 4 deployment templates to honor `.nodeSelector` values
- `packages/twenty-docker/twenty/Dockerfile` — the multi-stage Dockerfile.
  Has our `ENV NX_DAEMON=false` at the top of `common-deps`
- `seed-project-object.mjs` — reference script for creating custom objects via
  the metadata GraphQL API

## Pushing changes — pick the workflow by change type

### (A) Data-only changes (custom objects, fields, records, settings)

No rebuild. Run the seed script against the running instance:

```bash
TWENTY_BASE=https://twenty.ker4s.com \
TWENTY_EMAIL='you@example.com' \
TWENTY_PASSWORD='...' \
node seed-project-object.mjs
```

Same script works against `http://localhost:3000` (local docker-compose) — only
the base URL changes. Signup is disabled on an initialized workspace; use
`signIn` credentials you registered through the UI.

### (B) Code/source changes (backend, frontend, chart templates, values)

Build happens on **workbench**, not the Mac — Mac is arm64 and qemu cross-build
takes 30+ min; workbench is 32-core native amd64, ~5-10 min.

```bash
# 1. Edit on Mac, commit locally
cd ~/Desktop/projects/twenty
git commit -am "your change"
git push origin homelab-deploy

# 2. Sync source to workbench
# NOTE: do NOT --exclude 'build' — that's a real source subdir in twenty-sdk
rsync -az --delete \
  --exclude node_modules --exclude .git --exclude dist --exclude '.yarn/cache' \
  ~/Desktop/projects/twenty/ nmelo@192.168.1.100:~/twenty-build/

# 3. Build + push on workbench
# --target twenty is REQUIRED — default is twenty-app-dev (all-in-one dev image)
SHA=$(git rev-parse --short HEAD)
ssh nmelo@192.168.1.100 "cd ~/twenty-build && docker build --target twenty \
  -f packages/twenty-docker/twenty/Dockerfile \
  -t registry.ker4s.com:30095/twenty:homelab-v1 \
  -t registry.ker4s.com:30095/twenty:$SHA . && \
  docker push registry.ker4s.com:30095/twenty:homelab-v1 && \
  docker push registry.ker4s.com:30095/twenty:$SHA"

# 4. Roll pods (same tag, new digest — pullPolicy:Always picks it up)
export KUBECONFIG=~/.kube/k3s.yaml
kubectl -n twenty rollout restart deploy/twenty-twenty-server deploy/twenty-twenty-worker
kubectl -n twenty rollout status deploy/twenty-twenty-server --timeout=5m
```

For chart/values changes only (no image rebuild):

```bash
cd ~/Desktop/projects/twenty
export KUBECONFIG=~/.kube/k3s.yaml
helm upgrade twenty packages/twenty-docker/helm/twenty \
  -n twenty -f k8s/homelab/values.yaml
```

## First-time deploy

Prereqs: `helm`, SSH to `nmelo@192.168.1.100`, kubeconfig pointed at homelab.

```bash
# 1. Sync source + build native on workbench
cd ~/Desktop/projects/twenty
rsync -az --delete \
  --exclude node_modules --exclude .git --exclude dist --exclude '.yarn/cache' \
  ~/Desktop/projects/twenty/ nmelo@192.168.1.100:~/twenty-build/
SHA=$(git rev-parse --short HEAD)
ssh nmelo@192.168.1.100 "cd ~/twenty-build && docker build --target twenty \
  -f packages/twenty-docker/twenty/Dockerfile \
  -t registry.ker4s.com:30095/twenty:homelab-v1 \
  -t registry.ker4s.com:30095/twenty:$SHA . && \
  docker push registry.ker4s.com:30095/twenty:homelab-v1 && \
  docker push registry.ker4s.com:30095/twenty:$SHA"

# 2. Mirror dep images (spilo + redis) amd64-only to local registry, once
for img in twentycrm/twenty-postgres-spilo:v0.43.5 redis/redis-stack-server:7.2.0-v10; do
  name="${img%:*}"; tag="${img#*:}"
  docker buildx imagetools create \
    --tag "registry.ker4s.com:30095/${name##*/}:${tag}-amd64" \
    --platform linux/amd64 "$img"
done

# 3. Helm install
export KUBECONFIG=~/.kube/k3s.yaml
helm upgrade --install twenty packages/twenty-docker/helm/twenty \
  --namespace twenty --create-namespace \
  -f k8s/homelab/values.yaml \
  --wait --timeout 10m

# 4. The chart's init container has a SQL bug that silently fails to create
#    the app user. If the server pod CrashLoops with auth failure, bootstrap
#    manually (see "Things learned" below), then:
kubectl -n twenty exec deploy/twenty-twenty-server -c server -- \
  sh -c 'cd /app/packages/twenty-server && yarn database:init:prod'
kubectl -n twenty rollout restart deploy/twenty-twenty-server

# 5. Create Cloudflare DNS record
CF_TOKEN=$(kubectl -n cert-manager get secret cloudflare-api-token-secret -o jsonpath='{.data.api-token}' | base64 -d)
ZONE_ID=$(curl -sS "https://api.cloudflare.com/client/v4/zones?name=ker4s.com" \
  -H "Authorization: Bearer $CF_TOKEN" | jq -r '.result[0].id')
curl -sS -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
  -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
  --data '{"type":"A","name":"twenty","content":"100.99.121.42","ttl":300,"proxied":false}'

# 6. Verify
kubectl -n twenty get pods,ingress
curl -sSI https://twenty.ker4s.com/healthz   # expect HTTP 200
```

## Uninstall / wipe

```bash
export KUBECONFIG=~/.kube/k3s.yaml
helm uninstall twenty -n twenty
kubectl -n twenty delete pvc --all      # frees local-path volumes on workbench
kubectl delete ns twenty
```

## Things learned the hard way

- **GraphQL endpoint topology.** `/graphql` is the dynamically-generated
  per-workspace *data* schema. **`/metadata` hosts both auth AND data-model
  admin** — `signIn`, `signUp`, `activateWorkspace`, `createOneObject`,
  `createOneField` all live there. Source of truth is the `@MetadataResolver()`
  class decorator at
  `packages/twenty-server/src/engine/api/graphql/graphql-config/decorators/metadata-resolver.decorator.ts`.
  DeepWiki and upstream README both get this wrong.

- **Signup is disabled after the first workspace is created** when
  `IS_MULTIWORKSPACE_ENABLED` isn't set. To seed a second user via API, use
  `signIn`, not `signUp`.

- **Postgres schemas ≠ GraphQL endpoints.** The metadata GraphQL endpoint is
  `/metadata`, but the corresponding Postgres schema is `core`. Workspace row
  data lives in `workspace_<hash>` schemas.

- **Dockerfile default target is `twenty-app-dev`, not `twenty`.** Building
  without `--target twenty` produces the all-in-one dev image (bundles
  postgres+redis, uses s6-overlay, `ENTRYPOINT ["/init"]`). That image has a
  different layout and no `/app/entrypoint.sh` — the chart's `command:
  ["yarn", "worker:prod"]` override then runs `yarn` from `/` and crashes with
  "Couldn't find a package.json file".

- **Don't manually CREATE the `core` schema before first boot.** The image's
  `/app/entrypoint.sh` runs `setup_and_migrate_db` which checks
  `SELECT EXISTS (... WHERE schema_name = 'core')`. If the schema exists it
  *skips* migrations, leaving the DB empty. Either let the init container
  do its thing, or run `yarn database:init:prod` inside the server pod after
  manual bootstrap.

- **The chart's `ensure-database-exists` init container has a SQL bug.** It
  uses psql `:'var'` substitution inside a `DO $$ ... $$` block — substitution
  doesn't apply inside dollar-quoted function bodies, so `twenty_app_user`
  never gets created and the server CrashLoops with auth failure. Workaround
  is the manual bootstrap SQL in "First-time deploy" step 4. TODO: patch the
  chart template.

- **`acme: true` in the chart hardcodes `letsencrypt-prod`.** We set
  `acme: false` and put `cert-manager.io/cluster-issuer: letsencrypt-cloudflare`
  in `annotations:` instead (DNS-01).

- **The chart has no scheduling hooks by default** — we patched 4 deployment
  templates (`deployment-server.yaml`, `deployment-worker.yaml`,
  `deployment-db-internal.yaml`, `deployment-redis-internal.yaml`) to honor
  `.{server,worker,db.internal,redisInternal}.nodeSelector` from values.

- **Docker Desktop default memory (~8 GB) is not enough.** The twenty-front
  build needs an 8 GB Node heap. Bump Docker Desktop → Resources → Memory to
  ≥16 GB. Moot for workbench builds (has 60 GB).

- **Cross-build from arm64 Mac to amd64 via qemu is unusably slow** (30+ min
  on this repo). Build on workbench instead (32-core native).

- **rsync exclusions bite — `--exclude build` was wrong.** `twenty-sdk` has a
  real source dir at `src/front-component-renderer/build/`. Excluding "build"
  broke the `twenty-sdk:build` nx target with a confusing "Could not resolve
  entry module" error cascaded into `lingui:extract`.

- **`NX_DAEMON=false` in the Dockerfile.** Without it, the nx daemon orphans
  in non-TTY SSH/CI contexts and exits with SIGINT (130) mid-build.
  Do NOT also set `CI=true` — that flips Yarn to `--immutable` and the
  upstream lockfile has drift that immutable mode rejects.

- **pullPolicy: Always is mandatory.** Same-tag rebuilds (moving tag like
  `:homelab-v1`) won't get picked up otherwise — kubelet treats the tag name
  as the cache key, not the digest. Setting `image.pullPolicy: Always` in
  values forces a fresh manifest fetch on every pod start.

- **DNS pattern.** All `*.ker4s.com` services point at workbench's Tailscale
  IP `100.99.121.42`, not the LAN IP. Cloudflare serves RFC1918 too
  (registry/unifi/truenas use LAN) but Tailnet is the convention.

- **macOS `mDNSResponder` negative-caches NXDOMAIN.** If you check DNS before
  a record propagates and get NXDOMAIN, later queries may keep returning
  NXDOMAIN even after the record is live. Flush with
  `osascript -e 'do shell script "killall -HUP mDNSResponder" with administrator privileges'`.

- **Cloudflare API token for DNS-01 is in-cluster.** No need to paste one —
  `kubectl -n cert-manager get secret cloudflare-api-token-secret` has a
  token scoped to `Zone:DNS:Edit` for `ker4s.com`, usable for record CRUD.

- **The registry is pinned to workbench via nodeSelector.** If workbench hits
  DiskPressure (default eviction: `nodefs.available<5%`) the registry pod
  gets evicted and every Twenty pod then ImagePullBackOffs because they pull
  from `registry.ker4s.com:30095`. Clear space on workbench with
  `docker system prune -af --filter 'until=24h'` (docker) — kubelet takes a
  few minutes to re-evaluate and clear the taint.

- **kubelet has `registry-pull-qps=5` rate limit by default.** When many pods
  start at once you'll see transient "pull QPS exceeded" — self-resolves
  within a minute or two.
