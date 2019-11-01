
|                              |
| ---------------------------- |
| itle: “PennMUSA”             |
| uthor: “Tyler Morgan-Wall”   |
| ate: “9/14/2019”             |
| utput: html\_document        |
| ditor\_options:              |
| chunk\_output\_type: console |

\#UPenn Masterclass: 3D Mapping and Visualization with R and Rayshader

Tyler Morgan-Wall (@tylermorganwall), Institute for Defense Analyses

Personal website: <https://www.tylermw.com> Rayshader website:
<https://www.rayshader.com> Github:
<https://www.github.com/tylermorganwall>

``` r
# install.packages("whitebox", repos="http://R-Forge.R-project.org")
remotes::install_github("giswqs/whiteboxR")
```

    ## Skipping install of 'whitebox' from a github remote, the SHA1 (af5f3c0d) has not changed since last install.
    ##   Use `force = TRUE` to force installation

``` r
whitebox::wbt_init()
library(ggplot2)
library(whitebox)
library(rayshader)
library(geoviz)
library(raster)
```

    ## Loading required package: sp

``` r
library(spatstat)
```

    ## Loading required package: spatstat.data

    ## Loading required package: nlme

    ## 
    ## Attaching package: 'nlme'

    ## The following object is masked from 'package:raster':
    ## 
    ##     getData

    ## Loading required package: rpart

    ## Registered S3 method overwritten by 'spatstat':
    ##   method      from  
    ##   plot.imlist imager

    ## 
    ## spatstat 1.61-0       (nickname: 'Puppy zoomies') 
    ## For an introduction to spatstat, type 'beginner'

    ## 
    ## Attaching package: 'spatstat'

    ## The following objects are masked from 'package:raster':
    ## 
    ##     area, rotate, shift

``` r
library(spatstat.utils)
library(suncalc)
library(sp)
setwd("~/Desktop/musa/")
```

# Hillshading with Rayshader

We’re going to start by demoing how digital elevation models (DEMs) have
traditionally been displayed. Here, we load in a DEM of the River
Derwent in Hobart, Tasmania. We’ll plot it using both the image and plot
functions to see how elevation data is traditionally displayed–mapping
elevation directly to color.

Let’s take a raster object and extract a bare R matrix to work with
rayshader. To convert the data from the raster format to a bare matrix,
we will use the rayshader function `raster_to_matrix()`. We will first
visualize the data with the base R `image()` function.

``` r
loadzip = tempfile() 
download.file("https://tylermw.com/data/dem_01.tif.zip", loadzip)
hobart_tif = raster::raster(unzip(loadzip, "dem_01.tif"))
unlink(loadzip)

hobart_mat = raster_to_matrix(hobart_tif)
unlink("dem_01.tif")

image(hobart_mat)
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

The `image()` function orients data differently than it is in
reality–the image is flipped vertically. Let’s use rayshader’s
`height_shade()` function, which performs the same mapping but orients
the resulting map correctly. Here, the default uses `terrain.colors()`
instead of the yellow-to-red mapping.

``` r
hobart_mat %>%
  height_shade() %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

Where in nature does color map to elevation? Usually, you get this kind
of mapping on large scale topographic features, such as a tree line on a
mountain. Here’s a nice illustration o from the 1848 book by Alexander
Keith Johnston showing that elevation-to-color mapping:

![Figure X: Height maps to elevation](images/topography_color.png)

On smaller scale (more human-sized) features like hills or dunes, the
color is dominated by instead by lighting and how the sun hits the
surface. The surfaces aren’t colored by elevation–rather, their colors
is largely determined by the angle the surface along with the direction
of the light.

Note the (sometimes drastic) change in surface color when the lighting
changes:

![Figure X: How lighting affects terrain colors. Great Sand Dunes
National Park, CO.](images/sanddunessmall.png)

Rather than color by elevation, let’s try coloring the surface by the
direction the slope is facing and the steepness of that slope. This is
implemented in the rayshader function, `sphere_shade()`. Here’s a video
explanation of how it works:

<br> <video controls>
<source src="https://www.tylermw.com/wp-content/uploads/2018/06/fullcombined_web.mp4" type="video/mp4">
</video> <br>

We’ll take this and plug it into the `sphere_shade()` function, which
performs this mapping of surface direction and slope to color.

