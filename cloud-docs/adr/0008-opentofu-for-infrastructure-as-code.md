# ADR-0008: OpenTofu for infrastructure as code

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Bootstrap), used from Phase 2

## Context

Cloud resources (projects, networks, flavors, images, quotas, tenant
onboarding, Kubernetes management VMs) must be created from reviewed code,
never by hand. Tenants will also automate their own resources, so
the tool should be one they already know from AWS-style clouds.

## Decision drivers

- Open-source licence with no restriction on offering a service
- Mature OpenStack provider
- Plan and apply workflow that fits pull-request review
- Encrypted remote state

## Options considered

1. **Terraform**: mature, but under the Business Source License since 1.6, which restricts offering competing hosted services.
2. **OpenStack Heat**: native, but little used outside OpenStack, and no tenant familiarity.
3. **OpenTofu**: Linux Foundation fork of Terraform under MPL-2.0, same language and providers, with built-in state encryption.
4. **Pulumi**: general-purpose languages, but a smaller OpenStack community.

## Decision

We will use **OpenTofu** with `terraform-provider-openstack`:

- Reusable modules live in `tofu-modules/`: `tenant`, `network`, `vm`, `k8s-cluster`, `postgres`. Root modules per environment live in `infra-live/lab/` and `infra-live/prod/`.
- Provider and module versions are pinned. `.terraform.lock.hcl` is committed.
- State is stored remotely in S3 on Ceph RGW once Phase 1 provides it, encrypted with OpenTofu state encryption. Until then, state stays local on the management host and is never committed. `*.tfstate` is already ignored.
- CI runs `tofu fmt -check`, `tofu validate` and `tflint` on every pull request, and posts `tofu plan` for review. `tofu apply` runs only from CI after merge, on the self-hosted runner (ADR-0001).
- Heat stays available to tenants, but the platform does not use it.

## Consequences

### Positive

- Open licence suitable for a service offered to customers.
- Terraform-compatible language, so tenants can reuse existing skills and modules.
- State encryption protects the credentials and addresses held in state.

### Negative

- The OpenTofu and Terraform feature sets are diverging. Mitigation: use only features documented by OpenTofu, and test modules with OpenTofu in CI.
- Remote state depends on RGW. Bootstrap order: Phase 1 brings up RGW before the first `infra-live` apply.

## Revisit when

- OpenTofu project health declines (releases, maintainers), or the OpenStack provider stops supporting OpenTofu.

## References

- Work plan: Phase 2 and Phase 8.2
- ADR-0001: CI runner placement
- ADR-0003: Folder layout
