---
title: OSAC Object Storage Add-On
authors:
  - fdupont@redhat.com
creation-date: 2026-04-30
last-updated: 2026-04-30
tracking-link:
  - TBD
see-also:
  - enhancements/osac-addon/README.md
  - enhancements/tenant-storage-tiers/README.md
  - "MGMT-24079: OSAC Use-Case Composability in Enclave Plugin"
  - "MGMT-23930: VAST Data Tenant Storage Onboarding"
replaces:
superseded-by:
---

# OSAC Object Storage Add-On

## Summary

Introduce tenant-facing S3-compatible object storage as an
**OSAC Add-On**, following the contract defined in the
[OSAC Add-On EP](../osac-addon/README.md). The add-on is
delivered as an Ansible collection (`osac.object_storage_odf`) with
roles following the ResourceAction naming convention, packaged
as an Enclave plugin for deployment.

OpenShift Data Foundation (ODF) is the first provider, with
Ceph RGW for native S3 and Noobaa/MCG for S3 with multi-cloud
federation. This add-on also serves as the **reference
implementation** of the OSAC Add-On model.

## Motivation

### Current state

OSAC manages block storage via StorageClasses and the
tenant-storage-tiers mechanism. There is no object storage
offering. Tenants must provision S3-compatible storage outside
OSAC.

### Problems

1. **No managed object storage.** Tenants cannot create S3
   buckets through the OSAC API.

2. **No tenant isolation for object storage.**

3. **No quota or billing integration.** Object storage
   consumption is invisible to Koku.

4. **No multi-cloud federation.**

### Comparison with VMware Cloud Director

VCD does not provide built-in object storage. CSPs extend VCD
using RDEs and Behaviors, requiring significant custom
development. OSAC provides object storage as a first-class
add-on following the standardized contract — no custom
extension framework needed.

### User Stories

- As a **tenant**, I want to create an S3-compatible bucket
  through the OSAC API.

- As a **tenant admin**, I want to manage S3 access keys for
  my organization.

- As a **CSP admin**, I want to deploy the object storage
  add-on backed by my ODF infrastructure.

- As a **CSP admin**, I want storage quotas per tenant and
  per bucket.

- As a **tenant**, I want to federate a bucket across cloud
  providers via MCG for disaster recovery.

### Goals

- Deliver object storage as the reference OSAC Add-On.
- Implement ODF with two backends: Ceph RGW and Noobaa/MCG.
- Support tenant-scoped buckets and credentials with quotas.
- Integrate with billing labels for Koku.

### Non-Goals

- Implementing an S3-compatible API server.
- Block or file storage.
- Data migration between providers.

## Proposal

### Resource Types

#### ObjectStoreClass (provider-defined)

```yaml
apiVersion: osac.io/v1
kind: ObjectStoreClass
metadata:
  name: odf-rgw
  labels:
    osac.io/addon: osac-object-storage
spec:
  provider: odf-rgw
  capabilities:
    - S3_COMPATIBLE
    - VERSIONING
    - LIFECYCLE_POLICIES
  region: us-east-1
  defaults:
    replication: 3
    maxBucketSizeGB: 1000
```

```yaml
apiVersion: osac.io/v1
kind: ObjectStoreClass
metadata:
  name: odf-mcg
  labels:
    osac.io/addon: osac-object-storage
spec:
  provider: odf-mcg
  capabilities:
    - S3_COMPATIBLE
    - VERSIONING
    - MULTI_CLOUD_FEDERATION
  region: us-east-1
  defaults:
    replication: 3
    maxBucketSizeGB: 500
    federationTargets:
      - aws-us-east-1
      - azure-westeurope
```

#### ObjectStoreBucket (tenant-scoped)

```yaml
apiVersion: osac.io/v1
kind: ObjectStoreBucket
metadata:
  name: acme-ml-datasets
  annotations:
    osac.openshift.io/tenant: acme-corp
    osac.io/billing-labels: "tier=premium,department=ml"
spec:
  objectStoreClass: odf-rgw
  versioning: true
  quotaGB: 500
  lifecycleRules:
    - name: archive-old-data
      prefix: raw/
      transition:
        days: 90
        storageClass: GLACIER
  federation:
    enabled: false
status:
  state: READY
  endpoint: https://s3.osac.example.com
  bucketName: acme-corp-acme-ml-datasets
  usedGB: 127
  objectCount: 45230
```

#### ObjectStoreAccount (tenant-scoped)

```yaml
apiVersion: osac.io/v1
kind: ObjectStoreAccount
metadata:
  name: ml-team-access
  annotations:
    osac.openshift.io/tenant: acme-corp
spec:
  objectStoreClass: odf-rgw
  bucketAccess:
    - bucketName: acme-ml-datasets
      permissions: [READ, WRITE]
    - bucketName: acme-model-artifacts
      permissions: [READ]
status:
  state: ACTIVE
  accessKeyID: AKIAIOSFODNN7EXAMPLE
  secretAccessKey:
    secretRef:
      name: ml-team-s3-credentials
      key: secret-access-key
  endpoint: https://s3.osac.example.com
```

### Ansible Collection: osac.object_storage_odf

Following the [OSAC Add-On convention](../osac-addon/README.md):

#### meta/addon.yaml

```yaml
name: osac-object-storage-odf
display_name: Object Storage (ODF)
description: S3-compatible object storage via OpenShift Data Foundation
version: 1.0.0

dependencies:
  osac_core: ">=0.1.0"
  operators:
    - name: odf-operator
      optional: false

resource_types:
  - name: ObjectStoreClass
    scope: provider
  - name: ObjectStoreBucket
    scope: tenant
  - name: ObjectStoreAccount
    scope: tenant
```

