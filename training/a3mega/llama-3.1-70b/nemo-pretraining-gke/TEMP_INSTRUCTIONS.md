# Testing recipes with Parallelstore

## Get access to a cluster provisioned through ClusterToolkit

### Configure environment settings

```
IP_ADDRESS=$(curl -sS ifconfig.me)
CLUSTER_ID=a3plus-benchmark-toolkit
REGION=us-central1
```

### Get cluster credentials

```
gcloud container clusters get-credentials $CLUSTER_ID \
    --region $REGION \
    --project supercomputer-testing
```

### Add authorized networks

```
gcloud container clusters update $CLUSTER_ID \
    --region $REGION \
    --project supercomputer-testing \
    --enable-master-authorized-networks \
    --master-authorized-networks $IP_ADDRESS/32

```

### Manage CSI driver

```
gcloud container clusters update $CLUSTER_ID \
  --location=$REGION \
  --update-addons=ParallelstoreCsiDriver=ENABLED
```

```
gcloud container clusters update $CLUSTER_ID \
    --location=$REGION \
    --update-addons=ParallelstoreCsiDriver=DISABLED
```

## Create a Parallelstore instance

### Configure the VPC

#### Create an IP range

```
IP_RANGE_NAME=jarekk-ps-range
NETWORK_NAME=a3plus-benchmark-toolkit-net

gcloud compute addresses create $IP_RANGE_NAME \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=24 \
  --description="Parallelstore VPC Peering" \
  --network=$NETWORK_NAME
```

#### Get the CIDR range associated with the range you created in the previoius step

```
CIDR_RANGE=$(
  gcloud compute addresses describe $IP_RANGE_NAME \
    --global  \
    --format="value[separator=/](address, prefixLength)"
)
```

#### Create a firewall rule

```
FIREWALL_NAME=jarekk-ps-firewall-rule

gcloud compute firewall-rules create $FIREWALL_NAME \
  --allow=tcp \
  --network=$NETWORK_NAME \
  --source-ranges=$CIDR_RANGE
```

#### Connect the peering

```
gcloud services vpc-peerings connect \
  --network=$NETWORK_NAME \
  --ranges=$IP_RANGE_NAME \
  --service=servicenetworking.googleapis.com
```

### Create an instance

```
export INSTANCE_ID=jarekk-ps-1
export PROJECT_ID=supercomputing-testing
export LOCATION=us-central1-c
export CAPACITY_GIB=20000
export DIRECTORY_STRIPE_LEVEL=directory-stripe-level-balanced
export FILE_STRIPE_LEVEL=file-stripe-level-max


gcloud beta parallelstore instances create $INSTANCE_ID \
  --capacity-gib=$CAPACITY_GIB \
  --location=$LOCATION \
  --network=$NETWORK_NAME \
  --project=$PROJECT_ID \
  --directory-stripe-level=$DIRECTORY_STRIPE_LEVEL \
  --file-stripe-level=$FILE_STRIPE_LEVEL
```


## Create a bucket with hierarchical namespace

```
BUCKET_NAME=jarekk-us-central1-checkpoints
BUCKET_LOCATION=us-central1

gcloud storage buckets create gs://$BUCKET_NAME \
--location=$BUCKET_LOCATION \
--uniform-bucket-level-access \
--enable-hierarchical-namespace
```

## Run a workload

```
export PROJECT_ID=supercomputing-testing
export REGION=us-central1
export CLUSTER_REGION=us-central1
export CLUSTER_NAME=a3plus-benchmark-toolkit
export GCS_BUCKET=jarekk-us-central1
export ARTIFACT_REGISTRY=us-east4-docker.pkg.dev/supercomputer-testing/jarekk-recipes-dev

export REPO_ROOT=`git rev-parse --show-toplevel`
export RECIPE_ROOT=$REPO_ROOT/training/a3mega/llama-3.1-70b/nemo-pretraining-gke


```


```
helm install -f values.yaml \
    --set-file nemo_config=nemo_config.yaml \
    --set workload.image=${ARTIFACT_REGISTRY}/nemo_workload:24.07 \
    --set workload.gcsBucketForDataCataPath=${GCS_BUCKET} \
    $USER-llama-3-1-70b-checkpointing-nemo \
    $REPO_ROOT/src/helm-charts/nemo-training
```

```
helm uninstall $USER-llama-3-1-70b-checkpointing-nemo 
```