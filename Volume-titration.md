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

    ## -- Attaching packages --------------------------------------- tidyverse 1.3.0 --

    ## v ggplot2 3.3.0     v purrr   0.3.3
    ## v tibble  3.0.0     v dplyr   0.8.5
    ## v tidyr   1.0.2     v stringr 1.4.0
    ## v readr   1.3.1     v forcats 0.5.0

    ## -- Conflicts ------------------------------------------ tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
## Load ggplot theme and defaults
source("R/ggplot_settings.R")

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
    ##              1777              1777              1777              1777 
    ##  Lung_50ul_1_500k  Lung_50ul_4_500k           Doublet          Negative 
    ##              1681              1681                 0                 0

``` r
object <- subset(object, subset=dilution == "DF4" & cellsAtStaining == "1000k")
object
```

    ## An object of class Seurat 
    ## 33572 features across 3554 samples within 3 assays 
    ## Active assay: RNA.kallisto (33514 features)
    ##  2 other assays present: HTO.kallisto, ADT.kallisto
    ##  3 dimensional reductions calculated: pca, tsne, umap

``` r
Idents(object) <- object$fineCluster
```

## Cell type and tissue overview

``` r
p.tsne.volume <- DimPlot(object, group.by="volume", reduction="tsne", pt.size=0.1, combine=FALSE)[[1]] + theme_get() + facet_wrap(~"Volume") + scale_color_manual(values=color.volume)

p.tsne.cluster <- DimPlot(object, group.by="supercluster", reduction="tsne", pt.size=0.1, combine=FALSE)[[1]] + theme_get() + scale_color_manual(values=color.supercluster) + facet_wrap(~"Cell types")

p.tsne.finecluster <- DimPlot(object, label=TRUE, label.size=3, reduction="tsne", group.by="fineCluster", pt.size=0.1, combine=FALSE)[[1]] + theme_get() + facet_wrap(  ~"Clusters") + guides(col=F)
```

    ## Warning: Using `as.character()` on a quosure is deprecated as of rlang 0.3.0.
    ## Please use `as_label()` or `as_name()` instead.
    ## This warning is displayed once per session.

``` r
p.tsne.cluster + p.tsne.finecluster + p.tsne.volume
```

![](Volume-titration_files/figure-gfm/tsnePlots-1.png)<!-- -->

## Overall ADT counts

Samples stained in reduced volume have slightly reduced ADT counts.

``` r
ADTcount.Abtitration <- data.frame(FetchData(object, vars=c("volume")), count=apply(GetAssayData(object, assay="ADT.kallisto", slot="counts"),2,sum)) %>% group_by(volume) %>% summarise(sum=sum(count))

p.ADTcount.Abtitration <- ggplot(ADTcount.Abtitration, aes(x=volume, y=sum/10^3, fill=volume)) + geom_bar(stat="identity", col="black") + scale_fill_manual(values=color.volume) + scale_y_continuous(expand=c(0,0,0,0.05)) + guides(fill=FALSE) + ylab("ADT UMI count (x10^6)") + theme(panel.grid.major=element_blank(), axis.title.x=element_blank(), panel.border=element_blank(), axis.line = element_line())

p.ADTcount.Abtitration
```

![](Volume-titration_files/figure-gfm/ADTcounts-1.png)<!-- -->

## ADT sum rank by cell

Samples stained in reduced volume have reduced ADT counts largely evenly
distributed among cells

``` r
source("R/foot_plot.R")
p.ADTrank.Abtitration <- foot_plot(data=object$nCount_ADT.kallisto, group=object$group, linetype=object$volume, barcodeGroup=object$supercluster, draw.line=TRUE, draw.barcode=TRUE, draw.points=FALSE, draw.fractile=FALSE, trans="log10", barcode.stepSize = 0.1, colors=color.supercluster) + 
  labs(linetype="Volume", color="Cell type", y="ADT UMI count")
```

    ## Loading required package: ggrepel

``` r
p.ADTrank.Abtitration
```

![](Volume-titration_files/figure-gfm/ADTrank-1.png)<!-- -->

## Load Ab panel annotation and concentrations

``` r
abpanel <- data.frame(readxl::read_excel("data/Supplementary_Table_1.xlsx"))
rownames(abpanel) <- abpanel$Marker

markerStats <- read.table("data/markerByClusterStats.tsv")
markerStats.PBMC <- markerStats[markerStats$tissue == "PBMC",]
marker.order <- markerStats.PBMC$marker[order(-markerStats.PBMC$conc_µg_per_mL, -markerStats.PBMC$UMItotal)]
```

