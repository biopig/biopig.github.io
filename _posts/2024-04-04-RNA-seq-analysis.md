---
layout: post
title: 转录组分析大致流程
categories: [RNA-seq, 转录组]
description: 转录组分析大致流程
keywords: RNA-seq, 转录组
---

### 转录组分析

- **注意**

1. 本例子中，样品名称为a1和b1，各3个重复，fastq文件格式为`a1-1_1.clean.fq.gz`和`a1-1_2.clean.fq.gz`，最终需要根据自己的文件格式进行修改。
2. 所有这些代码都可以放到一个bash脚本中运行，不需要在命令行直接运行

3. 数据分析时每一步过程都在对应的文件荚中运行，下面是本例子大致的文件夹

   ```shell
   raw_data\
   1.clean_data\
   2.alignment\
   3.get_expre\
   4.diff\
   5.enrich\
   6.wgcna\
   7.call_variant\
   8.variant_fileter\
   ```

#### 0.构建基因组index

使用STAR构建index，这里面有个参数`--sjdbOverhang 149`，其中149是`read长度-1`，现在绝大多数都是双端150bp测序，但是还是需要根据自己实际的长度设置。

```shell
genomeDir=~/workspace/reference/sequence/genome_seq/arabidopsis/Arabidopsis_thaliana.TAIR11.dna.toplevel.fa
genomeGTF=~/workspace/reference/annotation/arabidopsis/Araport11_GTF_genes_transposons.current.gtf

STAR \
	--runMode genomeGenerate \
	--runThreadN 30 \
	--genomeDir Wm82.a4.v1 \
	--genomeFastaFiles $genomeDir \
	--sjdbGTFfile $genomeGTF \
	--sjdbOverhang 149
```

#### 1.去接头和过滤

本例子中，使用cutadapt去除测序序列中的接头，这里面的接头用的是illumina测序平台的接头，现在市面上用的最多的也是这种接头。

```shell
genomeDir=~/workspace/reference/sequence/genome_seq/arabidopsis/Arabidopsis_thaliana.TAIR10.dna.toplevel.fa

for i in a1 b1
do
    for a in `seq 3`
    do
        cutadapt \
                -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA \
                -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT \
                --max-n 0 -m 1 -j 20 \
                -o ../1.clean_data/"$i"-"$a"_1.clean.fq.gz \
                -p ../1.clean_data/"$i"-"$a"_2.clean.fq.gz \
                ../raw_data/"$i"-"$a"_1.clean.fq.gz \
                ../raw_data/"$i"-"$a"_2.clean.fq.gz
    done
done
```



#### 2.比对

```shell
genomeIndex=~/workspace/reference/sequence/genome_fa/arabidopsis/star/

for i in a1 b1
do
	for a in `seq 3`
	do
		STAR \
			--runThreadN 10 \
			--genomeDir $genomeIndex \
			--readFilesCommand gunzip -c \
			--readFilesIn ../1.clean_data/"$i"-"$a"_1.clean.fq.gz ../1.clean_data/"$i"-"$a"_2.clean.fq.gz \
			--outSAMtype BAM SortedByCoordinate \
			--outFileNamePrefix "$i"-"$a" \
			--seedMultimapNmax 1000000 \
			--quantMode GeneCounts \
			--sjdbOverhang 149
	done
done
```

#### 3.得到基因表达量

```shell
#关于如何构建txdb，需要在R里面进行，下面列出一个例子make_txdb
txdb=~/workspace/reference/annotation/arabidopsis/ath_v44_txdb_from_gtf.txdb

for i in a1 b1
do
	for a in `seq 3`
	do
		awk -v OFS="\t" -v i="$i_$a" \
		'BEGIN{print "Gene_ID",i}{print $1,$4}' \
		../2.alignment/"$i"ReadsPerGene.out.tab \
		|grep -v "N" >"$i".expression.txt
	done
done

paste *expression.txt |cut -f 1,2,4,6,8,10,12 >all.gene.count &&\
rm *expression.txt &&\

Rscript normalizeGeneCounts.R all.gene.count $txdb CPM
```

