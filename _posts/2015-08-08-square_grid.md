---
layout: post
title: Creating square grid dummy variables for regression analysis
bibliography: bibliography.bib
tags: [R, spatial, ggplot2, grid, over]
categories: [R, spatial]
date: "08 August, 2015"
---
<!--{% include JB/setup %} -->

---------------

In the course of my work on the estimation of the impacts of brownfields cleanup on housing prices in NY, I used 0.25- by 0.25-mile square grid fixed effects, which have a tighter control on unobservable neighbor characteristics than census tract fixed effects. The use of square grid fixed effects originate from Linn (2013). In this post, I will show you how I created the square grid dummy variables. Specifically, we'll learn to:

1. Create a raster layer consisting of square grids that covers five counties in NY: Bronx, Queens, Kings, New York, and Richmond.
	* identify how many miles one latitude and longitude degrees are, respectively
	* identify the bounding box 
	* identify the number of rows and columns of the final layer
	* create a SpatialGridDataFrame from scratch based on the information from the previous steps

2. Overlay point data (property locations) on the grids to identify which property belongs to which grid.

Let's first load the packages we are going to use in this session.


{% highlight r %}
library(knitr)
library(rgdal)
library(data.table)
library(dplyr)
library(ggplot2)
library(raster)
{% endhighlight %}



<br />

Now, we need to know the geographic extent of the three states in order to see how many miles one latitude and longitude degrees are, respectively. One latitude degree is similar in distance from the south to north end of the globe and is equivalent to 69 miles more or less. On the other hand, one latitude degree means a very different distance depending on where you are. One longitude degree is the longest at the equator and is about 69 miles. However, as you go farther away from the equator, one longitude degree means a smaller distance. So, if we would like to create square grids, we need to know the conversion rate at the geographic extent of our interest. You can find this information [here](http://www.csgnetwork.com/degreelenllavcalc.html). At latitude of 41, 1 latitude degree = 69.01 miles and 1 longitude degree = 52.28 miles. As we would like to create 0.25- by 0.25-mile square grids, the step sizes (distance between the centroids of adjacent grids) are the following:


{% highlight r %}
lat_step_size <-	1/69.01/4
lon_step_size <-	1/52.28/4
{% endhighlight %}

<br />

Now, let's import a NY county shape file (can be downloaded from [here](http://www.arcgis.com/home/item.html?id=7f4850fb7d18496ca6925f209d2d1275)) using **readOGR( )** that comes with the **rgdal** package. 


{% highlight r %}
ny_state <- readOGR(dsn = ".", "NY_counties_clip")
{% endhighlight %}



{% highlight text %}
## OGR data source with driver: ESRI Shapefile 
## Source: ".", layer: "NY_counties_clip"
## with 62 features
## It has 20 fields
{% endhighlight %}

<br />

We then generate a subset of the SpatialPolygonDataFrame to contain only Bronx, Queens, Kings, New York, and Richmond counties. 


{% highlight r %}
counties <- ny_state[ny_state@data$NAME %in% c('Bronx','Queens','Kings','New York','Richmond'),]
{% endhighlight %}

<br />

To get the spatial extent of these counties, we can extract this information from the **bbox** slot of **counties**.


{% highlight r %}
counties@bbox
{% endhighlight %}



{% highlight text %}
##         min       max
## x -74.23694 -73.70027
## y  40.50600  40.91595
{% endhighlight %}

<br />

Using this information, we define the geographic coordinate of the lower left corner of the gridded layer and its number of rows and columns. We then feed these information into the **GridTopology( )** function to create a SpatialGridDataFrame consisting of 0.25- by 0.25- mile grids.


{% highlight r %}
#--- latitude ---#
left_lon <- counties@bbox[1,1]-0.01
num_col <- ceiling((counties@bbox[1,2]-left_lon)/lat_step_size)

#--- longitude ---#
down_lat <- counties@bbox[2,1]-0.01
num_row <- ceiling((counties@bbox[2,2]-down_lat)/lon_step_size)

#--- create SpatialGridDataFrame ---#
gt <- GridTopology(c(left_lon,down_lat),c(lon_step_size,lat_step_size),c(num_row,num_col))
spg <- SpatialGrid(gt)
ny_grids <- SpatialGridDataFrame(spg, data.table(id = 1:(num_col*num_row)))
{% endhighlight %}

<br />

Now, let's import fake property location data that I created. We then use the **coordinates( )** function to transform the data into a SpatialPointsDataFrame.


{% highlight r %}
prop <- fread('fake_poperty.csv')
head(prop)
{% endhighlight %}



{% highlight text %}
##         lat       lon prop_id
## 1: 40.80627 -73.92422       1
## 2: 40.80561 -73.92262       2
## 3: 40.80981 -73.92068       3
## 4: 40.81193 -73.91994       4
## 5: 40.81169 -73.91904       5
## 6: 40.81255 -73.91737       6
{% endhighlight %}



{% highlight r %}
coordinates(prop) <- c('lon','lat')
prop
{% endhighlight %}



{% highlight text %}
## class       : SpatialPointsDataFrame 
## features    : 10000 
## extent      : -74.25467, -73.70042, 40.49932, 40.91101  (xmin, xmax, ymin, ymax)
## coord. ref. : NA 
## variables   : 1
## names       : prop_id 
## min values  :       1 
## max values  :   10000
{% endhighlight %}

<br />

To identify which property falls within which grid cell in the gridded layer created above, we use the **over( )** function. 


{% highlight r %}
which_poly <- over(prop,ny_grids) %>% data.table()
which_poly <- cbind(prop@data$prop_id,which_poly)
setnames(which_poly,names(which_poly),c('pid','grid_id'))
which_poly
{% endhighlight %}



{% highlight text %}
##          pid grid_id
##     1:     1    5700
##     2:     2    5789
##     3:     3    5613
##     4:     4    5613
##     5:     5    5614
##    ---              
##  9996:  9996    9121
##  9997:  9997    8603
##  9998:  9998    8969
##  9999:  9999    9707
## 10000: 10000    9893
{% endhighlight %}

<br />

You can see that each property is assigned a unique grid id value. Now you can use this information to create grid dummies or grid-year dummies to implement estimation similar to those done in @linn13.

<br />
<br />

## Session Information

---------------


{% highlight text %}
## R version 3.2.0 (2015-04-16)
## Platform: x86_64-apple-darwin13.4.0 (64-bit)
## Running under: OS X 10.10 (Yosemite)
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
## [1] methods   stats     graphics  grDevices utils     datasets  base     
## 
## other attached packages:
## [1] raster_2.4-15    ggplot2_1.0.1    dplyr_0.4.2      data.table_1.9.4
## [5] rgdal_1.0-4      sp_1.1-1         knitr_1.10.5    
## 
## loaded via a namespace (and not attached):
##  [1] Rcpp_0.12.0      magrittr_1.5     MASS_7.3-43      munsell_0.4.2   
##  [5] colorspace_1.2-6 lattice_0.20-33  R6_2.1.0         stringr_1.0.0   
##  [9] plyr_1.8.3       tools_3.2.0      parallel_3.2.0   grid_3.2.0      
## [13] gtable_0.1.2     DBI_0.3.1        assertthat_0.1   digest_0.6.8    
## [17] reshape2_1.4.1   formatR_1.2      evaluate_0.7     stringi_0.5-5   
## [21] scales_0.2.5     chron_2.3-47     proto_0.3-10
{% endhighlight %}

<br />
<br />

## References

---------------

{% reference linn13 %}