#### Roles

```
roles/
├── object_store_bucket.create.main/
│   ├── meta/osac.yaml
│   └── tasks/main.yml
├── object_store_bucket.delete.main/
│   ├── meta/osac.yaml
│   └── tasks/main.yml
├── object_store_account.create.main/
│   ├── meta/osac.yaml
│   └── tasks/main.yml
└── object_store_account.delete.main/
    ├── meta/osac.yaml
    └── tasks/main.yml
```

#### Role Metadata Example

```yaml
# roles/object_store_bucket.create.main/meta/osac.yaml
resource_type: ObjectStoreBucket
event: Create
phase: main
priority: 100
failure_policy: Fail

parameters:
  - name: quota_gb
    type: integer
    required: false
    default: 100
    description: Maximum bucket size in GiB
  - name: versioning
    type: boolean
    required: false
    default: false
    description: Enable S3 object versioning
  - name: object_store_class
    type: string
    required: true
    description: ObjectStoreClass to use

outputs:
  - name: bucket_name
    type: string
    description: Created bucket name (tenant-prefixed)
  - name: endpoint
    type: string
    description: S3 endpoint URL
  - name: bucket_id
    type: string
    description: Internal bucket identifier
```

#### Role Implementation (ODF RGW)

```yaml
# roles/object_store_bucket.create.main/tasks/main.yml
- name: Resolve ObjectStoreClass
  # Read class to determine backend (RGW vs MCG)

- name: Create RGW bucket
  ansible.builtin.uri:
    url: "{{ rgw_admin_endpoint }}/bucket"
    method: PUT
    body_format: json
    body:
      bucket: "{{ osac_tenant }}-{{ osac_resource.metadata.name }}"
      tenant: "{{ osac_tenant }}"
  register: created_bucket
  when: object_store_class == "odf-rgw"

- name: Configure quota
  ansible.builtin.uri:
    url: "{{ rgw_admin_endpoint }}/user?quota"
    method: PUT
    body_format: json
    body:
      max_size_kb: "{{ quota_gb * 1048576 }}"
      enabled: true

- name: Report outputs
  ansible.builtin.set_stats:
    data:
      osac_state: READY
      osac_conditions:
        - type: Provisioned
          status: "True"
      bucket_name: "{{ osac_tenant }}-{{ osac_resource.metadata.name }}"
      endpoint: "{{ rgw_s3_endpoint }}"
      bucket_id: "{{ created_bucket.json.bucket_id }}"
```

### Workflow

#### Bucket creation

1. Tenant creates ObjectStoreBucket via OSAC API.
2. Fulfillment-service validates (quota, tenant auth).
3. osac-operator looks up cached ResourceActions for
   `ObjectStoreBucket.Create` from `osac.object_storage_odf`
   collection.
4. Generates AAP Workflow, submits.
5. Role creates Ceph RGW bucket, reports outputs via
   `set_stats`.
6. Operator updates resource status.

#### Bucket federation (MCG)

1. Tenant creates ObjectStoreBucket with `objectStoreClass:
   odf-mcg` and `federation.enabled: true`.
2. Role creates Noobaa BucketClass + ObjectBucketClaim.
3. Noobaa fulfills with federated bucket.
4. Data replicates across configured backing stores.

#### Rollback

If bucket creation fails after partial setup (e.g., bucket
created but quota config failed), the AAP Workflow runs the
compensating `object_store_bucket.delete.main` role, which
receives the `bucket_name` output from the failed create and
tears it down.

### Enclave Plugin (Packaging)

```
plugins/osac-object-storage/
├── plugin.yaml
├── defaults/
│   └── defaults.yaml
└── tasks/
    ├── early-validate.yaml  # check ODF is available
    ├── deploy.yaml          # install CRDs, create *Class CRs
    └── post-validate.yaml   # verify S3 endpoint
```

```yaml
# plugin.yaml
name: osac-object-storage
type: addon
order: 200
description: S3-compatible object storage for OSAC tenants
mirror: plugin
dependencies:
  plugins:
    - osac-core
  operators:
    - odf-operator
```

### Other Providers

| Provider | Collection | Status |
|---|---|---|
| ODF (RGW + MCG) | `osac.object_storage_odf` | First provider |
| MinIO | `osac.object_storage_minio` | Planned |
| VAST Data | `vast.osac_storage` | Onboarding (MGMT-23930) |
| Dell ECS | `dell.osac_object_storage` | Future |
| Scality RING | `scality.osac_object_storage` | Future |

Each provider ships its own collection with independent
lifecycle and versioning.

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| ODF RGW admin API requires elevated privileges | Dedicated AAP service account with scoped RBAC |
| Noobaa federation and data residency | Federation opt-in per bucket; CSP controls backing stores |
| S3 credential leakage | Credentials in K8s Secrets with tenant RBAC; rotation via account update |
| Quota enforcement lag | OSAC tracks requested quota; Ceph enforces at storage level |

## Test Plan

- Lint `osac.object_storage_odf` collection with `osac addon lint`.
- Integration tests with Kind: verify operator discovery, CRUD,
  AAP Workflow dispatch.
- E2E with ODF: bucket creation via RGW, S3 access, federation
  via MCG, quota enforcement.
- Rollback: fail mid-create, verify compensating delete runs.

## Graduation Criteria

### Alpha

- `osac.object_storage_odf` collection published to Hub.
- ODF RGW provider (buckets, credentials, quotas).
- Enclave plugin packaging.
- CLI support.

### Beta

- ODF MCG provider with federation.
- Billing labels integration.

### GA

- Partner providers (MinIO, VAST Data).
- Lifecycle policies and versioning.
