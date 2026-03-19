# acm_hcp_argocd_demo
This demo will show how to manage OpenShift cluster using ACM and Hosted Control plane with ArgoCD.

## Repository structure

```
acm_hcp_argocd_demo/
├── hcp/                              # HyperShift manifests (Kustomize target)
│   ├── kustomization.yaml
│   ├── hosted_cluster.yaml           # HostedCluster resource
│   └── nodePool.yaml                 # NodePool resource
└── argocd/                           # ACM GitOps + ArgoCD artifacts
    ├── kustomization.yaml
    ├── acm-placement-configmap.yaml  # Teaches ArgoCD how to read ACM PlacementDecisions
    ├── managedclustersetbinding.yaml # Binds ClusterSet to openshift-gitops namespace
    ├── placement.yaml                # ACM Placement selecting the hub (local-cluster)
    ├── gitopscluster.yaml            # Registers clusters with ArgoCD via ACM
    ├── appproject.yaml               # ArgoCD AppProject scoping HyperShift resources
    └── applicationset.yaml           # ApplicationSet deploying hcp/ via clusterDecisionResource
```

## Deployment order

> **Prerequisites:** ACM and the OpenShift GitOps (ArgoCD) operators must be installed on the hub cluster before applying these resources.

### Step 1 — Apply the ACM GitOps + ArgoCD artifacts

This registers the hub cluster with ArgoCD and creates the ApplicationSet that will watch the `hcp/` directory.

```bash
oc apply -k argocd/
```

Resources are applied in the following order by the kustomization:

1. `acm-placement-configmap.yaml` — ConfigMap (`acm-placement`) that teaches the ArgoCD `clusterDecisionResource` generator the API shape of ACM `PlacementDecision` objects
2. `managedclustersetbinding.yaml` — binds the `default` ManagedClusterSet into the `openshift-gitops` namespace
3. `placement.yaml` — ACM Placement that selects the hub cluster (`local-cluster: "true"`)
4. `gitopscluster.yaml` — ACM GitOpsCluster that registers the selected clusters with the ArgoCD instance
5. `appproject.yaml` — ArgoCD AppProject (`hcp-project`) whitelisting HyperShift resource kinds
6. `applicationset.yaml` — ArgoCD ApplicationSet that generates an Application from the placement and syncs `hcp/`

### Step 2 — ArgoCD automatically reconciles the HCP manifests

Once the ApplicationSet is running, ArgoCD detects the placement result and automatically creates an `Application` that deploys the contents of `hcp/` (the `HostedCluster` and `NodePool` resources) to the hub cluster in the `local-cluster` namespace.

To manually verify or trigger the sync:

```bash
oc apply -k hcp/
```
