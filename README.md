# OpenStack private cloud platform

Infrastructure-as-code, GitOps configuration and documentation for an AWS-like private cloud:
OpenStack 2026.1 (Kolla-Ansible) on Ceph, Kubernetes via Cluster API, and platform services delivered by Argo CD.

## Layout

| Folder | Purpose | Applied by |
| --- | --- | --- |
| `cloud-docs/` | ADRs, design document, deployment plan, requirements (user stories), runbooks | People |
| `infra-baremetal/` | MAAS config, netplan templates, switch configs, cephadm specs | Ansible (management host) |
| `infra-openstack/` | Kolla-Ansible `globals.yml`, inventory, `config/` overrides, SOPS-encrypted secrets | Kolla-Ansible (CI runner) |
| `infra-live/` | OpenTofu root modules per environment (`lab/`, `prod/`) | CI pipeline |
| `tofu-modules/` | Reusable OpenTofu modules (tenant, network, vm, k8s-cluster, postgres) | Referenced by `infra-live/` |
| `platform-gitops/` | Argo CD app-of-apps, operators, policies, monitoring | Argo CD |

Each folder maps to one planned repository and can be split out later with its history ([ADR-0003](cloud-docs/adr/0003-monorepo-with-one-folder-per-planned-repository.md)).

## Conventions

- Changes reach `main` only through pull requests that pass CI (`yamllint`, `gitleaks`), merged with rebase so every commit is kept ([ADR-0001](cloud-docs/adr/0001-use-github-for-source-control-and-ci.md)). This is enforced by the `protect-main` ruleset on GitHub and by the local pre-push hook below.
- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`, `chore:`...).
- Secrets are committed only when SOPS-encrypted (`*.sops.yml`); plain-text secrets are blocked by `.gitignore` and CI.
- Every change is rehearsed in the lab before production.
- After cloning, run `git config core.hooksPath .githooks` to enable the local guard that blocks direct pushes to `main`.
