# PodDisruptionBudget — Production Patterns

[![CI](https://github.com/alice101-dev/k8s-pdb-production-patterns/actions/workflows/ci.yml/badge.svg)](https://github.com/alice101-dev/k8s-pdb-production-patterns/actions/workflows/ci.yml)

How to configure a **PodDisruptionBudget (PDB)** that actually protects a
service — and never deadlocks cluster maintenance. Ships a complete, hardened
example stack: PDB + Deployment + HPA + Service + NetworkPolicy in a
`restricted` Pod-Security namespace.

## Why a PDB

A node drain (upgrade, autoscaler consolidation, spot reclaim) evicts pods.
Without a PDB, two drains in a row can evict **every** replica of a service at
once. A PDB caps how many pods of one app may be down from *voluntary*
disruptions:

```
drain node-1 ──▶ evicts hello-web-1   ✅ allowed  (1 of 2 down)
drain node-2 ──▶ evicts hello-web-2   ⛔ BLOCKED until a replacement is Ready
```

## The rules this repo encodes

| Rule | Why |
| --- | --- |
| Use **`maxUnavailable: 1`**, not `minAvailable` | Guarantees forward progress: drains can always evict one pod at a time. `minAvailable` percentages are computed against the *current* (HPA-scaled) count and **round up** — easily becoming "0 disruptions allowed". |
| **Never PDB a single replica** | `minAvailable: 30%` with 1 replica rounds up to 1 → the node becomes undrainable forever. HPA here enforces `minReplicas: 2`. |
| Set **`unhealthyPodEvictionPolicy: AlwaysAllow`** | (K8s ≥ 1.31) Without it, a crash-looping app holds its PDB hostage and blocks node maintenance. |
| **Omit `replicas`** on the Deployment | The HPA owns the count; declaring it resets scale on every `kubectl apply`. |
| **Spread replicas** with `topologySpreadConstraints` | A PDB is pointless if both replicas share one node: zone + hostname spread (soft) keeps eviction *and* hardware failure domains apart. |
| **`preStop` sleep + `maxUnavailable: 0` rollout** | Zero-downtime deploys: keep serving while endpoint removal propagates. |

## Hardened by default

The example workload passes the **restricted** Pod Security Standard (enforced
by a namespace label) and 90+ Checkov policies: unprivileged nginx (UID 101,
non-root), read-only root filesystem with tmpfs mounts, all capabilities
dropped, `RuntimeDefault` seccomp + AppArmor, no service-account token, and a
default-deny NetworkPolicy (ingress 8080 from the namespace, egress DNS only).

## Deploy & verify

```bash
kubectl apply -f namespace.yaml
kubectl apply -f pdb.yaml -f deployment.yaml -f service.yaml -f hpa.yaml -f networkpolicy.yaml

# Watch the budget
kubectl get pdb -n staging hello-web
# NAME        MAX UNAVAILABLE   ALLOWED DISRUPTIONS
# hello-web   1                 1

# Prove it: drain a node hosting one replica — the second eviction waits
# until the replacement pod is Ready on another node.
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

## Testing & security scanning

Every push and pull request runs through [GitHub Actions](.github/workflows/ci.yml):

```bash
kubeconform -strict -summary ./*.yaml    # manifest schema validation
checkov -d . --framework kubernetes      # static security analysis
```

## References

- [Kubernetes: Disruptions & PDBs](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes: unhealthyPodEvictionPolicy](https://kubernetes.io/docs/tasks/run-application/configure-pdb/#unhealthy-pod-eviction-policy)
- [Kubernetes: Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes: Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
