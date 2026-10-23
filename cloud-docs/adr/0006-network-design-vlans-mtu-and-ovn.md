# ADR-0006: Network design: VLAN layout, MTU and OVN

- **Status:** Accepted, with the public address block still to be allocated
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations), built in Phase 1

## Context

Network layout is the hardest thing to change once tenants exist. VIPs,
provider networks and floating IPs all depend on it. The starter
footprint is one rack with two top-of-rack switches and six servers,
each with two 25 GbE ports, plus out-of-band management.

## Decision drivers

- Separate traffic classes so a storage storm cannot starve the APIs
- Tenant traffic overlay with a standard 1500-byte inner MTU
- Distributed routing so no single network node is a bottleneck
- Management networks never reachable from the internet or tenants

## Options considered

1. **Flat network**: simple, but no isolation and no QoS by traffic class.
2. **VLAN-segmented underlay with Neutron ML2/OVS**: proven, but OVS agents and centralised L3 agents are more moving parts.
3. **VLAN-segmented underlay with Neutron ML2/OVN** and distributed floating IPs.

## Decision

We will use two switches in MLAG, each server dual-homed with an LACP
bond (`bond0`), and these VLANs:

| VLAN | Name | Purpose | Subnet | MTU |
| --- | --- | --- | --- | --- |
| 10 | oob | BMC, IPMI, Redfish | 10.10.0.0/24 | 1500 |
| 20 | provision | MAAS PXE (DHCP only here) | 10.20.0.0/24 | 1500 |
| 30 | mgmt | Internal APIs, SSH, MariaDB, RabbitMQ | 10.30.0.0/24 | 9000 |
| 40 | tunnel | Geneve overlay between hypervisors | 10.40.0.0/24 | 9000 |
| 50 | storage | Ceph public network | 10.50.0.0/24 | 9000 |
| 60 | replication | Ceph cluster network | 10.60.0.0/24 | 9000 |
| 70 | external | Public API VIP, router gateways, floating IPs | public block (to be allocated) | 1500 |
| 80 | lb-mgmt | Octavia amphora management | 10.80.0.0/22 | 1500 |

- Neutron uses **ML2/OVN** with **distributed floating IPs**. Tenant networks use Geneve on VLAN 40, keeping a 1500-byte inner MTU over the 9000-byte underlay.
- The internal VIP is `10.30.0.100`. Each environment uses a different `keepalived_virtual_router_id`.
- Security zones:
  - public: VLAN 70
  - guest: tenant overlay on VLAN 40
  - management: VLANs 10, 20, 30, 80, never routed to the internet or tenant networks
  - data: VLANs 50, 60

  Traffic crosses zones only through the firewalls or HAProxy.
- The lab uses the same VLAN IDs on its own switches, with different subnets recorded in NetBox.
- **Public block:** request a /24 from the ISP; a /26 is the minimum (day one needs about 50 public addresses). Recorded in the deployment plan, not here, when allocated.

NetBox is the source of truth for every address. This ADR fixes the
layout; individual allocations live in NetBox.

## Consequences

### Positive

- Storage, overlay and API traffic are isolated and can be monitored separately.
- OVN removes the Neutron L3, DHCP and metadata agents. Traffic for floating IPs leaves directly from the hypervisor hosting the VM. Source-NAT traffic for VMs without a floating IP leaves through gateway chassis on the control nodes, scheduled with HA across all three.
- One layout serves lab and production.

### Negative

- Jumbo frames must be configured end to end. Mitigation: a Phase 1 exit check with `ping -M do -s 8972` across VLANs 40, 50 and 60.
- VLAN 70 must reach every hypervisor (distributed floating IPs). Multi-rack growth requires stretching it or using BGP. Recorded in the scale-out path.
- Until the public block is allocated, external endpoints cannot be finalised.

## Revisit when

- The platform grows beyond one rack (spine-leaf design and BGP for floating IPs).
- A second site is added (separate region, ADR-0014).

## References

- Work plan: "Network", "Capacity model"
- ADR-0014: Region and availability zones
