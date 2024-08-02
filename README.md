# CNL Halite Sample - Bioinformatics Pipeline for 16S Amplicon Analysis

This repository contains the code and instructions for performing a 16S amplicon analysis using QIIME2 on halite samples provided by the Canadian Nuclear Laboratory (CNL). It also includes the method to infer the presence of hydrogenase genes based on the taxonomic classification from the amplicon data.

## Table of Contents

- [Prerequisites](#Prerequisites)
- [Raw Data Storage](#raw-data-storage)
- [QIIME2 Installation](#qiime2-installation)
- [QIIME2 Pipeline](#qiime2-pipeline)
  - [Importing Data](#importing-data)
  - [Summarizing Imported Data](#summarizing-imported-data)
  - [Sequence Quality Control and Feature Table Construction](#sequence-quality-control-and-feature-table-construction)
  - [Taxonomic classifications](#Taxonomic-classifications)
  - [Dealing with Controls](#dealing-with-controls)
  - [Genus-level Taxonomic Classifications](#genus-level-taxonomic-classifications)
- [GTotree Pipeline](#GTotree-Pipeline)
	- [Download Representative Genomes from GTDB](#download-representative-genomes-from-gtdb)
  	- [Identify Hydrogenase Genes](#identify-hydrogenase-genes)

## Prerequisites
- Access to a Unix-based command line system, here we used the Cedar system from the Digial Research Alliance of Canada (DRAC) [https://alliancecan.ca/en](https://alliancecan.ca/en).


## Raw Data Storage

1. **Create a folder for raw data storage:**
        
    ```bash
    mkdir -p /project/def-careg421/ruizhang/CNL/raw_data
    ```

2. **Download data from Nanuq:**
        
    ```bash
    cd /project/def-careg421/ruizhang/CNL/raw_data
    read -p "Login: " login && read -p "Password: " -s password && echo -n "j_username=$login&j_password=$password" > .auth.txt && chmod 600 .auth.txt && wget -O - "https://ces.genomequebec.com/nanuqMPS/readsetList?projectId=24071&tech=NextSeq" --no-cookies --no-check-certificate --post-file .auth.txt | wget --no-cookies --no-check-certificate --post-file .auth.txt -ci -; rm -f .auth.txt
    ```

3. **Copy raw data to the scratch folder for analysis:**
        
    ```bash
    cp /scratch/ruizhang/CNL/qiime2/raw_data
    ```

## QIIME2 Installation

4. **Install QIIME2 using a Docker image via Apptainer:**
        
    ```bash
    module load StdEnv/2023
    module load apptainer

    cd $SCRATCH
    export APPTAINER_CACHEDIR=/scratch/ruizhang

    apptainer build qiime2-202405.sif docker://quay.io/qiime2/amplicon:2024.5

    cp qiime2-202405.sif $PROJECT/software

    apptainer run -C /project/def-careg421/ruizhang/software/qiime2-202405.sif qiime tools --help
    ```

## QIIME2 Pipeline

### Importing Data

5. **Creating the "manifest file" locally, then transfer to Cedar:**

    ```text
    /project/def-careg421/ruizhang/CNL/qiime2/CNL_amplicon_manifest_file.txt
    ```

6. **Move the file to the `$SCRATCH` folder, replace the file directories with the directories where the raw data is actually stored:**
        
    ```bash
    mv /project/def-careg421/ruizhang/CNL/qiime2/CNL_amplicon_manifest_file.txt /scratch/ruizhang/CNL/qiime2
    ```

7. **Importing data via the manifest file:**
        
    ```bash
    module load apptainer

    export TMPDIR=$SCRATCH/tmp 
    export SLURM_TMPDIR=$SCRATCH/tmp

    input=$SCRATCH/CNL/qiime2
    output=$SCRATCH/CNL/qiime2

    apptainer run -C /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime tools import \
    --type 'SampleData[PairedEndSequencesWithQuality]' \
    --input-path $input/CNL_amplicon_manifest_file.txt \
    --output-path $output/CNL_paired-end-demux.qza \
    --input-format PairedEndFastqManifestPhred33V2
    ```

8. **Importing data via the manifest file (alternative):**
        
    ```bash
    module load apptainer

    export TMPDIR=$SCRATCH/tmp 
    export SLURM_TMPDIR=$SCRATCH/tmp

    input=$SCRATCH/CNL/qiime2
    output=$SCRATCH/CNL/qiime2

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 \
    /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime tools import \
    --type 'SampleData[PairedEndSequencesWithQuality]' \
    --input-path $input/CNL_amplicon_manifest_file.txt \
    --output-path $output/CNL_paired-end-demux.qza \
    --input-format PairedEndFastqManifestPhred33V2
    ```

### Summarizing Imported Data

9. **Summarizing the imported data by creating a `.qzv` visualization:**
        
    ```bash
    input=$SCRATCH/CNL/qiime2
    output=$SCRATCH/CNL/qiime2

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime demux summarize \
    --i-data $input/CNL_paired-end-demux.qza \
    --o-visualization $output/CNL_paired-end-demux.qzv
    ```

### Sequence Quality Control and Feature Table Construction

#### Running DADA2 for quality control

To detect and correct errors in Illumina amplicon sequence data, create a FeatureTable QIIME 2 artifact containing counts of each unique sequence in each sample. Additionally, generate a FeatureData artifact that maps feature identifiers in the FeatureTable to the sequences they represent. These steps ensure accurate quantification and identification of sequences across samples.

10. **Running DADA2 for quality control and feature table construction:**
        
    ```bash
    #!/bin/bash
    #SBATCH -c 32          
    #SBATCH --mem=125G   
    #SBATCH -t 4:00:0
    #SBATCH -J=CNL_qiime2_dada2.sh
    #SBATCH --mail-user=zzhan186@uottawa.ca
    #SBATCH --mail-type=ALL
    #SBATCH --output=%x-%j.out
    #SBATCH --account=def-careg421

    module load StdEnv/2023
    module load apptainer

    input=$SCRATCH/CNL/qiime2
    output=$SCRATCH/CNL/qiime2

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime dada2 denoise-paired \
    --i-demultiplexed-seqs $input/CNL_paired-end-demux.qza \
    --p-trunc-len-f 0 \
    --p-trunc-len-r 0 \
    --o-representative-sequences $output/CNL_rep-seqs-dada2.qza \
    --o-table $output/CNL_table-dada2.qza \
    --o-denoising-stats $output/CNL_stats-dada2.qza \
    --p-n-threads 0
    ```

#### FeatureTable and FeatureData summaries

note: "CNL_sample-metadata.tsv" this metadata file needs to be created in advance. Check out this site [https://docs.qiime2.org/2024.5/tutorials/metadata/]() for more info. 

11. **FeatureTable and FeatureData summaries:**
        
    ```bash
    module load apptainer

    input=$SCRATCH/CNL/qiime2
    output=$SCRATCH/CNL/qiime2

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-table summarize \
    --i-table $input/CNL_table-dada2.qza \
    --o-visualization $output/CNL_table.qzv \
    --m-sample-metadata-file $output/CNL_sample-metadata.tsv

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-table tabulate-seqs \
    --i-data $input/CNL_rep-seqs-dada2.qza \
    --o-visualization $output/CNL_rep-seqs.qzv
	```

### Taxonomic classifications

12. **Download the latest GreenGene taxonomic classifier for full length and v4 region of the 16S rRNA sequences:**

	```bash
	cd $SCRATCH/CNL/qiime2
	wget https://ftp.microbio.me/greengenes_release/2022.10/sklearn-1.4.2-compatible-nb-classifiers/2022.10.backbone.full-length.nb.sklearn-1.4.2.qza --no-check-certificate
	
	wget https://ftp.microbio.me/greengenes_release/2022.10/sklearn-1.4.2-compatible-nb-classifiers/2022.10.backbone.v4.nb.sklearn-1.4.2.qza --no-check-certificate
	```

	
#### determine taxonomic composition of the sample

13. **Using the full length 16S classifier:**

		```bash
		input=$SCRATCH/CNL/qiime2
		output=$SCRATCH/CNL/qiime2
		
		apptainer exec -B $PWD:/home \
		-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
		qiime feature-classifier classify-sklearn \
		--i-classifier $input/2022.10.backbone.full-length.nb.sklearn-1.4.2.qza \
		--i-reads $input/CNL_rep-seqs-dada2.qza \
		--o-classification $output/CNL_taxonomy-full-length.qza
		
		apptainer exec -B $PWD:/home \
		-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
		qiime metadata tabulate \
		--m-input-file $input/CNL_taxonomy-full-length.qza \
		--o-visualization $output/CNL_taxonomy-full-length.qzv
		```

14. **Using a more strict confidence value:**

	```bash
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime feature-classifier classify-sklearn \
	--i-classifier $input/2022.10.backbone.full-length.nb.sklearn-1.4.2.qza \
	--i-reads $input/CNL_rep-seqs-dada2.qza \
	--o-classification $output/CNL_taxonomy-full-length-strict.qza \
	--p-confidence 0.9
	
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime metadata tabulate \
	--m-input-file $input/CNL_taxonomy-full-length-strict.qza \
	--o-visualization $output/CNL_taxonomy-full-length-strict.qzv
	```
		
	"what does a more strict confidence value mean? see this post: [https://forum.qiime2.org/t/how-is-the-confidence-calculated-with-taxa-assignments/179/3]()

	The basic classification method is to decompose the read into a bag of overlapping 8-mers, then feed that as input to the machine learning (Naive Bayes by default) classifier. The confidence of a classification is calculated by bootstrapping (subsampling) the bag of 8-mers 100 times, and seeing how many times the subsample comes up with the same classification as the full read. If the confidence parameter is between zero and one, the classifier will start at the top taxonomic level and work its way down the levels until the calculated confidence falls below the value of the input parameter. At that point it will truncate the classification to the last good level and report the calculated confidence in the Confidence column."


15. **using the V4 region 16S classifier:**

	```bash
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime feature-classifier classify-sklearn \
	--i-classifier $input/2022.10.backbone.v4.nb.sklearn-1.4.2.qza \
	--i-reads $input/CNL_rep-seqs-dada2.qza \
	--o-classification $output/CNL_taxonomy-v4.qza
	
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime metadata tabulate \
	--m-input-file $input/CNL_taxonomy-v4.qza \
	--o-visualization $output/CNL_taxonomy-v4.qzv
	```	

16. **Using a more strict confidence value:**

	```bash
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime feature-classifier classify-sklearn \
	--i-classifier $input/2022.10.backbone.v4.nb.sklearn-1.4.2.qza \
	--i-reads $input/CNL_rep-seqs-dada2.qza \
	--o-classification $output/CNL_taxonomy-v4-strict.qza \
	--p-confidence 0.9
		  
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime metadata tabulate \
	--m-input-file $input/CNL_taxonomy-v4-strict.qza \
	--o-visualization $output/CNL_taxonomy-v4-strict.qzv
	```

17. **view the taxonomic composition of the samples using bar plots:**

	```bash
	input=$SCRATCH/CNL/qiime2
	output=$SCRATCH/CNL/qiime2
	
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime taxa barplot \
	--i-table $input/CNL_table-dada2.qza \
	--i-taxonomy $input/CNL_taxonomy-full-length.qza \
	--m-metadata-file $input/CNL_sample-metadata.tsv \
	--o-visualization $output/CNL_taxa-bar-plots-full-length.qzv
	
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime taxa barplot \
	--i-table $input/CNL_table-dada2.qza \
	--i-taxonomy $input/CNL_taxonomy-v4.qza \
	--m-metadata-file $input/CNL_sample-metadata.tsv \
	--o-visualization $output/CNL_taxa-bar-plots-v4.qzv

	# strict version
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime taxa barplot \
	--i-table $input/CNL_table-dada2.qza \
	--i-taxonomy $input/CNL_taxonomy-full-length-strict.qza \
	--m-metadata-file $input/CNL_sample-metadata.tsv \
	--o-visualization $output/CNL_taxa-bar-plots-full-length-strict.qzv
	
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime taxa barplot \
	--i-table $input/CNL_table-dada2.qza \
	--i-taxonomy $input/CNL_taxonomy-v4-strict.qza \
	--m-metadata-file $input/CNL_sample-metadata.tsv \
	--o-visualization $output/CNL_taxa-bar-plots-v4-strict.qzv
	```

The taxonomy profiles obtained using a full-length 16S rRNA gene classifier are similar to those obtained with a V4-specific classifier, and there is no significant change in community profiles when increasing the confidence level from 0.7 to 0.9. However, since hits are detected in our kit control and blank filter control, the next step is to address these issues.


### Dealing with Controls

#### Removing chimeric sequences using q2-vsearch

18. **Run de novo chimera checking:**
        
    ```bash
    module load apptainer
    input=$SCRATCH/CNL/qiime2
    output=$SCRATCH/CNL/qiime2

    apptainer exec -B $PWD:/home\
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime vsearch uchime-denovo \
    --i-table $input/CNL_table-dada2.qza \
    --i-sequences $input/CNL_rep-seqs-dada2.qza \
    --output-dir $output/CNL_uchime-dn-out
    ```

19. **Visualize summary stats:**
        
    ```bash
    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime metadata tabulate \
    --m-input-file $output/CNL_uchime-dn-out/stats.qza \
    --o-visualization $output/CNL_uchime-dn-out/stats.qzv
    ```

20. **Exclude chimeras and "borderline chimeras":**
        
    ```bash
    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-table filter-features \
    --i-table $input/CNL_table-dada2.qza \
    --m-metadata-file $input/CNL_uchime-dn-out/nonchimeras.qza \
    --o-filtered-table $output/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qza

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-table filter-seqs \
    --i-data $input/CNL_rep-seqs-dada2.qza \
    --m-metadata-file $input/CNL_uchime-dn-out/nonchimeras.qza \
    --o-filtered-data $output/CNL_uchime-dn-out/rep-seqs-nonchimeric-wo-borderline.qza

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-table summarize \
    --i-table $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qza \
    --o-visualization $output/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qzv
    ```

21. **Check the resulting bar graph for changes compared to without chimera removal:**
        
    ```bash
    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-classifier classify-sklearn \
    --i-classifier $input/2022.10.backbone.v4.nb.sklearn-1.4.2.qza \
    --i-reads $input/CNL_uchime-dn-out/rep-seqs-nonchimeric-wo-borderline.qza \
    --o-classification $output/CNL_taxonomy-v4-chimera-removal.qza

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime metadata tabulate \
    --m-input-file $input/CNL_taxonomy-v4-chimera-removal.qza \
    --o-visualization $output/CNL_taxonomy-v4-chimera-removal.qzv

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime taxa barplot \
    --i-table $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qza \
    --i-taxonomy $input/CNL_taxonomy-v4-chimera-removal.qza \
    --m-metadata-file $input/CNL_sample-metadata.tsv \
    --o-visualization $output/CNL_taxa-bar-plots-v4-chimera-removal.qzv
    ```

Okay, not much difference, what were present in the controls are still there.


#### Alpha and beta diversity analysis

22. **Generate a tree for phylogenetic diversity analyses:**
        
    ```bash
    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime phylogeny align-to-tree-mafft-fasttree \
    --i-sequences $input/CNL_uchime-dn-out/rep-seqs-nonchimeric-wo-borderline.qza \
    --o-alignment $output/CNL_aligned-rep-seq-nonchimeric-wo-borderline.qza \
    --o-masked-alignment $output/masked-aligned-rep-seqs.qza \
    --o-tree $output/CNL_unrooted-tree.qza \
    --o-rooted-tree $output/CNL_rooted-tree.qza
    ```

23. **Alpha and beta diversity analysis:**
   
    An important parameter for this script is --p-sampling-depth, which determines the even sampling (rarefaction) depth. Since most diversity metrics are sensitive to varying sampling depths across samples, the script will randomly subsample the counts from each sample to the provided value. For instance, with --p-sampling-depth 500, the counts in each sample will be subsampled to a total count of 500. Samples with counts lower than this value will be excluded from the diversity analysis. It's crucial to choose this value carefully by reviewing the table.qzv file, aiming for a high value to retain more sequences per sample while excluding as few samples as possible. We chose a depth of 2579 to retain all samples except BlankpcrCES3, which only had 142 counts.
        
    ```bash
    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime diversity core-metrics-phylogenetic \
    --i-phylogeny $output/CNL_rooted-tree.qza \
    --i-table $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qza \
    --p-sampling-depth 2579 \
    --m-metadata-file $input/CNL_sample-metadata.tsv \
    --output-dir $output/core-metrics-results
    ```

	By examining the PCoA plots, especially the ones based on bray-curtis distance, jaccard distance, and weighted UNIFRAC, we found that the controls samples (kit control and filter control) tend to be separated from the other core samples. This is actually a good news and align with the scenario one described above - 'If the controls are very dissimilar from your biological samples, or there is no overlap between your biological samples and your controls you do not have to filter them from your data, but describe them in your study'. 

#### Filtering out features in control samples

24. **Filter out features in control samples:**
        
    ```bash
    module load apptainer
    input=$SCRATCH/CNL/qiime2
    output=$SCRATCH/CNL/qiime2

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-table filter-samples \
    --i-table $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qza \
    --m-metadata-file $input/CNL_sample-metadata.tsv  \
    --p-where "[sample-control]='control'" \
    --o-filtered-table $output/control-only-table.qza

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-table filter-features \
    --i-table $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qza \
    --m-metadata-file $input/CNL_sample-metadata.tsv \
    --p-where "[sample-control]='control'" \
    --p-exclude-ids TRUE \
    --o-filtered-table $output/table-nonchimeric-wo-borderline-filter.qza

    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-table summarize \
    --i-table $input/table-nonchimeric-wo-borderline-filter.qza \
    --m-sample-metadata-file $input/CNL_sample-metadata.tsv \
    --o-visualization $output/table-nonchimeric-wo-borderline-filter-summ.qzv
    ```

25. **Filtering features found in control samples from the other samples:**

	caution is to be exercised here, we don't want to remove all features in the control sample, see post: [https://forum.qiime2.org/t/filtering-features-found-in-controls/1739]()
        
    ```bash
    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime tools export \
    --input-path $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qza \
    --output-path $output/table-biome

	# install a virtual environment to install the biom format convertion tool, biom-format

    module load python/3.11
    cd $SCRATCH
    virtualenv --no-download biom-format
    source $SCRATCH/biom-format/bin/activate
    pip install biom-format
    
	# convert biom format table to tsv
    biom convert -i $input/table-biome/feature-table.biom -o $input/table-biome/feature-table.tsv --to-tsv
    ```
  Upon examining the feature table, we observed that many features present in the kit control are exclusive to it, while certain other features are found across all samples. To avoid eliminating features that are common to all samples, we will focus on those features that predominantly appear in the kit control but not in the other samples.
  
	Creating a metadata file (`CNL_control_features_metadata.tsv`) with the features found in control samples


26. **Excluding the control feature in the feature table:**

	```bash
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime feature-table filter-features \
	--i-table $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qza \
	--m-metadata-file $input/CNL_control_features_metadata.tsv \
	--p-where "[exclude]='yes'" \
	--p-exclude-ids \
	--o-filtered-table $output/table-nonchimeric-wo-borderline-filter.qza
	
	apptainer exec -B $PWD:/home \
	-B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
	qiime feature-table summarize \
	--i-table $input/table-nonchimeric-wo-borderline-filter.qza \
	--o-visualization $output/table-nonchimeric-wo-borderline-filter-summ.qzv
  	```
  	
	comparing results for before and after filtering, we see that feature frequency dropped a lot in the control samples, but rarely changed in the actual samples
    
	```bash
	| Sample ID     | Before filtering frequency | After filtering frequency |
	|---------------|----------------------------|---------------------------|
	| 103d-10       | 10339                      | 10127                     |
	| 103d-10-filt  | 24007                      | 24007                     |
	| 103d-100      | 3070                       | 3070                      |
	| 103d-100-filt | 7776                       | 7776                      |
	| 103l-10       | 55872                      | 55872                     |
	| 103l-100      | 3348                       | 3348                      |
	| 103l-100-filt | 4343                       | 4343                      |
	| 104-10        | 5874                       | 5874                      |
	| 104-10-filt   | 4315                       | 4315                      |
	| 104-100       | 2580                       | 2580                      |
	| 104-100-filt  | 7096                       | 7096                      |
	| BlankpcrCES3  | 142                        | 127                       |
	| FLT           | 7726                       | 2959                      |
	| KT            | 6199                       | 1207                      |
	```

### Genus-level Taxonomic Classifications

#### Remove control samples from the feature table

27. **Remove control samples from the feature table:**
        
    ```bash
    module load apptainer
    input=$SCRATCH/CNL/qiime2
    output=$SCRATCH/CNL/qiime2

    apptainer exec -B $

	PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime feature-table filter-samples \
    --i-table $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline.qza \
    --m-metadata-file $input/CNL_sample-metadata.tsv \
    --p-where "[sample-control]='sample'" \
    --o-filtered-table $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline-wo-control.qza
    ```

#### Filter feature table by genus level

28. **Filter feature table by genus level:**
        
    ```bash
    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime taxa filter-table \
    --i-table $input/CNL_uchime-dn-out/table-nonchimeric-wo-borderline-wo-control.qza \
    --i-taxonomy $input/CNL_taxonomy-v4-chimera-removal.qza \
    --p-include g__ \
    --o-filtered-table $output/CNL_filtered_genus-table.qza
    ```

#### Export taxonomy table

29. **Export taxonomy table:**
        
    ```bash
    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime tools export \
    --input-path $input/CNL_filtered_genus-table.qza \
    --output-path $output/CNL_exported-filtered-genus-table

    module load python/3.11
    source $SCRATCH/biom-format/bin/activate
    biom convert -i $input/CNL_exported-filtered-genus-table/feature-table.biom -o $output/CNL_exported-filtered-genus-table/feature-table.tsv --to-tsv
    ```

#### Taxonomy bar plot

30. **Generate taxonomy bar plot for the filtered samples at genus level:**
        
    ```bash
    apptainer exec -B $PWD:/home \
    -B $SCRATCH/CNL/qiime2 /project/def-careg421/ruizhang/software/qiime2-202405.sif \
    qiime taxa barplot \
    --i-table $input/CNL_filtered_genus-table.qza \
    --i-taxonomy $input/CNL_taxonomy-v4-chimera-removal.qza \
    --m-metadata-file $input/CNL_sample-metadata.tsv \
    --o-visualization $output/CNL_taxa-bar-plots-genus-filtered.qzv
    ```
    
#### Obtaining the list of genera found in the halite samples, check out file "CNL_genera.csv"    

	list="
	Pseudomonas_E
	Tepidicella
	Massilia
	Brevundimonas
	Corynebacterium
	Chryseobacterium
	JC017
	Aeromonas
	Xanthomonas_A
	Moheibacter
	Acinetobacter
	Bifidobacterium
	Comamonas_F
	Bacillus_AZ
	Prevotella
	Daeguia
	SDRK01
	Cloacibacterium
	Robertmurraya
	Burkholderia
	Bog-532
	Paenibacillus_I
	Brevibacillus_D
	Caldalkalibacillus
	Sphingobacterium
	Achromobacter
	Neisseria
	Anaerococcus
	Stenotrophomonas_A
	Palsa-1150
	Kaistia
	Methylobacterium"

## GTotree Pipeline

### Download Representative Genomes from GTDB

31. **Download representative genomes from GTDB using GToTree:**
        
    ```bash
    module load StdEnv/2020
    module load python/3.11
    module load hmmer/3.3.2
    module load muscle
    module load trimal
    module load taxonkit
    module load fasttree/2.1.11
    module load prodigal/2.6.3

    source /project/def-careg421/ruizhang/software/GTDB-tk/bin/activate

    export PATH="/project/def-careg421/ruizhang/software/GToTree-1.8.4/bin/:$PATH"
    export GToTree_HMM_dir="/project/def-careg421/ruizhang/software/GToTree-1.8.4/hmm_sets"
    export GTDB_dir="/project/def-careg421/ruizhang/software/release214"
    export PATH="/project/def-careg421/ruizhang/software/bit-1.8.63/bit/:$PATH"

    cd $PROJECT/CNL/GTDB_rep_genomes/

    list="Pseudomonas_E Tepidicella Massilia Brevundimonas Corynebacterium Chryseobacterium JC017 Aeromonas Xanthomonas_A Moheibacter Acinetobacter Bifidobacterium Comamonas_F Bacillus_AZ Prevotella Daeguia SDRK01 Cloacibacterium Robertmurraya Burkholderia Bog-532 Paenibacillus_I Brevibacillus_D Caldalkalibacillus Sphingobacterium Achromobacter Neisseria Anaerococcus Stenotrophomonas_A Palsa-1150 Kaistia Methylobacterium"

    for genus in $list
    do 
    gtt-get-accessions-from-GTDB \
    -t $genus \
    --do-not-check-GTDB-version \
    --RefSeq-representatives-only
    done

    cat *RefSeq-rep-accs.txt > combined_RefSeq-rep-accs.txt

    bit-dl-ncbi-assemblies \
    -w combined_RefSeq-rep-accs.txt \
    -f protein -j 4

    mv *faa.gz genome_files/
    rm ncbi_assembly_info.tsv
    ```

### Identify Hydrogenase Genes

32. **Identify hydrogenase genes from the representative genomes:**
        
    ```bash
    cd /project/def-careg421/ruizhang/CNL/GTDB_rep_genomes/genome_files
    zgrep "\-hydrogenase" *
    ```


### result
out of 873 genomes, 57 genomes contain more than 1 hydrogenase annotations


	GCA_000318135.1.faa.gz:>EKX87396.1 ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium durum F0235]
	GCA_000411375.1.faa.gz:>EPD69116.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium pyruviciproducens ATCC BAA-1742]
	GCA_000550805.1.faa.gz:>AHI22492.1 Ni/Fe-hydrogenase B-type cytochrome subunit [Corynebacterium vitaeruminis DSM 20294]
	GCA_001020985.1.faa.gz:>AKK05087.1 Ni,Fe-hydrogenase I large subunit [Corynebacterium mustelae]
	GCA_001020985.1.faa.gz:>AKK05088.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium mustelae]
	GCA_001021025.1.faa.gz:>AKK02509.1 Ni,Fe-hydrogenase I large subunit [Corynebacterium epidermidicanis]
	GCA_001021025.1.faa.gz:>AKK02510.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium epidermidicanis]
	GCA_001274895.1.faa.gz:>ALA66824.1 Ni/Fe-hydrogenase B-type cytochrome subunit [Corynebacterium lactis RW2-5]
	GCA_001941445.1.faa.gz:>APT84023.1 Ni/Fe-hydrogenase B-type cytochrome subunit [Corynebacterium aquilae DSM 44791]
	GCA_002028325.1.faa.gz:>AQY66208.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Pseudomonas veronii]
	GCA_002797535.1.faa.gz:>PJJ64406.1 Ni/Fe-hydrogenase subunit HybB-like protein [Chryseobacterium geocarposphaerae]
	GCA_002812985.1.faa.gz:>PJC91310.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas lusitana]
	GCA_002906925.1.faa.gz:>POG22288.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas bestiarum]
	GCA_003054045.1.faa.gz:>PTX09809.1 Ni,Fe-hydrogenase I cytochrome b subunit [Sphingobacterium faecium]
	GCA_003634775.1.faa.gz:>RKS96436.1 Ni/Fe-hydrogenase subunit HybB-like protein [Chryseobacterium defluvii]
	GCA_003641245.1.faa.gz:>AYJ33693.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium xerosis]
	GCA_003813965.1.faa.gz:>AZA14345.1 putative Ni/Fe-hydrogenase B-type cytochrome subunit [Corynebacterium choanae]
	GCA_003814005.1.faa.gz:>AZA10296.1 putative Ni/Fe-hydrogenase 1 B-type cytochrome subunit [Corynebacterium pseudopelargi]
	GCA_004114895.1.faa.gz:>QAU53422.1 putative Ni/Fe-hydrogenase 1 B-type cytochrome subunit [Corynebacterium pelargi]
	GCA_006716485.1.faa.gz:>TQM19033.1 Ni/Fe-hydrogenase subunit HybB-like protein [Chryseobacterium aquifrigidense]
	GCA_007859215.1.faa.gz:>TWT26809.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium canis]
	GCA_008693705.1.faa.gz:>QET78674.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas veronii]
	GCA_009734385.1.faa.gz:>QGU02711.1 putative Ni/Fe-hydrogenase 1 B-type cytochrome subunit [Corynebacterium kalinowskii]
	GCA_010974825.1.faa.gz:>NEX89228.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas rivipollensis]
	GCA_014117265.1.faa.gz:>QMV85960.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium hindlerae]
	GCA_014138435.1.faa.gz:>MBA9065533.1 Ni/Fe-hydrogenase subunit HybB-like protein [Methylobacterium fujisawaense]
	GCA_014169735.1.faa.gz:>BBT66182.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas caviae]
	GCA_014645495.1.faa.gz:>GGP02788.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Cloacibacterium rupense]
	GCA_014892695.1.faa.gz:>QFI55906.1 [NiFe]-hydrogenase assembly chaperone HybE [Aeromonas simiae]
	GCA_014892695.1.faa.gz:>QFI55908.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas simiae]
	GCA_016026615.1.faa.gz:>QPR55969.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas allosaccharophila]
	GCA_016127195.1.faa.gz:>QQB19845.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas jandaei]
	GCA_016728645.1.faa.gz:>QQU88962.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium glucuronolyticum]
	GCA_016728725.1.faa.gz:>QQU97661.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium amycolatum]
	GCA_017310215.1.faa.gz:>QSR81651.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas hydrophila]
	GCA_017876455.1.faa.gz:>MBP2331681.1 Ni/Fe-hydrogenase b-type cytochrome subunit [Corynebacterium freneyi]
	GCA_020341435.1.faa.gz:>UCA09726.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas enteropelogenes]
	GCA_020405345.1.faa.gz:>UCM51643.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas dhakensis]
	GCA_020423125.1.faa.gz:>UCP13422.1 Ni/Fe-hydrogenase cytochrome b subunit [Aeromonas media]
	GCA_021568315.1.faa.gz:>MCF4007184.1 Ni/Fe-hydrogenase, b-type cytochrome subunit [Corynebacterium uropygiale]
	GCA_900100075.1.faa.gz:>SDI62807.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium jejuense]
	GCA_900100495.1.faa.gz:>SDI03518.1 Ni,Fe-hydrogenase III large subunit [Pseudomonas benzenivorans]
	GCA_900100495.1.faa.gz:>SDI03551.1 Ni,Fe-hydrogenase III small subunit [Pseudomonas benzenivorans]
	GCA_900102035.1.faa.gz:>SDE93593.1 Ni/Fe-hydrogenase 1 B-type cytochrome subunit [Pseudomonas extremaustralis]
	GCA_900111495.1.faa.gz:>SEW42060.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium wanjuense]
	GCA_900113845.1.faa.gz:>SFI99762.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Methylobacterium brachiatum]
	GCA_900114535.1.faa.gz:>SFL24252.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Methylobacterium pseudosasicola]
	GCA_900116225.1.faa.gz:>SFS44807.1 Ni,Fe-hydrogenase I cytochrome b subunit [Sphingobacterium wenxiniae]
	GCA_900129245.1.faa.gz:>SHF77756.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium arachidis]
	GCA_900129755.1.faa.gz:>SHH03380.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium oranimense]
	GCA_900142445.1.faa.gz:>SHL39284.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium polytrichastri]
	GCA_900142615.1.faa.gz:>SHK84768.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium contaminans]
	GCA_900143185.1.faa.gz:>SIO19655.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium scophthalmum]
	GCA_900156685.1.faa.gz:>SIS51607.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium piscicola]
	GCA_900156735.1.faa.gz:>SIS94412.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium ureilyticum]
	GCA_900156825.1.faa.gz:>SIT12811.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium gambrini]
	GCA_900182655.1.faa.gz:>SMO53411.1 Ni/Fe-hydrogenase 2 integral membrane subunit HybB [Chryseobacterium rhizoplanae]
	GCA_900205605.1.faa.gz:>SNW30465.1 putative Ni/Fe-hydrogenase B-type cytochrome subunit [Corynebacterium belfantii]
	GCA_900240005.1.faa.gz:>VBB11127.1 formate dehydrogenase-O subunit gamma,Uncharacterized secreted protein,Ni/Fe-hydrogenase, b-type cytochrome subunit,Cytochrome b(N-terminal)/b6/petB [Burkholderia stabilis]
	GCA_900240005.1.faa.gz:>VBB13231.1 NADH-quinone oxidoreductase chain 5,NADH dehydrogenase subunit C,Ni,Fe-hydrogenase III component G,NADH (or F420H2) dehydrogenase, subunit C,Respiratory-chain NADH dehydrogenase, 30 Kd subunit [Burkholderia stabilis]
	GCA_900478035.1.faa.gz:>SQI26419.1 Ni/Fe-hydrogenase B-type cytochrome subunit [Corynebacterium renale]
	GCA_900496975.1.faa.gz:>SSW70895.1 putative Ni/Fe-hydrogenase B-type cytochrome subunit [Achromobacter veterisilvae]
	GCA_900496975.1.faa.gz:>SSW72463.1 putative Ni/Fe-hydrogenase B-type cytochrome subunit [Achromobacter veterisilvae]

### Data
All metadata and visualization files generated in this pipeline can be found in the data folder: [https://github.com/rzhan186/CNL_halite/data](https://github.com/rzhan186/CNL_halite/data)


### Support
For any questions about the this bioinformatic pipeline, please contact:

Rui Zhang [rui.zhang@uottawa.ca]() <br>
Biology Department, University of Ottawa