- make_txdb.R

  ```R
  library(GenomicFeatures)
  txdb<-makeTxDbFromGFF(file = "~/workspace/reference/annotation/arabidopsis/Araport11_GTF_genes_transposons.current.gtf",format="gtf")
  saveDb(txdb, file="ath_v44_txdb_from_gtf.txdb")
  ```

- normalizeGeneCounts.R

  ```R
  #Rscript normalizeGeneCounts.R all.gene.count ath_v44_txdb_from_gtf.txdb CPM
  library(GenomicFeatures)
  
  Args<-commandArgs(T)
  counts<-read.table(Args[1],header=T,row.names=1)
  TxDb<-loadDb(Args[2])
  
  normalizeGeneCounts <- function(counts, TxDb, method) {
    require("GenomicFeatures")
    length <- width(genes(TxDb))/10 ^ 3 #gene length per kb
    if (method == "CPM") {
      normalized <-
        as.data.frame(apply(counts,2, function(x) (x/sum(x)) * 10 ^ 6))
    } else if (method == "RPKM") {
      normalized <-
        as.data.frame(t(apply(counts, 1, "/", colSums(counts) / 10 ^ 6)) / length)
    } else if (method == "TPM") {
      normalized <-
        as.data.frame(t(apply(
          counts / length, 1, "/", colSums(counts / length) / 10 ^ 6
          )))
    } else {
      stop("method must be CPM, RPKM or TPM")
    }
      return(normalized)
  }
  
  normalized_count<-normalizeGeneCounts(counts,TxDb,Args[3])
  write.table(normalized_count,file="all.gene.count.CPM.result",append=F,quote=F,sep="\t")
  q(save="no")
  ```

  

#### 4.差异表达分析

run_DE_analysis.pl来自trinity软件

```shell
~/biosoft/trinityrnaseq-Trinity-v2.8.5/Analysis/DifferentialExpression/run_DE_analysis.pl \
	--matrix ../3.get_expre/all.gene.count \
	--method DESeq2 \
	--samples_file ../sample.txt \
	--contrasts ../contrasts.txt \
	--output ./
```

```shell
file=$1

cat $file |grep -v sample|awk '{if(($4 > 1 ||$4 < -1 ) && $7 < 0.05) print $0}' |awk -v OFS="\t" 'BEGIN{printf"geneID \t sampleA \t sampleB \t logFC \t logCPM \t PValue \t FDR \n"}''{if($4>0){$8="ups"}else{$8="downs"};print $0}' >"$file".filter &&\

cat "$file".filter |grep -v sample |grep ups |grep -v ^$ | cut -f 1 >"$file".filter.genes.up &&\
cat "$file".filter |grep -v sample |grep downs |grep -v ^$ | cut -f 1 >"$file".filter.genes.down &&\
cat "$file".filter.genes.up "$file".filter.genes.down >"$file".filter.all.genes
```

- sample.txt

  ```shell
  a1	a1-1
  a1	a1-2
  a1	a1-3
  b1	b1-1
  b1	b1-2
  b1	b1-3
  ```

- contrasts.txt

  ```shell
  a1	b1
  ```

#### 5.富集分析

