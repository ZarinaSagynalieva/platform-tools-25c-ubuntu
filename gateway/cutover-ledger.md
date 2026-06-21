# Cutover Ledger (Story 1.3)

Per-app record of the migration from ingress-nginx (NLB) to the Gateway API
(shared ALB). One row per app cut over.

| App | Old path | New path | Status | Verified |
|------|----------|----------|--------|----------|
| `hello` | NLB + ingress-nginx (`hello.local`) | ALB + shared-gateway (`hello.local`) | Migrated, old decommissioned | HTTP 200 via ALB; NLB torn down; no orphan LBs |

**Confirmation:** ingress-nginx is fully removed (the `ingress-nginx`
namespace no longer exists) and its classic NLB
(`k8s-ingressn-ingressn-17ab07b8ba`) has been torn down from AWS. Only the
shared ALB (`k8s-gateways-sharedga-ab802c5d58`) remains in the account, so
exactly one load balancer bills. Story 2.1 (Grafana) was unaffected and still
serves through the same ALB.
