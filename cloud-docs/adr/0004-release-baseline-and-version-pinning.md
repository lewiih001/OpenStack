# ADR-0004: Release baseline and version pinning

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations), applies to all phases

## Context

The platform combines independently released projects: OpenStack, Ceph,
Kubernetes, operators and tooling. Each has its own release cadence and
support window. An unpinned dependency (`latest` image tag, floating
chart version, unversioned module) can change behaviour between a lab
rehearsal and a production deploy, which defeats lab-first testing.

## Decision drivers

- What is rehearsed in the lab is exactly what reaches production
- Stay inside upstream support windows
- Prefer releases that allow skip-level upgrades
- Avoid brand-new minor or major releases in production until patched

## Options considered

1. **Track latest everywhere**: newest features, but frequent breakage and no reproducibility.
2. **Pin to versions chosen ad hoc**: reproducible, but drifts out of support with no rule for when to move.
3. **Pin every version in Git, chosen by a written policy per component.**

## Decision

We will pin every version in Git (image tags, Helm chart versions,
OpenTofu provider and module versions, Kolla release branch, tool
versions in CI) and choose versions by this baseline:

| Component | Baseline (October 2026) | Selection rule |
| --- | --- | --- |
| Host OS | Ubuntu 24.04 LTS | LTS only (ADR-0005) |
| OpenStack | 2026.1 Gazpacho, Kolla-Ansible `stable/2026.1` | SLURP releases only |
| Ceph | 20.2.x Tentacle via cephadm | Latest patch of the current stable major; a new major only after its first point releases |
| Kubernetes (platform clusters) | 1.36.x | N-1 minor in production; move to the newest minor once it has reached at least patch .3 |
| Cluster API OpenStack provider | 0.15.x | Version compatible with the Kubernetes minor in use |
| OpenTofu | 1.13.x | Latest minor; provider `terraform-provider-openstack` pinned with a lock file |
| Argo CD | 3.5.x | Latest minor after its first patch |
| Keycloak | 26.x | Latest supported major |
| CloudNativePG | 1.30.x | Latest minor |

The table records the decision at this date. The exact pinned versions
live in code, and a version bump is a pull request rehearsed in the lab
like any other change.

This supersedes the work plan's Kubernetes 1.35 baseline. Kubernetes 1.37
is current upstream, so 1.36 is the N-1 choice. Local learning
clusters (`kind`) may run the newest version.

## Consequences

### Positive

- Lab and production run byte-identical versions.
- Upgrades are planned events (ADR-0013), not surprises.
- SLURP releases allow OpenStack upgrades once a year instead of every six months.

### Negative

- Pinned versions age, and security patches require deliberate bumps. Mitigation: a monthly patch review and automated update pull requests (Renovate) once CI is complete in Phase 4.
- N-1 Kubernetes means new features arrive about four months later. Accepted.

## Revisit when

- An upstream project changes its support policy or release cadence.
- A security advisory requires moving faster than the selection rule allows.

## References

- Work plan: "Software baseline"
- ADR-0005: Host operating system
- ADR-0013: Upgrade cadence
