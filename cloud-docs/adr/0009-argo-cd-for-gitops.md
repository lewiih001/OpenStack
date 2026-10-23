# ADR-0009: Argo CD for GitOps delivery to Kubernetes

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations), used from Phase 3

## Context

Several Kubernetes clusters (`mgmt`, `svc`, `stage`, `prod`) will run
platform services (operators, policies, observability, identity) and
products. Their desired state must come from Git, drift must be visible
and corrected, and no one should need `kubectl apply` access to
production.

## Decision drivers

- Pull-based reconciliation, so CI never holds cluster credentials
- One view of every cluster and application, with drift and health
- Multi-cluster management from one control point
- SSO with Keycloak and RBAC per team

## Options considered

1. **CI pushes with `kubectl` or Helm**: simple, but CI holds production credentials, and drift goes unnoticed.
2. **Flux**: pull-based and lightweight, with no built-in UI. Multi-cluster needs more assembly.
3. **Argo CD**: pull-based, with a UI, ApplicationSets for multi-cluster and SSO with RBAC.

## Decision

We will use **Argo CD** (3.5.x per ADR-0004), running on the `mgmt`
cluster and managing every cluster from `platform-gitops/`:

- App-of-apps and ApplicationSets generate one application per component per cluster.
- Automated sync with self-heal and prune for platform components. Production product releases use automated sync after promotion from `stage`.
- Keycloak OIDC login. Argo CD projects and RBAC map to Keycloak groups.
- Secrets are never stored in Git in plaintext: SOPS-encrypted for bootstrap, External Secrets from OpenBao at runtime.
- Argo CD gets read-only access to the repository.

## Consequences

### Positive

- Git is the only way to change a cluster; every change is reviewed and auditable.
- Drift is detected and reverted automatically.
- CI never holds production cluster credentials.

### Negative

- Argo CD itself is critical infrastructure. Mitigation: it runs on the `mgmt` cluster, is backed up with Velero, and can be re-bootstrapped from Git.
- In the monorepo, Argo CD can read the whole repository (ADR-0003). Accepted, because no plaintext secrets are committed.

## Revisit when

- Argo CD's resource footprint becomes significant at scale (consider sharding the controller).
- The project changes licence or governance.

## References

- Work plan: Phase 3 and Phase 4
- ADR-0003: Monorepo
- ADR-0010: Identity
