# Deployment Control of ReplicaSet States - Analysis

## Overview

This document analyzes how Kubernetes Deployments control ReplicaSet states and identifies cases where ReplicaSets may ignore or delay responding to Deployment orders.

## How Deployment Controls ReplicaSet States

### 1. **Direct Spec Updates**

The Deployment controller directly updates ReplicaSet specifications by calling the Kubernetes API:

**Key Functions:**
- `scaleReplicaSet()` in `pkg/controller/deployment/sync.go:412`
- Updates `ReplicaSet.Spec.Replicas` directly via `client.AppsV1().ReplicaSets().Update()`

```go
func (dc *DeploymentController) scaleReplicaSet(ctx context.Context, rs *apps.ReplicaSet, newScale int32, deployment *apps.Deployment) (bool, *apps.ReplicaSet, error) {
    sizeNeedsUpdate := *(rs.Spec.Replicas) != newScale
    if sizeNeedsUpdate {
        rsCopy := rs.DeepCopy()
        *(rsCopy.Spec.Replicas) = newScale
        rs, err = dc.client.AppsV1().ReplicaSets(rsCopy.Namespace).Update(ctx, rsCopy, metav1.UpdateOptions{})
    }
    return scaled, rs, err
}
```

### 2. **State Control Mechanisms**

#### A. **Rolling Update Strategy** (`pkg/controller/deployment/rolling.go`)

1. **Scale Up New ReplicaSet** (`reconcileNewReplicaSet`)
   - Calculates desired replicas for new RS based on deployment spec
   - Scales new RS up to deployment replicas count
   - Respects `maxSurge` limits

2. **Scale Down Old ReplicaSets** (`reconcileOldReplicaSets`)
   - Calculates `maxScaledDown` based on `maxUnavailable`
   - Cleans up unhealthy replicas first
   - Scales down proportionally while maintaining availability

3. **Proportional Scaling** (`scale()` in `sync.go:307`)
   - Distributes replicas proportionally across old and new RS
   - Uses annotations (`deployment.kubernetes.io/desired-replicas`, `deployment.kubernetes.io/max-replicas`)
   - Ensures total replicas don't exceed `deployment.spec.replicas + maxSurge`

#### B. **Recreate Strategy** (`pkg/controller/deployment/recreate.go`)

1. **Scale Down Old RS** (`scaleDownOldReplicaSetsForRecreate`)
   - Scales all old RS to 0
   - Waits for all pods to terminate

2. **Scale Up New RS** (`scaleUpNewReplicaSetForRecreate`)
   - Only scales up after old pods are terminated
   - Scales to full deployment replica count

#### C. **Scaling Events** (`sync.go:307`)

When deployment is paused or during scaling events:
- Scales proportionally across all active RS
- Updates annotations on RS for tracking

### 3. **ReplicaSet Lifecycle Management**

#### A. **Creation** (`sync.go:146`)

- Creates new RS when pod template changes
- Sets `OwnerReference` pointing to Deployment
- Adds `pod-template-hash` label for uniqueness

#### B. **Adoption/Orphaning** (`deployment_controller.go:531`)

- Uses `ControllerRefManager` to adopt orphaned RS
- Releases RS that no longer match selector
- Handles RS with changed labels

#### C. **Cleanup** (`sync.go:443`)

- Deletes old RS beyond `revisionHistoryLimit`
- Only deletes RS with 0 replicas and no deletion timestamp

### 4. **Status Synchronization**

Deployment reads RS status to update its own status:
- `calculateStatus()` aggregates RS statuses
- Tracks: `Replicas`, `UpdatedReplicas`, `ReadyReplicas`, `AvailableReplicas`
- Updates deployment conditions based on RS states

## Cases Where ReplicaSet May Ignore Deployment Orders

### 1. **Expectations Mechanism** ⚠️ **PRIMARY REASON**

**Location:** `pkg/controller/replicaset/replica_set.go:728`

The ReplicaSet controller uses an **expectations mechanism** that prevents it from syncing if expectations aren't satisfied:

```go
rsNeedsSync := rsc.expectations.SatisfiedExpectations(logger, key)
if rsNeedsSync && rs.DeletionTimestamp == nil {
    manageReplicasErr = rsc.manageReplicas(ctx, activePods, rs)
}
```

**When RS ignores orders:**
- RS controller is waiting for pod creation events that haven't arrived
- RS controller is waiting for pod deletion events that haven't arrived
- Pod operations are in-flight and expectations haven't been observed

**Impact:** Deployment may update `RS.Spec.Replicas`, but RS controller won't act until expectations are satisfied.

### 2. **API Update Conflicts**

**Location:** `pkg/controller/deployment/sync.go:425`

If multiple controllers update the same RS simultaneously:
- API server may reject updates with conflict errors
- Deployment controller will retry, but RS may have different state

