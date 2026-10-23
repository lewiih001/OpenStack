# Identity and access

### US-001: Sign in and get a project-scoped token

| Field | Value |
| --- | --- |
| Story | As a tenant, I sign in with my Keycloak account and MFA, so that I can use the OpenStack API and dashboard for my project. |
| Input | Keycloak username, password and MFA code; target project |
| Output | A Keystone token scoped to that project, with the roles mapped from my Keycloak groups |
| SLA | Token issued within 2 s (p95) once MFA is completed; 99.9% success per month |
| Verified by | Canary: OIDC token exchange for the canary user every 5 min, duration exported to Prometheus |
| Phase | 4 (lab: Phase 2 with Keystone local users) |

### US-002: Offboard a person from every tool at once

| Field | Value |
| --- | --- |
| Story | As an operator, I disable a person's Keycloak account, so that they lose access to every platform tool from one place. |
| Input | Account disabled in Keycloak |
| Output | New sign-ins fail on OpenStack, Argo CD, Grafana and Harbor; existing sessions end at token expiry |
| SLA | New sign-ins rejected within 1 min; every existing token expired within 60 min (Keystone token lifetime 1 h) |
| Verified by | Lab test script: disable a test user, then attempt sign-in to each tool and reuse an old token after 60 min |
| Phase | 4 |

### US-003: Automation is limited to one project

| Field | Value |
| --- | --- |
| Story | As automation (a CI job), I use an application credential, so that I can manage resources in my project and nothing else. |
| Input | Application credential ID and secret scoped to one project |
| Output | API calls in that project succeed; calls against any other project return 403 or 404 |
| SLA | 100% of cross-project calls denied |
| Verified by | Lab test: canary credential lists servers in its own project (pass) and in another project (must fail) |
| Phase | 2 |
