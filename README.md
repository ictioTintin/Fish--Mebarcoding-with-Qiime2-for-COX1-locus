# Metabarcoding in Fish with Qiime2 for the COX1 Locus

_Martin Holguin Osorio_  
_June 2021_  
_Version 3_  

## Creating the Metadata File and Preparing the Workspace
The file `metadata.tsv` must follow a specific order and organization to work correctly. More info in the [Qiime2 metadata tutorial](https://docs.qiime2.org/2020.11/tutorials/metadata/). It is recommended to use the Google Sheets add-on **keemei** to create the `metadata.tsv` file quickly and easily. More [info here](https://keemei.qiime2.org).

* After downloading `metadata.tsv` from Google Sheets:

```bash
# open qiime2 in a terminal
conda activate qiime2-2021.8

# navigate to the working directory where metadata is stored (in this case /home/martin/eDNA/1 )
cd /home/martin/eDNA/6/COI

# create a folder "datos" to store raw data reads
mkdir datos

# create a folder for process outputs (results)
mkdir salidas
```

## Data Import
After reviewing the nature of the data (demultiplexed, paired-end sequencing with barcodes included in the sequences), I determined that another command is required for data import. I left the raw data filenames unchanged and placed them inside the `datos` folder.

```bash
#import data into qiime2
qiime tools import   --type 'SampleData[PairedEndSequencesWithQuality]'   --input-path datos/   --input-format CasavaOneEightSingleLanePerSampleDirFmt   --output-path salidas/multiplexed-seqs.qza
```

## Demultiplexing
Separate information from each sample contained in the FASTQ files using the reference barcodes provided in the `metadata.tsv` file.

```bash
#demultiplex
qiime cutadapt demux-paired   --i-seqs salidas/multiplexed-seqs.qza   --m-forward-barcodes-file metadata.tsv   --m-forward-barcodes-column BarcodeSequence   --p-error-rate 0   --o-per-sample-sequences salidas/demultiplexed-seqs.qza   --o-untrimmed-sequences salidas/untrimmed.qza   --verbose
  
#create summary of demultiplexing and visualization
qiime demux summarize   --i-data salidas/demultiplexed-seqs.qza   --o-visualization salidas/demultiplexed-seqs.qzv

#open visualization in browser
qiime tools view salidas/demultiplexed-seqs.qzv
# the file "untrimmed.qza" contains all unassigned sequences
```

## Denoising with DADA2
This step denoises the data, removes chimeras, and outputs ASVs.

```bash
qiime dada2 denoise-paired   --i-demultiplexed-seqs salidas/demultiplexed-seqs.qza   --p-trunc-len-f 240   --p-trunc-len-r 200   --o-table salidas/table.qza   --o-representative-sequences salidas/rep-seqs.qza   --o-denoising-stats salidas/denoising-stats.qza
```

## Taxonomic Assignment
Assign taxonomy to ASVs using a reference database.

```bash
qiime feature-classifier classify-consensus-vsearch   --i-query salidas/rep-seqs.qza   --i-reference-reads ref_seqs.qza   --i-reference-taxonomy ref_tax.qza   --p-perc-identity 0.97   --o-classification salidas/taxonomy.qza
```

## Phylogeny
Build a phylogenetic tree for downstream diversity analysis.

```bash
qiime phylogeny align-to-tree-mafft-fasttree   --i-sequences salidas/rep-seqs.qza   --o-alignment salidas/aligned-rep-seqs.qza   --o-masked-alignment salidas/masked-aligned-rep-seqs.qza   --o-tree salidas/unrooted-tree.qza   --o-rooted-tree salidas/rooted-tree.qza
```

## Filtering
Filter the feature table and sequences based on metadata or taxonomy.

```bash
qiime taxa filter-table   --i-table salidas/table.qza   --i-taxonomy salidas/taxonomy.qza   --p-exclude mitochondria,chloroplast   --o-filtered-table salidas/table-no-mitochondria-chloroplast.qza
```

## Diversity Analyses
Perform alpha and beta diversity analyses.

```bash
qiime diversity core-metrics-phylogenetic   --i-phylogeny salidas/rooted-tree.qza   --i-table salidas/table-no-mitochondria-chloroplast.qza   --p-sampling-depth 10000   --m-metadata-file metadata.tsv   --output-dir salidas/core-metrics-results
```

## Exporting Results
Export tables and sequences for external analysis.

```bash
qiime tools export   --input-path salidas/table.qza   --output-path exported-data

qiime tools export   --input-path salidas/taxonomy.qza   --output-path exported-data
```

---
