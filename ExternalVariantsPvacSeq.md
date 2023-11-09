### Run pVACseq on externally identified variants

The following process fills an increasingly common use case. 

Specifically the case where we have run the immuno.wdl pipeline and have produced variant calls and neoantigen candidates but we have have additional variants identified by an external group, pipeline or commercial tumor portrait. Since these variants were not identified by the immuno pipeline and are not in the VCF we can not run pVACseq on them.  The following recovers these variants by creating a new VCF representing them, annotating this VCF with all the same steps performed in immuno.wdl and then runs pVACseq on that VCF.

#### Local dependencies

The following assumes you have gcloud installed and have authenticated to use the google cloud project below

#### Step 1. Connect to a Google VM with Docker to perform the analysis

Set up Google cloud configurations and make sure the right one is activated:

```bash
export GCS_PROJECT=jlf-rcrf
export GCS_VM_NAME=external-variants-pvacseq

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

```bash
[compute]
region = us-central1
zone = us-central1-c
[core]
account = <email address associated with rcrf account>
disable_usage_reporting = True
project = jlf-rcrf
```

Launch a VM based on ubuntu but with additional dependencies installed

```bash

#list available machine images
gcloud beta compute machine-images list

#launch a VM based on one of these images and provision appropriate resources (machine type and disk)
gcloud compute instances create $GCS_VM_NAME \ 
       --service-account=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com \ 
       --source-machine-image=jlf-adhoc-v1 --network=cloud-workflows --subnet=cloud-workflows-default \
       --boot-disk-size=500GB --boot-disk-type=pd-ssd --machine-type=e2-standard-8

#log into that instance
gcloud compute ssh jlf-adhoc-image-test

```


#### Step 2. Gather inputs

The following inputs should be obtained prior to proceeding:

**Reference files**
- Reference Protein fasta (protein sequences for all known coding transcripts)
- Chromosome alias file
- VEP Cache
- Reference genome

**Input data files (from a completed immuno.wdl run for the current tumor)**
- HLA Alleles for current case
- Phased VCF
- Tumor DNA BAM/CRAM (and index BAI)
- Normal DNA BAM/CRAM (and index BAI)
- Tumor RNA BAM (and index BAI)
- Gene expression file (e.g. from Kallisto)
- Transcript expression file (e.g. from Kallisto)

Example commands to obtain each of these inputs
```bash
...

```


#### Step 3. Use the [ClinGen Allele Registry](https://reg.clinicalgenome.org/redmine/projects/registry/genboree_registry/landing) to resolve variants to HGVS

This step depends a bit on the starting format of the variant obtained externally.  The goal is to resolve them to a list of genomic HGVS expressions. Starting formats might look like:

e.g. ERBB3 p.S846I

This works out to:

NC_000012.12:g.56097861G>T / NM_001982.4:c.2537G>T / NP_001973.2:p.Ser846Ile
NC_000012.12:g.56097861G>T / ENST00000267101.8:c.2537G>T / ENSP00000267101.4:p.Ser846Ile 

Create a simple text file: external-variants-hgvs.txt with a single HGVS g. entry: `NC_000012.12:g.56097861G>T` on each line

#### Step 4. Use VEP to create an annotated VCF - using the HGVS format as input

docker: "mgibio/vep_helper-cwl:vep_105.0_v1"

`isub -m 64 -n 8 --preserve false -i 'mgibio/vep_helper-cwl:vep_105.0_v1'`

Using the docker image above, annotate the variant with Ensembl VEP as follows

```
/usr/bin/perl -I /opt/lib/perl/VEP/Plugins /usr/bin/variant_effect_predictor.pl \
--format hgvs \
--vcf \
--fork 4 \
--term SO \
--transcript_version \
--cache \
--symbol \
-o annotated.vcf \
-i /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/erbb3-variant-hgvs.txt \
--synonyms /storage1/fs1/mgriffit/Active/griffithlab/pipeline_test/malachi/refs_backup/May-2023/griffith-lab-workflow-inputs/human_GRCh38_ens105/reference_genome/chromAlias.ensembl.txt \
--flag_pick \
--dir /storage1/fs1/mgriffit/Active/griffithlab/pipeline_test/malachi/refs_backup/May-2023/griffith-lab-workflow-inputs/human_GRCh38_ens105/vep_cache \
--fasta /storage1/fs1/mgriffit/Active/griffithlab/pipeline_test/malachi/refs_backup/May-2023/griffith-lab-workflow-inputs/human_GRCh38_ens105/aligner_indices/bwamem2_2.2.1/all_sequences.fa \
--check_existing \
--plugin Frameshift --plugin Wildtype  \
--everything \
--assembly GRCh38 \
--cache_version 105 \
--species homo_sapiens
```

#### Step 5. Add tumor/normal genotype information to the VCF

`isub -m 64 -n 8 --preserve false -i 'griffithlab/vatools:latest'`

Using the docker image above, use VA tools to add genotype columns to the VCF

```
vcf-genotype-annotator /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.vcf JLF-100-016-tumor 0/1 -o /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.1.vcf

vcf-genotype-annotator /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.1.vcf JLF-100-016-normal 0/0 -o /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.2.vcf
```

#### Step 6. Adding DNA and RNA count/VAF data to the VCF

Work in progress. 

https://pvactools.readthedocs.io/en/latest/pvacseq/input_file_prep/readcounts.html


#### Step 7.

Work in progress.

https://pvactools.readthedocs.io/en/latest/pvacseq/input_file_prep/expression.html


#### Step 8. Run pVACseq on the genotyped VCF

`isub -m 64 -n 8 --preserve false -i 'susannakiwala/pvactools:latest'`

Using the docker image above, use pVACseq to perform neoantigen analysis on the VCF

```
#MAKE SURE THE FOLLOWING HLA ALLELES ARE UPDATED TO REFLECT THE CURRENT TUMOR SAMPLE
export HLA_ALLELES="HLA-A*01:01,HLA-A*02:13,HLA-B*08:01,HLA-B*57:03,HLA-C*07:01,HLA-C*06:02"

export PEPTIDE_FASTA=/storage1/fs1/gillandersw/Active/Project_0001_Clinical_Trials/annotation_files_for_review/Homo_sapiens.GRCh38.pep.all.fa.gz

pvacseq run /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.2.vcf \
            JLF-100-016-tumor \
            $HLA_ALLELES \
            all /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/pvacseq/ \
            -e1 8,9,10,11 -e2 12,13,14,15,16,17,18 --iedb-install-directory /opt/iedb \
            -b 500  -m median -k -t 8 --run-reference-proteome-similarity \
            --aggregate-inclusion-binding-threshold 1500 \
            --peptide-fasta $PEPTIDE_FASTA \
            -d 100 --normal-sample-name JLF-100-016-normal --problematic-amino-acids C \
            --allele-specific-anchors \
            --maximum-transcript-support-level 1 --pass-only 

```


#### Step 9. Retrieve results files from VM and clean up



