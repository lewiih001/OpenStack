# ADR-0012: Naming conventions

- **Status:** Accepted, with the public DNS domain still to be chosen
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations), applies to all phases

## Context

Names end up in DNS, certificates, inventories, monitoring labels, NetBox,
OpenTofu resources and runbooks. Renaming later breaks certificates,
dashboards and automation. Names must be predictable enough that a
person can tell from a name what something is and where it is.

## Decision drivers

- A name tells you site, role and instance without a lookup
- Stable across growth (second rack, second site)
- Valid in DNS labels and Kubernetes resource names
- No secrets or customer names in infrastructure names

## Decision

General rules: lowercase letters, digits and hyphens only; at most 63
characters; no environment or customer data that may change.

| Object | Pattern | Examples |
| --- | --- | --- |
| Site | `<city code><n>` | `nbo1` |
| Physical host | `<site>-<role><nn>` | `nbo1-ctl01`, `nbo1-hci01`, `nbo1-mgmt01`, `nbo1-lab01` |
| Switch, firewall | `<site>-<role><nn>` | `nbo1-tor01`, `nbo1-fw01` |
| Region and AZ | ADR-0014 | `nbo1`, `nbo1-a` |
| Kubernetes cluster | `<site>-<purpose>` | `nbo1-mgmt`, `nbo1-svc`, `nbo1-stage`, `nbo1-prod` |
| OpenStack project | `<org>-<team or product>-<env>` | `tek-inventory-prod`, `platform-svc-prod` |
| OpenTofu resource | `snake_case` of the role | `openstack_networking_network_v2.tenant_private` |
| Git branch | `<type>/<short-description>` | `ci/add-tflint`, `docs/adr-0016-ca-hierarchy` |
| DNS | `cloud.<domain>` delegated to the platform | `api.cloud.<domain>`, `*.apps.cloud.<domain>`, `<tenant>.cloud.<domain>` |

Role codes: `ctl` control and network, `hci` hyperconverged compute and
storage, `cmp` dedicated compute, `sto` dedicated storage, `mgmt`
management host, `lab` lab server, `tor` top-of-rack switch, `fw`
firewall.

The public domain (`<domain>`) is still to be chosen and registered. It
will be recorded by updating this ADR's status line.

## Consequences

### Positive

- Names are self-describing in alerts, logs and inventories.
- The site prefix makes a second site a non-breaking addition.

### Negative

- Discipline is required. Mitigation: OpenTofu variable validation and a CI check on inventory host names, added with Phase 1 code.

## Revisit when

- A naming collision or limit is hit (for example, more than 99 hosts of one role at one site).

## References

- Work plan: "Naming and addressing"
- ADR-0014: Region and availability zones
