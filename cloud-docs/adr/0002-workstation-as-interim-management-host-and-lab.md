# ADR-0002: Use the GPU workstation as the interim management host and lab

- **Status:** Proposed. To be accepted once the workstation's CPU, operating system and network interfaces are confirmed.
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations) to 1 (Bare metal and Ceph)

## Context

The work plan assumes a dedicated management host (deploy tooling, CI
runner, NetBox, PKI tooling) and a three-node lab that rehearses every
change before production. Neither exists yet. Production servers are not
procured.

The available hardware is:

- a macOS 13 Intel laptop, used for development, `kind` clusters and editing; too limited to run OpenStack
- one GPU workstation with 24 GB RAM and a 1 TB SSD, not always reachable

The three-node lab with Ceph and HA control plane needs real servers
(64 GB RAM or more each) and cannot be simulated on this hardware.

## Decision drivers

- Start Phase 1 work (Kolla-Ansible, OpenStack APIs, OpenTofu against a real cloud) without waiting for procurement
- Keep management services isolated from a lab that is rebuilt often
- Rebuild the lab from code in minutes, not hours
- Never let the interim setup leak into production design

## Options considered

1. **Wait for production-class hardware** before any OpenStack work: no risk of shortcuts, but it blocks learning and Phase 1 for months.
2. **Kolla-Ansible all-in-one directly on the workstation**, alongside the runner and NetBox: simplest, but Kolla takes over Docker and host networking, and every lab rebuild disrupts management services.
3. **Workstation as management host, with the all-in-one lab inside a KVM virtual machine.**
4. **Rent cloud servers for the lab**: works, but brings recurring cost and nested virtualisation limits on most providers.

## Decision

We will use the GPU workstation in two roles, separated by virtualisation:

| Role | Where it runs | Planned allocation |
| --- | --- | --- |
| Management host: GitHub self-hosted runner, NetBox (containers), deploy tooling | Host OS (Ubuntu 24.04 LTS) | about 6 GB RAM, about 150 GB disk |
| Lab: Kolla-Ansible 2026.1 all-in-one | KVM VM with nested virtualisation, two virtual NICs | 16 GB RAM, about 500 GB disk |
| Headroom for the host | Host OS | about 2 GB RAM |

The lab VM runs only the core services first: Keystone, Glance, Nova,
Neutron with OVN, Placement, Cinder (LVM backend) and Horizon. Heavier
services (Octavia, Magnum, Designate, Manila) are enabled one at a time
for testing and disabled afterwards. Before each rebuild the VM is
snapshotted, so a failed experiment can be rolled back in minutes.

Ceph, HA control plane and multi-node behaviour are **not** tested here.
They wait for the three-node lab.

## Consequences

### Positive

- Phase 1 can start now on real OpenStack APIs.
- Lab rebuilds never touch the runner or NetBox.
- Snapshots make destructive experiments cheap.
- The host, the lab and production all run the same OS (Ubuntu 24.04 LTS).

### Negative

- **Single point of failure.** One machine holds both management and lab. Mitigation: NetBox data is backed up off the machine, and everything else is rebuilt from the repository.
- **Not always reachable.** CI jobs that need the self-hosted runner wait in the queue while it is offline. Mitigation: lint and validation jobs use GitHub-hosted runners. Kubernetes platform work runs on `kind` on the laptop.
- **Nested virtualisation is slower** than bare metal. Acceptable for functional tests; not used for performance numbers.
- **The 24 GB RAM limit caps lab scope.** Mitigation: enable heavy services one at a time.
- **Risk of lab shortcuts reaching production.** Mitigation: the lab inventory lives under `infra-live/lab/`, production under `infra-live/prod/`, and production-only settings (HA, Ceph, TLS everywhere) are reviewed against the design document, not against the lab.

## Revisit when

- Production-class lab servers (three nodes, 64 GB RAM or more each) are available: the lab moves there and this workstation keeps only the management role or is retired.
- Production is deployed: the management host moves to dedicated hardware inside the management network.
- The workstation turns out not to support nested virtualisation (fall back to option 2, with the runner and NetBox moved to the laptop or a small VPS).

## To accept this ADR

- CPU model and core count (`lscpu`), and whether virtualisation is enabled in firmware (`egrep -c '(vmx|svm)' /proc/cpuinfo` is greater than 0)
- Operating system currently installed
- Number of physical network interfaces

## References

- Work plan, Phase 1: lab
- ADR-0001: GitHub runner placement
- Kolla-Ansible quickstart (all-in-one requirements: 2 network interfaces, 8 GB RAM minimum, 40 GB disk)
