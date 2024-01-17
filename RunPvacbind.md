## Run pVACbind on a single sequence of interest

### Preamble
Simplified tutorial that just launches a VM and runs a single pVACbind command

### Local dependencies
The following assumes you have gcloud installed and have authenticated to use the google cloud project below

Set up Google cloud configurations and make sure the right one is activated:
```bash 
export GCS_PROJECT=jlf-rcrf
export GCS_VM_NAME=mg-test-peptide-mods 

#list possible configs that are set up
gcloud config configurations list

#activate the rcrf config
gcloud config configurations activate rcrf

#login if needed (only needs to be done once)
gcloud auth login 

#view active config/login (should show the correct project "jlf-rcrf", zone, and email address)
gcloud config list

```

Configure these configurations to use a specific zone. Once the config is setup and you have logged into at least once the config files should look like this:

`cat ~/.config/gcloud/configurations/config_rcrf`

```
[compute]
region = us-central1
zone = us-central1-c
[core]
account = <email address associated with rcrf account>
disable_usage_reporting = True
project = jlf-rcrf
```

### Launching a Google VM to perform the predictions
Launch GCP instance set up for ad hoc analyses (including docker)

```bash
gcloud compute instances create $GCS_VM_NAME \ 
       --service-account=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com \ 
       --source-machine-image=jlf-adhoc-v1 --network=cloud-workflows --subnet=cloud-workflows-default \
       --boot-disk-size=250GB --boot-disk-type=pd-ssd --machine-type=e2-standard-8
```

### Log into the GCP instance and check status

```bash
gcloud compute ssh $GCS_VM_NAME 

#confirm start up scripts have completed. use <ctrl> <c> to exit
journalctl -u google-startup-scripts -f

#check for expected disk space
df -h 

```

### Configure Docker to work for current user

```bash
sudo usermod -a -G docker $USER
sudo reboot

```

Logout and login to get this change to take effect and test the docker install
```bash
exit

gcloud compute ssh $GCS_VM_NAME 

docker run hello-world

```

Set some environment variables. Make sure the sample name and HLA alleles are correct for the current case!
```bash
export SAMPLE_NAME="jlf-100-000"
export HLA_ALLELES=""
export WORKING_DIR=$HOME/pvacbind
export INFILE=$HOME/input_peptides.fa

mkdir -p $WORKING_DIR
```

At this point you need to input the peptide sequences of interest into $HOME/input_peptides.fa

### Enter a pVACtools docker environment to run pVACbind on the peptide sequences of interest

```bash
docker pull griffithlab/pvactools:latest
docker run -it -v $HOME/:$HOME/ --user $(id -u):$(id -g) --env HOME --env SAMPLE_NAME --env HLA_ALLELES --env WORKING_DIR griffithlab/pvactools:latest /bin/bash
cd $HOME

   pvacbind run $INFILE $SAMPLE_NAME $HLA_ALLELES all $WORKING_DIR -e1 8,9,10,11 -e2 12,13,14,15,16,17,18 --n-threads 8 --iedb-install-directory /opt/iedb/ 1>$WORKING_DIR/stdout.txt 2>$WORKING_DIR/stderr.txt

### Retrieve final result files to local system

Files to be kept:

- ...
- ...
- ...

```bash
#leave the GCP VM
exit

export SAMPLE_NAME="jlf-100-000"

mkdir ${SAMPLE_NAME}_modified_peptide_results
cd ${SAMPLE_NAME}_modified_peptide_results

gcloud compute scp $USER@$GCS_VM_NAME:${SAMPLE_NAME}.all_epitopes.all_modifications.tsv ${SAMPLE_NAME}.all_epitopes.all_modifications.tsv

gcloud compute scp $USER@$GCS_VM_NAME:${SAMPLE_NAME}.all_epitopes.all_modifications.problematic.tsv ${SAMPLE_NAME}.all_epitopes.all_modifications.problematic.tsv

gcloud compute scp $USER@$GCS_VM_NAME:${SAMPLE_NAME}.problematic.summary.complete.tsv ${SAMPLE_NAME}.problematic.summary.complete.tsv


```

### Once the analysis is done and results retrieved, destroy the Google VM on GCP to avoid wasting resources

```bash

gcloud compute instances delete $GCS_VM_NAME

```
