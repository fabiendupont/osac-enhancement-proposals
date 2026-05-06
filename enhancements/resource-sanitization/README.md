---
title: resource-sanitization
authors:
  - Fabien Dupont
creation-date: 2026-04-29
last-updated: 2026-04-29
tracking-link:
  - TBD
see-also:
  - enhancements/osac-addon/README.md
  - enhancements/unified-compute-model/README.md
  - enhancements/metal3-compute-backend/README.md
  - enhancements/inventory-provisioning-separation/README.md
  - enhancements/tenant-specific-storageclasses/README.md
  - enhancements/tenant-storage-tiers/README.md
replaces:
superseded-by:
---

# Resource Sanitization

## Summary

Define a standard `reclaimPolicy` semantic for OSAC resources that
carry tenant data or hardware state. When a resource is released
(deleted by a tenant, scaled down from a group, or reclaimed after
tenant offboarding), the provider-configured policy determines what
happens to the underlying infrastructure.

This proposal covers:

- **ComputeInstance (bare metal)**: boot a sanitization image that
  performs drive erasure, GPU memory wipe, TPM/BIOS reset, and
  optional firmware attestation before returning hardware to the
  available pool.
- **ComputeInstance (virtual)**: backing volume erasure, memory
  page zeroing, GPU state cleanup after VFIO passthrough.
- **Storage volumes**: alignment with Kubernetes PersistentVolume
  `reclaimPolicy` semantics, with OSAC-level policy enforcement.

Network resources (VirtualNetwork, Subnet, SecurityGroup) are
excluded — they are logical constructs with no data residue. Their
deletion is idempotent.

## Motivation

### Current state

OSAC has no sanitization semantics for any resource type:

- **Bare-metal compute**: When a bare-metal ComputeInstance is
  deleted, the AAP deprovision workflow (`delete.yaml`) sets the
  BareMetalHost to `spec.online: false` and waits for
  deprovisioning. It does not erase drives, wipe GPU memory, or
  reset TPM/BIOS. The machine returns to the available pool with
  the previous tenant's data intact.

- **Virtual compute**: When a VM ComputeInstance is deleted,
  KubeVirt destroys the VirtualMachine and releases QEMU
  resources. However:
  - Backing PVCs may be retained depending on StorageClass
    `reclaimPolicy`, leaving tenant data on storage blocks.
  - Hugepages released by QEMU may not be zeroed before
    re-allocation to another tenant's VM on the same node.
  - GPU memory (framebuffer, SRAM) retains data after a VFIO
    passthrough device is detached — neither KubeVirt nor the
    GPU Operator wipes GPU state between VM assignments.

- **Storage**: The tenant-specific-storageclasses EP and
  tenant-storage-tiers EP defer entirely to Kubernetes
  `StorageClass.reclaimPolicy` (Delete, Retain, Recycle). OSAC
  does not enforce or validate the reclaim policy. A provider
  could accidentally configure `Retain` on a shared StorageClass,
  leaving tenant PVs accessible to the next tenant.

### Problems

1. **Data leakage between tenants (bare metal).** A machine
   returned to the pool retains the previous tenant's data on
   local NVMe drives, GPU memory (SRAM), and TPM state.

2. **Data leakage between tenants (VM).** A deleted VM's backing
   volume blocks may be re-allocated to another tenant without
   zeroing. GPU framebuffer retains data after VFIO detach.
   Hugepages may be re-assigned without clearing.

3. **No standard for what "clean" means.** Different deployment
   scenarios require different sanitization levels. A development
   environment may accept a quick wipe; a regulated environment
   may require cryptographic erasure with attestation.

4. **Storage reclaim policy is unmanaged.** OSAC relies on the
   Kubernetes StorageClass `reclaimPolicy` but does not validate
   that the policy meets the provider's security requirements.

5. **No sanitization audit trail.** There is no record of what
   was done, when, and whether it succeeded. Compliance
   frameworks (SOC 2, NVIDIA SEC21) require evidence.

### How public clouds and NCP requirements address this

