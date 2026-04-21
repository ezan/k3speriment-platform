# k3speriment-platform

Platform layer for the k3speriment homelab — Helm charts and cluster tooling managed by Argo CD via the app-of-apps pattern.

Infrastructure (Azure VMs + k3s + Argo CD) is provisioned by [k3speriment-infra](https://github.com/ezan/k3speriment-infra). Once `argocd.repo_url` in that repo points here, Argo CD takes over and syncs everything below.

## Repository layout

```
k3speriment-platform/
├── bootstrap/              # Argo CD root Application points here (app-of-apps root)
│   ├── project-platform.yaml   # AppProject scoping all platform apps
│   ├── infrastructure.yaml     # Application → apps/infrastructure/
│   └── observability.yaml      # Application → apps/observability/
├── apps/
│   ├── infrastructure/     # Child Applications: cert-manager, ESO, reloader, ...
│   └── observability/      # Child Applications: kube-prometheus-stack, loki, promtail
└── charts/                 # Any local Helm charts authored here
```

## What's installed

**Infrastructure**
- **external-secrets** — Syncs secrets from Azure Key Vault into Kubernetes
- **reloader** — Restarts Deployments when their Secrets/ConfigMaps change
- **argocd-server Ingress** — exposes the Argo CD UI at `https://argocd.k3s.local` via Traefik

**Observability**
- **kube-prometheus-stack** — Prometheus, Grafana, exporters, default dashboards and alert rules (Alertmanager disabled — rules evaluate but don't deliver)
- **loki** — Log aggregation (single-binary mode)
- **promtail** — Log collector shipping to Loki
- Grafana is pre-wired with the Loki datasource and exposed at `https://grafana.k3s.local`
- Custom PrometheusRule `homelab-health` adds alerts for `DiskPressure`, `MemoryPressure`, and data-disk < 15% free

Storage for stateful workloads uses the default `local-path` StorageClass, which `k3speriment-infra` configures to write to `/mnt/data/k3s-local-path` on the 128 GB data disk.

## Accessing the UIs

TLS is handled by Traefik's default self-signed certificate (browser will warn — accept and proceed).

Hostnames `argocd.k3s.local` and `grafana.k3s.local` don't resolve publicly. Add an `/etc/hosts` entry on your workstation pointing both names at the Traefik LoadBalancer IP (or the k3s node public IP if Traefik uses host networking):

```
<traefik-ip>  argocd.k3s.local grafana.k3s.local
```

Find the Traefik LB IP with:
```
kubectl -n kube-system get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Grafana admin password is `changeme` in values for now — rotate once ESO + `ClusterSecretStore` are wired to Key Vault.

## Bootstrapping

1. In `k3speriment-infra`, set `argocd.repo_url = "https://github.com/ezan/k3speriment-platform.git"` in `terraform/environments/alpha.tfvars`.
2. Run the `Deploy Infrastructure` workflow.
3. Argo CD's root Application points at `bootstrap/`, which brings up the AppProject and the two app-of-apps. Those in turn sync everything under `apps/`.
4. Grafana initial credentials are created by the kube-prometheus-stack chart; retrieve with:
   ```
   kubectl -n monitoring get secret kube-prometheus-stack-grafana -o jsonpath='{.data.admin-password}' | base64 -d
   ```

## TODO (deferred from MVP)

- `ClusterSecretStore` for ESO pointing at the environment's Azure Key Vault (blocked on picking an auth mode: workload identity vs. VM managed identity via IMDS).
- Alertmanager + delivery (Slack/email/webhook) — currently rules evaluate but fire nowhere.
- Promote Grafana admin password out of the repo into Key Vault via ESO once ClusterSecretStore lands.
