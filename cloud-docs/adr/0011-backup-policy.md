# ADR-0011: Backup and restore policy

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations), applies from Phase 1

## Context

Replication (Ceph size 3, Galera, CloudNativePG replicas) protects
against hardware failure, not against deletion, corruption, ransomware
or a site loss. Each layer needs a defined backup, retention and tested
restore. "Nothing holds real data until it is monitored and backed up"
is a release rule in the work plan.

## Decision drivers

- Every stateful component has a named backup method, schedule and retention
- Restores are tested regularly, not assumed
- Backups cannot be deleted by a compromised admin account within retention
- An offsite copy survives the loss of the site

## Options considered

1. **Rely on replication only**: no protection against deletion or corruption.
2. **One backup product for everything**: simpler, but poorly suited to databases (point-in-time recovery) and Kubernetes objects.
3. **Native backup per layer, all landing in object storage with Object Lock and replicated offsite.**

## Decision

We will back up each layer with its native tool:

| What | Method | Schedule | Retention | Restore target |
| --- | --- | --- | --- | --- |
| Infrastructure code, Kolla config and inventory | Git (GitHub plus every clone) | Every change | Forever | Minutes |
| OpenTofu state | RGW state bucket with versioning (ADR-0008) | Every apply | 90 days of versions | Minutes |
| OpenStack databases (MariaDB) | `kolla-ansible mariadb_backup`, to RGW | Daily full, hourly incremental | 30 days | 1 h |
| OVN databases | Not backed up; rebuilt from Neutron (`neutron-ovn-db-sync-util` repair), rehearsed | — | — | 1 h |
| Tenant volumes | Cinder Backup to Ceph, opt-in per project | Daily | 14 days | 1 h per volume |
| PostgreSQL (CloudNativePG) | Base backup and continuous WAL to RGW (Barman Cloud plugin) | Daily + continuous | 30 days | 30 min, point-in-time |
| Kubernetes objects and PVCs | Velero with CSI snapshots to RGW | Daily | 14 days | 1 h per namespace |
| Cluster API management state | `clusterctl move --to-directory` to RGW | Daily | 30 days | 2 h |
| Harbor, Keycloak, NetBox | Their own backup tools to RGW | Daily | 30 days | 2 h |
| Offsite copy of all backup buckets | Replication to a second site or external S3 provider | Daily | 90 days | 24 h |

Rules:
- Backup buckets use **Object Lock** (compliance mode within retention).
- Every backup job exports success and age metrics. A backup older than 26 hours pages on-call.
- A **restore drill** runs on the first Monday of every month and is logged in `cloud-docs/runbooks/`.
- Encryption keys needed to restore (SOPS age keys, Barbican KEK) are backed up separately from the data they protect.
- Until RGW exists (Phase 1), Phase 0 artefacts (NetBox, age keys) are backed up to two separate encrypted offline disks.

Recovery objectives for a full site loss today: RPO 24 h, RTO 2 to 3
days, by rebuilding from code plus the offsite copy.

## Consequences

### Positive

- Every stateful component has a named, tested restore path.
- Object Lock protects backups from deletion or ransomware during retention.

### Negative

- Storage cost: backups and the offsite copy roughly double data stored. Included in capacity reviews.
- Monthly drills take engineering time. Accepted; untested backups are not backups.
- Site-loss recovery is slow until a second region exists. Accepted for the starter footprint.

## Revisit when

- A second site exists: move to RBD mirroring, RGW multisite and CNPG replica clusters (RPO minutes, RTO hours).
- A product or customer requires a stricter RPO or RTO.

## References

- Work plan: Phase 7.3 Backups, 7.4 Disaster recovery
- ADR-0007: Ceph
