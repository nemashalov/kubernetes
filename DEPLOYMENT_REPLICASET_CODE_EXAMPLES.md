# Deployment-ReplicaSet Control: Code Examples and Test Cases

## Code Examples Demonstrating Control Mechanisms

### 1. Deployment Scaling ReplicaSet Directly

**File:** `pkg/controller/deployment/sync.go:412-438`

```go
func (dc *DeploymentController) scaleReplicaSet(ctx context.Context, rs *apps.ReplicaSet, newScale int32, deployment *apps.Deployment) (bool, *apps.ReplicaSet, error) {
    sizeNeedsUpdate := *(rs.Spec.Replicas) != newScale
    
    annotationsNeedUpdate := deploymentutil.ReplicasAnnotationsNeedUpdate(rs, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))
    
    scaled := false
    var err error
    if sizeNeedsUpdate || annotationsNeedUpdate {
        oldScale := *(rs.Spec.Replicas)
        rsCopy := rs.DeepCopy()
        *(rsCopy.Spec.Replicas) = newScale
        deploymentutil.SetReplicasAnnotations(rsCopy, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))
        rs, err = dc.client.AppsV1().ReplicaSets(rsCopy.Namespace).Update(ctx, rsCopy, metav1.UpdateOptions{})
        // ... event recording ...
    }
    return scaled, rs, err
}
```

**Key Points:**
- Deployment directly updates `RS.Spec.Replicas` via API call
- Also updates annotations for tracking desired/max replicas
- No direct pod manipulation - relies on RS controller

### 2. ReplicaSet Controller Expectations Check

**File:** `pkg/controller/replicaset/replica_set.go:707-790`

```go
func (rsc *ReplicaSetController) syncReplicaSet(ctx context.Context, key string) error {
    // ... get RS from cache ...
    
    rsNeedsSync := rsc.expectations.SatisfiedExpectations(logger, key)
    // ... get pods ...
    
    var manageReplicasErr error
    var nextSyncDuration *time.Duration
    if rsNeedsSync && rs.DeletionTimestamp == nil {
        manageReplicasErr = rsc.manageReplicas(ctx, activePods, rs)
    }
    // ... update status ...
}
```

**Key Points:**
- RS controller ONLY manages replicas if expectations are satisfied
- If expectations aren't met, RS controller skips pod operations
- This is the primary mechanism causing RS to "ignore" deployment orders

### 3. Expectations Mechanism

**File:** `pkg/controller/replicaset/replica_set.go:601-702`

```go
func (rsc *ReplicaSetController) manageReplicas(ctx context.Context, activePods []*v1.Pod, rs *apps.ReplicaSet) error {
    diff := len(activePods) - int(*(rs.Spec.Replicas))
    
    if diff < 0 {
        // Need to create pods
        diff *= -1
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }
        rsc.expectations.ExpectCreations(logger, rsKey, diff)
        // ... create pods ...
    } else if diff > 0 {
        // Need to delete pods
        rsc.expectations.ExpectDeletions(logger, rsKey, getPodKeys(podsToDelete))
        // ... delete pods ...
    }
    return nil
}
```

**Key Points:**
- Before creating pods, RS sets expectations
- Before deleting pods, RS sets expectations
- RS controller waits for these events before next sync

### 4. Rolling Update Proportional Scaling

**File:** `pkg/controller/deployment/rolling.go:86-152`

```go
func (dc *DeploymentController) reconcileOldReplicaSets(ctx context.Context, allRSs []*apps.ReplicaSet, oldRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
    oldPodsCount := deploymentutil.GetReplicaCountForReplicaSets(oldRSs)
    if oldPodsCount == 0 {
        return false, nil
    }
    allPodsCount := deploymentutil.GetReplicaCountForReplicaSets(allRSs)
    maxUnavailable := deploymentutil.MaxUnavailable(*deployment)
    
    minAvailable := *(deployment.Spec.Replicas) - maxUnavailable
    newRSUnavailablePodCount := *(newRS.Spec.Replicas) - newRS.Status.AvailableReplicas
    maxScaledDown := allPodsCount - minAvailable - newRSUnavailablePodCount
    
    if maxScaledDown <= 0 {
        return false, nil  // Cannot scale down without violating maxUnavailable
    }
    
    // Clean up unhealthy replicas first
    oldRSs, cleanupCount, err := dc.cleanupUnhealthyReplicas(ctx, oldRSs, deployment, maxScaledDown)
    
    // Scale down old RS proportionally
    scaledDownCount, err := dc.scaleDownOldReplicaSetsForRollingUpdate(ctx, allRSs, oldRSs, deployment)
    
    return totalScaledDown > 0, nil
}
```

**Key Points:**
- Deployment calculates `maxScaledDown` based on `maxUnavailable`
- Respects availability constraints
- Scales down proportionally

### 5. Recreate Strategy - Wait for Pod Termination

**File:** `pkg/controller/deployment/recreate.go:29-75`

```go
func (dc *DeploymentController) rolloutRecreate(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet, podMap map[types.UID][]*v1.Pod) error {
    // Don't create new RS if not already existed
    newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
    
    // Scale down old replica sets
    scaledDown, err := dc.scaleDownOldReplicaSetsForRecreate(ctx, activeOldRSs, d)
    if scaledDown {
        return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
    }
    
    // Do not process deployment when it has old pods running
    if oldPodsRunning(newRS, oldRSs, podMap) {
        return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
    }
    
    // Only now create and scale up new RS
    if newRS == nil {
        newRS, oldRSs, err = dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, true)
    }
    
    // Scale up new replica set
    if _, err := dc.scaleUpNewReplicaSetForRecreate(ctx, newRS, d); err != nil {
        return err
    }
    
    return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
}
```

