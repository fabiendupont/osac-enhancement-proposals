# Building an OSAC Add-On

This guide explains how to build an OSAC Add-On — a self-contained
Ansible collection that extends the OSAC platform with a new provider
or capability.

For the full specification, see
[enhancements/osac-addon/README.md](enhancements/osac-addon/README.md).

## Quick Start

### 1. Create the collection

```bash
ansible-galaxy collection init mycompany.osac_networking
```

### 2. Add `meta/addon.yaml`

```yaml
name: mycompany-networking
display_name: MyCompany Networking Provider
version: 1.0.0
dependencies:
  osac_core: ">=0.1.0"
resource_types:
  - name: VirtualNetwork
    scope: tenant
  - name: Subnet
    scope: tenant
```

### 3. Create ResourceAction roles

Each resource needs roles with entry point tasks. The operator
dispatches to these based on the `collection` field on the *Class CR.

```
roles/
├── virtual_network/
│   ├── meta/osac.yaml       # resource_type, parameters, outputs
│   └── tasks/
│       ├── create.yaml       # provisioning logic
│       └── delete.yaml       # deprovisioning logic
└── subnet/
    ├── meta/osac.yaml
    └── tasks/
        ├── create.yaml
        └── delete.yaml
```

### 4. Define the role contract in `meta/osac.yaml`

```yaml
resource_type: VirtualNetwork
parameters:
  - name: ipv4_cidr
    type: string
    required: true
    description: IPv4 CIDR block
outputs:
  - name: network_id
    type: string
    description: Created network ID
  - name: subnet_id
    type: string
    description: Default subnet ID
```

### 5. Report outputs via `set_stats`

```yaml
- name: Report outputs
  ansible.builtin.set_stats:
    data:
      osac_state: READY
      network_id: "{{ result.id }}"
      subnet_id: "{{ result.subnet_id }}"
```

### 6. Create playbooks

```
playbooks/
├── create.yml
└── delete.yml
```

Playbooks read `osac_resource` (the full K8s resource spec)
from `extra_vars` provided by the operator.

### 7. Mark helper roles as internal

If a role has `create.yaml`/`delete.yaml` but is called by
another role (not directly by the operator), mark it:

```yaml
# roles/my_helper/meta/osac.yaml
internal: true
description: Internal helper for network setup
```

### 8. Lint

```bash
pip install osac-addon-lint
osac-addon-lint /path/to/collection
osac-addon-lint /path/to/collection --strict
```

### 9. Publish

```bash
ansible-galaxy collection publish mycompany-osac_networking-1.0.0.tar.gz
```

## Collection Naming Convention

| Owner | Pattern | Example |
|---|---|---|
| OSAC-maintained | `osac.<domain>_<provider>` | `osac.compute_kubevirt` |
| Partner single-domain | `<partner>.osac_<domain>` | `dell.osac_object_storage` |
| Partner multi-domain | `<partner>.osac` | `netris.osac` |

## Role I/O Contract

### Inputs (extra_vars from operator)

| Variable | Description |
|---|---|
| `osac_resource` | Full K8s resource spec + metadata |
| `osac_resource_type` | Resource kind (e.g., `ComputeInstance`) |
| `tenant_target_namespace` | Resolved tenant namespace |

### Outputs (set_stats)

| Variable | Required | Description |
|---|---|---|
| `osac_state` | yes | `READY`, `FAILED`, `DELETING` |
| `osac_conditions` | no | Condition updates |
| Declared outputs | no | From `meta/osac.yaml` outputs |

## Reference Collections

| Collection | Description |
|---|---|
| [osac.compute_kubevirt](https://github.com/fabiendupont/osac.compute_kubevirt) | KubeVirt VM provisioning (simplest example) |
| [osac.kubernetes_hcp](https://github.com/fabiendupont/osac.kubernetes_hcp) | Hosted Control Planes (multi-role example) |
| [osac.service](https://github.com/fabiendupont/osac.service) | Shared utility roles |

## Linter

[osac-addon-lint](https://github.com/fabiendupont/osac-addon-lint) validates:
- `galaxy.yml` and `meta/addon.yaml` structure
- Role metadata (`resource_type`, parameters, outputs)
- Playbook create/delete pairs
- Rollback completeness (every create has a delete)
- No legacy EDA references
- `internal: true` for helper roles
