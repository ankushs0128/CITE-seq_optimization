CITE-seq optimization - Volume titration
================
Terkild Brink Buus
30/3/2020

## Load libraries etc.

``` r
set.seed(114)
require("Seurat")
```

    ## Loading required package: Seurat

``` r
require("tidyverse")
```

    ## Loading required package: tidyverse

    ## -- Attaching packages -------------------------------------------------------------------------------- tidyverse 1.3.0 --

    ## v ggplot2 3.3.0     v purrr   0.3.3
    ## v tibble  2.1.3     v dplyr   0.8.5
    ## v tidyr   1.0.2     v stringr 1.4.0
    ## v readr   1.3.1     v forcats 0.5.0

    ## -- Conflicts ----------------------------------------------------------------------------------- tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
theme_set(theme_bw() + 
          theme(
            axis.text.x=element_text(angle=45, hjust=1), 
            panel.grid.minor = element_blank(), 
            strip.background=element_blank(), 
            strip.text=element_text(face="bold", size=10), 
            legend.position = "bottom"))

## Load helper functions (ggplot themes, biexp transformation etc.)
source("R/Utilities.R")

## Load predefined color schemes
source("R/color.R")

outdir <- "C:/Users/Terkild/OneDrive - Københavns Universitet/Koralovlab/ECCITE-seq/20200106 Titration 1"
```

## Load Seurat object

``` r
object <- readRDS(file="data/5P-CITE-seq_Titration.rds")

## Show number of cells from each sample
table(object$group)
```

    ## 
    ## PBMC_50ul_1_1000k PBMC_50ul_4_1000k PBMC_25ul_4_1000k  PBMC_25ul_4_200k 
    ##              1881              2167              2447              3070 
    ##  Lung_50ul_1_500k  Lung_50ul_4_500k           Doublet          Negative 
    ##              1876              1897                 0                 0

``` r
#' For some reason, the proportion of monocytes are higher in the 200k sample
#' To avoid this skewing comparisons, we subsample according to fineClusters, 
#' We stick with 2100 cells as this was used for the 1000k sample in the previous comparison
object <- subset(object, subset=dilution == "DF4" & volume == "25µl")
object.1000k <- subset(object, subset=cellsAtStaining=="1000k")
object.1000k <- subset(object.1000k, cells=sample(ncol(object.1000k),2100))
object.200k <- subset(object, subset=cellsAtStaining=="200k")

rbind(table(object.1000k$fineCluster),table(object.200k$fineCluster))
```

    ##        0   1   2   3   4 5   6 7 8   9  10 11 12 13 14 15 16 17 18 19 20 21 22
    ## [1,] 664 270 236 197 201 1 164 0 6 100  93 22  3  0  1 43 40  5 31  0  6  0  3
    ## [2,] 750 316 426 418 420 4 193 4 6 115 134 30 17  0  2 92 66  8 46  0  4  0  0
    ##      23 24
    ## [1,]  5  9
    ## [2,] 12  7

``` r
clusters <- levels(object$fineCluster)
cells.1000k <- c()
cells.200k <- c()
for(i in seq_along(clusters)){
  curCluster <- clusters[i]
  cluster.cells.1000k <- which(object.1000k$fineCluster == curCluster)
  cluster.cells.200k <- which(object.200k$fineCluster == curCluster)
  cluster.min.count <- min(c(length(cluster.cells.1000k),length(cluster.cells.200k)))
  
  cells.1000k <- append(cells.1000k,cluster.cells.1000k[sample(length(cluster.cells.1000k),cluster.min.count)])
  cells.200k <- append(cells.200k,cluster.cells.200k[sample(length(cluster.cells.200k),cluster.min.count)])
}

length(cells.1000k)
```

    ## [1] 2093

``` r
length(cells.200k)
```

    ## [1] 2093

``` r
cells <- c(colnames(object.1000k)[cells.1000k],colnames(object.200k)[cells.200k])

object <- subset(object, cells=cells[sample(length(cells),length(cells))])
DimPlot(object, split.by="cellsAtStaining", reduction="tsne")
```

![](Cell-number-titration_files/figure-gfm/loadSeurat-1.png)<!-- -->

``` r
rm(object.1000k)
rm(object.200k)


## While cell numbers are close, comparison is easier if we downsample to the same number of cells
#Idents(object) <- object$group
#object <- subset(object, subset=dilution == "DF4" & volume == "25µl", downsample=2100)
#object

