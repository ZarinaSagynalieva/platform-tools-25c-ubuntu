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
Deployment is handled by GitHub Actions —
[`.github/workflows/deploy-monitoring.yaml`](../.github/workflows/deploy-monitoring.yaml):

- Runs automatically on pushes that touch `eks-monitoring/**`.
- Can be triggered manually via **workflow_dispatch** (optionally pinning a chart version).

The workflow assumes the AWS role into the `ubuntu-dev-cluster` EKS cluster, adds the
`prometheus-community` Helm repo, and runs `helm upgrade --install kube-prometheus-stack`
against `values.yaml`.

### Viewing the metrics

Grafana and Prometheus are exposed as `ClusterIP` services. Port-forward to reach them:

```bash
# Grafana -> http://localhost:3000  (user: admin)
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80

# Get the auto-generated Grafana admin password
kubectl -n monitoring get secret kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

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
