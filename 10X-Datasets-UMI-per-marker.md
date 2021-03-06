CITE-seq optimization - 10X datasets: UMI per marker plot
================
Terkild Brink Buus
30/3/2020

## Load utilities

Including libraries, plotting and color settings and custom utility
functions

``` r
set.seed(114)
require("tidyverse", quietly=T)
library("Matrix", quietly=T)

## Load ggplot theme and defaults
source("R/ggplot_settings.R")

## Load helper functions
source("R/Utilities.R")

outdir <- "figures"
data.10X <- "data/data.10X.datasets.Rdata"
```

## Load data

10X datasets were preprocessed in the [Load unfiltered data
vignette](Load-unfiltered-data.md)

``` r
load(file=data.10X)
```

## Draw UMI per marker

These three 10X dataset used the same panel of antibodies at three
conditions, 3’ V3 chemistry at \~1,000 and \~10,000 cells or 5’ at
10,000 cells using TotalSeqB or TotalSeqC antibodies, respectively.

``` r
## Extract data from list into a combined data.frame
for(i in seq_along(data.10X.datasets)){
  dataset <- data.10X.datasets[i]
  
  kallisto <- data.10X.datasets.adt.kallisto[[dataset]]
  cells <- intersect(data.10X.datasets.gex.aboveInf[[dataset]],colnames(kallisto))
  
  
  total <- Matrix::rowSums(kallisto)
  Cell <- Matrix::rowSums(kallisto[,cells])
  EmptyDrop <- total-Cell

  add <- data.frame(Dataset=dataset,Marker=names(Cell),Cell,EmptyDrop)
  
  if(i == 1){
    plotData <- add
  } else {
    plotData <- rbind(plotData,add)
  }
}

## Convert data into "long format" for plotting with ggplot
plotData <- plotData %>% pivot_longer(c(-Marker, -Dataset))

## Rename isotype controls to get shorter names
plotData$Marker <- gsub("isotype_control_","",plotData$Marker)
plotData$subset <- factor(as.character(plotData$name), levels=c("EmptyDrop","Cell"))
plotData$Dataset <- factor(as.character(plotData$Dataset), levels=data.10X.datasets)

## Make plot
data.10X.markerBarplot <- ggplot(plotData, aes(x=Marker, y=value/10^6, fill=subset)) + 
  geom_bar(stat="identity", position="dodge", color="black", width=0.65) + 
  scale_y_continuous(expand=c(0,0,0.05,0)) + 
  scale_fill_manual(values=c("lightgrey","black")) + 
  labs(y=bquote("ADT UMI counts ("~10^6~")")) + 
  coord_flip() + 
  facet_wrap(~Dataset, nrow=1, scales="free_x") + 
  theme(axis.title.y=element_blank(), 
        legend.position=c(1,0.98), 
        legend.justification=c(1,1), 
        legend.title=element_blank(),
        legend.background=element_blank())
```

## Final figure

``` r
## Include knee_plots from preprocessing in the figure
data.10X.GEXrank <- cowplot::plot_grid(plotlist=data.10X.datasets.knee_plots, 
                                       labels=data.10X.datasets, 
                                       hjust=-0.65, 
                                       vjust=1.6, 
                                       label_size=7, 
                                       nrow=1)

p.figure <- cowplot::plot_grid(data.10X.GEXrank, data.10X.markerBarplot, 
                               labels=c("A", "B"), 
                               ncol=1, 
                               rel_heights=c(2,3), 
                               label_size=panel.label_size, 
                               vjust=panel.label_vjust, 
                               hjust=panel.label_hjust)


png(file=file.path(outdir,"Supplementary Figure S6.png"), 
    width=figure.width.full, 
    height=5, 
    units = figure.unit, 
    res=figure.resolution, 
    antialias=figure.antialias)

  p.figure

dev.off()
```

    ## png 
    ##   2

``` r
p.figure
```

![](10X-Datasets-UMI-per-marker_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->