```shell
library(AnnotationDbi)
library(AnnotationHub)
library(clusterProfiler)
library(enrichplot)
#library(xlsx)
library(pheatmap)

maize<-loadDb("~/workspace/reference/annotation/arabidopsis/org.At.tair.db.sqlite")
gene_CPM<-read.table("../3.get_expre/all.gene.count.CPM.result",header=T)


for (i in c("up","down")) {
 for (a in c("0","30","1d","6h")) {
  input<-paste("../3.diff-expre/all.gene.count.2k-",a,"_vs_B1-",a,".edgeR.DE_results.filter.genes.",i,sep = "")
  go_output<-paste(a,i,"GOenrich_result.txt",sep = "-")
  kegg_output<-paste(a,i,"KEGGenrich_result.txt",sep = "-")
  go_pdf_output<-paste(a,i,"GO_enrich.pdf",sep = "-")
  kegg_pdf_output<-paste(a,i,"KEGG_enrich.pdf",sep = "-")
  
  result<-read.table(input,header = F)
  result1<-as.character(result$V1)
  
  go_result<-enrichGO(result1,maize,keyType = "TAIR",
                      ont = "ALL",pvalueCutoff = 0.05,
                      qvalueCutoff = 0.05)
  pdf(go_pdf_output,height = 5,width = 7)
  p1<-barplot(go_result,showCategory=10)
  p2<-dotplot(go_result,showCategory=10)
  print(p1)
  print(p2)
  dev.off()
  
  kegg_result<-enrichKEGG(result1,organism = "ath")
  pdf(kegg_pdf_output,height = 5,width = 7)
  p3<-barplot(kegg_result,showCategory=10)
  p4<-dotplot(kegg_result,showCategory=10)
  print(p3)
  print(p4)
  dev.off()  
  
  go_result1<-go_result@result
  kegg_result1<-kegg_result@result
  write.table(go_result1,go_output,quote = F,sep = "\t",row.names = F)
  write.table(kegg_result1,kegg_output,quote = F,sep = "\t",row.names = F)
  
  for (bb in seq(5)) {
    gene_cluster1<-kegg_result1[bb,8] %>% strsplit("/") %>% as.data.frame()
    gene_cluster1<-gene_CPM[gene_cluster1[,1],]
    pdf_name<-paste(kegg_result1[bb,2],"_",i,".pdf",sep = "")
    pdf(pdf_name,height = 6,width = 6)
    p5<-pheatmap(gene_cluster1,cluster_cols = F,cluster_rows = T,
             scale = "row",show_colnames = T,angle_col = 45,
             fontsize_col = 10,
             color = colorRampPalette(c("blue", "white", "red"))(50),
             main = kegg_result1[bb,2])
    print(p5)
    dev.off()
    
  }
 }
}

q(save="no")
```

#### 6.WGCNA