- **AWS**: EBS volumes are zeroed on release. Dedicated Hosts
  support scrubbing between allocations. Nitro hypervisor zeroes
  memory on instance termination.
- **Azure**: SSDs are cryptographically erased. Confidential VMs
  use per-VM encryption keys destroyed on deallocation.
- **NVIDIA v2.2 (SEC21)**: "Cryptographic erase of drives,
  SRAM/GPU memory wipe, TPM reset, BIOS reset between tenants."
- **NVIDIA v2.2 (STG02)**: "Cryptographic erase between tenants
  with firmware attestation. Optional skip flag for non-tenant-
  impacting break/fix."

### User Stories

- As a **cloud provider**, I want to define a sanitization policy
  per ComputeInstanceClass, so that resources are cleaned to a
  level appropriate for my security requirements before being
  reused.

- As a **cloud provider**, I want to skip sanitization for
  non-tenant-impacting operations (e.g., break/fix where the
  machine stays with the same tenant).

- As a **cloud provider**, I want an audit trail of sanitization
  actions for compliance evidence.

- As a **cloud provider**, I want to enforce a minimum storage
  reclaim policy across all tenant StorageClasses.

### Goals

- Define a `reclaimPolicy` semantic that applies uniformly across
  OSAC resource types.
- Implement sanitization for both bare-metal and VM
  ComputeInstances.
- Enforce storage reclaim policy alignment between OSAC and
  Kubernetes StorageClasses.
- Provide sanitization status and audit trail.

### Non-Goals

- Implementing the sanitization image itself — the provider
  builds and maintains a sanitization OS image that performs the
  actual erasure, wipe, and reset operations on boot. OSAC
  defines the policy and orchestration (boot the image, wait,
  check result), not the mechanism.
- Network resource sanitization — VirtualNetwork, Subnet, and
  SecurityGroup are logical resources with no data residue.

## Proposal

### Reclaim policy

A `reclaimPolicy` field is added to ComputeInstanceClass and to
the OSAC storage configuration. The policy determines what happens
to the underlying resource when a ComputeInstance is deleted or a
storage volume is released.

#### Bare-metal ComputeInstance

| Policy | Behavior |
|--------|----------|
| `Delete` | Boot sanitization image. Image performs: cryptographic erase of local drives, GPU memory wipe, TPM clear, BIOS reset to known-good state. On success, machine returns to pool as `available`. |
| `Retain` | Machine stays in `allocated` state. No sanitization. Provider must manually inspect and release. For forensics or compliance holds. |
| `Sanitize` | Same as `Delete` plus firmware attestation: after sanitization, the image verifies TPM PCR values and drive firmware against known-good baselines. Reports pass/fail via callback. If attestation fails, machine moves to `error` state. |

#### Virtual ComputeInstance

| Policy | Behavior |
|--------|----------|
| `Delete` | Delete VM. Delete all backing PVCs (override StorageClass `Retain` if needed). If GPU passthrough was used, trigger GPU state cleanup on the host node. Verify hugepage zeroing configuration. |
| `Retain` | Delete VM but retain backing PVCs for inspection. Tag PVCs with tenant and deletion timestamp. |
| `Sanitize` | Same as `Delete` plus: block-level zeroing of backing volume before PV release, explicit hugepage zeroing verification, GPU diagnostics to confirm clean state. |

#### Storage volumes

| Policy | Behavior |
|--------|----------|
| `Delete` | Delete PV and underlying storage. Standard Kubernetes `Delete` behavior. |
| `Retain` | Retain PV. Standard Kubernetes `Retain` behavior. |

The default policy is `Delete` if not specified.

### Data residue vectors by hardware type

