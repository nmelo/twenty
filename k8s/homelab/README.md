# Twenty on k3s homelab

Deployment of [Twenty CRM](https://twenty.com) to the personal k3s cluster,
exposed at `https://twenty.ker4s.com`.

## What's here

- `values.yaml` — Helm values override for the community Helm chart at
  `packages/twenty-docker/helm/twenty`.

## Install

```bash
export KUBECONFIG=~/.kube/k3s.yaml

helm upgrade --install twenty packages/twenty-docker/helm/twenty \
  --namespace twenty --create-namespace \
  -f k8s/homelab/values.yaml \
  --wait --timeout 10m
```

## Uninstall

```bash
helm uninstall twenty -n twenty
kubectl delete pvc -n twenty -l app.kubernetes.io/instance=twenty
kubectl delete ns twenty
```

## DNS

Add a Cloudflare A record:

- Name: `twenty`
- Target: a cluster node IP reachable from wherever you query
  (`192.168.1.100` for LAN, a Tailscale IP for Tailnet, or tunnel target
  for public).
- Cert issuance uses DNS-01 via `letsencrypt-cloudflare`, so the cert
  will issue regardless of whether the host is publicly reachable.

## Notes

- The chart is community-maintained, not core-team — expect rough edges.
- Bundled Postgres (spilo 3.3-p2) and Redis, deliberately isolated from
  the cluster-wide `postgres` namespace so Twenty can't impact other
  workloads.
- Storage class defaults to k3s `local-path`. Because PVCs use
  `WaitForFirstConsumer`, the db pod pins to whichever node schedules
  it first — back up that node's `/var/lib/rancher/k3s` if you care
  about the data.
