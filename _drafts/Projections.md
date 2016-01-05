---
title: "Projections"
author: "nabil Abd..."
date: "November 18, 2015"
output: html_document
---



## Introduction 

Typically, computations with spatial data (e.g., kriging) require that there be a 
projection associated with the coordinates. The problem is, how do you find the 
projection your coordinates are in? And even if you know what the projection is, 
then how do you assign/associate it to your data? 

In the following, we seek to present some motivation for why it's important to 
be aware of how to work with projections, as well as what one can do with them, 
including common use-cases that typically arise in my work.

### Motivation: An Equivalence 

If you noticed on the [intro to kriging](http://rpubs.com/nabilabd/118172) for this 
series, there was an example with the meuse dataset:



```r
suppressPackageStartupMessages({
  library(sp)
  library(gstat)
  library(devtools)
  library(magrittr)
})
```

```
## Warning: package 'gstat' was built under R version 3.1.3
```

```r
dev_mode()
```

```
## Dev mode: ON
```

```r
library(dplyr, warn=F, quietly = T)

# re-initialize the two datasets
initialize <- function() {data(meuse); data(meuse.grid)}

data(meuse)
coordinates(meuse) <- ~ x + y

lzn.vgm <- variogram(log(zinc) ~ 1, meuse) # calculates sample variogram values
lzn.fit <- fit.variogram(lzn.vgm, vgm(1, "Sph", 900, 1))
```

However, note that in this kriging example, there was no 
projection specified with the coordinates of the `meuse` SPDF or for `meuse.grid`; rather, just the variables which represented the coordinates of the spatial domain. As mentioned in ASDAR, p.216-7, if no projection is specified, the predicted values from the `krige` function are simply the prediction of a linear model. To see this equivalence, we can perform both kinds of predictions and look at the difference between them: 


```r
data(meuse.grid)
coordinates(meuse.grid) <- ~ x + y

# krige predictions
lzn.kriged <- krige(log(zinc) ~ sqrt(dist), meuse, meuse.grid) # NB: different results if model=lzn.fit is set
```

```
## [ordinary or weighted least squares prediction]
```

```r
orig_kriging_res <- lzn.kriged %>% as.data.frame %>% tbl_df
orig_kriging_res
```

```
## Source: local data frame [3,103 x 4]
## 
##         x      y var1.pred  var1.var
##     (dbl)  (dbl)     (dbl)     (dbl)
## 1  181180 333740  6.994379 0.1952303
## 2  181140 333700  6.994379 0.1952303
## 3  181180 333700  6.712531 0.1932143
## 4  181220 333700  6.462898 0.1919190
## 5  181100 333660  6.994379 0.1952303
## 6  181140 333660  6.712531 0.1932143
## 7  181180 333660  6.501786 0.1920905
## 8  181220 333660  6.373262 0.1915663
## 9  181060 333620  6.900438 0.1944931
## 10 181100 333620  6.712531 0.1932143
## ..    ...    ...       ...       ...
```

```r
# lm works with dataframes, so prepare data
meuse_df <- as.data.frame(meuse)
mgrid_df <- as.data.frame(meuse.grid)

# calculate linear model predictions
linmod <- lm(log(zinc) ~ sqrt(dist), data = meuse_df)
model_res <- predict.lm(linmod, mgrid_df)

# compare predictions
orig_kriging_res2 <- orig_kriging_res %>% 
  mutate(mod_res = unname(model_res), difference = mod_res - var1.pred)

# the linear model and kriging results are the same
range(orig_kriging_res2$difference) # diff taken since the two columns aren't showing as "equal"
```

```
## [1] -1.065814e-14  3.552714e-15
```

Footnote: Initially regressed to a constant, which is silly. In this case, though, using a constant instead of, e.g., `sqrt(dist)`, then inverse distance weighting is used for the kriging.

The take-away, then, is that the "kriging" performed on the `meuse` dataset from before, does not actually contain the results that we want. To calculate actua kriging results, we need to assign the spatial data with the appropriate projection.

## Projecting

Once you know you want to project your spatial data, first you need to identify and find the appropriate projection, assign it, and then use it.

### Finding Projection Information

First, before we can assign a projection, sometimes we have to find how to identify what projection exactly we're looking to work with. Sometimes, if
you're lucky, there might be a column in the dataset containing the corresponding projection, like in some EPA air quality datasets, for example. But if that's not the case, then you can try using some of the sites below in `References` or do a quick Google search. As for precisely what information you're looking for, that's mentioned in the next section.

### Projection Assignment

To assign a projection, you'd need one of two things: 

1. The epsg code
2. the complete proj4 string

Once you know the name of the projection you're looking for (e.g., "Lambert Conformal Conical"), some basic Google-fu skills can go a long way in finding either of the two requirements needed for assigning a projection. 

The assignment would be done as follows, depending on which of two you have:


```r
proj4string(my_df) <- CRS("+init=epsg:EPSG_CODE")
proj4string(my_df) <- CRS("PROJ4_STRING")
```

The benefit of the first method (i.e., specifying the epsg code), is that it's 
usually much shorter than the proj4 string, while both methods provide basically 
the same information. Also, if you are ever need the proj4 string but you know the epsg code, then you can assign the epsg code, and the spatial object would automatically 
contain the proj4 string information. For example, 


```r
# reloading data so it is a dataframe and not SpatialPoints df
data(meuse) 
meuse %>% glimpse
```

```
## Observations: 155
## Variables: 14
## $ x       (dbl) 181072, 181025, 181165, 181298, 181307, 181390, 181165...
## $ y       (dbl) 333611, 333558, 333537, 333484, 333330, 333260, 333370...
## $ cadmium (dbl) 11.7, 8.6, 6.5, 2.6, 2.8, 3.0, 3.2, 2.8, 2.4, 1.6, 1.4...
## $ copper  (dbl) 85, 81, 68, 81, 48, 61, 31, 29, 37, 24, 25, 25, 93, 31...
## $ lead    (dbl) 299, 277, 199, 116, 117, 137, 132, 150, 133, 80, 86, 9...
## $ zinc    (dbl) 1022, 1141, 640, 257, 269, 281, 346, 406, 347, 183, 18...
## $ elev    (dbl) 7.909, 6.983, 7.800, 7.655, 7.480, 7.791, 8.217, 8.490...
## $ dist    (dbl) 0.00135803, 0.01222430, 0.10302900, 0.19009400, 0.2770...
## $ om      (dbl) 13.6, 14.0, 13.0, 8.0, 8.7, 7.8, 9.2, 9.5, 10.6, 6.3, ...
## $ ffreq   (fctr) 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,...
## $ soil    (fctr) 1, 1, 1, 2, 2, 2, 2, 1, 1, 2, 2, 1, 1, 1, 1, 1, 1, 1,...
## $ lime    (fctr) 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1, 1, 1,...
## $ landuse (fctr) Ah, Ah, Ah, Ga, Ah, Ga, Ah, Ab, Ab, W, Fh, Ag, W, Ah,...
## $ dist.m  (dbl) 50, 30, 150, 270, 380, 470, 240, 120, 240, 420, 400, 3...
```

```r
coordinates(meuse) <- ~ x + y # specify coordinates

# note that no projection is assigned yet to the SPDF:
slotNames(meuse)
```

```
## [1] "data"        "coords.nrs"  "coords"      "bbox"        "proj4string"
```

```r
slot(meuse, "proj4string")
```

```
## CRS arguments: NA
```

```r
# assigning epsg value results in proj4string assignment
proj4string(meuse) <- CRS("+init=epsg:28992") # see below about the number
slot(meuse, "proj4string") 
```

```
## CRS arguments:
##  +init=epsg:28992 +proj=sterea +lat_0=52.15616055555555
## +lon_0=5.38763888888889 +k=0.9999079 +x_0=155000 +y_0=463000
## +ellps=bessel
## +towgs84=565.4171,50.3319,465.5524,-0.398957388243134,0.343987817378283,-1.87740163998045,4.0725
## +units=m +no_defs
```

The next section illustrates how the projection is made use of in actual computations. And if you're wondering why not just keep it simple and specify the longitude/latitude coordinates, there is to be an example below in the next section also, demonstrating why that won't always work.

### Kriging with Projections

The help file for the `meuse` dataset indicates that the `x` and `y` fields 
are in RDH Netherlands-local coordinates. With some quick Googling, we find a web page [here](http://static-content.springer.com/esm/art%3A10.1186%2F1476-072X-11-41/MediaObjects/12942_2012_517_MOESM5_ESM.txt) that contains the EPSG code, 28992.



```r
data(meuse.grid)
coordinates(meuse.grid) <- ~ x + y
proj4string(meuse.grid) <- CRS("+init=epsg:28992") # or, 3857

lzn.vgm2 <- variogram(log(zinc) ~ 1, meuse) # calculates sample variogram values
lzn.fit2 <- fit.variogram(lzn.vgm, vgm(1, "Sph", 900, 1))

proj_kriging_res <- krige(log(zinc) ~ 1, meuse, meuse.grid, model=lzn.fit2)
```

```
## [using ordinary kriging]
```

```r
projected_res <- proj_kriging_res %>% as.data.frame %>% tbl_df
projected_res
```

```
## Source: local data frame [3,103 x 4]
## 
##         x      y var1.pred  var1.var
##     (dbl)  (dbl)     (dbl)     (dbl)
## 1  181180 333740  6.499624 0.3198084
## 2  181140 333700  6.622356 0.2520205
## 3  181180 333700  6.505166 0.2729855
## 4  181220 333700  6.387590 0.2955290
## 5  181100 333660  6.764492 0.1779424
## 6  181140 333660  6.635513 0.2022045
## 7  181180 333660  6.497551 0.2277395
## 8  181220 333660  6.361480 0.2524917
## 9  181060 333620  6.904623 0.1099896
## 10 181100 333620  6.780228 0.1280849
## ..    ...    ...       ...       ...
```

By comparing these kriging results using projected data with `orig_kriging_res` above, 
we can see the predicted values are rather different.

### But What's Wrong Mercator?  

At this point, you might be wondering, why do I have to find some obscure and foreign (re: non-Amurrican) projection when I can just use regular long-lat coordinates? 

Sometimes, even trying to assign the Mercator projection to your spatial data 
will result in an error, like if the coordinates are outside the range possible 
for long/lat coordinates. For example, where we use [this intro to coordinate reference systems](https://www.nceas.ucsb.edu/~frazier/RSpatialGuides/OverviewCoordinateReferenceSystems.pdf) to find that the long-lat code is 4326, try running the following code, which should produce an error: 


```r
data("meuse")
coordinates(meuse) <- ~ x + y
proj4string(meuse) <- CRS("+init=epsg:4326")
```

Additionally, in my experience there have been occasions where the kriging would 
not work if the data were unprojected (I think the `automap` package makes an 
assumption about the projection if none is specified), but that was some time back 
and the author has yet to find a reproducible example for how it fails.

#### Conversions

If you have coordinates in one reference system (e.g., RDH, to continue the Netherlands 
example from above), and would like to convert them coordinates you're more 
familiar with (e.g., long-lat), then you can use the `rgdal::spTransform` function. 
Note that depending on the computer system you're working with, it might take a little 
bit of effort to install `rgdal` (or, it did for me, but not sure how much of that 
was just me).

An important fact to keep in mind is that `spTransform` takes some spatial object, not a 
dataframe, as an argument, and returns a SpatialPoints object as well. So, if you're working 
with dataframes, you might have to do some converting back and forth in the process.

As an example, let's try to convert the RDH coordinates to long-lat. For this, 
you need the EPSG code or proj4 string of the CRS you want to convert to. The process is 
then relatively straightforward: 


```r
# select coordinates, and form SPDF
rdh_coords <- meuse %>% as.data.frame %>% select(x, y) 
rdh_coords %>% glimpse
```

```
## Observations: 155
## Variables: 2
## $ x (dbl) 181072, 181025, 181165, 181298, 181307, 181390, 181165, 1810...
## $ y (dbl) 333611, 333558, 333537, 333484, 333330, 333260, 333370, 3333...
```

```r
coordinates(rdh_coords) <- ~ x + y
proj4string(rdh_coords) <- CRS("+init=epsg:28992")

# convert to long/lat coords
longlat_coords <- rdh_coords %>% spTransform(CRS("+init=epsg:4326"))
longlat_coords %>% str
```

```
## Formal class 'SpatialPoints' [package "sp"] with 3 slots
##   ..@ coords     : num [1:155, 1:2] 5.76 5.76 5.76 5.76 5.76 ...
##   .. ..- attr(*, "dimnames")=List of 2
##   .. .. ..$ : chr [1:155] "1" "2" "3" "4" ...
##   .. .. ..$ : chr [1:2] "x" "y"
##   ..@ bbox       : num [1:2, 1:2] 5.72 50.96 5.76 50.99
##   .. ..- attr(*, "dimnames")=List of 2
##   .. .. ..$ : chr [1:2] "x" "y"
##   .. .. ..$ : chr [1:2] "min" "max"
##   ..@ proj4string:Formal class 'CRS' [package "sp"] with 1 slot
##   .. .. ..@ projargs: chr "+init=epsg:4326 +proj=longlat +datum=WGS84 +no_defs +ellps=WGS84 +towgs84=0,0,0"
```

```r
longlat_coords %>% as.data.frame %>% glimpse
```

```
## Observations: 155
## Variables: 2
## $ x (dbl) 5.758536, 5.757863, 5.759855, 5.761746, 5.761863, 5.763040, ...
## $ y (dbl) 50.99156, 50.99109, 50.99089, 50.99041, 50.98903, 50.98839, ...
```

In my work, this kind of conversion to long-lat coordinates can be extremely 
useful, to have some kind of sense of what the coordinates mean in terms of something 
I'm more familiar with. In particular, if I have coordinates in some non-Mercator 
projection, I keep those original coordinates, then also convert them to 
long-lat, and simply keep track of both, along with an identifier (e.g., grid cell ID) 
which uniquely identifies each element in the spatial domain. 


## Conclusion

Projections are important. And if you're working with spatial (or spatio-temporal) data 
beyond just assigning coordinates and times, if you want to do actual computations with 
that data, then knowing how to assign and make use of different projections will 
probably come in handy.


## References 

The following are helpful references relating to the topic

* [SpatialReference.org](http://spatialreference.org/)
* For [more detailed notes](http://www.remotesensing.org/geotiff/proj_list/) and 
  general information on various projections
* For [more on proj4 parameters](http://proj.maptools.org/gen_parms.html)
* Melanie Wood has a lot of very useful [work](https://www.nceas.ucsb.edu/~frazier/RSpatialGuides/OverviewCoordinateReferenceSystems.pdf) on working with spatial data in R.


## Workspace


```r
session_info()
```

```
## Session info --------------------------------------------------------------
```

```
##  setting  value                       
##  version  R version 3.1.2 (2014-10-31)
##  system   x86_64, darwin13.4.0        
##  ui       X11                         
##  language (EN)                        
##  collate  en_US.UTF-8                 
##  tz       America/New_York            
##  date     2016-01-05
```

```
## Packages ------------------------------------------------------------------
```

```
##  package    * version     date       source                          
##  assertthat   0.1         2013-12-06 CRAN (R 3.1.0)                  
##  DBI          0.3.1       2014-09-24 CRAN (R 3.1.2)                  
##  devtools   * 1.9.1       2015-09-11 CRAN (R 3.1.2)                  
##  digest       0.6.8       2014-12-31 CRAN (R 3.1.2)                  
##  dplyr      * 0.4.3.9000  2015-12-21 Github (hadley/dplyr@4f2d7f8)   
##  evaluate     0.8         2015-09-18 CRAN (R 3.1.3)                  
##  FNN          1.1         2013-07-31 CRAN (R 3.1.1)                  
##  formatR      1.2.1       2015-09-18 CRAN (R 3.1.3)                  
##  gstat      * 1.1-0       2015-10-18 CRAN (R 3.1.3)                  
##  intervals    0.15.0      2014-09-19 CRAN (R 3.1.1)                  
##  knitr        1.11        2015-08-14 CRAN (R 3.1.2)                  
##  lattice      0.20-33     2015-07-14 CRAN (R 3.1.3)                  
##  lazyeval     0.1.10.9000 2015-08-08 Github (hadley/lazyeval@ecb8dc0)
##  magrittr   * 1.5         2014-11-22 CRAN (R 3.1.2)                  
##  memoise      0.2.1       2014-04-22 CRAN (R 3.1.0)                  
##  R6           2.1.1       2015-08-19 CRAN (R 3.1.3)                  
##  Rcpp         0.12.2      2015-11-15 CRAN (R 3.1.3)                  
##  rgdal        1.0-7       2015-09-06 CRAN (R 3.1.2)                  
##  sp         * 1.2-1       2015-10-18 CRAN (R 3.1.2)                  
##  spacetime    1.1-4       2015-04-24 CRAN (R 3.1.3)                  
##  stringi      1.0-1       2015-10-22 CRAN (R 3.1.3)                  
##  stringr      1.0.0       2015-04-30 CRAN (R 3.1.3)                  
##  xts          0.9-7       2014-01-02 CRAN (R 3.1.0)                  
##  zoo          1.7-12      2015-03-16 CRAN (R 3.1.3)
```



