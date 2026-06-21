# Migration Plan: ingress-nginx → Gateway API + ALB (Story 1.3)

How the `hello` app was moved off ingress-nginx (a classic NLB) onto the
Gateway API with an AWS ALB, with zero downtime. This records the plan as it
was actually executed.

- Account: `804501987589`
- Cluster: `ubuntu-dev-cluster` (us-east-1)
- App: `hello` in namespace `demo`, hostname `hello.local`

## Goal

Replace ingress-nginx with the Gateway API + a shared internet-facing ALB,
with no downtime for `hello`. The old path had to keep serving traffic until
the new path was proven, and rollback had to stay trivial the whole time.

## Starting state (old path)

- `ingress-nginx` controller in namespace `ingress-nginx`, fronted by a
  classic internet-facing **NLB** (`k8s-ingressn-ingressn-17ab07b8ba`).
- A classic `Ingress` (`demo/hello`, class `nginx`, host `hello.local`)
  routing to Service `hello:80`.

## Target state (new path)

- GatewayClass `aws-alb` + shared `Gateway` (`gateway-system/shared-gateway`)
  fronted by an **ALB** (`k8s-gateways-sharedga-ab802c5d58`).
- An `HTTPRoute` (`demo/hello`, host `hello.local`) routing to the same
  Service `hello:80`, plus a `TargetGroupConfiguration` (`targetType: ip`)
  so the ALB targets IPv6 pod IPs directly.

Both paths point at the **same backend Service**, so they could run side by
side without duplicating the app.

## Parallel-run approach (both load balancers live at once)

The new path was built up **alongside** the old one — nothing was deleted
during build-out:

1. Created the `gateway-system` namespace, `LoadBalancerConfiguration`
   (internet-facing, dualstack, 3 public subnets), GatewayClass, and the
   shared Gateway. The ALB was provisioned by the controller.
2. Added the `TargetGroupConfiguration` for the `hello` Service
   (`targetType: ip`) and the `HTTPRoute` attached to `shared-gateway`.
3. Left the old `Ingress` and ingress-nginx (and its NLB) **running** the
   entire time.

During this window, `hello.local` was reachable through **both** the NLB and
the ALB simultaneously.

## How both paths were verified

Both load balancers were tested with a host-header request and both returned
**HTTP 200** with the echo-server JSON:

    # Old path (NLB / ingress-nginx)
    curl -H "Host: hello.local" http://<nlb-dns>/        # HTTP 200

    # New path (ALB / shared-gateway)
    curl -H "Host: hello.local" \
      http://k8s-gateways-sharedga-ab802c5d58-20889655.us-east-1.elb.amazonaws.com/   # HTTP 200

The new path was only considered "proven" once it returned 200 independently
of the old path.

## Rollback strategy

Rollback was trivial at every step because the old path was never touched
until the new one was proven:

- If the ALB path had failed, traffic would simply have continued on the NLB
  path (still live), and the new Gateway/HTTPRoute resources could be deleted
  with no impact on `hello`.
- No DNS or client cutover was forced during build-out — both endpoints
  answered, so there was no single moment that could break the app.

## Decommission steps actually executed (this story)

Once the ALB path was confirmed working, the old path was retired. These are
the exact steps that were run, with a read-only audit first:

1. **Read-only audit** — confirmed the NLB (`k8s-ingressn-...`) and the
   `ingress-nginx` namespace belonged **only** to this story, and that
   nothing else (including Story 2.1 Grafana) used the NLB or that namespace.
   Both HTTPRoutes (`demo/hello`, `monitoring/grafana`) attach to the ALB via
   `shared-gateway`, not the NLB.
2. **Deleted the old classic Ingress** (the `demo/hello` Service and pods were
   left in place — the HTTPRoute still uses them):

       kubectl delete ingress hello -n demo

3. **Deleted ingress-nginx** (removes the controller's LoadBalancer Service,
   which triggers AWS to tear down the NLB):

       kubectl delete -f ingress-nginx/deploy.yaml

4. **Verified the NLB is gone from AWS** — the load balancer list showed only
   the ALB; an explicit by-name check for `k8s-ingressn-ingressn-17ab07b8ba`
   returned `0` (no orphan, no extra billing).
5. **Verified the app still serves 200 via the ALB** — `hello.local` returned
   HTTP 200 echo-server JSON through `k8s-gateways-sharedga-...`.
6. **Verified Story 2.1 (Grafana) untouched** — the `monitoring/grafana`
   HTTPRoute was still present and `Accepted=True` on the ALB.
7. **Confirmed ingress-nginx fully removed** — the `ingress-nginx` namespace
   no longer exists (`NotFound`).

## Result

`hello` now serves only through the ALB + `shared-gateway`. ingress-nginx and
its NLB are fully decommissioned, leaving exactly one load balancer (the ALB)
in the account. See `cutover-ledger.md` for the per-app record.
