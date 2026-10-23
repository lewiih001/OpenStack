# Deployment plan

- **Status:** Draft v0.1. Values become final when entered in NetBox (Phase 0.7), which then becomes the source of truth.
- **Owner:** Lewis (lewiih001)
- **Last reviewed:** 2026-10-23 (Phase 0)

The design document says **why**; this plan says **how and with what**:
the hardware list, every address, the deployment-specific settings, and the
prerequisites that must be true before Phase 1 starts. Once NetBox is
running, the address tables here are exported from it rather than edited
by hand.

## 1. Hardware list

| Host | Role | Model and spec | BMC | Rack position |
| --- | --- | --- | --- | --- |
| `nbo1-ctl01` to `nbo1-ctl03` | Control and network | 2 × 16 cores, 256 GB, 2 × 960 GB SSD RAID 1, 2 × 25 GbE | Redfish | To be assigned in NetBox |
| `nbo1-hci01` to `nbo1-hci03` | Hyperconverged | 2 × 32 cores, 512 or 768 GB, 2 × 480 GB SSD + 6 × 3.84 TB NVMe, 2 × 25 GbE | Redfish | To be assigned in NetBox |
| `nbo1-mgmt01` | Management host | 16 cores, 128 GB, 2 × 3.84 TB SSD RAID 1, 2 × 10 GbE | Redfish | To be assigned in NetBox |
| `nbo1-tor01`, `nbo1-tor02` | Top-of-rack switches (MLAG) | 25/100 GbE, MTU 9216 | Console | To be assigned in NetBox |
| `nbo1-fw01`, `nbo1-fw02` | Firewalls (HA pair) | Two upstream links | Console | To be assigned in NetBox |
| `lab1-lab01` to `lab1-lab03` | Lab | 16 cores, 128 GB, 1 × 480 GB + 2 × 1.92 TB SSD, 2 × 10 GbE | Redfish | Separate rack or bench |

Requirements for every server:
- TPM 2.0
- redundant PSUs
- a working Redfish BMC
- the same BIOS, BMC, NIC and NVMe firmware versions across all servers of one role

**Interim:** the GPU workstation (24 GB, 1 TB) acts as management host and
all-in-one lab until this hardware exists (ADR-0002).

## 2. Addressing plan

### 2.1 Subnets

| VLAN | Name | Production subnet | Lab subnet | Gateway | MTU |
| --- | --- | --- | --- | --- | --- |
| 10 | oob | 10.10.0.0/24 | 172.16.10.0/24 | `.1` on the OOB router | 1500 |
| 20 | provision | 10.20.0.0/24 | 172.16.20.0/24 | none (MAAS on the management host) | 1500 |
| 30 | mgmt | 10.30.0.0/24 | 172.16.30.0/24 | `.1` on the firewall, egress to mirrors and updates only | 9000 |
| 40 | tunnel | 10.40.0.0/24 | 172.16.40.0/24 | none (not routed) | 9000 |
| 50 | storage | 10.50.0.0/24 | 172.16.50.0/24 | none (not routed) | 9000 |
| 60 | replication | 10.60.0.0/24 | 172.16.60.0/24 | none (not routed) | 9000 |
| 70 | external | public block from the ISP | lab public or NAT range | ISP or firewall | 1500 |
| 80 | lb-mgmt | 10.80.0.0/22 | 172.16.80.0/22 | none (Octavia only) | 1500 |

The lab reuses VLAN IDs on its own switches with `172.16.<vlan>.0/24`
subnets, so a lab address can never be mistaken for production.

### 2.2 Host numbering inside each /24

The same last octet identifies a host on every VLAN, so `.21` is always
`ctl01`, whatever the network.

| Range | Use |
| --- | --- |
| `.1` | Gateway, where the VLAN is routed |
| `.2` to `.9` | Network devices (switches, firewalls, OOB router) |
| `.10` to `.19` | Management hosts (`mgmt01` = `.11`) |
| `.20` to `.49` | Control nodes (`ctl01` = `.21`, `ctl02` = `.22`, `ctl03` = `.23`) |
| `.50` to `.99` | Hyperconverged and compute nodes (`hci01` = `.51` ...) |
| `.100` to `.109` | Virtual IPs (internal API VIP = `10.30.0.100`) |
| `.110` to `.199` | Reserved for growth (dedicated storage nodes from `.150`) |
| `.200` to `.254` | DHCP pools where needed (MAAS commissioning on VLAN 20 only) |

### 2.3 Which host gets an address on which VLAN

