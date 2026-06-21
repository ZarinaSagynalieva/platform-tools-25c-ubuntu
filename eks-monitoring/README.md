# EKS Monitoring with Prometheus and Grafana stack

## Overview and Architecture
Monitoring is provided by the community
[`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/README.md)
Helm chart, which bundles:

- **Prometheus** + **Prometheus Operator** — scrapes and stores cluster/app metrics.
- **Grafana** — dashboards for the collected metrics (ships with default Kubernetes dashboards).
- **Alertmanager** — alert routing.
- **node-exporter** and **kube-state-metrics** — node-level and Kubernetes object metrics.

Everything is deployed into the `monitoring` namespace. Prometheus, Grafana, and
Alertmanager each get a `gp2` (EBS) persistent volume so data survives pod restarts.

The chart configuration lives in [`values.yaml`](./values.yaml). Notably, the EKS
managed control-plane scrape targets (`kube-controller-manager`, `kube-scheduler`,
`etcd`) are disabled because they are not reachable on a managed control plane.

## How to Run/Execute
Deployment is handled by the `deploy-monitoring` job in
[`.github/workflows/deploy-platform-tools.yaml`](../.github/workflows/deploy-platform-tools.yaml),
which runs automatically on pushes to `feature/**` and `main`.

The job assumes the AWS role into the `ubuntu-dev-cluster` EKS cluster, adds the
`prometheus-community` Helm repo, and runs `helm upgrade --install kube-prometheus-stack`
against `values.yaml`. It runs independently of the other platform-tool steps, so a
failure elsewhere in the workflow does not block the monitoring deploy.

### Viewing the metrics

**Grafana is exposed through the shared ALB Gateway** (Gateway API) at host
`grafana.local`. Two manifests in this folder wire it up (applied by the workflow):

- [`targetgroupconfig.yaml`](./targetgroupconfig.yaml) — forces `targetType: ip`
  (required: ClusterIP Service on an IPv6-only cluster).
- [`httproute.yaml`](./httproute.yaml) — attaches `grafana.local` to `shared-gateway`
  in `gateway-system` and routes `/` to `kube-prometheus-stack-grafana:80`.

Because `grafana.local` is not a real DNS name, send the Host header to the ALB
(get the ALB DNS from the Gateway), or add it to `/etc/hosts`:

```bash
ALB=$(kubectl -n gateway-system get gateway shared-gateway \
  -o jsonpath='{.status.addresses[0].value}')

# Reach Grafana through the ALB (302 -> /login on success)
curl -I -H "Host: grafana.local" "http://$ALB/"

# Or map it for browser access, then open http://grafana.local
echo "$(dig +short $ALB | head -1) grafana.local" | sudo tee -a /etc/hosts

# Grafana login user: admin
kubectl -n monitoring get secret kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

Prometheus stays internal (`ClusterIP`); reach it with a port-forward:

```bash
# Prometheus -> http://localhost:9090
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```

## Issues
- On EKS the control plane is AWS-managed, so `kube-controller-manager`, `kube-scheduler`,
  and `etcd` have no scrapable endpoints. They are disabled in `values.yaml` to avoid
  permanent `TargetDown` alerts.

## Resources
- kube-prometheus-stack chart: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
- Helm: https://helm.sh/docs/

## Additional Information
- To expose Grafana publicly later, switch its service to a `LoadBalancer` or add an
  ingress-nginx `Ingress` (ingress-nginx is already deployed in this repo).
