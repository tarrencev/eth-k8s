# eth-k8s

Kubernetes manifests for deploying and syncing an eth1 (openetehreum) and eth2 (prysm) node to Google Kubernetes Engine. The full sync can be done in ~2days within the $300 of GCP credits (aka for free).

## Quickstart

Before getting started, be sure to setup a GCP account and the [gcloud cli](https://cloud.google.com/sdk/gcloud).

### Create cluster

First we need to create a kubernetes cluster. We're setting it up with Config Connector which will allow us to manage GCP resources using kubernetes manifests.

```
gcloud container clusters create prod \
    --release-channel rapid \
    --addons ConfigConnector \
    --workload-pool=fourth-silo-311012.svc.id.goog \
    --enable-stackdriver-kubernetes
```

### Enable + Conigure config connector

Setup for [config connector](https://cloud.google.com/config-connector/docs/overview). We create a service account and kubernetes namespace.

```bash
# Config connector service account setup.
gcloud iam service-accounts create config-connector
gcloud projects add-iam-policy-binding fourth-silo-311012 \
    --member="serviceAccount:config-connector@fourth-silo-311012.iam.gserviceaccount.com" \
    --role="roles/owner"
gcloud iam service-accounts add-iam-policy-binding \
    config-connector@fourth-silo-311012.iam.gserviceaccount.com \
    --member="serviceAccount:fourth-silo-311012.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
    --role="roles/iam.workloadIdentityUser"

# Config connector kubernetes namespace.
kubectl apply -f configconnector.yml
kubectl create namespace config-connector
kubectl annotate namespace \
    config-connector cnrm.cloud.google.com/project-id=fourth-silo-311012
```

### Create eth node pool

We're going to deploy a `n1-standard-4` autoscaling node pool.

```
kubectl apply -n config-connector -f eth-nodepool.yml
```

### Deploy openethereum

```
kubectl apply -f eth-openethereum.yml
```

At this point you'll need to wait for the sync to complete. You can watch the progess by tailing the logs:

```
kubectl get pods -o wide
kubectl logs -f openethereum-deployment-XXXXXX
```

Once sync is complete, we can setup the Prysm node.

### Deploy prysm

```
kubectl apply -f eth-prysmbeacon.yml
```
