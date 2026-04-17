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
   missing.

3. **Bare metal is blocked.** HostPool/Host were removed pending
   redesign. The bare-metal-operator repo is empty. There is no
   bare-metal compute path in OSAC today.

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
backend: "baremetal"
capabilities:
  cores: 96
  memoryGiB: 480
  gpus:
    count: 4
    model: "B200"
  storage:
    bootDiskGiB: 960
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
backend: "virtual"
capabilities:
  cores:
    min: 4
    max: 32
  memoryGiB:
    min: 16
    max: 128
  gpus:
    count: 1
    model: "A100"
  storage:
    bootDiskGiB:
      min: 50
      max: 500
templates:
  - name: "kubevirt-a100-paris"
    site: "paris"
```

For bare metal, capabilities are fixed (the hardware is what it is).
For VMs, capabilities can specify ranges that the tenant selects
within.

The `backend` field (`baremetal` or `virtual`) indicates the
provisioning model. Tenants may or may not see this field —
providers can choose to expose or hide it.

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
  image: "rhel-9.6-gpu"
  sshKeys:
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
status:
  state: "RUNNING"
  ipAddress: "10.100.0.10"
  conditions: [...]
```

The existing ComputeInstance fields from the vmaas and vm-api-fields
EPs are preserved. The changes are:

- `template` field is replaced by `computeInstanceClass`
- `template_parameters` is removed — standardized fields replace
  freeform parameters
- `region` is added (optional) — narrows template selection within
  the class
- `image` and `sshKeys` are string references (to Image and SSHKey
  resources defined in a separate EP; until that EP lands, they
  remain inline values as today)

For bare-metal classes, VM-specific fields (`cores`, `memoryGiB`,
`bootDisk`, `runStrategy`) are either ignored or validated as fixed
values matching the class capabilities. The validation logic is:

- If the class capability is a fixed value (e.g., `cores: 96`),
  the instance field must match or be omitted
- If the class capability is a range (e.g., `cores: {min: 4, max: 32}`),
  the instance field must fall within the range
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
  image: "rhel-9.6-gpu"
  sshKeys:
    - "my-workstation-key"
  subnet: "my-vpc-subnet"
  placementPolicy:
    strategy: "pack"
    affinityKey: "nvlink-domain"
status:
  readyReplicas: 8
  instances:
    - "my-training-pool-0"
    - "my-training-pool-1"
    - ...
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

### Workflow

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
| ComputeInstance | Add `computeInstanceClass` field. Add `region` field. Remove `template` and `template_parameters` fields. |

#### Deprecated resources

| Resource | Replaced by | Migration |
|---|---|---|
| ComputeInstanceTemplate (current, parameter-definition style) | ComputeInstanceClass + ComputeInstanceTemplate (Ansible role) | Templates become classes; parameter definitions become class capabilities |
| HostPool (removed) | ComputeInstanceGroup | Same scaling semantics, unified with VM |
| Host (removed) | ComputeInstance with bare-metal class | Same lifecycle, unified API |
| HostClass (removed) | ComputeInstanceClass | Hardware profiles become classes |
| HostType (current) | ComputeInstanceClass | Merged |

### Implementation Details

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
- New `osac.service.select_template` utility role that queries AAP
  inventory for capacity and selects a template from the class
- Existing template roles (`ocp_virt_vm`) adapted to receive
  standardized variables instead of freeform `template_parameters`

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

Replacing `template` with `computeInstanceClass` on ComputeInstance
is a breaking API change.

*Mitigation:* Support both fields during a transition period. If
`template` is specified and `computeInstanceClass` is not, look up
the class that references that template and use it. Log a
deprecation warning.

#### ComputeInstanceGroup controller complexity

Managing a set of ComputeInstances with scaling, placement, and
failure handling adds controller complexity.

*Mitigation:* Start with simple first-fit scaling (no placement
policy). Add placement strategies incrementally in follow-up EPs.

#### Backend-specific semantics leaking into the unified model

Different backends have different capabilities (VMs support live
migration, bare metal doesn't; bare metal has fixed cores, VMs
have configurable cores).

*Mitigation:* The `backend` field on ComputeInstanceClass
communicates the provisioning model. Field validation adapts based
on whether capabilities are fixed or ranges. The API shape is the
same; the validation rules differ.

### Drawbacks

- Adds a level of indirection (Class → Template → Role) that
  increases conceptual complexity for providers setting up the
  catalog.
- Tenants who were using `template_parameters` for VM customization
  must switch to standardized fields.

## Alternatives

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

TBD — the migration path section above covers the data migration.
API versioning and backward compatibility during transition will be
detailed when targeting a release.
