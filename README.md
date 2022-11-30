---
title: "Long-read transcriptomics workshop"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)

# Install needed packages
if (!requireNamespace("dplyr", quietly = TRUE))
  install.packages("dplyr")
if (!requireNamespace("ggplot2", quietly = TRUE))
  install.packages("ggplot2")
if (!requireNamespace("gridExtra", quietly = TRUE))
  install.packages("gridExtra")
if (!requireNamespace("devtools", quietly = TRUE))
  install.packages("devtools")
if (!requireNamespace("bio3d", quietly = TRUE))
  install.packages("bio3d")
#devtools::install_github("dzhang32/ggtranscript")

# BiocManager packages
if (!requireNamespace("BiocManager", quietly = TRUE))
  install.packages("BiocManager")
if (!requireNamespace("rtracklayer", quietly = TRUE))
  BiocManager::install("rtracklayer")


# Load needed packages
library(dplyr)
library(rtracklayer)
library(ggplot2)
library(gridExtra)
library(bio3d)
library(Biostrings)
library(stringr)
#library(tidyr)
library(ggtranscript)

# library(magrittr)
```

# FILES (LOAD TO IGV)

"sample1.isoseq.duplicate_1/sample1.clustered.hq.aligned.bam"
-LR-BAM produced by isoseq pipeline. Contains "high-quality" reads, prior to SQANTI3 filtering

"reference/gencode.v38.annotation.sorted.gtf"
-load into this notebook and IGV (.idx file provided)

"sample1.sqanti3.duplicate_1/sample1.transcriptome.sqanti3_classification.filtered_lite.sorted.gtf"
-custom GTF generated by SQANTI, can be loaded into IGV (.idx file provided)

"Aligned.out.sorted.subset.bam"
-BAM file containing only selected regions specified in "ensGTF.genes.filt.bed"
-smaller, so easier to download 


```{r}
# REFERENCE
ensGTF <- rtracklayer::import(con=file.path(getwd(), "reference/gencode.v38.annotation.sorted.gtf"), format="gtf")
ensGTF.df <- as(ensGTF, "data.frame")
ensGTF.tx <- ensGTF.df[ensGTF.df$type == "transcript",]
ensGTF.tx <- ensGTF.tx[c("gene_id", "transcript_id", "gene_name", "transcript_type")]

# count of the # of transcripts (ENST IDs) for each ENSG
ensGTF.tx.summ <-  ensGTF.tx %>%
  count(gene_id)
hist(ensGTF.tx.summ$n, breaks="FD", xlim=c(1,40) )
```

# Load SQANTI3 output
```{r}
#SQANTI
sqanti_fields <- c("isoform", "associated_gene", "associated_transcript", "exons", "structural_category", "FL", "subcategory", "CDS_length")

sqanti <- read.csv(file=file.path(getwd(),"sample1.sqanti3.duplicate_1", "sample1.transcriptome.sqanti3_classification.filtered_lite_classification.txt"), sep="\t", header=T)[sqanti_fields]

# Select for novel transcripts
sqanti.novel <- (sqanti %>% 
                   filter(associated_transcript == "novel") %>%
                   mutate(associated_transcript=isoform))[,c("associated_transcript", "FL", "associated_gene")]
colnames(sqanti.novel) <- c("associated_transcript", "FL", "gene_name")

# Remove novel transcripts, Count FL per-transcript ( known/associated transcripts)
sqanti.known.FL_counts <- sqanti[grep(sqanti$associated_transcript, pattern= "ENST"),] %>%
  dplyr::group_by(associated_transcript) %>% 
  dplyr::summarise(FL = sum(FL), gene_name=unique(associated_gene))

