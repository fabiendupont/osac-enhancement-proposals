---
title: dns-api
authors:
  - dmanor
creation-date: 2026-03-17
last-updated: 2026-03-26
tracking-link:
  - TBD
see-also:
  - "/enhancements/networking"
  - enhancements/osac-addon/README.md
  - enhancements/vpn-service/README.md
  - enhancements/load-balancer-service/README.md
replaces:
  - N/A
superseded-by:
  - N/A
---

# DNS API

## Summary

OSAC currently hardcodes AWS Route 53 as the DNS backend for managing DNS
records during cluster provisioning (CaaS). This enhancement introduces a
pluggable internal DNS API that abstracts DNS record management behind a
class-agnostic interface. Deployers of OSAC will be able to use any
supported DNS backend (AWS Route 53, Cloudflare, Azure DNS, Google Cloud DNS,
etc.) by selecting a DNS class and supplying the appropriate
credentials, without modifying the core provisioning logic.

## Motivation

OSAC's Cluster-as-a-Service provisioning creates DNS records for each managed
cluster (API endpoint, API-internal endpoint, and wildcard ingress). A pluggable
DNS layer will make OSAC deployable in environments that use any DNS backend,
whether a commercial DNS service (such as AWS Route 53, Cloudflare, or Azure
DNS), a self-hosted DNS server using the RFC 2136 protocol (such as BIND or
PowerDNS), or any other provider. This abstraction also establishes a pattern
that future OSAC services (VMaaS, BareMetal-as-a-Service) can reuse when they
need DNS record management.

### User Stories

- As an OSAC deployer using a commercial DNS service (such as Route 53 or
  Cloudflare DNS), I want to configure OSAC to create DNS records via my
  provider's API so that I can use my existing DNS infrastructure.

- As an OSAC deployer using a self-hosted DNS server (such as BIND or
  PowerDNS), I want to configure OSAC to manage DNS records via the RFC 2136
  protocol so that records are resolvable within my private network.

- As an OSAC template developer, I want a simple, generic interface for
  creating and deleting DNS records so that I do not need to write
  backend-specific logic in my templates.

- As an OSAC platform operator, I want to manage DNS provider credentials
  through standard Kubernetes/OpenShift credential mechanisms (e.g., Secrets)
  so that secrets are handled consistently and securely.

### Goals

- Introduce per-provider DNS collections following the
  [OSAC Add-On convention](../osac-addon/README.md):
  `osac.dns_route53` (existing behavior),
  `osac.dns_rfc2136` (for self-hosted DNS servers like BIND
  or PowerDNS), and additional providers as needed (e.g.,
  `osac.dns_cloudflare`, `osac.dns_azure`).
- Enable third-party organizations to create custom DNS provider
  collections in their own namespace (e.g.,
  `myorg.osac_dns`) without modifying OSAC code.
- Refactor all roles that currently call `amazon.aws.route53`
  directly to use the new DNS abstraction via provider
  collections.
- Allow deployers to select their DNS provider via a DnsClass
  resource that references the provider collection, without
  modifying roles or playbooks.

### Non-Goals

- Providing a user-facing DNS API through the Fulfillment Service or CLI.
  This enhancement is scoped to the internal Ansible automation layer.
- Managing DNS zones or delegations. The DNS API only manages individual
  records within an existing zone.
- Supporting DNS record types beyond A and AAAA records. CNAME and other
  record types may be added later as needed.
- DNS integration for VMaaS or BareMetal-as-a-Service. This enhancement
  targets CaaS cluster provisioning, but the DNS abstraction is designed
  to be reused by VMaaS and BMaaS when they require DNS record management.
- Replacing the `wait_for_dns` role. DNS propagation verification remains
  class-agnostic (uses `dig`) and does not need changes.

## Proposal