Idents(object) <- object$fineCluster
table(Idents(object),as.character(object$group))
```

    ##     
    ##      PBMC_25ul_4_1000k PBMC_25ul_4_200k
    ##   0                664              664
    ##   1                270              270
    ##   2                236              236
    ##   3                197              197
    ##   4                201              201
    ##   5                  1                1
    ##   6                164              164
    ##   8                  6                6
    ##   9                100              100
    ##   10                93               93
    ##   11                22               22
    ##   12                 3                3
    ##   14                 1                1
    ##   15                43               43
    ##   16                40               40
    ##   17                 5                5
    ##   18                31               31
    ##   20                 4                4
    ##   23                 5                5
    ##   24                 7                7

## Cell type and tissue overview

``` r
p.tsne.number <- DimPlot(object, group.by="cellsAtStaining", reduction="tsne", combine=FALSE)[[1]] + theme_get() + facet_wrap(~"Volume") + scale_color_manual(values=color.number)

p.tsne.cluster <- DimPlot(object, group.by="supercluster", reduction="tsne", combine=FALSE)[[1]] + theme_get() + scale_color_manual(values=color.supercluster) + facet_wrap(~"Cell types")

p.tsne.finecluster <- DimPlot(object, label = T, reduction="tsne", group.by="fineCluster", combine=FALSE)[[1]] + theme_get() + facet_wrap(  ~"Clusters") + guides(col=F)
```

    ## Warning: Using `as.character()` on a quosure is deprecated as of rlang 0.3.0.
    ## Please use `as_label()` or `as_name()` instead.
    ## This warning is displayed once per session.

``` r
p.tsne.cluster + p.tsne.finecluster + p.tsne.number
```

![](Cell-number-titration_files/figure-gfm/tsnePlots-1.png)<!-- -->

## Overall ADT counts

Samples stained with diluted Ab panel have reduced ADT counts.

``` r
ADTcount.Abtitration <- data.frame(FetchData(object, vars=c("cellsAtStaining")), count=apply(GetAssayData(object, assay="ADT.kallisto", slot="counts"),2,sum)) %>% group_by(cellsAtStaining) %>% summarise(sum=sum(count))

p.ADTcount.Abtitration <- ggplot(ADTcount.Abtitration, aes(x=cellsAtStaining, y=sum/10^3, fill=cellsAtStaining)) + geom_bar(stat="identity", col="black") + scale_fill_manual(values=color.number) + scale_y_continuous(expand=c(0,0,0,0.05)) + guides(fill=FALSE) + ylab("ADT UMI count (x10^6)") + theme(panel.grid.major=element_blank(), axis.title.x=element_blank(), panel.border=element_blank(), axis.line = element_line())

p.ADTcount.Abtitration
```

![](Cell-number-titration_files/figure-gfm/ADTcounts-1.png)<!-- -->

## ADT sum rank by cell

Samples stained with diluted Ab panel have reduced ADT counts evenly
distributed among cells

``` r
source("R/foot_plot.R")
p.ADTrank.Abtitration <- foot_plot(data=object$nCount_ADT.kallisto, group=object$group, linetype=object$cellsAtStaining, barcodeGroup=object$supercluster, draw.line=TRUE, draw.barcode=TRUE, draw.points=FALSE, draw.fractile=FALSE, trans="log10", barcode.stepSize = 0.1, colors=color.supercluster) + labs(linetype="Number", color="Cell type") + ylab("ADT UMI count")
```

    ## Loading required package: ggrepel

``` r
p.ADTrank.Abtitration
```

![](Cell-number-titration_files/figure-gfm/ADTrank-1.png)<!-- -->

## Load Ab panel annotation and concentrations

``` r
abpanel <- data.frame(readxl::read_excel("data/Supplementary_Table_1.xlsx"))
rownames(abpanel) <- abpanel$Marker
```

``` r
ADT.matrix <- data.frame(GetAssayData(object, assay="ADT.kallisto", slot="counts"))
ADT.matrix$marker <- rownames(ADT.matrix)
ADT.matrix$conc <- abpanel[ADT.matrix$marker,"conc_µg_per_mL"]
ADT.matrix <- ADT.matrix %>% pivot_longer(c(-marker,-conc))
cell.annotation <- FetchData(object, vars=c("cellsAtStaining"))

ADT.matrix.agg <- ADT.matrix %>% group_by(cellsAtStaining=cell.annotation[name,"cellsAtStaining"], marker, conc) %>% summarise(sum=sum(value))

marker.order <- ADT.matrix.agg$marker[order(-ADT.matrix.agg$conc[ADT.matrix.agg$cellsAtStaining == "200k"], ADT.matrix.agg$sum[ADT.matrix.agg$cellsAtStaining == "200k"])]

ADT.matrix.agg$marker.byConc <- factor(ADT.matrix.agg$marker, levels=marker.order)

ann.markerConc <- abpanel[marker.order,]
ann.markerConc$Marker <- factor(marker.order, levels=marker.order)

lines <- length(marker.order)-cumsum(sapply(split(ann.markerConc[rev(marker.order),"Marker"],ann.markerConc[rev(marker.order),"conc_µg_per_mL"]),length))+0.5
lines <- data.frame(breaks=lines[-length(lines)])