# Combine known and novel transcript dfs
sqanti.total <- rbind(sqanti.known.FL_counts, sqanti.novel)
```

# Explore genes with many isoforms
# Some interesting genes: MCM7, TAF6, CTSD--> misquantification?
# ACTG1 -->
```{r}
num <- 7
iso.counts <- table(sqanti$associated_gene)
iso.counts[iso.counts > num]
```

# Generates bed file (subsets BAM to make it a more manageable size)
# Can also download full BAM file if needed
samtools view -b -L ../ensGTF.genes.filt.bed Aligned.out.sorted.bam > Aligned.out.sorted.subset.bam

```{r}
if(FALSE){
  ensGTF.genes <- ensGTF.df[ensGTF.df$type == "gene",]
  ensGTF.genes.filt <- ensGTF.genes %>% filter(gene_name %in% names(iso.counts[iso.counts > num]) )
  ensGTF.genes.filt.gr <- makeGRangesFromDataFrame(ensGTF.genes.filt)
  export.bed(ensGTF.genes.filt.gr,con=file.path(getwd(), 'ensGTF.genes.filt.bed') )
}
```


# Exercise: find examples of the different sources of transcript variation
- Choose some of the genes above to explore in IGV, create a powerpoint containing visualizations of the different forms of transcript variation not seen in the reference transcriptome
-we explored this in the introductory slides (including all exons, no introns and the complete UTRs), alternative TSSs (transcription start sites) and TTSs (transcription termination sites))


# Prepare/load quantification tool output files
```{r}
#KALLISTO
kallisto <- read.csv(file.path(getwd(), "kallisto-rsem-NOV-23-2022", "abundance_transcripts.tsv"), header =T, sep="\t")[c("target_id", "tpm")]
names(kallisto) <- c("tx_id", "tpm_kallisto")
# remove all except ENST
kallisto$tx_id <- sapply(strsplit(as.character(kallisto$tx_id), "\\|"), "[[", 1)

# RSEM
rsem <- read.csv(file.path(getwd(), "kallisto-rsem-NOV-23-2022", "rsem.isoforms.results"), header =T, sep="\t" )[c("transcript_id", "TPM")]
names(rsem) <- c("tx_id", "tpm_rsem")

# Keep only ENSTs which have LR counts
joined <- inner_join(sqanti.known.FL_counts, kallisto, by=c("associated_transcript"="tx_id") ) %>% inner_join(rsem, by=c("associated_transcript"="tx_id")) 
#joined.filt <- joined %>% filter(gene_name %in% names(iso.counts[iso.counts > num]) )

# Keep all transcripts, including novel ones
joined.full <- full_join(sqanti.total, kallisto, by=c("associated_transcript"="tx_id") ) %>% full_join(rsem, by=c("associated_transcript"="tx_id")) 
tmp <- full_join(sqanti.total, kallisto, by=c("associated_transcript"="tx_id") ) %>% full_join(rsem, by=c("associated_transcript"="tx_id")) 

# Add gene names to ENST IDs without them --> Makes investigation challenging for novel txs
idxs <- which(is.na(joined.full$gene_name))
df <- inner_join(data.frame(transcript_id=joined.full[idxs,]$associated_transcript ), ensGTF.tx[c("transcript_id", "gene_name")])
joined.full.all_txs <- joined.full
joined.full.all_txs$gene_name[idxs] <- df$gene_name

# Filter for genes with many isoforms
joined.full.filt <- joined.full %>% filter(gene_name %in% names(iso.counts[iso.counts > num]) )
```

#SCATTERPLOTS : LR FL counts versus SR TPMs
-using "joined" object, containing only known transcripts which also have FL counts
```{r}
scatter <- function(title="", joined, x, y, alpha=0.05, pseudo=1e-4, sample, ylim, 
                              xlab="tool prediction", ylab="FL LR count"){
  text <- paste0("R=", round(cor(log(x + pseudo), log(y+ pseudo), method='pearson'), 6), ", RMSD: ", bio3d::rmsd(log(x + pseudo), log(y + pseudo))) 
  annotations <- data.frame(
    xpos = c(Inf), ypos =  c(-Inf),
    annotateText = c(text),
    hjustvar = c(1),
    vjustvar = c(-4))
  
  p <- ggplot(joined, aes(x=log(x + pseudo), y=log(y + pseudo))) +   
    geom_point(alpha=alpha, colour='DarkBlue') + 
    labs(x = xlab, y = ylab, title = title) +
    theme(axis.title = element_text(size=10), legend.position="none")  +
    geom_text(data = annotations, colour='Black', aes(x = xpos, y = ypos, 
                                      hjust = hjustvar, vjust = vjustvar, label = annotateText)) 
  return(p)
}

p.kallisto <- scatter(joined=joined, x=joined$tpm_kallisto, y=joined$FL, xlab="kallisto log(tpm)", ylab="log(FL counts)")
p.kallisto

p.rsem <- scatter(joined=joined, x=joined$tpm_rsem, y=joined$FL, xlab="rsem log(tpm)", ylab="log(FL counts)")
p.rsem

```
# Exercise: 
What do you notice about these scatterplots? What might be happening with the outliers? 

# Exercise: 
Explore cases of quantification discrepancy among the 3 tools (look at "joined.full" for all genes, and "joined.full.filt" for selected genes with many isoforms to narrow down the search. 
Especially interesting are cases where novel transcripts have high FL counts

# EXPLORE NOVEL PACBIO GTF
```{r}
# SET OUR GENE OF INTEREST
gene <- "MCM7"
gene <- "TAF6"
gene <- "ACTG1"
gene <- "RAE1"

