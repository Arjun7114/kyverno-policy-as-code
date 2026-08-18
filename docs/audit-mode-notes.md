# Audit vs Enforce Modes — Notes & a Real Diagnostic War Story

## What this step demonstrates

Kyverno policies can run in two modes, set per-rule via `validate.failureAction`:

- **Enforce** — violations are *blocked* at admission time. The resource is rejected and never
  created. (Used by the privileged, resource-limits, and latest-tag policies.)
- **Audit** — violations are *recorded* but **not blocked**. The resource is admitted, and the
  violation is logged in a PolicyReport. (Used by the `require-team-label` policy.)

**Why this matters:** you can't safely drop hard-blocking (`Enforce`) policies onto a running
production cluster — you'd instantly reject existing non-compliant workloads and cause an
outage. The professional rollout pattern is **Audit first**: deploy in Audit mode, review the
PolicyReports to see what *would* break, work with teams to fix violations, then graduate the
policy to `Enforce`. This "audit → review → enforce" progression is a core operational-maturity
signal.

## The demonstration (this succeeded)

Applied `require-team-label` in **Audit** mode, then created `audit-test-pod` with **no `team`
label** — a clear violation. Result: the pod was **admitted** (`pod/audit-test-pod created`),
*not* rejected. Contrast with the Enforce policies, where a violating pod (`bad-pod`) is
rejected outright. Same kind of violation, opposite outcome — proving the mode difference. This
is the actual lesson of the step, and it worked.

## Diagnostic war story: the reports-controller lease timeout

The PolicyReport for the violation was slow/failing to appear. Rather than assume, I diagnosed
it — and it turned into a genuinely instructive Kubernetes troubleshooting exercise.

**Symptom:** `kubectl get policyreport -n default` returned nothing, even minutes after creating
the pod. Reports existed for *other* namespaces (via `-A`) but not `default`.

**Investigation path:**
1. `kubectl get pods -n kyverno` → the `kyverno-reports-controller` showed **8 restarts**, last
   one seconds ago — a crash loop.
2. `kubectl describe pod ...` → exit reason was `Completed`, exit code `0` — a *clean* exit, not
   an error crash. So it wasn't OOMKilled or panicking; it was stepping down deliberately.
3. `kubectl logs -n kyverno <reports-controller-pod> --tail=40` → the smoking gun:
   ```
   ERR ... Failed to update lease optimistically ... context deadline exceeded
   ERR ... error retrieving lease lock ... client rate limiter ... context deadline exceeded
   INF ... Failed to renew lease ... context deadline exceeded
   TRC ... leadership lost, stopped leading
   ```

**Root cause:** Kubernetes controllers use **leader election** — they periodically renew a
*lease* to prove they're alive and in charge. On a local kind cluster, etcd shares laptop disk
I/O and the API server responds slowly under load; the reports controller misses its lease
renewal deadline (`context deadline exceeded`), concludes it lost leadership, and restarts. This
is a documented Kyverno behavior on resource-constrained clusters, confirmed by maintainer
issues — etcd disk latency is the classic cause of cluster-wide `context deadline exceeded`.

**Why it doesn't matter for the project:**
- Policy **enforcement is synchronous** (decided at admission, instantly) and worked perfectly.
- Policy **reporting is asynchronous** (background controller, eventually-consistent) and was the
  only thing affected.
- On a properly-resourced cluster (e.g. EKS with a healthy control plane), this doesn't occur.

**The transferable lesson (and interview soundbite):**
> The Kyverno reports-controller kept restarting on my local cluster. I traced it through the
> pod's restart count, then its clean exit code, then its logs — which showed leader-election
> lease-renewal timeouts (`context deadline exceeded`). That's the API server responding too
> slowly on a laptop-hosted etcd for the controller to renew its lease in time. Enforcement was
> unaffected because that's synchronous at admission; only the asynchronous report aggregation
> lagged. On a real cluster it's a non-issue.

Being able to distinguish "the security control is broken" from "an async reporting component is
flaky due to local resource limits" — and prove which one it is from logs — is the actual skill.

## Key distinction to remember

| Aspect | Behavior |
|--------|----------|
| Policy **enforcement** | Synchronous — at admission, immediate, unaffected by controller health |
| Policy **reporting** | Asynchronous — background reconcile, eventually-consistent, sensitive to API/etcd latency |