```SHELL
library(WGCNA)
library(xlsx)
library(stringr)
library(edgeR)
library(dplyr)
library(pheatmap)
setwd("6.wgcna")

#################1、读取文件，并进行数据的过滤#####
dataExpr<-read.table("../3.get_expre/all.gene.count.CPM.result",header = T,row.names = 1)    #读取基因表达数据
data_mad<-apply(X = dataExpr,1,mad)
dataExpr<-dataExpr[which(data_mad > 1),]    #过滤掉mad小于1的基因，https://blog.csdn.net/qq_32649321/article/details/118193970
data_expr<-data.frame(id=rownames(dataExpr),dataExpr)     #这个修改的数据后面会用到
dataExpr_change <- t(dataExpr)    #转置
traitData<-read.table("all_trait.txt",header = T,row.names=1)   #读取表型数据
nGenes = ncol(dataExpr_change)    #基因的数目
nSamples = nrow(dataExpr_change)  #样品的数目

nGenes = ncol(dataExpr_change)    #基因的数目
nSamples = nrow(dataExpr_change)  #样品的数目
#清空回收站
collectGarbage()
#################2、筛选软阈值########################
powers = c(c(1:10), seq(from = 12, to=36, by=2))    #选取一系列的软阈值power
#根据power计算软阈值
sft = pickSoftThreshold(dataExpr_change, powerVector = powers, verbose = 3,networkType = "signed hybrid")

#下面要画两个图，一个是R^2和power的关系，一个是平均值和power的关系
pdf("selet_softpower.pdf",width = 12,height = 5)
par(mfrow = c(1,2),mar = c(3, 6, 3, 3))
cex1 = 0.9
plot(sft$fitIndices[,1], sft$fitIndices[,2],
     xlab="Soft Threshold (power)",ylab="Scale Free Topology Model Fit,signed R^2",type="n",
     main ="Scale independence")

text(sft$fitIndices[,1], sft$fitIndices[,2], labels=powers,cex=cex1,col="red")  #text的参数为添加文本的位置，文本信息
abline(h=0.85,col="red")

plot(sft$fitIndices[,1], sft$fitIndices[,5],
     xlab="Soft Threshold (power)",ylab="Mean Connectivity", type="n",
     main ="Mean connectivity")
text(sft$fitIndices[,1], sft$fitIndices[,5], labels=powers, cex=cex1,col="red")
dev.off()
#根据上面的结果，下面使用power=10进行分析。有个疑问的地方就是为什么曲线图是先下降后上升的
#######################3、一步构建网络#############
enableWGCNAThreads()
net<- blockwiseModules(dataExpr_change, power = 14,
                       TOMType = "unsigned", minModuleSize = 30,
                       reassignThreshold = 0, mergeCutHeight = 0.25,
                       numericLabels = TRUE, pamRespectsDendro = FALSE,
                       saveTOMs = TRUE,
                       saveTOMFileBase = "femaleMouseTOM",
                       verbose = 3,maxBlockSize = 10000)

table(net$colors)   #查看颜色对应的模块数目，0代表的是没有对应的模块的基因
mergedColors<-labels2colors(net$colors)   #将labels转换为颜色值

pdf("cluster_dendrogram.pdf",width = 11,height = 5)
par(mfrow= c(1,length(net$blockGenes)))
for (i in 1:length (net$blockGenes)) {  
  plotDendroAndColors(net$dendrograms[[i]], mergedColors[net$blockGenes[[i]]],
                      "Module colors", 
                      dendroLabels = F, hang = 0.03,
                      addGuide = TRUE, guideHang = 0.05)
}
dev.off()
#保存一些重要的数据
moduleLabels = net$colors
moduleColors = labels2colors(net$colors)
MEs = net$MEs;
geneTree = net$dendrograms[[1]];
MEs_col<-net$MEs
colnames(MEs_col)<-paste0("ME",labels2colors(as.numeric(str_replace_all(colnames(MEs),"ME",""))))
plotEigengeneNetworks(MEs_col, "Eigengene adjacency heatmap", 
                      marDendro = c(3,3,2,4),
                      marHeatmap = c(3,4,2,2), plotDendrograms = T, 
                      xLabelsAngle = 90)
#########################4、根据模块和表型鉴定关键基因###########################
#使用一步构建的模块进行分析根据颜色标签重新计算eigengene
MEs0 = moduleEigengenes(dataExpr_change, moduleColors)$eigengenes
#对eigengene排序，使相似的排在一起
MEs = orderMEs(MEs0)

# 输出每个基因所在的模块，以及与该模块的KME值
file.remove('All_Gene_KME.txt')
for(module in substring(colnames(MEs),3)){
  if(module == "grey") next
  ME=as.data.frame(MEs[,paste("ME",module,sep="")])
  colnames(ME)=module
  datModExpr=dataExpr_change[,moduleColors==module]
  datKME = signedKME(datModExpr, ME)
  datKME=cbind(datKME,module=rep(module,length(datKME)))
  datKME<-data.frame(id=rownames(datKME),datKME)
  datKME<-left_join(datKME,data_expr,by="id")
  write.table(datKME,quote = F,sep = "\t",row.names = F,append = T,file = "All_Gene_KME.txt",col.names = F)
}

moduleTraitCor = cor(MEs, traitData, use = "p")   #计算eigengene和表型的相关系数
moduleTraitPvalue = corPvalueStudent(moduleTraitCor, nSamples)    #相关系数的检验

#画图时要显示的相关系数和p值，数据的整理
textMatrix = paste(signif(moduleTraitCor, 2), "\n(",
                   signif(moduleTraitPvalue, 1), ")", sep = "")
dim(textMatrix) = dim(moduleTraitCor)

pdf("module_trait_relationships.pdf",width = 7,height = 5)    #绘制模块之间相关性
par(mar = c(3, 10, 3, 3))
labeledHeatmap(Matrix = moduleTraitCor,
               xLabels = colnames(traitData),
               yLabels = colnames(MEs),
               ySymbols = colnames(MEs),
               colorLabels = FALSE,
               colors = greenWhiteRed(50),
               textMatrix = textMatrix,
               setStdMargins = FALSE,
               cex.text = 0.5,
               zlim = c(-1,1),
               main = paste("Module-trait relationships"))
dev.off()
#################5、可视化基因网络，可以先不做###########
#1、计算TOM
TOM<- TOMsimilarityFromExpr(dataExpr_change, power=10)
probes = colnames(dataExpr_change)
dimnames(TOM) <- list(probes, probes)

# Export the network into edge and node list files Cytoscape can read
# threshold 默认为0.5, 可以根据自己的需要调整，也可以都导出后在cytoscape中再调整
cyt = exportNetworkToCytoscape(TOM,
                               edgeFile = paste("MPV_F1_DF_genes", ".edges.txt", sep=""),
                               nodeFile = paste("MPV_F1_DF_genes", ".nodes.txt", sep=""),
                               weighted = TRUE, threshold = 0.2,
                               nodeNames = probes, nodeAttr = moduleColors)

##################6、感兴趣性状的模块的具体基因分析##########
# 发现跟 Luminal 亚型 最相关的是  pink 模块,所以接下来就分析这两个
Luminal = as.data.frame(design[,3])
names(Luminal) = "Luminal"
module = "pink"
  # names (colors) of the modules
modNames = substring(names(MEs), 3)
geneModuleMembership = as.data.frame(cor(datExpr, MEs, use = "p"))
## 算出每个模块跟基因的皮尔森相关系数矩阵
## MEs是每个模块在每个样本里面的
## datExpr是每个基因在每个样本的表达量
MMPvalue = as.data.frame(corPvalueStudent(as.matrix(geneModuleMembership), nSamples))
names(geneModuleMembership) = paste("MM", modNames, sep="")
names(MMPvalue) = paste("p.MM", modNames, sep="")
geneModuleMembership[1:4,1:4]
## 只有连续型性状才能只有计算
## 这里把是否属 Luminal 表型这个变量0,1进行数值化
Luminal = as.data.frame(design[,3])
names(Luminal) = "Luminal"
geneTraitSignificance = as.data.frame(cor(datExpr, Luminal, use = "p"))
GSPvalue = as.data.frame(corPvalueStudent(as.matrix(geneTraitSignificance), nSamples))
names(geneTraitSignificance) = paste("GS.", names(Luminal), sep="")
names(GSPvalue) = paste("p.GS.", names(Luminal), sep="")

module = "pink"
  column = match(module, modNames)
  moduleGenes = moduleColors==module
  png("step6-Module_membership-gene_significance.png",width = 800,height = 600)
  #sizeGrWindow(7, 7)
  par(mfrow = c(1,1))
  verboseScatterplot(abs(geneModuleMembership[moduleGenes, column]),
                     abs(geneTraitSignificance[moduleGenes, 1]),
                     xlab = paste("Module Membership in", module, "module"),
                     ylab = "Gene significance for Luminal",
                     main = paste("Module membership vs. gene significance\n"),
                     cex.main = 1.2, cex.lab = 1.2, cex.axis = 1.2, col =module)
  dev.off()
  ##===============================step 8：模块导出 =========
  # 找到感兴趣的基因模块进行下一步的分析,导出模块内部基因的连接关系，进入其它可视化软件,比如 cytoscape软件等等。
  # Recalculate topological overlap
  TOM = TOMsimilarityFromExpr(dataExpr_change, power = 10); 
  module = c("black")   #Select module，cyt文件阈值是0.4(下面会提到)
  # Select module probes
  probes = colnames(dataExpr_change) ## 我们例子里面的probe就是基因
  inModule = (moduleColors==module)
  modProbes = probes[inModule]
  ## 也是提取指定模块的基因名
  # Select the corresponding Topological Overlap
  modTOM = TOM[inModule, inModule]
  dimnames(modTOM) = list(modProbes, modProbes)
  ## 模块对应的基因关系矩
  cyt = exportNetworkToCytoscape(
    modTOM,
    edgeFile = paste("CytoscapeInput-edges-", paste(module, collapse="-"), ".txt", sep=""),
    nodeFile = paste("CytoscapeInput-nodes-", paste(module, collapse="-"), ".txt", sep=""),
    weighted = TRUE,
    threshold = 0.1,
    nodeNames = modProbes, 
    nodeAttr = moduleColors[inModule]
  )
  
  datKME<-read.table("All_Gene_KME.txt",header = F,row.names = 1)
  black_genes<-datKME[datKME$V3=="black",3:14]
  colnames(black_genes)<-colnames(dataExpr)[1:12]
  
  pdf("black_heatmap.pdf",width = 5,height = 5)
  par(mar = c(5, 15, 5, 5))
  pheatmap(black_genes,cluster_cols = F,cluster_rows = T,scale = "row",show_rownames = F,show_colnames = T,angle_col = 45,
           fontsize_col = 10,color = colorRampPalette(c("blue", "white", "red"))(50))
  dev.off()
  
  pdf("aaa.pdf")
  for(i in moduleColor){
    module<-i
    probes<-colnames(dataExpr_change)
    inModule<-(moduleColors==module)
    modProbes<-probes[inModule]
    modTOM<- TOM[inModule, inModule]
    dimnames(modTOM)<-list(modProbes, modProbes)
    black_genes<-datKME[datKME$V3==module,3:14]
    colnames(black_genes)<-colnames(dataExpr)[1:12]
    pheatmap(black_genes,cluster_cols = F,cluster_rows = T,scale = "row",show_rownames = F,show_colnames = T,angle_col = 45,
             fontsize_col = 10,color = colorRampPalette(c("blue", "white", "red"))(50),main = module)
  }
  dev.off()
```

