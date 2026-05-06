---
title: inventory-provisioning-separation
authors:
  - Fabien Dupont
creation-date: 2026-04-17
last-updated: 2026-04-29
tracking-link:
  - TBD
see-also:
  - enhancements/unified-compute-model/README.md
  - enhancements/osac-addon/README.md
  - enhancements/carbide-integration/README.md
  - enhancements/metal3-compute-backend/README.md
  - enhancements/image-and-sshkey-resources/README.md
  - enhancements/catalog-items/README.md
  - enhancements/cost-metric-mapping/README.md
  - enhancements/composable-catalog-items/README.md
  - https://github.com/osac-project/enhancement-proposals/pull/20 (Region/AZ)
replaces:
superseded-by:
---

# Inventory and Provisioning Separation in AAP Backends

## Summary

Establish a design principle and architectural pattern for OSAC's
AAP backends: host discovery (inventory) and host provisioning
must be separate, independently swappable concerns.

- **Inventory** is sourced from AAP dynamic inventory plugins
  (NetBox, NICo, OpenStack/Ironic, or any other source). It answers
  "what machines exist and are available?"
- **Provisioning** is implemented by ComputeInstanceTemplate Ansible
  roles (Metal3, KubeVirt, NICo, ESI). It answers "provision this
  machine."

A shared host selection utility in `osac.service` bridges the two:
given a ComputeInstanceClass and placement policy, it queries AAP
inventory for matching available hosts and returns the selected set
to the provisioning role.

This EP also formalizes three role contracts — input, output, and
inventory host_vars — and introduces a compliance checking
mechanism that validates template roles against the OSAC API
schema at build time, preventing contract drift between Ansible
roles and the fulfillment-service API.

## Motivation

### Current state

In the current ESI backend, inventory query and provisioning are
entangled in the same Ansible collection (`massopencloud.esi`):

```yaml
# massopencloud.esi.host / list_hosts.yaml
- openstack.cloud.baremetal_node_info:     # ← queries Ironic directly
  register: baremetal_node_info_result
- set_fact:
    node_list: "{{ ... | selectattr('resource_class', 'equalto', host_class) }}"
    host_list: "{{ node_list | map('massopencloud.esi.ironic_node_to_osac_host') }}"
```

The `list_hosts` task directly calls the OpenStack Ironic API to
find available machines. The same collection then provisions on
those machines. Switching from ESI to Metal3 requires replacing
both the "find machines" and "provision machines" logic — they
share the same backend connection and are not independently
swappable.

### Problems

1. **Backend lock-in in host discovery.** The Carbide integration
   EP describes a similar pattern: its template role would query
   NICo's API for available instances by type. Each backend
   re-implements host discovery differently.

2. **No shared placement logic.** When HostPool existed, host
   selection was in `manage_hosts/tasks/label_new_hosts.yml` — a
   first-fit algorithm specific to ESI's labeling model. A Metal3
   backend would need to re-implement the same logic with different
   labeling.

3. **Inventory and provisioning use the same credentials.** The ESI
   backend uses OpenStack credentials for both listing hosts and
   provisioning them. This prevents using a separate, richer
   inventory source (like NetBox) while provisioning via a simpler
   engine (like Metal3).

### Desired state

```
AAP Dynamic Inventory              ComputeInstanceTemplate Role
  │                                   │
  │ "What's available?"               │ "Provision this machine"
  │ (read-only, any source)           │ (mutating, specific backend)
  │                                   │
  ├── netbox.netbox.nb_inventory      ├── metal3_bm_rhel role
  ├── nvidia.bare_metal.bmm           ├── nico_bm role
  ├── openstack.cloud inventory       ├── esi_bm role
  └── custom plugin                   └── kubevirt_vm role
```

The inventory source and the provisioning backend are independently
chosen. A provider can use NetBox for inventory and Metal3 for
provisioning. Or NICo for both. Or NetBox for inventory and NICo
for provisioning.

### User Stories

