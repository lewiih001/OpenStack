# ADR-0007: Ceph as the unified block, object and file storage backend

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Bootstrap), built in Phase 1

## Context

The cloud must offer block volumes (EBS-like), object storage (S3-like),
shared file systems (EFS-like), image storage and VM root disks. Tenant
VMs must be able to restart on another hypervisor when one fails, which
requires their disks to live on shared storage rather than local disks.

## Decision drivers

- One storage system for block, object, file and images
- No single point of failure; survive the loss of one node
- Live migration and evacuation without copying disks
- Native integration with Cinder, Glance, Nova, Manila and Kubernetes CSI

## Options considered

1. **Local disks plus LVM/iSCSI for Cinder, Swift for objects**: several systems to run, and VM disks are tied to their hypervisor.
2. **Ceph deployed by Kolla-Ansible**: no longer supported by Kolla.
3. **External Ceph cluster deployed with cephadm**, consumed by OpenStack and Kubernetes.

## Decision

We will run Ceph 20.2 Tentacle deployed with **cephadm**, outside Kolla,
and consume it as an external cluster:

| Service | Ceph component | Pool or service |
| --- | --- | --- |
| Cinder volumes | RBD | `volumes` |
| Glance images | RBD | `images` |
| Nova ephemeral and root disks | RBD | `vms` |
| Cinder backups | RBD | `backups` |
| S3 object storage | RGW, behind Kolla HAProxy | RGW pools |
| Manila shared file systems | CephFS | CephFS volume |
| Kubernetes persistent volumes | RBD and CephFS via Ceph CSI | per-cluster pools |

Layout:
- MONs and MGRs on the three control nodes.
- OSDs on the NVMe drives of the three hyperconverged nodes, from a spec file in `infra-baremetal` that never selects OS disks.
- Replication size 3, failure domain host.

Operating rules:
- PG autoscaler on
- `osd_memory_target` 8 GB on hyperconverged nodes
- dashboard and Prometheus module enabled
- the cluster never exceeds 70% full
- failed disks replaced with `ceph orch osd rm <id> --replace`

## Consequences

### Positive

- One system serves every storage service, with one set of skills and one set of alerts.
- VM disks are shared, so live migration and evacuation work without copying data.
- Replication size 3 across hosts survives the loss of one node.

### Negative

- Usable capacity is raw ÷ 3, and planning stays under 70%: 69 TB raw gives about 16 TB safely usable at start.
- Hyperconverged nodes share CPU and RAM with OSDs. Mitigation: reserve 8 cores and 64 GB per node (`reserved_host_memory_mb`).
- Ceph is operationally demanding. Mitigation: lab rehearsal of disk replacement, node loss and monitor recovery before production.

## Revisit when

- Around eight compute nodes: move OSDs to dedicated storage nodes in sets of three.
- A Ceph major release passes its first point releases (ADR-0013).

## References

- Work plan: Phase 1.3 Ceph with cephadm
- ADR-0004: Release baseline
