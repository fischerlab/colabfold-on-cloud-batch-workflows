# Colabfold inference pipeline on Google Cloud Batch and Workflows

This repository maintains code samples demonstrating how to operationalize ColabFold batch inference using Google Workflows and Cloud Batch.
This repository was developed exclusively for demonstration purposes.
No modifications were made to the ColabFold software. The Colabfold version used in this solution is the v1.3.0.

## Solutions architecture

The following diagram depicts the high level architecture of the solution:

![Architecture](/images/colabfold-cloud-batch-arch.png)

- The Colabfold execution is encapsulated in a Python script packaged in a Docker container image.
- The Colabfold software has not been modified. This solution aims at operationalize the Colabfold execution at scale.
- ColabFold inference execution is executed as a container Cloud Batch job.
- The Cloud Batch jobs are orchestrated with Google Workflows. We refer to a Google Workflows workflow that orchestrates the inference steps as the ColabFold inference pipeline. 
- The feature engineering step calls the MMSeq2 APIs.
- The prediction and relaxation steps are executed in parallel on GPU equipped machines. The relaxation step is optional.
- Artifacts generated by the inference pipeline - MSAs, predictions, PDB structures, etc - are stored in Google Cloud Storage.
- Metadata generated by the inference pipeline - pipeline run parameters, MSA properties, prediction metrics, artifact lineage, etc  are managed in Cloud Firestore.


## Environment setup

This section outlines the steps to configure the demo environment.

### Select a Google Cloud project

In the Google Cloud Console, on the project selector page, [select or create a Google Cloud project](https://console.cloud.google.com/projectselector2). You need to be a project owner in order to set up the environment.

### Enable the required services

From [Cloud Shell](https://cloud.google.com/shell/docs/using-cloud-shelld.google.com/shell/docs/using-cloud-shell), run the following commands to enable the required Cloud APIs:

```bash
export PROJECT_ID=<YOUR_PROJECT_ID>
 
gcloud config set project $PROJECT_ID
 
gcloud services enable \
cloudbuild.googleapis.com \
compute.googleapis.com \
cloudresourcemanager.googleapis.com \
iam.googleapis.com \
container.googleapis.com \
cloudtrace.googleapis.com \
iamcredentials.googleapis.com \
monitoring.googleapis.com \
logging.googleapis.com \
file.googleapis.com \
workflows.googleapis.com \
batch.googleapis.com \
cloudfunctions.googleapis.com 
```

### Grant permissions to Cloud Build service account

```
PROJECT_NUMBER=$(gcloud projects list --filter="$(gcloud config get-value project)" --format="value(PROJECT_NUMBER)")

gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com" \
        --role="roles/run.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com" \
        --role="roles/workflows.admin"
```

### Create a Cloud Storage bucket

The ColabFold inference pipeline uses Cloud Storage to manage ColabFold model parameters and artifacts created during inference runs.

Create a bucket in the region in which you intend to run your inference jobs. Make sure to check that your preferred [region is supported by Cloud Batch](https://cloud.google.com/batch/docs/locations)

```
export REGION=<YOUR REGION>
export BUCKET_NAME=<gs://your_bucket_name>

gsutil mb -l $REGION $BUCKET_NAME
```

## Deploy the solution's components

The components of the solution - the container image, the inference workflow and the metadata update service - can be built and deployed with Cloud Build.

#### Clone the repo

```
git clone https://github.com/GoogleCloudPlatform/colabfold-on-cloud-batch-workflows
```

#### Submit a Cloud Build job

If you want you can modify the default container image and workflow names.

```
IMAGE_NAME=colabfold-batch
WORKFLOW_NAME=colabfold-workflow
REGION=us-central1

SUBSTITUTIONS=\
_REGION=$REGION,\
_IMAGE_NAME=$IMAGE_NAME,\
_WORKFLOW_NAME=$WORKFLOW_NAME

gcloud builds submit colabfold-on-cloud-batch-workflows --config=colabfold-on-cloud-batch-workflows/cloudbuild.yaml --substitutions $SUBSTITUTIONS --machine-type=e2-highcpu-8
```

#### Create a Workbench to execute the experiment

In the sandbox environment, an instance of Vertex Workbench is used as a development/experimentation environment to customize, start, and analyze inference pipelines runs. There are a couple of setup steps that are required before you can use example notebooks.

From the Cloud Shell, create a new Workbench user-managed notebook.

```
gcloud notebooks instances create colabfold-workbench \
--vm-image-project=deeplearning-platform-release \
--vm-image-family=common-cpu-notebooks \
--machine-type=n1-standard-4 \
--location=us-central1-a
```

Connect to JupyterLab on your Vertex Workbench instance and start a JupyterLab terminal.

From the JupyterLab terminal clone the demo repository:
`git clone https://github.com/GoogleCloudPlatform/vertex-ai-alphafold-inference-pipeline.git`


## Getting started


You can use the utility functions in the `src/workflow_executor` module to configure and submit inference pipeline runs. The  module contains two functions:

- `prepare_args_for_experiment` - This function formats the runtime parameters for the Google Workflows workflow that implements the pipeline. It also sets default values for a number of runtime parameters
- `execute_workflow` - This function executes the  workflow.

Refer to function `doc strings` for full descriptions of the function signatures.

The `1-submit_colabfold_run.ipynb` notebook demonstrates how to use the `src/workflow_executor` module for configuring and starting pipeline runs. 

If you want to analyze pipeline runs you can walk through the `2-metadata-exploration.ipynb` notebook that demonstrates common analysis techniques.
