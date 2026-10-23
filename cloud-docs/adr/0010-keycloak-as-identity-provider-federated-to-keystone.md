# ADR-0010: Keycloak as the identity provider, federated to Keystone

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations), built in Phase 4

## Context

People sign in to many surfaces: the OpenStack dashboard and CLI, Argo
CD, Grafana, Harbor, the developer portal and Kubernetes. Separate
accounts per tool multiply offboarding risk and make MFA inconsistent.
Automation (CI, OpenTofu, operators) needs credentials that are not a
person's password.

## Decision drivers

- One login, one MFA policy and one offboarding action for every tool
- Groups mapped to OpenStack projects and roles, and to Kubernetes RBAC
- Standard protocols (OIDC, SAML) and a path to connect a directory later
- Least privilege for automation

## Options considered

1. **Local accounts in each tool**: no single offboarding point, and MFA is inconsistent.
2. **Keystone as the only identity store**: OpenStack-only. Other tools would still need their own accounts.
3. **Keycloak as the central identity provider**, federated into Keystone over OIDC and used by every other tool.

## Decision

We will run **Keycloak** (26.x) as the single identity provider:

- Keystone federates to Keycloak over OIDC. Dashboard (Skyline/Horizon) and CLI users sign in through Keycloak.
- Keycloak groups map to Keystone domains, projects and roles (`reader`, `member`, `manager`) through federation mapping rules kept in Git.
- MFA (TOTP or WebAuthn) is required for every human account. Admin roles require WebAuthn.
- Argo CD, Grafana, Harbor and Kubernetes API access use Keycloak OIDC with group-based RBAC.
- Automation uses Keystone **application credentials** (scoped, revocable, never a password), stored in OpenBao.
- Break-glass local admin accounts exist per system. They are kept in the password manager with two named custodians, and every use is audited.

Keycloak runs on the `svc` cluster with three replicas and a
CloudNativePG database.

Until federation is live in Phase 4, platform operators use Keystone
local accounts with long random passwords, held in the password manager
and limited to named operators. No tenant accounts are created before
federation. When federation goes live, the local operator accounts are
disabled, except the break-glass accounts above.

## Consequences

### Positive

- One offboarding action removes access everywhere.
- Consistent MFA and audit trail.
- Tenants can later connect their own directory through Keycloak.

### Negative

- Keycloak becomes critical: if it is down, people cannot sign in. Mitigation: three replicas, database backups (ADR-0011), and break-glass accounts.
- A circular dependency risk: Keycloak runs on a cluster that the cloud hosts. Mitigation: platform operators keep break-glass access to the infrastructure layers that does not depend on Keycloak.

## Revisit when

- A customer or regulator requires a specific external identity provider.
- Keycloak's resource or operational cost becomes significant.

## References

- Work plan: Phase 4.4 Identity, Phase 7.2 Security baseline
- ADR-0015: Keystone domain layout