This proposal introduces per-provider DNS Ansible collections following the
[OSAC Add-On convention](../osac-addon/README.md). Each DNS provider ships its
own collection (e.g., `osac.dns_route53`, `osac.dns_cloudflare`) with
ResourceAction roles (`record.create.main`, `record.delete.main`). A `DnsClass`
resource references the provider collection, and the operator dispatches to it
— no dispatcher role needed. This follows the collection-per-provider model
where each collection is self-contained and independently versioned.

Third-party organizations create their own DNS provider collections
(e.g., `myorg.osac_dns`) without modifying OSAC code.

All roles that currently manage DNS records will use the provider collection
referenced by the DnsClass instead of directly invoking backend-specific
modules.

### Workflow Description

**OSAC deployer** is the person deploying and configuring the OSAC platform.

**OSAC template developer** is a developer creating or modifying Ansible
templates for cluster or VM provisioning.

#### Deployer Configures DNS Provider

1. The deployer creates a `DnsClass` resource referencing the provider
   collection (e.g., `collection: osac.dns_route53`).
2. The deployer provides provider-specific configuration variables. Each
   provider collection manages its own backend-specific details (e.g., zone,
   credentials) so that callers don't need to know about them.
3. The deployer provides credentials through standard Kubernetes/OpenShift
   mechanisms (e.g., Secrets), which are then made available to the automation
   layer.

#### Cluster Provisioning Creates DNS Records

1. The external access role determines the DNS records needed (API,
   API-internal, wildcard ingress).
2. Instead of calling `amazon.aws.route53` directly, it includes the
   `record.create.main` role from the DnsClass's provider collection.
3. The provider role creates the DNS record using the appropriate Ansible
   module.
4. The existing `wait_for_dns` role verifies DNS propagation (unchanged).

#### Cluster Deletion Removes DNS Records

1. The external access role's destroy tasks include the `record.delete.main`
   role from the DnsClass's provider collection.
2. The provider role deletes the records.

### API Extensions

This enhancement introduces a `DnsClass` resource (private API, provider-
defined) that references the DNS provider collection. It does not modify the
tenant-facing Fulfillment API.

### Implementation Details/Notes/Constraints

#### Provider Collections (per-provider, no dispatcher)

Following the [OSAC Add-On convention](../osac-addon/README.md),
each DNS provider ships its own collection with ResourceAction
roles:

```text
osac.dns_route53/
  meta/
    addon.yaml
  roles/
    record.create.main/
      meta/osac.yaml
      tasks/main.yml
    record.delete.main/
      meta/osac.yaml
      tasks/main.yml

osac.dns_cloudflare/
  roles/
    record.create.main/
      meta/osac.yaml
      tasks/main.yml
    record.delete.main/
      meta/osac.yaml
      tasks/main.yml

osac.dns_rfc2136/
  roles/
    record.create.main/
      meta/osac.yaml
      tasks/main.yml
    record.delete.main/
      meta/osac.yaml
      tasks/main.yml
```

Third-party organizations create their own provider collections
(e.g., `myorg.osac_dns`) following the same convention,
without modifying any OSAC code.

The default DnsClass uses `osac.dns_route53` for backward
compatibility.

#### Role I/O Contract (meta/osac.yaml)

Each provider collection's `record.create.main` role declares
its contract in `meta/osac.yaml` following the OSAC Add-On
convention:

```yaml
# osac.dns_route53/roles/record.create.main/meta/osac.yaml
resource_type: DnsRecord
event: Create
phase: main
priority: 100
failure_policy: Fail

parameters:
  - name: dns_record_name
    type: string
    required: true
    description: Fully qualified domain name for the record
  - name: dns_record_type
    type: string
    required: false
    default: A
    description: "DNS record type: A or AAAA"
  - name: dns_record_value
    type: string
    required: true
    description: The value for the record (IP address)
  - name: dns_record_ttl
    type: integer
    required: false
    default: 1800
    description: TTL in seconds
  - name: dns_record_overwrite
    type: boolean
    required: false
    default: true

outputs:
  - name: record_id
    type: string
    description: Created record identifier
```

