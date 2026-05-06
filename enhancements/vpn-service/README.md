---
title: OSAC VPN Service Add-On
authors:
  - fdupont@redhat.com
creation-date: 2026-04-30
last-updated: 2026-04-30
tracking-link:
  - TBD
see-also:
  - enhancements/osac-addon/README.md
  - enhancements/networking/README.md
replaces:
superseded-by:
---

# OSAC VPN Service Add-On

## Summary

Introduce VPN-as-a-Service for OSAC tenants as an **OSAC
Add-On**, following the contract defined in the
[OSAC Add-On EP](../osac-addon/README.md). The add-on is
delivered as per-provider Ansible collections following the
ResourceAction naming convention.

Two initial providers: **OVN-Kubernetes IPSec**
(`osac.vpn_ovnk`) and **WireGuard**
(`osac.vpn_wireguard`).

## Motivation

### Current state

OSAC provides tenant-isolated VirtualNetworks, Subnets, and
SecurityGroups. Tenants can attach ComputeInstances to private
networks and allocate PublicIPs for external access. However,
there is no way to establish encrypted tunnels between:

- Two tenant VirtualNetworks in different regions
- A tenant VirtualNetwork and an external (on-premises) network
- Two tenants who need a secure interconnect

### Problems

1. **No site-to-site connectivity.** Tenants with hybrid
   deployments cannot securely connect OSAC workloads to
   on-premises infrastructure without manual VPN configuration
   inside their VMs.

2. **No inter-region encryption.** Traffic between
   VirtualNetworks in different regions traverses the network
   unencrypted at the overlay level.

3. **No tenant-managed VPN.** Tenants who need VPN access for
   remote users must deploy and manage their own VPN appliance
   VMs, duplicating effort across tenants.

### Comparison with VMware Cloud Director

VCD provides two VPN services via NSX Edge Gateways:

| Dimension | VCD VPN | OSAC VPN (proposed) |
|---|---|---|
| IPSec site-to-site | NSX Edge IPSec (IKEv1/v2) | OVN-K IPSec (IKEv2) |
| SSL/remote access VPN | NSX SSL VPN-Plus | WireGuard peer-based |
| Configuration | Edge Gateway UI/API | VPNConnection CR via OSAC API |
| Scope | Per Edge Gateway (per Org VDC) | Per VirtualNetwork (tenant-scoped) |
| Provider model | NSX-only | Pluggable VPNClass (OVN-K, WireGuard, future) |

### User Stories

- As a **tenant**, I want to create a site-to-site IPSec VPN
  between my OSAC VirtualNetwork and my on-premises network so
  that workloads can communicate securely across locations.

- As a **tenant**, I want to connect two VirtualNetworks in
  different regions via an encrypted tunnel so that my
  multi-region deployment has secure east-west traffic.

- As a **CSP admin**, I want to offer VPN connectivity as a
  managed service with pluggable backends so that I can choose
  the VPN technology that fits my infrastructure.

- As a **remote user**, I want a WireGuard peer configuration
  so that I can securely access resources in my tenant's
  VirtualNetwork from my laptop.

### Goals

- Introduce VPNClass and VPNConnection as first-class OSAC
  resources.
- Implement OVN-Kubernetes IPSec as the default VPN provider
  for site-to-site tunnels.
- Implement WireGuard as a lightweight provider for
  peer-to-peer and remote access use cases.
- Follow the pluggable provider pattern (VPNClass dispatches
  to AAP roles).
- Integrate with VirtualNetwork — VPN connections attach to
  VirtualNetworks.

### Non-Goals

- Full-mesh VPN across all tenant networks (tenants configure
  individual connections).
