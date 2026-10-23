# Storage

### US-011: Create and attach a block volume

| Field | Value |
| --- | --- |
| Story | As a tenant, I create a volume and attach it to my VM, so that my data outlives the VM, like EBS. |
| Input | 100 GB volume of type `ssd`, target VM |
| Output | Volume `in-use`, visible in the guest as a new disk |
| SLA | Created and attached within 30 s (p95); 99.9% success per month |
| Verified by | Canary: create, attach, detach and delete a 1 GB volume every 5 min |
| Phase | 2 |

### US-012: Store and fetch objects over S3

| Field | Value |
| --- | --- |
| Story | As a tenant, I use any S3 client with my EC2-style credentials, so that my applications store files without code changes. |
| Input | `openstack ec2 credentials create`, then `aws s3 cp` with the RGW endpoint |
| Output | Objects stored in my bucket, readable with the same credentials and denied to others |
| SLA | 1 MiB put and get each within 1 s (p95); 99.9% availability per month |
| Verified by | Canary: put, get and delete an object every 5 min against the public endpoint |
| Phase | 1 (RGW), exposed to tenants in Phase 2 |

### US-013: Share a file system between VMs

| Field | Value |
| --- | --- |
| Story | As a tenant, I mount one shared file system on several VMs, so that they read and write the same files, like EFS. |
| Input | Manila CephFS share, access rule for my client, mounted on two VMs |
| Output | A file written on one VM is readable on the other |
| SLA | Share available within 2 min of creation; a write visible on the second VM within 5 s |
| Verified by | Lab test: create share, mount on two VMs, write and read a checksum file |
| Phase | 6 |

### US-014: Back up and restore a volume

| Field | Value |
| --- | --- |
| Story | As a tenant, I back up a volume and restore it, so that I recover from deletion or corruption. |
| Input | Cinder backup of a 10 GB volume, then restore to a new volume |
| Output | Restored volume with a checksum identical to the original |
| SLA | Backup and restore of 10 GB each within 15 min; checksum match 100% |
| Verified by | Monthly restore drill (ADR-0011), logged in `cloud-docs/runbooks/` |
| Phase | 2 (feature), 7 (drill) |