plot_txs <- function(gene, txs=c(), gtf.df){
  
  gtf.df.gene <- gtf.df %>% filter(gene_name == gene)
  
  if(length(txs) > 0){
    gtf.df.gene <- gtf.df %>% filter(gene_name == gene) %>% filter(transcript_id %in% txs)
  }
  # extract exons
  exons <- gtf.df.gene %>% dplyr::filter(type == "exon")
  
  p <- exons %>%
    ggplot(aes(
      xstart = start,
      xend = end,
      y = transcript_id
    )) +
    geom_intron(
      data = to_intron(exons, "transcript_name"),
      aes(strand = strand)
    )
  #p <- p +  geom_range(aes(fill = transcript_type)) 
  p <- p +  geom_range(fill="yellow") 
  return(p)
}

sqanti.gtf <- rtracklayer::import(con=file.path(getwd(), "sample1.sqanti3.duplicate_1", "sample1.transcriptome.sqanti3_classification.filtered_lite.gtf" ), format="gtf")
# join sqanti gtf to annotations
sqanti.gtf.df <- as(sqanti.gtf, "data.frame")
sqanti.gtf.df.joined <- inner_join(sqanti.gtf.df, sqanti, by=c("transcript_id"="isoform"))

# Add in fields needed for plot_txs function to work
# replace "transcript_id" column with "associated_transcript" only if 
sqanti.gtf.df.joined<- sqanti.gtf.df.joined %>% mutate(transcript_id=paste(transcript_id, associated_transcript, sep="_") )
sqanti.gtf.df.joined$gene_name <- sqanti.gtf.df.joined$associated_gene
sqanti.gtf.df.joined$transcript_name <- sqanti.gtf.df.joined$transcript_id
sqanti.gtf.df.joined$transcript_type <- sqanti.gtf.df.joined$structural_category

# keep only "transcript" entries 
sqanti.gtf.tx <- sqanti.gtf.df.joined %>% filter(type == "transcript")

# find common columns between gtf files
cols <- intersect(colnames(ensGTF.df), colnames(sqanti.gtf.df.joined))
gtf.combined <- rbind(sqanti.gtf.df.joined[cols], ensGTF.df[cols])

# PLOT ALL TRANCRIPTS

# SQANTI gtf
gtf <- gtf.combined
print(plot_txs(gene=gene, gtf=gtf))
gtf.filt <- gtf %>% filter(gene_name == gene & type == "transcript")
dput(gtf.filt$transcript_id)

# !! PB.4883.13 !! -- > combination_of_known_junctions

# PLOT SUBSET OF TRANSCRIPTS
# can subset transcripts to isolate cases of interest
# In this case, I removed "longer" transcripts so the shorter ones can be seen more easily

# ACTG1
txs <- c("PB.4883.10_ENST00000680727.1", "PB.4883.12_ENST00000680727.1", 
"PB.4883.7_ENST00000576209.5", "PB.4883.9_ENST00000680727.1", 
"PB.4883.8_novel", "PB.4883.5_ENST00000576209.5", "PB.4883.6_ENST00000680727.1", 
"PB.4883.2_ENST00000575842.5", "PB.4883.1_ENST00000573283.7", 
"PB.4883.3_ENST00000679410.1", "PB.4883.4_novel", "PB.4883.13_novel", 
 "ENST00000576544.6", "ENST00000573283.7", 
"ENST00000681842.1", "ENST00000679480.1", "ENST00000570382.2", 
 "ENST00000680727.1", "ENST00000680227.1", 
"ENST00000681092.1", "ENST00000574671.6", "ENST00000576214.3", 
"ENST00000679410.1",  "ENST00000572105.7", 
"ENST00000679535.1", "ENST00000575842.5", "ENST00000576917.5", 
"ENST00000576209.5", "ENST00000575087.5", "ENST00000644774.2", 
"ENST00000571721.6", "ENST00000571691.6")

# RAE1
txs <- c("PB.5459.1_ENST00000371242.6", "PB.5459.3_novel", "PB.5459.2_ENST00000395841.7", 
"PB.5459.4_ENST00000395841.7", "PB.5459.5_novel", "ENST00000371242.6", 
"ENST00000527947.5", "ENST00000395841.7", "ENST00000492498.5", 
"ENST00000411894.5", "ENST00000429339.5", "ENST00000395840.6", 
"ENST00000452119.1", "ENST00000462438.1")
print(plot_txs(gene=gene, txs=txs, gtf=gtf))

```
# Exercise: 
Upon creating an interesting visual, compare to IGV. 
How does the read coverage look?