**Scenario:**
- Deployment updates RS.Spec.Replicas to 5
- User manually updates RS.Spec.Replicas to 3
- RS controller processes the manual update first
- Deployment's update may conflict or be overwritten

### 3. **ReplicaSet Deletion State**

**Location:** `pkg/controller/replicaset/replica_set.go:757`

```go
if rsNeedsSync && rs.DeletionTimestamp == nil {
    manageReplicasErr = rsc.manageReplicas(ctx, activePods, rs)
}
```

**When RS ignores orders:**
- RS has `DeletionTimestamp` set (being deleted)
- RS controller stops managing replicas during deletion
- Deployment may still try to scale it, but RS won't respond

### 4. **Cache Staleness**

**Location:** `pkg/controller/deployment/deployment_controller.go`

Both controllers use informers with cached data:
- Deployment controller may see stale RS state
- RS controller may see stale pod state
- Updates may be based on outdated information

**Scenario:**
- Deployment reads cached RS with `Spec.Replicas=3`
- RS was actually updated to `Spec.Replicas=5` by another process
- Deployment calculates wrong scale target

### 5. **Burst Limits**

**Location:** `pkg/controller/replicaset/replica_set.go:611`

```go
if diff > rsc.burstReplicas {
    diff = rsc.burstReplicas
}
```

**When RS ignores orders:**
- Large scale operations are throttled by `burstReplicas` (default 500)
- RS controller creates/deletes pods in batches
- Full scale operation may take multiple sync cycles

### 6. **Pod Creation/Deletion Failures**

**Location:** `pkg/controller/replicaset/replica_set.go:601`

If pod operations fail:
- RS controller will retry, but expectations may block further operations
- Failed pods may prevent RS from reaching desired state
- Deployment sees incorrect RS status

### 7. **Controller Sync Delays**

**Location:** Both controllers use work queues

**When RS ignores orders:**
- RS controller queue may be backlogged
- RS sync may be delayed due to rate limiting
- Deployment updates RS spec, but RS controller hasn't processed it yet

### 8. **Orphaned ReplicaSets**

**Location:** `pkg/controller/deployment/deployment_controller.go:247`

**When RS ignores orders:**
- RS loses OwnerReference (orphaned)
- Deployment may not adopt it immediately
- RS operates independently until adoption

### 9. **Label Selector Mismatch**

**Location:** `pkg/controller/controller_ref_manager.go:323`

```go
match := func(obj metav1.Object) bool {
    return m.Selector.Matches(labels.Set(obj.GetLabels()))
}
```

**When RS ignores orders:**
- RS labels change and no longer match deployment selector
- Deployment releases the RS (orphans it)
- RS continues operating independently

### 10. **MinReadySeconds Delay**

**Location:** `pkg/controller/replicaset/replica_set.go:777`

```go
if updatedRS.Spec.MinReadySeconds > 0 &&
    updatedRS.Status.ReadyReplicas != updatedRS.Status.AvailableReplicas {
    nextSyncDuration = ptr.To(time.Duration(updatedRS.Spec.MinReadySeconds) * time.Second)
    rsc.queue.AddAfter(key, *nextSyncDuration)
}
```

**When RS ignores orders:**
- RS waits for `MinReadySeconds` before marking pods available
- Status updates are delayed
- Deployment may see incorrect available replica count

## Key Code Locations

### Deployment Controller
- **Main sync:** `pkg/controller/deployment/deployment_controller.go:596`
- **Scaling:** `pkg/controller/deployment/sync.go:412`
- **Rolling update:** `pkg/controller/deployment/rolling.go:31`
- **Recreate:** `pkg/controller/deployment/recreate.go:29`

### ReplicaSet Controller
- **Main sync:** `pkg/controller/replicaset/replica_set.go:707`
- **Manage replicas:** `pkg/controller/replicaset/replica_set.go:601`
- **Expectations check:** `pkg/controller/replicaset/replica_set.go:728`

### Controller Reference Management
- **Adopt/Release:** `pkg/controller/controller_ref_manager.go:319`

## Recommendations

1. **Monitor Expectations:** Check RS controller logs for expectation-related delays
2. **Handle Conflicts:** Implement retry logic for API conflicts
3. **Status Reconciliation:** Deployment should periodically reconcile RS status
4. **Avoid Manual RS Updates:** Don't manually modify RS owned by Deployment
5. **Watch for Orphaning:** Monitor for RS losing OwnerReference

## Conclusion

While Deployment has direct control over ReplicaSet specifications, the ReplicaSet controller operates independently and may delay or ignore orders due to:
- **Expectations mechanism** (most common)
- **API conflicts**
- **Deletion states**
- **Cache staleness**
- **Rate limiting**

The system is designed to eventually converge, but temporary inconsistencies are expected during normal operations.