#### 7.检测变异位点

```shell
#在使用之前，现将代码阅读一遍，明确如何使用
# for i in name;do sh rna_seq_paired_file.sh $i;done

sample=$1

threads=15
mem=2

#工作的位置
raw_file_dir=/public/home/yangjp/workspace/rna_seq/maize/wanjiong/1.alignment/
analysis_dir=/public/home/yangjp/workspace/rna_seq/maize/wanjiong/1.variant

#软件的位置
gatk=/public/home/yangjp/biosoft/gatk-4.1.3.0/gatk-package-4.1.3.0-local.jar
picard=/public/home/yangjp/biosoft/gatk-4.1.3.0/picard.jar

#文件的位置
reference=/public/home/yangjp/workspace/rna_seq/maize/wanjiong/reference_zheng58/Zm-Zheng58-REFERENCE-CAAS_FIL-1.0.fa
reference_prefix=Zm-Zheng58-REFERENCE-CAAS_FIL-1.0.fa

reference_dir=/public/home/yangjp/workspace/rna_seq/maize/wanjiong/reference_zheng58/
known_vcf=/public/home/yangjp/workspace/rna_seq/maize/wanjiong/reference_zheng58//Zm_Z58_mays.vcf
known_vcf_dir=/public/home/yangjp/workspace/rna_seq/maize/wanjiong/reference_zheng58/
known_vcf_prefix=Zm_Z58_mays.vcf

#判断一些重要文件是否存在，不存在则创建文件
if [ ! -e ${reference_dir}/${reference_prefix}.dict ]
then java -jar -Xmx${mem}G $picard CreateSequenceDictionary R=$reference O=${reference_dir}/${reference_prefix}.dict  #为参考序列生成一个dict文件
fi

if [ ! -e ${reference_dir}/${reference_prefix}.fa.fai ]
then samtools faidx ${reference}
fi

#对bam文件构建索引
if [ ! -e ${raw_file_dir}/${sample}_sorted.bam.bai ]
then samtools index -@ $threads ${raw_file_dir}/${sample}_sorted.bam
fi

if [ ! -e ${known_vcf} ]
then
	bcftools mpileup --threads $threads -f ${reference} ${raw_file_dir}/${sample}_sorted.bam | bcftools call -o ${known_vcf} -O v --threads 30 -mv &&\
	vcfutils.pl varFilter -d 20 -Q 30 ${known_vcf} >${known_vcf}.filter
	cat ${known_vcf}.filter |grep \# > ${known_vcf} &&\
	cat ${known_vcf}.filter |grep 1\/1 >> ${known_vcf}
fi &&\



if [ ! -e ${known_vcf_dir}/${known_vcf_prefix}.idx ]
then java -jar -Xmx${mem}G $gatk IndexFeatureFile -F $known_vcf  #为参考的变异文件构建索引
fi &&\

#标记重复
java -jar -Xmx${mem}G $gatk MarkDuplicates -I ${raw_file_dir}/${sample}_sorted.bam -M ${analysis_dir}/${sample}.markdup_matrics.txt -O ${analysis_dir}/${sample}.sorted.markdup.bam --REMOVE_DUPLICATES true &&\

#对bam文件加头文件
java -jar -Xmx${mem}G $picard AddOrReplaceReadGroups I=${analysis_dir}/${sample}.sorted.markdup.bam O=${analysis_dir}/${sample}.addhead.bam RGLB=${sample}ID RGPL=illumina RGPU=${sample}PU RGSM=${sample} &&\
rm ${analysis_dir}/${sample}.sorted.markdup.bam &&\
rm ${analysis_dir}/${sample}.markdup_matrics.txt &&\

java -jar -Xmx${mem}G $gatk SplitNCigarReads -R $reference -I ${analysis_dir}/${sample}.addhead.bam -O ${analysis_dir}/${sample}.SplitNCigarReads.bam &&\
rm ${analysis_dir}/${sample}.addhead.bam &&\

#碱基质量的重矫正
java -jar -Xmx${mem}G $gatk BaseRecalibrator -I ${analysis_dir}/${sample}.SplitNCigarReads.bam --known-sites $known_vcf -O ${analysis_dir}/${sample}_baserecalibrator.list -R $reference && \
java -jar -Xmx${mem}G $gatk ApplyBQSR -bqsr ${analysis_dir}/${sample}_baserecalibrator.list -I ${analysis_dir}/${sample}.SplitNCigarReads.bam -O ${analysis_dir}/${sample}_applybqsr.bam && \
rm ${analysis_dir}/${sample}.SplitNCigarReads.bam &&\
rm ${analysis_dir}/${sample}.SplitNCigarReads.bai &&\
rm ${analysis_dir}/${sample}_baserecalibrator.list &&\

#变异的检测，生成vcf文件
java -jar -Xmx${mem}G $gatk HaplotypeCaller -I ${analysis_dir}/${sample}_applybqsr.bam -O ${analysis_dir}/${sample}_haplotypecaller.vcf -R $reference --native-pair-hmm-threads 10 &&\
#rm ${analysis_dir}/${sample}_applybqsr.bam &&\
#rm ${analysis_dir}/${sample}_applybqsr.bai &&\

#get the SNP variants
java -jar -Xmx${mem}G $gatk SelectVariants -select-type SNP -O ${analysis_dir}/${sample}_haplotypecaller_SNP.vcf -V ${analysis_dir}/${sample}_haplotypecaller.vcf &&\

#https://zhuanlan.zhihu.com/p/34878471
#这一步就是对变异位点进行质过滤
vcftools --vcf ${analysis_dir}/${sample}_haplotypecaller_SNP.vcf --remove-filtered-all --recode --recode-INFO-all --minDP 20 --out aaa &&\
cat aaa.recode.vcf |grep -v "\.\/\." |grep -v "1\/2" >${analysis_dir}/aaa.vcf &&\
rm aaa.recode.vcf &&\

java -jar -Xmx${mem}G $gatk VariantFiltration -R $reference -V ${analysis_dir}/aaa.vcf -O ${analysis_dir}/${sample}_haplotypecaller_SNP.filted.vcf --filter-name "My_filter" --filter-expression "QD < 10.0 || MQ < 40.0 || FS > 60.0 || SOR > 3.0 || AF < 0.5" &&\
rm aaa.vcf &&\

cat ${analysis_dir}/${sample}_haplotypecaller_SNP.filted.vcf |grep \# >${analysis_dir}/${sample}_haplotypecaller_SNP.filted.PASS.vcf &&\
cat ${analysis_dir}/${sample}_haplotypecaller_SNP.filted.vcf |grep PASS >${analysis_dir}/aaaa.vcf &&\
cat ${analysis_dir}/aaaa.vcf |grep -v \# |cut -f 10 |cut -d : -f 2 |awk -v FS=',' -v OFS="\t" '{$3=$2/($1+$2);print $3}' >${analysis_dir}/vaf.txt &&\
paste ${analysis_dir}/aaaa.vcf ${analysis_dir}/vaf.txt |awk '{if($11>0.9) print}' |cut -f 1-10 >>${analysis_dir}/${sample}_haplotypecaller_SNP.filted.PASS.vcf &&\
java -jar -Xmx${mem}G $gatk IndexFeatureFile -F ${analysis_dir}/${sample}_haplotypecaller_SNP.filted.PASS.vcf &&\

rm ${analysis_dir}/${sample}_haplotypecaller_SNP.filted.vcf &&\
rm ${analysis_dir}/${sample}_haplotypecaller_SNP.filted.vcf.idx &&\
rm ${analysis_dir}/aaaa.vcf &&\
rm ${analysis_dir}/vaf.txt
```

