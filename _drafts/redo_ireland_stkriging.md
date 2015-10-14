---
title: "redo ireland spacio-temporal kriging"
author: "nabil"
date: "January 20, 2015"
output: html_document
---



```r
library(sp)
```

```
## Warning: package 'sp' was built under R version 3.1.3
```

```r
library(plyr)
```

```
## Warning: package 'plyr' was built under R version 3.1.3
```

```r
library(dplyr, warn=F)
library(tidyr)
library(ggmap)
```

```
## Warning: package 'ggmap' was built under R version 3.1.3
```

```
## Loading required package: ggplot2
## Google Maps API Terms of Service: http://developers.google.com/maps/terms.
## Please cite ggmap if you use it: see citation('ggmap') for details.
```

```r
library(stringr)
```

```
## Warning: package 'stringr' was built under R version 3.1.3
```

```r
library(magrittr, warn=F)
library(spacetime)
```

```
## Warning: package 'spacetime' was built under R version 3.1.3
```

```r
library(lubridate, warn=F)
```


```r
library(gstat)
```

```
## Warning: package 'gstat' was built under R version 3.1.3
```

```r
data(wind) 

# add columns of numerical coordinates
wind.loc[, c("y","x")] <- 
  wind.loc %>% 
  select(Latitude, Longitude) %>% 
  sapply(function(colm) colm %>% as.character %>% char2dms %>% as.numeric)

# specify spatial coordinates, and which projection used
coordinates(wind.loc) = ~ x + y
proj4string(wind.loc) = "+proj=longlat +datum=WGS84"
```



```r
# Plotting locations of the stations

#' slightly enlarge the default bounding box for sp objects
#' 
#' @param . an SPDF object
re_bound <- . %>% bbox %>% as.data.frame %>% 
  mutate(diff = max - min, min = min -.2 * diff, max = max + .2 * diff) %>% 
  select(-diff) %>% as.matrix %>% set_rownames(c("x", "y"))

# obtain map and form positions for station labels
my_map <- wind.loc %>% re_bound %>% get_map(., source = "osm")
offsets <- wind.loc %>% as.data.frame %>% select(Station, x, y) %>% 
  mutate(y = y + .1)

# map of Ireland's stations, for context
# todo: fix axes and labels
my_map %>% 
  ggmap(base_layer = ggplot(aes(x=x, y=y), data = wind.loc %>% as.data.frame)) + 
  geom_point(size=3, color="red") + 
  annotate("text", x = offsets$x, y = offsets$y, label=offsets$Station, size=4)
```

![plot of chunk unnamed-chunk-3](figure/unnamed-chunk-3-1.png) 



```r
wind <- wind %>% 
  mutate(
    date_time = ISOdate(year+1900, month, day), 
    jday = yday(date_time)
  )
```





```r
# normalize wind speed by subtracting the mean
normed_wind <- wind %>% 
  gather(stations, windspeed, RPT:MAL) %>% 
  mutate(windspeed = sqrt(0.5148 * windspeed)) %>% 
  mutate(windspeed = windspeed - mean(windspeed)) # need a second 'mutate'

# calculate average daily windspeed, and fitted values
day_means <- normed_wind %>% 
  group_by(jday) %>% 
  summarize(mwspeed = mean(windspeed)) %>% 
  {
    fitted_vals <- loess(mwspeed ~ jday, data=., span=.1)$fitted
    mutate(., smoothed_y = fitted_vals)
  }

# what did the paper do?
velocities <- normed_wind %>% 
  left_join(day_means) %>% 
  group_by(stations, jday) %>% 
  mutate(windspeed2 = windspeed - smoothed_y)
```

```
## Joining by: "jday"
```




```r
# convert points to spatial object
stations <- 4:15
wind.loc <- wind.loc[match(names(wind[stations]), wind.loc$Code),]
pts <- wind.loc %>% 
  coordinates %>% 
  set_rownames(wind.loc$Station) %>% 
  SpatialPoints(proj4string = CRS("+proj=longlat +datum=WGS84"))
```



Transform the coordinates


```r
library(rgdal)
```

```
## rgdal: version: 1.0-7, (SVN revision 559)
##  Geospatial Data Abstraction Library extensions to R successfully loaded
##  Loaded GDAL runtime: GDAL 1.11.3, released 2015/09/16
##  Path to GDAL shared files: /usr/local/Cellar/gdal/1.11.3/share/gdal
##  Loaded PROJ.4 runtime: Rel. 4.9.2, 08 September 2015, [PJ_VERSION: 492]
##  Path to PROJ.4 shared files: (autodetected)
##  Linking to sp version: 1.1-1
```

```r
# transform coordinates to utm 
utm29 <- CRS("+proj=utm +zone=29 +datum=WGS84")
pts <- pts %>% spTransform(utm29)
```



