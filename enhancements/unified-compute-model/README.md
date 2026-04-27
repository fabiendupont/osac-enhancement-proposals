---
title: unified-compute-model
authors:
  - Fabien Dupont
creation-date: 2026-04-17
last-updated: 2026-04-17
tracking-link:
  - TBD
see-also:
  - enhancements/vmaas/README.md
  - enhancements/vm-api-fields/README.md
  - enhancements/bare-metal-fulfillment/README.md
  - enhancements/carbide-integration/README.md
  - enhancements/image-and-sshkey-resources/README.md
  - enhancements/metal3-compute-backend/README.md
  - enhancements/inventory-provisioning-separation/README.md
  - enhancements/catalog-items/README.md
  - https://github.com/osac-project/enhancement-proposals/pull/20 (Region/AZ)
  - https://github.com/osac-project/enhancement-proposals/pull/35 (VM Image Management)
replaces:
  - enhancements/bare-metal-fulfillment/README.md (HostPool, Host, HostClass concepts)
superseded-by:
---

# Unified Compute Model

## Summary

Introduce a unified compute resource model that serves both virtual
machine and bare-metal workloads through the same tenant-facing API.
The model consists of four resource types:

- **ComputeInstanceClass**: A provider-defined SKU that describes
  what the tenant can order (capabilities, pricing).
- **ComputeInstanceTemplate**: A provider-defined Ansible role that
  describes how to provision infrastructure for a given backend
  (Metal3, KubeVirt, NICo, ESI).
- **ComputeInstance**: A tenant-created resource representing a
  running compute unit (VM or bare-metal machine).
- **ComputeInstanceGroup**: A tenant-created resource representing
  a scaled set of ComputeInstances with placement policy.

This replaces the current split between `ComputeInstance` /
`ComputeInstanceTemplate` (VM-only) and the removed `HostPool` /
`Host` / `HostClass` (bare-metal-only) with a single set of
resources that work for both.

### Related Proposals

This EP is part of a set of four related proposals. This proposal
defines the resource model and API. The
[image-and-sshkey-resources](../image-and-sshkey-resources/README.md)
EP extracts Image and SSHKey from inline fields into first-class
catalog resources consumed by ComputeInstance. The
[metal3-compute-backend](../metal3-compute-backend/README.md) EP
provides the first bare-metal ComputeInstanceTemplate
implementation using Metal3/Ironic with NVLink and InfiniBand
partition support. The
[inventory-provisioning-separation](../inventory-provisioning-separation/README.md)
EP establishes the architectural boundary between host discovery
and provisioning in AAP backends, which all template roles
(including Metal3) follow. These proposals can be reviewed
independently but are designed to be implemented together.

## Motivation

### Current state

OSAC has two separate paths for compute provisioning:

- **Virtual machines**: `ComputeInstance` and
  `ComputeInstanceTemplate` (implemented, running in production at
  MOC)
- **Bare metal**: `HostPool`, `Host`, and `HostClass` (designed in
  the bare-metal-fulfillment EP, implemented, then removed in April
  2026 pending redesign)

These two paths have separate APIs, separate resource types, separate
controllers, and separate AAP playbooks — despite providing
fundamentally similar capabilities to tenants: compute with an OS,
network attachment, SSH access, and lifecycle management.

### Problems

1. **Duplicate API surface.** Tenants must learn two different APIs
   for compute. The VM path has `ComputeInstance` with fields like
   `image`, `cores`, `memoryGiB`, `sshKey`. The bare-metal path
   (when it existed) had `HostPool` with `hostSets`, `hostClass`,
   and network attachments. Same concepts, different names and
   structures.

2. **No SKU catalog.** `ComputeInstanceTemplate` serves dual duty
   as both the tenant-facing catalog entry (title, description,
   parameter definitions) and the implementation recipe (Ansible
   role). Tenants are exposed to implementation details like
   template parameter names (`vm_cpu_cores` vs `cores`).
   The vm-api-fields EP partially addressed this by adding
   standardized fields, but the template/catalog split remains
   missing. The
   [Catalog Items EP](../catalog-items/README.md)
   addresses this for the presentation layer; this proposal
   addresses it at the resource model level with
   ComputeInstanceClass.

