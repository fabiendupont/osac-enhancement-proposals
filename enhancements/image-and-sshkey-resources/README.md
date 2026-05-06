---
title: image-and-sshkey-resources
authors:
  - Fabien Dupont
creation-date: 2026-04-17
last-updated: 2026-04-17
tracking-link:
  - TBD
see-also:
  - enhancements/unified-compute-model/README.md
  - enhancements/osac-addon/README.md
  - enhancements/vm-api-fields/README.md
  - enhancements/metal3-compute-backend/README.md
  - enhancements/inventory-provisioning-separation/README.md
  - enhancements/catalog-items/README.md
  - https://github.com/osac-project/enhancement-proposals/pull/35 (VM Image Management)
replaces:
superseded-by:
---

# Image and SSHKey as Catalog Resources

## Summary

Decouple OS image selection and SSH key management from inline fields
on ComputeInstance into first-class OSAC resources:

- **Image**: A provider-defined catalog of bootable OS images with
  metadata (architecture, boot method, compatibility with
  ComputeInstanceClasses).
- **SSHKey**: A tenant-managed SSH public key that can be referenced
  by multiple ComputeInstances.

## Motivation

### Current state

The vm-api-fields EP added `image` (sourceType + sourceRef) and
`sshKey` (string) as inline fields on ComputeInstance. This works
for simple cases but has limitations:

