\# Kyverno: Policy-as-Code for Kubernetes Security



Enforcing Kubernetes security best practices at admission time using

Kyverno policies — blocking insecure workloads before they ever run.



\## Status

🚧 In progress



\## What this does

Policy-as-code that validates, mutates, and audits Kubernetes resources:

blocking privileged containers, requiring resource limits, disallowing the

`:latest` image tag, and more — enforced automatically by Kyverno's

admission controller.



\## Tech Stack

Kubernetes, Kyverno, Helm, kind, YAML



\## Project Structure

\- `policies/` — Kyverno policy definitions

\- `tests/` — sample compliant / non-compliant manifests

\- `docs/` — notes and screenshots

