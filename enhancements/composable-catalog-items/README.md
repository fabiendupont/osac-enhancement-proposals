---
title: composable-catalog-items
authors:
  - Fabien Dupont
creation-date: 2026-04-29
last-updated: 2026-04-30
tracking-link:
  - TBD
see-also:
  - enhancements/catalog-items/README.md
  - enhancements/unified-compute-model/README.md
  - enhancements/inventory-provisioning-separation/README.md
  - enhancements/cost-metric-mapping/README.md
  - enhancements/osac-addon/README.md
replaces:
superseded-by:
---

# Composable Catalog Items

## Summary

Extend the catalog system with two new concerns — **Blueprints**
and **catalog item enhancements** — that separate orchestration
from presentation and enable composable, billable service
offerings.

- **Blueprint** is a new resource type that defines a DAG of
  catalog items to provision as a unit. Blueprints declare
  parameters, resource dependencies, and output wiring.
  Blueprints are authored by cloud providers and — with
  guardrails — by tenant blueprint authors. A catalog item can
  reference a Blueprint instead of a Template, making the
  catalog item the tenant-facing presentation of a composed
  service.

- **Catalog item enhancements** add three capabilities to the
  existing catalog item model
  ([catalog-items EP](../catalog-items/README.md)):
  **inheritance** (extend a parent, narrow constraints,
  accumulate billing labels), **billing labels** (propagate to
  provisioned resources for Koku cost attribution), and
  **Koku-sourced cost preview** (query Koku at request time,
  degrade cleanly when Koku is absent).

The separation is deliberate:

| Concern | Resource | Audience | Responsibility |
|---|---|---|---|
| Orchestration | Blueprint | Provider engineers, tenant blueprint authors | What gets provisioned, in what order, with what wiring |
| Presentation | CatalogItem | Tenant admins, tenant users | What tenants see, what they can configure, billing labels |

A Blueprint is to a composed service what a Template is to an
atomic one. A CatalogItem wraps either one with presentation
and policy.

```
Template (atomic, one Ansible role)
  └── CatalogItem (presentation + policy)

Blueprint (composed, DAG of catalog items)
  └── CatalogItem (presentation + policy)
```

## Motivation

### Current state

The [catalog-items EP](../catalog-items/README.md) introduces
`ClusterCatalogItem` and `ComputeInstanceCatalogItem` as a
presentation layer over templates. A catalog item references one
template, defines field overrides (editable/locked with defaults
and JSON Schema validation), and controls tenant visibility via
`published` and `tenant` fields.

### Problems

1. **No composition.** A "Managed Llama 3 70B Inference" offering
   requires a GPU ComputeInstance, networking, vLLM deployment,
   and monitoring. Today, this must be a single monolithic Ansible
   role that does everything. There is no way to compose existing
   catalog items into a higher-level service.

2. **No pricing.** Catalog items carry no cost information.
   Tenants cannot estimate costs before ordering. Cloud providers
   have no mechanism to surface rates from their billing system.
   There is no integration point between the catalog and Koku.

3. **No inheritance.** A tenant admin who wants to offer "Corporate
   ML Workstation" (same as the provider's "ML Workstation" but
   with locked security group and storage tier) must create a new
   catalog item from scratch, duplicating all field definitions.
   There is no way to extend an existing catalog item and narrow
   its constraints.

4. **No billing labels.** When a tenant orders from a catalog item,
   the provisioned resources carry no metadata linking them to the
   catalog offering, pricing tier, or SLA level. Koku has no
   signal to apply differentiated rates per catalog item.

5. **Composition and presentation are entangled.** Without a
   separate orchestration resource, composition logic (DAG,
   dependencies, output wiring) must live inside the catalog item
   — mixing what the tenant sees with how provisioning works.

### Comparison with VMware Cloud Director vApp Model

VMware Cloud Director (VCD) offers **vApps** — groups of VMs
deployed as a single unit with shared networking. vApps are the
primary mechanism for composing multi-VM services in VCD and a
key feature for CSPs migrating to OSAC.

OSAC Blueprints are the cloud-native successor to the vApp
model, with significant advantages:

| Dimension | VCD vApp | OSAC Blueprint |
|---|---|---|
| Resource types | VMs only | Any catalog item (VMs, clusters, networks, storage, AI inference, …) |
| Composition model | Flat list of VMs with shared network | DAG with dependency ordering and output wiring |
| Parameterization | OVF properties (string-only, no validation) | Typed parameters with JSON Schema validation |
| Cross-service wiring | Manual (user copies IPs between VMs) | Expression language (`{{ network.outputs.subnet_id }}`) |
| Cost visibility | None at vApp level | Koku-sourced cost preview with per-child breakdown |
| Inheritance | None (copy entire vApp template) | Extend parent, narrow constraints, accumulate billing labels |
| Tenant authoring | Limited to vApp template upload | Tenant blueprint authors compose from published catalog items |
| Provisioning | vSphere-only (VM placement + networking) | Pluggable via AAP Workflow Job Templates |
| Lifecycle | Start/stop/suspend the group | Order lifecycle with individual resource status tracking |