1. **No image catalog.** Tenants must know the exact OCI registry
   URL for their image. There is no way for the provider to curate
   a list of validated, supported images. Tenants can use arbitrary
   images — a security concern. The
   [VM Image Management EP](https://github.com/osac-project/enhancement-proposals/pull/35)
   addresses the upload and storage mechanics (qcow2 upload, OCI
   registry, PVC caching) but explicitly defers the provider-managed
   shared image catalog to a future enhancement — this proposal
   provides that catalog layer, adding metadata (architecture, boot
   method, compatibility) that the upload pipeline does not cover.

2. **No image-hardware compatibility.** An image built for x86
   can be selected for an ARM bare-metal class. An image without
   NVIDIA drivers can be selected for a GPU class. There is no
   validation because the image has no metadata.

3. **No boot method awareness.** The ComputeInstanceTemplate role
   needs to know whether to use Ignition, cloud-init, or Kickstart
   to inject SSH keys and userData. This is an image property, but
   it's not exposed — templates must hardcode or guess.

4. **SSH keys are disposable.** Each ComputeInstance carries its own
   SSH key string. The same key is duplicated across instances.
   There is no way to rotate a key, revoke a compromised key, or
   audit which keys have access to which instances.

### How public clouds solve this

- AWS: AMI (image catalog with architecture, root device type,
  compatibility), Key Pair (tenant-managed, referenceable)
- GCP: Image (catalog with family, architecture), SSH keys
  (project-level metadata)
- Azure: VM Image (marketplace with publisher, offer, SKU), SSH
  Public Key resource

### User Stories

- As a **cloud provider**, I want to define a catalog of validated
  OS images with metadata, so that tenants can only use images that
  are compatible with the hardware they're ordering.

- As a **tenant**, I want to browse available images and select one
  by name when creating a ComputeInstance, instead of providing a
  raw OCI URL.

- As a **tenant**, I want to register my SSH public keys once and
  reference them by name across multiple ComputeInstances.

- As a **tenant**, I want to rotate or revoke an SSH key without
  recreating my ComputeInstances.

- As an **infrastructure provider** (ComputeInstanceTemplate
  author), I want to know the image's boot method so I can generate
  the correct configuration format (Ignition, cloud-init, or
  Kickstart).

### Goals

- Introduce Image as a provider-defined catalog resource.
- Introduce SSHKey as a tenant-managed resource.
- Modify ComputeInstance to reference Image and SSHKey by name.
- Enable image-class compatibility validation.
- Expose boot method as an image property for template roles.

### Non-Goals

- Catalog item presentation and field-level policies — the
  [Catalog Items EP](../catalog-items/README.md)
  addresses how catalog entries are presented to tenants. This EP
  defines the Image resource itself; Catalog Items could reference
  Images when defining field defaults and editability.
- Image upload, registry storage, and PVC caching — the
  [VM Image Management EP](https://github.com/osac-project/enhancement-proposals/pull/35)
  covers the upload pipeline and on-cluster caching. This EP
  defines the catalog metadata layer on top of those stored images.
  The two are complementary: PR #35 handles how images get into
  the registry; this EP handles how tenants discover and select
  them with validation.
- Image building or CI/CD pipelines.
- SSH key injection mechanism (that is the template role's
  responsibility, unchanged).
- Volume management (pre-existing volume attachment is a future EP).

## Proposal

### Image (provider-defined, tenant-visible)

An Image represents a bootable OS image in the provider's catalog.

```yaml
id: "rhel-9.6-gpu"
title: "RHEL 9.6 with NVIDIA Drivers"
description: "Image-mode RHEL 9.6, CUDA 12.8, NVIDIA driver 570, pre-validated for B200 and H100 GPUs"
sourceType: "registry"
sourceRef: "quay.ncp.dog8.cloud/images/rhel-bootc:9.6-cuda12.8"
os: "rhel"
version: "9.6"
architecture: "aarch64"
bootMethod: "ignition"
checksum: "sha256:a1b2c3d4e5f6..."
compatibility:
  - "gpu-b200-4"
  - "gpu-h100-8"
  - "gpu-a100-1-vm"
```

Fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier |
| `title` | string | yes | Human-readable name |
| `description` | string | no | Markdown description |
| `sourceType` | string | yes | Image source type (e.g., `registry`) |
| `sourceRef` | string | yes | Image reference (e.g., OCI URL) |
| `os` | string | no | Operating system name |
| `version` | string | no | OS version |
| `architecture` | string | no | CPU architecture (`aarch64`, `x86_64`) |
| `bootMethod` | string | yes | Configuration injection method: `ignition`, `cloud-init`, or `kickstart` |
| `compatibility` | string[] | no | List of ComputeInstanceClass IDs this image is validated for. If empty, no compatibility check is performed. |
| `checksum` | string | no | Image checksum for integrity verification (e.g., `sha256:abc123...`). Used by bare-metal backends (Metal3/Ironic) to verify the image after download. |
| `status` | ImageStatus | auto | Current operational status (state + message). States: PENDING, READY, FAILED. |

**Validation at ComputeInstance creation:**

- If the Image has a non-empty `compatibility` list, the selected
  ComputeInstanceClass must be in that list. Otherwise the request
  is rejected with 400.
- If the Image has an `architecture` and the ComputeInstanceClass
  has a fixed architecture in its capabilities, they must match.

**Boot method usage:**

The ComputeInstanceTemplate role receives `image_boot_method` as a
variable. The role uses this to determine how to inject `sshKeys`
and `userData`:

| Boot method | SSH key injection | userData format |
|---|---|---|
| `ignition` | `passwd.users[].sshAuthorizedKeys` | Ignition JSON |
| `cloud-init` | `ssh_authorized_keys` in userdata | cloud-config YAML |
| `kickstart` | `%post` script | Kickstart directives |

### SSHKey (tenant-managed)

An SSHKey represents an SSH public key owned by a tenant user.

```yaml
id: "a1b2c3d4"
name: "my-workstation"
publicKey: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... user@workstation"
fingerprint: "SHA256:xyzabc123..."
```

Fields:

| Field | Type | Required | Mutable | Description |
|---|---|---|---|---|
| `id` | string | auto | no | Unique identifier |
| `name` | string | yes | yes | Human-readable name (unique per tenant) |
| `publicKey` | string | yes | yes | SSH public key (OpenSSH format) |
| `fingerprint` | string | computed | no | Key fingerprint (computed on create/update) |

**Scoping:** SSHKeys are scoped to the authenticated user's
organization. A tenant-admin can list all keys in their org.
A tenant-user can only see and manage their own keys.

**Lifecycle:** SSHKeys exist independently of ComputeInstances.
Creating, updating, or deleting an SSHKey does not affect running
instances. Key injection happens at instance provisioning time —
the template role reads the keys and injects them into the OS
configuration. Updating a key after provisioning does not
retroactively update instances (same as AWS Key Pairs).

### ComputeInstance changes

The `image` and `sshKey` inline fields are deprecated. New fields
`imageRef` and `sshKeyRefs` are added alongside them:

**Before (current):**

```yaml
spec:
  image:
    sourceType: "registry"
    sourceRef: "quay.io/rhel:9.6"
  sshKey: "ssh-ed25519 AAAA..."
```

**After:**

```yaml
spec:
  imageRef: "rhel-9.6-gpu"
  sshKeyRefs:
    - "my-workstation"
    - "ci-deploy-key"
```

- `imageRef` (new field) references an Image resource by name.
  The inline `image` object field is deprecated but preserved for
  backward compatibility.
- `sshKeyRefs` (new field) is a list of SSHKey name references.
  The inline `sshKey` string field is deprecated but preserved.

**Reference resolution:** At creation time, the fulfillment-service
resolves the references and denormalizes the data into the
ComputeInstance record:

- `resolvedImage` — a point-in-time snapshot of the Image resource
  (sourceType, sourceRef, bootMethod, checksum). This is stored on
  the private ComputeInstance spec so that the operator and AAP
  workflow can read all image data without API callbacks.
- `resolvedSshKeys` — the actual SSH public key strings resolved
  from the SSHKey resources.

This denormalization is intentional: the instance uses the image as
it was when created, not as it may be updated later (same semantics
as AWS AMIs). If an Image is deleted, the instance retains its
resolved data.

### API Extensions

#### New resources

| Resource | API path | Methods | Access |
|---|---|---|---|
| Image | `/api/fulfillment/v1/images` | GET (list) | All tenants |
| Image | `/api/fulfillment/v1/images/{id}` | GET, POST, PATCH, DELETE | Provider-only for mutation |
| SSHKey | `/api/fulfillment/v1/ssh_keys` | GET (list), POST | Tenant (own keys) |
| SSHKey | `/api/fulfillment/v1/ssh_keys/{id}` | GET, PATCH, DELETE | Tenant (own keys) |

#### Modified resources

| Resource | Change |
|---|---|
| ComputeInstance | Add `imageRef` (string reference to Image), `sshKeyRefs` (list of SSHKey references), `resolvedImage` (denormalized snapshot), `resolvedSshKeys` (resolved public keys). Deprecate inline `image` object and `sshKey` string. |

### Implementation Details/Notes/Constraints

#### Database

New tables (generic schema):
- `images` — provider-defined catalog
- `ssh_keys` — tenant-managed keys

No CRDs needed — these are catalog/data resources, not
reconciliation targets.

#### Migration

- Existing ComputeInstance records with inline `image` objects
  are preserved in `data` jsonb. The fulfillment-service accepts
  both inline objects (legacy) and string references (new) during
  a transition period.
- Existing ComputeInstance records with `sshKey` (singular) are
  read as equivalent to `sshKeys: [<inline-key>]`.

### Risks and Mitigations

#### Image catalog management burden

Providers must create and maintain Image resources.

*Mitigation:* A default set of common images (RHEL, RHCOS, Ubuntu)
can be shipped as sample data. Providers customize for their
environment.

#### SSHKey lifecycle disconnect

Updating an SSHKey does not update running instances.

*Mitigation:* Document this clearly. This matches public cloud
behavior (AWS, GCP, Azure all work this way). For key rotation on
running instances, tenants use OS-level mechanisms
(`ssh-copy-id`, Ansible, etc.).

### Drawbacks

- Adds two new resource types to the API surface.
- Requires migration of existing inline image/sshKey data.

## Alternatives (Not Implemented)

### Keep image and SSH key inline

Continue with the current vm-api-fields model.

*Why not:* No image catalog, no validation, no boot method
awareness, no key lifecycle.

### Image catalog without SSHKey resource

Decouple Image but keep SSH keys inline.

*Why not:* SSH keys have independent lifecycle needs (rotation,
revocation, audit) that justify a separate resource. Every public
cloud provides this.

## Open Questions [optional]

1. **Image versioning.** Should each OS version be a separate
   Image resource (e.g., `rhel-9.5-gpu`, `rhel-9.6-gpu`), or
   should a single Image resource carry a version field that can
   be updated? Separate resources are simpler but create catalog
   sprawl.

2. **SSHKey deletion policy.** Should deleting an SSHKey be blocked
   if it is referenced by running ComputeInstances, or allowed
   regardless? AWS allows deletion (keys are injected at launch
   time only). Blocking prevents accidental loss of access metadata
   but adds lifecycle coupling.

## Test Plan

TBD

## Graduation Criteria

TBD

## Upgrade / Downgrade Strategy

Inline `image` and `sshKey` fields on ComputeInstance are
preserved and never removed. The fulfillment-service accepts both
inline objects (legacy) and string references (new) during the
transition period. No tenant action is required on upgrade.

On downgrade, Image and SSHKey table records remain in PostgreSQL
but are inaccessible to older code. Existing ComputeInstances
with `resolvedImage` and `resolvedSshKeys` continue to work
because the resolved data is stored directly on the instance
record.

## Version Skew Strategy

Image and SSHKey are fulfillment-service PostgreSQL resources
with no CRDs and no hub cluster representation. Reference
resolution happens in the fulfillment-service at ComputeInstance
creation time. The osac-operator and AAP receive only the
resolved data (`resolvedImage`, `resolvedSshKeys`) — they are
unaware of Image and SSHKey resources.

## Support Procedures

- **400 on imageRef:** The Image resource does not exist, or its
  `compatibility` list does not include the selected
  ComputeInstanceClass. Check `GET /api/fulfillment/v1/images`
  and verify the class ID is in the compatibility list.
- **400 on sshKeyRefs:** One or more SSHKey names do not exist or
  belong to a different organization. Check
  `GET /api/fulfillment/v1/ssh_keys` for the tenant.
- **Stale resolvedImage after Image update:** By design —
  instances use the image as it was at creation time. Re-provision
  to pick up updated image data.

## Infrastructure Needed [optional]

None.
