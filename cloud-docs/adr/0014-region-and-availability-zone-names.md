# ADR-0014: Region and availability zone names

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations), configured in Phase 2

## Context

OpenStack region and availability zone names appear in the Keystone
service catalog, every tenant's `clouds.yaml`, OpenTofu code, Kubernetes
topology labels and documentation. Renaming them after tenants exist
breaks tenant automation. OpenStack's defaults (`RegionOne`, `nova`)
say nothing about location and do not scale to a second site.

## Decision drivers

- Names stable for the life of the platform
- Growth to a second rack and a second site without renaming
- Consistent with host naming (ADR-0012)

## Decision

- **Region:** `nbo1` (`openstack_region_name: "nbo1"` in Kolla). A future second site becomes a separate region, for example `nbo2` or `<city code>1`, sharing Keystone.
- **Availability zones:** `nbo1-a` for Nova and Cinder from day one, set as `default_availability_zone` in the Nova configuration override and `storage_availability_zone` in the Cinder override under `infra-openstack/config/`. A second rack with independent power and switching becomes `nbo1-b`.
- Nova host aggregates carry the AZ. Kubernetes nodes get the standard `topology.kubernetes.io/region` and `topology.kubernetes.io/zone` labels with the same values.
- The lab uses region `lab1` and AZ `lab1-a`, so lab credentials can never target production by mistake.

## Consequences

### Positive

- Tenants write the final region and AZ names into their automation from day one.
- Adding `nbo1-b` or a second region is additive, not a rename.

### Negative

- A single AZ at start means `nbo1-a` offers no AZ-level redundancy yet. Tenants must not assume it does. This is stated in tenant documentation.

## Revisit when

- A second rack with independent power and network exists (add `nbo1-b`).
- A second site is built (add a region).

## References

- Work plan: Phase 2.5 (availability zones)
- ADR-0006: Network design
- ADR-0012: Naming conventions
