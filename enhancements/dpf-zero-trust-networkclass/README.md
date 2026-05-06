---
title: dpf-zero-trust-networkclass
authors:
  - Fabien Dupont
creation-date: 2026-04-29
last-updated: 2026-04-29
tracking-link:
  - TBD
see-also:
  - enhancements/osac-addon/README.md
  - enhancements/networking/README.md
  - enhancements/unified-compute-model/README.md
  - enhancements/metal3-compute-backend/README.md
replaces:
superseded-by:
---

# DPF Zero Trust NetworkClass Provider

## Summary

Implement a `dpf-zero-trust` NetworkClass provider for OSAC that
leverages the DPF (DOCA Platform Framework) Operator and BlueField
DPU hardware enforcement for tenant network isolation. When a
tenant creates a VirtualNetwork with `networkClass: dpf-zero-trust`,
network policies are programmed on the DPU via DPUServiceChain CRDs
and HBN (Host-Based Networking) configuration, making them
**tamper-proof even for bare-metal tenants with root access**.

This is the second NetworkClass provider for OSAC, alongside
`udn-net` (OVN-Kubernetes UDN). The tenant-facing API
(VirtualNetwork, Subnet, SecurityGroup) is identical — only the
enforcement mechanism differs.

## Motivation

### Current state

The networking EP defines a single NetworkClass: `udn-net`, which
uses OVN-Kubernetes User Defined Networks for tenant isolation.
`udn-net` works well for VM-based multi-tenancy where the
hypervisor is the trust boundary.

For bare-metal multi-tenancy — the primary NCP use case — `udn-net`
is insufficient. A bare-metal tenant with root access runs on the
host kernel directly. OVN-K network policies are enforced in the
host kernel's OVS datapath, which a privileged tenant could bypass,
inspect, or modify.

### Why DPF Zero Trust

The DPU sits between the host and the physical network. All traffic
entering or leaving the host passes through the DPU's datapath.
Network policies programmed on the DPU are enforced in hardware
(OVS-DOCA on BlueField ARM cores), outside the host OS trust
boundary. The host cannot see, modify, or bypass these policies.

This is materially different from `udn-net`:

| Capability | `udn-net` | `dpf-zero-trust` |
|-----------|-----------|-------------------|
| Enforcement point | Host kernel (OVS) | DPU hardware (OVS-DOCA) |
| Bare-metal tenants | Cannot guarantee isolation | Hardware-enforced isolation |
| Policy visibility | Tenant can inspect OVS flows | Policies invisible to tenant |
| Throughput | Software datapath | Hardware-accelerated |
| CPU overhead | Host CPU processes OVN | Offloaded to DPU ARM cores |
| Telemetry | In-guest agents required | DPU-level telemetry (outside tenant boundary) |
| Trust model | Host OS is trusted | DPU is trust boundary |

On BlueField Astra (BlueField-4, Vera Rubin platform, 2026), the
zero-trust model strengthens further: the DPU's control plane is
fully isolated via out-of-band connectivity, and DOCA Argus
provides infrastructure-level telemetry without in-guest agents.
The `dpf-zero-trust` provider would gain these capabilities
transparently — same OSAC API, stronger enforcement underneath.

### User Stories

- As a **cloud provider**, I want to offer hardware-enforced
  network isolation for bare-metal tenants, so that tenants with
  root access cannot bypass network policies.

- As a **tenant**, I want to use the same VirtualNetwork/Subnet/
  SecurityGroup API regardless of whether my compute is VM or
  bare metal.

- As a **cloud provider**, I want to run both `udn-net` (for VMs)
  and `dpf-zero-trust` (for bare metal) in the same deployment.

### Goals

- Implement a `dpf-zero-trust` NetworkClass provider using DPF
  Operator CRDs.
- Reconcile VirtualNetwork as DPU-level VPC (VxLAN VNI + VRF via
  HBN).
- Reconcile Subnet as DPU-level network segment.
- Reconcile SecurityGroup as DPU-level ACL flows (OVS-DOCA).
- Support the same tenant-facing API as `udn-net`.

### Non-Goals

- Managing DPU lifecycle (firmware, BFB flashing) — that is the
  DPF Operator's responsibility. The NetworkClass provider
  configures services on already-running DPUs.
- Replacing `udn-net` — both providers coexist.
- GPU fabric isolation (NVLink, InfiniBand) — handled by the
  ComputeInstanceGroup GPU fabric workflow, not by the
  NetworkClass provider.

## Proposal

### NetworkClass definition

```yaml
apiVersion: o-sac.openshift.io/v1alpha1
kind: NetworkClass
metadata:
  name: dpf-zero-trust
spec:
  implementationStrategy: "dpf-zero-trust"
  capabilities:
    supportsIpv4: true
    supportsIpv6: true
    supportsDualStack: true
  constraints: {}
status:
  state: Ready
```

The provider registers as a handler for
`implementationStrategy: "dpf-zero-trust"` in the osac-operator.

### Reconciliation flow

#### VirtualNetwork creation

