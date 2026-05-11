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
