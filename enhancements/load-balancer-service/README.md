---
title: OSAC Load Balancer Service Add-On
authors:
  - fdupont@redhat.com
creation-date: 2026-04-30
last-updated: 2026-04-30
tracking-link:
  - TBD
see-also:
  - enhancements/osac-addon/README.md
  - enhancements/networking/README.md
  - "MGMT-23730: PublicIP & PublicIPPool - Floating IP Management"
replaces:
superseded-by:
---

# OSAC Load Balancer Service Add-On

## Summary

Introduce Load-Balancer-as-a-Service for OSAC tenants as an
**OSAC Add-On**, following the contract defined in the
[OSAC Add-On EP](../osac-addon/README.md). The add-on is
delivered as per-provider Ansible collections following the
ResourceAction naming convention, built on the Kubernetes
Gateway API. Initial providers: `osac.lb_envoy_gateway`
(L7) and `osac.lb_metallb` (L4).

Tenants can create load balancers with L4 (TCP/UDP) and L7
(HTTP/HTTPS) capabilities, backed by pluggable implementations
from the Red Hat ecosystem.

## Motivation

### Current state

OSAC provides VirtualNetworks, Subnets, SecurityGroups, and
PublicIPs for tenant networking. Tenants can expose individual
ComputeInstances via PublicIPs. However, there is no way to
distribute traffic across multiple instances, perform health
checking, or terminate TLS at a shared entry point.

### Problems

1. **No traffic distribution.** Tenants with multiple
   ComputeInstances running the same service must implement
   their own load balancing (e.g., HAProxy VM), duplicating
   effort.

2. **No health checking.** If a backend instance fails, traffic
   continues to be sent to it. There is no OSAC-managed health
   check mechanism.

3. **No L7 routing.** Tenants cannot route traffic based on
   HTTP host, path, or headers without deploying their own
   reverse proxy.

4. **No TLS termination.** Each ComputeInstance must terminate
   TLS individually. There is no shared TLS entry point with
   certificate management.

### Comparison with VMware Cloud Director

VCD provides load balancing via NSX Edge Gateways and the
Advanced Load Balancer (ALB/Avi) integration:

| Dimension | VCD Load Balancing | OSAC Load Balancing (proposed) |
|---|---|---|
| L4 | NSX Edge LB (TCP/UDP pools) | MetalLB IP + Gateway API TCPRoute/UDPRoute |
| L7 | ALB/Avi virtual services | Gateway API HTTPRoute/TLSRoute via Envoy Gateway or Istio |
| Configuration | Edge Gateway UI/API | LoadBalancer CR via OSAC API |
| Health checks | Active health monitors per pool | Gateway API backend health checks |
| TLS | Certificate management per virtual service | cert-manager integration |
| Scope | Per Edge Gateway (per Org VDC) | Per tenant VirtualNetwork |
| Provider model | NSX + ALB only | Pluggable LoadBalancerClass |
| Rate limiting | ALB WAF policies | Kuadrant RateLimitPolicy |
| Multi-cluster DNS | Not built-in | Kuadrant DNSPolicy |

### User Stories

- As a **tenant**, I want to create a load balancer that
  distributes HTTP traffic across my ComputeInstances based on
  URL path, so that I can run a microservices architecture.

- As a **tenant**, I want my load balancer to stop sending
  traffic to unhealthy backends so that my service remains
  available during instance failures.

- As a **tenant**, I want to terminate TLS at the load balancer
  with an automatically managed certificate so that I do not
  have to configure TLS on each instance.

- As a **CSP admin**, I want to offer multiple load balancer
  tiers (L4-only via MetalLB, L7 via Envoy Gateway) so that
  tenants can choose the capability level they need.

- As a **tenant**, I want a TCP load balancer for my database
  cluster so that clients connect to a single endpoint with
  automatic failover.

### Goals

- Introduce LoadBalancerClass, LoadBalancer, LoadBalancerListener,
  and LoadBalancerTarget as OSAC resources.
- Build on Kubernetes Gateway API as the underlying mechanism.
- Build on PublicIP infrastructure (MGMT-23730) for IP
  allocation.
