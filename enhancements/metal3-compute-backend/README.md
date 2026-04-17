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
replaces:
superseded-by:
---

# Metal3 ComputeInstanceTemplate Backend

## Summary

Implement a Metal3/Ironic-based ComputeInstanceTemplate backend for
OSAC that provisions bare-metal compute instances using BareMetalHost
custom resources. This backend integrates with NVLink partition
management (nvidia.nmx) and InfiniBand partition management
(nvidia.ufm) for GPU-accelerated bare-metal deployments.

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

- As a **cloud provider**, I want NVLink partition isolation for
  multi-tenant GPU inference workloads, without requiring NICo.

- As a **cloud provider**, I want InfiniBand tenant isolation for
  HPC/training workloads, without requiring NICo.

- As a **tenant**, I want to provision bare-metal GPU servers
  through the same ComputeInstance API I use for VMs.

### Goals

- Implement a Metal3 ComputeInstanceTemplate as an Ansible role in
  `osac.templates`.
- Support RHEL (image-mode/bootc) and RHCOS images via Metal3.
- Integrate nvidia.nmx for NVLink partition management on NVL72.
- Integrate nvidia.ufm for InfiniBand PKEY tenant isolation.
- Follow the inventory/provisioning separation pattern from EP 3.
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

The template role `osac.templates.metal3_bm` implements three
entry points: `install.yaml`, `delete.yaml`, and `status.yaml`.

### Provisioning workflow (install.yaml)

The role receives pre-selected hosts from `osac.service.select_hosts`
(per the inventory/provisioning separation EP).

```
Input:
  compute_instance_hosts:     # pre-selected from AAP inventory
  compute_instance_image:     # resolved Image (sourceRef, bootMethod)
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
4. Create NVLink partition (if multi-tenant GPU)
   │ Condition: class has gpus.count > 0 AND
   │            host shares NVLink domain with other tenants
   │ Action: nvidia.nmx.partition (state: present)
   │         - name: "tenant-{{ tenant_id }}"
   │         - members: GPU UUIDs from host_vars
   │
5. Create InfiniBand partition (if IB fabric)
   │ Condition: class has infiniband interfaces
   │ Action: nvidia.ufm.pkey (state: present)
   │         - pkey: computed from tenant_id
   │         - guids: from host_vars
   │
6. Configure networking
   │ Action: depends on NetworkClass of the referenced subnet
   │ - dpf-ovn-vpc: DPUService CRD for OVN VPC (via kubernetes.core)
   │ - udn-net: UDN CR (via kubernetes.core)
   │
7. Report success
   │ Set compute_instance_ip_address from BMH status
   │ Set compute_instance_state: "RUNNING"
```

### Deprovisioning workflow (delete.yaml)

```
Steps:

1. Remove NVLink partition
   │ nvidia.nmx.partition (state: absent)
   │
2. Remove InfiniBand partition
   │ nvidia.ufm.pkey (state: absent)
   │
3. Clean up networking
   │ Remove DPUService or UDN CRs
   │
4. Deprovision BareMetalHost
   │ Set BMH spec.online: false
   │ Wait for BMH → deprovisioning → available
   │
5. Update inventory
   │ osac.service.update_host_state (state: available, tenant: none)
```

### AAP execution environment

The Metal3 backend requires these collections:

| Collection | Purpose |
|---|---|
| `kubernetes.core` | BareMetalHost CRD management |
| `nvidia.nmx` | NVLink partition management |
| `nvidia.ufm` | InfiniBand PKEY management |
| `netbox.netbox` | Inventory (if NetBox is the inventory source) |

These should NOT be added to a monolithic shared execution
environment. Instead, the Metal3 backend should ship its own EE
image that extends the OSAC core EE:

```
EE: osac-core (base — shared by all backends)
  ├── kubernetes.core, community.general, ansible.utils
  ├── ansible.controller, ansible.eda
  └── osac.service, osac.templates, osac.workflows

EE: osac-metal3-gpu (this backend)
  ├── FROM osac-core
  ├── nvidia.ufm, nvidia.nmx
  └── netbox.netbox (if NetBox inventory)
```

Each `ComputeInstanceTemplate` specifies which EE its AAP job
template requires. The osac-operator passes this to AAP when
launching the job. NCP operators install only the EEs they need —
a VM-only deployment never pulls nvidia.ufm.

This pluggable EE model applies to all backends (ESI, NICo,
KubeVirt). The `osac-aap-ee` repo should evolve from a single
monolithic EE definition to a set of layered Containerfiles
(core + per-backend).

### Inventory requirements

Hosts provisioned via Metal3 must have these host_vars in AAP
inventory (per the inventory/provisioning separation EP):

| host_var | Source | Required |
|---|---|---|
| `hostclass` | Inventory plugin | Yes |
| `state` | Inventory plugin | Yes |
| `site` | Inventory plugin | Yes |
| `bmc_address` | Inventory plugin | Yes |
| `bmc_credentials_secret` | Inventory plugin | Yes |
| `boot_mac_address` | Inventory plugin | Yes |
| `nvlink_domain` | Inventory plugin | For GPU hosts |
| `gpu_uuids` | Inventory plugin | For NVLink partitioning |
| `ib_guids` | Inventory plugin | For IB partitioning |
| `rack` | Inventory plugin | For placement |
| `root_device` | Inventory plugin | Optional (default: /dev/nvme0n1) |

### ComputeInstanceClass examples

**Bare-metal GPU (inference):**

```yaml
id: "gpu-b200-4"
title: "4x B200 GPU Tray"
backend: "baremetal"
capabilities:
  cores: 96
  memoryGiB: 480
  gpus:
    count: 4
    model: "B200"
  storage:
    bootDiskGiB: 960
  networking:
    interfaces:
      - type: "ethernet"
        speed: "400Gbps"
      - type: "infiniband"
        speed: "400Gbps"
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
backend: "baremetal"
capabilities:
  cores: 64
  memoryGiB: 256
  storage:
    bootDiskGiB: 480
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

*Mitigation:* BareMetalHost resources can be created by the
inventory system (NetBox sync job or manual registration). This
is a day-0 setup task, not a per-tenant operation.

#### NVLink/IB partition management requires NMX-M and UFM

These are NVIDIA-specific services that must be deployed for GPU
multi-tenancy.

*Mitigation:* NVLink and IB partition steps are conditional — they
only run when the ComputeInstanceClass includes GPU or IB
capabilities. Non-GPU deployments work without NMX-M or UFM.

### Drawbacks

- Metal3/Ironic adds infrastructure to the management cluster.
- BareMetalHost pre-registration is a manual step (or requires
  inventory sync automation).

## Alternatives

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
handles VPC/NVLink/IB.

*Why this is possible but not required:* The unified compute model
supports multiple backends. A provider can deploy both a Metal3
template and a NICo template for the same ComputeInstanceClass.
This EP focuses on the Metal3-only path; hybrid deployment is a
provider configuration choice, not an architectural constraint.

## Test Plan

TBD — will cover:
- BareMetalHost creation and provisioning lifecycle
- Image boot method handling (Ignition, cloud-init)
- NVLink partition creation and deletion (with nvidia.nmx)
- IB PKEY creation and deletion (with nvidia.ufm)
- ComputeInstanceGroup scaling (add/remove bare-metal hosts)
- Deprovisioning and inventory state update

## Graduation Criteria

TBD
