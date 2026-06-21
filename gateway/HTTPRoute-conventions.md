# HTTPRoute Conventions

Rules for adding and structuring HTTPRoutes on the `shared-gateway`. Follow
these so every app routes the same way and stays easy to review.

## 1. Where the HTTPRoute lives

The HTTPRoute lives in the **app's own namespace, next to its Service** — not
in `gateway-system`. The app team owns its routing.

For our example app `hello`:

- Namespace: `demo`
- Service: `hello` (ClusterIP, port 80) — in `demo`
- HTTPRoute: `hello` — in `demo`

## 2. parentRefs always point to the shared Gateway

Every HTTPRoute attaches to the one shared Gateway. `parentRefs` therefore
always names `shared-gateway` in `gateway-system`:

    spec:
      parentRefs:
        - name: shared-gateway
          namespace: gateway-system

This is a **cross-namespace attachment** (route in `demo`, Gateway in
`gateway-system`). It is allowed because the Gateway's listener sets
`allowedRoutes.namespaces.from: All`. No other permission object is needed for
the attachment — see the ReferenceGrant rule below, which is a separate thing.

## 3. The ReferenceGrant rule (read this carefully)

There are two different cross-namespace directions in Gateway API, and they
are governed by two different mechanisms. Mixing them up is the most common
source of confusion.

### Direction A — Route → Gateway (the `parentRef`)

Controlled by the **Gateway's `allowedRoutes`**, NOT by ReferenceGrant.
Our `shared-gateway` uses `from: All`, so any namespace may attach. This is
why `hello` in `demo` can attach to the Gateway in `gateway-system` with no
extra object.

### Direction B — Route → backend Service (the `backendRef`)

Controlled by **ReferenceGrant**. The rule:

- **Same namespace** (route and Service together): **no ReferenceGrant
  needed.** A route may always reference a backend in its own namespace.
- **Different namespace** (route's `backendRef` points to a Service in
  another namespace): **ReferenceGrant IS required**, created in the
  *backend's* namespace, explicitly allowing HTTPRoutes from the route's
  namespace to reference that Service. Without it the route is rejected with
  `RefNotPermitted` and traffic does not flow.

### Our case: no ReferenceGrant needed

For `hello`, the HTTPRoute and its `backendRef` Service are **both in
`demo`**:

    # HTTPRoute (namespace: demo)
    spec:
      parentRefs:
        - name: shared-gateway          # cross-namespace attach -> allowed by allowedRoutes: All
          namespace: gateway-system
      rules:
        - backendRefs:
            - name: hello                # SAME namespace (demo) -> no ReferenceGrant
              port: 80

So `hello` needs **no ReferenceGrant**. The only cross-namespace relationship
is the `parentRef` to the Gateway, which `allowedRoutes: All` already permits.

### When you WOULD need one

If you ever put the HTTPRoute in one namespace and its backend Service in
another (for example, a route in `team-a` pointing at a Service in
`shared-svc`), you would add a ReferenceGrant in the **backend's** namespace
(`shared-svc`), granting routes from the route's namespace (`team-a`)
permission to reference the Service:

    apiVersion: gateway.networking.k8s.io/v1beta1
    kind: ReferenceGrant
    metadata:
      name: allow-team-a-routes
      namespace: shared-svc            # the BACKEND's namespace grants the reference
    spec:
      from:
        - group: gateway.networking.k8s.io
          kind: HTTPRoute
          namespace: team-a            # where the route lives
      to:
        - group: ""
          kind: Service
          name: <service-name>         # omit name to allow all Services in this ns

Keeping the route and Service in the same namespace (our convention) avoids
this entirely.

## 4. hostnames convention

- List the public hostname(s) the route serves under `spec.hostnames`.
- One app, one hostname where possible (`hello.local` for `hello`,
  `grafana.zarina.click` for Grafana).
- Use a real DNS name for anything reachable from outside; `*.local` style
  names are only for host-header testing through the ALB.

    spec:
      hostnames:
        - hello.local

## 5. path rules

- Use explicit `matches` with a `path` type.
- `PathPrefix` `/` means "all paths for this host" — the common default for a
  single-service app.
- Order and specificity follow the Gateway API spec: more specific path
  matches win.

    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /
        backendRefs:
          - name: hello
            port: 80

## 6. IPv6 note: backends need TargetGroupConfiguration (targetType: ip)

An HTTPRoute alone is not enough on this IPv6-only cluster. Each backend
Service also needs a `TargetGroupConfiguration` with `targetType: ip` so the
ALB targets pod IPs directly. See `README.md` for the full explanation and
manifest.
