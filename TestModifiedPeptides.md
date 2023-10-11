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
#count the number of entries in the input file
wc -l peptide_table.tsv

#make sure the names are unique
cut -f 1 peptide_table.tsv | sort | uniq | wc -l

#create a fasta formatted file of all candidates
cut -f 1,3 peptide_table.tsv | perl -ne 'chomp; @l=split("\t",$_); print ">$l[0]\n$l[1]\n"' > modified_peptides.fa

#create dirs for processing the N-term and C-term sequences separately
mkdir n-term
mkdir c-term

#divide the fasta records into two files, one for n-term and one for c-term
grep --color=never -A 1 "n-term" modified_peptides.fa > n-term/modified_peptides_n-term.fa
grep --color=never -A 1 "c-term" modified_peptides.fa > c-term/modified_peptides_c-term.fa

#make sure nothing has been lost
wc -l modified_peptides.fa 
wc -l n-term/modified_peptides_n-term.fa
wc -l c-term/modified_peptides_c-term.fa 

```

### Set up sub-peptide fasta sequences for each target class I prediction length
The following assumes that:

- 1. proposed modifications are only at the *start* (N-terminus) or *end (C-terminus) of the peptide sequence
- 2. every peptide in the file has a unique ID on the header line
- 3. every peptide has a proposed modification
- 4. the modification is marked with a "|" (proposed modification to left for N-term modified sequences, and to the right for C-term modified sequences)


Set some environment variables. Make sure the sample name and HLA alleles are correct for the current case!
```bash
export SAMPLE_NAME="jlf-100-026"
export HLA_ALLELES="HLA-A*02:01,HLA-A*24:02,HLA-B*07:02,HLA-B*35:02,HLA-C*04:01,HLA-C*07:02"
```

Create the N-terminal fasta files first
```bash
export WORKING_DIR=$HOME/n-term
export INPUTS_DIR=$WORKING_DIR/pvacbind_inputs
export RESULTS_DIR=$WORKING_DIR/pvacbind_results
export INFILE=$WORKING_DIR/modified_peptides_n-term.fa

mkdir -p $INPUTS_DIR
mkdir -p $RESULTS_DIR

for LENGTH in 8 9 10 11
do
   #Create input files for each test length so that each test sequence will contain at least one modified base (e.g. 7 AA before modifications for 8-mer test)
   echo "Creating input fasta to test peptides of length: $LENGTH"
   export LENGTH_FASTA=$INPUTS_DIR/${LENGTH}-mer-test.fa
   export LENGTH_RESULT_DIR=$RESULTS_DIR/${LENGTH}-mer-test
   mkdir -p $LENGTH_RESULT_DIR
   cat $INFILE | perl -sne 'chomp; if($_ =~/^\>/){print "$_\n"}elsif($_ =~ /(\w+)\|(\w+)/){$before=$1; $after=$2; $sub=substr($after, 0, $length-1); print "$before$sub\n"}' -- -length=$LENGTH > $LENGTH_FASTA
done 
```

Now create the C-terminal fasta files

```bash
export WORKING_DIR=$HOME/c-term
export INPUTS_DIR=$WORKING_DIR/pvacbind_inputs
export RESULTS_DIR=$WORKING_DIR/pvacbind_results
export INFILE=$WORKING_DIR/modified_peptides_c-term.fa

mkdir -p $INPUTS_DIR
mkdir -p $RESULTS_DIR

for LENGTH in 8 9 10 11
do 
   #Create input files for each test length so that each test sequence will contain at least one modified base (e.g. 7 AA before modifications for 8-mer test)
   echo "Creating input fasta to test peptides of length: $LENGTH"
   export LENGTH_FASTA=$INPUTS_DIR/${LENGTH}-mer-test.fa
   export LENGTH_RESULT_DIR=$RESULTS_DIR/${LENGTH}-mer-test
   mkdir -p $LENGTH_RESULT_DIR
   cat $INFILE | perl -sne 'chomp; $length2=$length-1; if($_ =~/^\>/){print "$_\n"}elsif($_ =~ /(\w+)\|(\w+)/){$before=$1; $after=$2; $sub=substr($before, -$length2); print "$sub$after\n"}' -- -length=$LENGTH > $LENGTH_FASTA
done
```

### Enter a pVACtools docker environment to run pVACbind on the sub-peptide sequences containing modified AAs

```bash
docker pull griffithlab/pvactools:4.0.5
docker run -it -v $HOME/:$HOME/ --user $(id -u):$(id -g) --env HOME --env SAMPLE_NAME --env HLA_ALLELES griffithlab/pvactools:4.0.5 /bin/bash
cd $HOME

for LENGTH in 8 9 10 11
do 
   #process n-term fasta for this length
   echo "Running pVACbind for length: $LENGTH (n-term sequences)"
   export LENGTH_FASTA=$HOME/n-term/pvacbind_inputs/${LENGTH}-mer-test.fa
   export LENGTH_RESULT_DIR=$HOME/n-term/pvacbind_results/${LENGTH}-mer-test
   pvacbind run $LENGTH_FASTA $SAMPLE_NAME $HLA_ALLELES all_class_i $LENGTH_RESULT_DIR -e1 $LENGTH --n-threads 8 --iedb-install-directory /opt/iedb/ 1>$LENGTH_RESULT_DIR/stdout.txt 2>$LENGTH_RESULT_DIR/stderr.txt

   #process c-term fasta for this length
   echo "Running pVACbind for length: $LENGTH (c-term sequences)"
   export LENGTH_FASTA=$HOME/c-term/pvacbind_inputs/${LENGTH}-mer-test.fa
   export LENGTH_RESULT_DIR=$HOME/c-term/pvacbind_results/${LENGTH}-mer-test
   pvacbind run $LENGTH_FASTA $SAMPLE_NAME $HLA_ALLELES all_class_i $LENGTH_RESULT_DIR -e1 $LENGTH --n-threads 8 --iedb-install-directory /opt/iedb/ 1>$LENGTH_RESULT_DIR/stdout.txt 2>$LENGTH_RESULT_DIR/stderr.txt
done

```











