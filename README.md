# Running the WASHU Immunogenomics Workflow on Google Cloud - RCRF version

## Preamble
This doc demonstrates how to run the WASHU immunogenomics pipeline (immuno.wdl) on Google Cloud.

The same principles described in this tutorial should work for any of our collection of WDL files found here: https://github.com/wustl-oncology/analysis-wdls.

This workflow run will be accomplished by first setting up workflow definitions, input data and 
reference files and a YAML config file on a user local system and a Google VM. The user will set 
up a Google Cloud environment and all the inputs will be staged to this cloud environment. Next 
Cromwell will be used to execute the pipeline using the specified input and reference files, and 
finally the results will be stored in a cloud bucket and cloud resources and intermediate files 
will be cleaned up. 

You will interact with the Google Cloud in several ways:
1. From your local system using command line tools, including `gcloud` and `gsutil`.
2. We will create a Google Virtual Machine (VM) and start a Cromwell service on it. You will then 
login to this VM and perform some commands and monitoring of workflow progress there. Cromwell on 
this VM will orchestrate creation and use of many additional worker VMs that complete all the compute 
tasks of the workflow.
3. The Google Cloud Console in your web browser may be used to visualize/monitor usage of cloud 
resources. In the Console you will see most relevant resources in the "Cloud Storage" and 
"Compute Engine" sections.

This version assumes that your are staging input data files from an RCRF AWS S3 Cloud Bucket. All input files 
will be staged to a Google Storage Bucket. During the workflow, all results files will be also be stored 
in this Bucket. At the end of the workflow the final results will be copied back to an RCRF AWS S3 Bucket.

After completing the workflow, ALL resources used on the Google cloud can be destroyed. One exception 
to this may occur if you need custom reference files that you wish to persist for many analyses. However, for 
this demonstration, reference files will be accessed from a separate public bucket that we have created to 
support this workflow.

A brief note on command line sessions. Almost everything below will occur at the command line. It is very 
easy to get confused about the kind of sessions used. There are three types that will be used:
1. A terminal session on your local system (e.g. using the Terminal App on a Mac laptop)
2. Within session (1) you may login (via `gcloud compute ssh`) to the Google Virtual Machine where Cromwell is running.
3. Within session (2) you may login to an interactive docker session using specific docker images.

### Source of instructions
This documents describes a specific example of how to run a specific pipeline (immuno). The steps below are taken from the following link where you will find a more generic set of documentation that explains in detail how to run any WDL pipeline on the Google Cloud using tools created to assist this process. 
https://github.com/wustl-oncology/cloud-workflows/tree/main/manual-workflows

### Prerequisites (for local system)
- google-cloud-sdk
- docker
- git

### Setting up a Google Cloud account
In order to do analysis on the Google Cloud, you will need access to the RCRF GCP account. Before proceeding, log into it and make sure you have access to the project: 'jlf-rcrf'.

Some notes on account set up once your are logged in:
- When you log into the GCP console make sure the correct project is selected (top left corner): 'jlf-rcrf'. 
- Create billing alerts! In the Google Cloud Web Console, select: Billing -> Budgets & alerts -> Create Budget. How you set up your alerts will depend on your anticipated level of use/expenditure. For example, you might set at $500 budget and then set up alerts to be sent at 50%, 100%, 200%, ..., X% of that budget.
- Choose a name for the Google bucket that will be used for this tutorial. We recommend the following convention: 'malachi-jlf-immuno' below. This will help keep multiple parallel analyses separate.
 
Some notes on quotas:
- If you have not been using your account for high performance computing, it is likely that by default you have quotas in place that will cause the very large immuno.wdl workflow to fail. For example, you may not be able to use enough CPUs, IP addresses and disk space at once to get this workflow to run. If you see errors that mention quotas, or sound like fundamental network failures, check your quotas in the Google Cloud Console (IAM & Admin -> Quotas) and work with your Google account contact to increase your quotas. If your institution has already been using Google Cloud in any serious way, this is less likely to be a problem. 

