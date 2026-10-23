# Design document: OpenStack private cloud platform

- **Status:** Living document, reviewed at every phase gate
- **Owner:** Lewis (lewiih001)
- **Last reviewed:** 2026-10-23 (Phase 0)

This document records **why** the platform is built the way it is. It
summarises decisions and points to the ADR that made each one. It is not
a tutorial: procedures live in `cloud-docs/runbooks/`, and addresses and
hardware details live in the deployment plan and NetBox.

## 1. Background

The platform is a private, AWS-like cloud in Nairobi for deploying
Lewis's products (starting with Tek Inventory) and, later, customer
workloads. Products need both stateless services and stateful databases
(PostgreSQL, MySQL, Valkey, Kafka, RabbitMQ). They are delivered on
virtual machines and on Kubernetes.

The platform starts small (six servers plus a management host and a lab),
is built entirely from code, and grows by adding nodes without
redesign. It must be operable by a very small team, which drives three
principles:

1. **Everything as code.** No change reaches production except through a reviewed pull request (ADR-0001, ADR-0008, ADR-0009).
2. **Lab first.** Every change, including every upgrade, is rehearsed in a lab running the same versions (ADR-0002, ADR-0004, ADR-0013).
3. **One tool per layer.** Each layer is owned by exactly one tool, which changes only that layer (section 4).

## 2. Requirements

Functional requirements are written as user stories with numeric SLAs
in [`cloud-docs/requirements/`](../requirements/README.md) (US-001 to
US-020 at Phase 0). Each becomes an automated test and then a
production monitoring check.

Platform-level service level objectives (v1.0):

| Service | SLO |
| --- | --- |
| OpenStack APIs and dashboard | 99.9% monthly availability |
| Kubernetes API (prod) | 99.9% |
| PostgreSQL (per database) | 99.95%, RPO 5 min, RTO 30 min |
| Object storage | 99.9% |
| Full site loss (single site) | RPO 24 h, RTO 2 to 3 days (ADR-0011) |

Non-functional requirements:
- **Security:** MFA for every person (ADR-0010), TLS on every internal and external endpoint, secrets only SOPS-encrypted in Git, CIS Level 1 hosts (ADR-0005).
- **Capacity:** survive the loss of one hyperconverged node with all platform services running (N+1, section 3.3).
- **Supportability:** only upstream-supported releases in production (ADR-0004, ADR-0013).

## 3. Physical architecture

### 3.1 Starter footprint

| Role | Qty | CPU | RAM | Disks | Network | Runs |
| --- | --- | --- | --- | --- | --- | --- |
| Control and network (`ctl`) | 3 | 2 × 16 cores | 256 GB | 2 × 960 GB SSD, RAID 1 | 2 × 25 GbE bonded + BMC | OpenStack APIs, OVN central, MariaDB Galera, RabbitMQ, HAProxy/Keepalived, Octavia, Designate, Ceph MON/MGR |
| Hyperconverged (`hci`) | 3 | 2 × 32 cores | 512 GB (768 GB preferred) | 2 × 480 GB SSD OS + 6 × 3.84 TB NVMe | 2 × 25 GbE bonded + BMC | Nova compute (KVM), OVN controller, Ceph OSDs, RGW |
| Management (`mgmt`) | 1 | 16 cores | 128 GB | 2 × 3.84 TB SSD, RAID 1 | 2 × 10 GbE + BMC | MAAS, Kolla-Ansible, cephadm, CI runner, registry and package mirrors |
| Lab (`lab`) | 3 | 1 × 16 cores | 128 GB | 1 × 480 GB + 2 × 1.92 TB SSD | 2 × 10 GbE | Small full replica of the stack |

Plus two top-of-rack switches in MLAG, two firewalls, two upstream
internet links, A/B power on separate UPS (about 6 kW and 14U at start),
and an out-of-band path (separate router or LTE modem) to the BMC network.

**Interim (Phase 0 to 1):** until this hardware exists, a 24 GB
workstation acts as management host and runs an all-in-one lab in a VM
(ADR-0002, Proposed).

### 3.2 Network

Two switches in MLAG, each server dual-homed with an LACP bond. Eight
VLANs separate traffic classes. Jumbo frames run on internal data paths,
so Geneve tenant traffic keeps a 1500-byte inner MTU. Neutron uses
ML2/OVN with distributed floating IPs. Full VLAN table, security zones
and rationale: **ADR-0006**.

```
                 Internet (2 ISPs)
                        |
                 [ fw01 ]  [ fw02 ]
                        |
      VLAN 70 external (API VIP, floating IPs)
                        |
   +---------- [ tor01 ]==MLAG==[ tor02 ] ----------+
   |    LACP bond0 per server, VLANs 30-80 tagged   |
   |                                                |
 ctl01..03                 hci01..03            mgmt01
 (APIs, DBs, OVN,          (KVM, OSDs,          (MAAS, Kolla,
  Ceph MON/MGR)             OVN controller)      runner, mirrors)

 VLAN 10 oob ── BMCs ── separate OOB router / LTE
```

### 3.3 Storage and capacity

Ceph (cephadm, external to Kolla) provides block, object and file
storage with replication size 3 across hosts. Usable capacity is about
16 TB at a 70% fill ceiling from 69 TB raw (**ADR-0007**).

RAM, not CPU, is the binding limit. Per hyperconverged node, 448 GB is
available to guests after reserving 64 GB and 8 cores for the host and
OSDs. With one node down (N+1), the cluster offers 896 GB of guest RAM.
The platform's own Kubernetes clusters and load balancers take about
480 GB of that (53%). Buying 768 GB nodes lowers this to about 34%.
Re-run the capacity model at every monthly review.

Public addresses: about 50 needed on day one. Request a /24, with /26
as the minimum (ADR-0006).