``` r
#Example of data stored in hobart_mat object:
hobart_mat[1:10,1:10]
```

    ##       [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8] [,9] [,10]
    ##  [1,]  750  749  748  745  743  743  743  743  744   744
    ##  [2,]  751  751  751  749  749  748  747  748  749   749
    ##  [3,]  750  753  754  753  752  751  751  752  753   754
    ##  [4,]  749  753  756  757  756  755  757  758  758   760
    ##  [5,]  749  754  757  759  760  760  761  763  765   767
    ##  [6,]  755  760  761  765  769  769  767  768  770   773
    ##  [7,]  762  767  768  771  772  773  772  772  775   777
    ##  [8,]  768  771  771  772  774  774  775  776  778   781
    ##  [9,]  769  771  771  771  773  775  776  777  779   782
    ## [10,]  767  770  770  770  771  773  775  776  779   782

``` r
hobart_mat %>%
  sphere_shade() %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

We note right away that it’s much more apparent in this image that there
is a body of water (the River Derwent) in the middle of this image,
weaving between the two mountains. While this feature wasn’t apparent in
the more tradition height-to-color mapping, it becomes immediately
visible when we color by slope.

In fact, rayshader includes two functions, `detect_water()` and
`add_water()`, that allow you to detect and add bodies of water directly
to the hillshade via the elevation data. This works by looking for
large, relatively flat, connected regions in a DEM. Usually, large
extremely flat areas are water, but occasionally these functions can
result in false positives. `detect_water()` provides the parameters
`min_area` and `cutoff` in `detect_water()` so the user can try to .
`min_area` specifies the minimum number of connected flat points that
constitute a body of water. `cutoff` determines how vertical a surface
as to be to be defined as flat: `cutoff = 1.0` only accepts areas
pointing straight up as flat, while `cutoff = 0.99` allows for areas
that are slightly less than flat to be classified as water. The default
is `cutoff = 0.999`.

``` r
hobart_mat %>%
  sphere_shade() %>%
  add_water(detect_water(hobart_mat, min_area = 10), color="blue") %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

If increase the minimum required area to be classified as water, we will
fix the flat areas at the top of the map that are being erroneously
catagorized as water.

``` r
hobart_mat %>%
  sphere_shade() %>%
  add_water(detect_water(hobart_mat,  min_area = 200), color="blue") %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

The default is to look for flat regions 1/400th the area of the matrix.
There’s nothing special about this number, but it’s a default that
tended to work fairly well for several datasets I tested it on, but it’s
a parameter you can adjust to suit your dataset. Here we’ll also use the
default water color.

``` r
hobart_mat %>%
  sphere_shade() %>%
  add_water(detect_water(hobart_mat)) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

Rayshader’s name comes the method it uses to calculate hillshades:
raytracing, which realisticly simulates how light travels across the
elevation model. Most traditional methods of hillshading only use the
local angle that the surface makes with the light, and do not take into
account areas that actually cast a shadow. This basic type of
hillshading is sometimes referred to as “lambertian”, and is implemented
in the function `lamb_shade()`.

``` r
hobart_mat %>%
  lamb_shade(zscale=33) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

To shade surfaces using raytracing, rayshader draws rays originating
from each point towards a light source, specified using the `sunangle`
and `anglebreaks` argument. The light by default has a fixed angular
width the size of the sun, but the distribution can also be set in the
argument `anglebreaks`. Here are two gifs showing how rayshader
calculates shadows with `ray_shade`:

![Figure X: Demonstration of how ray\_shade() raytraces
images](images/ray.gif)

![Figure X: ray\_shade() determines the amount of shadow at a single
source by sending out rays and testing for intersections with the
heightmap. The amount of shadow is proportional to the number of rays
that don’t make it to the light.](images/bilinear.gif)

Let’s add a layer of shadows to this map, using the `add_shadow()` and
`ray_shade()` functions. We layer our shadows to our `sphere_shade()`
base color layer.

``` r
hobart_mat %>%
  lamb_shade(zscale=33) %>%
  add_shadow(ray_shade(hobart_mat, zscale=33, sunaltitude = 3, lambert = FALSE), 0.3) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

We can combine all these together to make our final 2D map:

``` r
hobart_mat %>%
  sphere_shade() %>%
  add_water(detect_water(hobart_mat), color="lightblue") %>%
  add_shadow(ray_shade(hobart_mat,zscale=33, sunaltitude = 3,lambert = FALSE), max_darken = 0.5) %>%
  add_shadow(lamb_shade(hobart_mat,zscale=33,sunaltitude = 3), max_darken = 0.5) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

We can adjust the highlight/sun direction using the `sunangle` argument
in both the `sphere_shade()` function and the `ray_shade()` function.

We’ll start witht the default angle: 315 degrees, or the light from the
Northwest. One question you might ask yourself:

``` r
#Default angle: 315 degrees.