#### 8.变异位点过滤

```shell
sample=$1

vcftools --vcf ${sample}_haplotypecaller_SNP.vcf --remove-filtered-all --recode --recode-INFO-all --minDP 20 --out aaa &&\
cat aaa.recode.vcf |grep -v "\.\/\." |grep -v "1\/2" >aaa.vcf &&\
rm aaa.recode.vcf &&\
rm aaa.log &&\

gatk VariantFiltration \
	-V aaa.vcf \
	-O ${sample}_haplotypecaller_SNP.filted.vcf \
	--filter-name "My_filter" \
	--filter-expression " QD < 10.0 || MQ < 40.0 || FS > 60.0 || SOR > 3.0 || AF < 0.5 " &&\
rm aaa.vcf &&\

cat ${sample}_haplotypecaller_SNP.filted.vcf |grep \# >${sample}_haplotypecaller_SNP.filted.PASS.vcf &&\
cat ${sample}_haplotypecaller_SNP.filted.vcf |grep PASS >aaaa.vcf &&\
cat aaaa.vcf |grep -v \# |cut -f 10 |cut -d : -f 2 |awk -v FS=',' -v OFS="\t" '{$3=$2/($1+$2);print $3}' >vaf.txt &&\
paste aaaa.vcf vaf.txt |awk '{if($11>0.9) print}' |cut -f 1-10 >>${sample}_haplotypecaller_SNP.filted.PASS.vcf &&\
rm aaaa.vcf
#rm vaf.txt
```