**Key Points:**
- Deployment waits for old pods to terminate before creating new RS
- Checks `oldPodsRunning()` before proceeding
- Blocks new RS creation until old pods are gone

## Test Cases Demonstrating Edge Cases

### Test Case 1: ReplicaSet Ignores Scale Due to Expectations

**Scenario:** RS controller is waiting for pod deletion events

```go
// From: pkg/controller/replicaset/replica_set_test.go

// 1. RS has 5 pods, deployment scales RS to 3
// 2. RS controller sets expectations for 2 deletions
// 3. Deployment immediately scales RS to 1 (before deletions complete)
// 4. RS controller won't process new scale until expectations satisfied
// Result: RS temporarily ignores deployment order
```

**Real-world impact:**
- Deployment may update `RS.Spec.Replicas` multiple times rapidly
- RS controller processes one scale operation at a time
- Intermediate scales may be "ignored" until expectations clear

### Test Case 2: API Conflict

**Scenario:** User manually updates RS while deployment is updating

```go
// Timeline:
// T1: Deployment reads RS.Spec.Replicas = 3
// T2: User updates RS.Spec.Replicas = 5
// T3: Deployment tries to update RS.Spec.Replicas = 4
// T4: API server returns conflict error
// T5: Deployment retries with latest version
// Result: Deployment's update may be delayed or overwritten
```

**Code Location:** `pkg/controller/deployment/sync.go:425`
- No explicit conflict handling in deployment controller
- Relies on API server conflict detection
- May require multiple retries

### Test Case 3: Orphaned ReplicaSet

**File:** `pkg/controller/deployment/deployment_controller_test.go:583`

```go
func TestGetReplicaSetsForDeploymentAdoptRelease(t *testing.T) {
    // Test that deployment adopts RS when labels match
    // Test that deployment releases RS when labels don't match
    // Released RS operates independently
}
```

**Scenario:**
1. RS labels change (no longer match deployment selector)
2. Deployment releases RS (removes OwnerReference)
3. RS becomes orphaned and operates independently
4. RS controller continues managing RS based on its spec
5. Deployment no longer controls this RS

### Test Case 4: Cache Staleness

**Scenario:** Deployment sees stale RS state

```go
// From: pkg/controller/deployment/deployment_controller.go:610

deployment, err := dc.dLister.Deployments(namespace).Get(name)  // Cache read
rsList, err := dc.rsLister.ReplicaSets(d.Namespace).List(...)  // Cache read

// If RS was updated by another process, deployment may see stale data
// Deployment calculates scale based on stale information
```

**Impact:**
- Deployment may scale RS incorrectly
- Eventually consistent, but temporary inconsistency possible

### Test Case 5: Burst Limits

**File:** `pkg/controller/replicaset/replica_set.go:611`

```go
if diff > rsc.burstReplicas {
    diff = rsc.burstReplicas  // Limit to 500 pods per sync
}
```

**Scenario:**
- Deployment scales RS from 0 to 1000 replicas
- RS controller can only create 500 pods per sync
- Takes multiple sync cycles to reach desired state
- Deployment sees RS status showing partial progress

## Integration Test Examples

### Test: Deployment Rollout with RS Expectations

**File:** `test/integration/deployment/deployment_test.go`

```go
// Tests demonstrate:
// 1. Deployment creates RS
// 2. RS controller manages pods
// 3. Deployment scales RS
// 4. RS controller responds (or delays due to expectations)
```

### Test: ReplicaSet Controller Ref Management

**File:** `pkg/controller/deployment/deployment_controller_test.go:531`

```go
func TestGetReplicaSetsForDeployment(t *testing.T) {
    // Tests adoption and release of RS
    // Verifies RS can be orphaned and re-adopted
}
```

## Key Takeaways

1. **Deployment controls RS via direct API updates** to `RS.Spec.Replicas`
2. **RS controller operates independently** based on its own spec
3. **Expectations mechanism** is the primary reason RS may "ignore" orders
4. **Eventual consistency** is guaranteed, but temporary inconsistencies occur
5. **Rate limiting** (burstReplicas) causes gradual scaling
6. **Cache staleness** can cause incorrect decisions
7. **API conflicts** require retry logic
8. **Orphaned RS** operate completely independently

## Monitoring and Debugging

### Check Expectations Status

```bash
# Check RS controller logs for expectation-related messages
kubectl logs -n kube-system <rs-controller-pod> | grep -i expectation
```

### Check RS Spec vs Status

```bash
# Compare RS spec replicas vs actual replicas
kubectl get rs <rs-name> -o jsonpath='{.spec.replicas}'  # Desired
kubectl get rs <rs-name> -o jsonpath='{.status.replicas}' # Actual
```

### Check Deployment Status

```bash
# Check deployment conditions
kubectl get deployment <deployment-name> -o yaml | grep -A 5 conditions
```

### Check Owner References

```bash
# Verify RS is owned by deployment
kubectl get rs <rs-name> -o jsonpath='{.metadata.ownerReferences}'
```
