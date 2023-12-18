## Test solubility modifications to peptides

### Preamble
Starting with a list of peptides and proposed amino acid modifications on the N- or C-terminal ends (or automatically selected modifications), aimed at improving the solubility of synthesize long peptides, test for creation of strong binding peptides containing modified amino acids. Summarize these findings and avoid use of modified peptides that lead to predicted strong binding peptide containing these synthetic modifications.

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

### Create N-term and C-term modified peptide lists for processing
The following is an example that assumes you have saved your peptides in a tabular format "peptide_table.tsv" with no header and three columns: (a) unique peptide name, (b) peptide sequence, (c) parsable peptide sequence with a "|" separating the wild type sequence from the additional amino acids to improve solubility. This initial file could be created by copy/paste of the proposed peptides from an excel spreadsheet.

Example of format of the input peptides table:
```bash
CCND2.1.n-term-K	KKFAMYPLSMIATGSVG	K|KFAMYPLSMIATGSVG
CCND2.1.n-term-KK	KKKFAMYPLSMIATGSVG	KK|KFAMYPLSMIATGSVG
CCND2.1.n-term-KKK	KKKKFAMYPLSMIATGSVG	KKK|KFAMYPLSMIATGSVG
CCND2.1.n-term-R	RKFAMYPLSMIATGSVG	R|KFAMYPLSMIATGSVG
CCND2.1.n-term-RR	RRKFAMYPLSMIATGSVG	RR|KFAMYPLSMIATGSVG
CCND2.1.n-term-RRR	RRRKFAMYPLSMIATGSVG	RRR|KFAMYPLSMIATGSVG
CCND2.2.n-term-K	KTDFKFAMYPLSMIATGSVG	K|TDFKFAMYPLSMIATGSVG
CCND2.2.n-term-KK	KKTDFKFAMYPLSMIATGSVG	KK|TDFKFAMYPLSMIATGSVG
CCND2.2.n-term-R	RTDFKFAMYPLSMIATGSVG	R|TDFKFAMYPLSMIATGSVG
CCND2.2.n-term-RR	RRTDFKFAMYPLSMIATGSVG	RR|TDFKFAMYPLSMIATGSVG
SLC35E4.2.c-term-KK	GGGTSPARVFETWGISGATWDKK	GGGTSPARVFETWGISGATWD|KK
SLC35E4.2.c-term-RR	GGGTSPARVFETWGISGATWDRR	GGGTSPARVFETWGISGATWD|RR
SLC35E4.2.c-term-KR	GGGTSPARVFETWGISGATWDKR	GGGTSPARVFETWGISGATWD|KR
SLC35E4.2.c-term-RK	GGGTSPARVFETWGISGATWDRK	GGGTSPARVFETWGISGATWD|RK
SLC35E4.3.c-term-KK	GSRLSALSYVGSHSLFQEKK	GSRLSALSYVGSHSLFQE|KK
SLC35E4.3.c-term-RR	GSRLSALSYVGSHSLFQERR	GSRLSALSYVGSHSLFQE|RR
SLC35E4.3.c-term-KR	GSRLSALSYVGSHSLFQEKR	GSRLSALSYVGSHSLFQE|KR
SLC35E4.3.c-term-RK	GSRLSALSYVGSHSLFQERK	GSRLSALSYVGSHSLFQE|RK
```


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

At this point you should stop and check that these input files appear correct. For example, each sequence should start/end with an expected solubility modification, and it should be at the end expected based on whether it is a n-term (start) or c-term (end) modification.

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
   cat $INFILE | perl -sne 'chomp; if($_ =~/^\>/){print "$_\n"}elsif($_ =~ /(\w+)\|(\w+)/){$before=$1; $after=$2; $sub=substr($after, 0, $length-length($before)); print "$before$sub\n"}' -- -length=$LENGTH > $LENGTH_FASTA
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
   cat $INFILE | perl -sne 'chomp; if($_ =~/^\>/){print "$_\n"}elsif($_ =~ /(\w+)\|(\w+)/){$before=$1; $after=$2; $sub=substr($before, -($length-length($2))); print "$sub$after\n"}' -- -length=$LENGTH > $LENGTH_FASTA
