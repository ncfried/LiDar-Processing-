---
title: "Standard Metrics"
author: "Noah Fried"
date: "2025-02-18"
output:
  html_document:
    df_print: paged
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = FALSE)
```

# Multiband Stacked Raster Workflow Post Height Normalization 

# Libraries
library(lidR)
library(terra)
library(future)

rm(list = ls())
gc()

# Define output paths for first return metrics
folder<- readline(prompt = "Enter las directory path: ")
out_dir<- readline(prompt = "Enter output directory path: ")
out_suffix <- readline(prompt = "Enter suffix: ")
user_cell_size <- 1 #in meters

# Load Normalized Catalog 
ctg = suppressWarnings(readLAScatalog(folder, progress = TRUE, filter="-drop_withheld -drop_z_below -1"))

# Filter
opt_select(ctg) <- "xyzciReturnNumber"

## make output file name *remove outsuffix if not needed
opt_output_files(ctg)<-paste0(out_dir, "\\", out_suffix, "_{ID}")

plan(multisession, workers = 2)
set_lidr_threads(2L)

metrics_custom <- function(z, Classification, RetNum, threshold = 0) { 
  zq <- stats::quantile(z, probs = seq(0.05, 0.95, 0.05), na.rm = TRUE) # Quantiles of all elevations within the cell
  profile <- LAD(z, dz = 1, k = 0.5, z0 = 2) # Compute LAD profile 
 
   # Ground classified returns
  grndpts <- z[Classification %in% c(2, 7, 9, 11)]  # Ground, low points, water, road surface
  ngrndpts <- length(grndpts)
  
  # Subset for first returns
  z_first <- z[RetNum == 1]
  
  # metrics list
  list(
    # Metrics for all returns
    Num_Returns = length(z),                                  # Total number of returns
    Num_GrndRet = sum(Classification == 2, na.rm = TRUE),    # Number of ground returns
    SDRH = stats::sd(z, na.rm = TRUE),                       # Standard deviation
    RH95 = zq[19],                                           # 95th percentile
    RH90 = zq[18],                                           # 90th percentile
    IQR_50 = zq[10],                                         # Median (50th percentile)
    IQR_25 = zq[5],                                          # 25th percentile
    RH10 = zq[2],                                            # 10th percentile
    RH05 = zq[1],                                            # 5th percentile
    IQR = zq[15] - zq[5],                                    # Interquartile range (75th - 25th)
    MH = mean(z, na.rm = TRUE),                              # Mean of all normalized relative heights
    max_all = max(z, na.rm = TRUE),                          # Max height (all returns)
    skew_gt_2m = moments::skewness(z[z > 2], na.rm = TRUE),  # Skewness
    kurt_gt2m = moments::kurtosis(z[z > 2], na.rm = TRUE),   # Kurtosis
    RD = sum(z > threshold, na.rm = TRUE) / length(z),       # Relative density
    CC_gt2m = ((sum(z_first > 2, na.rm = TRUE) / length(z_first)) * 100),
    Num_1stRet = length(z_first),                             # Number of first returns
    LADall = mean(profile[, 2], na.rm = TRUE)                # Mean LAD (all returns)
  )
}

# Compute all  metrics
plan(multisession, workers = 4)
result <- lidR::pixel_metrics(
  ctg,
  ~metrics_custom(Z, Classification, ReturnNumber),  # Use default threshold = 0 thresh hold decides height 
  res = user_cell_size
)




