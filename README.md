TEMPTED Vignette
================

# tempted

<!-- badges: start -->
<!-- badges: end -->

The goal of tempted is to perform dimensionality reduction for
multivariate longitudinal data, with a special attention to longitudinal
mirobiome studies.

### Installation

You can install the development version of tempted from
[GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("pixushi/tfsvd")
```

To use our method, please cite the following paper:

[Rungang Han, Pixu Shi, Anru R. Zhang, Guaranteed Functional Tensor
Singular Value Decomposition](https://arxiv.org/abs/2108.04201)

### Load required packages

``` r
library(gridExtra)
library(ggplot2)
library(tempted)
```

### Read the example data

The example dataset is originally from [Bokulich, Nicholas A., et
al. (2016)](https://pubmed.ncbi.nlm.nih.gov/27306664/). After some data
preprocessing, the cleaned data we are using here contain one table for
OTU count, one table for meta data, and one table for the taxonomic
information of OTUs. The taxonomic table is not used for dimension
reduction, but will be helpful for interpretation of the results.

``` r
# match rows of different data frames
# check number of samples
table(rownames(count_table)==rownames(meta_table))
#> 
#> TRUE 
#>  852
metauni <- unique(meta_table[,c('studyid', 'delivery')])
```

### Prepare for TEMPTED

The function `tempted()` takes input in the format of a list. Each list
member corresponds to one subject and is a matrix with the first row for
time points, and the remaining $p$ rows for the observed values of $p$
features. With the list format, different subjects can have different
time points.

The function `format_tempted()` transforms the sample-by-bacteria count
table and the corresponding time points and subject IDs into the list
format that is accepted by `tempted()`. In the default option of
`format_tempted()`, the bacteria whose non-zero value are below 5% are
filtered out by option `threshold=0.95`, and a pseudo_count is added to
all the counts by option `pseudo_count` before being normalized into
proportions and going through log transformation. Option `feature_names`
allows the user to specify a subset of the columns of the count table to
be used for TEMPTED. When a subject has multiple samples from the same
time point, `format_tempted()` will only keep the first sample in the
data input.

Through option `transform`, the user can choose among the following
transformations: `none` for no transformation, `comp` for normalizing
into proportions, `ast` for arcsin square root transformation of the
proportions, `log_comp` for log transformation of the proportions, `clr`
for central log ratio transformation of the proportions, `logit` for
logit transformation of the proportions. The latter three requires the
`pseudo_count` option to be non-zero. The user can choose their desired
transformation by transforming their data before using
`format_tempted(..., transform="none")`.

In this example, the number of subjects is 42, the number of features is
795, and the number of time points in the first subject is 29.

``` r
# format the data frames into a list that can be used by TEMPTED
datlist <- format_tempted(count_table, meta_table$day_of_life, meta_table$studyid, threshold=0.95, pseudo_count=0.5, transform="clr")
length(datlist)
#> [1] 42
print(dim(datlist[[1]]))
#> [1] 796  29
```

### Run TEMPTED

The function `svd_centralize()` uses matrix SVD to fit a constant
trajectory (straight flat line) for all subject-feature pairs, and
remove it from the data. This step is optional. If it is not used, we
recommend the user to add 1 to the rank parameter `r` in `tempted()` and
in general the first component estimated by `tempted()` will reflect
this constant trajectory. In this example, we used `r=3`. Option
`smooth` allows the user to choose the smoothness of the estimated
temporal loading.

``` r
# centralize data using matrix SVD after log 
svd_tempted <- svd_centralize(datlist)
res_tempted <- tempted(svd_tempted$datlist, r = 3, resolution = 101, smooth=1e-5)
#> [1] "Calculate the 1th Component"
#> [1] "Convergence reached at dif=3.53146145162134e-05, iter=4"
#> [1] "Calculate the 2th Component"
#> [1] "Convergence reached at dif=4.3778326665428e-05, iter=7"
#> [1] "Calculate the 3th Component"
#> [1] "Convergence reached at dif=7.11030910416548e-05, iter=6"
# alternatively, you can replace the two lines above by the two lines below.
# the 2nd and 3rd component in the result below will be 
# similar to the 1st and 2nd component in the result above.
#svd_tempted <- NULL
#res_tempted <- tempted(datlist, r = 4, resolution = 101)
```

### Plot the subject loadings

Subject loadings are stored in variable `A.hat` of the `tempted()`
output.

``` r
A.data <- metauni
rownames(A.data) <- A.data$studyid
table(rownames(A.data)==rownames(res_tempted$A.hat))
#> 
#> TRUE 
#>   42
A.data <- cbind(res_tempted$A.hat, A.data)
npc <- ncol(res_tempted$A.hat)

# color by delivery mode
p_deliv_list <- vector("list", npc*(npc-1)/2)
ij <- 1
for (i in 1:(npc-1)){
  for (j in (i+1):npc){
    p_deliv_list[[ij]] <- local({
      ij <- ij
      i <- i
      j <- j
      ptmp <- ggplot(data=A.data, aes(x=A.data[,i], y=A.data[,j], color=delivery)) + 
      geom_point()  + 
        theme_bw() + 
      labs(x=paste('Component',i), y=paste('Component',j), title='subject loading') 
      ptmp
    })
    ij <- ij+1
  }
}

for (i in 1:length(p_deliv_list)){
  print(p_deliv_list[[i]])
}
```

<img src="man/figures/README-plot_sub_loading-1.png" width="100%" /><img src="man/figures/README-plot_sub_loading-2.png" width="100%" /><img src="man/figures/README-plot_sub_loading-3.png" width="100%" />

### Plot temporal loadings

Temporal loadings are stored in variable `Phi.hat` of the `tempted()`
output. We provide an R function `plot_time_loading()` to plot these
curves. Option `r` lets you decide how many components to plot.

``` r
ptime <- plot_time_loading(res_tempted, r=2) + 
  geom_line(size=1.5) + theme_bw() +
  labs(title='temporal loadings', x='days')
print(ptime)
```

<img src="man/figures/README-plot_time_loading-1.png" width="100%" />

### Plot the feature loadings

Feature loadings are stored in variable `B.hat` of the `tempted()`
output.

``` r
pfeature <- ggplot(as.data.frame(res_tempted$B.hat), aes(x=`Component 1`, y=`Component 2`)) + 
  geom_point() + 
  theme_bw() + 
  labs(x='Component 1', y='Component 2', title='feature loading')
print(pfeature)
```

<img src="man/figures/README-plot_feature_loading-1.png" width="100%" />

### Plot trajectory of top/bottom feature abundance ratio

The feature loadings can be used to rank features. The abundance ratio
of top ranking features over bottom ranking features corresponding to
each component can be a biologically meaningful marker. We provide an R
function `ratio_feature()` to calculate such the abundance ratio. The
abundance ratio is stored in the output variable `metafeature.ratio` in
the output. An TRUE/FALSE vector indicating whether the feature is in
the top ranking or bottom ranking is stored in variable `toppct` and
`bottompct`, respectively. Below are trajectories of the aggregated
features using observed data. Users can choose the percentage for the
cutoff of top/bottom ranking features through option `pct` (by default
`pct=0.05`). By default `absolute=TRUE` means the features are chosen if
they rank in the top `pct` percentile in the absolute value of the
feature loadings, and abundance ratio is taken between the features with
positive loadings over negative loadings. When `absolute=FALSE`, the
features are chose if they rank in the top `pct` percentile and has
positive loading or rank in the bottom `pct` percentile and has negative
loading. We also provide an option `contrast` allowing users to rank
features using linear combinations of the feature loadings from
different components.

``` r
datlist_raw <- format_tempted(count_table, meta_table$day_of_life, meta_table$studyid, 
                            threshold=0.95, transform='none')
contrast <- matrix(c(1,1,0,1,-1,0),3,2)
contrast
#>      [,1] [,2]
#> [1,]    1    1
#> [2,]    1   -1
#> [3,]    0    0

ratio_feat <- ratio_feature(res_tempted, datlist_raw,
                        contrast=contrast, pct=0.1)
tab_feat_obs <- ratio_feat$metafeature.ratio
head(tab_feat_obs)
#>        value subID timepoint          PC
#> 1 -3.0703781     1         0 Component 1
#> 2  3.0029270     1         0 Component 2
#> 3  4.8421196     1         0 Component 3
#> 4 -0.9521522     1         0  Contrast 1
#> 5 -2.9586049     1         0  Contrast 2
#> 6  1.2457846     1        36 Component 1
colnames(tab_feat_obs)[2] <- 'studyid'
tab_feat_obs <- merge(tab_feat_obs, metauni)
colnames(tab_feat_obs)
#> [1] "studyid"   "value"     "timepoint" "PC"        "delivery"

p_feat_obs <- ggplot(data=tab_feat_obs, 
                      aes(x=timepoint, y=value, group=studyid, color=delivery)) +
  geom_line() + facet_wrap(.~PC, scales="free", nrow=1) +
  labs(title="observed subject trajectory", x="days")

## observed, by mean and sd
reshape_feat_obs <- reshape(tab_feat_obs, 
                            idvar=c("studyid","timepoint") , 
                            v.names=c("value"), timevar="PC",
                            direction="wide")
colnames(reshape_feat_obs) <- 
  sub(".*value[.]", "",  colnames(reshape_feat_obs))
colnames(reshape_feat_obs)
#> [1] "studyid"     "timepoint"   "delivery"    "Component 1" "Component 2"
#> [6] "Component 3" "Contrast 1"  "Contrast 2"

p_feat_obs_summary <- plot_feature_summary(reshape_feat_obs[,-(1:4)], 
                     reshape_feat_obs$timepoint, 
                     as.factor(reshape_feat_obs$delivery), bws=30, nrow=1) +
  labs(title="observed subject trajectory summary", x="days")
              
grid.arrange(p_feat_obs, p_feat_obs_summary, nrow=2)
```

<img src="man/figures/README-plot_ratio_traj-1.png" width="100%" />

### Plot trajectory of aggregated features

The feature loadings can be used as weights to aggregate features. The
aggregation can be done using the low-rank denoised data tensor, or the
original observed data tensor. We provide an R function
`aggregate_feature()` to perform the aggregation. The aggregated
features using low-rank denoised tensor is stored in variable
`metafeature.aggregate.est` and the aggregated features using observed
data is stored in variable `metafeature.aggregate`. Below are
trajectories of the aggregated features using observed data. Only the
features with absolute loading in the top pct percentile (by default set
to 100%, i.e. all features, through option `pct=1`) are used for the
aggregation. We also provide an option `contrast` allowing users to
aggregate features using linear combinations of the feature loadings
from different components.

``` r
## observed, by individual subject
contrast <- matrix(c(1,0,1,1,0,-1),3,2)
contrast
agg_feat <- aggregate_feature(res_tempted, svd_tempted, datlist, 
                              contrast=contrast, pct=1)
tab_feat_obs <- agg_feat$metafeature.aggregate
head(tab_feat_obs)
colnames(tab_feat_obs)[2] <- 'studyid'
tab_feat_obs <- merge(tab_feat_obs, metauni)
colnames(tab_feat_obs)

p_feat_obs <- ggplot(data=tab_feat_obs, 
                      aes(x=timepoint, y=value, group=studyid, color=delivery)) +
  geom_line() + facet_wrap(.~PC, scales="free", nrow=1) +
  labs(title="observed subject trajectory", x="days")

## observed, by mean and sd
reshape_feat_obs <- reshape(tab_feat_obs, 
                            idvar=c("studyid","timepoint") , 
                            v.names=c("value"), timevar="PC",
                            direction="wide")
colnames(reshape_feat_obs) <- 
  sub(".*value[.]", "",  colnames(reshape_feat_obs))
colnames(reshape_feat_obs)

p_feat_obs_summary <- plot_feature_summary(reshape_feat_obs[,-(1:5)], 
                     reshape_feat_obs$timepoint, 
                     as.factor(reshape_feat_obs$delivery), bws=30, nrow=1) +
  labs(title="observed subject trajectory summary", x="days")
              

grid.arrange(p_feat_obs, p_feat_obs_summary, nrow=2)
```

<img src="man/figures/README-plot_aggfeat_traj-1.png" width="100%" />

### Scatterplot of samples using aggregated features

``` r
p_aggfeat_scatter <- ggplot(data=reshape_feat_obs, aes(x=`Component 1`, y=`Component 2`, 
                             color=timepoint, shape=delivery)) +
  geom_point() + scale_color_gradient(low = "#2b83ba", high = "#d7191c") + 
  labs(x='Component 1', y='Component 2', color='Day')
p_aggfeat_scatter
```

<img src="man/figures/README-plot_aggfeat_scatter-1.png" width="100%" />

``` r

# subsetting timepoint between 3 months and 1 year
p_aggfeat_scatter2 <- ggplot(data=dplyr::filter(reshape_feat_obs, timepoint<365 & timepoint>30),
                             aes(x=`Component 1`, y=`Component 2`, 
                             color=delivery)) +
  geom_point() + 
  labs(x='Component 1', y='Component 2', color='Delivery Mode')
p_aggfeat_scatter2
```

<img src="man/figures/README-plot_aggfeat_scatter-2.png" width="100%" />

### Plot trajectory of top features

The feature loadings can also be used to investigate the driving
features behind each component. Here we focus on the 2nd component and
provide the trajectories of the top features using the estimated and
observed data tensor, respectively.

``` r
# get index of top features
B.data <- as.data.frame(res_tempted$B.hat)
feature_sel <- rownames(B.data)[order(-abs(B.data[,2]))[1:4]]
feature_sel
# individual lines

tabular_feature_obs <- function(datlist, feature_sel){
  tab_obs <- NULL
  for (i in 1:length(feature_sel)){
    value <- unlist(sapply(datlist, function(x){x[feature_sel[i],]}, simplify=F))
    time_point <- unlist(sapply(datlist, function(x){x['time_point',]}, simplify=F))
    nobs <- sapply(datlist, function(x){ncol(x)}, simplify=F)
    subID <- unlist(mapply(function(i){rep(names(datlist)[i], nobs[i])},
                           1:length(nobs), SIMPLIFY=F))
    tmp <- data.frame(subID=subID, time=time_point, feature=feature_sel[i], value=value)
    rownames(tmp) <- NULL
    tab_obs <- rbind(tab_obs, tmp)
  }
  tab_obs$type <- 'observed'
  return(tab_obs)
}
tab_obs <- tabular_feature_obs(datlist, feature_sel)
head(tab_obs)
colnames(tab_obs)[1] <- "studyid"
tab_feature <- merge(tab_obs, metauni)
p_ftsel <- ggplot(data=tab_feature, aes(x=time, y=value, group=studyid, color=delivery)) + 
  geom_line() + facet_grid(~feature) + 
  ggtitle('features with top scores in 2nd component')

# summarized lines
reshape_tab_feature <- reshape(tab_feature, 
                               idvar=c("studyid","time") , 
                               v.names=c("value"), timevar="feature",
                               direction="wide")
feature_mat <- reshape_tab_feature[,grep('value',colnames(reshape_tab_feature))]
colnames(feature_mat) <- sub(".*value[.]", "",  colnames(feature_mat))
time_vec <- reshape_tab_feature$time
group_vec <- reshape_tab_feature$delivery
p_ftsel_sum <- plot_feature_summary(feature_mat, time_vec, group_vec, 
                                    bws=30, nrow=1)

print(grid.arrange(p_ftsel,p_ftsel_sum))
```

<img src="man/figures/README-plot_top_feature-1.png" width="100%" />
