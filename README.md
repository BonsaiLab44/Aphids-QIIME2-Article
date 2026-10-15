# QIIME 2 Amplicon Analysis Workflow

This README provides a general QIIME 2 workflow for single-end amplicon sequencing data, including import, demultiplexing summary, DADA2 denoising, taxonomy assignment, filtering, diversity analysis, PERMANOVA/PERMDISP, PICRUSt2, FUNGuild, and Spearman correlation analysis.

The provided script is compatible with qiime2-amplicon-2024.2 used in the main *Pentalonia nigronervosa* microbiome project using 16S rDNA and ITS amplicon sequencing; options for qiime2-amplicon-2026.1 are also provided. 

Please note that the codes and scripts provided are general templates. Directory paths, input file names, output file names, thread numbers, and additional code features should be modified according to the specific needs of each analysis.

---

## 1. Create Manifest File

Place all compressed FASTQ files in the working directory.

```bash
echo -e "sample-id\tabsolute-filepath" > manifest.tsv

for x in *.gz; do
  sample_id=$(echo "$x" | cut -f 1 -d "_")
  echo -e "Sample.${sample_id}\t$PWD/$x"
done >> manifest.tsv
```

---

## 2. Import Single-End Sequences

```bash
qiime tools import \
  --input-path manifest.tsv \
  --type 'SampleData[SequencesWithQuality]' \
  --input-format SingleEndFastqManifestPhred33V2 \
  --output-path demux-single-end.qza
```

---

## 3. Summarize Demultiplexed Sequences

```bash
qiime demux summarize \
  --i-data demux-single-end.qza \
  --o-visualization demux-single-end.qzv
```

```bash
qiime tools view demux-single-end.qzv
```

---

## 4. DADA2 Denoising

### For QIIME 2 2024.2

```bash
qiime dada2 denoise-single \
  --i-demultiplexed-seqs demux-single-end.qza \
  --p-trim-left 0 \
  --p-trunc-len 0 \
  --o-representative-sequences rep-seqs-dada2.qza \
  --o-table table-dada2.qza \
  --o-denoising-stats denoising-stats.qza
```
### For QIIME 2 2026.1 or newer

```bash
qiime dada2 denoise-single \
  --i-demultiplexed-seqs demux-single-end.qza \
  --p-trim-left 0 \
  --p-trunc-len 430 \
  --o-representative-sequences rep-seqs-dada2.qza \
  --o-table table-dada2.qza \
  --o-denoising-stats denoising-stats.qza \
  --o-base-transition-stats base-transition-stats.qza

---
```
## 5. Visualize DADA2 Statistics

```bash
qiime metadata tabulate \
  --m-input-file denoising-stats.qza \
  --o-visualization denoising-stats.qzv
```

---

## 6. Summarize Feature Table

### For QIIME 2 2024.2

```bash
qiime feature-table summarize \
  --i-table table-dada2.qza \
  --o-visualization table-dada2.qzv
```

---

### For QIIME 2 2026.1 or newer
```bash
qiime feature-table summarize \
  --i-table table-dada2.qza \
  --o-feature-frequencies feature-frequencies.qza \
  --o-sample-frequencies sample-frequencies.qza \
  --o-summary table-dada2.qzv

```

---
## 7. Visualize Representative Sequences

```bash
qiime feature-table tabulate-seqs \
  --i-data rep-seqs-dada2.qza \
  --o-visualization rep-seqs-dada2.qzv
```

---

## 8. Generate Phylogenetic Tree

```bash
qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences rep-seqs-dada2.qza \
  --o-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned-rep-seqs.qza \
  --o-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza
```

---

## 9. Taxonomic Classification

```bash
qiime feature-classifier classify-sklearn \
  --i-classifier classifier.qza \
  --i-reads rep-seqs-dada2.qza \
  --o-classification taxonomy.qza
```

---

## 10. Visualize Taxonomy Table

```bash
qiime metadata tabulate \
  --m-input-file taxonomy.qza \
  --o-visualization taxonomy.qzv
```

---

## 11. Taxa Barplot

```bash
qiime taxa barplot \
  --i-table table-dada2.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization taxa-barplot.qzv
```

---

## 12. Filter Table by Taxonomy

This step removes mitochondria, chloroplasts, eukaryotic sequences, and unassigned features.

```bash
qiime taxa filter-table \
  --i-table table-dada2.qza \
  --i-taxonomy taxonomy.qza \
  --p-exclude "mitochondria,Mitochondria,chloroplast,Chloroplast,Eukaryota,d__Eukaryota,k__Eukaryota,Unassigned,unassigned" \
  --o-filtered-table filtered-table.qza \
  --verbose
```

---