``` r
ADT.matrix <- data.frame(GetAssayData(object, assay="ADT.kallisto", slot="counts"))
ADT.matrix$marker <- rownames(ADT.matrix)
ADT.matrix$conc <- abpanel[ADT.matrix$marker,"conc_µg_per_mL"]
ADT.matrix <- ADT.matrix %>% pivot_longer(c(-marker,-conc))
cell.annotation <- FetchData(object, vars=c("volume"))

ADT.matrix.agg <- ADT.matrix %>% group_by(volume=cell.annotation[name,"volume"], marker, conc) %>% summarise(sum=sum(value))

ADT.matrix.agg$marker.byConc <- factor(ADT.matrix.agg$marker, levels=marker.order)

ann.markerConc <- abpanel[marker.order,]
ann.markerConc$Marker <- factor(marker.order, levels=marker.order)

lines <- length(marker.order)-cumsum(sapply(split(ann.markerConc[rev(marker.order),"Marker"],ann.markerConc[rev(marker.order),"conc_µg_per_mL"]),length))+0.5
lines <- data.frame(breaks=lines[-length(lines)])

p1 <- ggplot(ann.markerConc, aes(x=1, y=Marker, fill=conc_µg_per_mL)) + 
  geom_tile(col=alpha(col="black",alpha=0.2)) + 
  geom_hline(data=lines,aes(yintercept=breaks), linetype="dashed", alpha=0.5) + 
  scale_fill_viridis_c(trans="log2") + 
  scale_x_continuous(expand=c(0,0)) + 
  labs(fill="µg/mL") + 
  theme_get() + 
  theme(axis.ticks.x=element_blank(), axis.title = element_blank(), axis.text.x=element_blank(), panel.grid=element_blank(), legend.position="right", plot.margin=unit(c(0.1,0.1,0.1,0.1),"mm"))

p2 <- ggplot(ADT.matrix.agg, aes(x=marker.byConc,y=log2(sum))) + 
  geom_line(aes(group=marker), size=1.2, color="#666666") + 
  geom_point(aes(group=volume, fill=volume), pch=21, size=0.7) + 
  geom_vline(data=lines,aes(xintercept=breaks), linetype="dashed", alpha=0.5) + 
  scale_fill_manual(values=color.volume) +
  scale_y_continuous(breaks=c(9:17)) + 
  ylab("log2(UMI sum)") + 
  theme(axis.title.y=element_blank(), axis.text.y=element_blank(), legend.position="right", legend.justification="left", legend.title.align=0, legend.key.width=unit(0.2,"cm")) + 
  coord_flip()

library(patchwork)
titration.totalCount <- p1 + guides(fill=F) + p2 + guides(fill=F) + plot_spacer() + guide_area() + plot_layout(ncol=4, widths=c(1,30,0.1), guides='collect')

titration.totalCount
```

