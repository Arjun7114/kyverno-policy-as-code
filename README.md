# Kyverno: Policy-as-Code for Kubernetes Security

!\[Kyverno Policy CI](https://github.com/Arjun7114/kyverno-policy-as-code/actions/workflows/kyverno-ci.yaml/badge.svg)

Enforcing Kubernetes security and governance at **admission time** using Kyverno — blocking
insecure workloads before they ever run, auto-remediating unsafe defaults, and pairing
deploy-time policy with build-time image scanning for defense-in-depth.

Built and tested on a local Kubernetes cluster (kind), with every policy version-controlled as
YAML — the way policies belong in a GitOps workflow.

\---

## Why this project

Kubernetes will run whatever you give it — including privileged containers, images with no
resource limits, and workloads full of known CVEs. Catching these *after* deployment is too
late. This project implements **policy-as-code**: security rules expressed as version-controlled
YAML, enforced automatically by Kyverno's admission controller, so non-compliant workloads are
rejected (or fixed) at the moment of creation.

It covers both halves of a real security posture:

* **Deploy-time (Kyverno):** is this workload *configured* securely?
* **Build-time (Trivy):** is this image *free of known vulnerabilities*?

Neither tool alone is sufficient — a key insight this project demonstrates with real data below.

\---

## What's implemented

Five policies spanning validation, mutation, and both enforcement modes:

|Policy|Type|Mode|What it does|
|-|-|-|-|
|`disallow-privileged-containers`|Validate|Enforce|Blocks privileged containers (host escape risk)|
|`require-resource-limits`|Validate|Enforce|Requires CPU + memory limits (prevents node starvation)|
|`disallow-latest-tag`|Validate|Enforce|Requires explicit image tags (no mutable `:latest`)|
|`add-runasnonroot-default`|Mutate|—|Auto-injects `runAsNonRoot: true` if unset|
|`require-team-label`|Validate|Audit|Records missing `team` labels without blocking|

Plus **Trivy image scanning** for build-time vulnerability detection.

\---

## Key demonstrations

### 1\. Validation — blocking insecure workloads

A pod requesting `privileged: true` with no resource limits is rejected at admission, failing
multiple policies at once:

```
resource Pod/default/bad-pod was blocked due to the following policies:
  disallow-privileged-containers: Privileged containers are not allowed...
  require-resource-limits: CPU and memory limits are required for every container...
```

A compliant pod is admitted normally — the policies block only what's genuinely unsafe, not
legitimate work.

### 2\. Mutation — auto-remediating secure defaults

Rather than *rejecting* pods that forget `runAsNonRoot`, the mutation policy *adds* it
automatically. A pod submitted with no `securityContext` comes out of admission with
`runAsNonRoot: true` injected — a "secure by default" baseline the developer never had to write.

> \\\*\\\*Debugging note:\\\*\\\* the first attempt used `patchStrategicMerge` with `+()` anchors and
> silently failed — the created pod still had an empty `securityContext`. Caught by inspecting
> the \\\*live\\\* resource (`kubectl get pod -o yaml`) rather than trusting the "created" message; the
> container was still running as root (`uid: 0`). Switching to an explicit `patchesJson6902`
> JSON patch with a precondition fixed it. This then exposed that the standard `nginx` image
> can't run non-root — a real conflict the policy correctly surfaced. See
> \\\[`docs/mutation-notes.md`](docs/mutation-notes.md).

### 3\. Audit vs Enforce — the safe rollout pattern

Hard-blocking (`Enforce`) policies can't be dropped onto a running cluster without breaking
existing workloads. The professional pattern is **Audit first**: the `require-team-label` policy
runs in Audit mode, so a pod missing the label is *admitted* but the violation is *recorded* —
letting you see what would break before graduating to Enforce.

> \\\*\\\*Ops war story:\\\*\\\* the Kyverno reports-controller repeatedly restarted on the local cluster.
> Traced through restart count -> clean exit code -> logs, which revealed leader-election
> lease-renewal timeouts (`context deadline exceeded`) caused by slow API/etcd response on a
> laptop kind cluster. Enforcement (synchronous, at admission) was unaffected; only asynchronous
> report aggregation lagged. On a properly-resourced cluster this doesn't occur. See
> \\\[`docs/audit-mode-notes.md`](docs/audit-mode-notes.md).

### 4\. Build-time scanning — Trivy image vulnerability comparison

Kyverno checks *configuration*, not image *contents*. Trivy fills that gap. Scanning three base
images for HIGH/CRITICAL vulnerabilities produced a striking result:

|Image|HIGH|CRITICAL|Takeaway|
|-|:-:|:-:|-|
|`python:3.9-slim`|44|7|A common, convenient base is full of known CVEs|
|`nginxinc/nginx-unprivileged:1.27`|106|12|**Runtime-hardened != vulnerability-free**|
|`gcr.io/distroless/static-debian12`|0|0|Minimal image (4 packages) = tiny attack surface|

The middle row is the important one: `nginx-unprivileged` is *runtime-hardened* (runs as
non-root, which the Kyverno mutation policy enforces) yet ships **more** vulnerable packages than
the "vulnerable" Python image. "Secure" is multi-dimensional — an image can pass runtime policy
and still be full of CVEs. That's precisely why deploy-time policy (Kyverno) and build-time
scanning (Trivy) are both required. Scan outputs are in [`docs/`](docs/).

\---

## Concepts worth being able to explain

* **Admission control** — how Kyverno intercepts and validates/mutates resources at creation.
* **Validation vs mutation** — blocking bad config vs auto-fixing it ("secure by default").
* **Enforce vs Audit** — and why you roll out in Audit first, review reports, then enforce.
* **Synchronous enforcement vs asynchronous reporting** — why a violation is blocked instantly
but appears in a report only after a background reconcile.
* **`patchStrategicMerge` vs `patchesJson6902`** — and when the latter is more reliable.
* **Defense-in-depth** — deploy-time policy (Kyverno) + build-time scanning (Trivy) as
independent, complementary layers.
* **Runtime hardening vs vulnerability hygiene** — two distinct axes of image security.

\---

## Tech Stack

Kubernetes - Kyverno - Helm - kind - Trivy - YAML

## Project Structure

```
policies/                          # Kyverno policy definitions (the core artifacts)
  disallow-privileged-containers.yaml
  require-resource-limits.yaml
  disallow-latest-tag.yaml
  add-runasnonroot-default.yaml    # mutation policy
  require-team-label-audit.yaml    # audit-mode policy
tests/                             # compliant + violating manifests
  good-pod.yaml
  bad-pod.yaml
  compliant-pod.yaml
  mutation-test-pod.yaml
  audit-test-pod.yaml
docs/                              # notes + Trivy scan evidence
  mutation-notes.md
  audit-mode-notes.md
  scan-\\\*.txt
.gitignore
README.md
```

## Running it

```bash
# 1. Create a local cluster (kind) and install Kyverno
kind create cluster --name kyverno-lab
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

# 2. Apply the policies
kubectl apply -f policies/

# 3. Test enforcement (this pod is rejected)
kubectl apply -f tests/bad-pod.yaml

# 4. Test mutation (this pod comes out with runAsNonRoot injected)
kubectl apply -f tests/mutation-test-pod.yaml
kubectl get pod mutation-test-pod -o yaml | grep -A1 securityContext

# 5. Scan images for vulnerabilities
trivy image --severity HIGH,CRITICAL python:3.9-slim
```

## Notes

* Policies are cluster-agnostic YAML — they deploy identically to a managed cluster (e.g. EKS)
via the same GitOps flow (ArgoCD/Flux), where policy-as-code belongs.
* Deployed single-replica on a local cluster for learning; production would run Kyverno in HA
(multi-replica) mode.