- As a **cloud provider**, I want to choose my inventory system
  (NetBox, NICo, or OpenStack) independently from my provisioning
  backend (Metal3, NICo, or ESI).

- As an **infrastructure provider** (template author), I want to
  write a provisioning role that receives a list of pre-selected
  hosts, without needing to implement host discovery logic.

- As a **cloud provider**, I want shared placement logic (pack,
  spread, affinity) that works regardless of which inventory source
  or provisioning backend I use.

### Goals

- Define the boundary between inventory and provisioning in AAP
  backends.
- Provide a shared host selection utility in `osac.service`.
- Ensure ComputeInstanceTemplate roles receive pre-selected hosts,
  not raw inventory queries.
- Formalize input, output, and inventory host_vars contracts for
  template roles.
- Provide build-time and CI compliance checking that validates
  template role contracts against the OSAC API schema derived
  from proto definitions.

### Non-Goals

- Prescribing a specific inventory system (NetBox, NICo, etc.).
- Implementing placement algorithms beyond basic first-fit (advanced
  placement strategies like NVLink-aware packing are backend- and
  topology-specific and will evolve incrementally).
- Composed catalog item orchestration — the output contract
  enables it, but the workflow generator and catalog composition
  model are covered by a separate EP.
- Runtime contract enforcement as a hard gate — runtime
  validation is opt-in for debugging. Build-time and CI checks
  are the primary enforcement mechanism.

## Proposal

### Architectural boundary

The AAP workflow for provisioning a ComputeInstance (or a
ComputeInstanceGroup member) follows three phases:

```
Phase 1: Select hosts (shared)
  │
  │ Input:  ComputeInstanceClass capabilities, region, placement policy
  │ Source: AAP inventory (populated by dynamic inventory plugin)
  │ Output: list of selected host names
  │
  ▼
Phase 2: Provision (template-specific)
  │
  │ Input:  selected host names, image, sshKeys, subnet, userData
  │ Source: ComputeInstanceTemplate role
  │ Output: provisioned compute instances
  │
  ▼
Phase 3: Report outputs (template-specific)
  │
  │ Input:  provisioning results from Phase 2
  │ Action: export standardized outputs via set_stats
  │ Output: instance_id, ip_address, endpoint_url, etc.
  │
  ▼
Phase 4: Update inventory (shared)
  │
  │ Input:  provisioned host names, tenant, state
  │ Action: update AAP inventory host_vars (e.g., consumption, tenant, state)
```

Phase 1 and Phase 4 are **shared** across all backends — they
operate on AAP inventory, not on any specific backend API. Phases
2 and 3 are **template-specific** — each backend has its own
provisioning role and declares the outputs it produces.

### Host selection utility

A new role `osac.service.select_hosts` implements Phase 1:

```yaml
# Usage in a workflow:
- name: Select available hosts
  ansible.builtin.include_role:
    name: osac.service.select_hosts
  vars:
    select_hosts_class: "gpu-b200-4"
    select_hosts_count: 4
    select_hosts_region: "eu-west"
    select_hosts_placement:
      strategy: "pack"
      affinityKey: "nvlink_domain"

# Result:
#   select_hosts_result:
#     - { inventory_hostname: "host-01", site: "paris", nvlink_domain: "dom-3", ... }
#     - { inventory_hostname: "host-02", site: "paris", nvlink_domain: "dom-3", ... }
#     - { inventory_hostname: "host-03", site: "paris", nvlink_domain: "dom-3", ... }
#     - { inventory_hostname: "host-04", site: "paris", nvlink_domain: "dom-3", ... }
```

The role:

1. Queries AAP inventory for hosts matching the class (by
   `compute_instance_class` group or host_var).