p1 <- ggplot(ann.markerConc, aes(x=Marker, y=1, fill=conc_µg_per_mL)) + geom_bar(stat="identity")+ scale_fill_viridis_c(trans="log2") + theme_minimal() + theme(axis.title = element_blank(), axis.text.x=element_blank(), panel.grid=element_blank(), legend.position="right", plot.margin=unit(c(0,0,0,0),"cm")) + scale_y_continuous(expand=c(0,0)) + labs(fill="µg/mL") + geom_vline(data=lines,aes(xintercept=breaks), linetype="dashed", alpha=0.5) + coord_flip()

p2 <- ggplot(ADT.matrix.agg, aes(x=marker.byConc,y=log2(sum))) + geom_line(aes(group=marker), size=2, color="#666666") + geom_point(aes(group=cellsAtStaining, fill=cellsAtStaining), pch=21) + scale_fill_manual(values=color.number) + geom_vline(data=lines,aes(xintercept=breaks), linetype="dashed", alpha=0.5) + theme(axis.title.y=element_blank(), axis.text.y=element_blank(), legend.position="right", legend.justification="left", legend.title.align=0, legend.key.width=unit(0.2,"cm")) + ylab("log2(UMI sum)") + scale_y_continuous(breaks=c(9:17)) + coord_flip()

library(patchwork)
titration.totalCount <- p1 + guides(fill=F) + p2 + guides(fill=F) + plot_spacer() + guide_area() + plot_layout(ncol=4, widths=c(1,30,0.1), guides='collect')

titration.totalCount
```

![](Cell-number-titration_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

## Overall ADT counts

Samples stained with diluted Ab panel have reduced ADT counts.

``` r
scaleFUN <- function(x) sprintf("%.2f", x)

p.ADTcount.AbtitrationByConc <- ggplot(ADT.matrix.agg[order(-ADT.matrix.agg$conc, ADT.matrix.agg$sum),], aes(x=cellsAtStaining, y=sum/10^6, fill=conc)) + geom_bar(stat="identity", col=alpha(col="black",alpha=0.05)) + scale_fill_viridis_c(trans="log2", labels=scaleFUN, breaks=c(0.0375,0.15,0.625,2.5,10)
) + scale_y_continuous(expand=c(0,0,0,0.05)) + guides() + ylab("UMI count (x10^6)") + theme(panel.grid.major=element_blank(), axis.title.x=element_blank(), panel.border=element_blank(), axis.line = element_line(), legend.position="right") + labs(fill="µg/mL") + guides(fill=guide_colourbar(reverse=T))

p.ADTcount.AbtitrationByConc
```

![](Cell-number-titration_files/figure-gfm/ADTcountsByConc-1.png)<!-- -->

Plot change by dilution. We modified our approach of using a specific
fractile as this may be prone to outliers in small clusters (i.e. the
0.9 fractile of a cluster of 30 will be the \#3 higest cell making it
prone to outliers). We thus set a threshhold of the value to be lower
than the expression of the 10th highest cell value within the cluster
with highest expression.

``` r
ADT.matrix <- data.frame(GetAssayData(object, assay="ADT.kallisto", slot="counts"))
ADT.matrix$marker <- rownames(ADT.matrix)
ADT.matrix$conc <- abpanel[ADT.matrix$marker,"conc_µg_per_mL"]
ADT.matrix <- ADT.matrix %>% pivot_longer(c(-marker,-conc))

cell.annotation <- FetchData(object, vars=c("cellsAtStaining", "fineCluster"))


## nth function extracts the value at a set fractile or at the 10th event depending on which is lowest
nth <- function(value, nth=10, fractile=0.9){
  if(length(value)*(1-fractile) <= nth){
    newvalue <- sort(value, decreasing=TRUE)[nth]
  } else {
    newvalue <- quantile(value, probs=c(fractile))
  }
  return(newvalue)
}

ADT.matrix.agg <- ADT.matrix %>% group_by(cellsAtStaining=cell.annotation[name,"cellsAtStaining"], ident=cell.annotation[name,"fineCluster"], marker, conc) %>% summarise(sum=sum(value), median=quantile(value, probs=c(0.9)), nth=nth(value))

Cluster.max <- ADT.matrix.agg %>% group_by(marker) %>% summarize(ident=ident[which.max(nth)])

ADT.matrix.aggByClusterMax <- Cluster.max %>% left_join(ADT.matrix.agg)
```

    ## Joining, by = c("marker", "ident")

``` r
ADT.matrix.aggByClusterMax$marker.byConc <- factor(ADT.matrix.aggByClusterMax$marker, levels=marker.order)