- Support L4 (TCP/UDP) and L7 (HTTP/HTTPS) load balancing.
- Deliver as an OSAC Add-On with Enclave plugin packaging.

### Non-Goals

- Implementing a load balancer data plane — OSAC delegates to
  Gateway API implementations (Envoy Gateway, Istio, MetalLB).
- Global server load balancing (GSLB) — multi-region traffic
  distribution is a Kuadrant DNSPolicy concern.
- Web Application Firewall (WAF) — separate concern, potential
  future add-on.
- Service mesh features (mTLS, observability) beyond load
  balancing.

## Proposal

### Architecture: Gateway API Foundation

The Kubernetes Gateway API provides a natural foundation for
OSAC load balancing. Its resource model already separates
infrastructure concerns (GatewayClass, Gateway) from routing
concerns (*Route), with built-in multi-tenancy:

```
Gateway API (K8s native)          OSAC (tenant-facing)
─────────────────────────         ────────────────────
GatewayClass (infra)         ←    LoadBalancerClass
Gateway (infra/tenant)       ←    LoadBalancer
HTTPRoute (tenant)           ←    LoadBalancerTarget (L7)
TCPRoute (tenant)            ←    LoadBalancerTarget (L4)
```

OSAC wraps Gateway API with tenant isolation, quota enforcement,
and integration with VirtualNetworks and PublicIPs.

### Ansible Collections (per provider)

Following the [OSAC Add-On convention](../osac-addon/README.md),
each LB provider ships its own collection:

#### osac.lb_envoy_gateway

```yaml
# meta/addon.yaml
name: osac-lb-envoy-gateway
display_name: Load Balancer (Envoy Gateway)
description: L7 load balancing via Envoy Gateway
version: 1.0.0

dependencies:
  osac_core: ">=0.1.0"
  addons:
    - osac-networking
  operators:
    - name: gateway-api
      optional: false

resource_types:
  - name: LoadBalancerClass
    scope: provider
  - name: LoadBalancer
    scope: tenant
  - name: LoadBalancerTarget
    scope: tenant
```

```
roles/
├── load_balancer.create.main/
│   ├── meta/osac.yaml
│   └── tasks/main.yml
├── load_balancer.delete.main/
│   ├── meta/osac.yaml
│   └── tasks/main.yml
├── target.create.main/
│   ├── meta/osac.yaml
│   └── tasks/main.yml
└── target.delete.main/
    ├── meta/osac.yaml
    └── tasks/main.yml
```

#### osac.lb_metallb

Same role structure for L4 TCP/UDP. Each collection is
self-contained — no dispatcher role.

### Personas

| Persona | Role |
|---|---|
| CSP Admin | Defines LoadBalancerClasses, installs Gateway API implementations |
| Tenant Admin | Creates load balancers, configures listeners and targets |
| Tenant User | Consumes load-balanced services |

### Resource Types

#### LoadBalancerClass

Provider-defined. Specifies the Gateway API implementation and
capabilities.

```yaml
apiVersion: osac.io/v1
kind: LoadBalancerClass
metadata:
  name: envoy-l7
  labels:
    osac.io/addon: osac-load-balancer
spec:
  provider: envoy-gateway
  gatewayClassName: envoy-gateway
  capabilities:
    - HTTP_ROUTING
    - TLS_TERMINATION
    - HEALTH_CHECKS
    - RATE_LIMITING
  defaults:
    maxListeners: 10
    maxTargetsPerListener: 50
```

```yaml
apiVersion: osac.io/v1
kind: LoadBalancerClass
metadata:
  name: metallb-l4
  labels:
    osac.io/addon: osac-load-balancer
spec:
  provider: metallb
  gatewayClassName: metallb
  capabilities:
    - TCP_PROXY
    - UDP_PROXY
  defaults:
    maxListeners: 5
```

#### LoadBalancer

Tenant-scoped. Represents a load balancer instance. Maps to a
Kubernetes Gateway resource. Allocated a PublicIP.