2. Filters by region/site if specified (region semantics defined
   by the [Region/AZ EP](https://github.com/osac-project/enhancement-proposals/pull/20)).
3. Filters by `state: available` (or equivalent host_var).
4. Applies placement strategy (first-fit for `pack`, round-robin
   for `spread`, group-by for affinity).
5. Acquires a distributed lock via `osac.service.lease` (K8s Lease,
   per compute_instance_class) to prevent race conditions. The lock is held in
   a block/always pattern to guarantee release even on failure.
6. Returns the selected hosts as a list of dicts, each containing
   `inventory_hostname` and relevant host_vars.

### Inventory host_vars contract

For `select_hosts` to work across inventory sources, hosts must
have a minimum set of host_vars:

| host_var | Type | Description | Set by |
|---|---|---|---|
| `compute_instance_class` | string | ComputeInstanceClass this host can fulfill | Inventory plugin |
| `state` | string | `available`, `allocated`, `maintenance`, `error` | Inventory plugin + update role |
| `site` | string | Site/location identifier | Inventory plugin |
| `tenant` | string | Tenant ID (when allocated) | Update role |
| `nvlink_domain` | string | NVLink domain (for GPU hosts) | Inventory plugin |
| `rack` | string | Rack identifier | Inventory plugin |

Each inventory plugin maps its source data to these host_vars:

| host_var | NetBox source | NICo source | OpenStack/Ironic source |
|---|---|---|---|
| `compute_instance_class` | Custom field or DeviceType | InstanceType name | resource_class |
| `state` | Device status | Machine status | provision_state |
| `site` | Site name | Site name | — |
| `tenant` | Tenant name | Tenant name | extra.tenant |
| `nvlink_domain` | Custom field | Machine label | extra.nvlink_domain |
| `rack` | Rack name | Rack ID | — |

### Inventory update utility

A new role `osac.service.update_host_state` implements Phase 3:

```yaml
# Usage after provisioning:
- name: Mark hosts as allocated
  ansible.builtin.include_role:
    name: osac.service.update_host_state
  vars:
    update_host_names: "{{ select_hosts_result | map(attribute='inventory_hostname') | list }}"
    update_host_state: "allocated"
    update_host_tenant: "{{ tenant_id }}"
```

This role updates host_vars in the inventory source. The
implementation depends on which inventory source is configured:

- **NetBox**: `netbox.netbox.netbox_device` module to update device
  status and tenant
- **NICo**: `nvidia.bare_metal.machine` module to update machine
  labels
- **OpenStack/Ironic**: `openstack.cloud.baremetal_node` module to
  update extra fields

The role detects the inventory source from the host's
`inventory_source` var or from a global configuration variable.

### ComputeInstanceTemplate role contract

Template roles receive **pre-selected hosts** as input — they do
not query inventory themselves. The role interface:

```yaml
# Variables received by the template role:
compute_instance_hosts:            # list of selected hosts (from select_hosts)
compute_instance_image:            # resolved Image (sourceType, sourceRef, bootMethod, checksum)
compute_instance_ssh_keys:         # list of resolved SSH public key strings
compute_instance_subnet:           # subnet reference
compute_instance_security_groups:  # list of security group references
compute_instance_user_data:        # user data string
compute_instance_class:            # ComputeInstanceClass (capabilities)
```

**Reference resolution:** The `compute_instance_image` and
`compute_instance_ssh_keys` variables contain **resolved data**,
not name references. The fulfillment-service resolves `imageRef`
and `sshKeyRefs` at ComputeInstance creation time and stores the
results as `resolvedImage` and `resolvedSshKeys` on the private
ComputeInstance record. The operator passes this resolved data to
AAP — template roles never need to call the fulfillment API.

The role's responsibility is solely to **provision** — create the
backend resources (BareMetalHost, VirtualMachine, NICo Instance)
and report status. It does not search for available hosts.

### ComputeInstanceTemplate output contract

Template roles must declare and produce a standardized set of
outputs via `ansible.builtin.set_stats`. These outputs serve
three consumers:

- **The feedback controller** in osac-operator, which syncs
  provisioning results back to the fulfillment-service PostgreSQL
  database and updates the ComputeInstance CRD status.
- **Downstream workflow nodes** in composed catalog items, where
  one role's outputs feed into another role's inputs (e.g., a
  compute role produces `instance_id`, which a monitoring role
  consumes).
- **The fulfillment-service API**, which exposes provisioning
  results to tenants (IP address, endpoint URL, connection
  details).

#### Output declaration

Each template role declares its outputs in
`meta/osac_contract.yaml`:

```yaml
# meta/osac_contract.yaml
contract_version: "1"

inputs:
  required:
    - compute_instance_hosts
    - compute_instance_image
    - compute_instance_ssh_keys
    - compute_instance_subnet
    - compute_instance_class
  optional:
    - compute_instance_security_groups
    - compute_instance_user_data

outputs:
  required:
    instance_id:
      type: string
      description: Backend-specific instance identifier
    instance_state:
      type: string
      description: Provisioning result state
      enum: [running, error]
    ip_address:
      type: string
      description: Primary IP address assigned to the instance
  optional:
    hostname:
      type: string
      description: FQDN assigned to the instance
    endpoint_url:
      type: string
      description: Service endpoint URL (for serving workloads)
    additional_ips:
      type: list
      description: Secondary IP addresses
    storage_device:
      type: string
      description: Block storage device path
    backend_ref:
      type: string
      description: Backend-specific resource reference (e.g., BareMetalHost name, VirtualMachine UID)
```

#### Output production

Provider roles produce outputs at the end of `instance.create.main`
using `set_stats`, following the
[OSAC Add-On I/O contract](../osac-addon/README.md). The
`set_stats` module writes data to the AAP job artifacts, making
them available to the workflow engine and to any downstream
consumer.

```yaml
# At the end of instance.create.main:
- name: Report provisioning outputs
  ansible.builtin.set_stats:
    data:
      instance_id: "{{ created_instance.metadata.uid }}"
      instance_state: "running"
      ip_address: "{{ created_instance.status.ip }}"
      hostname: "{{ created_instance.status.hostname | default(omit) }}"
      backend_ref: "{{ created_instance.metadata.name }}"
```

All required outputs must be present when `instance_state` is
`running`. When `instance_state` is `error`, only `instance_id`
and `instance_state` are required — the role should also set an
`error_message` output with a human-readable failure description.

#### Output consumption by the feedback controller

The osac-operator feedback controller reads `set_stats` outputs
from the AAP job artifacts API and maps them to ComputeInstance
CRD status fields:

| Output | CRD status field |
|---|---|
| `instance_id` | `status.instanceId` |
| `instance_state` | `status.state` |
| `ip_address` | `status.ipAddress` |
| `hostname` | `status.hostname` |
| `endpoint_url` | `status.endpointUrl` |
| `backend_ref` | `status.backendRef` |

This replaces the current ad hoc status scraping where the
feedback controller parses backend-specific CRD fields
differently per template type. With the output contract, the
feedback controller has a single, backend-agnostic code path.

#### Delete outputs

Provider roles must also produce outputs at the end of
`instance.delete.main`:

```yaml
- name: Report deletion outputs
  ansible.builtin.set_stats:
    data:
      instance_id: "{{ instance_id }}"
      instance_state: "deleted"
```

#### Signal outputs

The optional `instance.signal.main` role, when implemented,
produces the same output schema as `instance.create.main`. This
enables periodic status reconciliation — the osac-operator
can invoke `instance.signal.main` to refresh the CRD status
without re-provisioning.

### Contract compliance checking

Template roles must comply with the contracts defined above —
the input contract (what variables the role expects), the output
contract (what `set_stats` keys the role produces), and the
inventory host_vars contract (what host_vars `select_hosts`
requires). Drift between these contracts and the OSAC API schema
causes silent failures: a role that stops producing `ip_address`
breaks the feedback controller; a role that expects a variable
the operator no longer sends fails at runtime.

The compliance checking mechanism catches these mismatches at
build time (EE image build and CI), not at provisioning time.

#### Contract schema source of truth

The fulfillment-service proto definitions are the authoritative
source for the OSAC API schema. The contract schemas are derived
from these protos:

- **Input contract** fields map to ComputeInstance proto spec
  fields (e.g., `compute_instance_image` maps to
  `ComputeInstance.spec.resolvedImage`)
- **Output contract** fields map to ComputeInstance proto status
  fields (e.g., `ip_address` maps to
  `ComputeInstance.status.ipAddress`)
- **Inventory host_vars** map to ComputeInstanceClass proto
  capability fields

A JSON Schema file is generated from the proto definitions
and published as part of the `osac.service` collection:

```
osac.service/
  schemas/
    contract_v1.json          # JSON Schema for osac_contract.yaml
    inputs_v1.json            # Input variable schema (from proto)
    outputs_v1.json           # Output variable schema (from proto)
    host_vars_v1.json         # Inventory host_vars schema
```

#### Build-time validation (EE image build)

A validation role `osac.service.validate_contracts` runs during
EE image build as a post-build check. It:

1. Discovers all roles in the EE that contain
   `meta/osac_contract.yaml`.
2. Validates each `osac_contract.yaml` against
   `contract_v1.json` (structural correctness).
3. Validates declared inputs against `inputs_v1.json` — are all
   required inputs from the OSAC API listed? Are there unknown
   inputs that suggest the role expects variables the operator
   will not provide?
4. Validates declared outputs against `outputs_v1.json` — does
   the role declare all required outputs? Are output types
   correct?
5. Reports warnings for optional fields not declared and errors
   for required fields missing or type mismatches.

```bash
# In the EE build pipeline:
ansible-playbook osac.service.validate_contracts \
  -e schema_dir=osac.service/schemas \
  -e role_dirs=osac.compute_kubevirt,osac.compute_metal3
```

A non-zero exit code fails the EE build if any required contract
field is missing or mistyped.

#### CI validation (per-role PR checks)

Each provider collection runs contract validation in CI on
every PR that modifies role code or `meta/osac.yaml`. This
aligns with the `osac addon lint` tool from the
[OSAC Add-On EP](../osac-addon/README.md):

```yaml
# .github/workflows/contract-check.yml
- name: Validate OSAC contract
  run: |
    osac addon lint osac.compute_metal3
```

The linter performs:

1. **Schema validation:** `meta/osac.yaml` declares all
   required fields and types match the OSAC API schema.
2. **Static analysis:** `instance.create.main` and
   `instance.delete.main` task files contain `set_stats` calls
   that produce all declared required outputs. This is a
   best-effort static check — it parses task YAML for
   `ansible.builtin.set_stats` calls and verifies the `data`
   keys match the declared outputs.
3. **Rollback completeness:** every `create.main` has a
   corresponding `delete.main`.

#### Runtime validation (optional, defense in depth)

The `osac.service.execute_catalog_item` role (used for
composed catalog items) can optionally validate outputs at
runtime after each workflow node completes:

```yaml
- name: Validate outputs from {{ node.name }}
  ansible.builtin.assert:
    that:
      - item in (ansible_stats.data | default({}))
    fail_msg: >
      Role {{ node.role }} did not produce required output '{{ item }}'.
      Check the role's set_stats call in instance.create.main.
  loop: "{{ node.contract.outputs.required | list }}"
```

Runtime validation is disabled by default (the linter and CI
checks are the primary enforcement). It can be enabled per
deployment via `osac_validate_outputs_at_runtime: true` for
debugging or during initial provider onboarding.

#### Contract versioning

The `contract_version` field in `osac_contract.yaml` tracks
breaking changes to the contract schema. When the OSAC API adds
a new required output field (e.g., a future `gpu_device_id`),
the contract version increments. Roles declaring an older
contract version receive a CI warning with a migration guide.
Roles must update to the current contract version before the
next EE release.

The version follows a simple integer scheme (1, 2, 3...), not
semver. Each version bump includes a changelog in
`osac.service/schemas/CHANGELOG.md` listing added, changed, and
removed fields.

#### Proto-to-schema generation

The JSON Schema files are generated from the fulfillment-service
proto definitions using a `buf` plugin or a standalone Go tool
that reads the proto descriptors and emits JSON Schema:

```bash
# In fulfillment-service CI, after buf generate:
go run ./cmd/gen-contract-schema \
  --proto proto/private/osac/private/v1/compute_instance_type.proto \
  --output schemas/
```

The generated schemas are published as part of the
`osac.service` collection. When a proto field changes (renamed,
removed, type changed), the schema regeneration produces a
different output, and any template role CI that depends on the
schema fails — surfacing the contract drift immediately.

This creates a closed feedback loop:

```
Proto change (fulfillment-service)
  → schema regeneration (fulfillment-service CI)
  → osac.service collection update
  → template role CI fails (contract mismatch)
  → role author updates osac_contract.yaml and set_stats calls
  → EE build validates all roles pass
```

### Example: workflow composition

```yaml
# Simplified ComputeInstance create workflow:
tasks:
  # Phase 1: Select hosts (shared, inventory-agnostic)
  - name: Select hosts for compute instance
    ansible.builtin.include_role:
      name: osac.service.select_hosts
    vars:
      select_hosts_class: "{{ compute_instance.spec.computeInstanceClass }}"
      select_hosts_count: 1
      select_hosts_region: "{{ compute_instance.spec.region | default(omit) }}"

  # Phase 2: Provision (provider-specific ResourceAction)
  # The operator reads the ComputeInstanceClass collection field
  # and invokes the instance.create.main role from that collection.
  # Note: compute_instance_image and compute_instance_ssh_keys contain
  # resolved data from the ComputeInstance's resolvedImage and resolvedSshKeys
  # fields, populated by the fulfillment-service at creation time.
  - name: Provision compute instance
    ansible.builtin.include_role:
      name: "{{ compute_instance_class_collection }}.instance.create.main"
    vars:
      compute_instance_hosts: "{{ select_hosts_result }}"

  # Phase 3: Report outputs (via set_stats, per OSAC Add-On convention)
  # The provider role's instance.create.main produces outputs via set_stats:
  #   instance_id, instance_state, ip_address, hostname, backend_ref
  # These are available in ansible_stats.data for downstream consumers
  # and in AAP job artifacts for the feedback controller.

  # Phase 4: Update inventory (shared, inventory-agnostic)
  - name: Update host state
    ansible.builtin.include_role:
      name: osac.service.update_host_state
    vars:
      update_host_names: "{{ select_hosts_result | map(attribute='inventory_hostname') | list }}"
      update_host_state: "allocated"
      update_host_tenant: "{{ compute_instance_tenant | default('') }}"
```

### What changes for existing backends

#### ESI backend

The `massopencloud.esi.host` list_hosts tasks are replaced by
`osac.service.select_hosts` querying AAP inventory (which could
still be sourced from OpenStack/Ironic). The ESI provisioning role
(`esi_bm`) keeps its attach/detach/clean tasks but stops querying
Ironic for host lists.

#### Carbide/NICo backend (Trey's EP)

The Carbide integration EP describes agent selection by instance
type. With this separation, host selection uses
`osac.service.select_hosts` (AAP inventory sourced from NICo via
`nvidia.bare_metal.bmm`). The NICo provisioning role (`nico_bm`)
receives pre-selected hosts and attaches them to VPCs.

### Implementation Details/Notes/Constraints

New roles in `osac.service`:
- `select_hosts` — host selection with placement and locking
- `update_host_state` — inventory state update (backend-aware)
- `validate_contracts` — build-time contract compliance checker
- `execute_catalog_item` — generic workflow generator for
  composed catalog items (see composable-catalog-items EP)

New CLI tool:
- `osac-contract-validate` — CI-oriented contract validator
  with static analysis of `set_stats` calls

New schemas in `osac.service`:
- `schemas/contract_v1.json` — JSON Schema for
  `osac_contract.yaml`
- `schemas/inputs_v1.json` — input variable schema (generated
  from proto)
- `schemas/outputs_v1.json` — output variable schema (generated
  from proto)
- `schemas/host_vars_v1.json` — inventory host_vars schema

New proto tooling in `fulfillment-service`:
- `cmd/gen-contract-schema` — generates JSON Schema from proto
  descriptors for ComputeInstance spec and status fields

Modified roles:
- Existing template roles refactored to ResourceAction naming
  (`instance.create.main`, `instance.delete.main`,
  `instance.signal.main`) in per-provider collections
  (`osac.compute_kubevirt`, `osac.compute_metal3`):
  - Not query inventory directly
  - Add `meta/osac.yaml` declaring inputs and outputs (per
    [OSAC Add-On convention](../osac-addon/README.md))
  - Produce standardized outputs via `set_stats`

AAP configuration:
- Dynamic inventory sources configured per deployment (NetBox,
  NICo, Ironic, or custom)
- Constructed inventory for cross-source group computation (e.g.,
  `available_gpu_b200` group)

### Risks and Mitigations

#### Inventory plugin latency

Dynamic inventory refresh may be slower than direct API queries.

*Mitigation:* AAP inventory caching. Smart inventories with
scheduled refresh intervals. For real-time accuracy, the
`select_hosts` role can optionally verify host state against the
source before selection.

#### Inventory host_vars contract enforcement

Different inventory plugins may not provide all required host_vars.

*Mitigation:* Document the contract. Provide example constructed
inventory configs that map source-specific fields to the standard
host_vars. Validate required host_vars in `select_hosts` and fail
with a clear error if missing. The `host_vars_v1.json` schema
enables automated validation of inventory plugin output in CI.

#### Contract drift between Ansible roles and OSAC API

Proto fields change (renamed, removed, type changed) in the
fulfillment-service. Template roles that produce outputs
matching the old field names silently break the feedback
controller.

*Mitigation:* The compliance checking mechanism creates a closed
feedback loop: proto changes regenerate JSON schemas, which fail
template role CI, which forces role authors to update before the
next EE build. Build-time validation in the EE pipeline is the
final gate — no EE image ships with contract-noncompliant roles.

#### Static analysis limitations

The `osac-contract-validate` CLI cannot follow dynamic
`include_role`, conditional `when` branches, or Jinja2-templated
`set_stats` keys. A role could pass static analysis but fail to
produce required outputs at runtime due to a conditional branch.

*Mitigation:* Static analysis is best-effort and catches the
common cases (missing `set_stats`, renamed keys, wrong module
name). Runtime validation is available as defense in depth for
initial template onboarding and debugging. Integration tests
in the template role's own CI should cover conditional branches.

#### State synchronization

AAP inventory may be stale — a host marked `available` in
inventory may have been allocated by another process.

*Mitigation:* The distributed lock in `select_hosts` (K8s Lease
per compute_instance_class) serializes allocation decisions. The provisioning
role validates that the host is still available before
provisioning and fails fast if not.

### Drawbacks

- Adds an inventory abstraction layer that may feel over-engineered
  for simple single-backend deployments.
- Requires inventory plugins to conform to the host_vars contract.
- The contract compliance mechanism adds build-time and CI
  dependencies between the fulfillment-service proto definitions
  and template role repositories. A proto change in
  fulfillment-service can break template role CI in a separate
  repo, requiring coordinated updates.
- Template role authors must maintain `meta/osac_contract.yaml`
  alongside the role code — an additional file to keep in sync.
  However, this is less work than debugging silent contract
  drift at runtime.

## Alternatives (Not Implemented)

### Keep inventory entangled with provisioning

Each backend implements its own host discovery.

*Why not:* Duplicates logic, prevents backend swapping, mixes
read-only (inventory) and mutating (provisioning) operations in
the same code path.

### Use OSAC fulfillment-service for host selection

Move host selection from AAP to the fulfillment-service Go code.

*Why not:* The fulfillment-service doesn't have access to
hardware inventory data (GPUs, NVLink domains, rack positions).
AAP inventory is the right place for this data, and Ansible is
the right tool for querying and filtering it.

## Open Questions [optional]

1. **Lock granularity.** Should the distributed lock in
   `select_hosts` be per-class or per-site-per-class?
   Per-class is simpler but serializes all allocations for a
   class across sites. Per-site-per-class allows parallel
   allocation at different sites but increases lock management
   complexity.

2. **Dry-run mode.** Should `select_hosts` support a dry-run mode
   that returns matching hosts without acquiring them? This would
   enable capacity planning UIs without the risk of accidental
   allocation.

3. **Contract schema distribution.** Should the JSON schemas
   generated from protos be distributed as part of the
   `osac.service` Ansible collection (bundled in the EE), as a
   separate Python package (`osac-contract-schemas`), or as a
   git submodule? The collection approach keeps everything in
   one artifact but creates a release dependency between
   fulfillment-service proto changes and collection releases.

4. **Third-party template roles.** How should external
   contributors (e.g., NICo backend authors at NVIDIA) validate
   their template roles against the OSAC contract? They need
   access to the schema files but may not have access to the
   fulfillment-service proto repository. Publishing schemas to
   a public registry (e.g., Ansible Galaxy metadata or a
   dedicated schema repository) would enable external
   validation.

5. **Output extensibility.** Should template roles be allowed
   to produce additional outputs beyond the contract (e.g.,
   backend-specific metadata like `ironic_node_uuid` or
   `kubevirt_vm_namespace`)? If so, should these be declared
   in `osac_contract.yaml` under an `extra` section, or left
   undeclared and passed through opaquely?

## Test Plan

TBD

## Graduation Criteria

TBD

## Upgrade / Downgrade Strategy

The new `osac.service.select_hosts` and
`osac.service.update_host_state` roles are added to the
`osac.service` Ansible collection in the EE. Upgrading means
deploying a new EE image with the updated collection. Existing
template roles are refactored to stop querying inventory directly
and instead receive pre-selected hosts — this is a breaking change
for template role interfaces. All template roles must be updated
in the same EE release.

Downgrading means reverting to the previous EE image, which
restores the old template role interfaces (direct inventory
queries). This is safe as long as no new-style template roles
were deployed.

## Version Skew Strategy

All components (select_hosts, update_host_state, template roles)
run in the same AAP EE image. The EE image is atomic — there is
no cross-component version skew within a single EE deployment.

## Support Procedures

- **Failed host selection:** Check AAP inventory for hosts matching
  the requested `compute_instance_class`, `state: available`, and `site`/region.
  Verify the dynamic inventory plugin is configured and synced.
- **"Host not available" after selection:** The distributed lock
  (K8s Lease) serializes allocation, but a host may have been
  allocated by another process between selection and provisioning.
  Check the AAP job output for lock acquisition and host state
  verification errors.
- **Inventory update failure:** If `update_host_state` fails, the
  host may remain marked as `available` in inventory despite being
  provisioned. Manually update the host state in the inventory
  source (NetBox, NICo, or Ironic).
- **Missing outputs in feedback controller:** If the
  ComputeInstance CRD status is not updating after provisioning,
  check the AAP job artifacts for `set_stats` output. If outputs
  are missing, the provider role's `instance.create.main` may
  not be producing them. Run `osac addon lint` against the
  collection to check compliance. Enable runtime validation
  (`osac_validate_outputs_at_runtime: true`) to get immediate
  failure feedback.
- **Contract validation failure in CI:** The provider role's
  `meta/osac.yaml` does not match the current OSAC API
  schema. Check which schema version the role declares and
  compare with the current schema in
  `osac.service/schemas/`. Review the OSAC Add-On EP
  for field changes since the role's last update.
- **EE build failure on contract check:** Run
  `osac.service.validate_contracts` locally with verbose output
  to identify which roles fail and which fields are missing or
  mistyped. Cross-reference with the proto definitions in
  `fulfillment-service/proto/`.

## Infrastructure Needed [optional]

None beyond existing OSAC infrastructure and AAP.
