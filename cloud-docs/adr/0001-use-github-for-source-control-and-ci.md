# ADR-0001: Use GitHub for source control, reviews and CI

- **Status:** Accepted
- **Date:** 2026-10-23
- **Deciders:** Lewis (lewiih001)
- **Phase:** 0 (Foundations)

## Context

The work plan placed a self-hosted GitLab in Phase 0 as the home for all
infrastructure code, merge requests and CI pipelines. Every later phase
depends on that system existing first: configuration is applied from
reviewed commits and never by hand.

Self-hosting GitLab before the cloud exists creates a bootstrap problem.
GitLab needs a highly available host, PostgreSQL, Redis, object storage,
backups, TLS and upgrades. The only available hardware today is one
workstation with 24 GB RAM that is not always reachable, and the
production servers are not yet procured. The repository already exists
on GitHub (`lewiih001/OpenStack`), with CI running on GitHub Actions.

## Decision drivers

- Available on day one, with no infrastructure to run before the cloud exists
- Strong supply-chain controls: pinned actions, least-privilege tokens, secret scanning
- Pull-request review and branch protection as the only path to `main`
- Low operational load for a team of one
- Ability to reach private networks (lab, management plane) from CI

## Options considered

1. **Self-hosted GitLab CE** on the workstation now, later on the cloud: full control and data on-premises, but a heavy service to run, back up and upgrade, and it would sit on the same single machine as the lab it deploys.
2. **GitLab.com (SaaS)**: no hosting, but adds a second platform and account while the repository and CI already work on GitHub.
3. **GitHub (SaaS) with GitHub Actions**, plus a self-hosted runner for jobs that need private network access.

## Decision

We will use GitHub for source control, pull requests and CI (GitHub
Actions). Jobs that only lint or validate run on GitHub-hosted runners.
Jobs that must reach private networks (Kolla-Ansible deploys, OpenTofu
plan and apply against the lab) will run on a self-hosted runner on the
management host (see ADR-0002). Those jobs will be added in later steps.

Rules that come with this decision:

- `main` changes only through pull requests that pass CI (`yamllint`, `gitleaks`), merged with rebase so each commit is preserved.
- Workflow tokens default to `contents: read`. Third-party actions are pinned to full commit SHAs.
- A self-hosted runner is registered **only while the repository is private**. On a public repository, a pull request from a fork can execute code on the runner and reach the private network.

## Consequences

### Positive

- CI and reviews work today, with nothing to host before the cloud exists.
- Hosted runners give clean, disposable environments for lint and validation jobs.
- The team keeps one platform and one identity for code, reviews and CI.

### Negative

- **Server-side protection depends on the plan.** GitHub Free does not enforce rulesets on private repositories. Mitigation: the versioned `.githooks/pre-push` guard, review discipline, and revisiting GitHub Pro. Accepted for now.
- **Dependency on an external SaaS.** A GitHub outage pauses changes, but running infrastructure is not affected. Mitigation: every clone holds the full history. A read-only mirror can be added later (see revisit triggers).
- **Code is hosted outside Kenya.** Acceptable while the repository holds no personal data and secrets are committed only SOPS-encrypted.
- **Self-hosted runner risk.** The runner holds network reach into the lab. Mitigation: private repository only, a dedicated unprivileged user, ephemeral job workspaces, and no production credentials until Phase 2 review.

## Revisit when

- A customer, regulator or contract requires source code to be hosted on-premises or in-country.
- The team grows beyond a few people, or needs per-folder access control (consider GitHub Team or self-hosted GitLab on the cloud itself).
- The repository becomes permanently private and server-enforced protection is required (upgrade to GitHub Pro).
- The platform reaches Phase 4 and can host its own forge with HA and backups.

## References

- Work plan, Phase 0: source control and CI
- ADR-0002: Workstation as interim management host and lab
- GitHub Docs: "Security hardening for GitHub Actions" (self-hosted runner guidance)
