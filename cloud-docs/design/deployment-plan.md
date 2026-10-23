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
