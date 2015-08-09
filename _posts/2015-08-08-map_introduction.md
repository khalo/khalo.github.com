---
layout: post
title: Mapping point data on polygons using gplot2( )
tags: [R, spatial, ggplot2, grid, over]
categories: [R, spatial]
date: "08 August, 2015"
---
{% include JB/setup %} 
 
---------------

In this post, I will show how to make a map of spatial points data (irrigation wells) overlayed on spatial polygons (counties in Nebraska) using __ggplot( )__. You will learn:

* how to transform a SpatialPolygonDataFrame into a data frame that is compatible with __ggplot( )__ for mapping using the __fortify( )__ function from the __ggplot2__ package

* change the coordinate reference system (CRS) using **spTransform( )** from the **sp** package.
    
* create a map of points data overlayed on spatial polygons.

Let's first load packages we are going to use in this session.


{% highlight r %}
library(rgdal)
library(data.table)
library(dplyr)
library(ggplot2)
library(raster)
{% endhighlight %}






<br />

We now import a Nebraska county shape file and select Chase, Dundy, and Perkins counties.


{% highlight r %}
NE_county <- readOGR(dsn = ".", "county_bound")
urnrd <- NE_county[NE_county@data$COUNTY_NAM %in% c("CHASE","DUNDY","PERKINS"),]
{% endhighlight %}

<br />

Unfortunately, a SpatialPolygonDataFrame cannot be used immediately for mapping using **ggplot( )**. The ggplot2 package, however, offers a function called **fortify( )**, which transforms a SpatialPolygonDataFrame into a data frame that is compatible with mapping using **ggplot( )**.


{% highlight r %}
urnrd_f <- fortify(urnrd, region = "COUNTY_NAM")
head(urnrd_f)
{% endhighlight %}



{% highlight text %}
##      long      lat order  hole piece   group    id
## 1 1083135 321605.2     1 FALSE     1 CHASE.1 CHASE
## 2 1083251 321603.6     2 FALSE     1 CHASE.1 CHASE
## 3 1083535 321599.5     3 FALSE     1 CHASE.1 CHASE
## 4 1083555 321599.4     4 FALSE     1 CHASE.1 CHASE
## 5 1084352 321588.2     5 FALSE     1 CHASE.1 CHASE
## 6 1084828 321573.1     6 FALSE     1 CHASE.1 CHASE
{% endhighlight %}

<br />

Now, you may have noticed that in the urnrd_f dataset, variables long and lat have values that cannot possibly be right as valid longitude and latitude. This is because the original shape file of Nebraska counties uses a different geographical representation of locations than longitude and latitude. You can see this by using **proj4string( )** function.


{% highlight r %}
proj4string(urnrd)
{% endhighlight %}



{% highlight text %}
## [1] "+proj=lcc +lat_1=40 +lat_2=43 +lat_0=39.83333333333334 +lon_0=-100 +x_0=500000.0000000001 +y_0=0 +datum=NAD83 +units=us-ft +no_defs +ellps=GRS80 +towgs84=0,0,0"
{% endhighlight %}

<br />

It turns out the Lambert conformal conic (lcc) projection method is used in this dataset as you can see from "+proj=lcc" in the output. On the other hand, the location of irrigation wells is represented simply by longitude and latitude, unprojected.


{% highlight r %}
wells <- fread('fake_wells.csv')
wells
{% endhighlight %}



{% highlight text %}
##            lat      long      gpm
##    1: 40.54449 -101.8650 817.9014
##    2: 40.49558 -101.3700 953.2952
##    3: 40.47911 -101.9014 934.3472
##    4: 40.61308 -101.9029 752.5348
##    5: 40.38938 -101.9422 856.6183
##   ---                            
## 2918: 40.87906 -101.9236 692.4241
## 2919: 40.55861 -101.7935 687.3473
## 2920: 40.80561 -101.5178 603.0804
## 2921: 40.32196 -101.6708 745.0353
## 2922: 40.09223 -101.4382 236.5328
{% endhighlight %}

