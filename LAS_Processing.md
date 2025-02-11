---
title: "Classification"
author: "Noah Fried"
date: "2024-10-18"
output: github_document
---

### LiDAR online resources
[For more in depth description visit the lidar online resource](https://r-lidar.github.io/lidRbook/io.html) 
This is an online textbook which contains a more in depth description of the content I will be discussing in this page

- This vignette is a file that provides in-depth information about the package, including how to use it, its functions, and practical examples. You can run this line in R and it will take you to the link I have provided below 
```{r message=FALSE, warning=FALSE, eval = FALSE}
browseVignettes(package = "lidR")
```

[Click to view Vingette](http://127.0.0.1:13483/session/Rvig.50c03f6417f2.html) 

# Install Required Packages and Libraries 
Before proceeding, ensure the necessary libraries are installed. The following packages are needed to process individual LAS files (more libraries will be introduced when working with catalogs): 
```{r, eval = FALSE}
install.packages("lidR")
install.packages("raster")
library(lidR)
library(raster)
```


# Exploring the Data
The first step in processing LiDAR data is to load the LAS file  using the **readLAS()** function. This allows you to start exploring your data.

```{r, eval = FALSE}
las = readLAS(r"(C:\Users\nf225627\Documents\Data\LAZ\LAZ_OKAMTW_557588_1587808_20180926_rp.laz)", , filter = "-drop_withheld")
```

### Display simple meta-data
```{r, eval = FALSE}
las
```
### Summarize the entire LAS file
```{r, eval = FALSE}
summary(las)
```
This summary will provide key information at a glance, such as:

- Extent
- Point Density
- Projection
- Total Number of Points
- Area Covered
- Memory Size

### Specify Projection or crs
```{r, eval = FALSE}
projection(las)
crs(las)
```

### Validate you LAS 
point count, classification, and CRS. It can catch issues that might cause errors in later steps.
```{r, eval = FALSE}
las_check(las)
```

### Classification count 
To see how many points are assigned to each classification, you can count the occurrences in the LAS file:
```{r, eval = FALSE}
classification_counts <- table(las@data$Classification) # Take a look at the LAS class
classification_counts
```

This step helps identify how your LAS file has been classified and whether any unwanted or missing values are present.

### ASPRS Class Descriptions

- **0**: Never Classified  
- **1**: Unclassified  
- **2**: Ground  
- **3**: Low Vegetation  
- **4**: Medium Vegetation  
- **5**: High Vegetation  
- **6**: Building  
- **7**: Low Point (Noise)  
- **8**: Reserved  
- **9**: Water  
- **10**: Rail  
- **11**: Road Surface  
- **12**: Reserved  
- **13**: Wire Guard (Shield)  
- **14**: Wire Conductor (Phase)  
- **15**: Transmission Tower  
- **16**: Wire Structure Connector (e.g., Insulator)  
- **17**: Bridge Deck  
- **18**: High Noise

# Plotting Your Data 
Prior to classifying your data it is helpful to get an initial look at your data. Looking at it in its entirtey is difficult because of how large the Las file can be there are several ways to deal with data this size 

### clip Circle and rectangle 
Visualizing the LAS data can be challenging due to file size. You can clip sections to focus on specific areas.

- Circle: 
Clip a circular area around a center point.
```{r, eval = FALSE}
las_circle <- clip_circle(las, xcenter = , ycenter = , radius = 30)
```

- Rectangle:
Clip a rectangular area.
```{r, eval = FALSE}
las_clp <- clip_rectangle(las, xmin, ymin, xmax, ymax)
```

- Transect 

- Check the size of the clipped files before proceeding:
```{r, eval = FALSE}
pryr::object_size(las)
pryr::object_size(las_circle)
pryr::object_size(las_clp)
```

# Cloth Simulation Function 
The CSF is highly effective for complex terrains. It simulates a cloth draping over the point cloud, distinguishing ground points from non-ground points.

```{r, eval = FALSE}
las_csf <- classify_ground(las, csf(sloop_smooth = TRUE))
writeLAS(las_csf, "C:/Users/nf225627/Documents/Data/LAZ/csf_classification.las")
```

This method works well in areas with varying slopes and terrain irregularities. Adjust CSF parameters as needed.


# Multi Scale Curvature Classification 
The MCC method is better suited for flat or gently sloping areas. It calculates terrain curvature and classifies points based on this.

```{r, eval = FALSE}
las <- classify_ground(las, mcc(1.5,0.3))
```

**1.5 (Window Size)**:
This is the spatial scale or window size at which the curvature is calculated. A larger value means that the algorithm looks at the terrain over a larger area to estimate the curvature. This helps in identifying broad, smoother terrain features. A smaller window size focuses on more localized features, capturing finer details but possibly misclassifying small objects (e.g., vegetation or buildings) as ground.

- Larger value (e.g., 2.0): Useful for gently sloping or flat terrain to avoid classifying small objects as ground.
- Smaller value (e.g., 1.0): Captures finer detail, better for complex or uneven terrains, but might misclassify non-ground features.


**0.3 (Curvature Threshold)**:
This is the curvature threshold used to distinguish ground points from non-ground points. The algorithm classifies a point as ground if its curvature is less than or equal to this threshold. A lower threshold means stricter classification, where only very flat or smooth points are considered ground. A higher threshold allows points with more curvature (e.g., mild slopes) to be classified as ground.

- Lower value (e.g., 0.2): Stricter classification, better for flat areas but might miss some legitimate ground points in more curved or sloped terrain.
- Higher value (e.g., 0.5): Allows more variation in curvature, suitable for areas with moderate slopes or terrain complexity, but could include more non-ground points.

### Plotting Your new Classified product 
You can visualize your classified LAS data in several ways: 

```{r, eval = FALSE}
plot(las_csf, size = 3, bg = "white", color = "Classification") 
```

To filter only ground points:
```{r, eval = FALSE}
gnd <- filter_ground(las_csf)
plot(gnd, size = 3, bg = "white", color = "Classification")
```
This helps visually validate the ground classification and height normalization. 




