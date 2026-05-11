# Architectural Decision Record: ODF Regional-DR Placement Strategy

## Context
When deploying stateful applications across multiple OpenShift clusters using **OpenShift Data Foundation (ODF)** and **OpenShift DR (ODR)**, a critical configuration choice must be made regarding how the Control Plane (ACM) reacts to cluster unavailability. 

In standard Kubernetes, the `tolerationSeconds` parameter is used to automatically evict and reschedule workloads when a node or cluster becomes unhealthy. In a Disaster Recovery (DR) context, this automatic behavior is dangerous.

## Decision: Omission of `tolerationSeconds`
For all workloads governed by a `DRPolicy`, the `Placement` resource must **omit** `tolerationSeconds`. This ensures that if a cluster becomes `Unreachable`, the application remains "stuck" (not rescheduled) until an administrator intervenes.

## Rationale

### 1. Prevention of Data Corruption (Split-Brain)
In a Regional-DR scenario, storage is replicated asynchronously between sites. If a network partition occurs and the application fails over automatically:
* The **Primary Cluster** might still be active but partitioned, continuing to write to storage.
* The **Secondary Cluster** might spin up the application and attempt to write to its local replica.
* **Result:** Data divergence or "split-brain," which can lead to permanent data loss or corruption once connectivity is restored.

### 2. Mandatory Fencing
Safety in DR requires **Fencing**—the process of ensuring the failed cluster is completely isolated from the storage backend. By preventing automatic failover, we ensure that an administrator has the opportunity to verify that the primary site is properly fenced before promoting the secondary site.

### 3. Orchestration via DRPlacementControl (DRPC)
By omitting the timeout, the responsibility for workload movement shifts from the generic Kubernetes scheduler to the **DRPlacementControl** custom resource. 

The recovery flow becomes:
1.  **Detection:** The primary cluster is lost. The application stays "stuck" on the primary.
2.  **Intervention:** An admin (or higher-level automation) triggers a `Failover` via the `DRPlacementControl` API.
3.  **Storage Transition:** ODF promotes the secondary storage to `Primary`.
4.  **Workload Relocation:** Only after storage is ready, the DRPC updates the Placement to move the workload to the secondary cluster.

## Comparison Table

| Feature | Standard Configuration | ODF DR Configuration |
| :--- | :--- | :--- |
| **`tolerationSeconds`** | Defined (e.g., 300s) | **None (Omitted)** |
| **Failover Trigger** | Automatic / Timeout-based | Manual / DRPC API-based |
| **Primary Goal** | High Availability (HA) | Data Integrity (DI) |
| **Risk Profile** | High risk of Split-Brain | Low risk; controlled recovery |

## Example Placement Configuration

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: stateful-app-placement
  namespace: my-app-namespace
spec:
  clusterSets:
    - global-dr-set
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            dr-role: primary
```

# Note: tolerations are not defined here. 
# This ensures the workload stays pinned to the cluster 
# even if it reports a 'NotReady' or 'Unreachable' status.
Conclusion
The lack of tolerationSeconds is a deliberate design choice to protect stateful data. It forces a coordinated recovery where storage readiness is guaranteed before the application layer is allowed to restart on a new site.
