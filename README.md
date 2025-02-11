---
title: "LiDar Catalog Processing in R"
author: "Noah Fried"
date: "2024-10-15"
output: github_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE, eval = FALSE)
```

Please view LidaR github for more indepth descriptions - [The lidR package](https://r-lidar.github.io/lidRbook/)

# Processing LiDAR Data in R

### A beginners Guide to processing LiDAR data in R

# Load the libraries

```{r load-libraries-setup, message=FALSE, warning=FALSE}
library(lidR)
library(progress)
library(terra)
library(future)

```

## Read in your Catalog

```{r  message=FALSE, warning=FALSE}
ctg <- readLAScatalog(r"(<your_directory/>_catalogFolder)", progress = TRUE)

```

You want this path file to lead to where all your .LAS files are stored. This **readLAScatalog()** function groups all of your LAS files into one big catalog allowing you use the entire catalog for future processing. It is important to name your catalog in away you will remember and that is intuitive.

## Checking your data

```{r}
summary(ctg)
```

-   The command **summary()** displays the details of the LAS catalog object. This includes metadata about the catalog such as the number of LAS files it contains, the total number of points, the geographic extent, and other relevant information.

-   By inspecting the catalog, you can quickly assess whether all expected files are present and confirm key attributes like point count and spatial extent.

```{r}
plot(ctg)
```

-The **plot()** function visualizes the LiDAR data stored in the *ctg* object. This function provides a graphical representation of the classified points, allowing you to see the distribution and classification of points in your area of interest.

-It’s also useful for exploring the spatial relationships between different features in the landscape, which can inform further analysis or decision-making

## Validating the Catalog

To ensure the integrity of my ***LAS catalog (ctg)*** and any derived datasets (classified and height normalized catalogs), I use the las_check() function. This function systematically verifies the consistency of key attributes, including file version, scale, offsets, point type, VLRs, and CRS. Running las_check() helps identify potential errors or inconsistencies before further processing, ensuring reliable results in LiDAR analysis.

```{r}
las_check(ctg)
```

## Index .lax files for more efficent processing

In order to make the processing of a catalog more efficient **.LAS** files must be indexed into **.LAX** files. It generates a spatial index that helps the lidR package quickly locate and access the LAS files and their associated data within the catalog based on spatial queries. This indexing allows for faster processing of point clouds and makes it easier to manage large data sets. Doing for each derived dataset will save you time and headaches while processing your Lidar.

```{r message=FALSE, warning=FALSE}
#lidR:::catalog_laxindex(ctg) 
```

## Parallel Processing

Parallel processing is a computational technique that involves the simultaneous execution of multiple processes or tasks.

```{r, eval = FALSE}
plan(multisession, workers = 2)
set_lidr_threads(2L)
```

-   The command **plan(multisession, workers = 2)** configures a parallel processing plan in R, enabling the execution of tasks using two worker processes. The multisession option allows each worker to operate in its own R session facilitating the execution of multiple tasks concurrently.

-   The function **set_lidr_threads(2L)** is specific to the lidR package, designed for processing LiDAR data. This function sets the number of threads utilized for data processing, enabling efficient handling of large data sets by leveraging parallel computing capabilities.

This should be ran immediately before your processing task.

## Chunking

The primary task is to break down data into smaller subsets that can be processed independently or in parallel. This division is particularly beneficial in managing memory usage, as processing data in chunks allows for effective utilization of system resources. Instead of loading an entire dataset into memory, which can lead to memory overload, chunking enables the system to handle smaller segments at a time. I have been using **chunk_size \<- 100** has given me the best results despite the warning. I think you need to play around with it and each catalog will require different chunk sizes.

Since the edges of chunks have fewer points to base calculations on, you create a **chunk_buffer** to incorporate data from neighboring chunks. This ensures that boundary points are processed with the same context as interior points, reducing edge effects and improving the accuracy of classification, interpolation, and other LiDAR processing tasks.

```{r message=FALSE, warning=FALSE}
opt_chunk_size(ctg) <- 250
opt_chunk_buffer<- 10
plot(ctg, chunk = TRUE)
```

# Processing a Catalog

Many of the functions for processing a catalog are the same as processing LAS files which makes it very convenient. Once you address indexing, parallel processing, and chunking you can start efficiently processing with the first step being ground classification. Following classification you would index again run parallel processing again and choose your method of height normalization. Finally once you ground classi

## Ground Classification

After running you're done executing parallel processing and setting chunk size and buffer next step is to run your ground classification. I use cloth simulation function which is particularly good at classifying complex/ steep terrain and dense forest. The basic idea is that it's as if a cloth is draped over the point cloud allowing the algorithm to pick up intricate features. In **mycsf** is a highly configurable function more options are available than what I put above you this is just how I use it.

```{r}
mycsf <- csf(sloop_smooth = TRUE, cloth_resolution = 1 )
ctg_classified <- classify_ground(ctg, mycsf)

```

## Digital Terrain Model

There are several ways to go about creating Digital Terrain Models and I am just going to talk about my own method. Utimatley you will use this DTM to height normalize.

```{r}
opt_output_files(ctg_classified)<- "<your_directory/>_classifiedcatalogFolder/{XLEFT}_{YBOTTOM}_dtm"
plan(multisession, workers = 2)
set_lidr_threads(2L)
dtm_ctg <- rasterize_terrain(ctg_classified, 1, tin(), pkg = "terra")
plot(dtm_ctg, col = gray(1:100/100), main = "1m DTM")
```

## Viewing just the ground

Viewing the ground in a catalog operates differently than it does for individual LAS files. This is a proper way to just retrieve the ground to help you visualize any missing points.

```{r}
opt_filter(ctg_classified) <- "-keep_class 2"
f = function (x){min = min(x)} # minimum function
ground_min_1m <- pixel_metrics(ctg_classified, func = ~f(Z), res = 1) #grab the minimum ground point, 
plot(ground_min_1m, main = "1m DTM with Data Gaps") 
```

## Heigh Normalization

### DTM

### TIN

As we know from the LAS normalization there are multiple ways to Height Normalize. I like using **Triangular Irregular Network** to height normalize because it does not require a raster to do

```{r}
opt_output_files(ctg_classified)<- "<your_directory/>_classifiedcatalogFolder/{XLEFT}_{YBOTTOM}_norm"
plan(multisession, workers = 2)
set_lidr_threads(2L)
ctg_normalized <- normalize_height(ctg_classified, tin())
```

## Viewing your data

In order to view your models you want to plot

```{r}
roi <- clip_circle(ctg_classified, x = , y = , radius = )
```

```{r}
plot(roi, bg = "white", size = 4)
```

```{r}
plot(roi, size = 3, bg = "white", color = "Classification")
```