## 13. Visualize Filtered Feature Table

### For QIIME 2 2024.2

```bash
qiime feature-table summarize \
  --i-table filtered-table.qza \
  --o-visualization filtered-table.qzv
```

---

### For QIIME 2 2026.1 or newer
```bash
qiime feature-table summarize \
  --i-table filtered-table.qza \
  --o-feature-frequencies filtered-feature-frequencies.qza \
  --o-sample-frequencies filtered-sample-frequencies.qza \
  --o-summary filtered-table.qzv

```

---
## 14. Filtered Taxa Barplot

```bash
qiime taxa barplot \
  --i-table table-dada2.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization taxa-barplot.qzv
```

## 15. Alpha Rarefaction

Without phylogeny:

```bash
qiime diversity alpha-rarefaction \
  --i-table filtered-table.qza \
  --p-max-depth 0 \
  --m-metadata-file metadata.tsv \
  --o-visualization alpha-rarefaction.qzv
```

With phylogeny:

```bash
qiime diversity alpha-rarefaction \
  --i-table filtered-table.qza \
  --i-phylogeny rooted-tree.qza \
  --p-max-depth 0 \
  --m-metadata-file metadata.tsv \
  --o-visualization alpha-rarefaction-phylogenetic.qzv \
  --verbose
```

---

## 16. Core Diversity Metrics

```bash
qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree.qza \
  --i-table filtered-table.qza \
  --p-sampling-depth 0 \
  --m-metadata-file metadata.tsv \
  --output-dir core-metrics-results
```

---

## 17. Alpha Diversity Group Significance

### Faith’s Phylogenetic Diversity

```bash
qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results/faith_pd_vector.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization faith-pd-group-significance.qzv
```

### Shannon Diversity

```bash
qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results/shannon_vector.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization shannon-group-significance.qzv
```

### Observed Features

```bash
qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results/observed_features_vector.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization observed-features-group-significance.qzv
```

---

## 18. Beta Diversity Significance: PERMANOVA

Replace `metadata-column-name` with the column of interest from the metadata file.

### Bray-Curtis PERMANOVA

```bash
qiime diversity beta-group-significance \
  --i-distance-matrix core-metrics-results/bray_curtis_distance_matrix.qza \
  --m-metadata-file metadata.tsv \
  --m-metadata-column metadata-column-name \
  --p-method permanova \
  --p-permutations 999 \
  --o-visualization bray-curtis-permanova-999.qzv
```

### Jaccard PERMANOVA

```bash
qiime diversity beta-group-significance \
  --i-distance-matrix core-metrics-results/jaccard_distance_matrix.qza \
  --m-metadata-file metadata.tsv \
  --m-metadata-column metadata-column-name \
  --p-method permanova \
  --p-permutations 999 \
  --o-visualization jaccard-permanova-999.qzv
```

### Weighted UniFrac PERMANOVA

```bash
qiime diversity beta-group-significance \
  --i-distance-matrix core-metrics-results/weighted_unifrac_distance_matrix.qza \
  --m-metadata-file metadata.tsv \
  --m-metadata-column metadata-column-name \
  --p-method permanova \
  --p-permutations 999 \
  --o-visualization weighted-unifrac-permanova-999.qzv
```

### Unweighted UniFrac PERMANOVA

```bash
qiime diversity beta-group-significance \
  --i-distance-matrix core-metrics-results/unweighted_unifrac_distance_matrix.qza \
  --m-metadata-file metadata.tsv \
  --m-metadata-column metadata-column-name \
  --p-method permanova \
  --p-permutations 999 \
  --o-visualization unweighted-unifrac-permanova-999.qzv
```

---

## 19. Beta Diversity Dispersion: PERMDISP

```bash
qiime diversity beta-group-significance \
  --i-distance-matrix core-metrics-results/bray_curtis_distance_matrix.qza \
  --m-metadata-file metadata.tsv \
  --m-metadata-column metadata-column-name \
  --p-method permdisp \
  --p-permutations 999 \
  --o-visualization bray-curtis-permdisp-999.qzv
```

---

## 20. Pairwise Beta Diversity Significance

```bash
qiime diversity beta-group-significance \
  --i-distance-matrix core-metrics-results/bray_curtis_distance_matrix.qza \
  --m-metadata-file metadata.tsv \
  --m-metadata-column metadata-column-name \
  --p-method permanova \
  --p-pairwise \
  --p-permutations 999 \
  --o-visualization bray-curtis-pairwise-permanova-999.qzv
```

---

# PICRUSt2 Functional Prediction

## 21. Run PICRUSt2 Full Pipeline

