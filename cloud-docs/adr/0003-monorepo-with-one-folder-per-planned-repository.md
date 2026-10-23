# ADR-0003: Start as a monorepo with one folder per planned repository

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Bootstrap)

## Context

The work plan splits the platform's code into six repositories, each
with its own owner tool and lifecycle:

| Planned repository | Holds | Applied by |
| --- | --- | --- |
| `cloud-docs` | ADRs, design, requirements, runbooks | People |
| `infra-baremetal` | MAAS, netplan, switch configs, cephadm specs | Ansible |
| `infra-openstack` | Kolla-Ansible globals, inventory, overrides | Kolla-Ansible |
| `infra-live` | OpenTofu root modules per environment | CI pipeline |
| `tofu-modules` | Reusable OpenTofu modules | Referenced by `infra-live` |
| `platform-gitops` | Argo CD apps, operators, policies | Argo CD |

With one engineer, six repositories means six sets of CI, protection
rules and settings. It also means cross-cutting changes (for example a
new network that touches bare metal, Kolla and OpenTofu at once) split
across several pull requests that must be merged in order.

## Decision drivers

- Low overhead for a team of one
- Atomic, reviewable changes across layers
- A clean path to separate repositories when the team or access needs grow
- Clear ownership boundaries from the start

## Options considered

1. **Six repositories now**: matches the target layout, but multiplies overhead, and cross-cutting changes need coordinated pull requests.
2. **One repository with free-form layout**: least overhead, but the boundaries blur and a later split becomes painful.
3. **One repository, one top-level folder per planned repository**, each self-contained and split out later with its history.

## Decision

We will keep a single repository (`lewiih001/OpenStack`) in which each
top-level folder corresponds exactly to one planned repository and
follows these rules:

- A folder never imports, references or relies on files in another folder by relative path. The one planned exception: `infra-live` consumes `tofu-modules` through versioned module sources.
- Each folder has its own `README.md` stating what it holds and what applies it.
- CI jobs added later are scoped to their folder with path filters.
- Shared repository files (`.github/`, `.githooks/`, `.editorconfig`, `.gitattributes`, `.gitignore`, `.yamllint.yml`) stay at the root.

When a split is needed, a folder is extracted with its full history
using `git filter-repo --subdirectory-filter <folder>`. It is then
removed from the monorepo in a single pull request that points to the
new repository.

## Consequences

### Positive

- One pull request can change every layer of a feature, reviewed and tested together.
- One CI configuration, one protection ruleset, one place to search.
- The folder boundaries make the eventual split mechanical.

### Negative

- **Argo CD gets read access to the whole repository**, not only `platform-gitops`. Acceptable because no plaintext secrets are ever committed (SOPS only, enforced by `.gitignore`, gitleaks and GitGuardian). Revisit before production.
- **No per-folder access control on GitHub.** Fine for one engineer; a trigger for splitting.
- **Every pull request runs all CI checks** until path filters are added. Acceptable while CI takes about a minute.

## Revisit when

- A second engineer or team needs a different level of access to one area.
- Argo CD in production must be limited to the GitOps folder only (split `platform-gitops` first).
- One folder needs its own release cadence or version tags (for example `tofu-modules` with semantic versions).
- Full CI time exceeds roughly ten minutes despite path filters.

## References

- Work plan: repository layout
- ADR-0001: GitHub for source control and CI
- git-filter-repo documentation (`--subdirectory-filter`)
