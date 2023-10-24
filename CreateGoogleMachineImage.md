## Create a VM to use for ad hoc analysis on the Google Cloud

### Local dependencies
The following assumes you have gcloud installed and have authenticated to use the google cloud project below

Set up Google cloud configurations and make sure the right one is activated:
```bash 
export GCS_PROJECT=jlf-rcrf
export GCS_VM_NAME=jlf-adhoc-v1

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
Launch GCP instance that with the latest Ubuntu OS as its base

```bash
gcloud compute instances create $GCS_VM_NAME --project $GCS_PROJECT \
       --service-account=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com --scopes=cloud-platform \
       --image-family ubuntu-2204-lts --image-project ubuntu-os-cloud \
       --network=cloud-workflows --subnet=cloud-workflows-default \
       --boot-disk-size=250GB --boot-disk-type=pd-ssd --machine-type=e2-standard-8

```

### Log into the GCP instance and check status

```bash
gcloud compute ssh $GCS_VM_NAME 

#confirm start up scripts have completed. use <ctrl> <c> to exit
journalctl -u google-startup-scripts -f

#check for expected disk space
df -h 

#OPTIONAL - do some basic security updates (hit enter if prompted with any questions)
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
sudo reboot

#wait a few seconds to allow reboot to complete and then login again
gcloud compute ssh $GCS_VM_NAME 

```

### Install Docker engine on the Google VM
If you know your analysis will not require running tasks within Docker, you can skip this step. However, the following just take a few minutes and will allow your user to run tools that are available as Docker images.

```bash
# set up repository
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# install docker engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

#add user
sudo usermod -a -G docker $USER
sudo reboot

```

The reboot will will kick you out of the instance. Log in again and test the docker install
```bash
gcloud compute ssh $GCS_VM_NAME 

# test install
docker run hello-world

```

# exit the VM, shut it down, and create a machine image from it
```bash

#leave the GCP VM
exit

#list current running instances
gcloud compute instances list

gcloud compute instances stop $GCS_VM_NAME

gcloud beta compute machine-images create jlf-adhoc-v1 --project=jlf-rcrf \ 
       --description="Based on Ubuntu 22.04 OS. Updated with all patches. Docker installed." \
       --source-instance=$GCS_VM_NAME --source-instance-zone=us-central1-c --storage-location=us-central1

```

### Once the image is created, destroy the Google VM on GCP to avoid wasting resources

```bash

gcloud compute instances delete $GCS_VM_NAME

```

### Test launch a VM based on the new image


```bash
gcloud compute instances create jlf-adhoc-image-test \ 
       --service-account=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com \ 
       --source-machine-image=jlf-adhoc-v1 --network=cloud-workflows --subnet=cloud-workflows-default \
       --boot-disk-size=250GB --boot-disk-type=pd-ssd --machine-type=e2-standard-8

gcloud compute ssh jlf-adhoc-image-test

gcloud compute instances delete jlf-adhoc-image-test

```