```bash
qiime picrust2 full-pipeline \
  --i-table filtered-table.qza \
  --i-seq filtered-rep-seqs.qza \
  --output-dir picrust2-output \
  --p-placement-tool epa-ng \
  --p-threads 4 \
  --p-hsp-method mp \
  --p-max-nsti 2 \
  --verbose
```

---

## 22. Visualize PICRUSt2 Pathway Abundance

### For QIIME 2 2024.2

```bash
qiime feature-table summarize \
  --i-table picrust2-output/pathway_abundance.qza \
  --o-visualization picrust2-output/pathway-abundance.qzv
```
---

## 23. Convert PICRUSt2 Pathway Abundance to Relative Frequency

```bash
qiime feature-table relative-frequency \
  --i-table picrust2-output/pathway_abundance.qza \
  --o-relative-frequency-table picrust2-output/pathway-abundance-relative-frequency.qza
```

---

## 24. Export PICRUSt2 Relative Frequency Table

```bash
qiime tools export \
  --input-path picrust2-output/pathway-abundance-relative-frequency.qza \
  --output-path picrust2-exported
```

---

## 25. Convert BIOM Table to TSV

```bash
biom convert \
  -i picrust2-exported/feature-table.biom \
  -o pathway-abundance-relative-frequency.tsv \
  --to-tsv
```

---

## 26. FUNGuild
### Info Funguild

```bash
https://github.com/UMNFuN/FUNGuild?tab=readme-ov-file
```

### Run FUNGuild

```bash
python Guilds_v1.1.py \
  -otu funguild-input.csv \
  -db fungi \
  -m \
  -u
```

---

## 27. Spearman Correlation Analysis in R

## 28. Required R Packages

```r
library(tidyverse)
library(pheatmap)
```

---

## 29. Spearman Correlation Heatmap

Input file should be a tab-delimited table with taxa as rows and samples as columns.

Example input file:

```text
Taxa    Sample_1    Sample_2    Sample_3
g__Taxon_A    10    20    5
g__Taxon_B    0     15    30
g__Taxon_C    8     3     12
```

Run the R code below:

```r
library(tidyverse)
library(pheatmap)

# Read abundance table
df <- read.delim(
  "abundance-table.tsv",
  header = TRUE,
  sep = "\t",
  check.names = FALSE,
  row.names = 1
)

# Define fungal taxa
fungi_names <- c(
  "g__Fungal_Taxon_1",
  "g__Fungal_Taxon_2",
  "g__Fungal_Taxon_3"
)

# Compute Spearman correlation matrix
cor_matrix <- cor(
  t(df),
  method = "spearman",
  use = "pairwise.complete.obs"
)

# Annotation: fungi vs bacteria
taxa_order <- colnames(cor_matrix)

group_vec <- ifelse(
  taxa_order %in% fungi_names,
  "Fungi",
  "Bacteria"
)

annotation <- data.frame(Group = group_vec)
rownames(annotation) <- taxa_order

annotation_colors <- list(
  Group = c(
    Fungi = "#e41a1c",
    Bacteria = "#377eb8"
  )
)

# Heatmap color palette
heat_colors <- colorRampPalette(
  c("#ffffe5", "#a1dab4", "#41b6c4", "#225ea8", "#0c2c84")
)(100)

# Plot heatmap
pheatmap(
  cor_matrix,
  color = heat_colors,
  breaks = seq(-1, 1, length.out = 101),
  legend_breaks = c(-1, 0, 1),
  legend_labels = c("-1", "0", "+1"),
  cluster_rows = TRUE,
  cluster_cols = TRUE,
  border_color = "grey80",
  show_colnames = TRUE,
  show_rownames = TRUE,
  angle_col = 45,
  fontsize = 9,
  fontsize_row = 9,
  fontsize_col = 9,
  annotation_col = annotation,
  annotation_row = annotation,
  annotation_colors = annotation_colors,
  main = "Spearman Correlation Among Selected Fungal and Bacterial Taxa"
)
```

---

# General Input Files

The workflow uses the following general input files:

```text
manifest.tsv
metadata.tsv
classifier.qza
demux-single-end.qza
table-dada2.qza
rep-seqs-dada2.qza
taxonomy.qza
filtered-table.qza
filtered-rep-seqs.qza
rooted-tree.qza
```

---

# General Output Files

The main outputs include:

```text
demux-single-end.qzv
table-dada2.qzv
rep-seqs-dada2.qzv
taxonomy.qzv
taxa-barplot.qzv
filtered-table.qza
filtered-rep-seqs.qza
alpha-rarefaction.qzv
core-metrics-results/
bray-curtis-permanova-999.qzv
bray-curtis-permdisp-999.qzv
picrust2-output/
pathway-abundance-relative-frequency.tsv
```
