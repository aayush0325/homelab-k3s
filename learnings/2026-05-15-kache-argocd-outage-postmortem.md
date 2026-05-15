# Postmortem: kache ArgoCD SyncFailure & SharedResourceWarnings

**Date:** 2026-05-15
**Status:** Resolved
**Severity:** High — kache-prod and kache-dev applications failed to sync
**Authors:** aayush0325

## Summary

After refactoring kache manifests into kustomize base/overlays with strategic merge patches (commit `fd31b2d`), both the `kache-prod` and `kache-dev` ArgoCD applications failed to sync. Two distinct issues surfaced:

1. **SyncError:** StatefulSet `redis-statefulset` was rejected by the API server because `volumeClaimTemplates[0].spec.accessModes` was missing — a required field.
2. **SharedResourceWarning:** All resources (ConfigMap, Deployment, Namespace, Services) were shared between both `kache-prod` and `kache-dev` applications, causing ArgoCD to warn about resource ownership conflicts.

## Timeline

| Time (IST) | Event |
|---|---|
| ~12:21 | First `RepeatedResourceWarning` — ConfigMap appeared 2x |
| ~12:26 | `SyncError` — volumeClaimTemplates accessModes missing (retried 5 times) |
| ~18:03 | Continued `RepeatedResourceWarning` — resources appearing 3x |
| ~18:18 | `SyncError` on kache-prod — same accessModes issue (2 retries) |
| ~18:22 | `SharedResourceWarning` — 5 resources shared between kache-prod and kache-dev |
| ~18:25 | Fix committed (`b4cde6a`) — added accessModes + storageClassName to prod patch |
| ~18:26 | Fix committed (`202ea39`) — added accessModes + storageClassName to dev patch + separated namespaces |
| ~18:26 | Both apps synced and healthy |

## Root Cause Analysis

### Issue 1: Missing `accessModes` in volumeClaimTemplates

After commit `fd31b2d` restructured manifests into kustomize overlays, the strategic merge patches for `redis-statefulset` in both dev and prod overlays only specified the fields they wanted to override (e.g., storage size, resources). However, kustomize does **not** deep-merge `volumeClaimTemplates` entries — it replaces the entire list entry matched by `name`.

This is because `volumeClaimTemplates` is a non-strategic-merge-list in the Kubernetes API for StatefulSet. It lacks `patchMergeKey` and `patchStrategy` directives, so kustomize replaces the whole entry rather than merging individual sub-fields. The fields `accessModes` and `storageClassName` from the base manifest were silently dropped.

**ArgoCD error message:**
```
StatefulSet.apps "redis-statefulset" is invalid: spec.volumeClaimTemplates[0].spec.accessModes: Required value: at least 1 access mode is required
```

**Affected files:**
- `manifests/kache/overlays/prod/redis-statefulset-patch.yaml`
- `manifests/kache/overlays/dev/redis-statefulset-patch.yaml`

### Issue 2: SharedResourceWarning — resources in same namespace

Both `kache-prod` and `kache-dev` ArgoCD applications were deploying to the same `kache` namespace with identical resource names. This caused ArgoCD to flag all resources as shared between two applications:

```
SharedResourceWarning: ConfigMap/kache-config is part of applications argocd/kache-prod and kache-dev
SharedResourceWarning: Deployment/kache-deployment is part of applications argocd/kache-prod and kache-dev
SharedResourceWarning: Namespace/kache is part of applications argocd/kache-prod and kache-dev
SharedResourceWarning: Service/kache-service-clusterip is part of applications argocd/kache-prod and kache-dev
SharedResourceWarning: Service/redis-service-headless is part of applications argocd/kache-prod and kache-dev
```

## Resolution

### Fix 1: Include all required fields in volumeClaimTemplates patches

Added `accessModes` and `storageClassName` to both overlay patches since kustomize replaces the entire volumeClaimTemplates entry rather than merging it.

**Commit:** `b4cde6a` — `fix(kache): add missing accessModes and storageClassName to prod volumeClaimTemplates patch`

Prod patch (`manifests/kache/overlays/prod/redis-statefulset-patch.yaml`):
```yaml
volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]      # added
      storageClassName: "local-path"         # added
      resources:
        requests:
          storage: "1Gi"
```

Dev patch (`manifests/kache/overlays/dev/redis-statefulset-patch.yaml`):
```yaml
volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]      # added
      storageClassName: "local-path"         # added
      resources:
        requests:
          storage: "256Mi"
```

### Fix 2: Separate namespaces for dev and prod

Added `namespace` directives to each overlay's `kustomization.yaml`:

- Dev → `namespace: kache-dev`
- Prod → `namespace: kache-prod`

Updated ArgoCD application destinations:
```
argocd app set kache-prod --dest-namespace kache-prod
argocd app set kache-dev --dest-namespace kache-dev
```

**Commit:** `202ea39` — `fix(kache): separate dev/prod namespaces and add missing volumeClaimTemplates fields`

## Known Upstream Issues

This is a well-documented limitation of kustomize's strategic merge patch handling for StatefulSet `volumeClaimTemplates`:

- **[kubernetes-sigs/kustomize#819](https://github.com/kubernetes-sigs/kustomize/issues/819)** — VolumeClaimTemplates overwritten by StrategicMergePatch (Feb 2019, closed). The original report of this exact bug: patching only `storage` causes `accessModes` and `storageClassName` to be lost.
- **[kubernetes-sigs/kustomize#1844](https://github.com/kubernetes-sigs/kustomize/issues/1844)** — Updating volumeClaimTemplates storage in StatefulSet (Nov 2019, closed). Same issue reported independently.
- **[kubernetes-sigs/kustomize#6000](https://github.com/kubernetes-sigs/kustomize/issues/6000)** — StatefulSet volumeClaimTemplates labels not properly merged after patch (Oct 2025, **still open**). Confirms the issue persists — labels are also lost when patching volumeClaimTemplates.
- **[kubernetes-sigs/kustomize#5110](https://github.com/kubernetes-sigs/kustomize/issues/5110)** — Allow custom merge keys (Mar 2023, open). Feature request to address the root cause.

## Lessons Learned

1. **Always include all required fields** in kustomize patches for `volumeClaimTemplates`. Since it's treated as a replace-by-name list (not a strategic merge list), any field not in the patch is silently dropped. This applies to `accessModes`, `storageClassName`, `volumeMode`, and any other sub-fields.

2. **Use separate namespaces for different environments.** Dev and prod overlays should deploy to distinct namespaces to avoid SharedResourceWarning and potential resource conflicts in ArgoCD.

3. **Validate rendered output.** Running `kubectl apply --dry-run=server` or `kustomize build` against the rendered manifests before pushing would have caught the missing `accessModes` field immediately.

4. **Test overlay patches independently.** When creating strategic merge patches, verify the full rendered output (`kustomize build`) to ensure no base fields are unexpectedly dropped.

## Action Items

- [x] Add `accessModes` and `storageClassName` to volumeClaimTemplates patches (both overlays)
- [x] Separate dev/prod into `kache-dev` / `kache-prod` namespaces
- [x] Update ArgoCD app destinations to match new namespaces
- [x] Prune old `kache` namespace resources from kache-dev app (`argocd app sync kache-dev --prune`)
- [ ] Add CI check to validate `kustomize build` output for each overlay before merge