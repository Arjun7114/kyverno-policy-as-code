# Mutation Policy — Debugging Notes & Lessons

These notes capture the real debugging journey behind the `add-runasnonroot-default`
mutation policy. This is strong interview material — it shows hands-on troubleshooting,
not just a happy-path tutorial.

## What the policy does

Automatically injects `securityContext.runAsNonRoot: true` into any Pod that doesn't already
specify it — enforcing a "secure by default" baseline. Instead of *blocking* pods that forget
the setting (validation), it *fixes* them at admission time (mutation). This shifts the burden
off developers: the platform guarantees the hardening.

## The debugging journey (3 real lessons)

### Lesson 1 — patchStrategicMerge with `+()` anchors silently failed
The first attempt used:
```yaml
mutate:
  patchStrategicMerge:
    spec:
      +(securityContext):
        +(runAsNonRoot): true
```
It applied cleanly (`READY: True`) but **did nothing** — the created pod still had
`securityContext: {}`. Cause: `patchStrategicMerge` *merges into* existing structure. When the
incoming pod has no `securityContext` at all, the nested "add-if-absent" anchors can no-op
instead of creating the whole branch.

**How it was caught:** by inspecting the *actual created resource*
(`kubectl get pod ... -o yaml`) rather than trusting the "pod created" message. The container
was running as `uid: 0` (root) — proof the mutation hadn't applied.

### Lesson 2 — JSON patch is more reliable for adding a missing path
The fix used an explicit JSON patch with a precondition:
```yaml
preconditions:
  all:
    - key: "{{ request.object.spec.securityContext.runAsNonRoot || 'notset' }}"
      operator: Equals
      value: "notset"
mutate:
  patchesJson6902: |-
    - op: add
      path: "/spec/securityContext"
      value:
        runAsNonRoot: true
```
- `patchesJson6902` issues an explicit "add this value at this path" — no merge ambiguity.
- The `preconditions` block replaces the "only if not already set" safety the `+()` anchor was
  meant to provide, done reliably: it only mutates when `runAsNonRoot` isn't already specified,
  so it never clobbers a developer's deliberate setting.

Result: the created pod now showed `securityContext.runAsNonRoot: true`. Mutation confirmed.

### Lesson 3 — the policy exposed a real image/security conflict
After the mutation worked, the test pod (standard `nginx`) failed with
`CreateContainerConfigError` and the message *"container has runAsNonRoot and image will run
as root."* This wasn't a bug — it was the policy **working correctly**: the standard nginx
image is built to run as root, which now violates the injected non-root requirement. Kubernetes
refused to start it.

**The insight:** this is defense-in-depth surfacing a genuine problem — an image that can't run
unprivileged. The fix for a clean demo was to use `nginxinc/nginx-unprivileged`, an image
designed for non-root execution. The pod then both mutated *and* ran (`Running 1/1`).

## Interview soundbite

> My first mutation attempt used patchStrategicMerge and silently failed — the created pod still
> had an empty securityContext. I caught it by inspecting the live resource instead of trusting
> the success message, saw the container was still running as root, and switched to an explicit
> JSON patch with a precondition. That worked — and then exposed that the standard nginx image
> can't run non-root, which is exactly the kind of issue you want a security policy to surface.

## Commands worth remembering

- Inspect what was *actually* created (not what you submitted):
  `kubectl get pod <name> -o yaml`
- Filter long YAML output on Windows:
  `kubectl get pod <name> -o yaml | findstr /C:"runAsNonRoot" /C:"securityContext"`
- Check the effective user a container runs as: look for `uid:` under `status...user.linux`.
