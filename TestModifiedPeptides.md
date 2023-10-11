## Test solubility modifications to peptides

### Preamble
Starting with a list of peptides and proposed amino acid modifications on the N- or C-terminal ends (or automatically selected modifications), aimed at improving the solubility of synthesize long peptides, test for creation of strong binding peptides containing modified amino acids. Summarize these findings and avoid use of modified peptides that lead to predicted strong binding peptide containing these synthetic modifications.

### Local dependencies
The following assumes you have gcloud installed and have authenticated to use the google cloud project below

```bash 
export GCS_PROJECT=jlf-rcrf
export GCS_VM_NAME=mg-test-peptide-mods 

gcloud auth login
gcloud config set project $GCS_PROJECT
gcloud config list
```

### Launching a Google VM to perform the predictions
Launch GCP instance that has the same base image as used by Cromwell / Google Life Sciences

```bash
gcloud compute instances create $GCS_VM_NAME --project $GCS_PROJECT \
       --service-account=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com --scopes=cloud-platform \
       --image-family ubuntu-2204-lts --image-project ubuntu-os-cloud \
       --zone us-central1-b  --network=cloud-workflows --subnet=cloud-workflows-default \
       --boot-disk-size=250GB --boot-disk-type=pd-ssd --machine-type=e2-standard-8

```

### Log into the GCP instance and check status

```bash
gcloud compute ssh $GCS_VM_NAME --zone us-central1-b

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
gcloud compute ssh $GCS_VM_NAME --zone us-central1-b

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
gcloud compute ssh $GCS_VM_NAME --zone us-central1-b

# test install
docker run hello-world


```

### Create N-term and C-term modified peptide lists for processing
The following is an example that assumes you have saved your peptides in a tabular format "peptide_table.tsv" with no header and three columns: (a) unique peptide name, (b) peptide sequence, (c) parsable peptide sequence with a "|" separating the wild type sequence from the additional amino acids to improve solubility. This initial file could be created by copy/paste of the proposed peptides from an excel spreadsheet.

```bash
wc -l peptide_table.tsv

#make sure the names are unique
cut -f 1 peptide_table.tsv | sort | uniq | wc -l



```










