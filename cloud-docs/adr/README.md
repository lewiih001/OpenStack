# Architecture decision records

An ADR records one significant decision: the context, the options, the
choice and its consequences. The decision in an accepted ADR is never
edited. To change a decision, write a new ADR that supersedes the old one
and update the old one's status line only.

Editorial corrections that do not change the decision (factual fixes,
clarifications, typos) are allowed through a normal pull request whose
commit message states what was corrected and why.

## Process

1. Copy `0000-template.md` to `NNNN-short-title.md` using the next number.
2. Open a pull request with status **Proposed**.
3. Add a row to the index below in the same commit.
4. Merge with status **Accepted** once decided. An ADR may also be merged as **Proposed** to record an
   open decision; its status line is updated in a later pull request when it is accepted or rejected.

## Index

| ADR | Title | Status | Date |
| --- | --- | --- | --- |
| [0001](0001-use-github-for-source-control-and-ci.md) | Use GitHub for source control, reviews and CI | Accepted | 2026-10-23 |
| [0002](0002-workstation-as-interim-management-host-and-lab.md) | Use the GPU workstation as the interim management host and lab | Proposed | 2026-10-23 |
| [0003](0003-monorepo-with-one-folder-per-planned-repository.md) | Start as a monorepo with one folder per planned repository | Accepted | 2026-10-23 |
| [0004](0004-release-baseline-and-version-pinning.md) | Release baseline and version pinning | Accepted | 2026-10-23 |
| [0005](0005-ubuntu-24-04-lts-as-host-and-kolla-base-os.md) | Ubuntu 24.04 LTS as host OS and Kolla base distribution | Accepted | 2026-10-23 |
| [0006](0006-network-design-vlans-mtu-and-ovn.md) | Network design: VLAN layout, MTU and OVN | Accepted | 2026-10-23 |
| [0007](0007-ceph-as-unified-storage-backend.md) | Ceph as the unified block, object and file storage backend | Accepted | 2026-10-23 |
| [0008](0008-opentofu-for-infrastructure-as-code.md) | OpenTofu for infrastructure as code | Accepted | 2026-10-23 |
| [0009](0009-argo-cd-for-gitops.md) | Argo CD for GitOps delivery to Kubernetes | Accepted | 2026-10-23 |
| [0010](0010-keycloak-as-identity-provider-federated-to-keystone.md) | Keycloak as the identity provider, federated to Keystone | Accepted | 2026-10-23 |
| [0011](0011-backup-policy.md) | Backup and restore policy | Accepted | 2026-10-23 |
| [0012](0012-naming-conventions.md) | Naming conventions | Accepted | 2026-10-23 |
| [0013](0013-upgrade-cadence.md) | Upgrade cadence | Accepted | 2026-10-23 |
| [0014](0014-region-and-availability-zone-names.md) | Region and availability zone names | Accepted | 2026-10-23 |
| [0015](0015-keystone-domain-and-project-layout.md) | Keystone domain and project layout | Accepted | 2026-10-23 |
