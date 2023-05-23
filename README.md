# Antimicrobial resistance (amr) detection from *Enterococcus faecium* (efm) whole-genome sequences (wgs)
ARIBA-based pipeline for genotypic prediction of AMR from whole-genome sequences of *Enterococcus faecium*

This GitHub project contains the data and code necessary to reproduce the findings of the study 'Improved accuracy of antibiotic resistance determination from *Enterococcus faecium* whole-genome sequences', and includes the following directories:
* amr_database: directory with database (catalogue) of AMR genetic determinants for *E. faecium* and scripts needed to transform it into an ARIBA-compliant format.
* ariba_amr_database: directory with input files and scripts to run ARIBA and call resistance from *E. faecium* genomes based on our curated catalogue of AMR genetic determinants for *E. faecium*.
* amrfinder: directory with scripts and files needed to run AMRFinderPlus from *E. faecium* assemblies, and parse AMRFinderPlus results.
* card_rgi: directory with input files and scripts to run CARD RGI from *E. faecium* genomes, and parse CARD RGI results.
* lre_finder: directory with files and scripts needed to run LRE-Finder on paired-end fastq files, and parse LRE-Finder reports.
* resfinder: directory with scripts and files needed to run ResFinder (and PointFinder) from *E. faecium* genomes (assemblies and paired-end fastq.gz files) and parse ResFinder reports.
* mapping: directory with the bash scripts used to run Snippy on paired-end fastq files and assemblies.
* phenotype_genotype: directory with the R scripts used to compare phenotypic AST results with WGS-derived genotypic predictions by various tool (described above), and our ARIBA-based predictions (based on our curated catalogue of AMR genetic determinants for *E. faecium*).
* docker: directory with the Dockerfile needed to create a Docker image to run our ARIBA-based pipeline (under construction).

# Required dependencies

## ARIBA-based pipeline

