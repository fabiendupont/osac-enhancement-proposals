---
title: cost-metric-mapping
authors:
  - Fabien Dupont
creation-date: 2026-04-29
last-updated: 2026-04-29
tracking-link:
  - TBD
see-also:
  - enhancements/osac-addon/README.md
  - enhancements/unified-compute-model/README.md
  - enhancements/inventory-provisioning-separation/README.md
  - enhancements/catalog-items/README.md
  - enhancements/composable-catalog-items/README.md
replaces:
superseded-by:
---

# Cost Metric Mapping

## Summary

Introduce a ConfigMap-driven mapping layer that translates
provider-native Prometheus metrics into Koku's existing cost
dimensions. Instead of defining OSAC-specific metrics, each
provider exports its own native telemetry and a per-provider
ConfigMap declares how those metrics map to Koku dimensions
(cpu_core_hours, memory_gb_hours, network_ingress_gb, etc.).
A lightweight controller reads these ConfigMaps and generates
Prometheus recording rules that produce the aggregated series
Koku already knows how to consume.

## Motivation

### Current state

OSAC provisions compute instances, networks, and storage across
multiple backends (KubeVirt, Metal3, NICo, ESI). Each backend
has its own telemetry:

- **KubeVirt**: exposes `kubevirt_vmi_*` Prometheus metrics
  natively
- **Metal3/Ironic**: exposes `ironic_*` metrics via the Ironic
  Prometheus exporter
- **NICo**: exposes `netris_*` metrics via its monitoring agent
- **Netris** (networking): exposes `netris_port_*` and
  `netris_switch_*` metrics

None of these feed into a billing system today. Project Koku
already consumes Prometheus/Thanos data for OpenShift cost
management and has an established set of cost dimensions
(cpu_core_hours, memory_gb_hours, storage_gb_hours,
network_ingress_gb, network_egress_gb, gpu_hours). Koku does
not understand provider-specific metric names.

### Problems

1. **No cost visibility.** OSAC resources are provisioned and
   consumed but there is no per-tenant cost attribution. Cloud
   providers cannot bill tenants for resource usage.

2. **Provider-specific metrics are fragmented.** Each backend
   emits metrics under its own namespace with its own label
   conventions. A central billing system would need custom
   integration code for every backend.

3. **Defining OSAC-specific metrics creates overhead.** Creating
   an `osac_*` metric namespace would require every provider to
   dual-export (native metrics for operations, OSAC metrics for
   billing) and would introduce cardinality pressure from
   relabeling.

### User Stories

- As a **cloud provider**, I want to see per-tenant cost
  breakdowns for OSAC-provisioned resources so that I can bill
  tenants accurately.

- As a **cloud provider**, I want to add a new provisioning
  backend (e.g., NICo) and have its costs appear in Koku
  without writing Koku integration code — only a ConfigMap
  describing how its native metrics map to cost dimensions.

- As an **infrastructure operator**, I want to reuse existing
  Prometheus metrics from my backend (KubeVirt, Ironic, Netris)
  for billing without deploying additional exporters or
  maintaining a separate metrics pipeline.

- As a **FinOps engineer**, I want OSAC resource costs to appear
  alongside OpenShift cluster costs in the same Koku dashboard,
  using the same dimensions and filters I already use.

- As a **tenant**, I want to see my compute, network, and
  storage costs broken down by resource type and time period so
  that I can optimize my usage.

### Goals

- Define a ConfigMap schema for mapping provider-native
  Prometheus metrics to Koku cost dimensions.
- Generate Prometheus recording rules from these ConfigMaps so
  that cost aggregation happens at the Prometheus level, not at
  query time.
- Support all OSAC resource types: compute (cpu, memory, gpu),
  network (ingress, egress, bandwidth), and storage (capacity,
  IOPS).
- Inject OSAC multi-tenancy labels (tenant_id, organization,
  compute_instance_class, site) into recorded metrics so Koku
  can attribute costs per tenant.
- Enable provider onboarding via ConfigMap only — no OSAC code
  changes required to add a new backend's cost mapping.

### Non-Goals

- Defining pricing tiers or rate cards — that is a Koku/provider
  concern outside OSAC.
- Implementing a billing service or invoice generation within
  OSAC.
- Replacing or modifying Koku's internal cost model — OSAC
  produces metrics that Koku already understands.
- Requiring providers to deploy additional Prometheus exporters
  — providers use their existing exporters.