```yaml
apiVersion: osac.io/v1
kind: LoadBalancer
metadata:
  name: acme-web-lb
  annotations:
    osac.openshift.io/tenant: acme-corp
    osac.io/billing-labels: "service=web,tier=production"
spec:
  loadBalancerClass: envoy-l7
  virtualNetwork: acme-production
  publicIPPool: external-v4
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRef:
          name: acme-web-cert
    - name: http-redirect
      protocol: HTTP
      port: 80
      redirect:
        scheme: https
        statusCode: 301
status:
  state: READY
  publicIP: 203.0.113.50
  gatewayRef:
    name: acme-corp-acme-web-lb
    namespace: osac-acme-corp
  conditions:
    - type: Ready
      status: "True"
    - type: Programmed
      status: "True"
```

#### LoadBalancerTarget

Tenant-scoped. Defines routing rules and backend targets for a
listener. Maps to HTTPRoute, TCPRoute, or UDPRoute.

```yaml
apiVersion: osac.io/v1
kind: LoadBalancerTarget
metadata:
  name: acme-web-api
  annotations:
    osac.openshift.io/tenant: acme-corp
spec:
  loadBalancer: acme-web-lb
  listener: https
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backends:
        - computeInstance: acme-api-server-1
          port: 8080
          weight: 50
        - computeInstance: acme-api-server-2
          port: 8080
          weight: 50
      healthCheck:
        path: /healthz
        intervalSeconds: 10
        timeoutSeconds: 5
        unhealthyThreshold: 3
    - matches:
        - path:
            type: PathPrefix
            value: /
      backends:
        - computeInstance: acme-frontend-1
          port: 3000
status:
  state: READY
  routeRef:
    name: acme-corp-acme-web-api
    namespace: osac-acme-corp
```

```yaml
# L4 example (TCP load balancing for database)
apiVersion: osac.io/v1
kind: LoadBalancerTarget
metadata:
  name: acme-db-pool
  annotations:
    osac.openshift.io/tenant: acme-corp
spec:
  loadBalancer: acme-db-lb
  listener: postgres
  backends:
    - computeInstance: acme-db-primary
      port: 5432
      weight: 100
    - computeInstance: acme-db-replica
      port: 5432
      weight: 0
  healthCheck:
    port: 5432
    intervalSeconds: 5
    unhealthyThreshold: 2
  sessionAffinity:
    type: ClientIP
    timeoutSeconds: 3600
```

### Workflow

#### L7 load balancer creation

1. Tenant creates LoadBalancer with `loadBalancerClass:
   envoy-l7`, listeners, and TLS config.
2. Fulfillment-service validates (quota, class capabilities,
   tenant auth), creates record.
3. osac-operator reads LoadBalancerClass `envoy-l7`, gets
   `collection: osac.lb_envoy_gateway`.
4. Looks up `load_balancer.create.main` in that collection.
5. Generates AAP Workflow, submits.
6. Role allocates PublicIP, creates a Kubernetes Gateway
   resource in the tenant's namespace referencing the Envoy
   Gateway GatewayClass, and creates cert-manager Certificate
   if TLS termination requested.
4. Tenant creates LoadBalancerTarget with routing rules.
5. AAP role creates HTTPRoute referencing the Gateway and
   targeting ComputeInstance Services.
6. Envoy Gateway programs the data plane.
7. Status updated with public IP and readiness.

### API Extensions

#### New resources (registered by add-on)

- `LoadBalancerClass` — private API (provider-defined)
- `LoadBalancer` — public API (tenant-facing)
- `LoadBalancerTarget` — public API (tenant-facing)

#### New CRDs

- `loadbalancers.osac.io`
- `loadbalancertargets.osac.io`

#### Proto definitions

```
load_balancer_class_type.proto
load_balancer_classes_service.proto
load_balancer_type.proto
load_balancers_service.proto
load_balancer_target_type.proto
load_balancer_targets_service.proto
```

### Implementation Details

#### Collections (per provider)

```
osac.lb_envoy_gateway collection:
  load_balancer.create.main     # provisions Gateway + PublicIP
  load_balancer.delete.main     # tears down Gateway + releases IP
  target.create.main            # creates HTTPRoute/TLSRoute
  target.delete.main            # removes Route

osac.lb_metallb collection:
  load_balancer.create.main     # provisions L4 Gateway + PublicIP
  load_balancer.delete.main     # tears down Gateway
  target.create.main            # creates TCPRoute/UDPRoute
  target.delete.main            # removes Route
```

