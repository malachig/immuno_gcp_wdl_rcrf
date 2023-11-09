## Run pVACseq on externally identified variants

The following process fills an increasingly common use case. 

Specifically the case where we have run the immuno.wdl pipeline and have produced variant calls and neoantigen candidates but we have have additional variants identified by an external group, pipeline or commercial tumor portrait. Since these variants were not identified by the immuno pipeline and are not in the VCF we can not run pVACseq on them.  The following recovers these variants by creating a new VCF representing them, annotating this VCF with all the same steps performed in immuno.wdl and then runs pVACseq on that VCF.

### Local dependencies

The following assumes you have gcloud installed and have authenticated to use the google cloud project below

### Step 1. Connect to a Google VM with Docker to perform the analysis

Set up Google cloud configurations and make sure the right one is activated:

```bash
export GCS_PROJECT=jlf-rcrf
export GCS_VM_NAME=mg-ext-vars-pvacseq

#list possible configs that are set up
gcloud config configurations list

#activate the rcrf config if it isnt already
gcloud config configurations activate rcrf

#login if needed (only needs to be done once)
gcloud auth login 

#set the project if needed (only needs to be done once)
gcloud config set project $GCS_PROJECT

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

#view currently running instances
gcloud compute instances list 

#log into the new instance
gcloud compute ssh $GCS_VM_NAME

```

Make sure you can run Docker sessions:

```bash
docker run hello-world

#if that doesn't work you may have to give your user permission to run docker first
#do the following and then log out and in, or reboot
sudo usermod -a -G docker $USER

```

### Step 2. Gather input files

The following inputs should be obtained prior to proceeding:

#### Reference files

- Reference Protein fasta (protein sequences for all known coding transcripts)
- Chromosome alias file
- VEP Cache
- Reference genome

```bash
mkdir ~/refs
cd ~/refs

gsutil cp gs://griffith-lab-workflow-inputs/human_GRCh38_ens105/rna_seq_annotation/Homo_sapiens.GRCh38.pep.all.fa.gz .
gsutil cp gs://griffith-lab-workflow-inputs/human_GRCh38_ens105/reference_genome/chromAlias.ensembl.txt .
gsutil cp gs://griffith-lab-workflow-inputs/human_GRCh38_ens105/aligner_indices/bwamem2_2.2.1/all_sequences.fa .
gsutil cp gs://griffith-lab-workflow-inputs/human_GRCh38_ens105/aligner_indices/bwamem2_2.2.1/all_sequences.fa.fai .
gsutil cp gs://griffith-lab-workflow-inputs/human_GRCh38_ens105/vep_cache.zip .
mkdir vep_cache
mv vep_cache.zip vep_cache && cd vep_cache
sudo apt install unzip
unzip vep_cache.zip
```

#### Input data files (from a completed immuno.wdl run for the current tumor)

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


### Step 3. Use the [ClinGen Allele Registry](https://reg.clinicalgenome.org/redmine/projects/registry/genboree_registry/landing) to resolve variants to HGVS

This step depends a bit on the starting format of the variant obtained externally.  The goal is to resolve them to a list of genomic HGVS expressions. Starting formats vary considerably. Description of even the common possibilities here are beyond this scope of this document.

Example variant info:

chr7-87193899-87193903-GCAGA-G (DMTF1|p.D610fs)
chr5-154802261-154802262-AG-A (LARP1|p.D583fs)
chr1-228279865-228279866-G-A (OBSCN|p.R2481H)
chr8-134601485-134601486-C-T (ZFAT|p.E745K)

Whatever the starting format the goal is to convert these to genomic (g.) HGVS entries and to doublecheck that the protein change there agrees with expectations from the external provider of the variants.

Use IGV, ClinGen Allele Registry and associated tools to either manually contruct and check the HGVS or resolve it from the predicted protein change.

For these four examples, covering both simple SNV and indels, this works out to:

**chr7-87193899-87193903-GCAGA-G (DMTF1|p.D610fs)**
- Manual HGVS: chr7 = NC_000007.14 -> NC_000007.14:g.87193900_87193903delCAGA (coordinates provided were left justified following VCF spec)
- ClinGen HGVS result (CA843372381): *NC_000007.14:g.87193902_87193905del* / ENST00000331242.12:c.1828_1831del | ENSP00000332171.7:p.Asp610LeufsTer11
- Asp610LeufsTer11 -> D610LfsTer11 -> D610fs (matches expected annotation)

**chr5-154802261-154802262-AG-A (LARP1|p.D583fs)**
- Manual HGVS: chr5 = NC_000005.10 -> NC_000005.10:g.154802262delG (coordinates provided were left justified folowing VCF spec)
- ClinGen HGVS result (CA447422638): *NC_000005.10:g.154802268del* / ENST00000336314.9:c.1747del / ENSP00000336721.4:p.Asp583ThrfsTer12
- Asp583ThrfsTer12 -> D583TfsTer12 -> D583fs (matches expected annotation)
- Note that in this case a transcript other than the MANE select had to be chosen to match the expected annotation of D583fs

**chr1-228279865-228279866-G-A (OBSCN|p.R2481H)**
- Manual HGVS: chr1 = NC_000001.11 -> NC_000001.11:g.228279866G>A
- ClinGen HGVS result (CA1434153): *NC_000001.11:g.228279866G>A* / ENST00000284548.16:c.7442G>A / ENSP00000284548.11:p.Arg2481His
- Arg2481His -> R2481H (matches expected annotation)
- Note that in this case a transcript other than the MANE select had to be chosen to match the expected annotation of D583fs

**chr8-134601485-134601486-C-T (ZFAT|p.E745K)**
- Manual HGVS: chr8 = NC_000008.11 -> NC_000008.11:g.134601486C>T
- ClinGen HGVS result (CA372078013): *NC_000008.11:g.134601486C>T* / ENST00000377838.8:c.2233G>A / ENSP00000367069.3:p.Glu745Lys
- Glu745Lys -> E745K (matches expected annotation)

Create a simple text file: `~/external-variants-hgvs.txt` with a single HGVS (g.) entry on each line like this:
```
NC_000007.14:g.87193902_87193905del
NC_000005.10:g.154802268del
NC_000001.11:g.228279866G>A
NC_000008.11:g.134601486C>T
```


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

### Step 5. Add tumor/normal genotype information to the VCF

`isub -m 64 -n 8 --preserve false -i 'griffithlab/vatools:latest'`

Using the docker image above, use VA tools to add genotype columns to the VCF

```
vcf-genotype-annotator /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.vcf JLF-100-016-tumor 0/1 -o /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.1.vcf

vcf-genotype-annotator /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.1.vcf JLF-100-016-normal 0/0 -o /storage1/fs1/mgriffit/Active/JLF_MCDB/cases/mcdb022/erbb3/erbb3-pvacseq/annotated.genotyped.2.vcf
```

### Step 6. Adding DNA and RNA count/VAF data to the VCF

Work in progress. 

https://pvactools.readthedocs.io/en/latest/pvacseq/input_file_prep/readcounts.html


### Step 7.

Work in progress.

https://pvactools.readthedocs.io/en/latest/pvacseq/input_file_prep/expression.html


### Step 8. Run pVACseq on the genotyped VCF

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


### Step 9. Retrieve results files from VM and clean up