hobart_mat %>%
  sphere_shade() %>%
  add_water(detect_water(hobart_mat), color="lightblue") %>%
  add_shadow(ray_shade(hobart_mat, zscale=33, sunaltitude = 5,lambert = FALSE), 
             max_darken = 0.5) %>%
  add_shadow(lamb_shade(hobart_mat,zscale=33,sunaltitude = 5), max_darken = 0.8) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

``` r
#45 degrees

hobart_mat %>%
  sphere_shade(sunangle = 45) %>%
  add_water(detect_water(hobart_mat), color="lightblue") %>%
  add_shadow(ray_shade(hobart_mat,sunangle = 45, zscale=33, sunaltitude = 5,lambert = FALSE), 
             max_darken = 0.5) %>%
  add_shadow(lamb_shade(hobart_mat,zscale=33,sunaltitude = 5), max_darken = 0.8) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-11-2.png)<!-- -->

``` r
#135 degrees

hobart_mat %>%
  sphere_shade(sunangle = 135) %>%
  add_water(detect_water(hobart_mat), color="lightblue") %>%
  add_shadow(ray_shade(hobart_mat,sunangle = 135, zscale=33, sunaltitude = 5,lambert = FALSE), 
             max_darken = 0.5) %>%
  add_shadow(lamb_shade(hobart_mat,zscale=33,sunaltitude = 5), max_darken = 0.8) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-11-3.png)<!-- -->

``` r
#225 degrees

hobart_mat %>%
  sphere_shade(sunangle = 225) %>%
  add_water(detect_water(hobart_mat), color="lightblue") %>%
  add_shadow(ray_shade(hobart_mat,sunangle = 225, zscale=33, sunaltitude = 5,lambert = FALSE), 
             max_darken = 0.5) %>%
  add_shadow(lamb_shade(hobart_mat,zscale=33,sunaltitude = 5), max_darken = 0.8) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-11-4.png)<!-- -->

We can also add the effect of ambient occlusion, which in cartography is
sometimes called the “sky view factor.” When light travels through the
atmosphere, it scatters. This scattering turns the entire sky into a
light source, so when less of the sky is visible (e.g. in a valley) it’s
darker than when the entire sky is visible (e.g. on a mountain ridge).

Let’s calculate the ambient occlusion shadow layer for the Hobart data,
and layer it onto the rest of the map.

``` r
hobart_mat %>%
  ambient_shade() %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/ambient_occlusion-1.png)<!-- -->

``` r
hobart_mat %>%
  sphere_shade() %>%
  add_water(detect_water(hobart_mat), color="lightblue") %>%
  add_shadow(ray_shade(hobart_mat, zscale=33, sunaltitude = 5,lambert = FALSE), 
             max_darken = 0.5) %>%
  add_shadow(lamb_shade(hobart_mat,zscale=33, sunaltitude = 5), max_darken = 0.7) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/ambient_occlusion-2.png)<!-- -->

``` r
hobart_mat %>%
  sphere_shade() %>%
  add_water(detect_water(hobart_mat), color="lightblue") %>%
  add_shadow(ray_shade(hobart_mat, zscale=33, sunaltitude = 5,lambert = FALSE), 
             max_darken = 0.5) %>%
  add_shadow(lamb_shade(hobart_mat,zscale=33, sunaltitude = 5), max_darken = 0.7) %>%
  add_shadow(ambient_shade(hobart_mat), max_darken = 0) %>%
  plot_map()
```

![](MusaMasterclass_files/figure-gfm/ambient_occlusion-3.png)<!-- -->

# 3D Mapping with Rayshader

Now that we know how to perform basic hillshading, we can begin the real
fun part: making 3D maps. In rayshader, we do that simply by swapping
out `plot_map()` with `plot_3d()`, and adding the heightmap to the
function call. We don’t want to re-compute the shadows every time we
replot the landscape, so lets save them to a
variable.

``` r
rayshadows = ray_shade(hobart_mat, sunaltitude=3, zscale=33, lambert = FALSE)
lambshadows = lamb_shade(hobart_mat, sunaltitude=3, zscale=33)
ambientshadows = ambient_shade(hobart_mat)

hobart_mat %>%
  sphere_shade() %>%
  add_water(detect_water(hobart_mat), color="lightblue") %>%
  add_shadow(rayshadows, max_darken = 0.5) %>%
  add_shadow(lambshadows, max_darken = 0.7) %>%
  add_shadow(ambientshadows, max_darken = 0) %>%
  plot_3d(hobart_mat, zscale=10,windowsize=c(1000,1000))

render_snapshot(clear=TRUE)
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-12-1.png)<!-- -->

This opens up an `rgl` window that displays the 3D plot. Draw to
manipulate the plot, and control/ctrl drag to zoom in and out. To close
it, we can either close the window itself, or type in
`rgl::rgl.close()`.