#### Route 53 Provider (`osac.dns_route53`)

```yaml
---
- name: "Create DNS record {{ dns_record_name }} via Route 53"
  amazon.aws.route53:
    state: present
    zone: "{{ dns_route53_zone }}"
    record: "{{ dns_record_name }}"
    type: "{{ dns_record_type }}"
    ttl: "{{ dns_record_ttl | default(1800) }}"
    value: "{{ dns_record_value }}"
    wait: true
    overwrite: "{{ dns_record_overwrite | default(true) }}"
```

#### Cloudflare Provider (`osac.dns_cloudflare`)

```yaml
---
- name: "Create DNS record {{ dns_record_name }} via Cloudflare"
  community.general.cloudflare_dns:
    state: present
    zone: "{{ dns_cloudflare_zone }}"
    record: "{{ dns_record_name }}"
    type: "{{ dns_record_type }}"
    value: "{{ dns_record_value }}"
    ttl: "{{ dns_record_ttl | default(1) }}"
    api_token: "{{ dns_cloudflare_api_token }}"
    solo: "{{ dns_record_overwrite | default(true) }}"
```

#### RFC 2136 Provider (`osac.dns_rfc2136`)

```yaml
---
- name: "Create DNS record {{ dns_record_name }} via RFC 2136"
  community.general.nsupdate:
    state: present
    server: "{{ dns_rfc2136_server }}"
    port: "{{ dns_rfc2136_port | default(53) }}"
    zone: "{{ dns_rfc2136_zone }}"
    record: "{{ dns_record_name }}"
    type: "{{ dns_record_type }}"
    ttl: "{{ dns_record_ttl | default(1800) }}"
    value: "{{ dns_record_value }}"
    key_name: "{{ dns_rfc2136_key_name }}"
    key_secret: "{{ dns_rfc2136_key_secret }}"
    key_algorithm: "{{ dns_rfc2136_key_algorithm | default('hmac-sha256') }}"
```

#### Custom Third-Party Provider

Any organization can create a custom DNS provider collection in
their own namespace (e.g., `myorg.osac_dns`). The collection
must contain `record.create.main` and `record.delete.main` roles
with `meta/osac.yaml` declaring the standard DNS parameters and
outputs. The deployer creates a DnsClass referencing
`collection: myorg.osac_dns`.

#### Refactored external access roles (create example)

The current DNS tasks in the relevant roles
follow the same pattern. example:

```yaml
# Before (hardcoded Route 53):
- name: Create dns records
  amazon.aws.route53:
    state: present
    zone: "{{ external_access_base_domain }}"
    record: "{{ item.name }}"
    type: A
    ttl: "{{ external_access_dns_ttl }}"
    value: "{{ item.addr | regex_replace('/.*$', '') }}"
    wait: true
    overwrite: true
  loop:
    - name: "{{ external_access_api_domain }}"
      addr: "{{ external_access_api_floating_ip }}"
    - name: "{{ external_access_api_int_domain }}"
      addr: "{{ external_access_api_floating_ip }}"
    - name: "*.{{ external_access_ingress_domain }}"
      addr: "{{ netris_l4lb_ingress_ip }}"
```

Would become:

```yaml
# After (provider-agnostic via DnsClass collection):
- name: Create dns records
  ansible.builtin.include_role:
    name: "{{ dns_class_collection }}.record.create.main"
  vars:
    dns_record_name: "{{ item.name }}"
    dns_record_type: A
    dns_record_ttl: "{{ external_access_dns_ttl }}"
    dns_record_value: "{{ item.addr | regex_replace('/.*$', '') }}"
  loop:
    - name: "{{ external_access_api_domain }}"
      addr: "{{ external_access_api_floating_ip }}"
    - name: "{{ external_access_api_int_domain }}"
      addr: "{{ external_access_api_floating_ip }}"
    - name: "*.{{ external_access_ingress_domain }}"
      addr: "{{ netris_l4lb_ingress_ip }}"
```