Each collection is self-contained — no dispatcher role.

#### Gateway API mapping

| OSAC Resource | Gateway API Resource |
|---|---|
| LoadBalancerClass | GatewayClass (1:1, CSP-managed) |
| LoadBalancer | Gateway (created in tenant namespace) |
| LoadBalancer.listeners | Gateway.spec.listeners |
| LoadBalancerTarget (HTTP) | HTTPRoute |
| LoadBalancerTarget (TCP) | TCPRoute |
| LoadBalancerTarget (UDP) | UDPRoute |

#### PublicIP integration

LoadBalancer reuses the PublicIP allocation from MGMT-23730.
The AAP role allocates a PublicIP from the specified pool and
configures it as the Gateway's external address. MetalLB
advertises the IP via L2/BGP.

#### TLS and certificate management

When a listener specifies `tls.mode: Terminate`, the AAP role
creates a cert-manager Certificate resource. cert-manager
provisions the certificate (via Let's Encrypt, internal CA, or
Vault) and stores it in a Secret referenced by the Gateway
listener.

#### Kuadrant integration (optional)

If Kuadrant is deployed, CSP admins can attach policies to
LoadBalancerClasses:

- `RateLimitPolicy` — per-tenant or per-route rate limiting
- `AuthPolicy` — additional auth requirements beyond OSAC's
  tenant model
- `DNSPolicy` — multi-cluster DNS-based traffic distribution

These are configured at the GatewayClass/Gateway level, not
exposed as OSAC resources. Kuadrant integration is optional and
does not affect the core LoadBalancer API.

#### Database

New tables via GenericDAO:
- `load_balancer_classes`
- `load_balancers`
- `load_balancer_targets`

#### Provider collections

| Provider | Collection | Use case |
|---|---|---|
| Envoy Gateway | `osac.lb_envoy_gateway` | Default L7, lightweight |
| MetalLB | `osac.lb_metallb` | L4 TCP/UDP on bare metal |
| Istio/Sail | `osac.lb_istio` | L7 with service mesh features |
| F5 BIG-IP | `f5.osac_lb` | Enterprise HW LB (partner) |
| NGINX | `nginx.osac_lb` | NGINX Ingress Controller (partner) |

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Gateway API TCPRoute/UDPRoute are beta, not GA | Monitor sig-network progress; TCPRoute is widely implemented |
| Multiple Gateway API implementations may conflict | One GatewayClass per implementation; LoadBalancerClass isolates tenants from implementation details |
| PublicIP pool exhaustion under many load balancers | Quota enforcement per tenant; shared Gateway with multiple listeners reduces IP consumption |
| Envoy Gateway not officially supported on OpenShift | OpenShift Router (HAProxy) supports Gateway API natively as of OCP 4.15; use as fallback |

## Test Plan

- Unit tests for LoadBalancer servers (GenericServer-based CRUD).
- Integration tests with Kind:
  - Deploy add-on, create LoadBalancerClass, LoadBalancer,
    LoadBalancerTarget.
  - Verify Gateway and HTTPRoute creation.
  - Verify PublicIP allocation.
- E2E tests:
  - L7: HTTP traffic routing to ComputeInstances via Envoy
    Gateway.
  - L4: TCP load balancing via MetalLB.
  - TLS termination with cert-manager.
  - Health check failover.

## Graduation Criteria

### Alpha

- LoadBalancerClass, LoadBalancer, LoadBalancerTarget API.
- Envoy Gateway provider (L7 HTTP/HTTPS).
- MetalLB provider (L4 TCP).
- PublicIP integration.
- CLI support.

### Beta

- TLS termination with cert-manager.
- Health checks and session affinity.
- Istio/Sail provider.
- Kuadrant rate limiting integration.
- UI plugin.

### GA

- Partner providers (F5, NGINX).
- UDP load balancing.
- Multi-cluster DNS via Kuadrant DNSPolicy.
