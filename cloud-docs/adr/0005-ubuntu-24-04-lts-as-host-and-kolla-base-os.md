# ADR-0005: Ubuntu 24.04 LTS as host OS and Kolla base distribution

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations), applies from Phase 1

## Context

Every physical server (control, hyperconverged, management, lab) needs
one operating system. Kolla-Ansible also builds its service containers
from a base distribution, and the host OS and container base should be
supported together by the same OpenStack release. MAAS must be able to
deploy the OS image.

## Decision drivers

- Supported by Kolla-Ansible 2026.1 as both host OS and container base
- Deployable by MAAS out of the box
- Long security support window
- One OS everywhere reduces operational knowledge to maintain

## Options considered

1. **Ubuntu 24.04 LTS**: Kolla-supported host and base, native MAAS images, standard security support to 2029 and longer with Ubuntu Pro.
2. **Rocky Linux**: supported by Kolla as host and container base, but MAAS support for it is weaker, and it would mix ecosystems with the Ubuntu-based tooling.
3. **Debian**: supported by Kolla, but a smaller ecosystem for MAAS images and Ceph packaging.

## Decision

We will run Ubuntu 24.04 LTS on every server, including the lab and the
management host, and use Ubuntu as the Kolla container base distribution
(`kolla_base_distro: "ubuntu"`).

Host baseline, applied by Ansible after MAAS deploys:
- CIS Ubuntu 24.04 Level 1
- chrony time sync
- SSH only from the management host
- swap disabled
- `unattended-upgrades` limited to security updates
- kernel updates in a monthly patch window (live-migrate guests first)

## Consequences

### Positive

- Host, containers, lab and production share one OS and one patch process.
- MAAS deploys it natively.

### Negative

- Ubuntu 24.04 standard support ends in 2029. Mitigation: plan a host OS upgrade with a later SLURP release, rehearsed in the lab.
- Single-distribution dependency. Accepted; Kolla supports others if ever needed.

## Revisit when

- A future Kolla-Ansible SLURP release drops Ubuntu 24.04 support.
- Ubuntu 26.04 LTS becomes a supported Kolla host OS and has had its first point release.

## References

- Kolla-Ansible support matrix for 2026.1
- ADR-0004: Release baseline