Just visualizing this on your screen is fun when exploring the data, but
we would also like to export our figure to an image file. If you want to
take a snapshot of the current view, rayshader provide the
`render_snapshot()` function. If you use this without a filename, it
will write and display the plot to the current device. With a filename,
it will write the image to a PNG file in the local directory. For
variety, let’s also change the background/shadow color (arguments
`background` and `shadowcolor`), depth of rendered ground/shadow
(arguments `soliddepth` and `shadowdepth`), and add a title to the plot.

``` r
hobart_mat %>%
  sphere_shade() %>%
  add_water(detect_water(hobart_mat), color="lightblue") %>%
  add_shadow(rayshadows, max_darken = 0.5) %>%
  add_shadow(lambshadows, max_darken = 0.7) %>%
  add_shadow(ambientshadows, max_darken = 0) %>%
  plot_3d(hobart_mat, zscale=10,windowsize=c(1000,1000), 
          phi = 40, theta = 135, zoom = 0.9, 
          background = "grey30", shadowcolor = "grey5", 
          soliddepth = -50, shadowdepth = -100)

render_snapshot(title_text = "River Derwent, Tasmania", 
                title_font = "Helvetica", 
                title_size = 50,
                title_color = "grey90")
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

``` r
render_snapshot(filename = "derwent.png")

#Delete the file
unlink("derwent.png")
```

If we want to programmatically change the camera, we can do so with the
`render_camera()` function. We can adjust four parameters: `theta`,
`phi`, `zoom`, and `fov` (field of view). Changing `theta` orbits the
camera around the scene, while changing `phi` changes the angle above
the horizon at which the camera is located. Here is a graphic
demonstrating this relation:

![Figure X: Demonstration of how ray\_shade() raytraces
images](images/spherical_coordinates_fixed.png)

`zoom` magnifies the current view (smaller numbers = larger
magnification), and `fov` changes the field of view. Higher `fov` values
correspond to a more pronounced perspective effect, while `fov = 0`
corresponds to a orthographic camera.

Here’s different views using the camera:

``` r
render_camera(theta = 90, phi = 30, zoom = 0.7, fov = 0)
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-14-1.png)<!-- -->

``` r
render_camera(theta = 90, phi = 30, zoom = 0.7, fov = 90)
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-14-2.png)<!-- -->

``` r
render_camera(theta = 120, phi = 20, zoom = 0.3, fov = 90)
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-14-3.png)<!-- -->

We can also use some post-processing effects to help guide our viewer
through our visualization’s 3D space. The easiest method of directing
the viewer’s attention is directly labeling the areas we’d like them to
look at, using the `render_label()` function. This function takes the
indices of the matrix coordinate area of interest `x` and `y`, and
displays a `text` label at altitude
`z`.

``` r
render_label(hobart_mat, "River Derwent", textcolor ="white", linecolor="white",
             x = 450, y = 260, z = 1400, textsize = 2.5, linewidth = 4, zscale = 10)
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-15-1.png)<!-- -->

``` r
render_label(hobart_mat, "Jordan River (not that one)", textcolor ="white", linecolor="white",
             x = 450, y = 140, z = 1400, textsize = 2.5, linewidth = 4, zscale = 10, dashed = TRUE)
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-15-2.png)<!-- -->

We can replace all existing text with the `clear_previous = TRUE`, or
clear everything by calling `render_label(clear_previous = TRUE)` with
no other arguments.

``` r
render_camera(zoom = 0.9, phi=50, theta=-45,fov=0)
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-16-1.png)<!-- -->

``` r
render_label(hobart_mat, "Mount Faulkner", textcolor ="white", linecolor="white",
             x = 135, y = 130, z = 2500, textsize = 2, linewidth = 3, zscale = 10, clear_previous = TRUE)
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-16-2.png)<!-- -->

``` r
render_label(hobart_mat, "Mount Dromedary", textcolor ="white", linecolor="white",
             x = 320, y = 390, z = 1000, textsize = 2, linewidth = 3, zscale = 10)
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-16-3.png)<!-- -->

``` r
render_label(clear_previous = TRUE)
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-16-4.png)<!-- -->

``` r
render_depth(focus = 0.70, preview_focus = TRUE)
```

    ## [1] "Focal range: 0.21476-0.756891"

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-17-1.png)<!-- -->

