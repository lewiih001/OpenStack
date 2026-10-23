# Network

### US-007: Publish a VM to the internet

| Field | Value |
| --- | --- |
| Story | As a tenant, I create a private network, a router and a floating IP, so that my VM serves traffic from the internet on the ports I allow. |
| Input | Network, subnet, router with external gateway, security group allowing TCP 443, floating IP |
| Output | The VM answers on the floating IP from outside the provider network; every other port is closed |
| SLA | Reachable within 30 s of floating IP association; closed ports drop 100% of connection attempts |
| Verified by | External Blackbox probe against a canary VM's floating IP; lab Rally `NeutronNetworks` scenarios |
| Phase | 2 |

### US-008: Tenants are isolated from each other

| Field | Value |
| --- | --- |
| Story | As a tenant, my private networks are unreachable from other tenants, so that my workloads are isolated by default. |
| Input | VMs in project A and project B, each on its own private network with overlapping CIDRs |
| Output | No traffic path between the two networks except through floating IPs and security groups |
| SLA | 0 packets delivered between tenant private networks |
| Verified by | Lab test: ping and TCP scan from a VM in project A to project B's private addresses, expected 100% loss |
| Phase | 2 |

### US-009: Highly available load balancer

| Field | Value |
| --- | --- |
| Story | As a tenant, I create a load balancer in front of my VMs, so that traffic is spread across them and survives the loss of one load balancer instance. |
| Input | Octavia load balancer with an HTTP listener, pool and health monitor over two VMs |
| Output | VIP serving traffic from healthy members, with an active-standby amphora pair |
| SLA | `ACTIVE` within 5 min; traffic interruption under 15 s when the active amphora is deleted |
| Verified by | Weekly canary create and delete; lab failover test with continuous HTTP requests |
| Phase | 2 |

### US-010: DNS records follow my resources

| Field | Value |
| --- | --- |
| Story | As a tenant, I get DNS names for my VMs and floating IPs in my zone, so that I do not manage records by hand. |
| Input | Zone `<tenant>.cloud.<domain>`, a port or floating IP with a DNS name |
| Output | A and PTR records created in Designate and resolvable from public resolvers |
| SLA | Record resolvable within 60 s of creation |
| Verified by | Canary: create a record and resolve it through an external resolver |
| Phase | 2 |