| Vector | Bare metal | VM (KubeVirt) | Sanitization action |
|--------|-----------|---------------|-------------------|
| Local NVMe/SSD | Tenant had direct access | Backing PVCs on shared storage | BM: sanitization image runs `nvme format`. VM: delete PVC, verify backend zeroes blocks |
| GPU memory (SRAM, framebuffer) | Direct access | VFIO passthrough | BM: sanitization image runs DCGM wipe. VM: GPU reset on host via DaemonSet |
| System memory | Direct access | Hugepages (2M/1G) | BM: BIOS reset clears. VM: verify kernel zeroes hugepages on free |
| TPM state | Physical TPM | vTPM (PVC-backed) | BM: sanitization image runs `tpm2_clear`. VM: delete vTPM PVC |
| BIOS/firmware | Tenant could modify (root) | Not applicable | BM: sanitization image resets via Redfish |
| Network config | Switch ACLs, PKEY membership | OVN flows | Handled by NetworkClass controllers on deletion, not by sanitization |

### ComputeInstanceClass with reclaimPolicy

```yaml
id: "gpu-b200-4"
title: "4x B200 GPU Tray"
hardwareType: "baremetal"
reclaimPolicy: "Sanitize"
sanitizationImage: "sanitize-rhel-gpu"
capabilities:
  coresFixed: 96
  memoryGibFixed: 480
  gpus:
    count: 4
    model: "B200"
```

```yaml
id: "gpu-a100-1-vm"
title: "1x A100 GPU (virtual)"
hardwareType: "virtual"
reclaimPolicy: "Delete"
capabilities:
  coresMin: 4
  coresMax: 32
  gpus:
    count: 1
    model: "A100"
```

GPU classes in regulated environments use `Sanitize`. Development
and VM classes typically use `Delete`. The provider sets the
policy per class — tenants cannot override it.

The `sanitizationImage` field (bare-metal classes only) references
an Image resource (per the image-and-sshkey-resources EP) that
contains the sanitization OS. This image is provider-built and
includes the tools needed for the sanitization level (nvme-cli,
DCGM, tpm2-tools, attestation agent). The image runs autonomously
on boot and reports results via a callback URL or status file.

### Skip flag for break/fix

When a machine requires maintenance but stays with the same
tenant, the break/fix workflow can set `skipSanitization: true`.
This bypasses sanitization and returns the machine to `available`
immediately. Only available to provider-level operations.

Aligns with NVIDIA v2.2 STG02: "Optional skip flag for
non-tenant-impacting break/fix."

### Sanitization workflows

#### Bare-metal sanitization

The bare-metal sanitization workflow is simple: boot a
sanitization image and wait for it to finish. The image does all
the heavy lifting.

```
ComputeInstance deletion
  │
  ▼
Phase 1: Deprovision (template role delete.yaml)
  │  BareMetalHost spec.online: false
  │  Wait for BMH → deprovisioning → available
  │
  ▼
Phase 2: Sanitize (osac.service.sanitize_compute)
  │  Read reclaimPolicy from ComputeInstanceClass
  │
  ├─ Delete / Sanitize:
  │    1. Set BMH spec.image to sanitizationImage
  │    2. Set BMH spec.online: true (PXE boot into
  │       sanitization image)
  │    3. Wait for sanitization image to complete
  │       (callback to OSAC webhook or poll BMH status)
  │    4. If Sanitize: check attestation result from
  │       callback payload
  │    5. Set BMH spec.online: false (power off)
  │    6. Clear BMH spec.image
  │    7. Success: record event, set host state: available
  │    8. Failure: record event, set host state: error
  │
  ├─ Retain:
  │    1. Record retention event
  │    2. Keep machine in allocated state
  │
  └─ skipSanitization:
       1. Record skip event with justification
       2. Return machine to pool immediately
```

The sanitization image is responsible for:
- Cryptographic erase of all local NVMe/SSD
- GPU memory wipe via DCGM
- TPM clear via tpm2-tools
- BIOS reset via Redfish (local BMC access)
- Attestation verification (Sanitize policy only)
- Reporting results via HTTP callback to OSAC

The AAP role only orchestrates the boot cycle and interprets the
result. It does not SSH into the node or run remote commands.

Provider API calls (e.g., Redfish BIOS reset from out-of-band if
the sanitization image cannot access the local BMC) can be added
as an optional pre-boot or post-boot step in the role.

#### VM sanitization