``` r
render_depth(focus = 0.7, focallength = 200)
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-17-2.png)<!-- -->

``` r
rgl::rgl.close()
```

One of the. Here, we’ll use the built-in `montereybay` dataset, which
includes both bathymetric and topographic data for Monterey Bay,
California. We’ll include a transparent water layer by setting `water =
TRUE` in `plot_3d()`, and add lines showing the edges of the water by
setting `waterlinecolor`.

``` r
montereybay %>%
  sphere_shade() %>%
  plot_3d(montereybay, theta=-45, water=TRUE, waterlinecolor = "white",windowsize = c(1000,1000))
```

    ## `montereybay` dataset used with no zscale--setting `zscale=50`.  For a realistic depiction, raise `zscale` to 200.

``` r
render_snapshot()
```

![](MusaMasterclass_files/figure-gfm/unnamed-chunk-18-1.png)<!-- -->

<https://maps.psiee.psu.edu/preview/map.ashx?layer=2021>

``` r
rgl::rgl.close()
```

Now that we know how to use rayshader, let’s use it to create and
visualize some data\! Back in 2015, the New York Times Upshot had a
fantastic piece by Quoctrung Bui and Jeremy White on mapping the shadows
of New York
City.

<https://www.nytimes.com/interactive/2016/12/21/upshot/Mapping-the-Shadows-of-New-York-City.html>

They took three days in the year and mapped out how much each spot in
the city spent in shadow: the winter solstice (December 21st), the
summer solstice (June 21st), and the autumnal equinox (Sept 22nd). They
worked with researchers at the Tandon school of Engineering at NYU to
develop a raytracer to perform these calculations. This type of analysis
is particularly timely with the explosion of so-called “pencil towers”
in Manhattan–extremely tall, skinny skyscrapers lining Central Park.
When

We are going to perform and visualize a similar analysis for erecting a
hypothetic “pencil tower” in West Philadelphia using lidar data from
Pennsylvania and rayshader.

Let’s start by loading some lidar data into R. Here’s a link to several
PDF files listing instructions on how to load data. We’ll start with
philly.pdf to get instructions for downloading

Let’s walk through downloading a specific dataset:

Backup: <https://www.tylermw.com/data/26849E233974N.zip>

Now, let’s load some lidar data into R. We’ll do this using the
`whitebox` package, which has several functions for manipulating and
transforming lidar data, among many other useful features. Here, we’re
going to load our lidar dataset of Penn Park, right by the Schuylkill.

While this runs, let’s talk
about

``` r
# whitebox::wbt_lidar_ransac_planes(path.expand("~/Desktop/musa/26849E233974N.las"), num_iter = 1,
#                                     output = path.expand("~/Desktop/musa/philly_level.las"))
# 
# whitebox::wbt_lidar_tin_gridding(path.expand("~/Desktop/musa/philly_level.las"),
#                                  output = path.expand("~/Desktop/musa/phillydem.tif"), minz=0,
#                                  resolution = 1, exclude_cls = c(3,4,5,7,9,18))
```

Backup: <https://www.tylermw.com/data/phillydem.tif>

``` r
LongLatToUTM = function(x,y,zone){
 xy = data.frame(ID = 1:length(x), X = x, Y = y)
 coordinates(xy) = c("X", "Y")
 proj4string(xy) = CRS("+proj=longlat +datum=NAD83") 
 res = spTransform(xy, CRS(paste("+proj=utm +zone=",zone," ellps=GRS80",sep='')))
 return(as.data.frame(res))
}

owin2Polygons = function(x, id="1") {
  stopifnot(is.owin(x))
  x = as.polygonal(x)
  closering = function(df) { df[c(seq(nrow(df)), 1), ] }
  pieces = lapply(x$bdry,
                   function(p) {
                     Polygon(coords=closering(cbind(p$x,p$y)),
                             hole=is.hole.xypolygon(p))  })
  z = Polygons(pieces, id)
  return(z)
}

tess2SP = function(x) {
  stopifnot(is.tess(x))
  y = tiles(x)
  nam = names(y)
  z = list()
  for(i in seq(y))
    z[[i]] = owin2Polygons(y[[i]], nam[i])
  return(SpatialPolygons(z))
}

