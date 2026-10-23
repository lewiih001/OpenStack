# Requirements as user stories

Every capability the platform offers is specified as a user story
**before** it is built. Each story is then turned into an automated test
that first gates the phase and then keeps running in production as a
monitoring check (the synthetic canary or a Prometheus alert).

## Story format

Each story has a stable ID and these fields:

| Field | Meaning |
| --- | --- |
| **Story** | "As a <user>, I <action>, so that <outcome>." |
| **Input** | What the user provides |
| **Output** | What the platform returns or guarantees |
| **SLA** | A measurable threshold (time, success rate, data loss) |
| **Verified by** | The automated test, and where it runs after go-live |
| **Phase** | The work plan phase whose exit criteria include it |

Rules:
- IDs (`US-001`, `US-002`...) are never reused. A retired story keeps its ID with status **Retired**.
- An SLA is a number, never "fast" or "quickly".
- A story is **Done** only when its test passes in the lab and runs against production.

## Users

| User | Who |
| --- | --- |
| Tenant | A team or customer consuming the cloud (including Tek Inventory) |
| Developer | An engineer shipping a product through the delivery pipeline |
| Operator | A platform engineer running the cloud |
| Automation | CI jobs, OpenTofu, operators acting with application credentials |

## Index

| Area | File | Stories |
| --- | --- | --- |
| Identity and access | [identity-and-access.md](identity-and-access.md) | US-001 to US-003 |
| Compute | [compute.md](compute.md) | US-004 to US-006 |
| Network | [network.md](network.md) | US-007 to US-010 |
| Storage | [storage.md](storage.md) | US-011 to US-014 |
| Kubernetes and delivery | [kubernetes-and-delivery.md](kubernetes-and-delivery.md) | US-015 to US-017 |
| Data services | [data-services.md](data-services.md) | US-018 to US-019 |
