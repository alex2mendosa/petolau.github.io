---
title: "CoronaDash app use case - Clustering countries' COVID-19 active cases trajectories"
author: "Peter Laurinec"
layout: post
published: true
status: publish
tags: test clustering shiny opendata
draft: no
---
 

 
COVID-19 disease spread hit the World really globally and also field of mathematicians/ statisticians/ machine learning researchers and related.
These experts want to help to understand for example future trends (forecast) of corona virus spread.
My motivation in this case was to create **interactive dashboard** about COVID-19 to inform about various scenarios in every country in the World through **data mining methods**.
 
I created **CoronaDash** `shinydashboard` application that is hosted on [**petolau.shinyapps.io** RStudio platform](https://petolau.shinyapps.io/coronadash/).
The dashboard provides various data mining/ visualization techniques for **comparing countries' COVID-19 data statistics** as:
 
  * extrapolating total confirmed cases by exponential smoothing model,
  * trajectories of cases/ deaths spread,
  * multidimensional clustering of countries' data/ statistics - with dendogram and table of clusters averages,
  * aggregated views for the whole World,
  * hierarchical clustering of countries' trajectories based on DTW distance and preprocessing by SMA (+ normalization), for fast comparison of large number of countries' COVID-19 magnitudes and trends.
 
The blog post will be about the last bullet of the above list - **clustering of countries' trajectories**.
This use case is challenging because of **clustering time series with different lengths**.
 
#### CovidR contest
 
I submitted my shiny application also to interesting initiative of eRum 2020 organizers - [**CovidR Contest**](https://milano-r.github.io/erum2020-covidr-contest/index.html).
If you like my contribution, please vote for me at [**erum2020-covidr-contest/laurinec-CoronaDash**](https://milano-r.github.io/erum2020-covidr-contest/laurinec-CoronaDash.html) gallery post :)
 
## Preprocessing of COVID-19 open-data
 
Firstly, load all the needed packages.

{% highlight r %}
library(data.table) # data handling
library(TSrepr) # time series representation, pls use dev version devtools::install_github("PetoLau/TSrepr")
library(ggplot2) # visualisations
library(dtwclust) # clustering using DTW distance
library(dygraphs) # interactive visualizations
library(ggrepel) # nice labels in ggplot
library(dendextend) # dendrograms
library(DT) # nice datatable
{% endhighlight %}
 
Data are coming from [Johns Hopkins CSSE GitHub repository](https://github.com/CSSEGISandData/COVID-19/tree/master/csse_covid_19_data/csse_covid_19_time_series), [GitHub repository by ulklc](https://github.com/ulklc/covid19-timeseries), and tests data are coming from [COVID19 API](https://github.com/ChrisMichaelPerezSantiago/covid19).
 
Read prepared data at *2020-05-24* snapshot.

{% highlight r %}
data_covid_ts <- fread("_rmd/data_covid_time_series_2020-05-24.csv")
data_covid_ts[, DateRep := as.Date(DateRep)] # transform character to Date class
 
# what we got
str(data_covid_ts)
{% endhighlight %}



{% highlight text %}
## Classes 'data.table' and 'data.frame':	19344 obs. of  17 variables:
##  $ Country                                       : chr  "Afghanistan" "Afghanistan" "Afghanistan" "Afghanistan" ...
##  $ DateRep                                       : Date, format: "2020-01-22" "2020-01-23" "2020-01-24" "2020-01-25" ...
##  $ Cases_cumsum                                  : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Cases                                         : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Deaths_cumsum                                 : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Deaths                                        : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Recovered_cumsum                              : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Recovered                                     : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Active_cases_cumsum                           : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ New cases per 1 million population            : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ New deaths per 1 million population           : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ New recovered cases per 1 million population  : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Death rate (%)                                : num  NA NA NA NA NA NA NA NA NA NA ...
##  $ Total active cases per 1 million population   : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Total deaths per 1 million population         : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Total cases per 1 million population          : int  0 0 0 0 0 0 0 0 0 0 ...
##  $ Total recovered cases per 1 million population: int  0 0 0 0 0 0 0 0 0 0 ...
##  - attr(*, ".internal.selfref")=<externalptr>
{% endhighlight %}
 
Different interesting statistics also computed "per 1 million population" for better comparison.
 
Transform data for 'since first 100th case' countries trajectories with same lengths:

{% highlight r %}
n_cases <- 100 # you can vary
statistic <- 'Total active cases per 1 million population'
 
# Cases greater than threshold
data_covid_100_cases <- copy(data_covid_ts[,
                                           .SD[DateRep >= .SD[Cases_cumsum >= n_cases,
                                                              min(DateRep,
                                                                  na.rm = T)]],
                                            by = .(Country)])
 
setorder(data_covid_100_cases,
         Country,
         DateRep)
    
# Create 'days since' column
data_covid_100_cases[, (paste0("Days_since_first_",
                               n_cases, "_case")) := 1:.N,
                     by = .(Country)]
   
# top N countries selection for analysis
data_cases_order <- copy(data_covid_100_cases[,
                                      .SD[DateRep == max(DateRep),
                                          get(statistic)],
                                      by = .(Country)
                                      ])
    
setnames(data_cases_order, "V1", statistic)
 
setorderv(data_cases_order, statistic, -1)
    
# subset data based on selected parameter -  top 82 countries and analyzed columns
data_covid_cases_sub <- copy(data_covid_100_cases[.(c(data_cases_order[1:82][!is.na(Country), Country],
                                                      "Slovakia")),
                                                  on = .(Country),
                                                 .SD,
                                                 .SDcols = c("Country",
                                                             paste0("Days_since_first_", n_cases, "_case"),
                                                             statistic)
                                                 ])
 
# Make same length time series from countries data
data_covid_trajectories <- dcast(data_covid_cases_sub,
                                 get(paste0("Days_since_first_", n_cases, "_case")) ~ Country,
                                 value.var = statistic)
    
setnames(data_covid_trajectories,
         colnames(data_covid_trajectories)[1],
         paste0("Days_since_first_", n_cases, "_case"))
 
DT::datatable(data_covid_trajectories,
              class = "compact",
              extensions = 'Scroller',
              options = list(
                dom = 't',
                deferRender = TRUE,
                scrollY = 270,
                scroller = TRUE,
                scrollX = TRUE
              ))
{% endhighlight %}

![plot of chunk unnamed-chunk-3](/images/post_13/unnamed-chunk-3-1.png)
 
Prepare trajectories' data for clustering:

{% highlight r %}
q_sma <- 3
 
# save stats of dt
n_col <- ncol(data_covid_trajectories)
n_row <- nrow(data_covid_trajectories)
    
n_row_na <- rowSums(data_covid_trajectories[, lapply(.SD, is.na)])
n_col_na <- colSums(data_covid_trajectories[, lapply(.SD, is.na)])
    
# remove all NA rows and cols - for sure
if (length(which(n_row_na %in% n_col)) != 0) {
 
  data_covid_trajectories <- copy(data_covid_trajectories[-which(n_row_na == n_col)])
      
}
    
if (length(which(n_col_na %in% n_row)) != 0) {
      
  data_covid_trajectories <- copy(data_covid_trajectories[, -which(n_col_na %in% n_row), with = FALSE])
      
}
 
# use SMA for preprocessing time series from my TSrepr package
data_covid_trajectories[,
                        (colnames(data_covid_trajectories)[-1]) := lapply(.SD, function(i)
                        c(rep(NA, q_sma - 1),
                        ceiling(repr_sma(i, q_sma))
                          )),
                        .SDcols = colnames(data_covid_trajectories)[-1]]
 
DT::datatable(data_covid_trajectories,
              class = "compact",
              extensions = 'Scroller',
              options = list(
                dom = 't',
                deferRender = TRUE,
                scrollY = 270,
                scroller = TRUE,
                scrollX = TRUE
              ))
{% endhighlight %}

![plot of chunk unnamed-chunk-4](/images/post_13/unnamed-chunk-4-1.png)
 
 
Define clustering function with DTW distance with additional data preprocessing necessary for dtwclust package.

{% highlight r %}
cluster_trajectories <- function(data, k, normalize = FALSE) {
  
  # transpose data for clustering
  data_trajectories_trans <- t(data[, .SD,
                                    .SDcols = colnames(data)[-1]])
  
  data_trajectories_trans_list <- lapply(1:nrow(data_trajectories_trans), function(i)
    na.omit(data_trajectories_trans[i,]))
  names(data_trajectories_trans_list) <- colnames(data)[-1]
 
  n_list <- sapply(1:length(data_trajectories_trans_list), function(i)
    length(data_trajectories_trans_list[[i]]))
  names(n_list) <- names(data_trajectories_trans_list)
 
  if (length(which(n_list %in% 0:1)) != 0) {
    
    data_trajectories_trans_list <- data_trajectories_trans_list[-which(n_list %in% 0:1)]
    
  }
  
  list_names <- names(data_trajectories_trans_list)
 
  # normalization
  if (normalize) {
    
    data_trajectories_trans_list <- lapply(names(data_trajectories_trans_list),
                                                function(i)
                                                  norm_z(data_trajectories_trans_list[[i]])
                                                 )
    names(data_trajectories_trans_list) <- list_names
    
  }
 
  hc_res <- tsclust(data_trajectories_trans_list,
                    type = "hierarchical",
                    k = k,
                    distance = "dtw_basic", # "dtw", "dtw_basic", "dtw2", "sdtw"
                    centroid = dba, # dba, sdtw_cent
                    trace = FALSE,
                    seed = 54321,
                    control = hierarchical_control(method = "ward.D2"),
                    args = tsclust_args(dist = list(norm = "L2"))
                    )
  
  return(hc_res)
  
}
{% endhighlight %}
 
Execute clustering

{% highlight r %}
clust_res <- cluster_trajectories(data = data_covid_trajectories,
                                  k = 14,
                                  normalize = TRUE)
 
clust_res
{% endhighlight %}



{% highlight text %}
## hierarchical clustering with 14 clusters
## Using dtw_basic distance
## Using dba centroids
## Using method ward.D2 
## 
## Time required for analysis:
##    user  system elapsed 
##    0.97    0.11    1.65 
## 
## Cluster sizes with average intra-cluster distance:
## 
##    size   av_dist
## 1    21 0.5011793
## 2     6 0.5170467
## 3     2 0.9447618
## 4    10 0.4015355
## 5    11 0.5774944
## 6     5 0.6441501
## 7     7 0.4719638
## 8     2 0.5682452
## 9     5 0.7722092
## 10    2 0.8360796
## 11    4 0.4361994
## 12    5 0.5527005
## 13    2 0.5363868
## 14    1 0.0000000
{% endhighlight %}
 
Prepare data for plotting:

{% highlight r %}
# prepare time series
data_clust_id <- data.table(Cluster = clust_res@cluster,
                            Country = names(clust_res@cluster))
 
data_plot <- melt(data_covid_trajectories,
                  id.vars = colnames(data_covid_trajectories)[1],
                  variable.name = "Country",
                  variable.factor = FALSE,
                  value.name = statistic,
                  value.factor = FALSE
                  )
 
data_plot <- copy(data_plot[.(data_clust_id$Country), on = .(Country)])
 
data_plot[data_clust_id,
          on = .(Country),
          Cluster := i.Cluster]
 
DT::datatable(data_plot,
              class = "compact",
              extensions = 'Scroller',
              filter = "top",
              options = list(
                dom = 't',
                deferRender = TRUE,
                scrollY = 270,
                scroller = TRUE,
                scrollX = TRUE
              ))
{% endhighlight %}

![plot of chunk unnamed-chunk-7](/images/post_13/unnamed-chunk-7-1.png)
 
Plot of cluster members....

{% highlight r %}
theme_my <- theme(panel.border = element_rect(fill = NA,
                                                  colour = "grey10"),
                      panel.background = element_blank(),
                      panel.grid.minor = element_line(colour = "grey85"),
                      panel.grid.major = element_line(colour = "grey85"),
                      panel.grid.major.x = element_line(colour = "grey85"),
                      axis.text = element_text(size = 12, face = "bold"),
                      axis.title = element_text(size = 13, face = "bold"),
                      plot.title = element_text(size = 16, face = "bold"),
                      strip.text = element_text(size = 12, face = "bold"),
                      strip.background = element_rect(colour = "black"),
                      legend.text = element_text(size = 14),
                      legend.title = element_text(size = 15, face = "bold"),
                      legend.background = element_rect(fill = "white"),
                      legend.key = element_rect(fill = "white"),
                      legend.position="bottom")
 
ggplot(data_plot,
       aes(get(colnames(data_plot)[1]),
           get(statistic),
           group = Country)) +
      facet_wrap(~Cluster,
                 ncol = ceiling(data_plot[, sqrt(uniqueN(Cluster))]),
                 scales = "free_y") +
      geom_line(color = "grey10",
                alpha = 0.75,
                size = 0.8) +
      scale_y_continuous(trans = 'log10') +
      labs(x = colnames(data_plot)[1],
           y = statistic) +
      theme_my
{% endhighlight %}

![plot of chunk unnamed-chunk-8](/images/post_13/unnamed-chunk-8-1.png)
 
Check some clusters interactively with dygraphs:

{% highlight r %}
data_clust_focus <- dcast(data_plot[.(c(2,6)), on = .(Cluster)],
                          Days_since_first_100_case ~ Country,
                          value.var = statistic)
 
dygraph(data_clust_focus,
        main = "Clusters n. 2 and 6") %>%
      dyAxis("x", label = colnames(data_clust_focus)[1]) %>%
      dyAxis("y", label = statistic) %>%
      dyOptions(strokeWidth = 2,
                drawPoints = F,
                pointSize = 3,
                pointShape = "circle",
                logscale = TRUE,
                colors = RColorBrewer::brewer.pal(ncol(data_clust_focus)-1, "Set2")) %>%
      dyHighlight(highlightSeriesOpts = list(strokeWidth = 2.5,
                                             pointSize = 4)) %>%
      dyLegend(width = 150, show = "follow",
               hideOnMouseOut = TRUE, labelsSeparateLines = TRUE)
{% endhighlight %}

![plot of chunk unnamed-chunk-9](/images/post_13/unnamed-chunk-9-1.png)
 
Check some clusters interactively with dygraphs:

{% highlight r %}
data_clust_focus <- dcast(data_plot[.(7), on = .(Cluster)],
                          Days_since_first_100_case ~ Country,
                          value.var = statistic)
 
dygraph(data_clust_focus,
        main = "Cluster n.7") %>%
      dyAxis("x", label = colnames(data_clust_focus)[1]) %>%
      dyAxis("y", label = statistic) %>%
      dyOptions(strokeWidth = 2,
                drawPoints = F,
                pointSize = 3,
                pointShape = "circle",
                logscale = TRUE,
                colors = RColorBrewer::brewer.pal(ncol(data_clust_focus)-1, "Set2")) %>%
      dyHighlight(highlightSeriesOpts = list(strokeWidth = 2.5,
                                             pointSize = 4)) %>%
      dyLegend(width = 150, show = "follow",
               hideOnMouseOut = TRUE, labelsSeparateLines = TRUE)
{% endhighlight %}

![plot of chunk unnamed-chunk-10](/images/post_13/unnamed-chunk-10-1.png)
 
Check some clusters interactively with dygraphs:

{% highlight r %}
data_clust_focus <- dcast(data_plot[.(1), on = .(Cluster)],
                          Days_since_first_100_case ~ Country,
                          value.var = statistic)
 
dygraph(data_clust_focus,
        main = "Cluster n. 1") %>%
      dyAxis("x", label = colnames(data_clust_focus)[1]) %>%
      dyAxis("y", label = statistic) %>%
      dyOptions(strokeWidth = 2,
                drawPoints = F,
                pointSize = 3,
                pointShape = "circle",
                logscale = TRUE,
                colors = RColorBrewer::brewer.pal(ncol(data_clust_focus)-1, "Set2")) %>%
      dyHighlight(highlightSeriesOpts = list(strokeWidth = 2.5,
                                             pointSize = 4)) %>%
      dyLegend(width = 150, show = "follow",
               hideOnMouseOut = TRUE, labelsSeparateLines = TRUE)
{% endhighlight %}

![plot of chunk unnamed-chunk-11](/images/post_13/unnamed-chunk-11-1.png)
 
Dendrogram to see nicer connections between countries in a tree:

{% highlight r %}
dend <- as.dendrogram(clust_res)
 
dend <- dend %>%
  color_branches(k = 14) %>%
  color_labels(k = 14) %>%
  set("branches_lwd", 1) %>%
  set("labels_cex", 0.8)
 
ggd1 <- as.ggdend(dend)
 
ggplot(ggd1,
       horiz = T)
{% endhighlight %}

![plot of chunk unnamed-chunk-12](/images/post_13/unnamed-chunk-12-1.png)
 
MDS 2D plot - we can use simply DTW distances.

{% highlight r %}
mds_classical <- cmdscale(clust_res@distmat, eig = FALSE, k = 2)
 
data_plot <- data.table(mds_classical,
                        Country = row.names(mds_classical),
                        Cluster = clust_res@cluster)
 
ggplot(data_plot, aes(x = get("V1"),
                      y = get("V2"),
                      label = Country,
                      color = as.factor(Cluster))) +
      geom_label_repel(size = 4.2,
                       alpha = 0.95,
                       segment.alpha = 0.35,
                       label.r = 0.1,
                       box.padding = 0.25,
                       label.padding = 0.3,
                       label.size = 0.35,
                       max.iter = 2500) +
      scale_color_manual(values = colorspace::rainbow_hcl(14, c = 90, l = 50)) +
      labs(x = NULL, y = NULL, color = NULL) +
      guides(color = FALSE) +
      theme_my
{% endhighlight %}

![plot of chunk unnamed-chunk-13](/images/post_13/unnamed-chunk-13-1.png)
 
## Recap
 
DTW hierarchical clustering of active cases.
CovidR contest, shinyapps.io, GitHub source code petolau + CoronaDash