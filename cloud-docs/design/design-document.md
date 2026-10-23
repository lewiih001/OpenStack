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

## 4. Service architecture

### 4.1 Layers and owning tools

The platform is a stack of layers. Each one is built on the layer below
and owned by exactly one tool. A change to a layer is made only through
its tool, from code in its folder (ADR-0003).

| Layer | Owning tool | Code in | Decided in |
| --- | --- | --- | --- |
| Bare metal and OS | MAAS + Ansible | `infra-baremetal/` | ADR-0005 |
| Storage | cephadm | `infra-baremetal/` (specs) | ADR-0007 |
| IaaS (OpenStack 2026.1) | Kolla-Ansible | `infra-openstack/` | ADR-0004, ADR-0006 |
| Cloud resources and tenants | OpenTofu | `infra-live/`, `tofu-modules/` | ADR-0008, ADR-0015 |
| Kubernetes clusters | Cluster API (CAPO), via OpenTofu | `infra-live/` | ADR-0004 |
| Platform and data services | Argo CD | `platform-gitops/` | ADR-0009 |
| Identity (all layers) | Keycloak federated to Keystone | `platform-gitops/` | ADR-0010 |

Source control and CI run outside the cloud they build: GitHub, with a
self-hosted runner on the management host for jobs that need private
network access (ADR-0001).

### 4.2 Service catalogue

| AWS equivalent | Platform service | Built with | Phase |
| --- | --- | --- | --- |
| EC2 | Virtual machines | Nova + KVM, Placement | 2 |
| EBS | Volumes, snapshots, backups | Cinder on Ceph RBD, Cinder Backup | 2 |
| S3 | Object storage (S3 and Swift APIs) | Ceph RGW with Keystone auth | 1 to 2 |
| EFS | Shared file systems | Manila on CephFS | 6 |
| VPC | Networks, routers, security groups, floating IPs | Neutron with OVN | 2 |
| ELB | Layer 4/7 load balancers | Octavia (active-standby) | 2 |
| Route 53 | DNS zones and records | Designate | 2 |
| KMS / Secrets Manager | Keys and secrets | Barbican; OpenBao + External Secrets | 2, 4 |
| IAM | Users, projects, roles, SSO | Keystone federated to Keycloak | 2, 4 |
| EKS | Managed Kubernetes | Cluster API Provider OpenStack | 3 |
| ECR | Registry with scanning and signing | Harbor + Trivy + Cosign | 4 |
| API Gateway | Public APIs, auth, rate limits | Kong via Gateway API | 4 |
| Lambda / Fargate | Functions and containers without a cluster | Knative | 6 |
| RDS PostgreSQL | Managed PostgreSQL with HA and point-in-time recovery | CloudNativePG + Barman Cloud plugin | 5 |
| RDS MySQL | Managed MySQL | Percona Operator for MySQL | 5 |
| ElastiCache | In-memory cache | Valkey | 5 |
| MSK / Amazon MQ | Kafka and RabbitMQ | Strimzi (KRaft), RabbitMQ Cluster Operator | 5 |
| CloudWatch | Metrics, logs, traces, alerts | Prometheus, Grafana, Loki, Tempo, Alertmanager, OpenTelemetry | 1 to 7 |
| AWS Backup | Backups and restores | Native tool per layer (ADR-0011) | 1 to 7 |
| CloudFormation | Infrastructure as code | OpenTofu + OpenStack provider | 0, 8 |
| Billing | Usage metering and showback | CloudKitty | 8 |

### 4.3 Kubernetes clusters

| Cluster | Purpose |
| --- | --- |
| `nbo1-mgmt` | Cluster API management, Argo CD |
| `nbo1-svc` | Shared platform services: Harbor, Keycloak, OpenBao, observability |
| `nbo1-stage` | Pre-production for products, one week ahead of prod on upgrades |
| `nbo1-prod` | Production products and data services |

All clusters run Kubernetes 1.36.x (ADR-0004), are created by Cluster
API from OpenTofu, and are configured only by Argo CD.

### 4.4 Explicitly not used

| Avoided | Reason | Instead |
| --- | --- | --- |
| Ingress-NGINX | Retired upstream | Gateway API with Kong |
| Bitnami images and charts | Free catalogue withdrawn | Upstream or operator-maintained images |
| Kafka with ZooKeeper | ZooKeeper removed from Kafka 4 | Strimzi with KRaft |
| Magnum Heat driver | Deprecated | Cluster API (Magnum CAPI driver for self-service) |
| Kubernetes Dashboard | Retired | Headlamp |
| Redis | Licence change in 2024 | Valkey |

## 5. Tenant architecture

### 5.1 Identity and organisation

- One Keystone domain per organisation (`platform`, `tek-inventory`, `tek-triangle`, one per customer). Projects per product and environment, for example `tek-inventory-prod` (**ADR-0015**).
- People sign in through Keycloak with MFA. Keycloak groups map to the `reader`, `member` and `manager` roles. Automation uses application credentials (**ADR-0010**).
- A new tenant is one reviewed `tenant` module block in `infra-live/prod/`. It creates the project, group mapping, quotas, network, router, security group, DNS zone and application credential in one apply.

### 5.2 Compute offerings

Flavor sizes double within each family so instances pack hosts with
little waste. Root disks live in the Ceph `vms` pool and are never 0 GB.

| Family | RAM per vCPU | Sizes (vCPU / RAM GB / root GB) | Use |
| --- | --- | --- | --- |
| `t1` | 2 GB | small 1/2/20, medium 2/4/40, large 4/8/40 | Development, CI runners, light services |
| `m1` | 4 GB | small 1/4/20, medium 2/8/40, large 4/16/80, xlarge 8/32/80 | General purpose, Kubernetes nodes |
| `r1` | 8 GB | large 2/16/40, xlarge 4/32/80, 2xlarge 8/64/80 | Databases, caches |
| `c1` | 2 GB, pinned CPUs | large 4/8/40, xlarge 8/16/40 | Latency-sensitive work; only once a pinned host aggregate exists |

- Allocation ratios are set per host: CPU 4:1 on shared cores, RAM 1:1. Pinned `c1` CPUs are never overcommitted.
- Images: Ubuntu 24.04, Rocky 10 and Debian 13, stored as raw for Ceph copy-on-write cloning, with the guest agent enabled so snapshots are filesystem-consistent.

### 5.3 Storage offerings

| Volume type | Behaviour |
| --- | --- |
| `ssd` (default) | Ceph RBD, replication 3 |
| `ssd-encrypted` | LUKS encryption, keys in Barbican |
| `ssd-limited` | Front-end QoS caps (for example 3,000 IOPS and 200 MB/s) for noisy workloads |

S3 buckets through RGW with Keystone-issued EC2 credentials, and CephFS
shares through Manila from Phase 6.

### 5.4 Placement, DNS and quotas

- Region `nbo1`, availability zone `nbo1-a`. A single AZ provides no AZ-level redundancy yet; tenants use server groups with anti-affinity for HA (**ADR-0014**).
- DNS: each tenant gets `<tenant>.cloud.<domain>` in Designate. APIs are at `api.cloud.<domain>` and Kubernetes workloads under `*.apps.cloud.<domain>` (**ADR-0012**; public domain to be chosen).
- Every project has explicit quotas (instances, cores, RAM, volumes, GB, floating IPs, load balancers, S3 bytes). No project relies on defaults.