* [ARIBA](https://github.com/sanger-pathogens/ariba) version >= 2.14.6
* [ARIBA](https://github.com/sanger-pathogens/ariba#required-dependencies) dependencies
* [Snippy](https://github.com/tseemann/snippy) version >= 4.6.0
* [fastaq](https://github.com/sanger-pathogens/Fastaq) version >= 3.17.0

## Other tools used
* [AMRFinderPlus](https://www.ncbi.nlm.nih.gov/pathogens/antimicrobial-resistance/AMRFinder/) version 3.10.18; amrfinder database version: 2021-12-21.1
* [CARD RGI](https://github.com/arpcard/rgi) rgi version v5.2.1; CARD database v3.1.4
* [ResFinder](https://bitbucket.org/genomicepidemiology/resfinder/src/master/) version 4.1.10; database downloaded 03/03/2022
* [LRE-Finder](https://bitbucket.org/genomicepidemiology/lre-finder/src/master/) version 1.0.0

# Installation and Usage

## 1. Download this GitHub project
And navigate to downloaded directory
```console
git clone https://github.com/francesccoll/amr_efm_wgs/
cd amr_efm_wgs
```

## 2. Download ARIBA and Snippy's Docker containers

Using Docker
```console
docker pull sangerpathogens/ariba
docker pull staphb/snippy:4.6.0
```

Or using Singularity
```console
singularity build ariba.sif docker://sangerpathogens/ariba:latest
singularity build snippy-4.6.0.sif docker://staphb/snippy:4.6.0
```

## 3. Download test data
The commands below will download the paired-end Illumina sequence data of a multi-drug resistant *E. faecium* isolate (ENA accession: ERR1542086)
```console
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR154/006/ERR1542086/ERR1542086_1.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR154/006/ERR1542086/ERR1542086_2.fastq.gz
```

## 4. Run ARIBA on raw sequence data (required)
When running ARIBA on a Docker/Singilarity container, make sure the current working directory is bound/mounted.
Edit the path below "/absolute/path/to/working/directory/" to match your system's working directory.
```console
singularity run -B "/absolute/path/to/working/directory/amr_efm_wgs/" ariba.sif ariba run ./amr_database/enterococci_amr_genes.v0.3_ariba/ ERR1542086_1.fastq.gz ERR1542086_2.fastq.gz ERR1542086_ariba
```
ARIBA results will be written into ERR1542086_ariba directory.
Note that the first positional argument of "ariba run" is the name of the output directory generated by `ariba prepareref`. The curated database of AMR genetic determinants was already prepared and processed by "ariba prepareref", and the resulting files available on this GitHub project (`./amr_database/enterococci_amr_genes.v0.3_ariba/`). See instructions below on how to update, prepare and process the curated database.

Next, run "ariba expandflag" command to expand the flag column in ARIBA's report (recommended)
```console
singularity run -B "/absolute/path/to/working/directory/amr_efm_wgs/" ariba.sif ariba expandflag ./ERR1542086_ariba/report.tsv ./ERR1542086_ariba/report.flag.tsv
```

## 5. Run Snippy on raw sequence data (recommended)
We noted that ARIBA may miss the detection of Multiple Nucleotide Polymorphism (MNP) and complex SNPs. It is thus recommended to run Snippy or a similar variant-calling pipeline to increase the sensitivity of detecting such variants.
```console
singularity run -B "/absolute/path/to/working/directory/amr_efm_wgs/" snippy-4.6.0.sif snippy --outdir ERR1542086_snippy --ref ./mapping/Enterococcus_faecium_Aus0004.CP003351.1.gb --R1 ERR1542086_1.fastq.gz --R2 ERR1542086_2.fastq.gz
```
Snippy results will be written into ERR1542086_snippy output directory.
Note that the *E. faecium* reference genome used for mapping (Enterococcus_faecium_Aus0004.CP003351.1.gb) is already available inside the `./mapping/` directory of this GitHub project.

## 6. Call genotypic resistance from ARIBA report and Snippy VCF files
And finally, use the script call_resistance_from_ariba_report.py:
```console
python3 ./scripts/call_resistance_from_ariba_report.py --look_up_table ./amr_database/enterococci_amr_genes.v0.3.look-up_table.csv --input_genes ./amr_database/genes_file.v0.3.txt --ariba_report ./ERR1542086_ariba/report.flag.tsv --vcf_file ./ERR1542086_snippy/snps.vcf --use_ariba_flag --output_prefix ERR1542086 --sample_id ERR1542086
```
Note that the originally-formatted curated database of AMR genetic determinants (enterococci_amr_genes.v0.3.look-up_table.csv) should be used by this script.

This script uses a default threshold of 90% nucleotide identity and 60% threshold of minimum gene length to detect assembled AMR genes from ARIBA reports.

It also includes the option `--use_ariba_flag` (recommended) to use ARIBA flags to determine if genes are labelled as assembled.

A minimum ratio of reads supporting alternative alleles of 10% to call resistance from heterozygous mutations (see option `--minimum_ratio_het`).

By default, the integrity of acquired AMR genes is checked by making sure genes are fully assembled without frameshifts or truncations, as determined by ARIBA. If genes contain frameshifts or truncations, then they are not used to call resistance. This behaviour can be disabled using the option `--ignore_lof_mutations`.

## 7. Interpreting output files
Output files will be named with the chosen `--output_prefix`. Three files will be obtained:
- `.pheno.txt`: predicted antibiotic susceptibility phenotype based on the detection of AMR genotypes. The file contains a header with antibiotic names, and a single line with the predicted susceptibility status for each antibiotic (*susceptible* or *resistance*).
- `.geno.txt`: AMR genetic determinants detected.
- `.geno.all.txt`: all acquired AMR genes considered and whether they were found.

## Extending and preparing the curated database (optional)
The curated database of AMR genetic determinants (`/amr_database/enterococci_amr_genes.v0.3.look-up_table.csv`) can be edited to add more AMR mutations and genes. If new AMR genes are added, their nucleotide sequence will need to be downloaded in FASTA format, and added to the file `/amr_database/enterococci_amr_genes.v0.3.gene_sequences.csv`.

Both files need to be converted to ARIBA-formatted files runnig the script below:
```console
python3 prepare_ariba_files_from_amr_database.py -t enterococci_amr_genes.v0.3.look-up_table.csv -g enterococci_amr_genes.v0.3.gene_sequences.csv -d amr_genes_sequences/ -m enterococci_amr_genes.v0.3.ariba_metadata.csv -f enterococci_amr_genes.v0.3.ariba_genes.fa
```

NOTE: `ariba prepareref` will remove some AMR genes with the message `"Does not look like a gene (tried both strands and all reading frames)"`. To avoid that, edit ARIBA metadata file (`enterococci_amr_genes.v0.3.ariba_metadata.csv`) to label the removed genes as non-coding, see example lines below:
`aad(6)	0	0	.	.	.`
`efmA	0	0	.	.	.`
`efrB	0	0	.	.	.`
`ermA-EU348758	0	0	.	.	.`

Finally, `ariba prepareref` can be run on the edited output files of `prepare_ariba_files_from_amr_database.py`:
```console
ariba prepareref -f enterococci_amr_genes.v0.3.ariba_genes.fa -m enterococci_amr_genes.v0.3.ariba_metadata.csv enterococci_amr_genes_ariba.v0.3 --force --verbose
```


# Citation
Coll F, *et al.* Improved accuracy of antibiotic resistance determination from *Enterococcus faecium* whole-genome sequences. (submitted for publication)
