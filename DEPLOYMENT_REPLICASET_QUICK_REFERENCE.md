# Deployment-ReplicaSet Control: Quick Reference

## Control Flow Summary

```
Deployment Controller                    ReplicaSet Controller
─────────────────────                    ─────────────────────
1. Watches Deployment changes            1. Watches ReplicaSet changes
2. Calculates desired RS state           2. Watches Pod changes
3. Updates RS.Spec.Replicas (API) ────>  3. Reads RS.Spec.Replicas
4. Updates RS annotations                4. Checks expectations
                                         5. If satisfied: manages pods
                                         6. Updates RS.Status
```

## How Deployment Controls ReplicaSet

| Control Aspect | Mechanism | Code Location |
|---------------|-----------|---------------|
| **Replica Count** | Direct API update to `RS.Spec.Replicas` | `sync.go:412` |
| **Scaling Strategy** | RollingUpdate vs Recreate logic | `rolling.go:31`, `recreate.go:29` |
| **Proportional Scaling** | Annotations + calculations | `sync.go:307` |
| **Lifecycle** | Create/Adopt/Release/Delete | `deployment_controller.go:531` |
| **Status Tracking** | Reads RS status, aggregates | `sync.go:495` |

## ReplicaSet States Controlled by Deployment

1. **Spec.Replicas** - Directly updated by deployment
2. **Annotations** - `deployment.kubernetes.io/desired-replicas`, `max-replicas`
3. **OwnerReference** - Points to deployment (for adoption/release)
4. **Labels** - Deployment adds `pod-template-hash` label
5. **Revision** - Deployment manages revision annotations

## When ReplicaSet May Ignore/Delay Deployment Orders

| Reason | Severity | Impact Duration | Mitigation |
|-------|----------|-----------------|------------|
| **Expectations not satisfied** | High | Seconds to minutes | Wait for pod events |
| **API update conflicts** | Medium | Milliseconds | Retry logic |
| **RS being deleted** | High | Until deletion completes | Check DeletionTimestamp |
| **Cache staleness** | Low | Until next resync | Eventual consistency |
| **Burst limits** | Low | Multiple sync cycles | Gradual scaling |
| **Pod operation failures** | Medium | Until retry succeeds | Error handling |
| **Queue backlog** | Low | Until queue clears | Rate limiting |
| **Orphaned RS** | High | Until re-adoption | Monitor OwnerRef |
| **Label mismatch** | High | Permanent if not fixed | Fix labels |
| **MinReadySeconds delay** | Low | Seconds | Status delay only |

## Key Code Functions

### Deployment Controller

```go
// Main sync entry point
syncDeployment()                    // deployment_controller.go:596

// Scaling operations
scaleReplicaSet()                   // sync.go:412
scaleReplicaSetAndRecordEvent()     // sync.go:403

// Rolling update
rolloutRolling()                    // rolling.go:31
reconcileNewReplicaSet()            // rolling.go:68
reconcileOldReplicaSets()           // rolling.go:86

// Recreate strategy
rolloutRecreate()                   // recreate.go:29
scaleDownOldReplicaSetsForRecreate() // recreate.go:78
scaleUpNewReplicaSetForRecreate()   // recreate.go:129

// RS lifecycle
getReplicaSetsForDeployment()        // deployment_controller.go:531
getNewReplicaSet()                  // sync.go:146
cleanupDeployment()                 // sync.go:443
```

### ReplicaSet Controller

```go
// Main sync entry point
syncReplicaSet()                    // replica_set.go:707

// Pod management
manageReplicas()                    // replica_set.go:601

// Expectations check
SatisfiedExpectations()             // controller/expectations.go

// Status updates
updateReplicaSetStatus()            // replica_set_utils.go:41
calculateStatus()                    // replica_set_utils.go:96
```

## State Transition Examples

### Rolling Update

```
Initial:  RS-old (10 pods), RS-new (0 pods)
Step 1:   RS-old (8 pods), RS-new (2 pods)  [Deployment scales both]
Step 2:   RS-old (6 pods), RS-new (4 pods)
Step 3:   RS-old (4 pods), RS-new (6 pods)
Step 4:   RS-old (2 pods), RS-new (8 pods)
Step 5:   RS-old (0 pods), RS-new (10 pods)
Final:    RS-old deleted, RS-new (10 pods)
```

### Recreate

```
Initial:  RS-old (10 pods), RS-new (nil)
Step 1:   RS-old (0 pods), RS-new (nil)     [Scale down old]
Step 2:   Wait for pods to terminate
Step 3:   RS-old (0 pods), RS-new (10 pods) [Create & scale new]
Final:    RS-old deleted, RS-new (10 pods)
```

## Common Issues and Solutions

### Issue: RS not scaling despite deployment update

**Check:**
1. RS expectations status (logs)
2. RS DeletionTimestamp
3. API conflicts (retries)
4. OwnerReference (is RS orphaned?)

**Solution:**
- Wait for expectations to clear
- Check RS controller logs
- Verify deployment owns RS

### Issue: RS scales too slowly

**Check:**
1. Burst limits (500 pods/sync)
2. Pod creation failures
3. Queue backlog

**Solution:**
- Normal for large scales
- Check pod events for failures
- Monitor RS controller queue

### Issue: RS operates independently

**Check:**
1. OwnerReference missing?
2. Labels match deployment selector?
3. RS orphaned?

**Solution:**
- Fix labels to match selector
- Deployment will re-adopt
- Or manually delete orphaned RS

## Debugging Commands

```bash
# Check deployment status
kubectl get deployment <name> -o yaml

# Check RS status
kubectl get rs -l <selector> -o wide

# Check RS expectations (via logs)
kubectl logs -n kube-system <rs-controller-pod> | grep expectation

# Check OwnerReference
kubectl get rs <name> -o jsonpath='{.metadata.ownerReferences}'

# Check annotations
kubectl get rs <name> -o jsonpath='{.metadata.annotations}'

# Watch RS changes
kubectl get rs -w

# Check pod controller refs
kubectl get pod <name> -o jsonpath='{.metadata.ownerReferences}'
```

## Related Files

- **Deployment Controller:** `pkg/controller/deployment/`
- **ReplicaSet Controller:** `pkg/controller/replicaset/`
- **Controller Ref Manager:** `pkg/controller/controller_ref_manager.go`
- **Deployment Utils:** `pkg/controller/deployment/util/deployment_util.go`
- **Tests:** `pkg/controller/deployment/deployment_controller_test.go`
- **Integration Tests:** `test/integration/deployment/`

## Further Reading

- `DEPLOYMENT_REPLICASET_CONTROL_ANALYSIS.md` - Detailed analysis
- `DEPLOYMENT_REPLICASET_CODE_EXAMPLES.md` - Code examples and test cases
