# ACM Placement Tolerations and Disaster Recovery Orchestration

## Overview

This document explains the relationship between Open Cluster Management (OCM/ACM)
`Placement` tolerations and Disaster Recovery (DR) orchestration via Ramen/DRPC.
It is intended to clarify a common architectural misconception: that adding tolerations
to a `Placement` resource will interfere with or block DR failover for protected workloads.

---

## Background

When a managed cluster goes offline, OCM automatically applies taints to that cluster:

- `cluster.open-cluster-management.io/unreachable`
- `cluster.open-cluster-management.io/unavailable`

By default, without tolerations defined in a `Placement`, ACM will immediately undeploy
any application assigned to the affected cluster. This can cause undesirable "flapping"
behaviour — particularly dangerous for stateful workloads backed by replicated storage.

The instinct to add tolerations is correct. The concern that doing so will block DR is not.

---

## How the Orchestration Layers Work

Understanding why tolerations and DR do not conflict requires understanding the two
distinct layers involved:

### Layer 1 — ACM Placement Controller (Generic Scheduler)

The standard `Placement` controller handles cluster selection based on labels, taints,
tolerations, and scheduling constraints. It is a general-purpose mechanism with no
awareness of storage state or replication status.

### Layer 2 — Ramen / DRPlacementControl (DR Orchestrator)

When an application is protected by a `DRPlacementControl` (DRPC) resource, the system
annotates the `Placement` to designate **Ramen** as the active reconciler:

```yaml
annotation: cluster.open-cluster-management.io/experimental-scheduling-disable: "true"
```

Once this annotation is present, **Ramen takes full control of placement decisions**.
The generic `Placement` controller is effectively bypassed. Changes to tolerations or
placement rules at this point do not influence where or when the workload moves —
Ramen decides that based on storage readiness and explicit administrator action.

---

## The Role of Tolerations in a DR Context

Tolerations serve a specific and important purpose in DR-protected environments:

- They **prevent premature deletion** of the application by the generic scheduler
  while the cluster is unreachable.
- They **buy time** for DRPC/Ramen to orchestrate a safe, storage-consistent failover.
- They do **not** suppress or interfere with Ramen once the DR annotation is applied.

A toleration defined without `tolerationSeconds` defaults to `nil`, meaning the workload
is pinned to the disconnected cluster indefinitely — until DRPC explicitly moves it.
This is the intended behaviour for stateful workloads under asynchronous replication.

> **Why indefinite pinning matters:** Regional DR uses asynchronous replication. If the
> generic scheduler reacted immediately to a downed cluster, the workload could spin up
> on the secondary site before storage replication has caught up — risking data corruption
> or a split-brain scenario. Tolerations prevent this race condition.

---

## Recommended Strategy by Workload Type

### Stateful Workloads (DR-Protected via DRPC)

1. Create a `DRPolicy` and `DRPlacementControl` on the hub cluster targeting the workload.
2. Define tolerations on the `Placement` **without** `tolerationSeconds` to pin the workload.
3. The system annotates the `Placement`, handing control to Ramen.
4. Failover or relocation is triggered manually by an administrator updating the
   DRPC `action` field to `Failover` or `Relocate`.

```yaml
tolerations:
  - key: "cluster.open-cluster-management.io/unreachable"
    operator: "Exists"
    effect: "NoSelect"
  - key: "cluster.open-cluster-management.io/unavailable"
    operator: "Exists"
    effect: "NoSelect"
# tolerationSeconds intentionally omitted — defaults to nil (infinite)
```

### Stateless Workloads (Standard Placement)

Omit tolerations entirely, or define a finite `tolerationSeconds` value. The generic
`Placement` controller will automatically relocate the workload to an available cluster
after the specified grace period.

```yaml
tolerations:
  - key: "cluster.open-cluster-management.io/unreachable"
    operator: "Exists"
    effect: "NoSelect"
    tolerationSeconds: 300  # Relocate after 5 minutes
```

### Summary Table

| Workload type | Reconciler      | Toleration strategy              | Failover         |
|---------------|-----------------|----------------------------------|------------------|
| Stateful      | Ramen / DRPC    | Infinite (no `tolerationSeconds`) | Manual (admin)  |
| Stateless     | ACM Placement   | Finite or omitted                | Automatic        |

---

## Fencing and Manual Intervention

Fencing is **not** an automatic feature of asynchronous Regional DR. In an
Active/Passive architecture:

- The primary site is **not** automatically isolated when it becomes unreachable.
- An administrator must **deliberately** trigger the failover sequence.
- This ensures the primary is confirmed down before the secondary begins writing
  to replicated storage, preventing data corruption.

Administrators initiate failover by patching the DRPC resource:

```yaml
spec:
  action: Failover
  failoverCluster: <secondary-cluster-name>
```

---

## Key Takeaways

- Tolerations and DR orchestration operate at **independent layers** and do not conflict.
- Once a `Placement` is annotated for DR, **Ramen controls placement** — toleration
  changes have no effect on DR behaviour.
- Tolerations are a **safety gate**: they suppress the generic scheduler to prevent a
  race condition with storage replication, not to block recovery.
- For stateful workloads, failover is always **human-initiated** via DRPC to ensure
  storage consistency before workload relocation.

---

## References

- [Open Cluster Management — Placement documentation](https://open-cluster-management.io/concepts/placement/)
- [Red Hat OpenShift DR — Regional DR documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation)
- [Ramen project (upstream)](https://github.com/RamenDR/ramen)
