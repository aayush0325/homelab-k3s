# Postmortem: kube-prometheus-stack CRD Annotation Size Limit

**Date:** 2026-05-15
**Status:** Resolved
**Severity:** High — kube-prometheus-stack application failed to sync, blocking monitoring stack deployment
**Authors:** aayush0325

## Summary

When deploying kube-prometheus-stack v85.0.2 via ArgoCD, the sync repeatedly failed with the error:

```
CustomResourceDefinition.apiextensions.k8s.io "prometheuses.monitoring.coreos.com" is invalid:
metadata.annotations: Too long: may not be more than 262144 bytes
```

Six CRDs (alertmanagerconfigs, alertmanagers, prometheusagents, prometheuses, scrapeconfigs, thanosrulers) exceeded the Kubernetes annotation size limit of 256KB. Because the CRDs failed to install, subsequent resources that depend on those CRD types (Alertmanager, Prometheus) also failed with "no matches for kind" errors.

The initial fix attempt — adding `ServerSideApply=true` to the sync options — was insufficient on its own because ArgoCD's server-side apply still stores the full resource in the `kubectl.kubernetes.io/last-applied-configuration` annotation, which exceeds the 256KB limit for large CRDs.

## Timeline

| Time (IST) | Event |
|---|---|
| ~18:07 | ArgoCD Application `kube-prometheus-stack` created, sync initiated |
| ~18:07 | Initial sync attempt begins |
| ~18:07-18:15 | ArgoCD retries sync — CRD annotation size errors on 6 CRDs |
| ~18:15 | Sync operation starts with 5 retries |
| ~18:15-18:21 | Repeated sync failures — `metadata.annotations: Too long: may not be more than 262144 bytes` for alertmanagerconfigs, alertmanagers, prometheusagents, prometheuses, scrapeconfigs, thanosrulers CRDs |
| ~18:21 | Sync fails after 5 retries. Phase: `Failed` |
| ~18:11 | First fix: added `ServerSideApply=true` to `kube-prometheus-stack.yaml`, committed and pushed |
| ~18:15 | Fix insufficient — same CRD annotation errors persist |
| ~18:17 | Second fix: split into two-phase deployment — CRDs app (sync-wave 0) with SSA, main app (sync-wave 1) with CRDs disabled |
| ~18:18 | Created `apps/kube-prometheus-stack-crds.yaml` and `values/kube-prometheus-stack/crds.yaml` |
| ~18:19 | Set `crds.enabled: false` in main app's `values.yaml`, added sync-wave annotations |

## Root Cause Analysis

Kubernetes enforces a 256KB (262144 bytes) limit on `metadata.annotations` for any resource. The Prometheus Operator CRDs are extremely large (the `prometheuses.monitoring.coreos.com` CRD alone is ~290KB). When ArgoCD applies these CRDs using client-side apply, it stores the entire resource manifest in the `kubectl.kubernetes.io/last-applied-configuration` annotation. Since the CRDs themselves already approach the 256KB limit, adding them as an annotation pushes them over the threshold.

This is a well-known issue that has affected the kube-prometheus-stack community for years:

- **[prometheus-community/helm-charts#1500](https://github.com/prometheus-community/helm-charts/issues/1500)** — Can't upgrade CRD from 0.50.0 to 0.52.0: `metadata.annotations: Too long: must have at most 262144 bytes` (Nov 2021)
- **[prometheus-community/helm-charts#2364](https://github.com/prometheus-community/helm-charts/issues/2364)** — `prometheuses.monitoring.coreos.com` CRD is too long (Aug 2022)
- **[prometheus-community/helm-charts#3435](https://github.com/prometheus-community/helm-charts/issues/3435)** — PrometheusAgentList too big without server-side apply (May 2023)
- **[prometheus-community/helm-charts#2612](https://github.com/prometheus-community/helm-charts/issues/2612)** — New chart proposition: kube-prometheus-stack-crds (led to the separate CRDs chart approach)

The root cause chain:

1. Prometheus Operator CRDs are very large YAML manifests (290KB+)
2. Client-side apply stores the full manifest as a `kubectl.kubernetes.io/last-applied-configuration` annotation
3. The annotation size exceeds K8s's 262144-byte limit
4. CRDs fail to apply → dependent custom resources (Alertmanager, Prometheus) fail because their CRDs don't exist

## Resolution

Split the deployment into two ArgoCD Applications using sync-waves:

### Phase 1: CRDs Application (`kube-prometheus-stack-crds`, sync-wave 0)

A dedicated Application that installs only the CRDs with `ServerSideApply=true`. Server-side apply avoids the annotation size limit because it doesn't store the full manifest in annotations — instead it uses managed fields.

**File:** `apps/kube-prometheus-stack-crds.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 85.0.2
      helm:
        valueFiles:
          - $values/values/kube-prometheus-stack/crds.yaml
    - repoURL: https://github.com/aayush0325/homelab-k3s.git
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**File:** `values/kube-prometheus-stack/crds.yaml`
```yaml
crds:
  enabled: true

prometheusOperator:
  enabled: false
alertmanager:
  enabled: false
grafana:
  enabled: false
kubeApiServer:
  enabled: false
kubelet:
  enabled: false
kubeControllerManager:
  enabled: false
coreDns:
  enabled: false
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
kubeProxy:
  enabled: false
kubeStateMetrics:
  enabled: false
nodeExporter:
  enabled: false
prometheus:
  enabled: false
prometheusAgent:
  enabled: false
thanosRuler:
  enabled: false
additionalPrometheusRulesMap: {}
defaultRules:
  create: false
```

### Phase 2: Main Application (`kube-prometheus-stack`, sync-wave 1)

The main chart deployment with CRDs disabled, ensuring it only deploys after the CRDs exist.

**File:** `apps/kube-prometheus-stack.yaml`
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

**File:** `values/kube-prometheus-stack/values.yaml`
```yaml
crds:
  enabled: false   # CRDs managed by kube-prometheus-stack-crds app
```

## Post-Fix: Prometheus Operator Stall After Sync

After the CRDs and main app both synced successfully, ArgoCD showed the `kube-prometheus-stack` app stuck in `Progressing` / `Syncing` state with the message:

```
waiting for healthy state of monitoring.coreos.com/Prometheus/kube-prometheus-stack-prometheus
```

**Symptoms:**
- Prometheus CR existed but had no status (`.status` was empty)
- Alertmanager CR existed but had no status
- No `prometheus-kube-prometheus-stack-prometheus-0` or `alertmanager-kube-prometheus-stack-alertmanager-0` pods were created
- All other components (Grafana, kube-state-metrics, node-exporter) were healthy and running
- Operator logs showed `TLS handshake error from ... remote error: tls: bad certificate` around startup time

**Root Cause:**

The Prometheus Operator pod was hit by a burst of TLS errors from the K8s API server trying to reach its admission webhook immediately after startup — before the operator had finished initializing its internal certificate watch. This caused the operator to miss the initial reconciliation of the Prometheus and Alertmanager CRs. Once the TLS errors subsided, the operator entered a steady state but never re-processed the CRs because it had already marked them as "observed" in its informer cache.

**Diagnosis commands:**
```bash
# Check if Prometheus/Alertmanager pods exist
kubectl get pods -n monitoring
# → Only grafana, operator, node-exporter, kube-state-metrics were running

# Check if CRs have status
kubectl get prometheus -n monitoring
# → DESIRED: 1, READY: 0, RECONCILED: blank

# Check operator logs for TLS errors
kubectl logs -n monitoring kube-prometheus-stack-operator-<pod> --tail=50
# → Repeated "TLS handshake error ... tls: bad certificate"

# Check if operator is reconciling (it wasn't - no reconcile logs)
kubectl logs -n monitoring kube-prometheus-stack-operator-<pod> | grep -iE 'reconcil|sync'
# → Empty
```

**Fix:**

Restarting the operator deployment forced it to re-establish watches and reconcile all CRs from scratch:

```bash
kubectl rollout restart deployment kube-prometheus-stack-operator -n monitoring
```

Within seconds, the operator created the StatefulSets for both Prometheus and Alertmanager, and both pods started initializing:

```
alertmanager-kube-prometheus-stack-alertmanager-0   0/2   PodInitializing
prometheus-kube-prometheus-stack-prometheus-0        0/2   PodInitializing
```

**Timeline:**
| Time (IST) | Event |
|---|---|
| ~18:27 | Main app sync completed, all resources "Synced" |
| ~18:27-18:38 | App stuck in "Progressing" — Prometheus/Alertmanager pods never created |
| ~18:38 | Diagnosed: operator TLS errors during startup, CRs not reconciled |
| ~18:38 | Ran `kubectl rollout restart deployment kube-prometheus-stack-operator -n monitoring` |
| ~18:39 | Prometheus and Alertmanager StatefulSets created, pods initializing |

## Lessons Learned

1. **ServerSideApply alone is not sufficient.** While SSA avoids the annotation size issue, ArgoCD only applies SSA on a per-resource basis. The first sync attempt still tried client-side apply for some resources before SSA kicked in. A dedicated CRDs app with SSA ensures CRDs are always applied server-side.

2. **Split CRD-heavy Helm charts into separate applications.** For charts with large CRDs (prometheus-operator, cert-manager, istio), deploy CRDs as a separate ArgoCD Application with `ServerSideApply=true` and a lower sync-wave number. This is a pattern recommended by both the ArgoCD and prometheus-community maintainers.

3. **Use ArgoCD sync-waves for ordering.** `argocd.argoproj.io/sync-wave: "0"` on the CRDs app and `"1"` on the main app ensures CRDs are installed before custom resources that depend on them.

4. **The `crds.enabled` flag in kube-prometheus-stack.** Since chart version 45+, the chart supports `crds.enabled: true/false` to toggle CRD installation separately from the main chart content. This makes the split deployment approach straightforward.

5. **ArgoCD multi-source for Helm + Git values.** Using the `sources` field with a `ref: values` git source allows referencing local values files from the same repo while deploying from an external Helm chart repo.

6. **Prometheus Operator can stall after initial deployment.** If the operator experiences TLS webhook errors during startup, it may fail to reconcile Prometheus/Alertmanager CRs. The CRs will exist with empty `.status` and no pods will be created. A simple operator pod restart triggers full reconciliation and resolves the stall.

7. **Check `.status` on CRs, not just `kubectl get pods`.** When Prometheus/Alertmanager pods are missing, checking the CR's status field (empty vs populated) immediately reveals whether the operator has reconciled it. No status = operator hasn't processed it.

8. **TLS handshake errors in operator logs are normal during startup** — the K8s API server probes the webhook endpoint before the operator is ready. However, if these errors coincide with the initial CR creation window, the operator may silently skip reconciliation.

## Action Items

- [x] Create `apps/kube-prometheus-stack-crds.yaml` with sync-wave 0 and SSA
- [x] Create `values/kube-prometheus-stack/crds.yaml` enabling only CRDs
- [x] Set `crds.enabled: false` in `values/kube-prometheus-stack/values.yaml`
- [x] Add sync-wave 1 annotation to `apps/kube-prometheus-stack.yaml`
- [x] Add `ServerSideApply=true` to both Applications
- [ ] Verify both Applications sync successfully after push
- [ ] Monitor for any residual SharedResourceWarning between the two apps
- [x] Diagnosed and resolved: Prometheus Operator stall after initial sync — restarted operator deployment to trigger CR reconciliation