- SD-WAN or traffic engineering (out of scope).
- VPN client software distribution (tenant's responsibility).
- Certificate authority management (use external CA or
  Keycloak-issued certificates).

## Proposal

### Ansible Collections (per provider)

Following the [OSAC Add-On convention](../osac-addon/README.md),
each VPN provider ships its own collection:

#### osac.vpn_ovnk

```yaml
# meta/addon.yaml
name: osac-vpn-ovnk
display_name: VPN Service (OVN-K IPSec)
description: IPSec VPN via OVN-Kubernetes
version: 1.0.0

dependencies:
  osac_core: ">=0.1.0"
  addons:
    - osac-networking

resource_types:
  - name: VPNClass
    scope: provider
  - name: VPNConnection
    scope: tenant
```

```
roles/
├── connection.create.main/
│   ├── meta/osac.yaml
│   └── tasks/main.yml
├── connection.delete.main/
│   ├── meta/osac.yaml
│   └── tasks/main.yml
└── connection.signal.main/
    ├── meta/osac.yaml
    └── tasks/main.yml
```

#### osac.vpn_wireguard

Same role structure, different implementation. Each collection
is self-contained — no dispatcher role, no conditional logic.

### Personas

| Persona | Role |
|---|---|
| CSP Admin | Defines VPNClasses, manages VPN infrastructure |
| Tenant Admin | Creates VPN connections between networks |
| Tenant User | Consumes VPN connectivity, configures remote peers |

### Resource Types

#### VPNClass

Provider-defined, specifies the VPN technology and configuration
defaults. Follows the NetworkClass pattern.

```yaml
apiVersion: osac.io/v1
kind: VPNClass
metadata:
  name: ovnk-ipsec
  labels:
    osac.io/addon: osac-vpn-ovnk
spec:
  collection: osac.vpn_ovnk
  capabilities:
    - SITE_TO_SITE
    - INTER_REGION
  defaults:
    ikeVersion: IKEv2
    encryptionAlgorithm: AES-256-GCM
    dhGroup: 20
```

```yaml
apiVersion: osac.io/v1
kind: VPNClass
metadata:
  name: wireguard
  labels:
    osac.io/addon: osac-vpn-wireguard
spec:
  collection: osac.vpn_wireguard
  capabilities:
    - SITE_TO_SITE
    - REMOTE_ACCESS
  defaults:
    listenPort: 51820
    persistentKeepalive: 25
```

#### VPNConnection

Tenant-scoped resource representing a VPN tunnel between two
endpoints.

```yaml
apiVersion: osac.io/v1
kind: VPNConnection
metadata:
  name: hq-to-osac
  annotations:
    osac.openshift.io/tenant: acme-corp
spec:
  vpnClass: ovnk-ipsec
  virtualNetwork: acme-production
  localEndpoint:
    publicIP: acme-public-ip-1
    subnets:
      - 10.100.0.0/16
  remoteEndpoint:
    address: 203.0.113.1
    subnets:
      - 192.168.0.0/16
    presharedKey:
      secretRef:
        name: hq-vpn-psk
        key: psk
  ike:
    version: IKEv2
    encryptionAlgorithm: AES-256-GCM
    integrityAlgorithm: SHA-384
    dhGroup: 20
    lifetime: 28800
status:
  state: ESTABLISHED
  conditions:
    - type: TunnelUp
      status: "True"
      lastTransitionTime: "2026-04-30T10:00:00Z"
    - type: RemoteReachable
      status: "True"
  tunnelIP: 10.255.0.1
  bytesIn: 1048576
  bytesOut: 524288
```

### Workflow

#### Site-to-site IPSec (OVN-K)

1. CSP admin creates VPNClass `ovnk-ipsec`.
2. Tenant creates VPNConnection referencing a VirtualNetwork
   and a PublicIP, with remote endpoint details.
3. Fulfillment-service validates the VPNConnection, creates a
   VPNConnection CR in Kubernetes.
4. osac-operator reads VPNClass `ovnk-ipsec`, gets
   `collection: osac.vpn_ovnk`.
5. Looks up cached `connection.create.main` role in that
   collection.
6. Generates AAP Workflow, submits.
7. The `connection.create.main` role configures OVN-K IPSec
   policies on the nodes hosting the VirtualNetwork's subnets.
7. OVN-K establishes the IPSec tunnel using the configured
   parameters.
8. Operator reads `set_stats` outputs and updates
   VPNConnection status (state, tunnel metrics).

#### WireGuard remote access

1. CSP admin creates VPNClass `wireguard`.
2. Tenant creates VPNConnection with `vpnClass: wireguard` and
   `type: REMOTE_ACCESS`.
3. Operator reads VPNClass `wireguard`, gets
   `collection: osac.vpn_wireguard`.
4. Same dispatch flow: looks up `connection.create.main` in
   that collection, generates AAP Workflow.
5. The role deploys a WireGuard pod (or KubeVirt VM) in the
   tenant's namespace, generates server keys, configures
   AllowedIPs from the VirtualNetwork's subnets.
5. Role reports outputs via `set_stats`: public key, endpoint,
   AllowedIPs.
6. Tenant retrieves peer configuration from VPNConnection
   status to configure their WireGuard client.

### API Extensions

#### New resources

- `VPNClass` — private API (provider-defined)
- `VPNConnection` — public API (tenant-facing, CRUD + status)

#### New CRDs

- `vpnconnections.osac.io` — reconciled by osac-operator

#### Proto definitions

```
vpn_class_type.proto
vpn_classes_service.proto
vpn_connection_type.proto
vpn_connections_service.proto
```

### Implementation Details

#### Collections (per provider)

```
osac.vpn_ovnk collection:
  connection.create.main     # provisions IPSec tunnel
  connection.delete.main     # tears down tunnel
  connection.signal.main     # status feedback

osac.vpn_wireguard collection:
  connection.create.main     # deploys WireGuard peer
  connection.delete.main     # removes peer
  connection.signal.main     # status feedback
```

Each collection is self-contained. No dispatcher role.

#### OVN-K IPSec implementation (osac.vpn_ovnk)

OVN-Kubernetes supports IPSec natively via the `ipsec-config`
ConfigMap in `openshift-ovn-kubernetes` namespace. The role:

1. Creates IPSec policies targeting the VirtualNetwork's OVN
   logical switch ports.
2. Configures IKE parameters via Libreswan (OVN-K's IPSec
   daemon).
3. Manages pre-shared keys or certificates in Kubernetes
   Secrets.

#### WireGuard implementation (osac.vpn_wireguard)

The `connection.create.main` role:

1. Deploys a WireGuard pod with `NET_ADMIN` capability in the
   tenant's namespace.
2. Attaches the pod to the VirtualNetwork's subnet via
   NetworkAttachmentDefinition.
3. Configures the WireGuard interface with server keys and
   allowed CIDRs.
4. Exposes the WireGuard endpoint via a PublicIP (from
   PublicIPPool).

#### Database

New tables following the generic schema from `dao_tables.go`:
- `vpn_classes`
- `vpn_connections`

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| OVN-K IPSec config is cluster-wide, may conflict with other IPSec policies | Scope policies to VirtualNetwork-specific OVN logical switches |
| WireGuard pods require NET_ADMIN, security concern in multi-tenant | Run in dedicated namespace with NetworkPolicy isolation; use SecurityContext constraints |
| IPSec performance overhead on tenant workloads | Document performance characteristics per VPNClass; offer WireGuard as lower-overhead alternative |
| Key management complexity | Integrate with cert-manager for certificate-based auth; support Secret-referenced PSKs |

## Test Plan

- Unit tests for VPN servers (GenericServer-based CRUD).
- Integration tests with Kind cluster:
  - Create VPNClass and VPNConnection.
  - Verify OVN-K IPSec policy creation (mock AAP).
  - Verify WireGuard pod deployment and configuration.
- E2E tests with multi-node cluster:
  - Establish IPSec tunnel between two VirtualNetworks.
  - Verify encrypted traffic flow via tcpdump.
  - Establish WireGuard tunnel and verify connectivity.

## Graduation Criteria

### Alpha

- VPNClass and VPNConnection API (private + public).
- OVN-K IPSec provider (site-to-site only).
- Basic CLI support (`osac vpn create/describe/delete`).

### Beta

- WireGuard provider (site-to-site + remote access).
- VPN metrics in Prometheus (tunnel status, bytes in/out).
- UI integration in osac-ui (when UI plugin system available).

### GA

- Certificate-based authentication (cert-manager integration).
- Inter-region VPN automation.
- Partner VPN providers (Libreswan standalone, strongSwan).