<br />

As you can imagine, in order for **ggplot( )** to correctly map both  counties and wells on a single map, they should have the same coordinate reference system (CRS). To do this, let's first transform the well data into a SpatialPointsDataFrame by using the **coordinates( )** function.


{% highlight r %}
coordinates(wells) <- c('long','lat')
proj4string(wells)
{% endhighlight %}



{% highlight text %}
## [1] NA
{% endhighlight %}

<br />

As you can see above, the CRS has not been declared for the wells data at this point. So, let's define one for the wells.


{% highlight r %}
proj4string(wells) <-  CRS("+proj=longlat")
{% endhighlight %}

<br />

Now, we can use **spTransform( )** to change the CRS of wells to that of urnrd_f. 


{% highlight r %}
wells_lcc <- spTransform(wells, urnrd@proj4string)
wells_lcc
{% endhighlight %}



{% highlight text %}
## class       : SpatialPointsDataFrame 
## features    : 2922 
## extent      : 1067358, 1295641, 66445.43, 431259.7  (xmin, xmax, ymin, ymax)
## coord. ref. : +proj=lcc +lat_1=40 +lat_2=43 +lat_0=39.83333333333334 +lon_0=-100 +x_0=500000.0000000001 +y_0=0 +datum=NAD83 +units=us-ft +no_defs +ellps=GRS80 +towgs84=0,0,0 
## variables   : 1
## names       :              gpm 
## min values  :                0 
## max values  : 1378.17256086539
{% endhighlight %}

<br />

You can see that wells now have the same CRS as county boundaries. As we only want a data frame of coordinates for mapping instead of a SpatialPointsDataFrame,


{% highlight r %}
wells_to_map <- wells_lcc@coords %>% data.table()
{% endhighlight %}

<br />

With all the data at our fingertips, let's create a map! Here is the map of selected counties in Nebraska.


{% highlight r %}
map <- ggplot(urnrd_f, aes(long, lat)) + 
 geom_polygon(aes(group=group),fill="#D3D3D3",colour="black",size=0.2) +
 coord_equal(ratio=1) +
 labs(x="", y="") +
 theme(
 	axis.ticks.y = element_blank(),
 	axis.text.y = element_blank(), 
 	axis.ticks.x = element_blank(),
 	axis.text.x = element_blank(),
 	panel.background = element_rect(fill='white')
 	)
{% endhighlight %}

![plot of chunk unnamed-chunk-1]({{ site.url }}/images/spatial_map-unnamed-chunk-1-1.png) 

<br />

Now, let's add wells onto this map.


{% highlight r %}
map <- map + 
	geom_point(data=wells_to_map,aes(x=long,y=lat),size=0.7)
{% endhighlight %}

![plot of chunk disp]({{ site.url }}/images/spatial_map-disp-1.png) 

<br />
<br />

### Session Information

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
## [1] maptools_0.8-36  raster_2.4-15    ggplot2_1.0.1    dplyr_0.4.2     
## [5] data.table_1.9.4 rgdal_1.0-4      sp_1.1-1         knitr_1.10.5    
## 
## loaded via a namespace (and not attached):
##  [1] Rcpp_0.12.0      magrittr_1.5     MASS_7.3-43      munsell_0.4.2   
##  [5] colorspace_1.2-6 lattice_0.20-33  R6_2.1.0         stringr_1.0.0   
##  [9] plyr_1.8.3       tools_3.2.0      parallel_3.2.0   grid_3.2.0      
## [13] gtable_0.1.2     DBI_0.3.1        rgeos_0.3-11     assertthat_0.1  
## [17] digest_0.6.8     reshape2_1.4.1   formatR_1.2      evaluate_0.7    
## [21] labeling_0.3     stringi_0.5-5    scales_0.2.5     foreign_0.8-65  
## [25] chron_2.3-47     proto_0.3-10
{% endhighlight %}


