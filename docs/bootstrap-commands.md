---
title: Bootstrap Command Log
description: Commands executed to prepare gitops-cluster-1 for Config Connector and Flux, plus the remaining steps
author: Ravi Cheetirala
ms.date: 2026-09-28
ms.topic: reference
keywords:
  - GKE
  - Config Connector
  - Flux
estimated_reading_time: 6
---

## Environment

| Setting           | Value                                                               |
|-------------------|---------------------------------------------------------------------|
| Project           | `project-025ac1f2-75d5-497d-a57`                                    |
| Cluster           | `gitops-cluster-1` (Standard, regional)                             |
| Location          | `australia-southeast1`                                              |
| Node pool         | `default-pool`                                                      |
| Git repository    | `https://github.com/ravi-cheetiralaav/gcp-config-connector.git`     |
| Flux CLI          | `2.9.5`                                                             |

> [!NOTE]
> The agent terminal does not write to PowerShell history, so the executed commands
> below were reconstructed from the session and confirmed against the cluster's GKE
> operation log (`gcloud container operations list`).

Set these variables before running any command:

```powershell
$PROJECT_ID = "project-025ac1f2-75d5-497d-a57"
$CLUSTER_NAME = "gitops-cluster-1"
$CLUSTER_LOCATION = "australia-southeast1"
$CNRM_GSA_NAME = "cnrm-controller"
$CNRM_NAMESPACE = "config-connector"
$CNRM_GSA_EMAIL = "$CNRM_GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com"
```

## Executed commands

### 1. Inspect the repository, tooling, and cluster access

```powershell
git remote -v
git status --short
gcloud --version
kubectl version --client
flux --version
kubectl config current-context
kubectl get nodes
```

Result: three nodes ready; the Flux CLI was not yet installed.

### 2. Run Flux pre-checks

```powershell
flux check --pre
```

### 3. Inspect cluster features

```powershell
gcloud container clusters describe $CLUSTER_NAME `
  --location $CLUSTER_LOCATION `
  --project $PROJECT_ID `
  --format="yaml(autopilot,workloadIdentityConfig,addonsConfig.configConnectorConfig,nodePools[].name,nodePools[].config.workloadMetadataConfig)"
```

Result: Standard cluster with Workload Identity and the Config Connector add-on
both disabled.

### 4. Enable the required APIs

```powershell
gcloud services enable `
  container.googleapis.com `
  compute.googleapis.com `
  iam.googleapis.com `
  storage.googleapis.com `
  monitoring.googleapis.com `
  --project $PROJECT_ID
```

### 5. Enable Workload Identity Federation for GKE

GKE operation: `UPDATE_CLUSTER`, started `2026-09-28T01:13:34Z`, status `DONE`.

```powershell
gcloud container clusters update $CLUSTER_NAME `
  --location $CLUSTER_LOCATION `
  --project $PROJECT_ID `
  --workload-pool="$PROJECT_ID.svc.id.goog"
```

### 6. Enable the GKE metadata server on the node pool

GKE operation: `UPGRADE_NODES`, started `2026-09-28T01:22:43Z`, status `DONE`.
This operation recreates the nodes.

```powershell
gcloud container node-pools update default-pool `
  --cluster $CLUSTER_NAME `
  --location $CLUSTER_LOCATION `
  --project $PROJECT_ID `
  --workload-metadata=GKE_METADATA
```

### 7. Enable the Config Connector add-on

GKE operation: `UPDATE_CLUSTER`, started `2026-09-28T01:37:53Z`, status `DONE`.
This created the `configconnector-operator-system` and `cnrm-system` namespaces.

```powershell
gcloud container clusters update $CLUSTER_NAME `
  --location $CLUSTER_LOCATION `
  --project $PROJECT_ID `
  --update-addons ConfigConnector=ENABLED
```

### 8. Verify the current state

```powershell
gcloud container clusters describe $CLUSTER_NAME `
  --region $CLUSTER_LOCATION `
  --project $PROJECT_ID `
  --format="yaml(workloadIdentityConfig,addonsConfig.configConnectorConfig,autopilot.enabled)"

gcloud container node-pools list `
  --cluster $CLUSTER_NAME `
  --region $CLUSTER_LOCATION `
  --project $PROJECT_ID `
  --format="table(name,config.workloadMetadataConfig.mode,status)"

gcloud container operations list `
  --region $CLUSTER_LOCATION `
  --project $PROJECT_ID `
  --sort-by=~startTime `
  --limit=6

gcloud iam service-accounts list --project $PROJECT_ID --format="value(email)"

kubectl get ns flux-system configconnector-operator-system cnrm-system config-connector --ignore-not-found
```

Confirmed state:

* `workloadPool: project-025ac1f2-75d5-497d-a57.svc.id.goog`
* `default-pool` metadata mode `GKE_METADATA`
* `configConnectorConfig.enabled: true`
* `configconnector-operator-system` and `cnrm-system` namespaces exist
* No `cnrm-controller` service account exists yet
* Flux is not installed yet (no `flux-system` namespace)

## Remaining commands

### 9. Create the Config Connector identity

```powershell
gcloud iam service-accounts create $CNRM_GSA_NAME `
  --project $PROJECT_ID `
  --display-name="Config Connector experiment controller"

$CNRM_ROLES = @(
  "roles/compute.instanceAdmin.v1",
  "roles/compute.networkAdmin",
  "roles/storage.admin",
  "roles/monitoring.metricWriter"
)

foreach ($ROLE in $CNRM_ROLES) {
  gcloud projects add-iam-policy-binding $PROJECT_ID `
    --member="serviceAccount:$CNRM_GSA_EMAIL" `
    --role=$ROLE `
    --condition=None
}

gcloud iam service-accounts add-iam-policy-binding $CNRM_GSA_EMAIL `
  --project $PROJECT_ID `
  --member="serviceAccount:$PROJECT_ID.svc.id.goog[cnrm-system/cnrm-controller-manager-$CNRM_NAMESPACE]" `
  --role="roles/iam.workloadIdentityUser"
```

### 10. Set the project ID in the Flux manifest

```powershell
$FLUX_FILE = ".\flux\config-connector.yaml"
$CONTENT = (Get-Content $FLUX_FILE -Raw).Replace("REPLACE_WITH_PROJECT_ID", $PROJECT_ID)
$CONTENT = $CONTENT.Replace("us-central1-a", "australia-southeast1-a").Replace("us-central1", "australia-southeast1")
Set-Content -Path $FLUX_FILE -Value $CONTENT -Encoding utf8
```

### 11. Bootstrap Flux against the repository

`flux bootstrap github` reads the PAT from the `GITHUB_TOKEN` environment variable.
The token is never passed on the command line.

```powershell
flux bootstrap github `
  --owner=ravi-cheetiralaav `
  --repository=gcp-config-connector `
  --branch=main `
  --path=clusters/gitops-cluster-1 `
  --personal
```

### 12. Register the Config Connector reconciliations

Place the Flux objects under the bootstrap path so Flux manages them from Git:

```powershell
Copy-Item .\flux\config-connector.yaml .\clusters\gitops-cluster-1\config-connector.yaml
git pull
git add .
git commit -m "feat(flux): add Config Connector reconciliations for gitops-cluster-1"
git push
flux reconcile kustomization flux-system --with-source
```

### 13. Verify

```powershell
flux get kustomizations --watch
kubectl get configconnectorcontext -n $CNRM_NAMESPACE
kubectl get pods -n cnrm-system -l cnrm.cloud.google.com/scoped-namespace=$CNRM_NAMESPACE
kubectl get gcp -n $CNRM_NAMESPACE
```