Where `dns_class_collection` is resolved from the DnsClass
resource's `collection` field (e.g., `osac.dns_route53`).

#### Refactored external access roles (destroy example)

Similarly, the delete tasks would change from direct Route 53
calls to:

```yaml
# After:
- name: Delete dns records
  ansible.builtin.include_role:
    name: "{{ dns_class_collection }}.record.delete.main"
  vars:
    dns_record_name: "{{ item }}"
    dns_record_type: A
  loop:
    - "api.{{ external_access_name }}.{{ external_access_base_domain }}"
    - "api-int.{{ external_access_name }}.{{ external_access_base_domain }}"
    - "*.apps.{{ external_access_name }}.{{ external_access_base_domain }}"
```

#### Configuration (group_vars)

Deployers configure their DNS class in group_vars:

```yaml
# group_vars/all/dns.yaml

# DNS provider is configured via DnsClass resource (collection field)
# Default DnsClass references osac.dns_route53

# Route 53 class-specific configuration
# dns_route53_zone: "example.com"
# (uses AWS credentials from environment/AAP credential)

# Cloudflare class-specific configuration
# dns_cloudflare_zone: "example.com"
# dns_cloudflare_api_token: "{{ vault_cloudflare_api_token }}"

# RFC 2136 class-specific configuration
# dns_rfc2136_server: "ns1.example.com"
# dns_rfc2136_port: 53
# dns_rfc2136_zone: "example.com"
# dns_rfc2136_key_name: "osac-update-key"
# dns_rfc2136_key_secret: "{{ vault_rfc2136_key_secret }}"
# dns_rfc2136_key_algorithm: "hmac-sha256"
```

#### Adding a New DNS Provider

To add support for a new DNS provider (e.g., Azure DNS):

1. Create a new collection (e.g., `osac.dns_azure`) with
   `record.create.main` and `record.delete.main` roles following
   the [OSAC Add-On convention](../osac-addon/README.md).
2. Add `meta/osac.yaml` to each role declaring the standard DNS
   parameters and outputs.
3. Implement the create/delete logic using the appropriate
   Ansible module (e.g., `azure.azcollection.azure_rm_dnsrecordset`).
4. Validate with `osac addon lint osac.dns_azure`.
5. Publish the collection to Automation Hub.
6. Create a DnsClass resource referencing
   `collection: osac.dns_azure`.

Third-party organizations follow the same steps in their own
namespace (e.g., `myorg.osac_dns`).

#### Provider Collections

| Provider | Collection | Ansible Dependency |
|----------|-----------|-------------------|
| Route 53 | `osac.dns_route53` | `amazon.aws` |
| Cloudflare | `osac.dns_cloudflare` | `community.general` |
| RFC 2136 | `osac.dns_rfc2136` | `community.general` |
| Azure DNS | `osac.dns_azure` | `azure.azcollection` |
| GCP DNS | `osac.dns_gcp` | `google.cloud` |

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Class module API differences | Different modules have different parameters and behaviors (e.g., Cloudflare uses `solo` vs Route 53 uses `overwrite`) | Abstract differences inside each driver role; expose only the common interface |
| Credential management varies per provider | Each DNS provider uses different authentication mechanisms | Document credential setup per provider; manage credentials via Kubernetes Secrets |
| Wildcard record support | Not all DNS backends handle wildcard DNS records identically | Test wildcard record creation/deletion for each supported class before release |
| Breaking existing deployments | Deployers currently using Route 53 implicitly | Default DnsClass references `osac.dns_route53` so existing deployments work without config changes |

### Drawbacks

- Adds a layer of indirection for DNS operations. Debugging DNS issues now
  requires understanding which DnsClass and provider collection is configured.