- Real-time billing or metering — cost aggregation operates on
  Prometheus recording rule intervals (typically 1-5 minutes)
  and Koku ingestion cycles.

## Proposal

### ConfigMap schema

Each provider deploys a ConfigMap in the `osac-system` namespace
that declares how its native Prometheus metrics map to Koku cost
dimensions:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: osac-cost-mapping-kubevirt
  namespace: osac-system
  labels:
    osac.io/cost-mapping: "true"
data:
  mapping.yaml: |
    provider: kubevirt
    resourceType: compute
    mappings:
      - dimension: cpu_core_hours
        sourceMetric: kubevirt_vmi_cpu_usage_seconds_total
        aggregation: rate
        unitConversion:
          from: seconds
          to: hours
          factor: 0.000277778   # 1/3600
        labels:
          tenant_id: "{{ label.osac_tenant_id }}"
          compute_instance_class: "{{ label.flavor }}"
          site: "{{ label.node | site_from_node }}"
          instance_id: "{{ label.name }}"

      - dimension: memory_gb_hours
        sourceMetric: kubevirt_vmi_memory_resident_bytes
        aggregation: avg
        unitConversion:
          from: bytes
          to: gb_hours
          factor: 9.31323e-10   # 1/(1024^3) then /3600 at rule eval
        labels:
          tenant_id: "{{ label.osac_tenant_id }}"
          compute_instance_class: "{{ label.flavor }}"

      - dimension: gpu_hours
        sourceMetric: kubevirt_vmi_gpu_allocation
        aggregation: avg
        unitConversion:
          from: count
          to: hours
          factor: 1
        labels:
          tenant_id: "{{ label.osac_tenant_id }}"
          gpu_model: "{{ label.gpu_model }}"
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: osac-cost-mapping-netris
  namespace: osac-system
  labels:
    osac.io/cost-mapping: "true"
data:
  mapping.yaml: |
    provider: netris
    resourceType: network
    mappings:
      - dimension: network_ingress_gb
        sourceMetric: netris_port_rx_bytes_total
        aggregation: increase
        unitConversion:
          from: bytes
          to: gb
          factor: 9.31323e-10
        labels:
          tenant_id: "{{ annotation.osac_io_tenant_id }}"
          virtual_network: "{{ label.network_name }}"

      - dimension: network_egress_gb
        sourceMetric: netris_port_tx_bytes_total
        aggregation: increase
        unitConversion:
          from: bytes
          to: gb
          factor: 9.31323e-10
        labels:
          tenant_id: "{{ annotation.osac_io_tenant_id }}"
          virtual_network: "{{ label.network_name }}"
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: osac-cost-mapping-ironic
  namespace: osac-system
  labels:
    osac.io/cost-mapping: "true"
data:
  mapping.yaml: |
    provider: ironic
    resourceType: compute
    mappings:
      - dimension: cpu_core_hours
        sourceMetric: ironic_node_cpus
        aggregation: avg
        unitConversion:
          from: count
          to: hours
          factor: 1
        matchCondition: "ironic_node_provision_state == 'active'"
        labels:
          tenant_id: "{{ label.osac_tenant_id }}"
          compute_instance_class: "{{ label.resource_class }}"
          site: "{{ label.conductor_group }}"

      - dimension: memory_gb_hours
        sourceMetric: ironic_node_memory_mb
        aggregation: avg
        unitConversion:
          from: mb
          to: gb_hours
          factor: 0.000976563   # 1/1024
        matchCondition: "ironic_node_provision_state == 'active'"
        labels:
          tenant_id: "{{ label.osac_tenant_id }}"

      - dimension: storage_gb_hours
        sourceMetric: ironic_node_local_gb
        aggregation: avg
        unitConversion:
          from: gb
          to: gb_hours
          factor: 1
        matchCondition: "ironic_node_provision_state == 'active'"
        labels:
          tenant_id: "{{ label.osac_tenant_id }}"
