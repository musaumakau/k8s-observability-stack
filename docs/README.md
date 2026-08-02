# k8s-observability-stack

Hands-on Kubernetes observability project on GKE, deployed GitOps-style via ArgoCD.

## Stack

- Metrics: Prometheus + Grafana + Alertmanager (kube-prometheus-stack)
- Logging: Loki + Grafana Alloy
- Tracing: OpenTelemetry Collector + Tempo
- SLOs/alerting: Sloth, multi-window burn-rate alerts
- Chaos validation: Chaos Mesh
- Cost visibility: OpenCost
- Sample workload: Online Boutique (multi-service demo app)

## Phases

1. Metrics foundation -- kube-prometheus-stack via ArgoCD
2. Sample workload -- deploy Online Boutique to generate real signal
3. Logging -- Loki + Alloy, correlated with metrics in Grafana
4. Tracing -- OTel Collector + Tempo, trace-to-logs/metrics correlation
5. SLOs & alerting -- Sloth-defined SLOs, Alertmanager routed to Slack/PagerDuty
6. Chaos & validation -- inject failure, prove detection end-to-end
7. Cost angle -- OpenCost per-namespace/pod cost visibility

## Bootstrap

1. Install ArgoCD on the cluster:

   ```bash
   kubectl create namespace argocd
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   helm install argocd argo/argo-cd \
     --namespace argocd \
     --set server.service.type=LoadBalancer \
     --version 7.7.11
   ```

2. Push this repo to `github.com/musaumakau/k8s-observability-stack`.

3. Apply the root app:

   ```bash
   kubectl apply -f bootstrap/app-of-apps.yaml
   ```

## Notes

- Cluster also runs Google Managed Prometheus (`gmp-system`/`gmp-public`).
  Left untouched intentionally -- different CRDs (`PodMonitoring` vs
  `ServiceMonitor`), no conflict, just don't wire it into Grafana.

## Known issues

- The `loki` StatefulSet shows persistent `OutOfSync` in ArgoCD due to
  a confirmed upstream bug where the Kubernetes API server injects
  `creationTimestamp: null` into `spec.volumeClaimTemplates[].metadata`
  on admission, and ArgoCD's diff engine fails to suppress it via any
  of the standard mechanisms -- see argoproj/argo-cd#24791, #16707,
  #11143.

  Tried and ruled out:
  - `ignoreDifferences` via `jsonPointers` (both wildcard and
    array-indexed paths)
  - `ignoreDifferences` via `jqPathExpressions`
  - Removing `ServerSideApply` from the sync options
  - Enabling `ServerSideDiff=true` via Application annotation
  - `managedFieldsManagers` targeting `kube-controller-manager`

  Verified cosmetic only: Loki is `Healthy`, all pods `Running`, and
  the full log pipeline (Alloy -> Loki -> Grafana) is confirmed
  working end-to-end. Fix would require either a Loki chart upgrade
  past `6.24.0` (unverified whether the upstream PR fixing this is
  merged) or an ArgoCD version bump past `2.13.2` -- deferred as
  out of scope for a cosmetic sync-status issue.