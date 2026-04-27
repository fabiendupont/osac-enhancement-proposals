---
title: metal3-compute-backend
authors:
  - Fabien Dupont
creation-date: 2026-04-17
last-updated: 2026-04-17
tracking-link:
  - TBD
see-also:
  - enhancements/unified-compute-model/README.md
  - enhancements/inventory-provisioning-separation/README.md
  - enhancements/image-and-sshkey-resources/README.md
  - enhancements/carbide-integration/README.md
  - https://github.com/osac-project/enhancement-proposals/pull/20 (Region/AZ)
  - https://github.com/osac-project/enhancement-proposals/pull/31 (Bare Metal Fulfillment v2)
replaces:
superseded-by:
---

# Metal3 ComputeInstanceTemplate Backend

## Summary

Implement a Metal3/Ironic-based ComputeInstanceTemplate backend for
OSAC that provisions bare-metal compute instances using BareMetalHost
custom resources. This backend provisions bare-metal compute
instances for GPU-accelerated and general-purpose deployments.

Metal3 is the standard Kubernetes-native bare-metal provisioning
system used by OpenShift. This backend enables OSAC to provision
bare metal without depending on NICo, ESI, or any other proprietary
infrastructure controller.

## Motivation

### Current state

OSAC has two compute provisioning backends:
- **KubeVirt** (`ocp_virt_vm` template role) — provisions VMs
- **ESI** (`massopencloud.esi` collection) — provisions bare metal
  at MOC using OpenStack Ironic

A third backend (NICo/Carbide) is proposed in the Carbide
integration EP but not yet implemented.

None of these use Metal3, despite Metal3 being the standard
bare-metal provisioning system in the OpenShift ecosystem. Metal3 is
GA, commercially supported by Red Hat, and deployed at scale in
OpenShift bare-metal IPI installations.

### Why Metal3

- **GA and supported.** Metal3 is included in OpenShift and
  supported by Red Hat. NICo is experimental. ESI is
  MOC-specific.
- **Kubernetes-native.** BareMetalHost CRDs managed by the Bare
  Metal Operator (BMO). No separate API server, database, or
  workflow engine.
- **Standard BMC protocols.** Redfish, IPMI, iDRAC — works with
  any server vendor.
- **OpenShift integration.** Machine API, HyperShift, Assisted
  Installer all integrate with Metal3.
- **Ecosystem.** CAPI provider (cluster-api-provider-metal3), CSI
  drivers, monitoring — mature ecosystem.

### User Stories

- As a **cloud provider**, I want to use Metal3 as my bare-metal
  provisioning backend, so that I can use GA, supported Red Hat
  technology.

- As a **tenant**, I want to provision bare-metal GPU servers
  through the same ComputeInstance API I use for VMs.

### Goals

- Implement a Metal3 ComputeInstanceTemplate as an Ansible role in
  `osac.templates`.
- Support RHEL (image-mode/bootc) and RHCOS images via Metal3.
- Follow the inventory/provisioning separation pattern from the
  [inventory-provisioning-separation](../inventory-provisioning-separation/README.md)
  EP.
- Support ComputeInstanceGroup scaling.

### Non-Goals

- Replacing Metal3 with a custom provisioning system.
- Managing the Metal3/Ironic deployment itself (the operator is
  pre-installed on the management cluster).
- DPU lifecycle management (handled by DPF Operator separately).
- Switch fabric automation (handled by Netris or nvidia.nvue
  separately).

## Proposal

### ComputeInstanceTemplate definition

```yaml
id: "metal3-b200-paris"
backend: "metal3"
site: "paris"
role: "metal3_bm"
roleCollection: "osac.templates"
```