Example quotas you might need to request:
- `cpus` -> `us-central1` -> `200`
- `preemptible_cpus` -> `us-central1` -> `200`
- `In-use IP addresses` -> `us-central1` -> `100`
- `Persistent Disk SSD (GB)` -> `us-central1` -> `10 TB`

### Interacting with Google buckets from your local system
Note that, if needed, you can use the following docker image to access `gsutil` for exploration of your google storage: `docker(google/cloud-sdk)`. Or alternatively, you can install the Google Cloud SDK on your system. This latter approach is assumed by the following instructions.

## Step-by-step instructions
Start by opening a Terminal session on your local system

### Set some Google Cloud and other environment variables on your local system
The following environment variables are used merely for convenience and should be customized to produce intuitive labeling for your own analysis:

```bash
export GCS_PROJECT=jlf-rcrf
export GCS_SERVICE_ACCOUNT=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com
export GCS_BUCKET_NAME=malachi-jlf-immuno
export GCS_BUCKET_PATH=gs://$GCS_BUCKET_NAME
export GCS_CASE_NAME=jlf-100-044
export GCS_INSTANCE_NAME=mg-immuno-${GCS_CASE_NAME}
export WORKING_BASE=~/Desktop/rcrf/$GCS_CASE_NAME
```

### Login to GCP and set the desired project on your local system
From the command line, you will need to authenticate your cloud access (using your google cloud account credentials). This generally only needs to be done once, though there is no harm to re-authenticating. The login command below will generate a custom URL to enter in your browser. Once you do this, you will be prompted to log into your Google account. If you have multiple Google accounts (e.g. one for your institution/company and a personal one) be sure to use the correct one.  Once you have logged in you will be presented with a long code. Enter this at the prompt generated by the login command below. Finally, set your desired Google Project. This Project should correspond to a Google Billing account in the Google console. If you are using Google Cloud for the first time, both billing and a project should be set up before proceeding. Configuring billing alerts is also probably wise at this point.
```bash
gcloud auth login
gcloud config set project $GCS_PROJECT
gcloud config list
```

## Local setup

### First create a working directory on your local system
The following directory on the local system will contain: (a) a git repository for the WDL workflows, including the immuno worflow, (b) a git repository for tools that help provision and manage our workflow runs on the cloud.

```bash
mkdir -p $WORKING_BASE
cd $WORKING_BASE
```

### Clone git repositories that have the workflows (pipelines) and scripts to help run them
The following repositories contain: this tutorial (immuno_gcp_wdl), the WDL workflows (analysis-wdls), and tools for running these on the cloud (cloud-workflows). Note that the command below will clone the main branch of each repo, but there are also stable releases.

```bash
mkdir git
cd git
git clone git@github.com:wustl-oncology/analysis-wdls.git
git clone git@github.com:wustl-oncology/cloud-workflows.git
```

The `gcloud config list` command can be used to remind yourself how you are currently authenticated to use Google Cloud Services. This can be helpful because on your host machine, you will be authenticated using your personal account. However, on the Google VM where the workflow will be orchestrated by Cromwell, you will be authenticated using a "service" account. 

### Set up cloud service account, firewall settings, and storage bucket
Run the following command and make note of the "Service Account" returned (e.g. "cromwell-server@test-immuno.iam.gserviceaccount.com"). Make sure this matches the value in $GCS_SERVICE_ACCOUNT (e.g. `echo $GCS_SERVICE_ACCOUNT`).

Note that you must define an appropriate IP or IP range in the following command that corresponds to where you intend to log in to the VM (e.g. your current external IP).

```bash

docker pull mgibio/cloudize-workflow:latest

docker run -it --env WORKING_BASE --env GCS_PROJECT --env GCS_BUCKET_NAME -v $WORKING_BASE/:$WORKING_BASE/ -v /$HOME/.config/gcloud:/root/.config/gcloud mgibio/cloudize-workflow:latest /bin/bash

cd $WORKING_BASE/git/cloud-workflows/manual-workflows/

bash resources.sh init-project --project $GCS_PROJECT --bucket $GCS_BUCKET_NAME --ip-range "128.252.0.0/16,65.254.96.0/19"

exit
```

