# Compute

### US-004: Boot a virtual machine

| Field | Value |
| --- | --- |
| Story | As a tenant, I boot an Ubuntu 24.04 VM from an image with a key pair and a private network, so that I can run a workload like an EC2 instance. |
| Input | Image `ubuntu-24.04`, flavor `m1.small`, key pair, private network |
| Output | VM in `ACTIVE` state, reachable over SSH with my key from inside the network |
| SLA | `ACTIVE` within 60 s and SSH reachable within 120 s (p95); 99.5% boot success per month |
| Verified by | Canary: boot and delete a `t1.small` VM every 5 min; Rally `NovaServers.boot_and_delete_server` in the lab |
| Phase | 2 |

### US-005: Maintenance without downtime

| Field | Value |
| --- | --- |
| Story | As an operator, I live-migrate VMs off a hypervisor before maintenance, so that tenants are not interrupted. |
| Input | `openstack server migrate --live-migration` for each VM on the host |
| Output | VMs running on another hypervisor with the same IPs and volumes |
| SLA | At most 2 consecutive lost pings (1 per second) per VM; 100% of migrations succeed or roll back |
| Verified by | Lab test: continuous ping to a test VM during live migration, loss counted |
| Phase | 2 |

### US-006: Recover VMs from a failed hypervisor

| Field | Value |
| --- | --- |
| Story | As a tenant, my VMs restart on a healthy hypervisor when theirs fails, so that a hardware fault does not become an outage of days. |
| Input | A hypervisor loses power |
| Output | Host fenced (confirmed off via BMC); VMs evacuated and `ACTIVE` elsewhere with their Ceph-backed disks intact |
| SLA | VMs `ACTIVE` again within 15 min (runbook) or 5 min (Masakari, if enabled); zero data written by two copies of one VM |
| Verified by | Lab drill: power off a hypervisor via its BMC and time the recovery; run quarterly |
| Phase | 7 |