```
Tenant creates VirtualNetwork (networkClass: dpf-zero-trust)
  │
  ▼
osac-operator (dpf-zero-trust reconciler):
  1. Create DPUServiceChain CRD on management cluster
     │  Defines the networking pipeline on the DPU:
     │  OVS-DOCA → HBN → physical fabric
     │
  2. Configure HBN VPC via DPUServiceChain parameters
     │  VxLAN VNI allocated from provider range
     │  VRF created for tenant isolation
     │  VNI → VRF mapping established
     │
  3. Update VirtualNetwork status
     │  state: Ready
     │  Store allocated VNI and VRF ID
```

The DPF Operator watches DPUServiceChain CRDs and programs the
BlueField DPUs in the deployment. The osac-operator does not
interact with DPUs directly — it creates CRDs, DPF does the rest.

#### Subnet creation

```
Tenant creates Subnet in VirtualNetwork
  │
  ▼
osac-operator (dpf-zero-trust reconciler):
  1. Configure HBN subnet within the VPC's VRF
     │  CIDR from Subnet spec (ipv4/ipv6)
     │  Bridge VLAN + SVI on DPU OVS bridge
     │
  2. For udn-net compatibility:
     │  Optionally create a UDN in the Subnet's namespace
     │  so that pods/VMs in the namespace can reach
     │  the DPU-enforced subnet
     │
  3. Update Subnet status
     │  state: Ready
     │  namespace: tenant-<id>-subnet-<name>
```

#### SecurityGroup reconciliation

```
Tenant creates/updates SecurityGroup
  │
  ▼
osac-operator (dpf-zero-trust reconciler):
  1. Translate SecurityGroup rules to OVS-DOCA ACL flows
     │  Protocol, port, source/destination CIDR
     │  Direction (ingress/egress)
     │
  2. Program flows on DPU via DPUServiceChain update
     │  Flows enforced in DPU hardware
     │  Host cannot see or modify these flows
     │
  3. Update SecurityGroup status
     │  state: Ready
```

#### ComputeInstance attachment

```
ComputeInstance created with subnet reference
  │
  ▼
DPU steers host traffic through enforced VPC:
  1. DPF configures SR-IOV VF on DPU interface
  2. VF passed to host (bare metal) or VM (VFIO)
  3. All traffic from VF goes through DPU datapath
  4. DPU enforces VPC isolation and SecurityGroup rules
  5. Host-side network config is minimal (IP assignment)
```

No host-level OVN configuration is needed for bare-metal
ComputeInstances using `dpf-zero-trust`. The DPU is the
network endpoint.

### Architecture

```
┌─────────────────────────────────────────────────┐
│                  OSAC                            │
│  osac-operator ──► dpf-zero-trust reconciler     │
│                    │                             │
│                    ├─► DPUServiceChain CRD        │
│                    ├─► HBN VPC config             │
│                    └─► OVS-DOCA ACL flows         │
└────────────────────┬────────────────────────────┘
                     │ CRDs on management cluster
                     ▼
┌─────────────────────────────────────────────────┐
│              DPF Operator                        │
│  Watches DPUServiceChain ──► Programs DPUs       │
└────────────────────┬────────────────────────────┘
                     │ RSHIM / out-of-band
                     ▼
┌─────────────────────────────────────────────────┐
│           BlueField DPU (per host)               │
│                                                  │
│  ┌──────────┐  ┌─────────┐  ┌────────────────┐  │
│  │ OVS-DOCA │→ │   HBN   │→ │ Physical Fabric│  │
│  │  (ACLs)  │  │(VxLAN/  │  │  (Spectrum/IB) │  │
│  │          │  │  VRF)   │  │                │  │
│  └──────────┘  └─────────┘  └────────────────┘  │
│                                                  │
│  All host traffic passes through this pipeline   │
│  Host OS cannot bypass or inspect these rules    │
└──────────────────────────────────────────────────┘
```

### Interaction with other OSAC components

#### Compute provisioning

The `dpf-zero-trust` NetworkClass works with any
ComputeInstanceTemplate backend (Metal3, NICo, ESI). The Metal3
backend provisions the bare-metal host; the NetworkClass provider
configures network isolation on the DPU. These are independent
concerns.

#### DPF Operator

The DPF Operator must be deployed on the management cluster with
DPUs registered and in `Ready` state. The `dpf-zero-trust`
provider does not manage DPU lifecycle — it assumes DPUs are
operational. If a DPU is in a degraded state, the provider
reports it on the VirtualNetwork status.

The DPF HCP Provisioner Operator provides DPU cluster control
planes via HyperShift. It is targeting GA in OCP 4.22 (June 2026).

#### PublicIP and NAT Gateway

PublicIP allocation and NAT Gateway for `dpf-zero-trust`
VirtualNetworks use the same OSAC resources as `udn-net`. The
implementation differs:

- **PublicIP**: Attached via HBN floating IP configuration on
  the DPU (instead of OVN-K EgressIP)
- **NAT Gateway**: Egress NAT configured on the DPU's HBN
  gateway (instead of OVN-K EgressIP CRD)

#### VPC peering

