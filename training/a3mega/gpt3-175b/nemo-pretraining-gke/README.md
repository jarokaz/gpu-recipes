# Pretrain GPT3-175B workloads on A3 Mega GKE Node pools

This recipe outlines the steps for running a GPT-3 175B pretraining workload on
[A3 Mega GKE Node pools](https://cloud.google.com/kubernetes-engine) by using the
[NVIDIA NeMo framework](https://github.com/NVIDIA/nemo).

## Orchestration and deployment tools

For this recipe, the following setup is used:

- Orchestration - [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine)
- Job configuration and deployment - Helm chart is used to configure and deploy the
  [Kubernetes Index Job](https://kubernetes.io/blog/2021/04/19/introducing-indexed-jobs).
  This job encapsulates the
  [NVIDIA NeMo Megatron GPT pretraining workload](https://github.com/NVIDIA/NeMo/blob/main/examples/nlp/language_modeling/megatron_gpt_pretraining.py).
  The chart generates the job's manifest, adhering to best practices for using GPUDirect-TCPXO
  with Google Kubernetes Engine (GKE), which includes setting optimal values for NVIDIA NCCL
  and the TCPXO NCCL plugin.

## Test environment

This recipe has been optimized for and tested with the following configuration:

- A cluster with 32 [a3-megagpu-8g](https://cloud.google.com/compute/docs/accelerator-optimized-machines#a3-mega-vms) machines
- Machine placement in the cluster is configured using a [compact placement policy](https://cloud.google.com/kubernetes-engine/docs/how-to/compact-placement)
- [GPUDirect-TCPXO](https://cloud.google.com/kubernetes-engine/docs/how-to/gpu-bandwidth-gpudirect-tcpx#required-features-capabilities)
   component versions:
    - NCCL Plugin: v1.0.3
    - RxDM sidecar: v1.0.9
- [NVIDIA NeMo NGC container image](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/nemo/tags): 24.07
- FP8 precision training
- Uses a mock pretraining dataset provided by the NeMo framework. By default, the job
  is configured to execute 50 training steps. If you want to change the number of training steps,
  see [Configure and submit a pretraining job](#configure-and-submit-a-pretraining-job).


## Prerequisites

Before running this recipe, ensure your environment is configured as follows:

- A GKE cluster with the following setup:
    - An A3 Mega node pool (32 nodes, 256 GPUs)
    - Topology-aware scheduling enabled
- An Artifact Registry repository to store the Docker image.
- A Google Cloud Storage (GCS) bucket to store results.
  *Important: This bucket must be in the same region as the GKE cluster*.
- A client workstation with the following pre-installed:
   - Google Cloud SDK
   - Helm
   - kubectl

To prepare the required environment, see
[GKE environment setup guide](../../../../docs/configuring-environment-gke-a3-mega.md).

## Run the recipe

It is recommended to use Cloud Shell as your client to complete the steps.
Cloud Shell comes pre-installed with the necessary utilities, including
`kubectl`, `the Google Cloud SDK`, and `Helm`.

### Launch Cloud Shell

In the Google Cloud console, start a [Cloud Shell Instance](https://console.cloud.google.com/?cloudshell=true).

### Configure environment settings

From your client, complete the following steps:

1. Set the environment variables to match your environment:

  ```bash
  export PROJECT_ID=<PROJECT_ID>
  export REGION=<CLUSTER_REGION>
  export CLUSTER_NAME=<CLUSTER_NAME>
  export GCS_BUCKET=<GCS_BUCKET>
  export ARTIFACT_REGISTRY=<ARTIFACT_REGISTRY>
  ```
  Replace the following values:

  - `<PROJECT_ID>`: your Google Cloud project ID
  - `<CLUSTER_REGION>`: the region where your cluster is located
  - `<CLUSTER_NAME>`: the name of your GKE cluster
  - `<GCS_BUCKET>`: the name of your Cloud Storage bucket. Do not include the `gs://` prefix
  - `<ARTIFACT_REGISTRY>`: the full name of your Artifact
    Registry in the following format: *LOCATION*-docker.pkg.dev/*PROJECT_ID*/*REPOSITORY*

2. Set the default project:

  ```bash
  gcloud config set project $PROJECT_ID
  ```

### Get the recipe

From your client, clone the `gpu-recipes` repository and set a reference to the recipe folder.

```
git clone https://github.com/ai-hypercomputer/gpu-recipes.git
cd gpu-recipes
export REPO_ROOT=`git rev-parse --show-toplevel`
export RECIPE_ROOT=$REPO_ROOT/training/a3mega/gpt3-175b/nemo-pretraining-gke
```

### Get cluster credentials

From your client, get the credentials for your cluster.

```
gcloud container clusters get-credentials $CLUSTER_NAME --region $CLUSTER_REGION
```

### Build and push a docker container image to Artifact Registry

To build the container, complete the following steps from your client:

1. Use Cloud Build to build and push the container image.

    ```bash
    cd $REPO_ROOT/src/docker/nemo-24.07
    gcloud builds submit --region=${REGION} \
        --config cloudbuild.yml \
        --substitutions _ARTIFACT_REGISTRY=$ARTIFACT_REGISTRY \
        --timeout "2h" \
        --machine-type=e2-highcpu-32 \
        --quiet \
        --async
    ```

  This command outputs the `build ID`.

2. You can monitor the build progress by streaming the logs for the `build ID`.
   To do this, run the following command.

   Replace `<BUILD_ID>` with your build ID.

   ```bash
   BUILD_ID=<BUILD_ID>

   gcloud beta builds log $BUILD_ID --region=$REGION
   ```

### Configure and submit a pretraining job

The default job setting is 50 training steps and fp8 precision. To execute the job with the
default settings, run the following command from your client:

```bash
cd $RECIPE_ROOT
helm install -f values.yaml \
    --set-file nemo_config=$REPO_ROOT/src/frameworks/nemo-configs/gpt3-175b-256gpus-fp8.yaml \
    --set workload.image=${ARTIFACT_REGISTRY}/nemo_workload:24.07 \
    --set workload.gcsBucketForDataCataPath=${GCS_BUCKET} \
    $USER-gpt3-175b-nemo \
    $REPO_ROOT/src/helm-charts/nemo-training
```

#### Configure job settings

You can overwrite any of the default
[NeMo configurations](../../../../src/frameworks/nemo-configs/gpt3-175b-256gpus-fp8.yaml)
for this job. To do this, we can set the new arguments using `--set workload.arguments`.

**Examples**

- To set the number of training steps to 100, run the following command from your client:

  ```bash
  cd $RECIPE_ROOT
  helm install -f values.yaml \
      --set-file nemo_config=$REPO_ROOT/src/frameworks/nemo-configs/gpt3-175b-256gpus-fp8.yaml \
      --set workload.image=${ARTIFACT_REGISTRY}/nemo_workload:24.07 \
      --set workload.gcsBucketForDataCataPath=${GCS_BUCKET} \
      --set workload.arguments="{trainer.max_steps=100}" \
      $USER-gpt3-175b-nemo \
      $REPO_ROOT/src/helm-charts/nemo-training
  ```

### Monitor the job

To check the status of pods in the indexed job, run the following command from your client:

```
kubectl get pods | grep $USER-gpt3-175b-nemo
```

To get the logs for one of the pods, run the following command from your client:

```
kubectl logs "<pod_name>"
```

### Analyze results

When completed, the job creates several artifacts, including logs and traces, and places them
in the configured Google Cloud Storage bucker as follows:

```
gs://${GCP_BUCKET}/nemo-experiments/<JOB_ID>
├── hparams.yaml
├── lightning_logs.txt
├── nemo_error_logs.txt
├── nemo_log_globalrank-[RANK]_localrank-[LOCAL].txt
├── dllogger
│   ├── rank-0
│   │   ├── dllogger.json
...
```

- `hparams.yaml`: the NeMo configuration used by the pretraining script. This includes
   the combined [configuration file](../../../../src/frameworks/nemo-configs/gpt3-175b-256gpus-fp8.yaml)
   and the command line overrides
- `lightning_logs.txt`: the log files generated by PyTorch Lightning, which is used by NeMo
- `nemo_error_logs.txt`: the warning and error logs generated by NeMo
- `nemo_log_globalrank-[RANK]_localrank-[LOCAL].txt`: the NeMo logs for each rank
- `dllogger/: The log captured by [NVIDIA DLLogger](https://github.com/NVIDIA/dllogger)`:
   DLLogger is configured to store logs on the rank 0 node. The log is in JSON format
   and includes loss, step_time, and other key metrics for each training step

Here is an example of an entry in the DLLogger log:

```json
DLLL {
  "timestamp": "1727463426.984522",
  "datetime": "2024-09-27 18:57:06.984522",
  "elapsedtime": "1645.119157",
  "type": "LOG",
  "step": 47,
  "data": {
    "reduced_train_loss": 7.890198707580566,
    "lr": 0.000036782606912311167,
    "global_step": 47,
    "consumed_samples": 98304,
    "train_backward_timing in s": 0.00004844665454584174,
    "grad_norm": 8.800999641418457,
    "train_step_timing in s": 25.115550994873047,
    "epoch": 0
  }
}
```

The DLLogger log can be used to calculate the Model FLOPS Utilization (MFU) metric,
as described in the next section.

### Calculate training performance metrics (MFU, TFLOPS, Average Step Time)

This section explains how to calculate key training performance metrics,
such as Model FLOPS Utilization (MFU), using the `dllogger.json` file generated during training.

We provide a tool called [training_metrics](../../../../src/utils/training_metrics/) to help
you easily compute these metrics. This tool can calculate the following metrics:

- *MFU*: Model FLOPS Utilization
- *Average training step time*: the average time taken for each training step
- *TFLOPS per GPU*: the number of Tera Floating Point Operations per second achieved by each GPU

To calculate training performance metrics using the `training_metrics` tool, complete the
following steps command from your client:

1. Download the `dllogger.json` file. The `dllogger.json` file is generated during the
   training session.

    To download the file, run the following command. Replace `<JOB_ID>` with the ID of your
    training session.

    ```bash
    gcloud storage cp gs://${GCS_BUCKET}/nemo-experiments/<JOB_ID>/dllogger/rank-0/dllogger.json \
        /path/to/your/local/dllogger.json
    ```

2. Run the
   [`process_training_results.py`](../../../../src/utils/training_metrics/process_training_results.py)
   script

   ```bash
   cd $REPO_ROOT/src/utils/training_metrics
   python3 process_training_results.py --file /path/to/your/local/dllogger.json \
   --batch_size 2048 \
   --num_accelerators 256 \
   --precision fp8 \
   --model_type gpt3-175b \
   --accelerator_type h100
   ```
**Note:** The `batch_size`, `num_accelerators`, `precision`, `model_type` and `accelerator_type` are the
specific values for this recipe running the default configuration. Average step time
is computed by default using the steps 10 to 30.

For more detailed information and advanced usage instructions of this tool,
see the [full documentation](../../../../src/utils/training_metrics/README.md)

### Uninstall the Helm release

You can delete the job and other resources created by the Helm chart.
To uninstall Helm, run the following command from your client:

```bash
helm uninstall $USER-gpt3-175b-nemo
```