![](Volume-titration_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

## Overall ADT counts

Samples stained with diluted Ab panel have reduced ADT counts.

``` r
scaleFUN <- function(x) sprintf("%.2f", x)

p.ADTcount.AbtitrationByConc <- ggplot(ADT.matrix.agg[order(-ADT.matrix.agg$conc, -ADT.matrix.agg$sum),], aes(x=volume, y=sum/10^6, fill=conc)) + 
  geom_bar(stat="identity", col=alpha(col="black",alpha=0.05)) + 
  scale_fill_viridis_c(trans="log2", labels=scaleFUN, breaks=c(0.0375,0.15,0.625,2.5,10)) + 
  scale_y_continuous(expand=c(0,0,0,0.05)) + 
  labs(fill="µg/mL", y=bquote("ADT UMI counts ("~10^6~")")) + 
  guides(fill=guide_colourbar(reverse=T)) + 
  theme(panel.grid.major=element_blank(), axis.title.x=element_blank(), panel.border=element_blank(), axis.line = element_line(), legend.position="right")

p.ADTcount.AbtitrationByConc
```

![](Volume-titration_files/figure-gfm/ADTcountsByConc-1.png)<!-- -->

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

cell.annotation <- FetchData(object, vars=c("volume", "fineCluster"))

ADT.matrix.agg <- ADT.matrix %>% group_by(volume=cell.annotation[name,"volume"], fineCluster=cell.annotation[name,"fineCluster"], marker, conc) %>% summarise(sum=sum(value), median=quantile(value, probs=c(0.9)), nth=nth(value))

Cluster.max <- markerStats.PBMC[,c("marker","fineCluster")]
Cluster.max$fineCluster <- factor(Cluster.max$fineCluster)

ADT.matrix.aggByClusterMax <- Cluster.max %>% left_join(ADT.matrix.agg)
```

    ## Joining, by = c("marker", "fineCluster")

    ## Warning: Column `fineCluster` joining factors with different levels, coercing to
    ## character vector

``` r
ADT.matrix.aggByClusterMax$marker.byConc <- factor(ADT.matrix.aggByClusterMax$marker, levels=marker.order)

p3 <- ggplot(ADT.matrix.aggByClusterMax, aes(x=marker.byConc, y=log2(nth))) + 
  geom_line(aes(group=marker), size=1.2, color="#666666") + 
  geom_point(aes(group=volume, fill=volume), pch=21, size=0.7) + 
  geom_vline(data=lines,aes(xintercept=breaks), linetype="dashed", alpha=0.5) + 
  geom_text(aes(label=paste0(fineCluster," ")), y=Inf, adj=1, size=1.5) + 
  scale_fill_manual(values=color.volume) + 
  scale_y_continuous(breaks=c(0:11), labels=2^c(0:11), expand=c(0.05,0.5)) +
  ylab("90th percentile UMI of expressing cluster") + 
  theme(axis.title.y=element_blank(), axis.text.y=element_blank(), legend.position="right", legend.justification="left", legend.title.align=0, legend.key.width=unit(0.2,"cm")) + 
  coord_flip()


titration.clusterCount <- p1 + theme(legend.position="none") + p3 + theme(legend.position="none") + plot_spacer() + plot_layout(ncol=4, widths=c(1,30,0.1), guides='collect')

titration.clusterCount
```

![](Volume-titration_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

## Titration examples

``` r
curMarker.name <- "CD31"
curMarker.DF1conc <- abpanel[curMarker.name, "conc_µg_per_mL"]

p.CD31 <- foot_plot_density_custom(marker=paste0("adt_",curMarker.name), wrap=NULL, group="volume", color=color.volume, legend=TRUE)
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

    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.

``` r
p.CD31
```

![](Volume-titration_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

## Final plot

``` r
library("cowplot")
A <- p.ADTcount.AbtitrationByConc + theme(legend.key.width=unit(0.3,"cm"), legend.key.height=unit(0.4,"cm"), legend.text=element_text(size=unit(5,"pt")), plot.margin=unit(c(0.3,0,0.5,0),"cm"))

D <- p.CD31 + theme(plot.margin=unit(c(0.5,0,0,0),"cm"))


B1 <- p1 + theme(text = element_text(size=10), plot.margin=unit(c(0.3,0,0,0),"cm"))
B.legend <- cowplot::get_legend(B1)
B1 <- B1 + theme(legend.position="none")
B2 <- p2
B2 <- B2 + theme(legend.position="none")
C1 <- p3 + theme(legend.position="none")


ADE <- cowplot::plot_grid(A,D,NULL, labels=c("A","D",""), label_size=panel.label_size, vjust=panel.label_vjust, hjust=panel.label_hjust, ncol=1, rel_heights = c(13,17,1.5))
BC <- cowplot::plot_grid(B1, B2, C1, nrow=1, align="h", axis="tb", labels=c("B", "", "C"), label_size=panel.label_size, vjust=panel.label_vjust, hjust=panel.label_hjust, rel_widths=c(2,10,10))

png(file=file.path(outdir,"Figure 3.png"), width=figure.width.full, height=4.5, units = figure.unit, res=figure.resolution, antialias=figure.antialias)
plot_grid(ADE, BC, nrow=1, rel_widths=c(1,4), align="v", axis="l")
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
  plot <- foot_plot_density_custom(marker=paste0("adt_",curMarker.name), wrap=NULL, group="volume", color=color.volume, legend=FALSE)
  plots[[i]] <- plot
}
```

    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.
    ## Scale for 'fill' is already present. Adding another scale for 'fill', which
    ## will replace the existing scale.

``` r
# a bit of a hack to get celltype legend
p.withLegend <- ggplot(data.frame(supercluster=object$supercluster),aes(color=supercluster,x=1,y=1)) + geom_point(shape=15, size=1.5)
p.legend <- cowplot::get_legend(p.withLegend + theme(legend.title=element_blank(),legend.margin=margin(0,0,0,0), legend.key.size = unit(0.15,"cm"),legend.position = c(0.98,1.1), legend.justification=c(1,1), legend.direction="horizontal"))

png(file=file.path(outdir,"Supplementary Figure 2A.png"), units=figure.unit, res=figure.resolution, width=figure.width.full, height=10, antialias=figure.antialias)
curPlots <- plot_grid(plotlist=plots[1:30],ncol=6, align="h", axis="tb")
cowplot::plot_grid(NULL, p.legend, curPlots, vjust=-0.5, hjust=panel.label_hjust, label_size=panel.label_size, ncol=1, rel_heights= c(0.5, 1.3, 70))
dev.off()
```

    ## png 
    ##   2

``` r
png(file=file.path(outdir,"Supplementary Figure 2B.png"), units=figure.unit, res=figure.resolution, width=figure.width.full, height=10, antialias=figure.antialias)
curPlots <- plot_grid(plotlist=plots[31:52],ncol=6, align="h", axis="tb")
cowplot::plot_grid(NULL, p.legend, curPlots, NULL, vjust=-0.5, hjust=panel.label_hjust, label_size=panel.label_size, ncol=1, rel_heights= c(0.5, 1.3, 70*(4/5),70*(1/5)))
dev.off()
```

    ## png 
    ##   2