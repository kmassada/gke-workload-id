# Workload Identity Setup

## env vars

```shell
export PROJECT_ID=`gcloud config get-value project`
export CLUSTER_NAME=test-this
export NODE_SA=$CLUSTER_NAME-node-sa
export ZONE=us-west1-a
export NAMESPACE=that
export CLUSTER_VERSION="1.13.11-gke.5"
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
 --default-max-pods-per-node "12" \
 --enable-autoupgrade \
 --enable-autoscaling \
 --enable-autorepair \
 --enable-stackdriver-kubernetes \
 --enable-ip-alias \
 --identity-namespace=$PROJECT_ID.svc.id.goog \
 --no-enable-basic-auth \
 --metadata disable-legacy-endpoints=true \
 --addons HorizontalPodAutoscaling,HttpLoadBalancing \
 --workload-metadata-from-node=GKE_METADATA_SERVER \
 --cluster-version $CLUSTER_VERSION
```

## setup workloadIdentity and bind to sa

```shell
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT_ID

kubectl create namespace $NAMESPACE-ns
kubectl create serviceaccount $NAMESPACE-sa \
 --namespace $NAMESPACE-ns

gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE-ns/$NAMESPACE-sa]" \
  $NODE_SA_ID

kubectl annotate serviceaccount $NAMESPACE-sa \
   --namespace $NAMESPACE-ns \
   iam.gke.io/gcp-service-account=$NODE_SA_ID
```

## prep gcs

```shell
gsutil mb gs://$PROJECT_ID/

echo 'stuff' > test.txt
gsutil cp test.txt gs://$PROJECT_ID/
```

## create workload and test

```shell
cat << EOF | kubectl apply --namespace $NAMESPACE-ns -f -
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

POD_NAME=`kubectl get pods -l run=cloud-sdk  --namespace $NAMESPACE-ns -o jsonpath='{.items[0].metadata.name}'`
kubectl --namespace $NAMESPACE-ns exec -it $POD_NAME -c cloud-sdk -- /bin/bash
```

```shell
curl "http://metadata.google.internal/computeMetadata/v1/instance/disks/0/" -H "Metadata-Flavor: Google"
```

## CLEANUP

```shell
gcloud container clusters resize $CLUSTER_NAME \
--node-pool new-$CLUSTER_NAME \
--zone $ZONE \
--num-nodes "0"

kubectl delete deploy/cloud-sdk --namespace $NAMESPACE-ns
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE
gcloud iam service-accounts delete $NODE_SA_ID
```