| Host | 10 oob (BMC) | 20 provision | 30 mgmt | 40 tunnel | 50 storage | 60 replication | 70 external | 80 lb-mgmt |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `mgmt01` | yes | yes (MAAS) | yes | no | yes (Ceph admin) | no | no | no |
| `ctl01` to `03` | yes | yes | yes | yes (OVN gateway) | yes (MON, MGR, clients) | no | interface, no IP | yes (Octavia health manager) |
| `hci01` to `03` | yes | yes | yes | yes | yes | yes (OSD replication) | interface, no IP | no |

VLAN 70 interfaces carry no host IP. Neutron and OVN own them for router
gateways and distributed floating IPs. The external API VIP and the
public addresses are allocated from the ISP block when it arrives.

## 3. Deployment-specific settings

The values that differ per environment. Everything else follows Kolla,
Ceph and OpenTofu defaults or the ADRs.

| Setting | Production | Lab | Where it lives |
| --- | --- | --- | --- |
| `openstack_region_name` | `nbo1` | `lab1` | `infra-openstack/globals.yml` (ADR-0014) |
| Default AZ (Nova, Cinder) | `nbo1-a` | `lab1-a` | `infra-openstack/` |
| `kolla_base_distro` | `ubuntu` | `ubuntu` | `globals.yml` (ADR-0005) |
| `openstack_release` | `2026.1` | `2026.1` | `globals.yml` (ADR-0004) |
| `network_interface` | `bond0.30` | lab equivalent | `globals.yml` |
| `tunnel_interface` | `bond0.40` | lab equivalent | `globals.yml` |
| `storage_interface` | `bond0.50` | lab equivalent | `globals.yml` |
| `neutron_external_interface` | `bond0.70` (no IP) | lab equivalent | `globals.yml` |
| `kolla_internal_vip_address` | `10.30.0.100` | `172.16.30.100` | `globals.yml` |
| `keepalived_virtual_router_id` | `51` | `52` (must differ) | `globals.yml` |
| `kolla_external_fqdn` | `api.cloud.<domain>` | `api.lab.cloud.<domain>` | `globals.yml` (ADR-0012) |
| `docker_registry` | `10.30.0.11:5000` (mirror on `mgmt01`) | lab management host | `globals.yml` |
| TLS internal and external | enabled | enabled | `globals.yml` |
| Nova `cpu_allocation_ratio` / `ram_allocation_ratio` | `4.0` / `1.0` | same | `infra-openstack/config/nova.conf` |
| Nova `reserved_host_memory_mb` on HCI | `65536` | sized to lab hosts | per-host `nova.conf` |
| Ceph `osd_memory_target` on HCI | 8 GB | sized to lab hosts | `infra-baremetal/` cephadm config |
| Ceph replication size | 3, failure domain host | 3 (three lab nodes) | cephadm pool settings (ADR-0007) |
| Octavia topology | `ACTIVE_STANDBY` | `ACTIVE_STANDBY` | `infra-openstack/config/octavia.conf` |

Secrets (`passwords.yml`, keyrings, application credentials) are never
listed here. They are generated once and stored SOPS-encrypted (Phase 0.4).

## 4. Prerequisites before Phase 1

| # | Prerequisite | Owner | Status |
| --- | --- | --- | --- |
| 1 | Hardware ordered and delivered (HCI RAM decision: 512 or 768 GB) | Lewis | Open |
| 2 | Rack space (about 14U), A/B power on separate UPS (about 6 kW), cooling | Lewis / facility | Open |
| 3 | Two upstream internet links, public block allocated (/24 requested, /26 minimum) | Lewis / ISPs | Open |
| 4 | Public domain registered; `cloud.<domain>` delegation planned | Lewis | Open |
| 5 | Firmware updated to identical versions per role (BIOS, BMC, NIC, NVMe) | Lewis | Open |
| 6 | BIOS: VT-x/AMD-V and IOMMU on, performance profile, UEFI + Secure Boot | Lewis | Open |
| 7 | Switches: MLAG, one LACP port-channel per server, VLANs trunked, MTU 9216 | Lewis | Open |
| 8 | Out-of-band path (separate router or LTE modem) to VLAN 10 | Lewis | Open |
| 9 | NTP: two internal chrony servers reachable from VLAN 30 | Lewis | Open |
| 10 | NetBox populated with racks, cables, VLANs and every address in section 2 | Lewis | Open (Phase 0.7) |
| 11 | SOPS + age keys created and backed up in two places | Lewis | Open (Phase 0.4) |
| 12 | Offline root CA and intermediate CA created | Lewis | Open (Phase 0.5) |