done
```

At this point you should stop and examine the individual peptide sequences designed to be fed into pVACbind for each length run. Note that for each length N (e.g. `-e1 8`) the corresponding fasta sequences will be N+1 or greater in length such that it is not possible to extract a substring of that length without including at least one modified amino acid.  pVACbind will extract all sub-strings of the specified length (e.g. 8 for `-e1 8`). 

For example, if the modification was an N-term-RRR modification to QPSTLVQRPTSLF. The 8-mer fasta file would include 7 amino acids and all three R's:

RRRQPSTLVQ 

pVACbind would then iterate through this sequence and test the following 8-mer sequences:

RRRQPSTL
RRQPSTLV
RQPSTLVQ

Note that all sequences are of the target length being test by pVACbind (8) and all tested sequences contain at least one modified amino acid. This approach makes the interpretation easy.  Any strong binding peptide that comes out of this analysis is something we want to avoid.


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

To check for successful completion of all jobs you can check the stdout logs that have been saved. There should be 8 successful jobs total, 4 lengths for n-term modified peptides and 4 lengths for c-term.

```bash
grep "Pipeline finished" */pvacbind_results/*/stdout.txt | wc -l
```

### Combine all the pVACbind results into a single file
Create a combined TSV file by concatenating all the individual "all_epitopes.tsv" files and avoiding redundant headers. Store this file locally (or in a cloud bucket) so that it can be accessed after the VM is destroyed.

```bash
#get the header line
grep -h "^Mutation" --color=never */pvacbind_results/*/MHC_Class_I/${SAMPLE_NAME}.all_epitopes.tsv | sort | uniq > header.tsv

#combine the results from all prediction runs and add the header on
cat */pvacbind_results/*/MHC_Class_I/${SAMPLE_NAME}.all_epitopes.tsv | grep -v "^Mutation" | cat header.tsv - > ${SAMPLE_NAME}.all_epitopes.all_modifications.tsv

```

### Evaluate the proposed modified peptide sequences
The goal of this analysis is to test whether any strong binding peptides are created that include the modified amino acid sequences included to improve solubility. For example, one could require that no such peptides exist where the median binding affinity is < 500nm OR median binding score percentile is < 1%.

For each candidate modified peptide sequence, summarize the number of such potentially problematic peptides. 

```bash

#pull out all the rows that correspond to strong binders according to default criteria (<500nm affinity OR <1 percentile score)
cut -f 1,2,4,5,8 ${SAMPLE_NAME}.all_epitopes.all_modifications.tsv | perl -ne 'chomp; @l=split("\t",$_); $median_affinity=$l[3]; $median_percentile=$l[4]; if ($median_affinity < 500 || $median_percentile < 1){print "$_\n"}' > ${SAMPLE_NAME}.all_epitopes.all_modifications.problematic.tsv

#summarize number of problematic results of each unique candidate proposed peptide
cat ${SAMPLE_NAME}.all_epitopes.all_modifications.problematic.tsv | grep -v "^Mutation" | cut -f 1 | sort | uniq -c | sed 's/^[ ]*//' | tr " " "\t" | awk 'BEGIN {FS="\t"; OFS="\t"} {print $2, $1}' > ${SAMPLE_NAME}.problematic.summary.tsv

#create a list of all unique peptide names for modified peptides to be summarized
cut -f 1 ${SAMPLE_NAME}.all_epitopes.all_modifications.tsv | grep -v "^Mutation" | sort | uniq > peptide_name_list.tsv

#create an output table with a count of problematic binders for all peptides (include 0 if that is the case)
join -t $'\t' -a 1 -a 2 -e'0' -o '0,2.2' peptide_name_list.tsv ${SAMPLE_NAME}.problematic.summary.tsv > ${SAMPLE_NAME}.problematic.summary.complete.tsv

```

### Retrieve final result files to local system

Files to be kept:

- ${SAMPLE_NAME}.all_epitopes.all_modifications.tsv
- ${SAMPLE_NAME}.all_epitopes.all_modifications.problematic.tsv
- ${SAMPLE_NAME}.problematic.summary.complete.tsv

```bash
#leave the GCP VM
exit

export SAMPLE_NAME="jlf-100-026"

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

### Final report generation and interpretation
Use the information in `${SAMPLE_NAME}.all_epitopes.all_modifications.tsv` and `${SAMPLE_NAME}.problematic.summary.complete.tsv` to produce summary spreadsheets.