For CSPs migrating from VCD, Blueprints cover the vApp use
case (grouped VM deployment) while extending far beyond it —
enabling true service composition across compute, networking,
storage, and application tiers. A "Managed Database" Blueprint
that provisions a VM, attaches it to a private network, creates
a storage volume, and configures the database is something VCD
cannot express as a single deployable unit.

### User Stories

- As a **provider engineer**, I want to define a blueprint that
  composes GPU compute, networking, and vLLM serving into a
  reusable service definition, so that multiple catalog items
  can present it to different audiences with different
  constraints.

- As a **cloud provider**, I want tenants to see estimated costs
  from my Koku cost model before ordering, without maintaining
  a second copy of pricing data in the catalog.

- As a **tenant blueprint author**, I want to compose published
  catalog items into a custom service definition for my
  organization — without access to templates or provider-internal
  blueprints — and visually wire them together in a workbench UI.

- As a **tenant admin**, I want to create an organization-specific
  variant of a provider's catalog item with locked security
  settings and a specific model, so that my team can order from
  a curated catalog without needing provider admin involvement.

- As a **tenant**, I want to see a cost breakdown before ordering
  a composed service (e.g., "GPU compute: $10/hr + inference
  tokens: $0.01/1K input") so that I can make informed
  provisioning decisions.

- As a **FinOps engineer**, I want OSAC-provisioned resources to
  carry billing labels from the catalog item they were ordered
  from, so that Koku can apply differentiated rates per catalog
  offering, model, and SLA tier.

- As an **operator**, I want OSAC to work without Koku deployed,
  with cost preview gracefully unavailable rather than the
  catalog being broken.

### Goals

- Introduce Blueprint as a separate resource type for service
  composition, distinct from CatalogItem presentation.
- Add billing labels that propagate to provisioned resources and
  are consumable by Koku.
- Source pricing from Koku's cost model API at query time — no
  pricing data stored in OSAC.
- Treat Koku as an optional capability with clean degradation.
- Enable catalog item inheritance with monotonically narrowing
  field constraints.
- Provide a cost preview endpoint that queries Koku and returns
  estimated costs with per-child breakdown.
- Support tenant-authored blueprints (compose from published
  catalog items) and tenant-authored catalog items (inherit and
  narrow from published items).
- Define API endpoints that support a visual blueprint workbench
  UI (validate, estimate, contract introspection).

### Non-Goals

- Implementing a billing service or invoice generation — that
  is Koku's responsibility.
- Storing or caching pricing data in OSAC — Koku is the single
  source of truth for rates.
- Real-time metering or usage-based billing within OSAC — OSAC
  sets billing labels; Koku meters via Prometheus.
- Implementing the blueprint workbench UI — this EP defines the
  API surface the workbench consumes; the UI is a separate
  concern.
- Changes to the existing flat catalog item model — inheritance,
  billing labels, and blueprint references are additive
  extensions; flat catalog items continue to work unchanged.

## Proposal

### Personas

Four personas interact with blueprints and catalog items,
with distinct privileges:

| Persona | Can create Templates | Can create Blueprints | Can create CatalogItems | Constraints |
|---|---|---|---|---|
| **Provider engineer** | Yes (via Ansible roles) | Yes (any resource type) | Yes (global) | None |
| **Cloud provider admin** | No | Yes (any resource type) | Yes (global or tenant-scoped) | None |
| **Tenant blueprint author** | No | Yes (scoped to org) | No | Can only reference published CatalogItems, not Templates or provider Blueprints directly |
| **Tenant admin** | No | No | Yes (scoped to org) | Can only inherit from published CatalogItems; can only narrow, never widen |

The **tenant blueprint author** is a new role distinct from
tenant admin. Tenant admins customize existing offerings via
catalog item inheritance (lock fields, narrow ranges). Tenant
blueprint authors compose new services from published building
blocks. Both are scoped to their organization. Neither has
access to templates or provider-internal blueprints.

### Blueprint resource type

A Blueprint defines a DAG of resources to provision as a unit.
Each node in the DAG references a catalog item (which in turn
references a template or another blueprint). The Blueprint
declares parameters, dependencies, and output wiring.

```proto
message Blueprint {
  string id = 1;
  Metadata metadata = 2;
  string title = 3;
  string description = 4;
  string tenant = 5;           // empty = provider-global
  map<string, BlueprintNode> resources = 6;
  map<string, ParameterDefinition> parameters = 7;
  map<string, string> billing_labels = 8;
}

message BlueprintNode {
  string catalog_item = 1;
  repeated string depends_on = 2;
  map<string, string> properties = 3;
}

message ParameterDefinition {
  string type = 1;              // string, integer, boolean
  string description = 2;
  bool required = 3;
  google.protobuf.Value default = 4;
  google.protobuf.Struct validation_schema = 5;
}
```

#### Blueprint example

```yaml
id: "bp-maas-llama3-70b"
title: "MaaS Llama 3 70B"
description: >
  GPU compute with vLLM serving Llama 3 70B.
  Includes dedicated subnet and monitoring.

parameters:
  name:
    type: string
    required: true
    description: Name for the inference service
  replicas:
    type: integer
    default: 1
    validation_schema: { minimum: 1, maximum: 8 }
  sla_tier:
    type: string
    default: standard
    validation_schema: { enum: ["standard", "premium"] }

resources:
  network:
    catalog_item: "inference-subnet"
    properties:
      name: "{{ parameters.name }}-net"

  gpu-compute:
    catalog_item: "gpu-a100-4-vm"
    depends_on: [network]
    properties:
      computeInstanceClass: "gpu-a100-4-vm"
      subnet: "{{ network.outputs.subnet_id }}"
      image: "rhel-9.6-vllm"
      replicas: "{{ parameters.replicas }}"

  monitoring:
    catalog_item: "prometheus-stack"
    depends_on: [gpu-compute]
    properties:
      targets: "{{ gpu-compute.outputs.instance_ids }}"

billing_labels:
  osac.io/service-type: "maas"
  osac.io/model: "llama-3-70b"
```

#### Expression language

Property values support a template expression syntax for
referencing blueprint parameters and node outputs:

| Expression | Resolves to |
|---|---|
| `{{ parameters.name }}` | Value of the `name` parameter provided at order time |
| `{{ network.outputs.subnet_id }}` | The `subnet_id` output from the `network` node |
| `{{ gpu-compute.outputs.instance_ids }}` | List output from `gpu-compute` |

Expressions support property access only — no function calls,
no arithmetic, no conditionals. This keeps the language simple
and avoids injection risks. The expression evaluator is
validated at blueprint creation time — unknown references are
rejected.

Conditional resources are not supported in the initial
implementation. All declared resources are provisioned.
Conditional composition is deferred to a future EP.

#### Blueprint depth and resource limits

- Maximum nesting depth: **3 levels** (a blueprint can
  reference catalog items backed by other blueprints).
  Configurable per deployment.
- Maximum total resources after graph expansion: **50**.
  Configurable per deployment.
- Both are validated at blueprint creation time.

#### Tenant blueprint authoring

Tenant blueprint authors can create blueprints scoped to their
organization under two constraints:

1. **Can only reference published catalog items.** Tenant
   blueprints compose from catalog items visible to the
   tenant's organization. They cannot reference templates
   directly or provider-internal blueprints. This ensures
   tenants build from sanctioned building blocks.

2. **Subject to provider-set limits.** Maximum depth, maximum
   resources, and any other deployment-level limits apply.

A tenant blueprint is invisible outside its organization.

#### Relationship to Templates

Templates and Blueprints are both "things that provision."
They differ in scope:

| | Template | Blueprint |
|---|---|---|
| Scope | Atomic (one Ansible role) | Composed (DAG of catalog items) |
| Author | Provider engineer (Ansible) | Provider engineer or tenant blueprint author |
| Execution | Single AAP Job Template | Generated AAP Workflow Job Template |
| Output contract | `meta/osac_contract.yaml` | Derived from children's contracts |
| Tenant-authorable | No | Yes (from published catalog items) |

A Blueprint's output contract is the union of its leaf nodes'
outputs. The blueprint workbench UI surfaces these for wiring.

### Catalog item enhancements

#### Blueprint reference

A catalog item can reference either a template (existing
behavior) or a blueprint:

```proto
message ComputeInstanceCatalogItem {
  // ... existing fields ...
  string template = 5;       // existing: references a Template
  string blueprint = 6;     // new: references a Blueprint (mutually exclusive with template)
  string parent = 7;
  map<string, string> billing_labels = 8;
}
```

When a catalog item references a blueprint, the catalog item's
`fields` definitions control which blueprint parameters the
tenant can configure. Non-editable fields are locked to
defaults; editable fields are exposed in the ordering form.

This means the same blueprint can back multiple catalog items
with different exposed parameters:

```
Blueprint "bp-maas-llama3-70b"
  ├── CatalogItem "MaaS Llama 3 70B — Standard" (sla_tier locked to "standard")
  ├── CatalogItem "MaaS Llama 3 70B — Premium"  (sla_tier locked to "premium")
  └── CatalogItem "MaaS Llama 3 70B — Custom"   (sla_tier editable)
```

#### Billing labels

A `billing_labels` map on catalog items declares labels
applied to all resources provisioned from the catalog item:

```yaml
id: "maas-llama3-70b-standard"
title: "Managed Llama 3 70B — Standard SLA"
blueprint: "bp-maas-llama3-70b"
billing_labels:
  osac.io/catalog-item: "maas-llama3-70b-standard"
  osac.io/sla-tier: "standard"
```

When provisioning, billing labels are resolved by merging:
1. Blueprint billing labels (from the blueprint definition)
2. Catalog item billing labels (from the catalog item and its
   inheritance chain)

Child catalog item labels override parent labels for the same
key. Catalog item labels override blueprint labels for the
same key.

The fulfillment-service applies resolved billing labels as:
- Kubernetes annotations on the resource CRD
  (`osac.io/billing-*` prefix)
- Labels in the fulfillment-service database record

These propagate to Prometheus metrics via ServiceMonitor
relabeling, making them available to Koku through the
[cost-metric-mapping EP](../cost-metric-mapping/README.md).

Billing labels are the key that connects OSAC resources to
Koku's cost model:

```
Koku cost model:
  tag: osac.io/service-type = "maas"
  metric: inference_input_token_cost_per_token = 0.01
  metric: inference_output_token_cost_per_token = 0.03

  tag: osac.io/sla-tier = "standard"
  metric: sla_discount_markup = 0%

  tag: osac.io/sla-tier = "premium"
  metric: sla_discount_markup = -20%
```

#### Inheritance

A catalog item can extend a parent via a `parent` field.

**Inheritance rules:**

1. **`template` or `blueprint` is inherited.** Only the root
   catalog item sets the backing reference. Children inherit it.

2. **`fields` merge with narrowing.** Child field definitions
   merge with parent's:
   - A child can add new field definitions.
   - A child **cannot** set `editable: true` on a field the
     parent set to `editable: false`. Editability only narrows.
   - A child can narrow a validation schema (reduce a range,
     remove enum values) but **cannot** widen it.
   - A child can set a `default` or lock an editable field.

3. **`billing_labels` accumulate.** Child labels merge with
   parent labels. Child values override parent values for the
   same key.

4. **`published` and `tenant` are independent.** A child can be
   published even if the parent is unpublished. A tenant-scoped
   child can extend a global parent.

5. **Depth capped at 4 levels** (configurable per deployment).

**Tenant authoring constraints for catalog items:**

1. Must extend a published provider catalog item (or another
   item within the same tenant). Cannot create root items with
   direct template/blueprint references.

2. Can only narrow, never widen. Cannot unlock parent-locked
   fields, widen validation ranges, or remove billing labels.

**Example: three-layer inheritance**

```
Layer 0: Blueprint "bp-maas-llama3-70b" (orchestration)

Layer 1: CatalogItem "MaaS Llama 3 70B" (cloud provider)
  blueprint: "bp-maas-llama3-70b"
  fields:
    - path: "parameters.replicas"
      editable: true
      validation_schema: { minimum: 1, maximum: 8 }
    - path: "parameters.sla_tier"
      editable: true
      validation_schema: { enum: ["standard", "premium"] }
  billing_labels:
    osac.io/service-type: "maas"
    osac.io/model: "llama-3-70b"

Layer 2: CatalogItem "Llama 3 70B Standard" (tenant admin)
  parent: "maas-llama3-70b"
  tenant: "acme-corp"
  fields:
    - path: "parameters.sla_tier"
      editable: false
      default: "standard"                  # locked
    - path: "parameters.replicas"
      editable: true
      validation_schema: { minimum: 1, maximum: 4 }  # narrowed
  billing_labels:
    osac.io/sla-tier: "standard"

Resolved billing labels:
  osac.io/service-type: "maas"       (from layer 1)
  osac.io/model: "llama-3-70b"       (from layer 1)
  osac.io/sla-tier: "standard"       (from layer 2)
```

### Koku as an optional capability

Koku integration is modeled as a capability that the
fulfillment-service discovers at startup and re-evaluates
periodically.

```proto
message CapabilityStatus {
  string name = 1;         // e.g., "cost-management"
  string state = 2;        // "available", "unavailable", "degraded"
  string endpoint = 3;
  string message = 4;
  google.protobuf.Timestamp last_checked = 5;
}
```

#### Discovery

1. **Configuration:** Optional `KOKU_API_URL` environment
   variable. When absent, cost-management is `unavailable`.

2. **Health probe:** When configured, the fulfillment-service
   calls `GET /api/cost-management/v1/status/` on startup and
   every 5 minutes. Success → `available`. Failure → `unavailable`.

3. **Capability API:**
   ```
   GET /api/fulfillment/v1/capabilities
   → { "capabilities": [
         { "name": "cost-management", "state": "available", ... }
       ] }
   ```

#### Degradation behavior

| State | Cost preview | Catalog listing | Ordering | Workbench |
|---|---|---|---|---|
| `available` | Returns estimates | Shows pricing | Normal | Estimate button enabled |
| `unavailable` | Returns `pricing_available: false` | No pricing | Normal | Estimate button grayed out |
| `degraded` | Returns `pricing_available: false` with message | No pricing | Normal | Estimate button grayed out |

Ordering never depends on Koku. Cost preview is a convenience,
not a gate.

### Cost preview (Koku-sourced)

A cost preview endpoint queries Koku at request time. OSAC
does not store pricing.

```
POST /api/fulfillment/v1/catalog_items/{id}/estimate
{
  "parameters": {
    "name": "my-inference",
    "replicas": 4,
    "sla_tier": "standard"
  }
}
```

The fulfillment-service:

1. Resolves billing labels from the catalog item chain and its
   backing blueprint (if any).
2. Checks cost-management capability. If `unavailable`, returns
   `pricing_available: false`.
3. Calls Koku's cost model API with resolved billing labels.
4. Computes the estimate from rates and resolved parameters.
5. Returns the response:

```json
{
  "pricing_available": true,
  "estimated_rate": 43.00,
  "unit": "hour",
  "currency": "USD",
  "breakdown": [
    {
      "resource": "network",
      "catalog_item": "inference-subnet",
      "billing_labels": { "osac.io/service-type": "networking" },
      "rate": 1.00,
      "unit": "hour"
    },
    {
      "resource": "gpu-compute",
      "catalog_item": "gpu-a100-4-vm",
      "billing_labels": { "osac.io/service-type": "compute" },
      "rate": 40.00,
      "unit": "hour",
      "note": "4 replicas × $10.00/hr"
    },
    {
      "resource": "monitoring",
      "catalog_item": "prometheus-stack",
      "billing_labels": { "osac.io/service-type": "monitoring" },
      "rate": 2.00,
      "unit": "hour"
    }
  ],
  "disclaimer": "Estimate based on current rates. Actual costs are determined by metered usage."
}
```

When Koku is unavailable:

```json
{
  "pricing_available": false,
  "message": "Cost management service is not configured.",
  "breakdown": []
}
```

### Blueprint Workbench

The blueprint workbench is a visual authoring environment for
building blueprints. This EP defines the API surface the
workbench consumes; the UI implementation is a separate concern.

#### Workbench capabilities

**Canvas view.** Drag catalog items from a sidebar onto a
canvas. Each node shows the catalog item's title, its declared
parameters (from the catalog item's field definitions), and its
declared outputs (from the backing template's
`meta/osac_contract.yaml`). Draw dependency edges between nodes.

**Property binding.** Click an edge to see the upstream node's
outputs and the downstream node's parameters. Bind visually:
drag `subnet_id` from the network node's output panel to the
`subnet` parameter on the gpu-compute node. The UI generates
the `{{ network.outputs.subnet_id }}` expression.

**Parameter exposure.** The author decides which node parameters
to expose as top-level blueprint parameters (configurable at
order time) vs. which to hardcode. Drag a node parameter up to
the "Blueprint Parameters" panel to expose it; type a value
inline to lock it.

**Live validation.** The workbench validates the DAG in real
time:
- Cycle detection (reject edges that create cycles)
- Missing required bindings (warn on unconnected required
  inputs)
- Output/input type mismatches (warn if a string output
  connects to an integer input)
- Depth and resource limit checks
- Contract compliance (does the child's output contract declare
  the output being referenced?)

**Cost preview.** An "Estimate Cost" button calls the cost
preview endpoint with the current parameter values. Shows a
live breakdown. When Koku is unavailable, the button is grayed
out with a tooltip explaining why.

**Dry-run validation.** A "Validate" button resolves all
expressions and checks the full graph without provisioning.
Reports which nodes would execute, in what order, with what
parameters.

#### API endpoints supporting the workbench

| Endpoint | Workbench use |
|---|---|
| `POST /api/fulfillment/v1/blueprints/validate` | Dry-run graph validation (cycles, depth, output refs) |
| `POST /api/fulfillment/v1/catalog_items/{id}/estimate` | Live cost preview |
| `GET /api/fulfillment/v1/capabilities` | Determine if cost preview is available |
| `GET /api/fulfillment/v1/catalog_items` | Populate the sidebar with available building blocks |
| `GET /api/fulfillment/v1/catalog_items/{id}/contract` | Retrieve the output contract for a catalog item's backing template/blueprint |

The `/contract` endpoint returns the declared inputs and
outputs from `meta/osac_contract.yaml` (for template-backed
items) or the derived output contract (for blueprint-backed
items). This is what populates the "available outputs" and
"required inputs" panels in the workbench.

### Workflow Description

#### Ordering a flat (non-composed) catalog item

No change from the existing
[catalog-items EP](../catalog-items/README.md). The tenant
creates a resource (Cluster, ComputeInstance) referencing the
catalog item. The fulfillment-service validates fields, resolves
inheritance, applies billing labels, and stores the resource.
The osac-operator triggers the template role in AAP.

#### Ordering a blueprint-backed catalog item

1. **Tenant creates an order** via the Order API, specifying
   the catalog item and parameter values.

2. **Fulfillment-service validates the order:**
   - Catalog item exists and is visible to the tenant.
   - Resolves inheritance (field definitions, billing labels).
   - Resolves the backing blueprint.
   - Validates parameters against the catalog item's field
     definitions (which constrain the blueprint's parameters).
   - Collects billing labels from the full chain (blueprint +
     catalog item inheritance).

3. **Fulfillment-service creates an Order record** in PostgreSQL
   with status `PENDING` and an Order CRD on the hub cluster.

4. **osac-operator detects the Order CRD** and triggers AAP with
   the `osac.service.execute_blueprint` role.

5. **`execute_blueprint` generates an AAP Workflow Job
   Template** from the blueprint's resource graph:
   - Topologically sorts nodes.
   - Creates one workflow node per blueprint resource.
   - Maps dependency edges to workflow success links.
   - Independent nodes execute in parallel.
   - Populates `extra_data` with resolved properties and AAP
     artifact expressions for output wiring.
   - Adds failure path nodes running `delete.yaml` in reverse
     order.

6. **AAP executes the workflow.** Outputs flow between nodes
   via `set_stats` artifacts (per the output contract from the
   [inventory-provisioning-separation EP](../inventory-provisioning-separation/README.md)).

7. **On success:** Order status moves to `READY`.

8. **On failure:** failure paths clean up completed nodes.
   Order status moves to `FAILED` with per-resource detail.

```
Blueprint graph:                  AAP Workflow Job Template:

  network                        ┌──────────────┐
    │                            │ network role │
    ├── gpu-compute              └──────┬───────┘
    │     │                             │ success
    │     └── monitoring         ┌──────▼───────┐
    │                            │ gpu-compute  │
                                 └──────┬───────┘
                                        │ success
                                 ┌──────▼───────┐
                                 │ monitoring   │
                                 └──────────────┘
```

#### Output wiring between nodes

| Blueprint expression | AAP artifact expression |
|---|---|
| `{{ network.outputs.subnet_id }}` | `{{ workflow.nodes.network.artifacts.subnet_id }}` |
| `{{ gpu-compute.outputs.instance_id }}` | `{{ workflow.nodes.gpu-compute.artifacts.instance_id }}` |

The workflow generator validates at blueprint creation time
that output references match declared outputs in each
referenced catalog item's contract.

### API Extensions

#### New resources

| Resource | API path | Methods |
|---|---|---|
| Blueprint | `/api/fulfillment/v1/blueprints` | GET (list), POST (create) |
| Blueprint | `/api/fulfillment/v1/blueprints/{id}` | GET, PATCH, DELETE |
| Order | `/api/fulfillment/v1/orders` | GET (list), POST (create) |
| Order | `/api/fulfillment/v1/orders/{id}` | GET, DELETE (cancel) |

```proto
message Order {
  string id = 1;
  Metadata metadata = 2;
  OrderSpec spec = 3;
  OrderStatus status = 4;
}

message OrderSpec {
  string catalog_item = 1;
  map<string, google.protobuf.Value> parameters = 2;
}

message OrderStatus {
  string state = 1;    // PENDING, PROVISIONING, READY, FAILED, DELETING
  string message = 2;
  repeated OrderResourceStatus resources = 3;
}

message OrderResourceStatus {
  string name = 1;
  string resource_id = 2;
  string resource_type = 3;
  string state = 4;
}
```

For flat (template-backed) catalog items, tenants continue to
create resources directly — the Order API is for
blueprint-backed items.

#### New endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/api/fulfillment/v1/blueprints/validate` | Dry-run blueprint graph validation |
| POST | `/api/fulfillment/v1/catalog_items/{id}/estimate` | Cost preview (queries Koku) |
| GET | `/api/fulfillment/v1/catalog_items/{id}/contract` | Output contract introspection |
| GET | `/api/fulfillment/v1/capabilities` | Capability status (cost-management, etc.) |

#### Modified resources

| Resource | Change |
|---|---|
| ComputeInstanceCatalogItem | Add `blueprint`, `parent`, `billing_labels` fields |
| ClusterCatalogItem | Add `blueprint`, `parent`, `billing_labels` fields |
| ComputeInstance | Add billing labels as CRD annotations |
| Cluster | Add billing labels as CRD annotations |

#### New CRDs

| CRD | Description |
|---|---|
| `Order` (`osac.openshift.io/v1alpha1`) | Watched by OrderController |

### Implementation Details/Notes/Constraints

#### Koku API integration

```go
type CostService interface {
    Available() bool
    GetRates(ctx context.Context, labels map[string]string) ([]Rate, error)
}
```

Two implementations:
- `KokuCostService` — queries Koku's cost model API. 3-second
  timeout.
- `NoOpCostService` — returned when `KOKU_API_URL` is absent.
  `Available()` returns false. All callers handle this.

The `CostService` is injected at startup. Only the cost preview
endpoint depends on it.

#### Database changes

New tables:
- `blueprints` — stores Blueprint records
- `orders` — stores Order records

Modified tables:
- `cluster_catalog_items` — `data` jsonb adds `blueprint`,
  `parent`, `billing_labels`
- `compute_instance_catalog_items` — same

#### Proto changes

New types: `Blueprint`, `BlueprintNode`,
`ParameterDefinition`, `Order`, `OrderSpec`, `OrderStatus`,
`OrderResourceStatus`, `CostEstimate`, `CostBreakdownEntry`,
`CapabilityStatus`.

New services: `BlueprintsService` (CRUD + validate),
`OrdersService` (Create, Get, List, Delete).

Modified types: `ComputeInstanceCatalogItem` and
`ClusterCatalogItem` add `blueprint`, `parent`,
`billing_labels`.

#### Validation at creation time

**For blueprints:**
- Resource graph is a valid DAG (no cycles).
- All referenced catalog items exist and are published (or
  within the same tenant for tenant blueprints).
- Output references in expressions match declared contracts.
- Depth and resource count within limits.
- Tenant blueprints only reference catalog items, not templates.

**For catalog items with inheritance:**
- Parent exists and is visible.
- Depth within limit.
- Field narrowing is valid (no widening).
- No circular chains.

#### AAP changes

New roles in `osac.service`:
- `execute_blueprint` — generates and launches AAP Workflow Job
  Templates from blueprint graphs.

New filter plugins:
- `resolve_graph` — topological sort and expression resolution.

#### osac-operator changes

New controller:
- `OrderController` — watches Order CRDs, triggers
  `execute_blueprint`, manages lifecycle.

Modified behavior:
- Applies billing labels as CRD annotations during resource
  creation.

#### Billing label propagation

1. Fulfillment-service resolves labels from blueprint +
   catalog item inheritance chain.
2. Stores as CRD annotations (`osac.io/billing-*`).
3. ServiceMonitor relabeling exposes as Prometheus labels.
4. Cost-metric-mapping ConfigMap references in recording rules.
5. Koku applies tag-based rates.

### Risks and Mitigations

#### Koku API latency

*Mitigation:* 3-second timeout. Returns
`pricing_available: false` on timeout.

#### Koku API version drift

*Mitigation:* `CostService` interface isolates API details.
Health probe validates compatibility.

#### Workflow generation latency

*Mitigation:* Cache workflow templates per blueprint. Regenerate
on blueprint update.

#### Artifact passing limits

*Mitigation:* Document size constraints. Recommend references
over enumeration for large outputs.

#### Partial failure and rollback

*Mitigation:* Failure path nodes run `delete.yaml` in reverse
order. Order status lists per-resource state for manual cleanup
if rollback fails.

#### Expression injection

*Mitigation:* Restricted evaluator — property access only. No
functions, filters, arithmetic, or Ansible fact access. Validated
at blueprint creation time.

#### Inheritance validation complexity

*Mitigation:* Subset checking for common constraint types only
(minimum/maximum, enum, minLength/maxLength). Warning for
complex schemas.

#### Blueprint sprawl

*Mitigation:* Tenant blueprints scoped to organization.
Provider blueprints can be unpublished. Blueprint listing
supports filtering by tenant, published state, and labels.

### Drawbacks

- Two new resource types (Blueprint, Order) increase the API
  surface.
- Providers must understand the Blueprint/CatalogItem separation
  — an additional concept beyond flat catalog items.
- AAP Workflow Job Template debugging requires AAP familiarity.
- Billing label propagation requires Prometheus
  ServiceMonitor configuration outside OSAC's control.
- Koku is a runtime dependency for cost preview (but not for
  ordering).
- The workbench UI, while not part of this EP, is significant
  development effort and is the primary authoring experience
  for tenant blueprint authors.

## Alternatives (Not Implemented)

### Embed composition in catalog items

Put the resource DAG directly in the catalog item (no separate
Blueprint resource).

*Why not:* Mixes orchestration with presentation. A single
blueprint should back multiple catalog items with different
parameter constraints. Embedding the DAG in each catalog item
duplicates the graph and prevents reuse.

### Store pricing on catalog items

Attach rate/unit/currency directly to catalog items.

*Why not:* Creates a second source of truth alongside Koku.
Pricing inevitably drifts. Querying Koku at request time
eliminates the sync problem.

### Full DAG of raw resource types

Blueprint nodes reference raw resource types (subnet, instance)
instead of catalog items.

*Why not:* OSAC uses opaque Ansible roles. A raw resource DAG
requires a generic resource executor — a fundamental
architecture change.

### Monolithic Ansible roles

Write one role per service stack.

*Why not:* Doesn't scale. No reuse. Adding a variant requires
a new role.

### Pre-built Workflow Job Templates

Generate workflows at blueprint creation time, not order time.

*Why not:* Parameters are tenant-specific and vary per order.
Per-order generation allows full resolution.

### Pricing in ComputeInstanceClass

*Why not:* Same class, different catalog items, different
rates. Pricing belongs in the billing system.

## Open Questions [optional]

1. **Order vs. direct creation.** Should blueprint-backed orders
   use the Order resource, or extend the existing
   ComputeInstance/Cluster creation flow?

2. **Cross-resource-type blueprints.** Should a blueprint mix
   compute and networking catalog items? How do field
   definitions work across types?

3. **Version pinning.** Should blueprint node references pin
   catalog item versions (e.g., `gpu-a100-4-vm@v2`)?

4. **Koku query interface.** Can Koku return rates filtered by
   tag? Coordinate with Koku team.

5. **Workflow caching.** When to invalidate cached workflow
   templates?

6. **Capability extensibility.** Generalize the capability model
   for other optional services, or keep it cost-management-only
   for now?

7. **Workbench persistence.** Should the workbench save
   in-progress blueprints as drafts, or only persist on
   explicit "Create"?

## Test Plan

TBD — will cover:
- Blueprint CRUD with graph validation
- Catalog item with `blueprint` reference
- Catalog item inheritance with narrowing
- Billing label accumulation (blueprint + catalog item chain)
- Blueprint validate endpoint (dry-run)
- Cost preview with Koku available / unavailable / timeout
- Capabilities endpoint
- Order lifecycle (success and failure with rollback)
- AAP Workflow generation from blueprint graph
- Output wiring via `set_stats`
- Tenant blueprint authoring (can only use published catalog
  items)
- Tenant catalog item authoring (can only narrow)
- Contract introspection endpoint

## Graduation Criteria

TBD

## Upgrade / Downgrade Strategy

**Upgrade:** New `blueprints` and `orders` tables created. New
fields on catalog items (`blueprint`, `parent`,
`billing_labels`) added to `data` jsonb. Existing flat catalog
items unaffected. Order CRD installed. osac-operator updated
with OrderController before fulfillment-service update.

**Downgrade:** New fields preserved in jsonb, ignored by older
code. Blueprint-backed catalog items and Orders inaccessible
to older code. Flat catalog items continue to work.

## Version Skew Strategy

Order CRD requires operator update before fulfillment-service.
Catalog item inheritance and billing labels are
fulfillment-service-only. Operator should be updated before
fulfillment-service starts writing billing label annotations.

## Support Procedures

- **Order stuck in PROVISIONING:** Check AAP Workflow Job
  status. Identify stuck node. Check job output.

- **Partial failure with orphans:** Order status lists
  per-resource states. Manually delete uncleaned resources.

- **Cost preview unavailable:** Check
  `GET /api/fulfillment/v1/capabilities`. Verify `KOKU_API_URL`
  and Koku reachability.

- **Unexpected cost estimates:** Estimates reflect Koku's
  current cost model. Inspect Koku cost model for matching
  tag-based rates.

- **Missing billing labels in Koku:** Check CRD annotations,
  ServiceMonitor relabeling, cost-metric-mapping ConfigMap.

- **Inheritance validation failure:** Error identifies the
  field violating narrowing constraints.

- **Blueprint validation failure:** Error identifies cycles,
  missing output references, or exceeded limits.

## Infrastructure Needed [optional]

None beyond existing OSAC infrastructure and AAP. Koku is
optional.
