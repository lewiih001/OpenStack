# ADR-0013: Upgrade cadence

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Bootstrap), applies after go-live

## Context

Pinning versions (ADR-0004) prevents surprise changes, but pinned
versions age out of support and accumulate security fixes. Each
upstream has a different cadence: OpenStack every six months (SLURP
yearly), Kubernetes roughly every four months with about fourteen
months of patch support, Ceph yearly majors. Without a schedule,
upgrades are deferred until they are emergencies.

## Decision drivers

- Never run an unsupported release in production
- Every upgrade rehearsed in the lab first
- Predictable maintenance windows, announced in advance
- Rollback or recovery path known before each upgrade

## Decision

| Layer | Cadence | Path |
| --- | --- | --- |
| OpenStack (Kolla-Ansible) | Yearly, SLURP to SLURP | 2026.1 to 2027.1, about three months after 2027.1 is released |
| Ceph | Yearly major, monthly patch review | `ceph orch upgrade` one major at a time, after the new major's first point releases |
| Kubernetes (platform clusters) | Every four months, one minor at a time | `stage` first, `prod` one week later, through Cluster API |
| Host OS | Monthly security patch window; major release with a SLURP upgrade | Live-migrate guests, patch one host at a time |
| Operators, charts and tools | Monthly review; patch releases as they ship | Automated update pull requests, lab first |
| Security fixes rated critical | Within 7 days, or immediately if exploited in the wild | Emergency change, same lab-first rule with a shortened soak |

Rules for every upgrade:
1. Read the upstream release notes and upgrade guide, and record the risks in the change pull request.
2. Take and verify backups (ADR-0011) immediately before.
3. Rehearse in the lab with the exact target versions and the production configuration.
4. Announce the maintenance window and silence only the alerts it is expected to trigger.
5. Upgrade production through a pull request, one layer at a time, never two layers in one window.
6. Soak at least one week before the next layer's upgrade.

The quarterly upstream release review (work plan, Phase 8) produces the
next quarter's upgrade schedule.

## Consequences

### Positive

- Production never falls out of upstream support.
- Upgrades become routine, low-drama events with rehearsed procedures.

### Negative

- Recurring engineering time every month and quarter. Accepted; it is cheaper than emergency upgrades.
- SLURP-only OpenStack upgrades mean new OpenStack features arrive up to a year late. Accepted for stability.

## Revisit when

- An upstream changes its release or support cadence.
- A second region exists (upgrade regions one at a time).

## References

- ADR-0004: Release baseline
- ADR-0011: Backup policy
- Work plan: "Release model"