3. **Bare metal is being redesigned.** The original HostPool/Host
   implementation was removed. A
   [redesigned proposal](https://github.com/osac-project/enhancement-proposals/pull/31)
   (BareMetalPool/HostLease) is under active review, but it
   maintains a separate resource hierarchy for bare metal rather
   than unifying with the existing ComputeInstance model.

4. **Backend lock-in at the API level.** The current
   `ComputeInstanceTemplate` embeds backend-specific parameter
   definitions. Switching from ESI to Metal3 requires changing the
   templates tenants see — not just the infrastructure
   implementation.

### How public clouds solve this

AWS, GCP, and Azure all use a single Instance API where the instance
type (class) determines whether the tenant gets bare metal or
virtual:

- AWS: same `RunInstances` API for `p5.48xlarge` (VM) and
  `p5.48xlarge.metal` (bare metal)
- GCP: same `instances.create` for all machine types
- Azure: same VMs API for all sizes

The tenant picks a class, provides an image and SSH key, specifies
networking. The platform handles the rest.

### User Stories

- As a **tenant**, I want to provision compute resources (VM or bare
  metal) through a single API, selecting from a catalog of classes
  defined by my provider.

- As a **tenant**, I want to scale a group of identical compute
  instances up or down with a single operation.

- As a **cloud provider**, I want to define compute classes (SKUs)
  that describe what tenants can order, independently from how those
  classes are provisioned.

- As a **cloud provider**, I want to add a new bare-metal backend
  (e.g., Metal3, NICo) without changing the tenant-facing API.

- As an **infrastructure provider**, I want to implement a
  provisioning backend as an Ansible role, without needing to
  understand OSAC's API or database.

### Goals

- Unify VM and bare-metal compute under a single set of resource
  types.
- Separate the tenant-facing catalog (ComputeInstanceClass) from
  the implementation recipe (ComputeInstanceTemplate).
- Re-enable bare-metal provisioning in OSAC through the unified
  model.
- Support multiple provisioning backends (Metal3, KubeVirt, NICo,
  ESI) behind the same tenant API.
- Introduce ComputeInstanceGroup for scaled sets with placement
  policies.

### Non-Goals

- Implementing a specific bare-metal backend (Metal3, NICo) — that
  is covered by separate EPs per backend.
- Changes to networking resources (VirtualNetwork, Subnet,
  SecurityGroup, NetworkClass) — those are unchanged.
- Cluster fulfillment changes — ClusterTemplate/ClusterOrder/Cluster
  remain separate. A future EP may align cluster node sets with
  ComputeInstanceClass.
- Image and SSHKey as first-class resources — proposed in a
  separate EP.

## Proposal

### Resource types

#### ComputeInstanceClass (provider-defined, tenant-visible)

A ComputeInstanceClass represents a compute offering in the
provider's service catalog — what the tenant can order. It
describes capabilities (hardware profile), references one or more
ComputeInstanceTemplates that can fulfill it, and optionally
carries pricing information.

```yaml
id: "gpu-b200-4"
title: "4x B200 GPU Tray"
description: "Dedicated NVL72 compute tray with 4 NVIDIA B200 GPUs, 96 ARM cores, 480GB RAM"
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

```yaml
id: "gpu-a100-1-vm"
title: "1x A100 GPU (virtual)"
description: "Virtual machine with 1 NVIDIA A100 GPU, configurable cores and memory"
hardwareType: "virtual"
capabilities:
  coresMin: 4
  coresMax: 32
  memoryGibMin: 16
  memoryGibMax: 128
  gpus:
    count: 1
    model: "A100"
  storage:
    bootDiskGibMin: 50
    bootDiskGibMax: 500
templates:
  - name: "kubevirt-a100-paris"
    site: "paris"
```

Capabilities use a fixed/min/max pattern via separate fields:
`coresFixed` (exact value) or `coresMin`/`coresMax` (range).
For bare metal, capabilities are fixed (the hardware is what it is).
For VMs, capabilities can specify ranges that the tenant selects
within. Fixed and min/max are mutually exclusive per dimension.

The `hardwareType` field (`baremetal` or `virtual`) describes the
physical nature of the compute resource. This is distinct from the
ComputeInstanceTemplate's `backend` field, which identifies the
provisioning provider (Metal3, KubeVirt, NICo, ESI). A bare-metal
class can be fulfilled by Metal3 or NICo templates; a virtual
class by KubeVirt or other hypervisor templates. Tenants may or
may not see this field — providers can choose to expose or hide
it.

The `templates` list maps the class to one or more
ComputeInstanceTemplates. Multiple templates enable multi-site
deployments where the same class is available at different
locations.

#### ComputeInstanceTemplate (provider-defined, internal)

A ComputeInstanceTemplate is an Ansible role that implements the
provisioning logic for a specific backend and site. It follows the
existing OSAC pattern where Templates are Ansible roles.

ComputeInstanceTemplates are not visible to tenants. They are
registered in the fulfillment-service database and referenced by
ComputeInstanceClasses.

```yaml
id: "metal3-b200-paris"
backend: "metal3"
site: "paris"
role: "metal3_bm_rhel"
roleCollection: "osac.templates"
```

```yaml
id: "kubevirt-a100-paris"
backend: "kubevirt"
site: "paris"
role: "ocp_virt_vm"
roleCollection: "osac.templates"
```

```yaml
id: "nico-b200-paris"
backend: "nico"
site: "paris"
role: "nico_bm"
roleCollection: "osac.templates"
```

The template role receives standardized variables from the
ComputeInstance spec (image, sshKeys, subnet, securityGroups,
userData, cores, memoryGiB) plus class-level context (capabilities,
site). The role is responsible for translating these into
backend-specific operations.

Template roles must implement three entry points:
- `install.yaml` — provision the compute instance
- `delete.yaml` — deprovision and clean up
- `status.yaml` — report current state (optional)

This aligns with the existing ClusterTemplate pattern.

#### ComputeInstance (tenant-created)

A ComputeInstance represents a running compute resource — either a
VM or a bare-metal machine. The tenant creates it by specifying a
ComputeInstanceClass and providing configuration.

```yaml
id: "my-gpu-node"
spec:
  computeInstanceClass: "gpu-b200-4"
  region: "eu-west"
  imageRef: "rhel-9.6-gpu"
  sshKeyRefs:
    - "my-workstation-key"
  subnet: "my-vpc-subnet"
  securityGroups:
    - "my-sg"
  userData: |
    #cloud-config
    hostname: my-gpu-node
  # VM-only fields (ignored for bare metal, validated against class ranges):
  cores: 16
  memoryGiB: 64
  bootDisk:
    sizeGiB: 100
  additionalDisks:
    - sizeGiB: 250
  runStrategy: "Always"
  # Server-populated (read-only):
  resolvedImage:
    sourceType: "registry"
    sourceRef: "quay.io/images/rhel:9.6-cuda12.8"
    bootMethod: "ignition"
    checksum: "sha256:abc123..."
  resolvedSshKeys:
    - "ssh-ed25519 AAAA..."
status:
  state: "RUNNING"
  ipAddress: "10.100.0.10"
  conditions: [...]
```

The existing ComputeInstance fields from the vmaas and vm-api-fields
EPs are preserved. The changes are:

- `template` field is deprecated (kept for backward compatibility
  with a migration path; see below). New instances use
  `computeInstanceClass`.
- `template_parameters` is deprecated — standardized fields replace
  freeform parameters. A per-deployment migration maps known
  parameter names (e.g., `vm_cpu_cores` -> `cores`).
- `region` is added (optional) — narrows template selection within
  the class. Region semantics are defined by the
  [Region/AZ EP](https://github.com/osac-project/enhancement-proposals/pull/20)
- `imageRef` is added as a reference to an Image resource (the
  inline `image` object field is deprecated; see the
  image-and-sshkey-resources EP)
- `sshKeyRefs` is added as a list of SSHKey resource references
  (the inline `sshKey` string field is deprecated)
- `resolvedImage` is populated by the server at creation time —
  a denormalized snapshot of the Image resource (sourceType,
  sourceRef, bootMethod, checksum) so the provisioning workflow
  has all data without API callbacks
- `resolvedSshKeys` is populated by the server at creation time —
  the actual public key strings resolved from the SSHKey resources

For bare-metal classes, VM-specific fields (`cores`, `memoryGiB`,
`bootDisk`, `runStrategy`) are either ignored or validated as fixed
values matching the class capabilities. The validation logic is:

- If the class has a fixed capability (e.g., `coresFixed: 96`),
  the instance field must match or be omitted
- If the class has a range capability (e.g., `coresMin: 4`,
  `coresMax: 32`), the instance field must fall within the range
- Fixed and min/max are mutually exclusive per dimension
- If the class capability is not specified, the instance field is
  passed through to the template role

#### ComputeInstanceGroup (tenant-created)

A ComputeInstanceGroup manages a set of identical ComputeInstances
with scaling and placement semantics. It replaces the removed
HostPool concept.

```yaml
id: "my-training-pool"
spec:
  computeInstanceClass: "gpu-b200-4"
  replicas: 8
  imageRef: "rhel-9.6-gpu"
  sshKeyRefs:
    - "my-workstation-key"
  subnet: "my-vpc-subnet"
  securityGroups:
    - "my-sg"
  userData: |
    #cloud-config
    hostname: training-node
  region: "eu-west"
  placementPolicy:
    strategy: "pack"
    affinityKey: "nvlink-domain"
status:
  state: "READY"
  readyReplicas: 8
  instances:
    - "my-training-pool-0"
    - "my-training-pool-1"
    - ...
  message: ""
```

The ComputeInstanceGroup controller creates and manages individual
ComputeInstance resources. Scaling up increases `replicas` — the
controller creates new ComputeInstances. Scaling down decreases
`replicas` — the controller selects instances for removal (last
created first, or by tenant selection) and deletes them.

The `placementPolicy` is optional and informs the AAP workflow
about placement preferences. The available strategies depend on
the backend:

- `pack` — fill racks/domains before spilling to the next
- `spread` — distribute across failure domains
- Backends that don't support placement ignore this field

### Workflow Description

#### ComputeInstance creation

1. Tenant creates a ComputeInstance via the fulfillment API,
   specifying a ComputeInstanceClass and configuration.

2. Fulfillment-service validates the request:
   - Class exists and is visible to the tenant
   - Field values are within class capability ranges
   - Referenced subnet and security groups exist

3. Fulfillment-service selects a ComputeInstanceTemplate:
   - Filters class templates by region (if specified)
   - Selects the first template with available capacity
   - Stores the selected template ID on the ComputeInstance record

4. Fulfillment-service creates a ComputeInstance record in
   PostgreSQL and a ComputeInstance CRD on the selected hub cluster.

5. osac-operator detects the CRD and triggers AAP with the selected
   template role.

6. AAP runs the template role (`install.yaml`), which provisions
   the compute resource using the backend (Metal3, KubeVirt, NICo,
   ESI).

7. Feedback controller syncs status from the CRD back to
   PostgreSQL.

#### ComputeInstanceGroup scaling

1. Tenant updates ComputeInstanceGroup replicas (e.g., 4 → 8).

2. Fulfillment-service updates the ComputeInstanceGroup record
   and CRD.

3. osac-operator compares desired replicas with current instance
   count.

4. For scale-up: controller creates new ComputeInstance CRDs
   (inheriting spec from the group).

5. For scale-down: controller selects instances for removal and
   deletes their CRDs, triggering deprovision workflows.

6. Each individual ComputeInstance follows the standard creation
   or deletion workflow.

### API Extensions

#### New resources

| Resource | API path | Methods |
|---|---|---|
| ComputeInstanceClass | `/api/fulfillment/v1/compute_instance_classes` | GET (list), GET (by id) |
| ComputeInstanceClass | `/api/fulfillment/v1/compute_instance_classes/{id}` | GET, POST, PATCH, DELETE (provider-only) |
| ComputeInstanceTemplate | (private API only) | CRUD (provider-only) |
| ComputeInstanceGroup | `/api/fulfillment/v1/compute_instance_groups` | GET (list), GET (by id) |
| ComputeInstanceGroup | `/api/fulfillment/v1/compute_instance_groups/{id}` | GET, POST, PATCH, DELETE |

#### Modified resources

| Resource | Change |
|---|---|
| ComputeInstance | Add `computeInstanceClass`, `region`, `imageRef`, `sshKeyRefs`, `resolvedImage`, `resolvedSshKeys` fields. Deprecate `template`, `template_parameters`, `image` (inline), `sshKey` (inline). |

#### Deprecated resources

| Resource | Replaced by | Migration |
|---|---|---|
| ComputeInstanceTemplate (current, parameter-definition style) | ComputeInstanceClass + ComputeInstanceTemplate (Ansible role) | Templates become classes; parameter definitions become class capabilities |
| HostPool (removed) | ComputeInstanceGroup | Same scaling semantics, unified with VM |
| Host (removed) | ComputeInstance with bare-metal class | Same lifecycle, unified API |
| HostClass (removed) | ComputeInstanceClass | Hardware profiles become classes |
| HostType (removed with HostPool/Host) | ComputeInstanceClass | Hardware type descriptors become classes |

### Implementation Details/Notes/Constraints

#### Database changes

New tables (following the existing generic schema from
`dao_tables.go`):

- `compute_instance_classes` — stores ComputeInstanceClass records
- `compute_instance_templates` — replaces current table content
  (Ansible role references instead of parameter definitions)
- `compute_instance_groups` — stores ComputeInstanceGroup records

Modified tables:

- `compute_instances` — `data` jsonb updated to include
  `computeInstanceClass` and `region` fields, remove `template`
  and `template_parameters`

No structural schema changes — all resource-specific data lives in
the `data jsonb` column.

#### CRD changes

New CRD:

- `ComputeInstanceGroup` (`osac.openshift.io/v1alpha1`) — watched
  by a new ComputeInstanceGroupController that manages child
  ComputeInstance CRDs

Modified CRD:

- `ComputeInstance` — add `computeInstanceClass` and `region`
  fields to spec

#### AAP changes

- Template role dispatch based on `ComputeInstanceTemplate.backend`
  instead of a hardcoded template name
- Existing template roles (`ocp_virt_vm`) adapted to receive
  standardized variables instead of freeform `template_parameters`
- Template selection is performed by the fulfillment-service (not
  AAP) based on region and class template list

#### Migration path

1. Existing `ComputeInstanceTemplate` records are migrated to
   `ComputeInstanceClass` records (title, description become class
   fields; parameter definitions become capability ranges).

2. New `ComputeInstanceTemplate` records are created for each
   backend-specific Ansible role.

3. Existing `ComputeInstance` records are updated to reference
   their class instead of their template.

4. The old `template` and `template_parameters` fields are
   preserved in the `data` jsonb for backward compatibility during
   the migration period.

### Risks and Mitigations

#### Breaking change for existing tenants

Deprecating `template` in favor of `computeInstanceClass` on
ComputeInstance requires a migration path.

*Mitigation:* Both fields are supported during a transition period.
If `template` is specified and `computeInstanceClass` is not, the
server validates the template exists and logs a deprecation warning.
A database migration (migration 32) creates ComputeInstanceClass
records from existing templates and backfills `computeInstanceClass`
on existing ComputeInstance records. The `template` and
`template_parameters` proto fields are preserved (not removed) to
avoid breaking wire compatibility with existing records.

#### ComputeInstanceGroup controller complexity

Managing a set of ComputeInstances with scaling, placement, and
failure handling adds controller complexity.

*Mitigation:* Start with simple first-fit scaling (no placement
policy). Add placement strategies incrementally in follow-up EPs.

#### Backend-specific semantics leaking into the unified model

Different backends have different capabilities (VMs support live
migration, bare metal doesn't; bare metal has fixed cores, VMs
have configurable cores).

*Mitigation:* The `hardwareType` field on ComputeInstanceClass
communicates the hardware model. Field validation adapts based
on whether capabilities are fixed or ranges. The API shape is the
same; the validation rules differ.

### Drawbacks

- Adds a level of indirection (Class → Template → Role) that
  increases conceptual complexity for providers setting up the
  catalog.
- Tenants who were using `template_parameters` for VM customization
  must switch to standardized fields.

## Alternatives (Not Implemented)

### Keep separate BM and VM resource types

Continue with `ComputeInstance` for VMs and design new BM-specific
resources (`HostPool`, `Host`) as the bare-metal-fulfillment EP
proposed.

*Why not:* Duplicates API surface, controller logic, and AAP
playbooks. Tenants must learn two APIs. Adding a new backend
(e.g., Metal3) requires implementing both BM resources and their
full lifecycle — the same work as implementing a ComputeInstance
backend, but with a separate code path.

### Add bare metal as a ComputeInstanceTemplate variant (no Class)

Keep the current `ComputeInstanceTemplate` model and add BM
templates alongside VM templates.

*Why not:* Templates expose implementation details to tenants
(backend-specific parameter names). No SKU/catalog layer for
providers to curate their offerings. No multi-site template
selection.

### Catalog Items as the presentation layer

The [Catalog Items EP](../catalog-items/README.md)
proposes `ClusterCatalogItem` and `ComputeInstanceCatalogItem` as
a presentation layer on top of templates. Catalog items control
which fields are editable by tenants, with defaults and JSON
Schema validation per field.

ComputeInstanceClass and Catalog Items address related but
distinct concerns. Catalog Items are a presentation/policy layer
(which fields can the tenant change?). ComputeInstanceClass is a
resource model layer (what hardware can the tenant order, and how
is it provisioned?). The two could coexist: a Catalog Item could
reference a ComputeInstanceClass and further constrain which
class fields are tenant-editable. This proposal does not depend
on or conflict with the Catalog Items EP.

### BareMetalPool / HostLease architecture (PR #31 v2)

The [updated bare-metal-fulfillment EP](https://github.com/osac-project/enhancement-proposals/pull/31)
proposes a redesigned bare-metal architecture with:

- **BareMetalPool** — a collection of hosts with profile-driven
  templates (pool-level and host-level Ansible roles)
- **HostLease** — an ephemeral lease on a host, reconciled by
  backend-specific operators
- **Three new operators** — BareMetalPool Operator (creates
  leases), Host Inventory Operator (assigns hosts from inventory),
  Host Management Operator (per HostClass, manages provisioning
  and power)

This approach keeps bare metal as a separate resource hierarchy
from ComputeInstance. We propose unification instead for these
reasons:

1. **Single tenant API.** Tenants use the same ComputeInstance
   resource for both VM and bare-metal workloads, matching the
   pattern used by AWS, GCP, and Azure. A separate BareMetalPool
   API doubles the surface tenants must learn.

2. **Consistent provisioning architecture.** Every provisioning
   path in OSAC today follows the same pattern: osac-operator
   watches CRDs and dispatches to AAP, where Ansible roles do the
   actual infrastructure work. This applies to clusters,
   ComputeInstances, VirtualNetworks, Subnets, SecurityGroups, and
   PublicIPPools. The unified model preserves this pattern. PR #31
   would introduce three new operators that directly reconcile
   bare-metal resources — the only operator-based provisioning
   path in OSAC, diverging from the established architecture.

3. **Multi-tenancy inherited.** ComputeInstance already has tenant
   scoping via organization annotations and OPA policies.
   BareMetalPool would need to implement these separately — an
   open concern raised in the PR #31 review.

4. **Backend swappability at the template level.** Switching from
   Metal3 to NICo means registering a new ComputeInstanceTemplate,
   not deploying a different Host Management Operator.

5. **Shared scaling semantics.** ComputeInstanceGroup handles both
   VM and bare-metal scaling with the same controller. BareMetalPool
   only covers bare metal.

Both approaches share important goals: backend pluggability,
template-driven provisioning, and separation of inventory from
provisioning. The architectural difference is whether bare metal
gets its own resource hierarchy (BareMetalPool → HostLease) or
is unified with compute (ComputeInstanceGroup → ComputeInstance).

## Open Questions [optional]

1. **Capabilities schema.** Should ComputeInstanceClass capabilities
   use a fixed JSON Schema, or should the structure be extensible
   per backend? A fixed schema simplifies validation but may not
   cover all hardware dimensions.

2. **ComputeInstanceGroup rolling updates.** When a group's image
   or class changes, should existing instances be updated in-place
   (rolling), or should the tenant create a new group? In-place
   updates add controller complexity.

3. **Capacity exhaustion.** When template selection fails because
   no site has available capacity, should the ComputeInstance remain
   in PENDING (retrying) or move to FAILED? Retry risks unbounded
   queue growth; fail requires tenant resubmission.

## Test Plan

TBD — will cover:
- ComputeInstance creation with bare-metal and virtual classes
- ComputeInstanceClass validation (fixed vs range capabilities)
- ComputeInstanceGroup scaling (up and down)
- Template selection (multi-site, capacity-based)
- Migration from old template model to new class model

## Graduation Criteria

TBD

## Upgrade / Downgrade Strategy

**Upgrade:** A database migration creates ComputeInstanceClass
records from existing ComputeInstanceTemplate records and
backfills the `computeInstanceClass` field on existing
ComputeInstance records. Both old (`template`) and new
(`computeInstanceClass`) fields are supported during the
transition period. No tenant action is required — existing
instances continue to work.

**Downgrade:** The `template` and `template_parameters` fields are
preserved in the `data` jsonb column and never removed. Downgrading
to pre-class code reads the old fields. ComputeInstanceClass,
ComputeInstanceTemplate (new-style), and ComputeInstanceGroup
records created after the upgrade would be inaccessible to the
older code and require manual cleanup.

## Version Skew Strategy

ComputeInstanceClass and ComputeInstanceTemplate are
fulfillment-service PostgreSQL resources with no hub cluster
representation. No version skew applies to these.

ComputeInstance and ComputeInstanceGroup CRDs require the
osac-operator to be updated before the fulfillment-service,
so that the operator recognizes the new CRD fields. The operator
handles both old-format (template-based) and new-format
(class-based) ComputeInstance CRDs during the transition.

## Support Procedures

- **ComputeInstance stuck in PENDING:** Check template selection —
  does the ComputeInstanceClass have templates for the requested
  region? Is there available capacity? Check the AAP job output
  for provisioning errors.
- **ComputeInstanceGroup readyReplicas mismatch:** Check individual
  ComputeInstance status within the group. The controller creates
  instances sequentially; a stuck instance blocks further scale-up.
- **Template selection failure:** Verify ComputeInstanceClass
  templates list, confirm the template exists in the database, and
  check region/site filtering.

## Infrastructure Needed [optional]

None beyond existing OSAC infrastructure.