The `site` field maps to the Region/AZ topology defined in the
[Region/AZ EP](https://github.com/osac-project/enhancement-proposals/pull/20).
A ComputeInstanceClass references one template per site, enabling
multi-site deployments where the same class is available at
different locations.

The template role `osac.templates.metal3_bm` implements three
entry points: `install.yaml`, `delete.yaml`, and `status.yaml`.

### Provisioning workflow (install.yaml)

The role receives pre-selected hosts from `osac.service.select_hosts`
(per the inventory/provisioning separation EP).

```
Input:
  compute_instance_hosts:     # pre-selected from AAP inventory
  compute_instance_image:     # resolved Image (sourceRef, bootMethod, checksum)
  compute_instance_ssh_keys:  # resolved SSH public key strings
  compute_instance_subnet:    # subnet reference
  compute_instance_class:     # ComputeInstanceClass
  compute_instance_user_data: # user data string

Steps:

1. Create BareMetalHost CR
   │ - spec.online: true
   │ - spec.bootMACAddress: from host_vars
   │ - spec.bmc.address: from host_vars (Redfish URL)
   │ - spec.bmc.credentialsName: from host_vars (Secret reference)
   │ - spec.image.url: from resolved Image sourceRef
   │ - spec.rootDeviceHints: from host_vars or class
   │
2. Generate OS configuration
   │ Based on image.bootMethod:
   │ - ignition: generate Ignition config with SSH keys + userData
   │ - cloud-init: generate cloud-init userdata with SSH keys
   │ - kickstart: generate Kickstart with SSH keys in %post
   │
3. Wait for BareMetalHost provisioning
   │ Watch BMH status.provisioning.state → "provisioned"
   │ Timeout: configurable (default 30 minutes)
   │
4. Report success
   │ Set compute_instance_ip_address from BMH status
   │ Set compute_instance_state: "RUNNING"

**Networking and fabric isolation** are not handled by the
template role. Network attachment (VPC, subnet, security groups),
NVLink partition management (nvidia.nmx), and InfiniBand PKEY
isolation (nvidia.ufm) are the responsibility of their respective
NetworkClass controllers. These controllers watch for
ComputeInstances attached to their subnets and create the
appropriate resources (DPUService CRDs for dpf-ovn-vpc, NVLink
partitions for GPU fabric, IB PKEYs for InfiniBand fabric).
This follows the same separation principle as the
inventory/provisioning separation: template roles provision
compute, network providers provision networking and fabric
isolation.
```

### Deprovisioning workflow (delete.yaml)

```
Steps:

1. Deprovision BareMetalHost
   │ Set BMH spec.online: false
   │ Wait for BMH → deprovisioning → available
   │
2. Clean up userdata Secret
   │ Delete the per-host userdata Secret
   │
3. Update inventory (handled by the workflow, not the template role)
   │ osac.service.update_host_state (state: available, tenant: none)

**Networking and fabric cleanup** is not handled by the template
role. NetworkClass controllers are responsible for removing
network and fabric isolation resources (NVLink partitions, IB
PKEYs, VPC attachments) when the ComputeInstance is deleted.
```

### AAP execution environment

The Metal3 template role requires `kubernetes.core` for
BareMetalHost CRD management. This should NOT be added to the
monolithic shared execution environment. Instead, the Metal3
backend should ship its own EE image that extends the OSAC core
EE:

```
EE: osac-core (base — shared by all backends)
  ├── kubernetes.core, community.general, ansible.utils
  ├── ansible.controller, ansible.eda
  └── osac.service, osac.templates, osac.workflows

EE: osac-metal3 (this backend)
  └── FROM osac-core
```

Inventory plugin collections (e.g., `netbox.netbox`) belong in
the AAP inventory configuration, not in the template role's EE —
the template role receives pre-selected hosts from
`osac.service.select_hosts` and never queries inventory directly.

GPU fabric collections (`nvidia.nmx`, `nvidia.ufm`) belong in
the EE of the network provider that handles NVLink/IB isolation,
not in the compute provisioning EE.

Each `ComputeInstanceTemplate` specifies which EE its AAP job
template requires. The osac-operator passes this to AAP when
launching the job.

This pluggable EE model applies to all backends (ESI, NICo,
KubeVirt). The `osac-aap-ee` repo should evolve from a single
monolithic EE definition to a set of layered Containerfiles
(core + per-backend).

### Inventory requirements

Hosts provisioned via Metal3 must have these host_vars in AAP
inventory (per the inventory/provisioning separation EP):

| host_var | Source | Required |
|---|---|---|
| `compute_instance_class` | Inventory plugin | Yes |
| `state` | Inventory plugin | Yes |
| `site` | Inventory plugin | Yes |
| `bmc_address` | Inventory plugin | Yes |
| `bmc_credentials_secret` | Inventory plugin | Yes |
| `boot_mac_address` | Inventory plugin | Yes |
| `rack` | Inventory plugin | For placement |
| `root_device` | Inventory plugin | Optional (default: /dev/nvme0n1) |

### ComputeInstanceClass examples

**Bare-metal GPU (inference):**

```yaml
id: "gpu-b200-4"
title: "4x B200 GPU Tray"
hardwareType: "baremetal"
capabilities:
  coresFixed: 96
  memoryGibFixed: 480
  gpus:
    count: 4
    model: "B200"
  storage:
    bootDiskGibFixed: 960
templates:
  - name: "metal3-b200-paris"
    site: "paris"
  - name: "metal3-b200-london"
    site: "london"
```

**Bare-metal CPU (storage/management):**

```yaml
id: "cpu-mgmt-64"
title: "64-core Management Node"
hardwareType: "baremetal"
capabilities:
  coresFixed: 64
  memoryGibFixed: 256
  storage:
    bootDiskGibFixed: 480
templates:
  - name: "metal3-cpu-paris"
    site: "paris"
```

### Integration with other OSAC components

#### Cluster fulfillment

ClusterTemplate node sets reference ComputeInstanceClasses. When
a ClusterOrder is created, the cluster template role uses the
Metal3 backend to provision worker nodes, then attaches them to
a HyperShift HostedCluster.

This aligns cluster fulfillment with the unified compute model —
cluster workers are ComputeInstances provisioned by a
ComputeInstanceTemplate.

#### Networking

The Metal3 backend delegates networking to the NetworkClass
mechanism. It does not implement VPC/subnet/NSG management
directly — it creates the compute resource and attaches it to the
tenant's network as specified by the subnet reference.

#### DPF (DPU lifecycle)

DPF Operator manages DPU firmware and services independently of
the Metal3 backend. The DPF Operator runs on the management cluster
and manages DPUs on all bare-metal hosts, regardless of which
ComputeInstanceTemplate provisioned them.

### Risks and Mitigations

#### Metal3/Ironic must be pre-deployed

The Metal3 backend assumes BMO and Ironic are running on the
management cluster.

*Mitigation:* This is standard for OpenShift bare-metal
deployments. Document prerequisites. The management cluster
setup guide covers Metal3 deployment.

#### BareMetalHost CRDs must be pre-created

Metal3 requires BareMetalHost resources to exist before
provisioning. These represent the physical machines.

*Mitigation:* BareMetalHost resources can be created by an
inventory sync job or manual registration. This is a day-0 setup
task, not a per-tenant operation.

### Drawbacks

- Metal3/Ironic adds infrastructure to the management cluster.
- BareMetalHost pre-registration is a manual step (or requires
  inventory sync automation).

## Alternatives (Not Implemented)

### Use NICo as the sole bare-metal backend

Rely on NICo for all bare-metal provisioning.

*Why not:* NICo is experimental with no support commitment. Metal3
is GA and supported. Providers should have a choice.

### Use ESI as the bare-metal backend

Continue with the MOC's ESI backend.

*Why not:* ESI is specific to OpenStack environments. Metal3 is
Kubernetes-native and works with any server that supports Redfish.

### Use NICo for GPU features, Metal3 for provisioning

Hybrid approach where Metal3 handles PXE/OS provisioning and NICo
handles networking and GPU fabric isolation.

*Why this is possible but not required:* The unified compute model
separates compute provisioning (template roles) from networking
(NetworkClass controllers). A provider can use Metal3 for compute
and NICo-based NetworkClass controllers for fabric isolation, or
use standalone nvidia.nmx/nvidia.ufm-based controllers. This EP
focuses on the compute provisioning path; the network provider
choice is independent.

### Relationship to BareMetalPool / HostLease (PR #31)

The [updated bare-metal-fulfillment EP](https://github.com/osac-project/enhancement-proposals/pull/31)
would implement Metal3 as a Host Management Operator — a Go
controller that reconciles HostLease CRDs by creating
BareMetalHost resources and managing their lifecycle.

This EP implements the same compute provisioning logic as a
ComputeInstanceTemplate Ansible role executed by AAP. The
BareMetalHost creation and OS configuration steps are equivalent.
The difference is execution
context: a Kubernetes operator (PR #31) versus an AAP-executed
Ansible role (this EP). The Ansible role approach is consistent
with how OSAC handles all provisioning today — KubeVirt VMs, ESI
bare metal, and cluster fulfillment all use AAP template roles.

## Open Questions [optional]

1. **BareMetalHost pre-creation.** Should the Metal3 role create
   BareMetalHost CRDs from inventory data at provisioning time, or
   should BMH CRDs be pre-created by a separate inventory sync job?
   Pre-creation simplifies the role but requires a day-0 sync
   mechanism.

2. **Provisioning timeout handling.** When a BareMetalHost fails to
   reach `provisioned` state within the timeout, should the role
   retry the same host, select a different host via
   `osac.service.select_hosts`, or fail the ComputeInstance?

## Test Plan

TBD — will cover:
- BareMetalHost creation and provisioning lifecycle
- Image boot method handling (Ignition, cloud-init)
- ComputeInstanceGroup scaling (add/remove bare-metal hosts)
- Deprovisioning and inventory state update

## Graduation Criteria

TBD

## Upgrade / Downgrade Strategy

The Metal3 backend is an Ansible role packaged in a per-backend
execution environment (EE) image. Upgrading means deploying a new
EE image version. Downgrading means reverting to the previous EE
image. The `status.yaml` entry point should handle
partially-provisioned states gracefully so that a role upgrade
mid-provisioning does not leave BareMetalHost resources in an
inconsistent state.

## Version Skew Strategy

The Metal3 role runs in AAP and interacts with BareMetalHost CRDs
managed by the Bare Metal Operator (BMO) on the management cluster.
The role must be compatible with the installed BMO version. The EE
image should pin and test against specific BMO versions to avoid
API incompatibilities.

## Support Procedures

- **ComputeInstance stuck in PENDING:** Check the AAP job output
  for the `metal3_bm` role. Verify that matching hosts exist in
  AAP inventory with `state: available` and correct `compute_instance_class`.
  Check BareMetalHost CR status on the management cluster.
- **BareMetalHost stuck in `provisioning`:** Check Ironic logs on
  the management cluster. Common causes: unreachable BMC, invalid
  image URL, network boot failure.

## Infrastructure Needed [optional]

- Metal3 / Ironic deployed on the management cluster
