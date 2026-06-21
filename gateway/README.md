# Shared Gateway (Gateway API + AWS ALB)

This directory holds the cluster's shared ingress layer, built on the
Kubernetes Gateway API and backed by a single internet-facing AWS
Application Load Balancer (ALB). It replaces the previous ingress-nginx
setup (a classic NLB), which was decommissioned in Story 1.3.

## What is deployed

| Resource | Kind | Namespace | Purpose |
|----------|------|-----------|---------|
| `aws-alb` | GatewayClass | cluster-scoped | Selects the AWS ALB controller (`gateway.k8s.aws/alb`). |
| `shared-gateway` | Gateway | `gateway-system` | One ALB, shared by all app teams. HTTP :80 listener. |
| `internet-facing-ipv6` | LoadBalancerConfiguration | `gateway-system` | ALB shape: scheme, dualstack, subnets. |
| per-app | TargetGroupConfiguration | app namespace | Forces `targetType: ip` for the app's Service. |
| per-app | HTTPRoute | app namespace | Routes a hostname/path to the app's Service. |

The ALB is provisioned automatically by the controller when the Gateway is
created. Its public DNS name appears in the Gateway status:

    kubectl get gateway shared-gateway -n gateway-system

## The shared Gateway

`shared-gateway` uses GatewayClass `aws-alb` and exposes a single HTTP
listener on port 80. Its listener allows routes from **all** namespaces, so
any app team can attach an HTTPRoute without changing the Gateway:

    listeners:
      - name: http
        protocol: HTTP
        port: 80
        allowedRoutes:
          namespaces:
            from: All

One ALB serves every route. Adding an app does not create a new load
balancer, so only one ALB bills for the whole cluster.

## ALB shape: LoadBalancerConfiguration

The Gateway points at a `LoadBalancerConfiguration` named
`internet-facing-ipv6` (via `spec.infrastructure.parametersRef`). It defines
how the ALB is built:

    spec:
      scheme: internet-facing
      ipAddressType: dualstack        # required for IPv6
      loadBalancerSubnets:
        - identifier: subnet-078433bd3c62a28ba
        - identifier: subnet-04e2241811b9e51e6
        - identifier: subnet-08bbf4f7e6dfaf907

- **scheme: internet-facing** — public ALB.
- **ipAddressType: dualstack** — REQUIRED. This cluster runs IPv6-only pods.
  Dualstack lets the ALB accept IPv4/IPv6 clients on the front and reach
  IPv6 pods on the back.
- **loadBalancerSubnets** — the three public subnets the ALB lives in (one
  per Availability Zone).

This resource is namespace-local to the Gateway, so it lives in
`gateway-system`.

## IPv6 requirement: TargetGroupConfiguration (targetType: ip)

Because the cluster is IPv6-only and apps are `ClusterIP` Services, the ALB
must send traffic directly to **pod IPs**, not node ports. The ALB Gateway
controller defaults to `instance` targets (which need a NodePort/LoadBalancer
Service), so each backend Service needs a `TargetGroupConfiguration` that
sets `targetType: ip`:

    apiVersion: gateway.k8s.aws/v1beta1
    kind: TargetGroupConfiguration
    metadata:
      name: <service-name>
      namespace: <app-namespace>
    spec:
      targetReference:
        group: ""
        kind: Service
        name: <service-name>
      defaultConfiguration:
        targetType: ip

This is the Gateway-API equivalent of the old NLB annotation
`service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip`. Without
it, the controller fails to reconcile the target group.

## How an app team adds a route

1. Deploy your app and a `ClusterIP` Service in your namespace.
2. Add a `TargetGroupConfiguration` for that Service with
   `targetType: ip` (see above).
3. Add an `HTTPRoute` in your namespace that:
   - sets `parentRefs` to `shared-gateway` in `gateway-system`, and
   - lists your hostname and path rules, with a `backendRef` to your Service.
4. Apply, then confirm it attached:

       kubectl get httproute -n <app-namespace>
       kubectl get gateway shared-gateway -n gateway-system   # attachedRoutes count rises

No Gateway or ALB changes are needed — the shared Gateway accepts routes
from every namespace. See `HTTPRoute-conventions.md` for the route rules,
and `migration-plan.md` for how the move off ingress-nginx was done.
