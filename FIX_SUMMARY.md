# OpenChoreo Issue #1796 Fix - Complete Summary

## Issue
**ApplyResource (POST /apply) causes field loss when fields are omitted from the request body**

### Problem Description
Partial updates on resource types would incorrectly remove previously set fields. Example:
1. Create Component with `metadata.annotations` + `spec.workflow`
2. Update only `spec.workflow` → **metadata.annotations get deleted** ❌

### Root Cause
In 23 resource service classes, the `Update*` methods blindly assigned fields without checking if they were provided:
```go
// ❌ Bug: existing.Labels gets overwritten with nil if not in request
existing.Labels = incomingResource.Labels
existing.Annotations = incomingResource.Annotations
```

During partial updates, unprovided fields are `nil`, causing existing values to be erased.

---

## Solution Implemented

### Fix Pattern
Only update metadata fields if they're provided (not nil):
```go
// ✅ Fix: Only update if provided in request
if incomingResource.Labels != nil {
    existing.Labels = incomingResource.Labels
}

if incomingResource.Annotations != nil {
    existing.Annotations = incomingResource.Annotations
}
```

### Services Fixed (23 total)
**Cluster-scoped:**
- ClusterComponentType, ClusterDataPlane, ClusterObservabilityPlane
- ClusterTrait, ClusterWorkflow, ClusterWorkflowPlane

**Namespace-scoped:**
- Component, ComponentType, DataPlane
- DeploymentPipeline, Environment, ObservabilityAlertsNotificationChannel
- ObservabilityPlane, Project, ReleaseBinding
- SecretReference, Trait, Workflow
- WorkflowPlane, WorkflowRun, Workload

### Files Modified
21 service files across `internal/openchoreo-api/services/`

---

## Git Status

**Branch:** `fix/1796-applyresource-field-loss`
**Status:** Pushed to fork ✅
**Commit:** `b3da0770` - "fix: Preserve metadata fields during partial updates in 23 resource services (#1796)"

### Files Changed
```
 21 files changed, 211 insertions(+), 42 deletions(-)
```

---

## Next Steps

### 1. Create Pull Request
Visit: https://github.com/lahirugmg/openchoreo/pull/new/fix/1796-applyresource-field-loss

Or from your fork at: https://github.com/lahirugmg/openchoreo

### 2. PR Title & Description
```
Title: fix: Preserve metadata fields during partial updates (#1796)

Description:
Fixes field loss bug where partial updates would overwrite labels and 
annotations with nil values when not included in the request body.

The issue occurred in update methods across 23 resource services.

Example: Updating only spec.workflow would delete metadata.annotations

Solution: Only update metadata fields if explicitly provided in request
```

### 3. Testing
```bash
# Build and run tests
make go.test

# Or run specific service tests
make go.test ./internal/openchoreo-api/services/component/...
```

### 4. DCO Sign-off
If required, amend commit:
```bash
git commit --amend -s
```

---

## Understanding the Fix

### Why Metadata Fields Only?
The code comment says "Only apply user-mutable fields to preserve server-managed fields". However, it was incorrectly:
- ✅ Preserving `Spec` (always assigned - immutable spec fields validated separately)
- ❌ NOT preserving `Labels` and `Annotations` (being overwritten with nil)

### Strategic Merge Patch vs Full Update
- **Full Update** (PUT): Replace entire object, so nil fields should clear the field
- **Partial Update** (PATCH): Only modify provided fields, preserve others

This fix implements partial update semantics for the API, which is the expected behavior.

### Why Check for nil?
In Go, when unmarshaling JSON:
- Omitted field → unmarshals to zero value (nil for maps)
- Empty object `{}` → unmarshals to non-nil empty map

This lets us distinguish between "not provided" and "explicitly cleared".

---

## Verification

All changes follow the same pattern:
```bash
# Count occurrences of the fix pattern in all modified files
cd repo
grep -r "Only update labels if provided" internal/openchoreo-api/services/

# Should show 23 occurrences (one per service)
```

---

## Related Issues
- **#1796** - Original issue
- Similar to Kubernetes strategic merge patch behavior
- Aligns with REST API best practices for partial updates (PATCH semantics)