owin2SP = function(x) {
  stopifnot(is.owin(x))
  y = owin2Polygons(x)
  z = SpatialPolygons(list(y))
  return(z)
}
```

``` r
# phillyraster = raster::raster("phillydem.tif")
# buildingmat = raster_to_matrix(phillyraster)
# buildingmatsmall = reduce_matrix_size(buildingmat,0.5)
```

``` r
# library(lubridate)
# 
# #30 minute buffer
# datetimephillyrise = ymd_hms("2019-06-21 04:35:00", tz = "EST")
# datetimephillyset = ymd_hms("2019-06-21 19:30:00", tz = "EST")
# 
# temptime = datetimephillyrise
# philly_shadows = list()
# sunanglelist = list()
# counter = 1
# while(temptime < datetimephillyset) {
#   sunangles = suncalc::getSunlightPosition(date = temptime, lat = 39.9526, lon = -75.1652)[4:5]*180/pi
#   print(temptime)
#   sunanglelist[[counter]] = temptime
#   # print(sunangles)
#   # philly_shadows[[counter]] = ray_shade(buildingmatsmall,
#   #                                 sunangle = sunangles$azimuth+180,
#   #                                 sunaltitude = sunangles$altitude,
#   #                                 lambert = FALSE, zscale=2,
#   #                                 multicore = TRUE)
#   temptime = temptime + duration("180s")
#   counter = counter + 1
# }
# 
# for(i in 1:280) {
#   magick::image_read(glue::glue("phillyshadow{i}.png")) %>%
#     magick::image_resize("800x800") %>%
#     magick::image_composite(magick::image_read("overlay.png")) %>%
#     magick::image_annotate(paste0("University of Pennsylvania, ", as.character(sunanglelist[[i]])), 
#                        location = paste0("+20+8"),
#                        size = 30, color = "black", 
#                        font = "Helvetica") %>%
#     magick::image_write(glue::glue("smallphilly{i}.png"))
# }
# 
# sphere_shade(buildingmatsmall,colorintensity = 20) -> sphereshaded
# 
# for(i in 1:length(philly_shadows)) {
# sphereshaded %>%
#   add_shadow(philly_shadows[[i]],0.3) %>%
#   save_png(filename=glue::glue("phillyshadow{i}"))
# }
# 
# shadowcoverage = philly_shadows[[1]]
# for(i in 2:15) {
#   shadowcoverage = shadowcoverage +  philly_shadows[[i]]
# }
# shadowcoverage = shadowcoverage/15
# 
# plot(phillyraster)
# ployval = clickpoly(nv=4,add=TRUE)
# test2 = rasterize(owin2SP(ployval),phillyraster,500)
# philly_with_building = merge(test2,phillyraster)
# 
# philly_with_building_mat = raster_to_matrix(philly_with_building)
# 
# philly_building_shadows = list()
# sunangles = suncalc::getSunlightPosition(date = as.POSIXct(sprintf("2019-06-21 %02d:00:00", i+4),tz="EST"), lat = 39.9526, lon = -75.1652)[4:5]*180/pi
# philly_building_shadows[[1]] = ray_shade(philly_with_building_mat,
#                                 sunangle = sunangles$azimuth+180, sunaltitude = sunangles$altitude,
#                                 lambert = FALSE,zscale=2,
#                                 multicore = TRUE)
# philly_with_building_mat %>%
#   sphere_shade(colorintensity = 20) %>%
#   add_shadow(philly_building_shadows[[1]],0.3) %>%
#   plot_map()
# 
# 
# for(i in 2:15) {
#   sunangles = suncalc::getSunlightPosition(date = as.POSIXct(sprintf("2019-06-21 %02d:00:00", i+4),tz="EST"), lat = 39.9526, lon = -75.1652)[4:5]*180/pi
#   print(sunangles)
#   philly_building_shadows[[i]] = ray_shade(philly_with_building_mat,
#                                   sunangle = sunangles$azimuth+180, sunaltitude = sunangles$altitude,
#                                   lambert = FALSE,zscale=2,
#                                   multicore = TRUE)
# }
# 
# shadowcoverage_building = philly_building_shadows[[1]]
# for(i in 2:15) {
#   shadowcoverage_building = shadowcoverage_building +  philly_building_shadows[[i]]
# }
# shadowcoverage_building = shadowcoverage_building/15
# 
# 
# rayrender:::fliplr(shadowcoverage-shadowcoverage_building) %>%
#   height_shade(texture = viridis::viridis(256)) %>%
#   plot_map()
# 
# phillydem2 = phillydem
# phillydem2[phillydem2<0] = 0
# 
# phillydem2 %>%
#   sphere_shade(colorintensity = 10) %>%
#   plot_3d(phillydem2,windowsize = c(1000,1000))
```

``` r
# whitebox::lidar_remove_outliers("/home/tyler/Downloads/26875E239254N.las", output = "/home/tyler/Downloads/pma.las")
# 
# whitebox::lidar_tin_gridding("/home/tyler/Downloads/pma.las", output = "/home/tyler/Downloads/pma.tif", exclude_cls = c(7))
# pma_tif = raster::raster("~/Downloads/pma.tif")
# 
# pma_tif = matrix(raster::extract(pma_tif, raster::extent(pma_tif)), 
#                     nrow = ncol(pma_tif), ncol = nrow(pma_tif))
# 
# shadowmap = ray_shade(pma_tif, multicore = TRUE)
# 
# watertif = pma_tif
# watertif[pma_tif < 10] = 10
# watertif2 = pma_tif
# watertif2[pma_tif > 10] = 0
# watertif2[pma_tif < 10] = 1
# 
# watertif %>%
#   sphere_shade(sunangle=315,zscale=1/10) %>%
#   add_water(rayshader:::fliplr(watertif2), color = "lightblue") %>%
#   add_shadow(ray_shade(watertif, sunangle = 315, multicore = TRUE), 0.2) %>%
#   add_shadow(ambient_shade(watertif), 0.2) %>%
#   plot_map()
# 
# watertif %>%
#   sphere_shade(sunangle=315,zscale=1/10) %>%
#   add_water(rayshader:::fliplr(watertif2), color = "lightblue") %>%
#   add_shadow(ray_shade(watertif, sunangle = 315, multicore = TRUE), 0.2) %>%
#   save_png("pma")
# 
# watertif %>%
#   sphere_shade(sunangle=315,zscale=1/10) %>%
#   add_water(rayshader:::fliplr(watertif2), color = "lightblue") %>%
#   add_shadow(ray_shade(watertif, sunangle = 315, multicore = TRUE), 0.2) %>%
#   plot_3d(pma_tif,windowsize = c(1000,1000))
# render_camera(zoom=0.22, phi=25, theta=105,fov=70)
# 
# angles = seq(0,360,length.out = 721)[-721]
# 
# phivechalf = 30 + 60 * 1/(1 + exp(seq(-7, 20, length.out = 180)/2))
# phivecfull = c(phivechalf, rev(phivechalf))
# 
# for(i in 1:720) {
#   render_camera(zoom=0.4, phi=20, theta=315+angles[i],fov=70)
#   render_depth(focallength=50, focus=0.75, title_text = "Philadelphia Museum of Art",filename = glue::glue("pma{i}"))
# 
#   # render_depth(focus=0.75,filename = glue::glue("pma{i}"))
# }
```

``` r
# 
# tiffiles = list.files("~/Desktop/musa/coastal/", pattern = "tif") 
# loadedrasters = list()
# 
# for(i in 1:length(tiffiles)) {
#   loadedrasters[[i]] = raster(paste0("~/Desktop/musa/coastal/", tiffiles[i]))
# }
# 
# UTMtoLatLong<-function(x,y,zone){
#  xy <- data.frame(ID = 1:length(x), X = x, Y = y)
#  sp::coordinates(xy) <- c("X", "Y")
#  sp::proj4string(xy) <- sp::CRS(paste("+proj=utm +zone=",zone,"+ellps=WGS84",sep=''))  ## for example
#  res <- sp::spTransform(xy, sp::CRS(paste("+proj=longlat +datum=WGS84",sep='')))
#  return(as.data.frame(res))
# }
# 
# LongLatToUTM<-function(x,y,zone){
#  xy <- data.frame(ID = 1:length(x), X = x, Y = y)
#  coordinates(xy) <- c("X", "Y")
#  proj4string(xy) <- CRS("+proj=longlat +datum=WGS84")  ## for example
#  res <- spTransform(xy, CRS(paste("+proj=utm +zone=",zone," ellps=WGS84",sep='')))
#  return(as.data.frame(res))
# }
# 
# UTMtoLatLong(773763,3311393,15)
# UTMtoLatLong(789095,3329281,15)
# 
# corner1=LongLatToUTM(-90.085064,29.964426,15)
# corner2=LongLatToUTM(-90.052810,29.928880,15)
# 
# corner1
# fullraster = do.call(merge,loadedrasters)
# 
# extent(fullraster)
# fff = crop(fullraster,extent(corner1$X,corner2$X,corner2$Y,corner1$Y))
# extent(fff)
# 
# coastalmat = raster_to_matrix(fff)
# 
# small_la = reduce_matrix_size(coastalmat, 0.2)
# 
# dim(small_la)
# 
# coastalmat %>%
#   sphere_shade(colorintensity = 50) %>%
#   add_shadow(ray_shade(coastalmat,zscale=1/20,multicore=TRUE),0.3) %>%
#   plot_3d(coastalmat,zscale=1/20,windowsize = c(1000,1000),soliddepth=-50)
# 
# render_water(coastalmat2, zscale=1/20)
# 
# render_camera(theta=300,phi=30,zoom=0.6)
# render_water(coastalmat2, zscale=1/20,waterdepth=0, waterlinecolor = "white")
# 
# 
# coastalmat2 = coastalmat
# coastalmat2[is.na(coastalmat2)] = 0
```

Florida: <http://dpanther2.ad.fiu.edu/Lidar/lidarNew.php>

Global elevation data:
<http://opentopo.sdsc.edu/raster?opentopoID=OTSRTM.082015.4326.1>

Extreme scenario: 10.7 ft sea level rise, 2100

Issue: version \`GLIBC\_2.27’ not found (required by
/home/tyler/R/x86\_64-pc-linux-gnu-library/3.6/whitebox/WBT/whitebox\_tools)

``` r
# library(xml2)
# library(sp)
# library(rgdal)
# library(geoviz)
# setwd("~/Desktop/musa/")
# 
# download.file("http://mapping.ihrc.fiu.edu/FLDEM/monroe/lidar/CH2MHill_Block2_D2/las/zip/LID2007_133434_e.zip", destfile = "1.zip")
# download.file("http://mapping.ihrc.fiu.edu/FLDEM/monroe/lidar/CH2MHill_Block2_D2/las/zip/LID2007_133435_e.zip", destfile = "2.zip")
# download.file("http://mapping.ihrc.fiu.edu/FLDEM/monroe/lidar/CH2MHill_Block2_D2/las/zip/LID2007_133734_e.zip", destfile = "3.zip")
# download.file("http://mapping.ihrc.fiu.edu/FLDEM/monroe/lidar/CH2MHill_Block2_D2/las/zip/LID2007_133735_e.zip", destfile = "4.zip")
# 
# unzip("1.zip")
# unzip("2.zip")
# unzip("3.zip")
# unzip("4.zip")
# 
# list.files()
# 
# #Function
# LongLatToUTM<-function(x,y,zone){
#  xy <- data.frame(ID = 1:length(x), X = x, Y = y)
#  coordinates(xy) <- c("X", "Y")
#  proj4string(xy) <- CRS("+proj=longlat +datum=WGS84")  ## for example
#  res <- spTransform(xy, CRS(paste("+proj=utm +zone=",zone," ellps=WGS84",sep='')))
#  return(as.data.frame(res))
# }
# 
# library(whitebox)
# unzip("~/Desktop/musa/LID2007_118454_e.zip", exdir = "~/Desktop/musa/")
# unzip("~/Desktop/musa/LID2007_118455_e.zip", exdir = "~/Desktop/musa/")
# unzip("~/Desktop/musa/LID2007_118754_e.zip", exdir = "~/Desktop/musa/")
# unzip("~/Desktop/musa/LID2007_118755_e.zip", exdir = "~/Desktop/musa/")
# 
# lasfiles = list.files("~/Desktop/musa", pattern = ".las", full.names = TRUE)
# rasters = list()
# lidR::readLASheader(lasfiles[1])
# 
# for(i in 1:length(lasfiles)) {
#   wbt_lidar_remove_outliers(lasfiles[i], 
#                                output = glue::glue("/home/tyler/Desktop/musa/temp.las"))
#   wbt_lidar_tin_gridding("/home/tyler/Desktop/musa/temp.las", 
#                                output = glue::glue("/home/tyler/Desktop/musa/dem{i}.tif"),
#                                resolution = 1, verbose_mode = TRUE, exclude_cls = c(7,9))
#   rasters[[i]] = raster::raster(glue::glue("/home/tyler/Desktop/musa/dem{i}.tif"))
# }
# 
# fullmiami = raster::merge(rasters[[5]],rasters[[6]],rasters[[7]],rasters[[8]])
# 
# raster::crs(fullmiami) = CRS("+init=epsg:3601")
# 
# extent(fullmiami)
# 
# miamibeach = matrix(raster::extract(fullmiami, raster::extent(fullmiami), buffer = 1000),
#                nrow = ncol(fullmiami), ncol = nrow(fullmiami))
# 
# miamibeach = matrix(raster::extract(rasters[[4]], raster::extent(rasters[[4]]), buffer = 1000),
#                nrow = ncol(rasters[[4]]), ncol = nrow(rasters[[4]]))
# 
# 
# miamireduced = rayshader:::subsample_rect(miamibeach,1000,1000)
# dim(miamireduced)
# miamireduced %>%
#   sphere_shade() %>%
#   add_shadow(ray_shade(miamireduced, multicore = TRUE,zscale=5),max_darken = 0.2) %>%
#   plot_3d(miamireduced, water = TRUE, zscale=5, waterdepth = 3,zoom=0.5,phi=45,windowsize = c(1000,1000))
# render_water(miamireduced, zscale=2.5, waterdepth = 3)
# render_camera(fov=70)
# LongLatToUTM(y=c(25.760942, 25.779401),  x=c(-80.144299, -80.123726), zone = 17)
```