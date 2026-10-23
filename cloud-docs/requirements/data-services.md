# Data services

### US-018: Get a highly available PostgreSQL database

| Field | Value |
| --- | --- |
| Story | As a tenant, I declare a PostgreSQL cluster in one manifest, so that I get a replicated, backed-up database without running it myself, like RDS. |
| Input | CloudNativePG `Cluster` with 3 instances, storage size and backup schedule |
| Output | Primary and two replicas, read-write and read-only endpoints, continuous WAL archiving to RGW |
| SLA | Ready within 10 min; automatic failover within 30 s when the primary's node is lost; 99.95% availability per month |
| Verified by | Lab test: delete the primary pod and node, measure write unavailability; CNPG metrics alert in production |
| Phase | 5 |

### US-019: Restore a database to a point in time

| Field | Value |
| --- | --- |
| Story | As a tenant, I restore my database to a moment before a bad change, so that I recover from mistakes such as a dropped table. |
| Input | Target timestamp within the 30-day retention window |
| Output | A new cluster with data exactly as at that timestamp |
| SLA | RPO 5 min, RTO 30 min |
| Verified by | Monthly restore drill (ADR-0011): insert marker rows, restore to a timestamp, verify markers before and after |
| Phase | 5 (feature), 7 (drill) |
