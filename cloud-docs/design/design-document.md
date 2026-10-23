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