```

### ConfigMap schema fields

| Field | Type | Description |
|---|---|---|
| `provider` | string | Provider identifier (kubevirt, netris, ironic, nico) |
| `resourceType` | string | OSAC resource category: `compute`, `network`, `storage` |
| `mappings[]` | list | One entry per Koku cost dimension |
| `mappings[].dimension` | string | Koku cost dimension name (must match Koku's schema) |
| `mappings[].sourceMetric` | string | Prometheus metric name from the provider's exporter |
| `mappings[].aggregation` | string | Prometheus aggregation: `rate`, `increase`, `avg`, `sum`, `max` |
| `mappings[].unitConversion.from` | string | Source unit (documentation only) |
| `mappings[].unitConversion.to` | string | Target unit (documentation only) |
| `mappings[].unitConversion.factor` | float | Multiplication factor applied in the recording rule |
| `mappings[].matchCondition` | string | Optional PromQL filter expression (e.g., only active nodes) |
| `mappings[].labels` | map | Label mapping from provider labels to Koku labels |

### Koku cost dimensions

The following dimensions are defined, matching Koku's existing
cost model:

| Dimension | Unit | Resource types |
|---|---|---|
| `cpu_core_hours` | core-hours | compute |
| `memory_gb_hours` | GB-hours | compute |
| `gpu_hours` | GPU-hours | compute |
| `storage_gb_hours` | GB-hours | storage |
| `storage_iops` | IOPS | storage |
| `network_ingress_gb` | GB | network |
| `network_egress_gb` | GB | network |

Providers map to whichever dimensions their metrics support.
A bare-metal provider might only produce cpu_core_hours,
memory_gb_hours, and storage_gb_hours. A networking provider
might only produce network_ingress_gb and network_egress_gb.

### Workflow Description

#### Provider onboarding

**cloud provider** is a human user responsible for deploying and
configuring OSAC.

1. The cloud provider deploys their backend (e.g., Metal3) and
   its Prometheus exporter (e.g., Ironic exporter), which emits
   native metrics.

2. The cloud provider creates a ConfigMap with the
   `osac.io/cost-mapping: "true"` label, declaring how native
   metrics map to Koku dimensions.

3. The cost mapping controller detects the ConfigMap, validates
   the schema, and generates a PrometheusRule CR containing the
   corresponding recording rules.

4. Prometheus picks up the new recording rules and begins
   evaluating them, producing `osac_cost:*` recorded series
   with standardized labels.

5. Koku (or Thanos) scrapes the recorded series. The cloud
   provider configures a Koku "OSAC" source pointing at the
   Prometheus/Thanos endpoint.

6. Costs appear in Koku dashboards, attributable by tenant,
   compute_instance_class, site, and resource type.

#### Recording rule generation

The controller translates each ConfigMap mapping entry into a
Prometheus recording rule. For example, the KubeVirt
cpu_core_hours mapping produces:

```yaml
groups:
  - name: osac_cost_kubevirt
    interval: 5m
    rules:
      - record: osac_cost:cpu_core_hours:rate5m
        expr: |
          sum by (tenant_id, compute_instance_class, site) (
            rate(kubevirt_vmi_cpu_usage_seconds_total[5m])
            * 0.000277778
          )
```

The `osac_cost:` prefix is a recording rule namespace, not a
new metric exported by any component. These rules aggregate
provider metrics into Koku-consumable series.

#### Metric flow

```
Provider Exporter          Prometheus              Koku/Thanos
  |                          |                        |
  | kubevirt_vmi_cpu_*       |                        |
  | netris_port_rx_*         |                        |
  | ironic_node_*            |                        |
  |--- native metrics ------>|                        |
                             |                        |
                             | recording rules        |
                             | (from ConfigMap)        |
                             |                        |
                             | osac_cost:cpu_core_     |
                             |   hours:rate5m          |
                             | osac_cost:network_      |
                             |   ingress_gb:increase5m |
                             |--- recorded series ---->|
                                                      |
                                                      | cost model
                                                      | (rate cards)
                                                      |
                                                      | tenant invoices