```
ComputeInstance deletion
  │
  ▼
Phase 1: Deprovision (template role delete.yaml)
  │  Delete VirtualMachine CR
  │  Wait for VM termination
  │
  ▼
Phase 2: Sanitize (osac.service.sanitize_vm)
  │  Read reclaimPolicy from ComputeInstanceClass
  │
  ├─ Delete:
  │    1. Delete all backing PVCs (force-delete if
  │       StorageClass has Retain)
  │    2. If GPU passthrough: trigger GPU reset on host
  │       node (nvidia-smi --gpu-reset via DaemonSet)
  │    3. Record sanitization event
  │
  ├─ Sanitize:
  │    1. Zero backing volume blocks before PV release
  │    2. GPU reset + dcgmi diagnostic to confirm clean
  │    3. Verify hugepage pool integrity on host
  │    4. Record sanitization event with verification
  │
  └─ Retain:
       1. Tag backing PVCs with tenant + timestamp
       2. Record retention event
```

### Sanitization status

ComputeInstance status includes sanitization state:

```yaml
status:
  state: "DEPROVISIONING"
  sanitization:
    state: "InProgress"
    startedAt: "2026-04-29T14:00:00Z"
    completedAt: null
    policy: "Sanitize"
```

Sanitization states: `Pending`, `InProgress`, `Completed`,
`Failed`, `Skipped`.

Detailed step-by-step progress (drive erase, GPU wipe, etc.) is
internal to the sanitization image and reported in the callback
payload. OSAC stores the summary result, not individual steps —
the sanitization image is a black box from OSAC's perspective.

Once sanitization completes, the final status is preserved as an
event in the fulfillment-service event stream (MGMT-23891) for
audit trail purposes.

### Storage reclaim policy enforcement

OSAC validates that the Kubernetes StorageClass `reclaimPolicy`
meets a minimum standard set by the provider:

1. The provider configures `minimumStorageReclaimPolicy: Delete`
   in the OSAC configuration.

2. When the tenant-specific-storageclasses controller resolves a
   StorageClass for a tenant, it checks the policy against the
   minimum.

3. If the StorageClass policy is weaker than the minimum (e.g.,
   `Retain` when the minimum is `Delete`), the controller rejects
   the StorageClass and logs a warning.

### API Extensions

#### Modified resources

| Resource | Change |
|---|---|
| ComputeInstanceClass | Add `reclaimPolicy` (string: `Delete`, `Retain`, `Sanitize`, default: `Delete`) and `sanitizationImage` (string, optional, bare-metal only). |
| ComputeInstance | Add `status.sanitization` (state, startedAt, completedAt, policy). |

#### New AAP roles

| Role | Collection | Description |
|---|---|---|
| `sanitize_compute` | `osac.service` | Bare-metal: boot sanitization image, wait for completion, check result, update state. |
| `sanitize_vm` | `osac.service` | VM: PVC cleanup, GPU reset, hugepage verification. |

### Implementation Details/Notes/Constraints

#### Database changes

- `compute_instance_classes` — `data` jsonb includes
  `reclaimPolicy` and `sanitizationImage`
- `compute_instances` — `data` jsonb includes
  `status.sanitization`

#### Sanitization image

The provider builds and maintains a sanitization OS image. A
reference image based on Image-mode RHEL with DCGM, nvme-cli,
and tpm2-tools can be provided as a starting point. The image
must:

- Run autonomously on boot (no external orchestration)
- Perform all sanitization actions for the hardware platform
- Report results via HTTP POST to a callback URL (passed as
  kernel parameter or cloud-init metadata)
- Power off or signal completion when done

This is the same pattern used by NICo's `sanitize-gpu-node`
Temporal workflow — the workflow boots an image, the image does
the work, the workflow polls for completion.

### Risks and Mitigations

#### Sanitization adds deprovisioning latency

Full bare-metal sanitization can take 10-30 minutes depending
on drive size and GPU count.

*Mitigation:* The `reclaimPolicy` is per-class. Dev classes use
`Delete` (no attestation). The `skipSanitization` flag handles
break/fix.

#### GPU reset on shared VM nodes

Running GPU reset on a node with other tenants' VMs using the
same GPU pool could impact those workloads.

