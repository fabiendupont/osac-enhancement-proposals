---
title: inventory-provisioning-separation
authors:
  - Fabien Dupont
creation-date: 2026-04-17
last-updated: 2026-04-17
tracking-link:
  - TBD
see-also:
  - enhancements/unified-compute-model/README.md
  - enhancements/carbide-integration/README.md
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

### Non-Goals

- Prescribing a specific inventory system (NetBox, NICo, etc.).
- Implementing placement algorithms beyond basic first-fit (advanced
  placement strategies like NVLink-aware packing are backend- and
  topology-specific and will evolve incrementally).
- Changes to the fulfillment-service or osac-operator — this EP
  is entirely within the AAP layer.

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
Phase 3: Update inventory (shared)
  │
  │ Input:  provisioned host names, tenant, state
  │ Action: update AAP inventory host_vars (e.g., consumption, tenant, state)
```

Phase 1 and Phase 3 are **shared** across all backends — they
operate on AAP inventory, not on any specific backend API. Phase 2
is **template-specific** — each backend has its own provisioning
role.

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
#     - { name: "host-01", site: "paris", nvlink_domain: "dom-3", ... }
#     - { name: "host-02", site: "paris", nvlink_domain: "dom-3", ... }
#     - { name: "host-03", site: "paris", nvlink_domain: "dom-3", ... }
#     - { name: "host-04", site: "paris", nvlink_domain: "dom-3", ... }
```

The role:

1. Queries AAP inventory for hosts matching the class (by
   `hostclass` group or host_var).
2. Filters by region/site if specified.
3. Filters by `state: available` (or equivalent host_var).
4. Applies placement strategy (first-fit for `pack`, round-robin
   for `spread`, group-by for affinity).
5. Acquires a distributed lock (K8s Lease, per hostclass) to
   prevent race conditions — following the pattern from the removed
   HostPool code.
6. Returns the selected hosts as a list.

### Inventory host_vars contract

For `select_hosts` to work across inventory sources, hosts must
have a minimum set of host_vars:

| host_var | Type | Description | Set by |
|---|---|---|---|
| `hostclass` | string | ComputeInstanceClass this host can fulfill | Inventory plugin |
| `state` | string | `available`, `allocated`, `maintenance`, `error` | Inventory plugin + update role |
| `site` | string | Site/location identifier | Inventory plugin |
| `tenant` | string | Tenant ID (when allocated) | Update role |
| `nvlink_domain` | string | NVLink domain (for GPU hosts) | Inventory plugin |
| `rack` | string | Rack identifier | Inventory plugin |

Each inventory plugin maps its source data to these host_vars:

| host_var | NetBox source | NICo source | OpenStack/Ironic source |
|---|---|---|---|
| `hostclass` | Custom field or DeviceType | InstanceType name | resource_class |
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
    update_host_names: "{{ select_hosts_result | map(attribute='name') }}"
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
compute_instance_image:            # resolved Image (sourceType, sourceRef, bootMethod)
compute_instance_ssh_keys:         # list of resolved SSH public key strings
compute_instance_subnet:           # subnet reference
compute_instance_security_groups:  # list of security group references
compute_instance_user_data:        # user data string
compute_instance_class:            # ComputeInstanceClass (capabilities)
```

The role's responsibility is solely to **provision** — create the
backend resources (BareMetalHost, VirtualMachine, NICo Instance)
and report status. It does not search for available hosts.

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

  # Phase 2: Provision (template-specific)
  - name: Provision compute instance
    ansible.builtin.include_role:
      name: "{{ selected_template.roleCollection }}.{{ selected_template.role }}"
      tasks_from: install.yaml
    vars:
      compute_instance_hosts: "{{ select_hosts_result }}"
      compute_instance_image: "{{ resolved_image }}"
      compute_instance_ssh_keys: "{{ resolved_ssh_keys }}"

  # Phase 3: Update inventory (shared, inventory-agnostic)
  - name: Update host state
    ansible.builtin.include_role:
      name: osac.service.update_host_state
    vars:
      update_host_names: "{{ select_hosts_result | map(attribute='name') }}"
      update_host_state: "allocated"
      update_host_tenant: "{{ tenant_id }}"
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

### Implementation Details

New roles in `osac.service`:
- `select_hosts` — host selection with placement and locking
- `update_host_state` — inventory state update (backend-aware)

Modified roles:
- Existing template roles refactored to not query inventory
  directly

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
with a clear error if missing.

#### State synchronization

AAP inventory may be stale — a host marked `available` in
inventory may have been allocated by another process.

*Mitigation:* The distributed lock in `select_hosts` (K8s Lease
per hostclass) serializes allocation decisions. The provisioning
role validates that the host is still available before
provisioning and fails fast if not.

### Drawbacks

- Adds an inventory abstraction layer that may feel over-engineered
  for simple single-backend deployments.
- Requires inventory plugins to conform to the host_vars contract.

## Alternatives

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

## Test Plan

TBD

## Graduation Criteria

TBD
