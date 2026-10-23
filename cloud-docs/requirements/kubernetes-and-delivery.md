# Kubernetes and delivery

### US-015: Get a Kubernetes cluster

| Field | Value |
| --- | --- |
| Story | As a tenant, I request a Kubernetes cluster through a pull request, so that I run containers without operating the control plane myself. |
| Input | OpenTofu `k8s-cluster` module block (3 control plane, 3 workers, Kubernetes 1.36) |
| Output | Cluster `Ready` through Cluster API, with a Keycloak-backed kubeconfig, Ceph CSI and a Gateway API controller installed |
| SLA | All nodes `Ready` within 20 min of merge; control plane survives the loss of one node |
| Verified by | Lab pipeline: create, run conformance smoke tests (Sonobuoy quick mode), delete |
| Phase | 3 |

### US-016: Ship from commit to production

| Field | Value |
| --- | --- |
| Story | As a developer, I merge to `main`, so that my change is built, scanned, signed and deployed to stage with no manual step. |
| Input | A merged pull request in an application repository |
| Output | Image in Harbor with no critical vulnerabilities and a Cosign signature, deployed to `stage` by Argo CD; unsigned images are rejected by Kyverno |
| SLA | Running in stage within 15 min of merge; 100% of unsigned images rejected |
| Verified by | A reference application pipeline run on every platform change; Kyverno policy test with an unsigned image |
| Phase | 4 |

### US-017: Run a container without a cluster

| Field | Value |
| --- | --- |
| Story | As a developer, I deploy a container image as a Knative service, so that it scales with traffic, including to zero. |
| Input | One Knative `Service` manifest in my team's namespace |
| Output | HTTPS URL serving the container; scales to zero when idle |
| SLA | First response from a scaled-to-zero service within 5 s (p95); ready URL within 2 min of deploy |
| Verified by | Canary: request a scaled-to-zero test service every 15 min, cold-start latency exported |
| Phase | 6 |