```

### API Extensions

This enhancement introduces one new component:

- **osac-cost-mapping-controller**: A controller (deployed in
  `osac-system`) that watches ConfigMaps with the
  `osac.io/cost-mapping: "true"` label and generates
  PrometheusRule CRs. The PrometheusRule CRD is provided by the
  prometheus-operator (already present on OpenShift clusters via
  the cluster-monitoring-operator).

No new CRDs are introduced. The ConfigMap schema is validated
by the controller at reconcile time; invalid mappings are
rejected with a condition on the ConfigMap (via an annotation
`osac.io/cost-mapping-status: error` and
`osac.io/cost-mapping-error: "<message>"`).

### Implementation Details/Notes/Constraints

#### Controller implementation

The osac-cost-mapping-controller is a small Go controller using
controller-runtime. It:

1. Watches ConfigMaps in `osac-system` with the
   `osac.io/cost-mapping` label.
2. Parses `mapping.yaml` from the ConfigMap data.
3. Validates the schema (known dimensions, valid aggregation
   types, required fields).
4. Generates a PrometheusRule CR named
   `osac-cost-<provider>` in the same namespace.
5. Sets an owner reference from the PrometheusRule to the
   ConfigMap, so deleting the ConfigMap cleans up the rules.

The controller is stateless — all state is in the ConfigMaps
and generated PrometheusRules.

#### Label injection

OSAC multi-tenancy labels must appear on the recorded metrics
for Koku to attribute costs. The label mapping in the ConfigMap
supports two sources:

- **`label.*`**: Maps from existing Prometheus metric labels
  (e.g., `label.osac_tenant_id`).
- **`annotation.*`**: Maps from Kubernetes annotations on the
  source resource (requires relabeling at the
  ServiceMonitor/PodMonitor level to expose annotations as
  metric labels).

Providers are responsible for ensuring that tenant_id is
available as a metric label. For KubeVirt, the osac-operator
already sets `osac.io/tenant-id` annotations on VirtualMachine
CRs. For bare-metal providers, the AAP
`update_host_state` role (from the
inventory-provisioning-separation EP) sets tenant labels in
inventory, which the exporter surfaces.

#### Cardinality management

Recording rules aggregate by `tenant_id`,
`compute_instance_class`, and `site` by default. Per-instance
labels (`instance_id`) are optional and should only be included
when the provider's cardinality allows it. The controller
enforces a maximum of 5 label dimensions per recording rule to
prevent cardinality explosion.

For reference, a deployment with 100 tenants, 20 classes, and
5 sites produces at most 10,000 time series per dimension per
provider — well within Prometheus operational limits.

#### Integration with Koku

Koku integration is out of scope for this EP but follows a
known pattern:

1. A new Koku source type "OSAC" is registered, pointing at the
   Prometheus/Thanos endpoint.
2. Koku queries `osac_cost:*` recorded series using PromQL.
3. Rate cards (price per cpu_core_hour, per GB of egress, etc.)
   are configured in Koku per provider or per
   compute_instance_class.
4. Koku produces per-tenant cost reports using its existing
   aggregation and reporting pipeline.

The key contract is the recorded series names and label
taxonomy — as long as OSAC produces `osac_cost:<dimension>:*`
series with `tenant_id` and resource-type labels, Koku can
consume them.

### Risks and Mitigations

#### Provider metrics may not exist or may change

Providers may change their metric names across versions,
breaking the mapping.

*Mitigation:* The controller validates that `sourceMetric`
exists in the Prometheus target at reconcile time (optional,
configurable). If a source metric disappears, the recording
rule produces no data (safe — Koku sees a gap, not incorrect
data). The ConfigMap annotation surface errors for operator
visibility.

#### Incorrect unit conversion factors

A wrong `factor` value could produce wildly incorrect cost
data.

*Mitigation:* Document reference factors for common
conversions (bytes->GB, seconds->hours) in the ConfigMap
schema documentation. Include a `from`/`to` unit field for
human readability and future automated validation.

#### Label mapping misalignment

If a provider's Prometheus labels don't include tenant_id,
costs cannot be attributed.

*Mitigation:* The controller validates that `tenant_id` is
present in the label mapping. If missing, the ConfigMap is
rejected with an error annotation. Providers must ensure
tenant scoping is available as a metric label — this is
already a requirement for OSAC multi-tenancy.

#### Prometheus storage pressure

Recording rules add time series to Prometheus.

*Mitigation:* Series are aggregated (not per-instance by
default), capped at 5 label dimensions. Recording rule
interval is configurable (default 5m). For large deployments,
a dedicated Prometheus instance (or Thanos receiver) can be
used for cost metrics, separate from operational monitoring.

### Drawbacks

- Adds a dependency on the prometheus-operator's
  PrometheusRule CRD, which is standard on OpenShift but may
  not be present on vanilla Kubernetes deployments.
- The label mapping syntax (template expressions) adds
  complexity to the ConfigMap schema.
- Providers must ensure their Prometheus exporters include
  OSAC tenant labels — this may require ServiceMonitor
  relabeling configuration that is not covered by this EP.

## Alternatives (Not Implemented)

### Define OSAC-specific metrics

Have every provider export metrics under an `osac_*` namespace
with a standardized schema.

*Why not:* Requires every provider to implement dual-export
(native metrics for operations, OSAC metrics for billing).
Increases cardinality. Providers already have working
exporters — adding another metric namespace is redundant.

### Push-based cost reporting

Have providers push cost records to a central OSAC billing
service (e.g., via gRPC or CSV upload).

*Why not:* Introduces a new service, a new data store, and a
new integration point. Prometheus is already deployed, already
scraping provider metrics, and already connected to
Koku/Thanos. The pull-based model reuses existing
infrastructure.

### Koku-native provider plugins

Write a Koku provider plugin for each OSAC backend that
queries provider APIs directly.

*Why not:* Couples Koku to OSAC's backend diversity. Each new
backend requires Koku-side code changes. The ConfigMap approach
keeps the mapping in OSAC's operational domain where providers
control it.

### Recording rules without ConfigMaps

Ship static recording rules per provider as part of the
provider's deployment manifests.

*Why not:* Works for known providers but doesn't support
custom or third-party backends. The ConfigMap approach is
declarative and doesn't require rebuilding or redeploying OSAC
components to add a new mapping.

## Open Questions [optional]

1. **Koku source type registration.** Does Koku support
   pluggable source types, or would "OSAC" need to be added
   to Koku's codebase? If the latter, this EP's value
   proposition depends on upstream Koku accepting the source
   type.

2. **Multi-cluster Prometheus.** In deployments with multiple
   hub clusters, each with its own Prometheus, how are cost
   metrics aggregated? Thanos is the expected answer, but the
   Thanos topology should be documented.

3. **Rate card management.** Where are rate cards (price per
   cpu_core_hour, per GB of egress) stored and managed? This
   EP intentionally leaves this to Koku, but providers may
   want to define suggested rates in the ConfigMap alongside
   the metric mapping.

4. **Retroactive cost calculation.** When a new ConfigMap is
   deployed, can Koku backfill costs from historical
   Prometheus data, or do costs only accrue from the moment
   the recording rule is created?

## Test Plan

TBD — will cover:
- ConfigMap validation (valid and invalid schemas)
- PrometheusRule generation from ConfigMap
- Recording rule correctness (unit conversion, label mapping)
- ConfigMap update and deletion (PrometheusRule lifecycle)
- End-to-end: provider metrics -> recording rules -> Koku
  query returns expected values
- Cardinality limits enforcement (max 5 label dimensions)

## Graduation Criteria

TBD

## Upgrade / Downgrade Strategy

**Upgrade:** Deploy the osac-cost-mapping-controller and create
ConfigMaps for each provider. The controller generates
PrometheusRules. No existing components are modified — this is
purely additive.

**Downgrade:** Delete the osac-cost-mapping-controller
deployment. The PrometheusRule CRs persist (no owner reference
to the controller Deployment) but can be manually deleted. No
impact on OSAC provisioning — cost mapping is an observability
concern, not a provisioning dependency.

## Version Skew Strategy

The osac-cost-mapping-controller only reads ConfigMaps and
writes PrometheusRules. It has no dependency on
fulfillment-service or osac-operator versions. Provider-native
metrics are emitted by the provider's own exporter, which
is versioned independently. The ConfigMap's `sourceMetric`
field must match the provider exporter's actual metric names
— this is a deployment-time configuration concern, not a
version skew issue.

## Support Procedures

- **No cost data in Koku:** Verify the ConfigMap exists with
  the `osac.io/cost-mapping: "true"` label. Check the
  controller logs for validation errors. Verify the
  PrometheusRule CR was created
  (`kubectl get prometheusrules -n osac-system`). Check that
  the source metric exists in Prometheus
  (`curl prometheus:9090/api/v1/query?query=<sourceMetric>`).

- **Incorrect cost values:** Check the `unitConversion.factor`
  in the ConfigMap. Query the recording rule output directly
  in Prometheus to verify the aggregation. Compare with raw
  provider metrics to identify the discrepancy.

- **ConfigMap rejected (error annotation):** Read the
  `osac.io/cost-mapping-error` annotation on the ConfigMap
  for the specific validation error. Common issues: unknown
  dimension name, missing tenant_id in label mapping, invalid
  aggregation type.

- **High cardinality alert:** Check which ConfigMap includes
  per-instance labels (instance_id). Remove instance_id from
  the label mapping or reduce the number of label dimensions
  to stay under the limit.

## Infrastructure Needed [optional]

None beyond existing OSAC infrastructure. The
prometheus-operator (PrometheusRule CRD) is required and is
standard on OpenShift clusters.