VPC peering between two `dpf-zero-trust` VirtualNetworks is
configured by linking VRFs in HBN. Cross-provider peering
(between a `udn-net` and a `dpf-zero-trust` VirtualNetwork)
requires a gateway — this is a future enhancement.

### Prerequisites

- DPF Operator GA deployed on management cluster
- DPF HCP Provisioner Operator GA (OCP 4.22, June 2026)
- BlueField-3 DPUs (current) or BlueField Astra/4 (enhanced)
- SR-IOV Network Operator for VF management

### Risks and Mitigations

#### DPUServiceChain CRD stability

The DPUServiceChain CRD is the primary interface between OSAC
and DPF. API changes in DPF could break the provider.

*Mitigation:* Pin and test against specific DPF Operator
versions. The provider's reconciler should handle CRD version
skew gracefully.

#### Cross-provider VPC peering

A tenant cannot peer a `udn-net` VirtualNetwork with a
`dpf-zero-trust` VirtualNetwork.

*Mitigation:* Document this limitation. In practice, bare-metal
(dpf-zero-trust) and VM (udn-net) workloads in the same tenant
would use the same NetworkClass. Cross-provider peering is a
future enhancement.

### Drawbacks

- Requires DPF Operator and BlueField DPUs — not available in
  all deployments.
- Adds a dependency on DPF CRD stability.
- Two NetworkClass providers mean two code paths to maintain and
  test.

## Alternatives (Not Implemented)

### Use udn-net for everything

Use OVN-K UDN for both VM and bare-metal tenants.

*Why not:* UDN policies are enforced in the host kernel. A
bare-metal tenant with root access can bypass them.

### Use NICo VPC as a NetworkClass provider

Implement a `nico-net` provider that calls NICo's VPC API.

*Why not:* NICo is experimental with no support commitment. DPF
is GA and supported by Red Hat. The `dpf-zero-trust` provider
uses the same underlying DPU capabilities that NICo uses (HBN,
OVS-DOCA) but through GA Red Hat-supported CRDs instead of
NICo's REST API.

### Use Netris SDN as a NetworkClass provider

Implement a `netris-net` provider for physical switch fabric.

*Why not:* Netris manages the physical switch fabric (Spectrum
switches), not the DPU datapath. A `netris-net` provider is
complementary — it could coexist with `dpf-zero-trust` for
different layers of the network.

## Open Questions

1. **DPUServiceChain vs direct HBN API.** Should the provider
   interact with DPF via DPUServiceChain CRDs (high-level,
   declarative) or directly configure HBN via NVUE on the DPU
   cluster (low-level, imperative)? DPUServiceChain is cleaner
   but may not expose all HBN configuration knobs.

2. **VNI allocation.** Who allocates VxLAN VNIs — OSAC or DPF?
   If OSAC allocates, it needs a VNI range and must avoid
   conflicts. If DPF allocates, OSAC must query the result.

3. **Multi-DPU consistency.** A ComputeInstanceGroup may span
   multiple hosts, each with its own DPU. How does the provider
   ensure consistent VPC configuration across all DPUs in the
   group? DPF handles this at the DPUSet level, but the provider
   must verify.

4. **BlueField Astra out-of-band control.** On BlueField-4,
   the DPU control plane is fully isolated via out-of-band
   connectivity. Does this change how the provider interacts
   with DPF, or is it transparent?

## Test Plan

TBD — will cover:
- VirtualNetwork creation and VPC isolation on DPU
- Subnet creation with DPU-level bridge/SVI
- SecurityGroup enforcement via OVS-DOCA ACLs
- ComputeInstance attachment via SR-IOV VF
- Cross-tenant traffic blocked at DPU level
- Intra-tenant traffic flows through DPU datapath
- PublicIP and NAT Gateway via HBN
- Scale: multiple VirtualNetworks across multiple DPUs
- Failover: DPU failure handling

## Graduation Criteria

TBD

## Upgrade / Downgrade Strategy

The `dpf-zero-trust` provider is a reconciler in the
osac-operator. Upgrading means deploying a new osac-operator
image. Downgrading means reverting — existing VirtualNetworks
with `networkClass: dpf-zero-trust` would become unreconciled.

## Version Skew Strategy

The provider creates DPUServiceChain CRDs consumed by the DPF
Operator. The osac-operator must be compatible with the installed
DPF Operator version. The provider should support at least the
current and previous DPF CRD versions.

## Support Procedures

- **VirtualNetwork stuck in Pending:** Check DPUServiceChain CRD
  status on the management cluster. Verify DPF Operator is
  running and DPUs are in Ready state.
- **SecurityGroup rules not enforced:** Check OVS-DOCA flow
  tables on the DPU (via DPU cluster access). Verify the
  DPUServiceChain includes the ACL configuration.
- **ComputeInstance cannot reach network:** Verify SR-IOV VF is
  created and assigned. Check HBN VPC configuration on the DPU.

## Infrastructure Needed

- DPF Operator deployed on management cluster
- DPF HCP Provisioner Operator (OCP 4.22)
- BlueField-3 or BlueField Astra DPUs on compute hosts
- SR-IOV Network Operator for VF management
