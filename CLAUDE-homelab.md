# Homelab deployment — Twenty on k3s

Deployment guide for running this fork on Nelson's homelab k3s cluster at
`https://twenty.ker4s.com`. The upstream repo's CLAUDE.md covers the app
itself; this file is specific to the homelab target.

## Cluster facts

- **kubeconfig**: `~/.kube/k3s.yaml`
- **Nodes**: workbench (control-plane, 192.168.1.100), spark1/spark2 workers
- **Storage class**: `local-path` only — PVCs pin pods to the first-bound node
- **Ingress class**: `traefik` (also `tailscale` available)
- **cert-manager**: installed; `letsencrypt-cloudflare` ClusterIssuer uses DNS-01
  so certs issue even when the host isn't publicly reachable
- **Image registry**: `registry.ker4s.com:30095` — NodePort to a pod pinned on
  workbench, valid Let's Encrypt cert, no auth

## Files

- `k8s/homelab/values.yaml` — Helm values override (image, ingress, resources,
  bundled PG/Redis)
- `k8s/homelab/README.md` — shorter install/uninstall reference
- `packages/twenty-docker/helm/twenty/` — the upstream community chart we apply
  the overrides to. **Disclaimer at the top: not maintained by the Twenty core
  team.**
- `packages/twenty-docker/twenty/Dockerfile` — the multi-stage Dockerfile we
  build from (server + front + emails + shared + ui in one image)

## First-time deploy

Prereqs: `helm`, `docker`, kubeconfig pointed at homelab, Docker Desktop VM
sized to **16 GB RAM minimum** (front build needs an 8 GB Node heap).

```bash
# 1. Build from the fork
cd ~/Desktop/projects/twenty
docker build \
  -f packages/twenty-docker/twenty/Dockerfile \
  -t registry.ker4s.com:30095/twenty:homelab-v1 \
  -t registry.ker4s.com:30095/twenty:$(git rev-parse --short HEAD) \
  .

# 2. Push
docker push registry.ker4s.com:30095/twenty:homelab-v1
docker push registry.ker4s.com:30095/twenty:$(git rev-parse --short HEAD)

# 3. Install/upgrade
export KUBECONFIG=~/.kube/k3s.yaml
helm upgrade --install twenty packages/twenty-docker/helm/twenty \
  --namespace twenty --create-namespace \
  -f k8s/homelab/values.yaml \
  --wait --timeout 10m

# 4. Verify
kubectl -n twenty get pods,ingress
kubectl -n twenty logs -l app.kubernetes.io/component=server -f
```

Add Cloudflare DNS record `twenty.ker4s.com → 192.168.1.100` (or a Tailscale
IP / tunnel target for remote access).

## Push code changes and redeploy

```bash
# After editing source — bump the tag so pods actually pull a new image
SHA=$(git rev-parse --short HEAD)
docker build \
  -f packages/twenty-docker/twenty/Dockerfile \
  -t registry.ker4s.com:30095/twenty:$SHA \
  -t registry.ker4s.com:30095/twenty:homelab-v1 \
  .
docker push registry.ker4s.com:30095/twenty:$SHA
docker push registry.ker4s.com:30095/twenty:homelab-v1

# Option A: re-run helm upgrade (idempotent)
helm upgrade twenty packages/twenty-docker/helm/twenty \
  -n twenty -f k8s/homelab/values.yaml \
  --set image.tag=$SHA

# Option B: just restart pods if the tag didn't change
kubectl -n twenty rollout restart deploy/twenty-twenty-server deploy/twenty-twenty-worker
```

**Always bump `image.tag` when shipping a change.** `:homelab-v1` is a moving
tag; k3s won't pull a new digest unless the reference changes. The commit-SHA
tag is the durable identity — use it in `--set image.tag=...`.

## Uninstall / wipe

```bash
helm uninstall twenty -n twenty
kubectl -n twenty delete pvc --all      # frees local-path volumes
kubectl delete ns twenty
```

## Things learned the hard way

- **GraphQL endpoint topology is counter-intuitive.** `/graphql` is the
  dynamically-generated per-workspace *data* schema (People, Companies, custom
  objects). **`/metadata` hosts both auth AND the data-model admin** —
  `signIn`, `signUp`, `activateWorkspace`, `createOneObject`, `createOneField`
  all live there. The deciding signal in source is the `@MetadataResolver()`
  class decorator (see
  `packages/twenty-server/src/engine/api/graphql/graphql-config/decorators/metadata-resolver.decorator.ts`).
  DeepWiki and the README both get this wrong; trust the decorator.

- **GraphQL introspection is disabled in the prebuilt image.** `{__schema{...}}`
  will return errors. Use the source for schema shapes (resolvers + DTOs under
  `packages/twenty-server/src/engine/core-modules/` and
  `packages/twenty-server/src/engine/metadata-modules/`).

- **Signup is disabled after the first workspace is created** when
  `IS_MULTIWORKSPACE_ENABLED` isn't set. The check is `multiworkspace OR
  workspaceCount==0` (see `sign-in-up.service.ts::isSignUpEnabled`). To seed a
  second user programmatically, use `signIn`, not `signUp`.

- **Postgres schemas ≠ GraphQL endpoints.** The metadata GraphQL endpoint is
  `/metadata`, but the corresponding Postgres schema is **`core`**, not
  `metadata`. Workspace row data lives in `workspace_<hash>` schemas.

- **Custom objects auto-get relation fields** to Note, Task, Attachment, and
  TimelineActivity — that's how the UI wires in the per-entity sidebar
  without per-object configuration.

- **`acme: true` in the Helm chart hardcodes `cert-manager.io/cluster-issuer:
  letsencrypt-prod`.** We set `acme: false` and put the annotation in
  `annotations:` instead so we can point at `letsencrypt-cloudflare` (DNS-01).

- **Docker Desktop default memory (~8 GB) is not enough** to build the
  twenty-front stage. Node heap is configured to `--max-old-space-size=8192`,
  so the VM needs ≥16 GB or the build OOMs partway. Settings → Resources →
  Memory → 16 GB minimum.

- **The k3s `registries.yaml` mirror (`localhost:5000 → http://localhost:5000`)
  is a red herring for our deploys.** We push to `registry.ker4s.com:30095`
  and reference the image by that same URL in values.yaml. k3s nodes pull
  directly over HTTPS; the LE cert is trusted by default. Only use the
  localhost:5000 mirror if you specifically need to optimize for
  workbench-local pulls.

- **The community chart isn't core-team maintained.** Expect template quirks
  like the duplicate cluster-issuer bug above. File bugs upstream or patch
  templates locally — don't assume the chart is authoritative.

## Seeding custom objects programmatically

See `seed-project-object.mjs` at the repo root for a working end-to-end
signIn → getAuthTokensFromLoginToken → createOneObject → createOneField flow
against `/metadata`. Reuse it as a template for bulk seeding via the API
instead of the UI.

```bash
TWENTY_BASE=https://twenty.ker4s.com \
TWENTY_EMAIL='you@example.com' \
TWENTY_PASSWORD='...' \
node seed-project-object.mjs
```
