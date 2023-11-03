## uORF strength vs TSS
Here we describe how we analyzed the TSS info vs uORF strength.

### TSS-seq and CAGE-seq data are download from [TAIR website](https://www.arabidopsis.org)  
[TSS-seq (two weeks old seedling) data from Nielsen et al., PLoS Genetics 2019](https://doi.org/10.1371/journal.pgen.1007969)   
[CAGE-seq (two weeks old seedling) data from Thieffry et al., The Plant Cell 2020](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7268790)  
The downloaded data were further processed into [RiboPlotR](https://github.com/hsinyenwu/RiboPlotR) tabular format with four columns: "total counts", "chromosome number", "P-site position" and "strand" (+ or -).  
The processed files and other files required for reproducing this research are uploaded to Mendeley data.  

### CAGE-seq
**uORF_mORF_Ribo_ratio is also known as Relative uORF Strength (RuS)**
```
###################################
# Weak uORF Oct 21 2023
###################################
#rm(list=ls())
gc()
#Load library
library(dplyr)
library(GenomicFeatures)
library(GenomicRanges)
library(Biostrings)
library(Rsamtools)
library(ORFik)
library(RiboPlotR)
library(ggplot2)
library(openxlsx)
library(DataScienceR)
library(multcompView)

#Load RT TuORFs
ORFs_max_filt <- read.delim("~/Desktop/Weak_uORF/ORFs_max_filt_expressed",header=T,sep="\t",stringsAsFactors = F,quote = "")
ORFs_max_filt_uORF <- ORFs_max_filt %>% filter(category=="uORF")
ORFs_max_filt_uORF_gene_id_undup <- as.data.frame(table(ORFs_max_filt_uORF$gene_id))
# Only select genes with one RiboTaper TuORF (simplify the analysis)
ORFs_max_filt_uORF_gene_id_undup <- ORFs_max_filt_uORF_gene_id_undup %>% filter(Freq==1)
ORFs_max_filt_uORF <- ORFs_max_filt_uORF %>% filter(gene_id %in% ORFs_max_filt_uORF_gene_id_undup$Var1)
ORFs_max_filt_uORF$ORF_pept <- paste0(ORFs_max_filt_uORF$ORF_pept,"*")
nrow(ORFs_max_filt_uORF) #1529 (only from transcripts with one RiboTaper uORFs)

#Make txdb and obtain 5'UTR sequences
GTF_path <- "~/Desktop/Weak_uORF/Araport11+CTRL_20181206_expressed.gtf"
FA <- FaFile("~/Desktop/Leaky_scanning/TAIR10_chr_all_2.fas")
txdb <- makeTxDbFromGFF(file = GTF_path,format="gtf", dataSource="Araport11",organism="Arabidopsis")
fiveUTR <- fiveUTRsByTranscript(txdb, use.names=T)
fiveUTR <- fiveUTR[names(fiveUTR) %in% ORFs_max_filt_uORF$transcript_id]
fiveUTR_seqs <- extractTranscriptSeqs(FA,fiveUTR)
fiveUTR_seqs <- fiveUTR_seqs[names(fiveUTR_seqs) %in% ORFs_max_filt_uORF$transcript_id]
fiveUTR_ORFs <- findMapORFs(fiveUTR, fiveUTR_seqs,startCodon = "ATG",longestORF=T,groupByTx=F,minimumLength=0)
length(fiveUTR_seqs) #1529
length(fiveUTR_ORFs) #5488

#Extend the 5'UTR to promoter region (+100 bp)
EXTEND <- function(x){
  if(as.character(strand(x))[1]=="+"){
    start(x)[1]=start(x)[1]-100
  }
  else{
    end(x)[1]=end(x)[1]+100
  }
  return(x)
}
fiveUTR2 <- lapply(1:length(fiveUTR),function(x) EXTEND(fiveUTR[[x]]))
names(fiveUTR2) <- names(fiveUTR)
fiveUTR2 <- GRangesList(fiveUTR2)

#Load CAGE-seq data
CAGE=read.delim(file="~/Desktop/Weak_uORF/wt_R123_genome_RiboPlotR.txt",header=F,stringsAsFactors=F,sep="\t")
colnames(CAGE) <- c("count","chr","start","strand")
CAGE$end <- CAGE$start
head(CAGE)

#Convert CAGE reads to GRanges
CAGE <- makeGRangesFromDataFrame(CAGE,keep.extra.columns=T)
CAGE

#Make an empty data.frame
OUT_df2 <- data.frame()

#Get pTUu by mapping CAGE reads to the 5'UTR+promoter range
for(i in 1:length(fiveUTR_seqs)){
  if(i %% 100 ==0) print(i)
  tx_id=names(fiveUTR_seqs[i])
  tx_strand <- as.character(strand(unlist(fiveUTR[tx_id]))[1])
  # not use the longest uORF parameter
  fiveUTR_ORFs <- findMapORFs(fiveUTR[tx_id], fiveUTR_seqs[tx_id],startCodon = "ATG",longestORF=F,groupByTx=F,minimumLength=8)
  if(length(fiveUTR_ORFs)==0) next
  else{
    uORF_starts <- c()
    for(j in 1:length(fiveUTR_ORFs)){
      if(tx_strand=="+"){
        uORF_starts[j] <- unlist(start(fiveUTR_ORFs[[j]])[1])
      } else {
        uORF_starts[j] <- unlist(end(fiveUTR_ORFs[[j]])[1])
      }
    }
    uORF_position_ranks <- c()
    if(tx_strand=="+"){
      uORF_position_ranks <- rank(uORF_starts)
    } else {
      uORF_position_ranks <- rank(-uORF_starts)
    }
    
    z=fiveUTR2[[tx_id]]
    aa=unlist(Map(`:`, start(z), end(z)))
    gr=GRanges(seqnames=suppressWarnings(as.character(seqnames(z))[1]), ranges=IRanges(start = aa,width=1),strand=tx_strand)
    ranges1 <- subsetByOverlaps(CAGE,gr)
    
    pTUu <- c()
    for(i in 1:length(uORF_starts)){
      if(tx_strand=="+"){
        ranges1a <- ranges1[start(ranges(ranges1)) > uORF_starts[i]] #counts downstream of uORFs starts
        ranges1b <- ranges1[start(ranges(ranges1)) < uORF_starts[i]] #counts upstream of uORFs starts
        Ca <- sum(ranges1a$count)
        Cb <- sum(ranges1b$count)
        pTUu[i] <- Cb/(Ca+Cb)*100 
      } else{
        #for "-" strand
        ranges1a <- ranges1[start(ranges(ranges1)) < uORF_starts[i]] #counts downstream of uORFs starts
        ranges1b <- ranges1[start(ranges(ranges1)) > uORF_starts[i]] #counts upstream of uORFs starts
        Ca <- sum(ranges1a$count)
        Cb <- sum(ranges1b$count)
        pTUu[i] <- Cb/(Ca+Cb)*100 
      }
    }
    
    uORF_seq=unname(sapply(1:length(fiveUTR_ORFs) ,function(x) as.character(translate(DNAString(paste(getSeq(FA,fiveUTR_ORFs[[x]]), collapse=""))))))
    
    out_df <- data.frame(gene_id=substr(tx_id,1,9),
                         tx_id=tx_id,
                         uORF_id=names(fiveUTR_ORFs),
                         pTUu=pTUu,
                         aa_seq=uORF_seq,
                         uORF_starts=uORF_starts,
                         uORF_strand=tx_strand,
                         uORF_position_ranks=uORF_position_ranks)
    OUT_df2 <- rbind(OUT_df2,out_df)
  }
}

#Save CAGE pTUu data
saveRDS(OUT_df2,file="~/Desktop/Weak_uORF/OUT_CAGE_WuORFs.RDS")
OUT_df2 <- readRDS("~/Desktop/Weak_uORF/OUT_CAGE_WuORFs.RDS")
dim(OUT_df2) #[1] 4471    8

#Next merge the main ORF info with uORF info (both from RiboTaper output)
#We need the Ribo-seq reads mapped to the uORFs and mORFs
ORFs_max_filt_mORF <- ORFs_max_filt %>% 
  filter(transcript_id %in% ORFs_max_filt_uORF$transcript_id,category=="ORFs_ccds") %>% 
  group_by(transcript_id) %>% top_n(n=1, wt = ORF_length)

length(unique(ORFs_max_filt_uORF$transcript_id)) #1529
length(unique(ORFs_max_filt_mORF$transcript_id)) #1455

ORFs_max_filt_mORF <- ORFs_max_filt_mORF %>% 
  mutate(mORF_P_sites=ORF_P_sites,mORF_length=ORF_length) %>% 
  dplyr::select(transcript_id,mORF_P_sites,mORF_length)

ORFs_max_filt_uORF_mORF <- inner_join(ORFs_max_filt_uORF,ORFs_max_filt_mORF,by="transcript_id") %>% 
  mutate(uORF_mORF_Ribo_ratio=(ORF_P_sites/ORF_length)/(mORF_P_sites/mORF_length)) %>% 
  mutate(mORF_norm_read_count=mORF_P_sites/mORF_length) %>% 
  mutate(TuORF_TE=(ORF_P_sites/ORF_RNA_sites)) %>% 
  mutate(TuORF_norm_read_count=ORF_P_sites/ORF_length)

nrow(ORFs_max_filt_uORF) #1529
nrow(ORFs_max_filt_uORF_mORF) #1455

#Merge OUT_df2 (CAGE data) with ORFs_max_filt_uORF_mORF
OUT_df5 <- inner_join(ORFs_max_filt_uORF_mORF, OUT_df2, by = c("transcript_id" = "tx_id", "ORF_pept" = "aa_seq"))
head(OUT_df5)
dim(OUT_df5) #[1] 1252   50, number smaller because the 10 aa criteria
length(unique(OUT_df5$gene_id.x)) #1252
OUT_df5$pTUu <- round(OUT_df5$pTUu,3)

ORFs_max_filt_uORFx <- ORFs_max_filt_uORF %>% filter(transcript_id %in% OUT_df2$tx_id)

###******************** Remove pTUu = NaN
OUT_df5 <- OUT_df5 %>% filter(pTUu>=0)
dim(OUT_df5) #[1] 1252   50

#Plot pTUu CAGE vs u-m Ribo ratio
OUT_df5$Category <- ifelse(OUT_df5$uORF_mORF_Ribo_ratio>1, "Strong", ifelse(OUT_df5$uORF_mORF_Ribo_ratio>0.2 , "Mid","Weak"))
OUT_df5$Category <- factor(OUT_df5$Category, levels=c("Strong","Mid","Weak"), labels=c("Strong >1","Mid 1~0.2","Weak <0.2"))                   
table(OUT_df5$Category)
A=pairwise_ks_test(value = OUT_df5$pTUu,group = OUT_df5$Category) #perform pairwise KS test 
A
multcompLetters(A*6,
                compare="<",
                threshold=0.05,
                Letters=letters,
                reversed = FALSE)
# Strong >1 Mid 1~0.2 Weak <0.2 
# "a"       "b"       "c" 
ggplot(OUT_df5, aes(pTUu,color=Category))+
  stat_ecdf(geom = "step",size=1)+
  ggtitle("Group by uORF/mORF Ribo 1, 0.2, 0")+
  xlim(0,100)+
  theme_classic() +
  theme(text = element_text(size = 15,face = "bold"), 
        plot.title = element_text(hjust = 0.5, size = 18,face = "bold"),
        plot.subtitle=element_text(hjust=0.5),
        legend.text = element_text(size=15),
        legend.title = element_text(size=17,face = "bold"),
        axis.text = element_text(size = 12),
        axis.title=element_text(size=15,face="bold"))
ggsave("~/Desktop/Weak_uORF/pTUu CAGE vs u-m Ribo ratio 1 0.2 0.pdf",width = 6,height = 4)
```
![image](https://github.com/hsinyenwu/Weak_uORFs/assets/4383665/a6309845-c467-4b45-a4ee-436845383910)

```
OUT_df5$Category <- ifelse(OUT_df5$pTUu> 80 , "Strong", ifelse(OUT_df5$pTUu> 40 , "Mid","Weak"))
OUT_df5$Category <- factor(OUT_df5$Category, levels=c("Strong","Mid","Weak"), labels=c("Strong 100~80 pTUu","Mid 80-40 pTUu","Weak 40~0 pTUu"))                   
table(OUT_df5$Category)
A=pairwise_ks_test(value = TE_CDS2$TE,group = TE_CDS2$Category)
A
multcompLetters(A*factorial(4-1),
                compare="<",
                threshold=0.05,
                Letters=letters,
                reversed = FALSE)
# Strong >1 Mid 1~0.2 Weak <0.2 
# "a"       "b"       "c" 
ggplot(OUT_df5, aes(x=uORF_mORF_Ribo_ratio,color=Category))+
  stat_ecdf(geom = "step",size=1)+
  ggtitle("Group by pTUu 80, 40, 0")+
  xlim(0,5)+
  theme_classic() +
  theme(text = element_text(size = 15,face = "bold"), 
        plot.title = element_text(hjust = 0.5, size = 18,face = "bold"),
        plot.subtitle=element_text(hjust=0.5),
        legend.text = element_text(size=15),
        legend.title = element_text(size=17,face = "bold"),
        axis.text = element_text(size = 12),
        axis.title=element_text(size=15,face="bold"))

ggsave("~/Desktop/Weak_uORF/uORF_mORF_Ribo_ratio vs pTUu 80 40 0 CAGE.pdf",width = 6,height = 4)
```
<img width="678" alt="image" src="https://github.com/hsinyenwu/Weak_uORFs/assets/4383665/746fee93-f139-433b-9307-3b06e42805b3">

```
RNA <- read.delim("~/Desktop/Weak_uORF/RSEM_Araport11_1st+2nd_Riboseq_RNA.genes.results_CDS_only",header=T,sep="\t",stringsAsFactors = F, quote="")
Ribo <- read.delim("~/Desktop/Weak_uORF/RSEM_Araport11_1st+2nd_Riboseq_Ribo.genes.results_CDS_only",header=T,sep="\t",stringsAsFactors = F, quote="")
TE_CDS <- data.frame(gene_id=RNA$gene_id,
                     RNA=RNA$TPM,
                     Ribo=Ribo$TPM,
                     TE=Ribo$TPM/RNA$TPM)
# Remove not translated transcripts
fiveUTRByGeneNames <- substr(names(fiveUTRsByTranscript(txdb, use.names=T)),1,9)
TE_CDS <- TE_CDS %>% filter(gene_id %in% fiveUTRByGeneNames)
# TE_CDS <- TE_CDS %>% filter(TE>0 & !is.infinite(TE))
TE_CDS <- TE_CDS %>% filter(RNA>0)
nrow(TE_CDS) #21805
TE_CDS_strong <- TE_CDS %>% filter(gene_id %in% OUT_df5[OUT_df5$Category=="Strong 100~80 pTUu",]$gene_id.x)
TE_CDS_strong$Category <- "Strong"
TE_CDS_Mid <- TE_CDS %>% filter(gene_id %in% OUT_df5[OUT_df5$Category=="Mid 80-40 pTUu",]$gene_id.x)
TE_CDS_Mid$Category <- "Mid"
TE_CDS_Weak <- TE_CDS %>% filter(gene_id %in% OUT_df5[OUT_df5$Category=="Weak 40~0 pTUu",]$gene_id.x)
TE_CDS_Weak$Category <- "Weak"
TE_CDS_RTuORFs <- TE_CDS %>% filter(gene_id %in% ORFs_max_filt_uORF$gene_id)
TE_CDS_RTuORFs$Category <- "RTuORFs"

TE_CDS_Other <- TE_CDS %>% filter(!(gene_id %in% c(OUT_df5$gene_id,TE_CDS_RTuORFs$gene_id)))
TE_CDS_Other$Category <- "Other"

TE_CDS2 <- rbind(TE_CDS_RTuORFs,TE_CDS_strong,TE_CDS_Mid,TE_CDS_Weak,TE_CDS_Other)
TE_CDS2$Category <- factor(TE_CDS2$Category, levels=c("RTuORFs","Strong","Mid","Weak","Other"), labels=c("RTuORFs","Strong 100~80 pTUu","Mid 80-40 pTUu","Weak 40~0 pTUu","Other"))                   
nrow(TE_CDS2)

table(TE_CDS2$Category)

A=pairwise_ks_test(value = TE_CDS2$TE,group = TE_CDS2$Category)
A
multcompLetters(A*factorial(4-1),
                compare="<",
                threshold=0.05,
                Letters=letters,
                reversed = FALSE)
table(TE_CDS2$Category)
ggplot(TE_CDS2, aes(TE,color=Category))+
  stat_ecdf(geom = "step",size=1)+
  ggtitle("TE with different pTUu")+
  xlim(0,3.5)+ 
  theme_classic()  +
  theme(text = element_text(size = 15,face = "bold"), 
        plot.title = element_text(hjust = 0.5, size = 18,face = "bold"),
        plot.subtitle=element_text(hjust=0.5),
        legend.text = element_text(size=15),
        legend.title = element_text(size=17,face = "bold"),
        axis.text = element_text(size = 12),
        axis.title=element_text(size=15,face="bold"))
ggsave("~/Desktop/Weak_uORF/TE vs pTUu 80 40 0 CAGE.pdf",width = 6,height = 4)
```
![image](https://github.com/hsinyenwu/Weak_uORFs/assets/4383665/e84a73d1-ef81-4f4f-80bb-118e3b594fe9)

```
#######
# uORF/mORF ratio vs mORF TE
OUT_df6 <- OUT_df5
OUT_df6$Category <- ifelse(OUT_df6$uORF_mORF_Ribo_ratio >1, "Strong", ifelse(OUT_df6$uORF_mORF_Ribo_ratio >0.2, "Mid","Weak"))
OUT_df6$Category <- factor(OUT_df6$Category, levels=c("Strong","Mid","Weak"), labels=c("Strong","Mid","Weak"))                   
TE_CDS_strong <- TE_CDS %>% filter(gene_id %in% OUT_df6[OUT_df6$Category=="Strong",]$gene_id.x)
TE_CDS_strong$Category <- "Strong"
TE_CDS_Mid <- TE_CDS %>% filter(gene_id %in% OUT_df6[OUT_df6$Category=="Mid",]$gene_id.x)
TE_CDS_Mid$Category <- "Mid"
TE_CDS_Weak <- TE_CDS %>% filter(gene_id %in% OUT_df6[OUT_df6$Category=="Weak",]$gene_id.x)
TE_CDS_Weak$Category <- "Weak"
TE_CDS_Other <- TE_CDS %>% filter(!(gene_id %in% OUT_df6$gene_id.x))
TE_CDS_Other$Category <- "Other"
TE_CDS2 <- rbind(TE_CDS_strong,TE_CDS_Mid,TE_CDS_Weak,TE_CDS_Other)
TE_CDS2$Category <- factor(TE_CDS2$Category, levels=c("Strong","Mid","Weak","Other"), labels=c("Strong >1","Mid 1~0.2","Weak <0.2","No RT_uORF"))                   
table(TE_CDS2$Category)
# Strong >1  Mid 1~0.2  Weak <0.2 No RT_uORF 
#   322        489        440      20554 
A=pairwise_ks_test(value = TE_CDS2$TE,group = TE_CDS2$Category)
A
multcompLetters(A*16,
                compare="<",
                threshold=0.01,
                Letters=letters,
                reversed = FALSE)

ggplot(TE_CDS2, aes(TE,color=Category))+
  stat_ecdf(geom = "step",size=1)+
  ggtitle("Group by uORF/mORF Ribo 1, 0.2, 0")+
  xlim(0,3.5)+
  theme_classic() +
  theme(text = element_text(size = 15,face = "bold"), 
        plot.title = element_text(hjust = 0.5, size = 18,face = "bold"),
        plot.subtitle=element_text(hjust=0.5),
        legend.text = element_text(size=15),
        legend.title = element_text(size=17,face = "bold"),
        axis.text = element_text(size = 12),
        axis.title=element_text(size=15,face="bold"))
ggsave("~/Desktop/Weak_uORF/Figure 1C TE for uORF_mORF_Ribo_ratio 1 0.2 0.pdf",width = 6,height = 4)
```
![image](https://github.com/hsinyenwu/Weak_uORFs/assets/4383665/18b42a3d-7fe4-4457-ba67-811a2338f876)

### TSS-seq 
Just use the TSS-Seq_WT_Nielsen_2019 file in Mendeley data for the above code.