- Each new DNS provider requires implementing and maintaining a collection,
  adding to the maintenance surface (though partner collections are maintained
  independently).
- Backend-specific features (e.g., Route 53 health checks, Cloudflare
  proxying) are not exposed through the common interface. Deployers who need
  these features would need to extend the driver roles.

## Alternatives (Not Implemented)

### Alternative 1: DNS API in the Fulfillment Service

Expose DNS record management as a first-class API in the Fulfillment Service
(similar to the Networking API), with a gRPC/REST interface and CRDs.

**Why not selected**: DNS record management during cluster provisioning is an
infrastructure concern handled entirely within the Ansible automation layer.
Adding it to the Fulfillment Service would introduce unnecessary complexity and
API surface for what is essentially an infrastructure-side operation. This alternative
could be revisited if tenant-facing DNS management becomes a requirement.

### Alternative 2: Task Files Within a Single Role (no separate collections)

Instead of separate collections per provider, implement each DNS backend as
task files within a single collection and dispatch via `include_tasks`.

**Why not selected**: This approach requires modifying the shared collection
for every new DNS provider. With collection-per-provider (following the
[OSAC Add-On convention](../osac-addon/README.md)), third-party organizations
create their own collections (e.g., `myorg.osac_dns`) with independent
lifecycle and versioning.

## Open Questions

1. Should the DNS role support batch operations (multiple records in a single
   call) for DNS classes that support atomic batch updates, or is per-record
   invocation via `loop` sufficient?

2. Should we define a standard Kubernetes Secret format for each DNS provider,
   or leave credential management to the deployer?

3. Which additional DNS providers should be included in the initial
   implementation beyond Route 53? Cloudflare is proposed as the second
   provider, but Azure DNS or Google Cloud DNS may be higher priority depending
   on the deployment targets.

## Test Plan

*Section not required until targeted at a release.*

- **Lint tests**: Validate each provider collection with `osac addon lint`.
- **Integration tests**: For each provider collection, test create and delete
  of A and AAAA records, including wildcard records, against a
  real DNS zone in a CI environment.
- **Migration test**: Verify that existing deployments using Route 53 continue
  to work with default DnsClass referencing `osac.dns_route53` without any
  configuration changes.

## Graduation Criteria

*Section not required until targeted at a release.*

- **Dev Preview**: `osac.dns_route53` collection working; relevant roles
  refactored to use provider collection dispatch.
- **Tech Preview**: At least one additional provider collection implemented;
  documentation for creating new DNS provider collections.
- **GA**: All supported providers documented and tested; migration guide
  published.

## Upgrade / Downgrade Strategy

- Existing deployments will use a default DnsClass referencing
  `osac.dns_route53`, preserving current behavior with no changes
  required.
- Both relevant roles will be updated to use the provider collection
  dispatch. Since this is an internal implementation change, it does not
  affect the external API or CRDs.
- Downgrade: reverting to a previous version restores the hardcoded
  Route 53 behavior.

## Version Skew Strategy

This enhancement is contained within the `osac.service` Ansible collection and
the `osac-aap` repository. There are no cross-component version skew concerns
since DNS record management is invoked synchronously during playbook execution.

## Support Procedures

- **DNS record creation fails**: Check the `dns_class` value and ensure
  class-specific credentials are configured. Review the Ansible task output
  for class-specific error messages (e.g., invalid API token, zone not found).
- **Unsupported class error**: The DNS role will fail with a clear error
  message if `dns_class` is set to a role that cannot be found or does not
  implement the expected entrypoints.
- **DNS propagation timeout**: The `wait_for_dns` role (unchanged) will report
  which records failed to resolve. This is independent of the DNS class and
  may indicate a backend-side delay or misconfiguration.

## Infrastructure Needed

No new infrastructure is required. Testing may require credentials for each
supported DNS class's API (AWS, Cloudflare, etc.) in the CI environment.