*Mitigation:* GPU reset targets the specific PCI device that was
assigned to the deleted VM via VFIO, not all GPUs on the node.

#### Sanitization image failure

If the sanitization image crashes or hangs, the machine is stuck
in sanitization state.

*Mitigation:* The AAP role has a configurable timeout (default
30 minutes). On timeout, the role powers off the machine and
sets state to `error`. The provider investigates manually.

### Drawbacks

- Adds deprovisioning latency for bare-metal instances.
- Requires the provider to build and maintain a sanitization
  image per hardware platform.
- VM GPU reset requires node-level privileges.

## Alternatives (Not Implemented)

### Delegate sanitization entirely to the backend

Let Metal3, NICo, or KubeVirt handle sanitization.

*Why not:* Different backends have different capabilities. A
standard policy at the OSAC level ensures consistent behavior
regardless of backend.

### Orchestrate sanitization steps from AAP

Run individual sanitization commands (nvme format, tpm2_clear,
etc.) remotely from AAP via SSH or Redfish.

*Why not:* Requires network access to a machine that should be
untrusted (it just had a tenant with root access). Booting a
sanitization image is simpler and more secure — the image runs
in a known-good environment, not on the tenant's OS.

### Per-tenant reclaim policy

Allow tenants to choose their own reclaim policy.

*Why not:* Sanitization is a provider-side security concern.
Tenants should not be able to weaken the provider's policy.

## Open Questions

1. **Sanitization image callback.** Should the callback be an
   HTTP POST to an OSAC webhook (requires network access from
   the sanitization image), or should the AAP role poll BMH
   status / Redfish for completion? Polling is simpler but less
   informative; callbacks provide detailed results.

2. **Attestation baseline storage.** Where should known-good TPM
   PCR values and firmware checksums be stored? Options: baked
   into the sanitization image, AAP inventory host_vars, or an
   external attestation service.

3. **VM hugepage zeroing.** Should OSAC verify kernel hugepage
   zeroing configuration at node onboarding time (proactive) or
   at VM deletion time (reactive)?

4. **Shared GPU nodes.** When multiple tenants share GPUs via MIG
   or time-slicing (not VFIO), should OSAC verify MIG partition
   cleanup on VM deletion, or trust the GPU Operator?

## Test Plan

TBD — will cover:
- Bare-metal sanitization: boot image, wait, check result
- VM sanitization: PVC cleanup, GPU reset
- Each policy (Delete, Retain, Sanitize) for both hardware types
- Skip flag behavior
- Attestation pass and fail scenarios
- Storage reclaim policy validation
- Timeout handling
- Sanitization status and event trail

## Graduation Criteria

TBD

## Upgrade / Downgrade Strategy

**Upgrade:** Migration adds `reclaimPolicy` to existing
ComputeInstanceClass records with default `Delete`. Existing
ComputeInstances in progress are not affected.

**Downgrade:** The `reclaimPolicy` field is preserved in `data`
jsonb and ignored by older code.

## Version Skew Strategy

`reclaimPolicy` is a fulfillment-service field. The sanitization
roles run in AAP. The fulfillment-service must be updated before
the AAP EE. If the EE is updated first, the roles default to
`Delete` when `reclaimPolicy` is absent.

## Support Procedures

- **ComputeInstance stuck in DEPROVISIONING:** Check if the
  sanitization image booted (BMH status). Check for timeout.
  Common causes: PXE failure, sanitization image crash, BMC
  unreachable.
- **Machine in error state after sanitization:** Attestation
  failed or sanitization timed out. Check the sanitization
  event in the event stream. Resolve hardware issue and manually
  transition to `available`.
- **VM PVCs retained after deletion:** Check if
  ComputeInstanceClass has `reclaimPolicy: Retain`. If
  unintentional, delete PVCs manually and update the class.

## Infrastructure Needed

- Sanitization boot image per hardware platform (provider-built,
  based on Image-mode RHEL reference)
- For VM sanitization: privileged DaemonSet or equivalent for GPU
  reset on KubeVirt nodes with VFIO passthrough
