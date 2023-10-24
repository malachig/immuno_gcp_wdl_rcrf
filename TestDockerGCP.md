# Testing a docker image in the same GCP VM environment used by Cromwell/WDL pipelines

## Local dependencies
The following assumes you have gcloud installed and have authenticated to use the google cloud project below


## Launching the VM

```bash
Launch GCP instance that has the same base image as used by Cromwell / Google Life Sciences
gcloud compute instances create mg-test-docker-image --project griffith-lab \
       --service-account=cromwell-server@griffith-lab.iam.gserviceaccount.com --scopes=cloud-platform \
       --image-family cos-stable --image-project cos-cloud \
       --zone us-central1-b --network=cloud-workflows --subnet=cloud-workflows-default \
       --boot-disk-size=250GB --boot-disk-type=pd-ssd --machine-type=custom-8-16384
```
 
## Staging input data to the GCP intance
In many instances some external data will be needed locally on the VM for some ad hoc analysis. How to stage this data really depends on where the data is located.

Example command to use `gcloud compute scp` from your laptop or elsewhere to push data directly to the instance:

```bash
gcloud compute scp Homo_sapiens.GRCh38.pep.all.fa.gz mgriffit@mg-test-docker-image:Homo_sapiens.GRCh38.pep.all.fa.gz --zone us-central1-b
```

Other options might include logging into the instance and then downloading files from a cloud bucket using: `gsutil cp` or `aws s3 cp`.

## Logging into the GCP instance

```bash
gcloud compute ssh mg-test-docker-image --zone us-central1-b
```

## Example analsis that uses a pvactools docker image

```bash
#Create a working dir and stage some test data
mkdir -p pvac-test/result1
mv Homo_sapiens.GRCh38.pep.all.fa.gz annotated.expression.vcf.gz annotated.expression.vcf.gz.tbi phased.vcf.gz phased.vcf.gz.tbi pvac-test/
cd pvac-test/

#Set some envs
export BASE=/home/mgriffit/pvac-test
export HLA_ALLELES="HLA-A*32:01,HLA-A*24:02,HLA-B*13:02,HLA-B*40:02,HLA-C*02:02,HLA-C*06:02,DQA1*01:01,DQA1*05:05,DQB1*03:01,DQB1*05:01,DRB1*01:02,DRB1*11:01"
export OUT=result1

#Enter an interactive docker session with the pvactools image you wish to test
docker run -it --env HOME --env BASE --env HLA_ALLELES --env OUT -v $HOME/:$HOME/ -v $HOME/.config/gcloud:/root/.config/gcloud susannakiwala/pvactools:4.0.4_python3.7_mhcflurry-cmdline /bin/bash

#Run the pvactools test command
cd $HOME/pvac-test/result1

/usr/local/bin/pvacseq run --iedb-install-directory /opt/iedb --pass-only -e1  8,9,10,11 -e2  12,13,14,15,16,17,18 -b 500 --aggregate-inclusion-binding-threshold 1500 \
--normal-sample-name Normal-ExomeDNA-DFCI --run-reference-proteome-similarity --peptide-fasta $BASE/Homo_sapiens.GRCh38.pep.all.fa.gz -m median -d 100 -p $BASE/phased.vcf.gz \
-c 0.0 --normal-cov 30 --tdna-cov 30 --normal-vaf 0.01 --tdna-vaf 0.1 --trna-vaf 0.1 --maximum-transcript-support-level 1 --problematic-amino-acids C --allele-specific-anchors \
--n-threads 8 $BASE/annotated.expression.vcf.gz TumorMet-ExomeDNA-Caris $HLA_ALLELES all $BASE/$OUT 1>$BASE/$OUT/stdout.txt 2>$BASE/$OUT/stderr.txt

#Exit the docker image
exit

```

## Cleanup
Make sure to store any files on te VM somewhere else (most likely a cloud bucket), before exiting and destroying the instance.

```bash
#Exit the Google VM
exit

#Delete the Google VM and all associated storage
gcloud compute instances delete mg-test-docker-image

```