This step should have created two new configuration files in your current directory: `cromwell.conf` and `workflow_options.json`. Take a look at these files to make sure they point to the expected cloud project, bucket name, etc.


### Start a Google VM that will run Cromwell and orchestrate completion of the workflow

Note that Cromwell produces a large quantity of database logging. To ensure we have enough space for a least a few runs and to localize intermediate and final results files from the workflow (which include numerous large BAMs) we will specify some extra disk space with `--boot-disk-size=250GB` (default would be 10GB). When not testing, this can probably be safely reduced to 20-40GB. Note that this argument must be listed last after the required arguments for the `start.sh` script.
```bash

docker run -it --env WORKING_BASE --env GCS_INSTANCE_NAME --env GCS_SERVICE_ACCOUNT --env GCS_PROJECT -v $WORKING_BASE/:$WORKING_BASE/ -v /$HOME/.config/gcloud:/root/.config/gcloud mgibio/cloudize-workflow:latest /bin/bash

cd $WORKING_BASE/git/cloud-workflows/manual-workflows/

bash start.sh $GCS_INSTANCE_NAME --server-account $GCS_SERVICE_ACCOUNT --project $GCS_PROJECT --boot-disk-size=250GB

exit

```

### Log into the VM and check status 

In this step we will confirm that we can log into our Cromwell VM with `gcloud compute ssh` and make sure it is ready for use.

After logging in, use journalctl to see if the instance start up has completed, and cromwell launch has completed.

