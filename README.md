# Workload Identity Setup

## env vars

```shell
export PROJECT_ID=`gcloud config get-value project`
export CLUSTER_NAME=albatros
export NODE_SA=$CLUSTER_NAME-node-sa
export WID_SA=$CLUSTER_NAME-wid-sa
export ZONE=us-west1-a
export NAMESPACE=workload-ns
export SERVICEACCOUNT=workload-sa
export CLUSTER_VERSION=`gcloud container get-server-config --zone $ZONE --format="value(validMasterVersions[0])"`
```

## create proper gke SAs

```shell
gcloud iam service-accounts create $NODE_SA --display-name "Node Service Account for $CLUSTER_NAME" --project $PROJECT_ID \
&& sleep 10 && \
export NODE_SA_ID=`gcloud iam service-accounts list --format='value(email)' --filter='displayName:Node Service Account for '"$CLUSTER_NAME"''`

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:${NODE_SA_ID} \
--role=roles/storage.objectViewer

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:${NODE_SA_ID} \
--role=roles/monitoring.metricWriter

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:${NODE_SA_ID} \
--role=roles/monitoring.viewer

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:${NODE_SA_ID} \
--role=roles/logging.logWriter
```

## create workload id sa

```shell
gcloud iam service-accounts create $WID_SA --display-name "Workload Identity Account for $CLUSTER_NAME" --project $PROJECT_ID \
&& sleep 10 && \
export WID_SA_ID=`gcloud iam service-accounts list --format='value(email)' --filter='displayName:Workload Identity Account for '"$CLUSTER_NAME"''`
```

## create cluster then resize to 0

```shell
gcloud beta container --project $PROJECT_ID clusters create $CLUSTER_NAME \
 --zone $ZONE \
 --machine-type "n1-standard-4" \
 --image-type "COS" \
 --disk-type "pd-standard" \
 --disk-size "100" \
 --scopes "storage-ro","logging-write","monitoring","service-control","service-management","trace" \
 --min-nodes "0" \
 --num-nodes "1"\
 --max-nodes "2" \
 --max-surge-upgrade 1 \
 --max-unavailable-upgrade 0 \
 --default-max-pods-per-node "60" \
 --enable-autoupgrade \
 --enable-autoscaling \
 --enable-autorepair \
 --enable-stackdriver-kubernetes \
 --enable-ip-alias \
 --workload-pool=$PROJECT_ID.svc.id.goog \
 --no-enable-basic-auth \
 --metadata disable-legacy-endpoints=true \
 --addons HorizontalPodAutoscaling,HttpLoadBalancing,ConfigConnector \
 --cluster-version $CLUSTER_VERSION
```

## setup workloadIdentity and bind to sa

```shell
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID

kubectl create namespace $NAMESPACE
kubectl create serviceaccount $SERVICEACCOUNT \
 --namespace $NAMESPACE

gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$SERVICEACCOUNT]" \
  $WID_SA_ID

kubectl annotate serviceaccount $SERVICEACCOUNT \
   --namespace $NAMESPACE \
   iam.gke.io/gcp-service-account=$WID_SA_ID
```

## test workload

Test workload with 2 prominent methods, either run adhoc pod or create a workload that goes on a node-pool

### adhoc pod

create adhoc pod

```shell
kubectl run -it \
  --generator=run-pod/v1 \
  --image google/cloud-sdk \
  --serviceaccount $SERVICEACCOUNT \
  --namespace $NAMESPACE \
  workload-identity-test
```

verify by curling metadata

```shell
curl -H 'Metadata-Flavor:Google' http://169.254.169.254/computeMetadata/v1/instance/id --keepalive-time 60
```

or verify by listing scopes

```shell
gcloud auth list
```

### nodepool and dedicated workload

first create a nodepool

#### create 

```shell
gcloud beta container node-pools create new-$CLUSTER_NAME --cluster $CLUSTER_NAME \
 --zone $ZONE \
 --machine-type "n1-standard-4" \
 --image-type "COS" \
 --disk-type "pd-standard" \
 --disk-size "100" \
 --scopes "storage-ro","logging-write","monitoring","service-control","service-management","trace" \
 --min-nodes "0" \
 --num-nodes "0"\
 --max-nodes "1" \
 --enable-autoupgrade \
 --enable-autoscaling \
 --enable-autorepair \
 --max-pods-per-node "12" \
 --workload-metadata-from-node=GKE_METADATA_SERVER
```

#### prep gcs

prep a bucket that pod will read or write from

```shell
gsutil mb gs://$PROJECT_ID/

echo 'stuff' > test.txt
gsutil cp test.txt gs://$PROJECT_ID/
```

#### create workload and test

pin workload to nodepool

```shell
cat << EOF | kubectl apply --namespace $NAMESPACE -f -
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    run: cloud-sdk
  name: cloud-sdk 
spec:
  replicas: 1
  selector:
    matchLabels:
      run: cloud-sdk
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: cloud-sdk
    spec:
      serviceAccountName: $NAMESPACE-sa
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cloud.google.com/gke-nodepool
                operator: In
                values:
                - new-$CLUSTER_NAME
      containers:
      - image: google/cloud-sdk
        command: [ "sleep" ]
        args: [ "infinity" ]  
        name: cloud-sdk
        resources: {}
EOF

POD_NAME=`kubectl get pods -l run=cloud-sdk  --namespace $NAMESPACE -o jsonpath='{.items[0].metadata.name}'`
kubectl --namespace $NAMESPACE exec -it $POD_NAME -c cloud-sdk -- /bin/bash
```

```shell
curl "http://metadata.google.internal/computeMetadata/v1/instance/id" -H "Metadata-Flavor: Google"
```

## CLEANUP

```shell
gcloud container clusters resize $CLUSTER_NAME \
--node-pool new-$CLUSTER_NAME \
--zone $ZONE \
--num-nodes "0"

kubectl scale deploy/cloud-sdk --replicas=0 --namespace $NAMESPACE


kubectl delete deploy/cloud-sdk --namespace $NAMESPACE
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE
gcloud iam service-accounts delete $NODE_SA_ID
```
