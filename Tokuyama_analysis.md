Tokuyama Re-analysis
================
Luis P Iniguez
4/9/2019

### Tokuyama M. et al. 2018

Tokuyama et al. (2018) published ERVmap, an open-source code to analyze transcription of unique sets of human ERVs. The authors also provided a curated list of 3,220 ERV proviral loci, and they included ERV loci with unique chromosomal locations that had been described as autonomous/proviral ERVs. These ERVs were either transcribed in various disease contexts or identified as ERVs based on sequence analysis in silico. The authors proved their method in human cell lines, primary cell types, SLE patients and breast cancer tissues.

Regarding SLE, the authors obtained peripheral blood mononuclear cells (PBMCs) from female SLE patients (20) and healthy (6) females, performed RNA sequencing and analyzed the RNA-seq libraries with ERVmap. They found 124 ERVs that were significantly elevated in SLE patients’ PBMCs compared with healthy controls.

RNA-seq samples from SLE and healthy donors could be found at [NCBI](https://www.ncbi.nlm.nih.gov/sra?linkname=bioproject_sra_all&from_uid=505280). Regarding the samples authors said: "Whole blood collected in heparin tubes were centrifuged to obtained plasma, and the rest of the blood was used to isolate PBMCs using Ficoll-Paque density centrifugation separation \[...\] RNA was isolated according to manufacturer’s protocol (RNeasy kit; Qiagen)\[...\]High-throughput sequencing was performed on the RNA samples using HiSeq and NextSeq Illumina sequencing machines \[...\] Roughly 100 million reads were obtained for each sample using 150-bp pair-end reads". RNA-seq was enriched with polyA

Another important issue regarding the samples used is that the sequenced with two different sequencing machines, HiSeq and NextSeq.

| Sample  | HiSeq | NextSeq | Total |
|---------|-------|---------|-------|
| SLE     | 16    | 4       | 20    |
| Healthy | 2     | 4       | 6     |

The authors provide the ERV annotations and for the differential expression analysis they mention: "The read counts are normalized by the size factors obtained from the cellular genes of the same sample, calculated using the DESeq2 normalization method. \[...\] reads were mapped to human genome (GRCh38) by TopHat2 (and) counts of reads for each gene were based on Ensembl annotation. After the counts are collected, DEseq2 was used to calculate size factor for each sample. Normalized counts for genes and ERVs as well as DEseq2 outputs are provided in the [GEO database](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE122459)."

The authors do not provide the information necessary for reproducing their results, since raw gene counts are mandatory for DESeq2 analysis. They also do not mention which method from DESeq was used to calculate differential expression. I assumed the authors used default parameters: Wald-test without any adaptive shrinkage estimator and these parameters are used for all DE analysis. Another important issue is that ERVmap could be downloaded but there is no further information about installation, and I could not run it.

Using the information provided in the GEO database I tried to reproduce their figures:

Fig 3A:

![](Tokuyama_analysis_files/figure-markdown_github/Fig3A-1.png)

Figure 3C is a heatmap about the ERVs differentially expressed, but I could not replicate the figure 100%, for two main motives; first the distance and the hierarchical clustering methods parameters are not specified and second, it seems the "normalized counts" provided are indeed raw counts. I can not normalized the counts because I will need the raw Gene count, which is not provided (discussed below). Nevertheless, I did the heatmap with the raw count data provided.

Figure 3C:

![](Tokuyama_analysis_files/figure-markdown_github/Fig3C-1.png)

Analysis of Tokuyama samples.
=============================

All the samples were analyzed with the same methodology. It consists of a mapping all reads with Bowtie2 followed by Telescope for HERV expression quantification. The annotations were made by Matthew Bendall and can be found [here](https://github.com/mlbendall/telescope_annotation_db/tree/master/builds). It is worth to mention that for further analysis the column of *final\_conf* was used for HERV read count. For gene quantification [kallisto](https://pachterlab.github.io/kallisto/about) was used and the gene annotations were pulled down from UCSC. More info about the gene annotations done by Matthew could be found [here](https://github.com/gwcbi/cbi_reference_genomes/blob/master/build_hg38full.sh). Overlapping annotations can be found in the file Telescope-ERVmap.txt.

``` bash
bowtie2 --no-unal --score-min L,0,1.6 -p -k 100 --very-sensitive-local \
  -x /path/to/bowtie2/index -S output.sam -1 _1.fastq -2 _2.fastq 
telescope assign --exp_tag HERV --theta_prior 200000 --max_iter 200 \
  --updated_sam output.sam HERVannotation.gtf
kallisto quant -b 100 -i /path/to/kallisto/index [...]
```

Results can be downloaded [here](https://drive.google.com/drive/folders/1oEBPbkFb6VCnNOPP-w0nW-pGF-wFc52W?usp=sharing)

``` r
cts_g <- as.matrix(read.delim("Gene_table.txt",row.names = 1))
cts_h <- as.matrix(read.delim("HERV_hiconf_table.txt",row.names = 1))
all<- round(rbind(cts_g,cts_h),0)
info <- read.delim("unique-ambiguously_table.txt", row.names=1)
coldata<-info[,c("Condition","Name","Platform")]
ann_colors <- list(Condition = c(SLE="red", Healthy="blue"),
                   Platform=c(HiSeq_2500='#F8766D',NextSeq_500='#00BFC4'))
teles_erv<-read.table("Telescope-ERVmap.txt",header = F)
```

Gene Count and HERV counts were concatenated for each sample. A principal component analysis was perfomed on the data normalized without taking into account the sequencing platform:

PCA:

``` r
mincount <- 10
minrep <- 3
all <- all[rowSums(all >= mincount) >= minrep,]
dds_v2 <- DESeqDataSetFromMatrix(countData = all,
                              colData = coldata,
                              design =~ Condition)
dds_Wald_effect_Condition2<-DESeq(dds_v2, fitType = "parametric")
rld_v3 <- vst(dds_Wald_effect_Condition2, blind=TRUE,fitType = "parametric")
PCA_LPI <- prcomp(t(assay(rld_v3)))
ploting_PCA_LPI<-data.frame(PCA_LPI$x[,1:2],Platform=coldata$Platform,Condition=coldata$Condition)
percentVar<-round(100*summary(PCA_LPI)$importance[2,1:2],0)
ggplot(data=ploting_PCA_LPI)+
  geom_point(aes(x=PC1,y=PC2,colour=Platform,shape=Condition),size=2)+
  xlab(paste0("PC1: ",percentVar[1],"% variance"))+
  ylab(paste0("PC2: ",percentVar[2],"% variance"))+
  coord_fixed(ratio=1.0)+
  #ggtitle("Gene Count + HERVs (telescope)")+
  theme(text = element_text(size=15),aspect.ratio=1)
```

![](Tokuyama_analysis_files/figure-markdown_github/PCA1-1.png)

PCA shows a clear bias for sequencing platform, therfore these paramater was used as a part of the model in the following analysis.

``` r
dds_v1 <- DESeqDataSetFromMatrix(countData = all,
                                 colData = coldata,
                                 design =~ Condition + Platform)
dds_Wald_effect_Condition<-DESeq(dds_v1, fitType = "parametric")
rld_v2 <- vst(dds_Wald_effect_Condition, blind=TRUE,fitType = "parametric")
Tele_res<-results(dds_Wald_effect_Condition,name = "Condition_SLE_vs_Healthy")
Tele_res<-Tele_res[grep(row.names(Tele_res),pattern = '^NM_|^NR',perl=T,invert = T),]
volc_Tele<-data.frame(x=Tele_res$log2FoldChange,y=-(log10(Tele_res$padj)),fill=1,name=rownames(Tele_res))
volc_Tele$fill[Tele_res$padj<0.05 & abs(Tele_res$log2FoldChange)>1]<-2
volc_Tele<-volc_Tele[!is.na(Tele_res$padj),]
volc_Tele$fill<-factor(volc_Tele$fill,levels=c(1,2))
ggplot(data=volc_Tele)+
  geom_point(aes(x=x,y=y,colour=fill), size=0.9)+
  scale_color_manual(values=c("black","red"))+
  coord_fixed(ratio=2)+
  #geom_text(aes(x=x,y=y,label=ifelse(fill==2,as.character(name),'')),hjust=0, vjust=0)+
  ylab("-log10(padj)")+xlab("log2FoldChange")+#ggtitle("Telescope HERVs")+
  theme(legend.position="none",text = element_text(size=15))
```

![](Tokuyama_analysis_files/figure-markdown_github/DE1-1.png)

The results presented here, with the same parameters and thresholds as Tokuyama, showed less HERVs upregulated (19) in SLE and some HERVs silenced (4).

``` r
DE_telescope2<-rownames(Tele_res)[Tele_res$padj<0.05 & abs(Tele_res$log2FoldChange)>1& !is.na(Tele_res$padj)]
pheatmap(assay(rld_v2)[DE_telescope2,],scale = "row",
         annotation_col = data.frame(Condition=coldata[,1], Platform=coldata[,3],row.names =colnames(assay(rld_v2))),
         show_rownames = T, color =colorRampPalette(c('blue','white','red'),space = "Lab")(50),
         border_color = NA, legend = T, legend_labels = "Z score",annotation_colors=ann_colors,
         cutree_rows=2,cutree_cols=2,
         cluster_rows = T,labels_col =as.character(coldata[,2]),fontsize = 10)
```

![](Tokuyama_analysis_files/figure-markdown_github/DE2-1.png)

Overlapped HERVs between Telescope and ERVmap annotations, left panel shows the normalized Z-score scaled values of Telescope annotations, the matched annotations in ERVmap are plotted in the right panel, also normalized Z-score scale values are illustrated.

``` r
twoHeat<-list()
twoHeat[[1]]<-pheatmap(assay(rld_v2)[as.character(teles_erv$V1[which( teles_erv$V1 %in% DE_telescope2)]),],scale = "row",
                       annotation_col = data.frame(Condition=coldata[,1], row.names =colnames(assay(rld_v3))),
                       show_rownames = T, color =colorRampPalette(c('blue','white','red'),space = "Lab")(50),
                       border_color = NA, legend = T, legend_labels = "Z score",annotation_legend = F,
                       annotation_colors=ann_colors, cluster_cols = F,
                       cluster_rows = F,labels_col =as.character(coldata[,2]),main = "Telescope",fontsize = 9,silent = T)[[4]]

twoHeat[[2]]<-pheatmap(akiko_erv[as.character(teles_erv$V2[which( teles_erv$V1 %in% DE_telescope2)]),],scale = "row",
                       annotation_col = data.frame(Condition=coldata[,1], row.names =colnames(akiko_erv)),
                       show_rownames = T, color =colorRampPalette(c('blue','white','red'),space = "Lab")(50),
                       border_color = NA, legend = T , legend_labels = "Z score",
                       annotation_colors=ann_colors, cluster_cols = F,
                       cluster_rows = F,labels_col =as.character(coldata[,2]),main = "ERVmap",fontsize = 9,silent = T)[[4]]

grid.arrange(arrangeGrob(grobs= twoHeat,ncol=2,widths = c(1,1)))
```

![](Tokuyama_analysis_files/figure-markdown_github/DE3-1.png) The differences in the results generated by ERVmap and Telescope could be due to different HERV annotations, and we found that seven out of the nine overlapped HERVs showed an overexpression pattern in both analysis and six of them are among their top 30 significantly differentially expressed.

The breadth of knowledge of HERVs is expanding and determination of locus specific expression profiles of these elements are starting to be important in several conditions. A wider range of HERV annotations leads to more accurate results, which will provide new insights into the understanding of HERV expression in SLE and other conditions.

### References

Tokuyama, Maria, Yong Kong, Eric Song, Teshika Jayewickreme, Insoo Kang, and Akiko Iwasaki. 2018. “ERVmap Analysis Reveals Genome-Wide Transcription of Human Endogenous Retroviruses.” *Proceedings of the National Academy of Sciences* 115 (50). National Academy of Sciences: 12565–72. doi:[10.1073/pnas.1814589115](https://doi.org/10.1073/pnas.1814589115).