p3 <- ggplot(ADT.matrix.aggByClusterMax, aes(x=marker.byConc, y=log2(nth))) + geom_line(aes(group=marker), size=2, color="#666666") + geom_point(aes(group=cellsAtStaining, fill=cellsAtStaining), pch=21) + scale_fill_manual(values=color.number) + geom_vline(data=lines,aes(xintercept=breaks), linetype="dashed", alpha=0.5) + theme(axis.title.y=element_blank(), axis.text.y=element_blank(), legend.position="right", legend.justification="left", legend.title.align=0, legend.key.width=unit(0.2,"cm")) + ylab("90th percentile UMI of expressing cluster") + scale_y_continuous(breaks=c(0:11), labels=2^c(0:11), expand=c(0.05,0.5)) + coord_flip() +
## To add a label indicating which cluster is used in the calculation, add:
  geom_text(aes(label=ident), y=Inf, adj=1, size=2.5)

titration.clusterCount <- p1 + theme(legend.position="none") + p3 + theme(legend.position="none") + plot_spacer() + plot_layout(ncol=4, widths=c(1,30,0.1), guides='collect')

titration.clusterCount
```

![](Cell-number-titration_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

## Titration examples

``` r
curMarker.name <- "CD31"
curMarker.DF1conc <- abpanel[curMarker.name, "conc_µg_per_mL"]

p.CD31 <- foot_plot_density_custom(marker=paste0("adt_",curMarker.name), wrap=NULL, group="cellsAtStaining", color=color.number, legend=TRUE)
```

    ## 
    ## ********************************************************

    ## Note: As of version 1.0.0, cowplot does not change the

    ##   default ggplot2 theme anymore. To recover the previous

    ##   behavior, execute:
    ##   theme_set(theme_cowplot())

    ## ********************************************************

    ## 
    ## Attaching package: 'cowplot'

    ## The following object is masked from 'package:patchwork':
    ## 
    ##     align_plots

``` r
p.CD31
```

![](Cell-number-titration_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

## Final plot

``` r
library("cowplot")
A <- p.ADTcount.AbtitrationByConc + theme(legend.key.width=unit(0.3,"cm"), legend.text = element_text(size=8))#p.ADTcount.Abtitration

D <- p.CD31 + theme(plot.margin=unit(c(0.5,0,0,0),"cm"))


B1 <- p1 + theme(text = element_text(size=10))
B.legend <- cowplot::get_legend(B1)
B1 <- B1 + theme(legend.position="none")
B2 <- p2
B2 <- B2 + theme(legend.position="none")
C1 <- p3 + theme(legend.position="none")

ADE <- cowplot::plot_grid(A,D,NULL, labels=c("A","D",""), hjust=0, vjust=-0.7, ncol=1, rel_heights = c(10,20,1.5))
BC <- cowplot::plot_grid(B1, B2, C1, nrow=1, align="h", axis="tb", labels=c("E", "", "F"), hjust=0, vjust=-0.5, rel_widths=c(2,10,10))

png(file=file.path(outdir,"Figure 4.png"), width=10, height=6, units = "in", res=300)
ggdraw(cowplot::plot_grid(NULL, cowplot::plot_grid(ADE, BC, nrow=1, rel_widths=c(1,4), align="v", axis="l"), ncol=1, rel_heights=c(1,10))) + draw_label(label="Figure 4",x=0, y=1, hjust=0, vjust=0.98, size = 20)
dev.off()
```

    ## png 
    ##   2

## Print individual titration plots (for each marker)

For supplementary figure

``` r
plots <- list()
markers <- intersect(abpanel[order(-abpanel$conc_µg_per_mL, abpanel$Marker),"Marker"],rownames(object[["ADT.kallisto"]]))

for(i in seq_along(markers)){
  curMarker.name <- markers[i]
  curLegend <- ifelse(i %in% c(1,31),TRUE,FALSE)
  plot <- foot_plot_density_custom(marker=paste0("adt_",curMarker.name), wrap=NULL, group="cellsAtStaining", color=color.volume, legend=curLegend)
  plots[[i]] <- plot
}

pdf(file=file.path(outdir,"Supplementary Figure 3 Numbers.pdf"), width=16, height=22)
ggdraw(plot_grid(NULL, plot_grid(plotlist=plots[1:30],ncol=6, align="h", axis="tb"), ncol=1, rel_heights=c(1,70))) + draw_label(label="Figure S3A",x=0, y=1, hjust=0, vjust=0.98, size = 20)
ggdraw(plot_grid(NULL, plot_grid(plotlist=plots[31:52],ncol=6, align="h", axis="tb"), NULL, ncol=1, rel_heights=c(1,70*(4/5),70*(1/5)))) + draw_label(label="Figure S3B",x=0, y=1, hjust=0, vjust=0.98, size = 20)
dev.off()
```

    ## png 
    ##   2