For details on how to recognize whether these processes have completed refer: [here](https://github.com/griffithlab/cloud-workflows/tree/main/manual-workflows#ssh-in-to-vm).

```bash
gcloud compute ssh $GCS_INSTANCE_NAME
journalctl -u google-startup-scripts -f
journalctl -u cromwell -f
exit
```

### Set some environment variables on the Google VM
The following can be saved in the .bashrc for convenience:

```bash
gcloud compute ssh $GCS_INSTANCE_NAME

export GCS_PROJECT=jlf-rcrf
export GCS_SERVICE_ACCOUNT=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com
export GCS_BUCKET_NAME=malachi-jlf-immuno
export GCS_BUCKET_PATH=gs://$GCS_BUCKET_NAME
export GCS_CASE_NAME=jlf-100-044

```

### Stage input data files to cloud bucket

First install the AWS CLI (this could be moved to the VM setup)

```bash
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Stage data from wherever it is coming from into the Google bucket. The following is
an example but will vary from case to case, and relies on you having access/credentials
for the data.

```bash
cd ~
mkdir data
cd data

aws s3 ls s3://rcrf-h37-data/JLF/JLF-100-044/Raw_Sequencing_Data/
aws s3 cp --recursive s3://rcrf-h37-data/JLF/JLF-100-044/Raw_Sequencing_Data/BG004015 .

gsutil cp -r ./BG004015/* gs://malachi-jlf-immuno/input_data/2023-07-28/
gsutil ls gs://malachi-jlf-immuno/input_data/2023-07-28/

rm -fr ~/data

```

### Obtain an example configuration (YAML) file on cromwell instance
Create a directory for YAML files and create one for the desired pipeline that points to the location of input files on your local system

```bash
gcloud compute ssh $GCS_INSTANCE_NAME
mkdir git
mkdir yamls
cd ~/git
git clone https://github.com/malachig/immuno_gcp_wdl_rcrf.git
cd ~/yamls
cp ~/git/immuno_gcp_wdl_rcrf/example_yamls/example_immuno_cloud-WDL.yaml .
mv example_immuno_cloud-WDL.yaml ${GCS_CASE_NAME}_immuno_cloud-WDL.yaml

```

You will now need to update this YAML to correspond to your data in the following ways:
- Add Google paths to input sequence data files staged above
- Add sequence metadata and sample names
- Add RNA-seq strand info
- If available, add Google bucket paths to validated variants VCF and VCF index
- If available, add known HLA alleles for the case
- If desired update any run time parameters

### Run the immuno workflow using everything setup thus far

While logged into the Google Cromwell VM instance:
```bash
source /shared/helpers.sh
submit_workflow /shared/analysis-wdls/definitions/immuno.wdl $CLOUD_YAML

```

### Monitor progress of the workflow run:

While the job is running you can see Cromwell logs live as they occur by doing this
```bash
journalctl -f -u cromwell

```

### Save information about the workflow run itself - Timing Diagram and Outputs List

After a workflow is run, before exiting and deleting your VM, make sure that the timing diagram and the list of outputs are available so you can make use of the data outside of the cloud.

First determine you WORKFLOW_ID. This can be done several ways. If the run was successful it should be reported at the bottom of the cromwell log as "$WORKFLOW_ID  completed with status Succeeded". Or you find it by the name of the directory where your run was stored in the Google bucket. Both of these approaches are illustrated here:

```bash
export GCS_BUCKET_PATH=gs://test-immuno-pipeline
gsutil ls $GCS_BUCKET_PATH/cromwell-executions/immuno/

journalctl -u cromwell | tail | grep "Workflow actor"
```

Now save the workflow information in your google bucket
```bash
export WORKFLOW_ID=<id from above>
source /shared/helpers.sh
save_artifacts $WORKFLOW_ID $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID
```

This command will upload the workflow's artifacts to your google bucket so they can be used after the VM is deleted. They can be found at paths:

```bash
$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/timing.html
$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json
```

Confirm that they were successfully transferred and logout of the Cromwell VM on GCP:
```bash
gsutil ls $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID
exit
```

The file `outputs.json` will simply be a map of output names to their GCS locations. The `pull_outputs.py` script can be used to retrieve the actual files.


### Pulling the Outputs from Google Cloud Bucket back to your local system or cluster
After the work in your compute instance is all done, including `save_artifacts`, and you want to bring your results back to the cluster, leverage the `pull_outputs.py` script with the generated `outputs.json` to retrieve the files.

On compute1 cluster, jump into a docker container with the script available

```bash
export WORKFLOW_ID=<id from above>

docker run -it --env WORKFLOW_ID --env GCS_BUCKET_PATH --env WORKING_BASE -v /Users/mgriffit/Desktop/pipeline_test/:/Users/mgriffit/Desktop/pipeline_test/ -v /Users/mgriffit/.config/gcloud:/root/.config/gcloud mgibio/cloudize-workflow:latest /bin/bash

```

Execute the script:

```bash
cd $WORKING_BASE
mkdir final_results
cd final_results

python3 /opt/scripts/pull_outputs.py --outputs-file=$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json --outputs-dir=$WORKING_BASE/final_results/
exit
```

Examine the outputs briefly:
```bash 
cd $WORKING_BASE/final_results
ls -l
du -h

```

### Estimate the cost of executing your workflow

On compute1 cluster, start an interactive docker session as describe below and then use the following python script to generate a cost estimate:

```bash
export WORKFLOW_ID=<id from above>

docker run -it --env WORKFLOW_ID --env GCS_BUCKET_PATH --env WORKING_BASE -v /Users/mgriffit/Desktop/pipeline_test/:/Users/mgriffit/Desktop/pipeline_test/ -v /Users/mgriffit/.config/gcloud:/root/.config/gcloud mgibio/cloudize-workflow:latest /bin/bash

cd $WORKING_BASE/git/cloud-workflows/scripts
python3 estimate_billing.py $WORKFLOW_ID $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/metadata/
exit
```

### Once the workflow is done and results retrieved, destroy the Cromwell VM on GCP to avoid wasting resources

Use the following commmand to destroy the Cromwell VM. 

```bash
gcloud compute instances delete $GCS_INSTANCE_NAME
```

You can empty the cloud bucket either in the Web Console or using commands like `gsutil rm -r $GCS_BUCKET_PATH/folder-name`.

Finally, you should perform a survey of Cloud Storage and Compute Engine sections in the Google Cloud Web Console to make sure everything has been cleaned up successfully..

