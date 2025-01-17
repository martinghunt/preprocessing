# Mycobacterial Pre-processing Pipeline #
  
Cleans and QCs reads with fastp and FastQC, classifies with Kraken2 & Mykrobe, removes non-bacterial content, and - by alignment to any minority genomes - disambiguates mixtures of bacterial reads.

Takes as input one directory containing pairs of fastq(.gz) or bam files.
Produces as output one directory per sample, containing the relevant reports & a pair of cleaned fastqs.

## Quick Start ## 
E.g. to run:
```
nextflow run main.nf -profile singularity --filetype fastq --input_dir fq_dir --pattern "*_R{1,2}.fastq.gz" --unmix_myco yes --output_dir .
nextflow run main.nf -profile docker --filetype bam --input_dir bam_dir --unmix_myco yes --output_dir .
```

## Checkpoints ##
Checkpoints used throughout this workflow to fail a sample/issue warnings:
 
 processes preprocessing_checkFqValidity or preprocessing_checkBamValidity
 1. (Fail) If sample does not pass fqtools 'validate' or samtools 'quickcheck', as appropriate.
 
 process preprocessing_countReads\
 2. (Fail) If sample contains < 100k pairs of raw reads.
 
 process preprocessing_fastp\
 3. (Fail) If sample contains < 100k pairs of cleaned reads, required to all be > 50bp (cleaning using fastp with --length_required 50 --average_qual 10 --low_complexity_filter --correction --cut_right --cut_tail --cut_tail_window_size 1 --cut_tail_mean_quality 20).

 process preprocessing_kraken2\
 4. (Fail) If the top family hit is not Mycobacteriaceae\
 5. (Fail) If there are fewer than 100k reads classified as Mycobacteriaceae \
 6. (Warn) If the top family classification is mycobacterial, but this is not consistent with top genus and species classifications\
 7. (Warn) If the top family is Mycobacteriaceae but no G1 (species complex) classifications meet minimum thresholds of > 5000 reads or > 0.5% of the total reads (this is not necessarily a concern as not all mycobacteria have a taxonomic classification at this rank) \
 8. (Warn) If sample is mixed or contaminated - defined as containing reads > the 5000/0.5% thresholds from multiple non-human species\
 9. (Warn) If sample contains multiple classifications to mycobacterial species complexes, each meeting the > 5000/0.5% thresholds\
 10. (Warn) If no species classification meets the 5000/0.5% thresholds\
 11. (Warn) If no genus classification meets the 5000/0.5% thresholds\
 12. (Fail) If no family classification meets the 5000/0.5% thresholds (redundant given point 5)
 
 process preprocessing_identifyBacterialContaminants\
 13. (Fail) If the sample is not contaminated and the top species hit is not one of the 10 supported Mycobacteria:\ abscessus|africanum|avium|bovis|chelonae|chimaera|fortuitum|intracellulare|kansasii|tuberculosis\
 14. (Fail) If the sample is not contaminated and the top species hit is contrary to the species expected (e.g. "avium" rather than "tuberculosis" - only tested if you provide that expectation)\
 15. (Warn) If the top species hit is supported by < 75% coverage\
 16. (Warn) If the top species hit has a median coverage depth < 10-fold\
 17. (Warn) If we are unable to associate an NCBI taxon ID to any given contaminant species, which means we will not be able to locate its genome, and thereby remove it as a contaminant\
 18. (Warn) If we are unable to determine a URL for the latest RefSeq genome associated with a contaminant species' taxon ID\
 19. (Warn) If no complete genome could be found for a contaminant species. The workflow will proceed with alignment-based contaminant removal, but you're warned that there's reduced confidence in detecting reads from this species
 
 process preprocessing_downloadContamGenomes\
 20. (Fail) If a contaminant is detected but we are unable to download a representative genome, and thereby remove it
 
 process preprocessing_summarise\
 21. (Fail) If after having taken an alignment-based approach to decontamination, Kraken still detects a contaminant species\
 22. (Fail) If after having taken an alignment-based approach to decontamination, the top species hit is not one of the 10 supported Mycobacteria\
 23. (Fail) If, after successfully removing contaminants, the top species hit is contrary to the species expected (e.g. "avium" rather than "tuberculosis" - only tested if you provide that expectation)

