# Upgrade Guide

This guide explains how to upgrade the FortiADC Kubernetes Controller Helm chart across versions, with special attention to the Custom Resource Definition (CRD) handling that a plain `helm upgrade` does not cover.

## How CRDs are packaged in the chart

The FortiADC Kubernetes Controller chart places CRDs in two different locations, and Helm treats them differently:

| Location | Contents | `helm install` | `helm upgrade` | `helm uninstall` |
|----------|----------|----------------|----------------|------------------|
| `templates/*_crd.yaml` | Fortinet-defined CRDs: `VirtualServer`, `RemoteServer`, `Host`, `FortiADCGatewayParameter`, `FortiADCVirtualServerPolicy` | Create | **Update** | Delete |
| `crds/*.yaml` | Upstream Kubernetes Gateway API CRDs: `GatewayClass`, `Gateway`, `HTTPRoute` | Create (only if absent) | **No update, no delete** | Delete (blocked if instances exist) |

This split is the Helm convention for CRDs:

- **Fortinet-defined CRDs** live under `templates/` because their schema evolves with each chart release (for example, `VirtualServer` moved from `v1alpha1` to `v1alpha2`). The controller is the only consumer of these CRDs, and instances can be re-reconciled after a schema change, so letting `helm upgrade` apply the new schema is safe and desirable.
- **Upstream Gateway API CRDs** live under `crds/` because they are a shared standard. A cluster may run multiple Gateway API controllers at the same time (for example, the FortiADC controller alongside Istio or Envoy Gateway). Letting `helm upgrade` overwrite or `helm uninstall` delete them would risk wiping out `Gateway` / `HTTPRoute` instances owned by other controllers. Helm therefore only creates them on `helm install` and never touches them on `helm upgrade`.

This means that whenever a chart release introduces a *new* CRD under `crds/`, or updates the schema of an existing one, you must apply it with `kubectl` yourself before upgrading the chart.

## Prerequisites

Before any upgrade:

1. Make sure `cert-manager` is installed (required since 3.1.0 for the admission webhook). See the [main README](../README.md#install-cert-managerio) for installation steps.
2. Make sure the namespace where the controller runs exists. The examples below use `fortiadc-ingress`.
3. Review the [Release Notes](../Release-Notes.md) for breaking changes between your current version and the target version.

## Upgrade from 3.1.x to 3.2.x

3.1.x charts do not contain the Gateway API CRDs at all — the `crds/` directory was introduced in 3.2.0. If you run `helm upgrade` directly, the chart will install the new controller binary, but the Gateway API CRDs (`GatewayClass`, `Gateway`, `HTTPRoute`) will be missing from the cluster. The controller pod will fail to register its Gateway API informers on startup and crash-loop until the CRDs are present.

### Step 1 — Install the Gateway API CRDs

Apply the three CRDs shipped with the 3.2.x chart. The commands below fetch them directly from this repository; if you have the chart extracted locally, you can `kubectl apply -f charts/fadc-k8s-ctrl-3.2.0/crds/` instead.

    kubectl apply -f https://raw.githubusercontent.com/fortinet/fortiadc-kubernetes-controller/main/charts/fadc-k8s-ctrl-3.2.0/crds/gatewayclass.yaml
    kubectl apply -f https://raw.githubusercontent.com/fortinet/fortiadc-kubernetes-controller/main/charts/fadc-k8s-ctrl-3.2.0/crds/gateway.yaml
    kubectl apply -f https://raw.githubusercontent.com/fortinet/fortiadc-kubernetes-controller/main/charts/fadc-k8s-ctrl-3.2.0/crds/httproute.yaml

### Step 2 — Wait for the CRDs to be established

    kubectl wait --for=condition=Established crd/gatewayclasses.gateway.networking.k8s.io \
                                        crd/gateways.gateway.networking.k8s.io \
                                        crd/httproutes.gateway.networking.k8s.io \
                                        --timeout=120s

### Step 3 — Upgrade the Helm chart

    helm repo update
    helm upgrade --debug --reset-values -n fortiadc-ingress first-release \
        fortiadc-kubernetes-controller/fadc-k8s-ctrl

### Step 4 — Verify

    helm history -n fortiadc-ingress first-release
    kubectl get -n fortiadc-ingress deployments
    kubectl get -n fortiadc-ingress pods
    kubectl logs -n fortiadc-ingress -f <pod name>

The controller pod should reach `Running` and its logs should not contain `failed to list` or `no matches for kind` errors related to `Gateway`, `GatewayClass`, or `HTTPRoute`. On a successful start the log should contain:

    Gateway API CRDs detected, enabling Gateway support
    Gateway API support enabled

> [!NOTE]
> If you already manage the Gateway API CRDs through another channel (for example, the upstream `gateway-api` Helm chart or another Gateway API controller's installation), `kubectl apply -f` is idempotent — it only updates the CRD schema and never deletes existing `Gateway` / `HTTPRoute` instances. Sharing one set of Gateway API CRDs across multiple controllers is supported.

> [!WARNING]
> Gateway API support is detected **once at controller startup**. The controller probes the API server for the `gateway.networking.k8s.io` API group when the process boots and, if the group is absent, it skips creating the Gateway/GatewayClass/HTTPRoute informers entirely and logs `Gateway API CRDs not found in cluster, disabling Gateway support`. Applying the CRDs to a cluster where the controller is already running will **not** turn Gateway support on — the running process never re-checks. If you see the "disabling Gateway support" message in a running controller, restart the pod after the CRDs are established:
>
>     kubectl rollout restart -n fortiadc-ingress deployment first-release-fadc-k8s-ctrl
>
> The restarted pod will detect the CRDs and create the Gateway API informers on boot.

## Upgrade between 3.2.x patch releases

Patch releases within the 3.2.x line may update the Gateway API CRD schemas under `crds/`. `helm upgrade` will not apply those updates, so re-run the `kubectl apply -f crds/` step before upgrading:

    # From a local checkout of the chart
    kubectl apply -f charts/fadc-k8s-ctrl-<target-version>/crds/

    # Or fetch from the repository
    kubectl apply -f https://raw.githubusercontent.com/fortinet/fortiadc-kubernetes-controller/main/charts/fadc-k8s-ctrl-<target-version>/crds/gatewayclass.yaml
    kubectl apply -f https://raw.githubusercontent.com/fortinet/fortiadc-kubernetes-controller/main/charts/fadc-k8s-ctrl-<target-version>/crds/gateway.yaml
    kubectl apply -f https://raw.githubusercontent.com/fortinet/fortiadc-kubernetes-controller/main/charts/fadc-k8s-ctrl-<target-version>/crds/httproute.yaml

    helm repo update
    helm upgrade --debug --reset-values -n fortiadc-ingress first-release \
        fortiadc-kubernetes-controller/fadc-k8s-ctrl

The Fortinet-defined CRDs under `templates/` are updated automatically by `helm upgrade`, so they do not require a separate `kubectl apply` step.

## Why not move the Gateway API CRDs into `templates/`?

Moving the Gateway API CRDs into `templates/` would make `helm upgrade` apply them automatically, but it has two serious drawbacks:

1. **`helm uninstall` would delete the CRDs.** When Helm deletes a CRD, Kubernetes cascades the deletion to every instance of that CRD — every `Gateway` and `HTTPRoute` in the cluster would be destroyed, including ones managed by other Gateway API controllers. This is unacceptable for a shared CRD.
2. **`helm upgrade --force` would recreate them on every forced upgrade**, which carries the same risk as `uninstall` + `install`.

Keeping the Gateway API CRDs under `crds/` and requiring a manual `kubectl apply` is the safer, Helm-recommended tradeoff: the user is in control of when and how the shared CRDs are updated.

## Rollback

To roll back to a previous chart release:

    helm rollback -n fortiadc-ingress first-release <REVISION_NUMBER>

    helm history -n fortiadc-ingress first-release   # list revisions

Rolling back the chart reverts the controller binary, the Deployment, RBAC, and the Fortinet-defined CRDs under `templates/`. It does **not** downgrade the Gateway API CRDs under `crds/` — those are no longer touched by Helm after the initial install. If you need to revert a Gateway API CRD schema change, use `kubectl apply -f crds/` with the older chart's CRD files.

## Quick reference

| Scenario | Command sequence |
|----------|------------------|
| Fresh install | `helm install ...` (Helm creates both `templates/` and `crds/` CRDs) |
| Upgrade within 3.2.x (no CRD schema change) | `helm repo update && helm upgrade ...` |
| Upgrade within 3.2.x (CRD schema changed) | `kubectl apply -f crds/` → `helm upgrade ...` |
| Upgrade from 3.1.x to 3.2.x | `kubectl apply -f crds/` → wait for `Established` → `helm upgrade ...` |
| Rollback chart | `helm rollback ...` (CRDs under `crds/` are not reverted) |
| Uninstall | `helm uninstall ...` (Fortinet CRDs under `templates/` are deleted; Gateway API CRDs under `crds/` are deleted only if no `Gateway`/`HTTPRoute` instances remain) |
