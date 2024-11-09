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

